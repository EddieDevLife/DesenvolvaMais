# Template de System Design: Aula 05 - Containers, CI/CD & Deploy sem Downtime

> **Contexto do Cenário: O Deploy que Quebrou o Checkout e o Rollback Impossível**
>
> Sua equipe está reestruturando a engenharia de release do serviço de **Checkout** após um incidente severo na Black Friday:
>
> 1. **O Incidente:** Para subir uma otimização urgente na Saga, o pipeline de produção recompilou a branch `main`, sobrescreveu a tag `checkout:latest` e iniciou a atualização. No boot da primeira instância nova, o **Flyway** renomeou a coluna `status_pagamento` para `estado_saga`.
> 2. **O Efeito em Cadeia:** Os 80% de servidores que ainda rodavam a versão anterior passaram a falhar imediatamente (`column "status_pagamento" does not exist`). Ao tentar acionar o rollback puxando `:latest`, a equipe descobriu que a imagem anterior havia sido sobrescrita e, mesmo restaurando um backup do binário antigo, ele continuou falhando porque o banco já estava alterado.
> 3. **A Carga:** 5.000 TPS ativos com múltiplas réplicas de aplicação conectadas ao mesmo banco PostgreSQL.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Empacotamento, Superfície de Ataque e Promoção Imutável

### 1.1 Anatomia da Imagem e Redução de Superfície de Ataque

- **Construção vs Execução:** Como você estruturará o empacotamento do serviço de Checkout para produção? Especifique o que deve ser usado na etapa de compilação, o que deve (e o que não deve) entrar na imagem final de execução, com qual permissão de usuário o processo rodará e qual o trade-off da sua escolha de imagem base para a segurança e para o diagnóstico em produção.
  > **Build multi-stage:** a imagem é construída em dois estágios, e só o segundo vai para produção.
  >
  > - **Estágio de compilação:** imagem com JDK 25 e Maven/Gradle, com as dependências em cache. Ele compila, roda os testes e gera o jar. Aqui ficam o código-fonte, o compilador, o gerenciador de pacotes e os arquivos de teste, e nada disso viaja para o estágio final.
  > - **Estágio de execução:** apenas um JRE mínimo (por exemplo, distroless Java 25 ou uma imagem JRE enxuta) com o jar em camadas (`layertools` do Spring Boot: dependências, loader, snapshot e aplicação em camadas separadas, para que o deploy de uma mudança só no código reutilize as camadas de dependências).
  > - **O que entra na imagem final:** o jar, os certificados necessários e nada mais.
  > - **O que não entra:** JDK, Maven/Gradle, código-fonte, testes, `.git`, caches de build, shell e gerenciador de pacotes (quando a base permitir), e **nenhum segredo** (credenciais chegam em runtime, ver 1.2). Um `.dockerignore` impede que `.git`, `.env` e arquivos locais sejam copiados.
  > - **Usuário:** o processo roda como **não-root**, com UID/GID fixo (por exemplo `10001`), definido com `USER` no Dockerfile. No Kubernetes, reforçamos com `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true` (com um `emptyDir` para `/tmp`) e `capabilities: drop: [ALL]`. Se alguém explorar uma falha na aplicação, ganha um usuário sem privilégios num filesystem somente leitura.
  > - **Base fixada por digest** (`FROM imagem@sha256:...`) e atualizada de forma controlada (Renovate/Dependabot), para que o build seja reproduzível e uma atualização da base seja uma mudança revisada.
  >
  > ```dockerfile
  > FROM eclipse-temurin:25-jdk AS build
  > WORKDIR /src
  > COPY . .
  > RUN ./mvnw -B -ntp verify && java -Djarmode=tools -jar target/checkout.jar extract --layers --destination /layers
  >
  > FROM gcr.io/distroless/java25-debian12:nonroot
  > WORKDIR /app
  > COPY --from=build /layers/dependencies/ ./
  > COPY --from=build /layers/spring-boot-loader/ ./
  > COPY --from=build /layers/snapshot-dependencies/ ./
  > COPY --from=build /layers/application/ ./
  > USER 65532:65532
  > ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "org.springframework.boot.loader.launch.JarLauncher"]
  > ```
  >
  > **Trade-off da imagem base:**
  > - **Distroless (ou equivalente mínimo):** sem shell, sem `apt`, sem `curl`. Reduz a superfície de ataque e o número de CVEs a tratar, e dificulta que um invasor "viva do ambiente". O preço é o diagnóstico: não dá para entrar com `exec` e investigar.
  > - **Imagem com shell (slim/Alpine):** diagnóstico mais fácil, mas mais pacotes, mais CVEs e mais ferramentas à disposição de um atacante. Alpine ainda usa `musl`, que pode se comportar de forma diferente da `glibc` com a JVM.
  > - **Como diagnosticamos mesmo assim:** com container efêmero de depuração (`kubectl debug` com uma imagem de ferramentas anexada ao pod, sem alterar a imagem de produção), Actuator, métricas, logs estruturados, JFR e heap dump gravados num volume. A observabilidade vem pronta no serviço, em vez de depender de shell no container.

