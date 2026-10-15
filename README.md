# DesenvolvaMais

Notas de estudo sobre arquitetura e resiliência de APIs com Spring (Spring Boot 4, Java 25, Resilience4j 2.4).

> Pesquisa compilada em outubro de 2026.

## Sumário

1. [Spring MVC vs Spring WebFlux e Virtual Threads](#1-spring-mvc-vs-spring-webflux-e-virtual-threads)
2. [Mecanismos de resiliência (Resilience4j)](#2-mecanismos-de-resiliência-resilience4j)
3. [Testando a resiliência quando a API externa cai](#3-testando-a-resiliência-quando-a-api-externa-cai)
4. [Cenários de risco no pagamento (Engenharia do Caos)](#4-cenários-de-risco-no-pagamento-engenharia-do-caos)
5. [Fontes](#fontes)

---

## 1. Spring MVC vs Spring WebFlux e Virtual Threads

| | **Spring MVC** | **Spring WebFlux** |
|---|---|---|
| Modelo | Uma thread por requisição (Servlet), parte do princípio de que o código bloqueia | Event loop não bloqueante, com poucas threads (em geral, uma por núcleo de CPU) |
| Servidor típico | Tomcat/Jetty, com pool grande (padrão de 200 threads no Tomcat) | Netty (também roda em Servlet) |
| Código | Imperativo, fácil de ler, depurar e testar | Reativo (`Mono`/`Flux`), curva de aprendizado bem maior |
| Dependências | Funciona com JPA, JDBC e qualquer biblioteca bloqueante | Exige tudo não bloqueante (R2DBC, WebClient). Um JDBC no meio quebra o modelo |
| Ponto forte | Simplicidade | Streaming, SSE, WebSockets e backpressure |

A própria documentação do Spring diz que, se você usa JPA ou JDBC, o MVC é a melhor escolha, e que a migração para WebFlux é desnecessária para muitas aplicações.

### Virtual Threads mudam a análise?

Sim, bastante. O principal motivo para adotar WebFlux era escalar chamadas de I/O sem esgotar o pool de threads. Com `spring.threads.virtual.enabled=true` (Boot 3.2+ e JDK 21+), o Tomcat atende cada requisição numa virtual thread. Bloquear passa a ser barato, e o MVC ganha essa escalabilidade sem abandonar o código imperativo.

Pontos de atenção:

- **Pinning resolvido no JDK 24+:** o JEP 491 fez com que `synchronized` não prenda mais a thread portadora, e o JDK 25 LTS já inclui essa correção. Ainda há pinning com código nativo (JNI/FFM), inicializadores de classe e I/O de arquivo local. Para detectar, use o evento JFR `jdk.VirtualThreadPinned`.
- **O gargalo muda de lugar:** sem o teto do pool de threads, quem satura é o pool de conexões (HikariCP), os file descriptors ou o limite da API externa. Por isso é preciso limitar a concorrência explicitamente, com Bulkhead ou `@ConcurrencyLimit` (ver seção 2).
- **ThreadLocal usado como cache deixa de funcionar,** porque cada tarefa ganha uma thread nova. Para contexto de requisição, prefira `ScopedValue` (finalizado no JDK 25).
- **Trabalho de CPU não ganha nada** com virtual threads.
- **A propriedade não tem efeito no WebFlux com Netty.**

### Conclusão

Para uma API REST típica, que acessa banco e chama APIs externas, MVC com Virtual Threads no JDK 25 é o padrão mais pragmático hoje. O WebFlux continua sendo a escolha certa quando o requisito é streaming ou backpressure, ou quando a stack já é toda reativa.

---

## 2. Mecanismos de resiliência (Resilience4j)

### Circuit Breaker e Retry (recapitulando)

O Circuit Breaker alterna entre os estados CLOSED, OPEN e HALF_OPEN. Abre quando a taxa de falhas ou de chamadas lentas passa de um limite e, enquanto está aberto, rejeita as chamadas com `CallNotPermittedException`. O Retry repete a chamada após uma falha, com espera fixa ou backoff exponencial.

### TimeLimiter, Bulkhead e RateLimiter

| Módulo | O que faz | Protege contra | Quando estoura |
|---|---|---|---|
| **TimeLimiter** | Define um tempo máximo para uma chamada **assíncrona** (`CompletableFuture`). Configura-se com `timeoutDuration` e `cancelRunningFuture` | Uma dependência lenta que prende recursos indefinidamente | `TimeoutException` |
| **Bulkhead** | Limita quantas chamadas rodam **ao mesmo tempo**. Há duas variantes: *Semaphore* (`maxConcurrentCalls`, padrão 25, e `maxWaitDuration`) e *ThreadPool* (pool fixo com fila) | Uma API lenta consumir todas as threads e conexões e derrubar o resto do sistema | `BulkheadFullException` |
| **RateLimiter** | Limita quantas chamadas são feitas **por período**: `limitForPeriod` a cada `limitRefreshPeriod`, esperando no máximo `timeoutDuration` por uma permissão | Estourar a cota da API externa, ou sobrecarga nos seus próprios endpoints | `RequestNotPermitted` |

A diferença essencial é que o Bulkhead controla **concorrência** (quantas chamadas simultâneas) e o RateLimiter controla **vazão** (quantas por segundo). Para chamadas síncronas com `RestClient`, configure também os timeouts de conexão e de leitura do cliente HTTP, porque o TimeLimiter atua sobre chamadas assíncronas.

### Ordem dos decorators no Spring Boot

Ordem padrão das anotações: `Retry( CircuitBreaker( RateLimiter( TimeLimiter( Bulkhead( chamada )))))`. O Retry fica por fora, então cada tentativa passa pelo Circuit Breaker. Dá para mudar a ordem com as propriedades `*AspectOrder`.

```java
@Retry(name = "pagamentos")
@CircuitBreaker(name = "pagamentos", fallbackMethod = "cobrancaPendente")
@Bulkhead(name = "pagamentos")
public CobrancaResponse cobrar(CobrancaRequest req) { ... }
```

### Versões

- O Resilience4j 2.4.0 lançou o `resilience4j-spring-boot4`. Ele ficou fora do BOM nessa versão, então pode ser preciso declarar a versão do artefato explicitamente.
- O Spring Framework 7 passou a ter `@Retryable` e `@ConcurrencyLimit` nativos (habilitados com `@EnableResilientMethods`), mas não tem Circuit Breaker nem RateLimiter. Para esses, o Resilience4j continua necessário.

### Idempotency-Key num POST

**O problema:** POST não é idempotente. Se a chamada der timeout e o cliente fizer retry, a mesma cobrança pode ser feita duas vezes. **Retry em POST sem idempotência é perigoso.**

**Como funciona:**

1. O cliente gera um UUID v4 por **operação lógica** e o envia no header `Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"`. Em todo retry daquela operação, reenvia **a mesma chave**.
2. O servidor procura a chave, junto com um hash do corpo da requisição e o escopo do cliente:
   - **Chave nova:** reserva a chave de forma atômica (estado PROCESSANDO), executa a operação e guarda o status e o corpo da resposta.
   - **Chave já concluída, mesmo payload:** devolve a resposta guardada sem executar de novo, inclusive se a resposta original tiver sido um erro.
   - **Chave ainda em processamento:** responde **409 Conflict**.
   - **Mesma chave com payload diferente:** responde **422 Unprocessable Content**.
   - **Header ausente numa operação que o exige:** responde **400 Bad Request**.
3. As chaves expiram após um tempo de vida (TTL). A Stripe, por exemplo, pode descartá-las depois de pelo menos 24 horas, e não guarda o resultado se a validação falhar antes da execução.

A especificação do IETF (`draft-ietf-httpapi-idempotency-key-header`) expirou na versão 07 sem virar RFC, mas o padrão é amplamente usado na prática.

**Implementação:** uma tabela `idempotency_keys` com restrição de unicidade em (`chave`, `cliente`) ou um `SETNX` com TTL no Redis. Sempre que possível, grave o resultado na mesma transação da operação de negócio.

---

## 3. Testando a resiliência quando a API externa cai

Os testes podem ser organizados em três camadas.

### a) WireMock (testes de integração)

Simula a API externa no nível HTTP:

- `serverError()` e respostas 503 servem para verificar o fallback e a abertura do circuito.
- `withFixedDelay(...)` maior que o seu timeout testa o TimeLimiter, o read timeout e o limite de chamadas lentas.
- `Fault.CONNECTION_RESET_BY_PEER`, `EMPTY_RESPONSE`, `MALFORMED_RESPONSE_CHUNK` e `RANDOM_DATA_THEN_CLOSE` simulam conexões quebradas. O `CONNECTION_RESET_BY_PEER` pode travar no Windows.
- Os **Scenarios** permitem simular "falha duas vezes e depois responde 200", útil para validar o Retry.
- O `verify(...)` confere quantas chamadas chegaram à API, o que mostra se o circuito aberto realmente parou de chamá-la, e se os retries de POST reenviaram o mesmo `Idempotency-Key`.

```java
@RegisterExtension
static WireMockExtension api = WireMockExtension.newInstance()
    .options(wireMockConfig().dynamicPort()).build();

@Test
void abreCircuitoEUsaFallbackQuandoApiCai() {
    api.stubFor(post("/cobrancas").willReturn(serverError()));
    for (int i = 0; i < 10; i++) {
        assertThat(service.cobrar(req()).status()).isEqualTo("PENDENTE"); // fallback
    }
    assertThat(cbRegistry.circuitBreaker("pagamentos").getState()).isEqualTo(State.OPEN);
}

@Test
void apiLentaNaoTravaNossoSistema() {
    api.stubFor(post("/cobrancas").willReturn(ok().withFixedDelay(5_000)));
    assertTimeout(Duration.ofSeconds(3), () -> service.cobrar(req()));
}
```

### b) Toxiproxy com Testcontainers

Injeta falhas no nível de rede (TCP), entre a aplicação e containers reais como o banco, o Redis ou um WireMock em container. Os "toxics" disponíveis incluem `latency` com jitter, `bandwidth` igual a 0 nos dois sentidos para cortar a conexão, `timeout`, `slicer` e `limitData`. Depois você remove o toxic e verifica que o sistema se recupera. É ideal para testar o comportamento do pool de conexões.

### c) Chaos Monkey for Spring Boot

Versão 4.0.0, compatível com Boot 4, voltado para homologação. Injeta latência e exceções nos seus próprios beans (`@Service`, `@Repository`, `@RestController`) e nas chamadas de saída feitas com `RestTemplate` ou `WebClient`. Também tem ataques de runtime (derrubar a aplicação, encher a memória, carregar a CPU). Para ativar, use o perfil `chaos-monkey` e `chaos.monkey.enabled=true`. Os ataques podem ser ligados e desligados pelo endpoint do Actuator.

### Próximo passo

Combinar teste de carga (Gatling ou k6) com essas falhas e acompanhar as métricas do Resilience4j no Actuator/Micrometer. Assim você vê se as threads e os pools saturam quando a dependência fica lenta, que é exatamente o cenário em que Virtual Threads sem Bulkhead podem dar problema.

---

PERGUNTAS:
## 4. Cenários de risco no pagamento (Engenharia do Caos)

Na Engenharia do Caos, você define como o sistema deveria se comportar, injeta a falha de propósito e verifica se esse comportamento se manteve. Estas são as perguntas de risco para o fluxo de pagamento.

### 4.1 "E se o gateway aprovar a cobrança, mas a resposta se perder no caminho? Marcamos como recusado um pagamento que foi aprovado?"

- **Risco:** o cliente é cobrado, mas o pedido aparece como recusado. Se ele tentar de novo e o sistema gerar uma nova Idempotency-Key, ele é cobrado duas vezes. No melhor caso, sobra um estorno manual para o suporte.

**R:** Não. No nosso desenho, timeout não é recusa. Um resultado desconhecido vira `PENDENTE_CONFIRMACAO`, e só o gateway pode dizer se a cobrança passou.

- **Antes da chamada:** o pagamento é gravado como `PROCESSANDO`, junto com a Idempotency-Key que será enviada ao gateway, numa transação curta que já faz commit.
- **Na resposta:** uma aprovação vira `APROVADO` e uma recusa explícita vira `RECUSADO`. Qualquer resultado incerto vira `PENDENTE_CONFIRMACAO`: read timeout, connection reset, 5xx depois dos retries, circuito aberto ou bulkhead cheio. Nesses dois últimos casos, uma tentativa anterior pode ter chegado ao gateway. O cliente vê "pagamento em processamento", nunca "recusado".
- **Resolução:** o webhook do gateway, ou um job de conciliação que roda a cada minuto, consulta a cobrança pela mesma chave. Se o gateway não conhecer a cobrança depois de uma janela de segurança (por exemplo, 15 min), o pagamento vira `FALHOU`, porque a cobrança nunca aconteceu, e o cliente pode tentar de novo.
- **Retry do cliente:** o frontend mantém a mesma Idempotency-Key enquanto o pedido não termina. Por isso, uma nova tentativa recebe o estado atual em vez de gerar outra cobrança.

```java
try {
    var resposta = gateway.cobrar(pagamento.idempotencyKey(), pedido);
    pagamento.mudarPara(resposta.aprovada() ? APROVADO : RECUSADO);
} catch (ResourceAccessException | HttpServerErrorException
         | CallNotPermittedException | BulkheadFullException e) {
    pagamento.mudarPara(PENDENTE_CONFIRMACAO); // resultado desconhecido: nunca RECUSADO
}
```

**Teste:** o stub do `POST /cobrancas` atrasa 5 s, e o stub de consulta responde `APROVADO`. O teste verifica três coisas: que o checkout devolve `PENDENTE_CONFIRMACAO`, que todos os POSTs ao gateway levaram a mesma chave (`findAll(postRequestedFor(...))`), e que, depois de rodar a conciliação, o pagamento está `APROVADO`.


### 4.2 "E se o gateway ficar lento (5 s por resposta) no pico de vendas? As threads e conexões ficam presas e o sistema inteiro cai junto?"

- **Risco:** efeito cascata. Com Virtual Threads, nada limita a concorrência, e o pool do HikariCP se esgota. Isso piora se a transação do banco ficar aberta durante a chamada HTTP. Além disso, os retries multiplicam a carga sobre um gateway que já está degradado (*retry storm*).

**R:** Não. A lentidão fica contida no checkout, e o resto do sistema continua respondendo.

- **Ninguém espera 5 s:** o read timeout do `RestClient` é de 3 s.
- **O circuito abre rápido:** chamadas acima de 2 s contam como lentas. Quando elas passam de 50% numa janela de 50 chamadas, o circuito abre, e o checkout passa a responder na hora com "pagamento em processamento" (`PENDENTE_CONFIRMACAO`, reprocessado depois).
- **Concorrência limitada:** o Bulkhead permite no máximo 200 chamadas simultâneas ao gateway, com espera de 50 ms. O excedente recebe resposta imediata em vez de se acumular.
- **Sem retry storm:** são no máximo 3 tentativas, com backoff exponencial (com jitter, via um `intervalFunction` randomizado), e nenhum retry quando o circuito está aberto ou o bulkhead está cheio. A carga sobre o gateway multiplica por no máximo 3, e só enquanto o circuito está fechado.
- **Banco fora da espera:** o fluxo usa duas transações curtas, uma para gravar `PROCESSANDO` e outra para gravar o resultado. A chamada HTTP acontece entre as duas, sem prender nenhuma conexão do HikariCP. Um `@Transactional` envolvendo a chamada ao gateway seguraria uma conexão por 5 s e esgotaria o pool do sistema inteiro.

```yaml
resilience4j:
  circuitbreaker.instances.gateway:
    slidingWindowSize: 50
    failureRateThreshold: 50
    slowCallDurationThreshold: 2s
    slowCallRateThreshold: 50
    waitDurationInOpenState: 30s
  bulkhead.instances.gateway:
    maxConcurrentCalls: 200
    maxWaitDuration: 50ms
  retry.instances.gateway:
    maxAttempts: 3
    waitDuration: 500ms
    enableExponentialBackoff: true
    exponentialBackoffMultiplier: 2
    ignoreExceptions:
      - io.github.resilience4j.circuitbreaker.CallNotPermittedException
      - io.github.resilience4j.bulkhead.BulkheadFullException
```

**Teste:** o Toxiproxy aplica `latency` de 5000 ms na frente do gateway, e o k6 gera carga no checkout e no catálogo ao mesmo tempo. Os critérios de sucesso são:

- p99 do catálogo igual ao baseline;
- `hikaricp_connections_pending` em zero;
- `resilience4j_circuitbreaker_state` indo para `OPEN` em segundos;
- checkout respondendo em menos de 100 ms depois que o circuito abre.


### 4.3 "E se o webhook de confirmação chegar duplicado, atrasado ou fora de ordem (o 'estornado' antes do 'aprovado')?"

- **Risco:** o pedido é liberado duas vezes, o estoque é baixado em dobro, ou um pagamento estornado volta para "aprovado".

**R:** Não. O webhook é processado de forma idempotente e só pode mover o pagamento para a frente.

- **Autenticidade:** a assinatura HMAC do webhook é validada antes de tudo. Assinatura inválida recebe 401.
- **Duplicado:** o ID do evento é inserido em `eventos_processados` com `INSERT ... ON CONFLICT DO NOTHING`. Se o ID já existia, a API responde 200 sem efeito nenhum, para o gateway parar de reenviar.
- **Fora de ordem:** a máquina de estados só aceita transições válidas, e a mesma regra vale para o checkout, a conciliação e o webhook. Um estorno é verdade final, então é aceito mesmo vindo de `PROCESSANDO` ou `PENDENTE_CONFIRMACAO`. Já um "aprovado" que chega depois do "estornado" é ignorado e registrado em log.
- **Concorrência com o POST:** o pagamento tem uma coluna `@Version`. Se o webhook e o checkout gravarem ao mesmo tempo, um deles recebe conflito, a transação inteira é desfeita (inclusive o registro do evento) e a API responde 5xx. O gateway então reenvia o webhook, que é reprocessado sobre o estado já atualizado.
- **Efeitos colaterais exatamente uma vez:** o registro do evento, a mudança de estado e uma entrada na tabela de outbox são gravados na mesma transação. O e-mail e a baixa de estoque saem do outbox, com consumidores que deduplicam pelo ID da mensagem.

```java
enum StatusPagamento {
    PROCESSANDO, PENDENTE_CONFIRMACAO, APROVADO, RECUSADO, FALHOU, ESTORNADO;

    private static final Map<StatusPagamento, Set<StatusPagamento>> TRANSICOES = Map.of(
        PROCESSANDO,          Set.of(APROVADO, RECUSADO, PENDENTE_CONFIRMACAO, ESTORNADO),
        PENDENTE_CONFIRMACAO, Set.of(APROVADO, RECUSADO, FALHOU, ESTORNADO),
        APROVADO,             Set.of(ESTORNADO),
        RECUSADO,             Set.of(),
        FALHOU,               Set.of(),
        ESTORNADO,            Set.of());

    boolean podeIrPara(StatusPagamento destino) {
        return TRANSICOES.get(this).contains(destino);
    }
}

@Transactional
public void processar(EventoGateway evento) {
    if (!eventos.registrarSeNovo(evento.id())) return;            // duplicado
    var pagamento = pagamentos.buscarPorChave(evento.idempotencyKey());
    if (!pagamento.status().podeIrPara(evento.status())) return;  // fora de ordem (logado)
    pagamento.mudarPara(evento.status());                         // @Version: conflito -> rollback
    outbox.registrar(new PagamentoMudouDeStatus(pagamento.id(), evento.status()));
}
```

**Teste:** o teste envia o mesmo webhook 3 vezes, envia `ESTORNADO` antes de `APROVADO`, e dispara o webhook enquanto o POST ainda está em andamento (WireMock com delay no gateway). Ele verifica que o estado final é `ESTORNADO`, que existe uma linha em `eventos_processados` por evento, e que os mocks de e-mail e de estoque foram chamados exatamente uma vez por mudança de estado.


---

## Fontes

- [Spring Framework: WebFlux, Applicability](https://docs.spring.io/spring-framework/reference/web/webflux/new-framework.html)
- [InfoQ: Virtual Threads after JDK 24](https://www.infoq.com/articles/virtual-threads-after-jdk24/)
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [BellSoft: Guide to Virtual Threads with Spring Boot](https://bell-sw.com/blog/a-guide-to-using-virtual-threads-with-spring-boot/)
- [Spring Framework 7: Resilience Features](https://docs.spring.io/spring-framework/reference/core/resilience.html)
- Resilience4j: [CircuitBreaker](https://resilience4j.readme.io/docs/circuitbreaker) · [Bulkhead](https://resilience4j.readme.io/docs/bulkhead) · [RateLimiter](https://resilience4j.readme.io/docs/ratelimiter) · [TimeLimiter](https://resilience4j.readme.io/docs/timeout) · [Spring Boot](https://resilience4j.readme.io/docs/getting-started-3)
- [Resilience4j: Spring Boot 4 support (issue #2371)](https://github.com/resilience4j/resilience4j/issues/2371)
- [IETF: The Idempotency-Key HTTP Header Field (draft-07)](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header-07)
- [Stripe: Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- WireMock: [Simulating Faults](https://wiremock.org/docs/simulating-faults/) · [Stateful Behaviour](https://wiremock.org/docs/stateful-behaviour/)
- [Testcontainers: Toxiproxy Module](https://java.testcontainers.org/modules/toxiproxy/)
- [Chaos Monkey for Spring Boot: Reference Guide](https://codecentric.github.io/chaos-monkey-spring-boot/latest/)


## 1. Escolha de Paradigma da API

- **A Escolha:** (X) Spring MVC com Virtual Threads ou ( ) Spring WebFlux (Reativo)?
- **Justificativa:** Considerando a capacidade de suportar 10k acessos concorrentes, por que você fez essa escolha técnica?
  > O checkout é um fluxo request-response dominado por I/O bloqueante: grava no banco (JPA/JDBC) e espera a resposta do parceiro de pagamentos. Na maior parte do tempo, cada requisição está apenas esperando. Com Virtual Threads (`spring.threads.virtual.enabled=true`, JDK 25), cada uma das 10k requisições concorrentes ganha sua própria virtual thread. Ela custa poucos KB e libera a thread portadora enquanto espera o I/O. O teto de 200 threads do Tomcat deixa de existir, e chegamos à mesma escalabilidade de I/O do WebFlux sem reescrever o sistema em estilo reativo.
  >
  > Escolhi MVC porque:
  > - o código continua imperativo: mais simples de ler, depurar e testar, com stack traces legíveis e `@Transactional` funcionando normalmente;
  > - o WebFlux exigiria uma stack 100% não bloqueante (R2DBC, WebClient), e um único driver bloqueante no event loop anularia a vantagem;
  > - não temos requisitos de streaming nem de backpressure, que são os casos em que o WebFlux realmente ganha;
  > - no JDK 24+ (JEP 491), `synchronized` não prende mais a thread portadora, o que remove o principal risco das virtual threads no Java 21.
  >
  > Cuidados para os 10k concorrentes: as virtual threads removem o limite de threads, mas não o limite dos recursos. 10k requisições não cabem num pool de ~50 conexões do HikariCP nem na cota do parceiro. Por isso:
  > - limitamos explicitamente as chamadas ao parceiro com Bulkhead (seção 2);
  > - nunca seguramos conexão ou transação do banco enquanto esperamos o parceiro;
  > - ajustamos `server.tomcat.max-connections` (padrão 8192) para acima de 10k;
  > - validamos o dimensionamento com teste de carga (k6 ou Gatling com 10k usuários virtuais).

## 2. Desenho de Resiliência

Dependemos de uma API terceira instável. Quais recursos do padrão de resiliência você vai utilizar?

- **Proteção contra Falhas e Lentidão:**
  Quais padrões (Circuit Breaker, TimeLimiter, RateLimiter, Bulkhead, Retry) você implementaria na chamada para o Parceiro de Pagamentos? Por que?
  > Uso todos, cada um contra um tipo de falha, na ordem `Retry( CircuitBreaker( RateLimiter( Bulkhead( chamada com timeout ))))`:
  >
  > - **Timeout / TimeLimiter:** a chamada é síncrona (`RestClient` numa virtual thread), então o limite de tempo fica no próprio cliente HTTP (connect de ~1 s e read de ~3 s). O TimeLimiter do Resilience4j só atua sobre `CompletableFuture` e seria usado apenas se a chamada fosse assíncrona. Sem timeout, um parceiro lento prende threads e conexões indefinidamente.
  > - **Circuit Breaker:** abre quando a taxa de falhas passa de 50% ou quando as chamadas acima de 3 s viram maioria (`slowCallRateThreshold`), numa janela das últimas ~50 chamadas. Aberto, ele falha na hora, sem gastar recursos com um parceiro fora do ar, e dá tempo para o parceiro se recuperar. O fallback nunca diz "aprovado": o pagamento fica `PENDENTE` e é reprocessado de forma assíncrona. Se o negócio não aceitar pedido sem autorização imediata, o checkout responde 503 com `Retry-After`.
  > - **Bulkhead (semáforo):** limita as chamadas simultâneas ao parceiro, com o limite dimensionado pela capacidade dele (por exemplo, 200) e espera curta. É essencial com Virtual Threads: sem ele, as 10k requisições iriam todas ao mesmo tempo para o parceiro, esgotariam as conexões e derrubariam junto o resto do sistema.
  > - **Retry:** no máximo 3 tentativas no total, com backoff exponencial e jitter, apenas para erros transitórios (timeout de conexão, connection reset, 502/503/504). Nunca retenta 4xx (cartão recusado, dados inválidos), nem `CallNotPermittedException` ou `BulkheadFullException`. Todo retry reenvia a **mesma** `Idempotency-Key`.
  > - **RateLimiter:** mantém nosso volume abaixo da cota contratada com o parceiro, evitando respostas 429 ou bloqueio. Se o parceiro não publicar limite, o Bulkhead já basta.
  >
  > As métricas do Resilience4j vão para o Actuator/Micrometer, com alerta quando o circuito abrir.

- **Proteção contra Duplicidade:**
  Como a `Idempotency-Key` será utilizada na requisição do Checkout para evitar cobranças duplas? Qual o ciclo de vida dessa chave?
  > A chave protege os dois trechos: do frontend para o nosso checkout, e do checkout para o parceiro.
  >
  > **Uso na requisição:**
  > 1. **Geração:** o frontend gera um UUID v4 quando o checkout é aberto, uma chave por tentativa de compra e não por clique. Duplo clique, refresh e retries de rede reenviam a mesma chave no header `Idempotency-Key` do `POST /checkout`.
  > 2. **Recebimento:** a API exige o header (400 se faltar) e valida o corpo antes de reservar a chave, então erros de validação não ficam gravados. Em seguida, calcula o hash SHA-256 do corpo e tenta inserir (`chave`, `cliente`, `hash`, status `PROCESSANDO`) na tabela `idempotency_keys`, que tem restrição de unicidade em (`chave`, `cliente`):
  >    - se inseriu, a chave é nova e o processamento segue;
  >    - se já existe, está concluída e tem o mesmo hash, a API devolve a resposta armazenada sem cobrar de novo;
  >    - se já existe e ainda está `PROCESSANDO`, responde 409 Conflict;
  >    - se já existe com hash diferente, responde 422 Unprocessable Content.
  > 3. **Chamada ao parceiro:** o checkout repassa ao parceiro uma chave estável (a mesma chave ou o ID do pagamento), e o Retry do Resilience4j reutiliza essa chave. Assim, se a cobrança passou e só a resposta se perdeu, o parceiro reconhece a repetição e não cobra duas vezes.
  > 4. **Conclusão:** o status e o corpo da resposta são gravados junto com o estado do pagamento, na mesma transação, e a chave vira `CONCLUIDA`. Se o resultado for incerto (timeout), o pagamento fica `PENDENTE_CONFIRMACAO`. Um retry do cliente com a mesma chave recebe esse estado, e a conciliação (consulta ao parceiro ou webhook) resolve o resultado final.
  >
  > **Ciclo de vida:** `gerada no cliente → PROCESSANDO → CONCLUIDA (resposta armazenada) → expirada após TTL de 24 h`. O TTL precisa ser maior que a janela de retries do cliente. Um job remove as chaves expiradas, e uma requisição que chegar depois disso é tratada como nova.
  >
  > Como última linha de defesa, um índice único parcial garante no máximo um pagamento aprovado por pedido. Assim, nem um bug na camada de idempotência gera duas cobranças para o mesmo pedido.
