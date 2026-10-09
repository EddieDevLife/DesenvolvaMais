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

## 4. Cenários de risco no pagamento (Engenharia do Caos)

Na Engenharia do Caos, você define como o sistema deveria se comportar, injeta a falha de propósito e verifica se esse comportamento se manteve. Estas são as perguntas de risco para o fluxo de pagamento.

### 4.1 "E se o gateway aprovar a cobrança, mas a resposta se perder no caminho? Marcamos como recusado um pagamento que foi aprovado?"

- **Risco:** o cliente é cobrado, mas o pedido aparece como recusado. Se ele tentar de novo e o sistema gerar uma nova Idempotency-Key, ele é cobrado duas vezes. No melhor caso, sobra um estorno manual para o suporte.
- **Experimento:** no WireMock, configure o `POST /cobrancas` com `withFixedDelay` maior que o read timeout, ou com `Fault.CONNECTION_RESET_BY_PEER`. Ao mesmo tempo, faça o stub de consulta do gateway responder "APROVADO" para aquela chave.
- **Defesa esperada:** timeout não significa falha. O pagamento deve ir para `PENDENTE_CONFIRMACAO`, e só vira `RECUSADO` com uma recusa explícita do gateway. Um job de conciliação consulta o gateway usando a mesma chave, e o webhook de confirmação também resolve o estado.
- **Como validar:** nenhum pagamento termina `RECUSADO` depois de um timeout, e após a conciliação o estado local é igual ao estado no gateway.

### 4.2 "E se o gateway ficar lento (5 s por resposta) no pico de vendas? As threads e conexões ficam presas e o sistema inteiro cai junto?"

- **Risco:** efeito cascata. Com Virtual Threads, nada limita a concorrência, e o pool do HikariCP se esgota. Isso piora se a transação do banco ficar aberta durante a chamada HTTP. Além disso, os retries multiplicam a carga sobre um gateway que já está degradado (*retry storm*).
- **Experimento:** use o Toxiproxy com `latency` de 5000 ms entre a aplicação e o gateway (ou o `withLogNormalRandomDelay` do WireMock), combinado com um teste de carga no k6 ou no Gatling.
- **Defesa esperada:**
  - timeouts curtos no cliente HTTP;
  - Bulkhead limitando as chamadas simultâneas ao gateway;
  - Circuit Breaker com `slowCallRateThreshold`;
  - retry com backoff, jitter e poucas tentativas, configurado para não retentar `CallNotPermittedException`;
  - nunca segurar conexão ou transação do banco durante a chamada externa.
- **Como validar:** os endpoints que não dependem do pagamento (catálogo, consulta de pedido) mantêm o p99 normal, o pool de conexões não satura, o circuito abre e o checkout responde rápido com "pagamento em processamento".

### 4.3 "E se o webhook de confirmação chegar duplicado, atrasado ou fora de ordem (o 'estornado' antes do 'aprovado')?"

- **Risco:** o pedido é liberado duas vezes, o estoque é baixado em dobro, ou um pagamento estornado volta para "aprovado".
- **Experimento:** reenvie o mesmo evento várias vezes, inverta a ordem dos eventos, atrase a entrega, e dispare o webhook enquanto o POST original ainda está em processamento.
- **Defesa esperada:**
  - deduplicação pelo ID do evento, com uma tabela de eventos processados com restrição de unicidade;
  - máquina de estados que só aceita transições válidas (`APROVADO → ESTORNADO`, nunca o contrário);
  - lock otimista (coluna de versão) na linha do pagamento;
  - validação da assinatura do webhook.
- **Como validar:** depois do replay e da reordenação, o estado final está correto, e os efeitos colaterais (e-mail, baixa de estoque) acontecem exatamente uma vez.

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