### 1.2 Rastreabilidade, Identidade do Artefato e Parametrização

- **Da Homologação à Produção:** Como sua esteira garante que o binário executado em produção é exatamente o mesmo que passou nos testes de homologação? Explique como as configurações mudam entre ambientes (dev, staging, prod) sem recompilar o artefato e como você identifica a versão no registry para que um rollback nunca puxe um binário sobrescrito por engano.
  > **Construir uma vez, promover o mesmo artefato.** A causa raiz do incidente foi recompilar a `main` a cada deploy e reusar a tag `latest`: o que foi testado e o que foi para produção eram coisas diferentes, e a versão anterior deixou de existir.
  >
  > - **Um build por commit:** o pipeline constrói a imagem **uma única vez**, no merge, e a publica com uma tag imutável derivada da versão e do commit (`checkout:2.14.0-a1b2c3d`). A tag `latest` não é usada em lugar nenhum.
  > - **Identidade pelo digest:** o registry devolve o digest (`sha256:...`), que é a identidade real do artefato. Os manifestos de deploy referenciam `checkout@sha256:...`, então não existe como "a mesma tag apontar para outro binário".
  > - **Tags imutáveis no registry:** a opção de *tag immutability* fica ligada (ECR, Harbor, Artifactory, GHCR via política). Tentar sobrescrever uma tag existente falha.
  > - **Promoção sem rebuild:** staging e produção recebem o **mesmo digest**. Promover é alterar a referência no repositório de manifestos (GitOps) ou adicionar uma tag de ambiente ao mesmo digest. Os testes de homologação rodam contra o digest que depois vai para produção.
  > - **Assinatura e procedência:** o pipeline assina a imagem (cosign), anexa o SBOM e a atestação de build (SLSA), e um *admission controller* no cluster só aceita imagens assinadas pelo nosso pipeline, por digest.
  > - **Retenção protegida:** a política do registry mantém no mínimo as últimas N versões já liberadas em produção, e as imagens em uso nunca são apagadas por limpeza automática.
  >
  > **Configuração por ambiente, sem recompilar (12-factor):**
  > - O artefato é idêntico em todos os ambientes. O que muda é injetado em runtime: variáveis de ambiente, `ConfigMap` e Spring profiles/`spring.config.import`, mantidos no repositório de configuração por ambiente (`dev/`, `staging/`, `prod/`).
  > - **Segredos** (senha do banco, chaves do PSP) vêm de um cofre (Vault, AWS Secrets Manager) por um operador (External Secrets) ou volume montado. Nunca entram na imagem, no Git ou em variáveis de build.
  > - **Feature flags** separam deploy de liberação: o código novo pode estar em produção desligado.
  >
  > **Rollback sem surpresa:** a versão de produção é registrada em Git (o commit que muda o digest). Voltar é reverter esse commit, e o digest anterior ainda está no registry, imutável e assinado. Se o pipeline tentar publicar uma tag que já existe, ele falha, em vez de sobrescrever.

