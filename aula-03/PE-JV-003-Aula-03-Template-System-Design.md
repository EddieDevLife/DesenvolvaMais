# Template de System Design: Aula 03 - Banco de Dados em Larga Escala

> **Contexto do Cenário: Checkout na Black Friday**
>
> Sua equipe está preparando a persistência da API de Checkout para o pico de tráfego da Black Friday:
>
> 1. **Armazenamento:** Transações financeiras de pedidos exigem garantias ACID rigorosas e consistência imediata, enquanto dados de catálogo e sessões de usuários demandam alta vazão de leitura e tolerância a partição.
> 2. **Concorrência:** A atualização de status do pedido tem baixa disputa por registro (apenas o comprador interage), mas a **reserva de estoque** sofre altíssima contenção com centenas de clientes disputando as últimas unidades do mesmo SKU promocional.
> 3. **Incidente em Produção:** Em testes de carga, o tempo de resposta p99 subiu para 4.8s e o pool HikariCP saturou devido a consultas lentas de pedidos (`WHERE customer_id = ?` e `WHERE order_status = 'processing'`).
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Adicione este bloco ao repositório contínuo do seu projeto.

---

## 1. Decisão de Armazenamento e Chave Primária (PK)

### 1.1 Motor de Banco de Dados e Paradigma de Persistência

- **A Escolha para o Checkout:** ( ) Relacional Puro (PostgreSQL) (X) Persistência Poliglota / Híbrida
- **Justificativa Arquitetural:** Avalie o Teorema CAP e o modelo de consistência (ACID vs BASE). Por que a sua escolha atende aos requisitos do Checkout sem comprometer a integridade financeira?
  > R: Para o sistema transacional central o ideal é um banco de dados relacional com forte suporte ACID, pois precisamos isolar estritamente as operações financeiras e de pedidos. E para os dados de alta vazão e baixa complexidade(por exemplo carrinhos e sessões de usuário) podemos fazer uma escolha mais performatica como o Redis(NoSQL)
- **Além do Modelo Documental (Ecossistema NoSQL):** A persistência poliglota vai muito além da dicotomia "Relacional vs MongoDB". Considerando as necessidades complementares de uma plataforma de e-commerce (como carrinhos efêmeros, busca textual de produtos, telemetria/logs, auditoria temporal ou análise de conexões/fraude), cite e comente pelo menos outros dois tipos/famílias de bancos NoSQL que poderiam coexistir na arquitetura, indicando para quais casos de uso específicos cada um seria a ferramenta ideal e por quê.
  > R1:  Banco de Dados Chave-Valor em Memória (In-Memory Key-Value Store)
Ferramenta de Referência: Redis / Dragonfly.

Caso de Uso Específico na Plataforma:

Carrinhos de Compras Efêmeros: Armazenamento rápido do estado do carrinho de usuários logados ou anônimos antes do fechamento do pedido.

Gerenciamento de Sessões e Rate Limiting: Controle de tokens de autenticação e proteção contra ataques de força bruta ou robôs de raspagem que esgotam o estoque artificialmente (scalpers).

Camada de Pré-reserva Atômica (Black Friday): Decremento rápido de estoque via scripts em memória antes da consolidação no banco relacional.

Por que esta escolha:

Performance Extrema: Operações em memória garantem latências na ordem de microssegundos (vazão de centenas de milhares de requisições por segundo por nó).

Recursos Nativos Avançados: Suporte a estruturas de dados complexas em memória (como Hashes, Sets e Sorted Sets) e expiração baseada em tempo (TTL), permitindo que carrinhos abandonados expirem automaticamente sem ocupar espaço permanente no banco relacional.

R2: Banco de Dados Colunar Distribuído (Wide-Column Store)
Ferramenta de Referência: Apache Cassandra / ScyllaDB.

Caso de Uso Específico na Plataforma:

Telemetria, Clickstream e Auditoria de Eventos: Registro em massa de cliques, visualizações de página, rastreamento de entrega de pacotes e logs de auditoria temporal de transações.

Por que esta escolha:

Escalabilidade de Escrita Linear: Projetado para receber um volume massivo de dados de escrita contínua (write-heavy) sem gargalos de I/O de disco.

Arquitetura Masterless (Tolerância a Partição - CAP Theorem): Se um nó do cluster falha, outro assume imediatamente sem downtime. Como os logs e eventos de telemetria aceitam consistência eventual, essa família NoSQL garante que a aplicação nunca pare de registrar dados por indisponibilidade do banco.

### 1.2 Estratégia da Chave Primária

- **A Escolha:** ( ) Sequence / Identity ( ) UUIDv4 (X) UUIDv7 / TSID
- **Impacto no Armazenamento:** Por que essa estratégia foi escolhida considerando inserções concorrentes e o impacto de fragmentação na árvore B-Tree (page splits)?
  > R: Impacto no Armazenamento:

Prevenção de Page Splits e Fragmentação na B-Tree: O UUIDv7 (ou TSID) incorpora um componente temporal (timestamp) de alta precisão nos bits mais significativos, seguido por bits de aleatoriedade. Isso torna a chave quase sequencial e ordenada no tempo. Diferente do UUIDv4 (que é puramente randômico e força inserções aleatórias em qualquer página da árvore B-Tree, gerando page splits custosos e degradação severa de I/O em alta concorrência), o UUIDv7 garante que novas inserções de pedidos ocorram predominantemente no final (append-only), mantendo a alta densidade de páginas nas folhas da B-Tree e eficiência máxima no cache do banco de dados.

Segurança e Distribuição: Além de otimizar o armazenamento e evitar o gargalo de concorrência global de uma Sequence tradicional (que expõe o volume exato de vendas e pode travar em arquiteturas distribuídas/sharded), o UUIDv7 preserva a unicidade global descentralizada sem sacrificar a performance de escrita esperada para o pico da Black Friday.

---

## 2. A Dualidade da Concorrência: Locks e Mitigações

Analise e defina a abordagem de concorrência para os dois fluxos críticos do negócio:

### 2.1 Mudança de Status do Pagamento (Baixa contenção por registro)

- **Abordagem de Lock:** (X) Lock Otimista (`@Version`) ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Por que essa abordagem é a mais adequada para este cenário específico?
  > R: Como a mudança de status do pedido possui baixa contenção por registro (apenas o adquirente/gateway de pagamento e o próprio comprador interagem com o ciclo daquele pedido específico de forma isolada), o Lock Otimista é a escolha ideal.

Performance e Escalabilidade: Ele não bloqueia a linha de dados no banco durante o processamento. O sistema assume que conflitos são raros; a versão da entidade é verificada no momento do commit (UPDATE ... WHERE id = ? AND version = current_version).

Evita o esgotamento do HikariCP: Como transações de pagamento aguardam retornos de APIs externas (gateway), usar locks pessimistas aqui seguraria a conexão do banco aberta e travada durante toda a chamada de rede externa, esgotando rapidamente o pool.

### 2.2 Controle de Estoque Concorrente (Alta contenção no mesmo registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou (X) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Centenas de usuários disputando os últimos itens na Black Friday. Como você garante que o saldo nunca fique negativo sem derrubar o banco?
  > R: Para produtos altamente disputados na Black Friday, o Lock Otimista puro falharia miseravelmente, gerando uma enxurrada de exceções de concorrência (OptimisticLockException) e frustrando o usuário.

Abordagem Segura e Escalável: Embora o Lock Pessimista (SELECT ... FOR UPDATE) garanta que o saldo nunca fique negativo em um banco relacional, executá-lo diretamente na tabela principal de estoque para centenas de usuários simultâneos criaria um gargalo intransponível (fila de espera em banco de dados).

Solução Arquitetural de Contenção (Padrão Híbrido): O decremento atômico deve ser descentralizado para uma camada em memória de alta performance (Redis) utilizando Scripts Lua (ou operações atômicas como DECRBY com validação de limite >= 0), onde centenas de threads conseguem validar e decrementar o estoque em microssegundos sem page locks no banco relacional. O banco relacional ACID só é acionado no fechamento definitivo do pedido para persistir a fatia já garantida, blindando o sistema contra o saldo negativo sem derrubar o banco.

### 2.3 Riscos e Mitigações