---

## 2. Estratégia de Rollout e Migração de Banco sem Downtime

### 2.1 O Modelo de Rollout em Produção

- **A Escolha:** (X) Canary Release (Liberação Gradual por Percentual)  ( ) Blue-Green Deployment (Alternância de Ambientes)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Canary Release:* Como o tráfego evoluirá sob 5.000 TPS, quais sinais definirão o avanço ou a reversão (rollback) automática e como lidar com o tempo em que duas versões diferentes da API atendem clientes simultaneamente?
  - *Se você escolheu Blue-Green Deployment:* Como você gerenciará o custo de infraestrutura duplicada no pico da Black Friday, o aquecimento das novas instâncias (conexões/cache) antes da virada de chave e o risco de virar 100% do tráfego de uma só vez?
  > **Por que canary:** no incidente, a versão ruim chegou a todas as réplicas antes que alguém pudesse reagir. O canary limita o dano pelo tamanho da amostra: um problema na `v2` afeta, no pior caso, uma fração pequena do tráfego, e a reversão acontece antes de ela crescer. O blue-green exigiria duplicar toda a infraestrutura no pico da Black Friday e transferiria 100% do tráfego de uma vez, o que concentra o risco justamente no momento mais caro de errar. Também importa que, num blue-green, a `v2` e a `v1` compartilhariam o mesmo banco, então o problema de compatibilidade de schema continua existindo.
  >
  > **Evolução do tráfego (executada por Argo Rollouts ou Flagger):**
  >
  > | Etapa | Tráfego na `v2` | Em 5.000 TPS | Pausa e análise |
  > |---|---|---|---|
  > | 1 | 1% | ~50 TPS | 10 min |
  > | 2 | 5% | ~250 TPS | 10 min |
  > | 3 | 20% | ~1.000 TPS | 15 min |
  > | 4 | 50% | ~2.500 TPS | 15 min |
  > | 5 | 100% | 5.000 TPS | `v1` mantida em standby no período de observação |
  >
  > Com 1% já temos ~50 requisições por segundo, ou cerca de 30 mil em 10 minutos: amostra suficiente para comparar taxas de erro e latência com a `v1`. Os percentuais e tempos são ilustrativos.
  >
  > **Sinais de avanço ou reversão automática** (comparando a `v2` com a `v1` rodando ao mesmo tempo, não com um valor fixo, para que um pico de tráfego não gere falso alarme):
  > - **Erros:** taxa de 5xx da `v2` acima da `v1` por margem definida (por exemplo, +0,5 ponto percentual) em janelas consecutivas.
  > - **Latência:** p99 do checkout acima de 1,2x o da `v1`.
  > - **Saturação:** conexões pendentes no HikariCP, CPU, GC e lag dos consumidores Kafka.
  > - **Sinais de negócio da Saga** (os que o incidente teria denunciado): taxa de pagamentos aprovados, pedidos parados em estados intermediários além do prazo, taxa de estornos, mensagens na DLQ.
  > - **Decisão:** se todas as análises passam, o rollout avança. Se alguma falha em duas janelas seguidas, o rollout é abortado e o tráfego volta a 100% para a `v1`, sem aprovação humana. A decisão de **avançar** para 100% pode exigir confirmação em dias de pico.
  >
  > **Duas versões atendendo ao mesmo tempo:**
  > - **API compatível nos dois sentidos:** só mudanças aditivas (campos novos opcionais); nada removido ou renomeado enquanto houver `v1` no ar.
  > - **Mensagens e eventos:** schemas versionados com compatibilidade verificada no schema registry, e consumidores que ignoram campos desconhecidos. Um evento publicado pela `v2` precisa ser lido pela `v1`, e vice-versa.
  > - **Banco:** o padrão expandir/contrair da seção 2.2.
  > - **Consistência para o cliente:** o roteamento é por hash do cliente (header ou cookie), para que o mesmo usuário não alterne entre versões no meio de uma compra.
  > - **Pods saudáveis antes do tráfego:** probes de *readiness*, desligamento gracioso (`preStop` + `server.shutdown=graceful`) e `PodDisruptionBudget`, para que subir e descer réplicas não derrube requisições em andamento.
  > - **Cuidado com conexões:** réplicas a mais significam mais conexões no PostgreSQL. O limite do pool (`maxPoolSize` x réplicas) é conferido antes do rollout, ou usamos um pooler (PgBouncer).