- **Análise Crítica:** Quais os riscos operacionais das abordagens de lock que você escolheu (ex: excesso de `OptimisticLockException` frustrando o cliente vs `ConnectionTimeoutException` e *deadlocks* enfileirando threads no Lock Pessimista)? Como sua arquitetura mitiga esses riscos?
  > R: Risco Operacional:

Indisponibilidade ou Split-Brain no Redis: Se o cluster do Redis falhar durante o pico da Black Friday, a camada de pré-reserva rápida cai, podendo direcionar todo o tráfego concorrente diretamente para o banco relacional, causando ConnectionTimeoutException, saturação do HikariCP e deadlocks.

Inconsistência entre Redis e o Banco Relacional: Se a pré-reserva ocorrer no Redis mas a transação falhar no banco SQL ao persistir o pedido final, o estoque em memória pode ficar dessincronizado (fantasmas de estoque).

Estratégia de Mitigação:

Alta Disponibilidade e Persistência no Redis: Utilizar um cluster Redis configurado com replicação Master-Slave e persistência ativa (AOF - Append Only File a cada segundo ou persistência híbrida) para garantir que, em caso de failover, o estado do estoque seja recuperado instantaneamente sem perda de dados.

Sincronização por Transações Compensatórias / TTL de Reserva: A pré-reserva no Redis deve ter um TTL estrito (ex: 10 minutos para concluir o checkout). Se o pagamento não for confirmado nesse intervalo, um script de limpeza devolve o item ao estoque automaticamente.

Circuit Breaker e Graceful Degradation: Proteger a base relacional com padrões de Circuit Breaker (ex: Resilience4j) na camada de acesso ao banco. Caso o banco apresente gargalos, o sistema deve rejeitar o checkout de forma controlada com uma página amigável de fila de espera em vez de deixar o pool de conexões estourar e derrubar a aplicação inteira.

---

## 3. Troubleshooting do Incidente e Gestão de Índices

### 3.1 Investigação de Incidente em Produção

- **Cenário:** Em testes de carga para a Black Friday, o tempo de resposta p99 do Checkout subiu repentinamente para 4.8s e o pool de conexões (HikariCP) saturou. Os logs apontam que a consulta de pedidos mais frequente (`WHERE customer_id = ? AND order_status = 'processing'`) está demorando segundos para responder.
- **Plano de Diagnóstico:** Qual é o seu processo técnico passo a passo para investigar a causa raiz dessa lentidão diretamente no banco de dados? Quais comandos, métricas ou planos você analisaria para comprovar a hipótese antes de propor qualquer mudança?
  > R: Passo 1: Captura e Análise do Plano de Execução (Execution Plan)
Antes de alterar qualquer estrutura, precisamos entender como o banco de dados está processando a consulta.

Comando (PostgreSQL/Oracle): Executar um EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) na query problemática preenchendo os parâmetros:

SQL
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders 
WHERE customer_id = '12345' AND order_status = 'processing';
O que analisar:

Verificar se o planejador está realizando um Seq Scan (Varredura Sequencial) na tabela orders. Em tabelas de e-commerce com milhões de pedidos durante a Black Friday, ler bloco a bloco do disco em busca de registros causa I/O massivo e explica os 4.8s de latência.

Avaliar o custo estimado vs. o custo real (Actual Rows vs. Estimated Rows) e a métrica de Buffers: shared hit= vs read=. Um alto volume de read indica que os dados não estão em cache e exigem leitura física em disco.

Passo 2: Inspeção de Métricas e Locks Ativos no SGBD
Para entender por que o pool do HikariCP saturou (esgotou as conexões livres), precisamos verificar se as conexões estão presas aguardando I/O de disco ou travadas em filas de locks.

Métricas e Consultas de Diagnóstico:

Conexões Ativas e Estados: Consultar a tabela de processos do banco (ex: pg_stat_activity no PostgreSQL) para identificar queries em execução prolongada (state = 'active') e a quanto tempo estão rodando.

SQL
pid | usename | datname | state  | query_start | query
Análise de Locks e Deadlocks: Verificar se há contenção ou bloqueios de tabela/linha concorrentes que façam com que múltiplos threads da aplicação fiquem bloqueados esperando uma linha ser liberada, retendo a conexão do HikariCP até estourar o connectionTimeout.