### 2.2 Evolução de Schema Sem Downtime e Plano de Rollback

- **Passo a Passo da Migração:** Descreva em etapas sequenciais como evoluir a coluna `status_pagamento` para `estado_saga` no PostgreSQL garantindo que as instâncias da versão antiga (`v1`) e da versão nova (`v2`) operem juntas sem erros de SQL.
  > **Regra central: nunca renomear nem apagar uma coluna enquanto existir uma versão que a usa.** O `RENAME` do incidente é uma mudança destrutiva num passo só. Em vez dele, aplicamos **expandir → migrar → contrair**, em releases separadas:
  >
  > **Etapa 0: estado atual.** A `v1` lê e grava `status_pagamento`.
  >
  > **Etapa 1: expandir (migração compatível com a `v1`).** Adicionamos a coluna nova ao lado da antiga. A `v1` nem percebe.
  > ```sql
  > SET lock_timeout = '3s';                                -- falha rápido em vez de travar o checkout
  > ALTER TABLE pedido ADD COLUMN estado_saga text;         -- nula, sem rewrite da tabela
  >
  > -- mantém as duas colunas sincronizadas, venha a escrita da v1 ou da v2
  > CREATE FUNCTION sincroniza_estado_saga() RETURNS trigger AS $$
  > BEGIN
  >   IF NEW.estado_saga IS NULL THEN NEW.estado_saga := NEW.status_pagamento;
  >   ELSIF NEW.status_pagamento IS NULL OR NEW.status_pagamento IS DISTINCT FROM OLD.status_pagamento THEN
  >     NEW.status_pagamento := COALESCE(NEW.status_pagamento, NEW.estado_saga);
  >   END IF;
  >   RETURN NEW;
  > END $$ LANGUAGE plpgsql;
  >
  > CREATE TRIGGER trg_sincroniza_estado_saga
  >   BEFORE INSERT OR UPDATE ON pedido
  >   FOR EACH ROW EXECUTE FUNCTION sincroniza_estado_saga();
  > ```
  > A lógica exata da sincronização depende do mapeamento entre os valores antigos e novos. O ponto é que ela fica no banco e vale para qualquer versão que escreva.
  >
  > **Etapa 2: preencher o histórico, em lotes e sem travar.**
  > ```sql
  > UPDATE pedido SET estado_saga = status_pagamento
  >  WHERE id BETWEEN :ini AND :fim AND estado_saga IS NULL;   -- lotes de ~10 mil linhas, com pausa entre eles
  > ```
  > Roda como job separado, fora do horário de pico, e é idempotente. Depois, `ALTER TABLE ... ADD CONSTRAINT ... CHECK (estado_saga IS NOT NULL) NOT VALID` seguido de `VALIDATE CONSTRAINT`, que valida sem bloquear escritas por muito tempo. Índices novos usam `CREATE INDEX CONCURRENTLY`.
  >
  > **Etapa 3: publicar a `v2` em canary.** A `v2` **lê `estado_saga`** e **grava as duas colunas**. Como a coluna antiga continua existindo e atualizada, a `v1` e a `v2` convivem sem nenhum erro de SQL. Durante o canary, a `v2` só grava em `status_pagamento` valores que a `v1` conhece; estados novos ficam atrás de feature flag.
  >
  > **Etapa 4: estabilizar.** Com a `v2` em 100% e um período de observação sem necessidade de voltar (por exemplo, uma ou duas semanas), liberamos a `v3`, que para de usar `status_pagamento`.
  >
  > **Etapa 5: contrair (release posterior).** Só quando já não for possível nem desejável voltar para a `v1`: remove-se o trigger e depois a coluna antiga (`DROP COLUMN`, com `lock_timeout`). Essa é a única etapa destrutiva, isolada num release próprio e com backup lógico antes.
  >
  > **Como o Flyway entra:**
  > - **A migração não roda no boot da aplicação** (`spring.flyway.enabled=false`). Ela roda como um **Job de pré-deploy**, uma única vez, antes do rollout. No incidente, a primeira réplica a subir alterou o banco enquanto 80% das outras ainda eram da versão antiga.
  > - Migrações com `CREATE INDEX CONCURRENTLY` usam `executeInTransaction=false`.
  > - Migrações já aplicadas nunca são editadas (o Flyway valida o checksum); correções entram como migrações novas.