Passo 3: Auditoria do Catálogo de Índices Existentes
Comando: Inspecionar os índices atuais da tabela orders para confirmar se existe um índice cobrindo as colunas da cláusula WHERE.

SQL
-- No PostgreSQL
SELECT indexname, indexdef 
FROM pg_indexes 
WHERE tablename = 'orders';
Validação da Hipótese: Se o resultado mostrar apenas a chave primária (id) ou índices isolados (ex: apenas em customer_id), a hipótese de falta de indexação composta está 100% comprovada. O banco é forçado a filtrar por um campo e varrer o restante linearmente.

Passo 4: Validação em Ambiente de Homologação / Staging
Simulação: Reproduzir a carga de teste (com o mesmo volume de dados da Black Friday) aplicando o índice composto sugerido anteriormente:

SQL
CREATE INDEX CONCURRENTLY idx_orders_customer_status 
ON orders(customer_id, order_status);
Comparação de Métricas: Reexecutar o EXPLAIN ANALYZE para garantir que o plano de execução mudou de um custoso Seq Scan para um eficiente Bitmap Index Scan ou Index Scan, derrubando o tempo de resposta de segundos para poucos milissegundos e liberando o pool do HikariCP instantaneamente.

### 3.2 Decisão de Engenharia: A Abordagem de Índices

- **Dilema no Pull Request:** Para solucionar a lentidão, um desenvolvedor da equipe abriu um PR criando índices individuais para todas as colunas da tabela de pedidos que aparecem em filtros de consulta.
- **Seu Posicionamento:** Como responsável técnico, você aprovaria esse PR? Qual é a sua avaliação crítica sobre essa estratégia e quais impactos colaterais sistêmicos devem ser ponderados antes de aprovar novos índices?
  > R: Eu NÃO aprovaria esse PR em sua forma atual.

Embora a intenção do desenvolvedor seja louvável (resolver a lentidão das consultas de pedidos em produção), a estratégia de criar índices individuais (single-column indexes) para cada coluna de filtro é um anti-padrão grave de banco de dados que trará consequências catastróficas para a performance geral do sistema, especialmente sob o estresse de uma Black Friday.

Abaixo está a avaliação crítica detalhada e os impactos colaterais sistêmicos que justificam o veto, servindo como registro formal para o nosso manual de arquitetura e repositório contínuo.

Avaliação Crítica e Impactos Colaterais Sistêmicos
1. Degradação Severa na Performance de Escrita (Write-Heavy Penalty)
O Problema: Cada índice adicional adiciona uma árvore B-Tree separada que o banco de dados precisa manter atualizada. A cada operação de INSERT, UPDATE ou DELETE na tabela de pedidos (orders), o SGBD precisa reescrever e rebalancear não apenas a tabela principal, mas cada um dos índices individuais.

Impacto na Black Friday: Em um cenário de checkout com milhares de requisições por segundo, o custo de E/S de disco para sincronizar múltiplos índices satura o subsistema de armazenamento, transformando o banco em um gargalo intransponível.

2. Explosão de Consumo de Memória e Buffer Pool Eviction
O Problema: Os índices precisam residir prioritariamente na memória RAM (shared_buffers / Buffer Pool) para serem performáticos. Índices excessivos e desnecessários ocupam espaço precioso em memória.

Impacto na Black Friday: Com indexes inflados, páginas de dados reais e quentes são expulsas da memória (cache misses), forçando o banco a buscar dados fisicamente no disco mecânico ou SSD a cada operação, elevando a latência (p99) de volta aos patamares inaceitáveis.

3. Ineficácia para Consultas Compostas (O Erro Lógico)
O Problema: A query problemática identificada no incidente utiliza múltiplos filtros simultâneos (WHERE customer_id = ? AND order_status = 'processing').

Impacto Técnico: Bancos relacionais como o PostgreSQL raramente conseguem combinar eficientemente dois índices single-column separados para atender a uma única cláusula WHERE composta (recorrendo a operações custosas de Bitmap Index Scan ou ignorando um deles). Portanto, os índices individuais criados pelo desenvolvedor sequer resolveriam a lentidão da consulta original.