- **Garantia de Rollback Imediato:** Caso a versão `v2` apresente degradação grave de performance com 20% do tráfego na Black Friday, qual é o procedimento exato de rollback e por que o seu desenho de banco de dados garante que a versão `v1` voltará a assumir 100% da carga sem quebrar nem perder dados gravados pela `v2`?
  > **Procedimento:**
  > 1. **Automático:** a análise do canary detecta a degradação (por exemplo, p99 acima do limite em duas janelas) e aborta o rollout. O controlador devolve 100% do tráfego à `v1`, que continua rodando durante todo o canary. A reversão acontece em segundos, sem rebuild e sem puxar imagem.
  > 2. **Manual, se necessário:** `kubectl argo rollouts abort checkout` (ou o equivalente da ferramenta), que tem o mesmo efeito.
  > 3. **Registro no Git:** um commit de *revert* na mudança do digest, para que o repositório de manifestos volte a refletir o que está em produção. O Argo CD não reaplica a `v2`.
  > 4. **Banco de dados: nenhuma ação.** Não existe migração para desfazer.
  > 5. **Depois:** o time faz a análise com os dados do canary (traces e métricas da `v2` nos 20%) e corrige com uma nova versão, nunca com um rollback do banco.
  >
  > **Por que a `v1` volta sem quebrar nem perder dados:**
  > - **O schema só foi expandido, nada foi removido ou renomeado.** `status_pagamento` continua lá, com o mesmo nome e o mesmo tipo, então todo SQL da `v1` continua válido.
  > - **Nada escrito pela `v2` fica só no formato novo:** a `v2` grava as duas colunas (e o trigger garante a sincronização), então os pedidos que ela criou ou atualizou aparecem corretamente para a `v1` em `status_pagamento`.
  > - **Estados novos ficaram atrás de feature flag**, então a `v1` nunca encontra um valor que não conhece.
  > - **A imagem da `v1` existe e é a mesma**, identificada por digest, imutável e assinada, e já está em execução.
  > - **Sem migração de volta (`down`):** reverter dados em produção é a operação mais perigosa. O desenho evita precisar dela: o caminho de volta é trocar a versão da aplicação, não desfazer o banco.

---

## 3. Guardrails de Segurança, Governança e Validação da Release

### 3.1 Gates de Segurança e Guardrails no Processo de Entrega

- **Guardrails do Commit ao Deploy:** Quais portões automatizados de qualidade e segurança devem proteger a sua esteira ao longo do fluxo de desenvolvimento (desde o Pull Request até a publicação e promoção da imagem)?
  > Cada portão é **bloqueante**: se falhar, a etapa seguinte não acontece.
  >
  > | Etapa | Portões automatizados |
  > |---|---|
  > | **Pull Request** | Branch protegida; revisão obrigatória (CODEOWNERS); testes unitários e de integração; cobertura mínima; SAST (CodeQL/Sonar); varredura de segredos (gitleaks); análise de dependências (SCA) e de licenças; lint do Dockerfile (hadolint); **lint de migrações** (`flyway validate` e uma regra, como o squawk, que reprova `RENAME`, `DROP`, `NOT NULL` sem valor padrão e índices sem `CONCURRENTLY` fora da fase de contração); testes de contrato |
  > | **Build** | Imagem construída uma única vez; varredura de vulnerabilidades (Trivy/Grype) reprovando CVEs críticas e altas com correção disponível; geração do SBOM; assinatura (cosign) e atestação de procedência; publicação por digest em tag imutável |
  > | **Homologação** | Deploy do **mesmo digest**; smoke e testes ponta a ponta; testes de coexistência da migração (3.2); teste de carga de referência |
  > | **Produção** | *Admission controller* (Kyverno/OPA Gatekeeper) exige imagem assinada, referenciada por digest (nunca `latest`), executando como não-root, com limites de CPU e memória; aprovação para promoção; canary com análise automática (seção 2.1) |
- **Governança e Controle em Produção:** Quais *guardrails* de processo e arquitetura impedem que alterações manuais não rastreadas cheguem à produção e como equilibrar segurança rigorosa com agilidade de resposta durante incidentes críticos sem perder a trilha de auditoria?
  > **Impedindo mudanças manuais não rastreadas:**
  > - **GitOps como único caminho:** o estado desejado de produção está num repositório. Argo CD aplica o que está no Git, com `selfHeal` ligado: qualquer alteração feita à mão no cluster é revertida automaticamente.
  > - **Acesso humano somente leitura** ao cluster e ao banco no dia a dia. Quem quer mudar algo abre um pull request.
  > - **Mudanças de banco só pelo Job de migração do pipeline**, com as migrações versionadas e revisadas. Nada de DDL manual em produção.
  > - **Rastro de auditoria:** o histórico do Git (quem aprovou e quando), os logs de auditoria do Kubernetes e do banco, e o registro de assinaturas e promoções do pipeline.
  >
  > **Agilidade durante incidentes sem perder a auditoria:**
  > - **O caminho mais rápido já é o caminho seguro:** o rollback é um *revert* de um commit (ou um comando de abort do rollout) para um digest conhecido, imutável e já testado. Em geral não é preciso nenhum acesso especial.
  > - **Pista expressa para correções urgentes:** um *hotfix* segue o mesmo pipeline, com a suíte longa de ponta a ponta dispensada, mas **mantendo** testes unitários, varredura de segredos e vulnerabilidades, assinatura e canary. Um incidente não é motivo para publicar uma imagem sem assinatura ou sem varredura.
  > - **Kill switches:** feature flags que desligam um recurso em segundos, sem deploy.
  > - **Acesso emergencial (*break-glass*):** quando for imprescindível agir direto no cluster ou no banco, o acesso é concedido sob demanda, por tempo limitado (por exemplo, 1 hora), com a aprovação do plantonista e de uma segunda pessoa. Toda a sessão é gravada e auditada, e o acesso expira sozinho.
  > - **Reconciliação obrigatória:** tudo o que foi feito à mão precisa virar um pull request em até 24 horas. Se não virar, o `selfHeal` reverte a mudança. Assim o incidente não deixa um desvio silencioso entre o Git e a produção.

### 3.2 Estratégia de Testes da Imagem e da Migração

- **Validação Pré-Tráfego:** Passar nos testes unitários durante o build do código não prova que a imagem final do container sobe nem que a migração de banco é retrocompatível.
- **Plano de Testes na Pipeline:** Descreva como você testará automaticamente a imagem construída e validará a coexistência da migração de banco de dados na esteira antes de liberar o deploy para o tráfego real de clientes.
  > **O princípio:** os testes rodam contra os artefatos que vão para produção: **a imagem construída** (não o jar, nem a IDE) e **um PostgreSQL real** (não H2), com os mesmos scripts de migração.
  >
  > **Testes da imagem:**
  > 1. **Estrutura:** `container-structure-test` confere o usuário não-root, a ausência de shell e de ferramentas de build, a porta exposta e os arquivos presentes.
  > 2. **Boot real:** o Testcontainers sobe a **imagem** com PostgreSQL e Kafka reais, com a mesma configuração que ela terá em produção, e espera `/actuator/health/readiness` ficar `UP`. Isso prova que a imagem inicia, que o profile de produção carrega e que as probes funcionam.
  > 3. **Smoke test:** cria um pedido e percorre o caminho feliz da Saga.
  > 4. **Varredura e política:** Trivy na imagem final e verificação das regras do admission controller (Kyverno CLI) sobre os manifestos.
  >
  > **Testes de coexistência da migração (o teste que teria evitado o incidente):**
  > 1. Sobe um PostgreSQL limpo e aplica as migrações **até a versão anterior**. Sobe a imagem da `v1` e executa seu conjunto de testes.
  > 2. Aplica a **nova migração** (expandir) com a `v1` ainda no ar. Executa de novo os testes da `v1` e uma carga sintética contra ela: se qualquer SQL falhar (`column does not exist`), o pipeline reprova.
  > 3. Sobe a `v2` **ao lado** da `v1`, no mesmo banco, e executa um fluxo misto: pedidos criados pela `v1` lidos pela `v2` e vice-versa, e verifica que as duas colunas estão consistentes.
  > 4. **Simulação de rollback:** derruba a `v2` no meio do fluxo, mantém a `v1`, e confirma que os dados escritos pela `v2` estão legíveis e corretos para a `v1`.
  > 5. **Comportamento do DDL:** executa a migração com `lock_timeout` curto sobre uma base de tamanho próximo ao de produção e verifica que nenhum comando mantém `ACCESS EXCLUSIVE` por mais que o limite.
  > 6. **Regra automática:** o lint de migrações reprova `RENAME`/`DROP` fora de um release marcado como "contração".
  >
  > ```java
  > @Testcontainers
  > class MigracaoCoexistenciaIT {
  >
  >     @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");
  >
  >     @Test
  >     void v1ContinuaFuncionandoDepoisDaMigracaoExpandir() {
  >         migrar(pg, "ate-versao-anterior");
  >         try (var v1 = subirImagem("checkout:v1-digest", pg)) {
  >             assertThat(v1.criarPedido()).isSuccessful();
  >             migrar(pg, "expandir-estado-saga");                  // nova migração, v1 ainda rodando
  >             assertThat(v1.criarPedido()).isSuccessful();         // sem 'column does not exist'
  >             assertThat(v1.consultarPedidos()).noneMatch(Falha::erroDeSql);
  >
  >             try (var v2 = subirImagem("checkout:v2-digest", pg)) {
  >                 var pedidoV2 = v2.criarPedido();
  >                 assertThat(v1.consultar(pedidoV2.id()).status()).isEqualTo(pedidoV2.estado());
  >             }                                                    // v2 cai: simula rollback
  >             assertThat(v1.consultarPedidos()).noneMatch(Falha::erroDeSql);
  >         }
  >     }
  > }
  > ```
  >
  > **Depois do pipeline, antes do tráfego real:**
  > - **Staging com a mesma topologia:** um rollout real (canary) com duas versões e carga de k6 durante a troca, exigindo zero 5xx. Também fazemos um **ensaio periódico de rollback**, para provar que ele funciona antes de ser necessário.
  > - **O canary em produção é o último portão:** a análise automática da seção 2.1 compara a `v2` com a `v1` com tráfego real, começando em 1%.
  >
  > **Critério de aceite da release:** a imagem sobe, a `v1` roda sobre o schema novo sem nenhum erro de SQL, a `v1` e a `v2` convivem, o rollback preserva os dados, e o digest promovido é o mesmo que passou na homologação.
