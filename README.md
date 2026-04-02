# CI&T Case - Fluxo de Caixa
####  Controle de fluxo de caixa diário com os lançamentos(débitos e créditos), também precisa de um relatório que disponibilize o saldo diário consolidado.

## Índice

## Sumário
[1. Visão Geral do Event Storming](#visao)

[2. Atores / Usuários](#atores)

[3. Comandos](#comandos)

[4. Eventos de Domínio](#dominios)

[5. Agregados](#agregados)

[6. Políticas / Regras de Negócio](#negocio)

[7. Leituras / Consultas (CQRS)](#cqrs)

[8. Sistemas Externos](#externo)

[9. Fluxo Completo do Sistema](#fluxo)

[10. Glossário do Event Storming](#glossario)


### Diagrama Domain Driven Design - Fluxo de Caixa
![Texto](/ddd-case.jpg)


### Diagrama do Event Storming - Fluxo de Caixa
![Texto](/event-storming.jpg)

<a id="visao"></a>
### 1. Visão Geral
O Event Storming é uma técnica de modelagem colaborativa que visa mapear todo o fluxo de negócio do sistema de Controle de Fluxo de Caixa, identificando comandos, eventos, agregados, atores, políticas, leituras e sistemas externos envolvidos no processo de registro de lançamentos e consolidação diária de saldos.

| Categoria  | Descrição |
| --- | ---  |
|Comandos |	Ações iniciadas pelo usuário ou sistema|
|Eventos de Domínio |	Ocorrências importantes no negócio |
|Agregados|	Entidades que garantem consistência transacional|
|Atores / Usuários|	Quem interage com o sistema|
|Políticas / Regras de Negócio|	Regras e validações do domínio|
|Leituras (CQRS)|	Consultas otimizadas separadas dos comandos|
|Sistemas Externos|	Infraestrutura e serviços de terceiros|


<a id="atores"></a>
### 2. Atores / Usuários
| Atores  | Descrição | Responsabilidades |
| --- | --- |---|
| Comerciante | Usuário final do sitema| Registrar lançamentos, consultar saldos, cancelar lançamentos, exportar extratos |
|Administrador|	Gestor do sistema|	Gerar relatórios gerenciais, configurar limites de alerta, auditar operações|
|Sistema (Automático)|	Processos automatizados|	Disparar alertas de saldo baixo, atualizar caches, consolidar saldos|

<a id="commandos"></a>
###  3. Comandos
Comandos representam ações ou intenções iniciadas por um ator.

|Comando|	Descrição|	Origem|
| --- | --- |---|
|Registrar Lançamento |	Criar um novo lançamento (débito ou crédito) |Comerciante|
|Consultar Saldo Diário|Buscar o saldo consolidado de uma data específica|Comerciante|
|Cancelar Lançamento|	Estornar um lançamento previamente registrado	|Comerciante|
|Gerar Relatório Período|	Produzir relatório financeiro de um intervalo	|Administrador|
|Exportar Extrato|	Gerar arquivo (PDF/Excel) com movimentações	|Comerciante|

### Detalhamento dos Comandos
##### Registrar Lançamento
- Entrada: Valor, tipo (débito/crédito), descrição, categoria, estabelecimento
- Pré-condições: Usuário autenticado, valor > 0
- Pós-condições: Lançamento persistido, evento publicado
#### Consultar Saldo Diário
- Entrada: Data (YYYY-MM-DD)
- Pré-condições: Usuário autenticado
- Pós-condições: Retorno do saldo (via cache ou banco)
### Cancelar Lançamento
- Entrada: ID do lançamento
- Pré-condições: Lançamento existe e não está cancelado
- Pós-condições: Status alterado para CANCELADO, saldo recalculado
#### Gerar Relatório Período
- Entrada: Data inicial, data final, formato (PDF/Excel)
- Pré-condições: Usuário com permissão de administrador
- Pós-condições: Arquivo gerado e disponibilizado para download
#### Exportar Extrato
- Entrada: Período, formato
- Pré-condições: Usuário autenticado
- Pós-condições: Arquivo exportado

<a id="dominios"></a>
### 4. Eventos de Domínio
Eventos representam fatos que ocorreram no sistema e são relevantes para o negócio.

|Evento|	Descrição|	Disparado por|
| --- | --- |---|
|LancamentoRegistrado|	Um novo lançamento foi criado|	Comando Registrar Lançamento|
|SaldoDiarioAtualizado|	O saldo de uma data foi recalculado|	Política de Cálculo de Saldo|
|LancamentoCancelado|	Um lançamento foi cancelado|	Comando Cancelar Lançamento|
|SaldoBaixoAlertado|	O saldo diário atingiu um limite crítico|	Política de Verificação de Limite|
|RelatorioPeriodoGerado|	Um relatório foi gerado com sucesso|	Comando Gerar Relatório|

### Detalhamento dos Eventos
###### LancamentoRegistrado

```
Json
{
  "id": "uuid",
  "valor": 150.00,
  "tipo": "CREDITO",
  "dataHora": "2025-03-31T10:30:00Z",
  "descricao": "Venda de produtos",
  "categoria": "VENDAS",
  "usuarioId": "uuid",
  "estabelecimento": "Loja Matriz",
  "occurredAt": "2025-03-31T10:30:00Z"
}
```

##### SaldoDiarioAtualizado
```
Json
{
  "data": "2025-03-31",
  "totalCreditos": 5000.00,
  "totalDebitos": 3200.00,
  "saldo": 1800.00,
  "quantidadeTransacoes": 15,
  "atualizadoEm": "2025-03-31T10:30:05Z"
}
```

##### LancamentoCancelado
```
Json
{
  "id": "uuid",
  "usuarioId": "uuid",
  "valor": 150.00,
  "dataHora": "2025-03-31T10:30:00Z",
  "canceladoEm": "2025-03-31T11:00:00Z"
}
```


##### SaldoBaixoAlertado

```
Json
{
  "usuarioId": "uuid",
  "data": "2025-03-31",
  "saldoAtual": -250.00,
  "limiteAlerta": 0,
  "emailUsuario": "comerciante@exemplo.com"
}
```

##### RelatorioPeriodoGerado
```
Json
{
  "periodoInicio": "2025-03-01",
  "periodoFim": "2025-03-31",
  "formato": "PDF",
  "usuarioId": "uuid",
  "geradoEm": "2025-03-31T23:59:59Z",
  "urlDownload": "/relatorios/2025-03-relatorio.pdf"
}
```

<a id="agregados"></a>
### 5. Agregados
Agregados são raízes de consistência transacional que garantem a 

integridade dos dados.

|Agregado	|Raiz|	Atributos Principais|	Comportamentos|
| --- | --- |---|---|
|Lancamento|	LancamentoId|	valor, tipo, dataHora, descrição, categoria, status|	registrar(), cancelar(), confirmar()|
|SaldoDiario|	Data|	totalCreditos, totalDebitos, saldo, quantidadeTransacoes|	atualizar(), recalcular(), verificarLimite()|
|SaldoMensal|	AnoMes|	saldoInicial, totalEntradas, totalSaidas, saldoFinal|	consolidarMensal(), projetar()|
|RelatorioPeriodo|	Periodo|	lista de lançamentos, resumoFinanceiro|	gerar(), exportar()

### Detalhamento dos Agregados
##### Lancamento
- Regras de consistência:
  - Valor deve ser maior que zero
  - Tipo deve ser DEBITO ou CREDITO
  - DataHora não pode ser futura
  - Status só pode ser alterado de CONFIRMADO para CANCELADO
- Eventos gerados:
  - LancamentoRegistrado (ao criar)
  - LancamentoCancelado (ao cancelar)
- SaldoDiario
  - Regras de consistência:
  - Data é única (uma entrada por dia)
  - Saldo = totalCreditos - totalDebitos
  - QuantidadeTransacoes é incrementada a cada lançamento
- Eventos gerados:
   - SaldoDiarioAtualizado (a cada atualização)
   - SaldoBaixoAlertado (quando saldo < limite)
##### SaldoMensal
- Regras de consistência:
    - Consolidado a partir dos SaldoDiario do mês
    - SaldoFinal do mês = SaldoInicial do próximo mês
##### RelatorioPeriodo
- Regras de consistência:
  - Período válido (início <= fim)
  - Relatório pode ser cacheado por 1 hora


<a id="negocios"></a>
### 6. Políticas / Regras de Negócio
Políticas definem comportamentos automáticos em resposta a eventos.

|Política|	Gatilho|	Ação|	Descrição|
|--|--|---|---|
|Validar |Lançamento|	LancamentoRegistrado|	Validar dados	Verifica se valor > 0, tipo válido, etc.|
|Calcular Saldo Diário|	LancamentoRegistrado|	Atualizar Saldo Diario|	Soma créditos e débitos do dia|
Verificar Limite de Alerta|	SaldoDiarioAtualizado|	Disparar alerta|	Se saldo < 0, notificar comerciante|
|Atualizar Cache (Redis)|	SaldoDiarioAtualizado|	Atualizar Redis|	Manter cache quente para consultas|
Registrar Auditoria|	Todos os eventos|	Salvar no MongoDB|	Log de todas as operações|

### Regras de Negócio Detalhadas
##### 6.1 Validação de Lançamento


```

SE valor <= 0 ENTÃO
    REJEITAR com erro "Valor deve ser maior que zero"
    
SE tipo NÃO ESTIVER EM ['DEBITO', 'CREDITO'] ENTÃO
    REJEITAR com erro "Tipo inválido"
    
SE dataHora > DataHoraAtual ENTÃO
    REJEITAR com erro "Data não pode ser futura"
    
SE descricao VAZIA ENTÃO
    REJEITAR com erro "Descrição obrigatória"


```    


##### 6.2 Cálculo de Saldo Diário

```
RECUPERAR SaldoDiario da data
SE NÃO EXISTE ENTÃO
    CRIAR SaldoDiario com saldo = 0
    
SE tipo = 'CREDITO' ENTÃO
    saldo = saldo + valor
    totalCreditos = totalCreditos + valor
SENÃO
    saldo = saldo - valor
    totalDebitos = totalDebitos + valor
    
quantidadeTransacoes = quantidadeTransacoes + 1
ultimaAtualizacao = DataHoraAtual

```

##### 6.3 Alerta de Saldo Baixo

```
RECUPERAR limiteAlerta do usuário (padrão = 0)
SE saldo < limiteAlerta ENTÃO
    DISPARAR evento SaldoBaixoAlertado
    ENVIAR e-mail/Slack para comerciante

```    

##### 6.4 Atualização de Cache (Redis)

```
CHAVE = "saldo:{data:yyyy-MM-dd}"
TTL = 24 horas
SE cache EXISTE ENTÃO
    ATUALIZAR valor do cache
SENÃO
    CRIAR cache a partir do Read Database

```

##### 6.5 Auditoria

```
PARA CADA evento recebido:
    CRIAR registro com:
        - id = UUID
        - tipo = nome do evento
        - dados = JSON do evento
        - usuarioId = extraído do evento
        - ocorridoEm = timestamp
        - origem = tópico/fonte
    SALVAR no MongoDB

```    


<a id="cqrs"></a>
### 7. Leituras / Consultas (CQRS)
No padrão CQRS, as operações de leitura são separadas das operações de escrita para otimização de performance.


|Consulta|	Descrição|	Fonte de Dados|	Cache|
|--|--|--|--|
|Consultar Saldo Diário|	Obter saldo de uma data específica|	Redis (cache) ou PostgreSQL|	✅ 24h|
Obter Extrato Período|	Listar todos lançamentos de um intervalo|	PostgreSQL|	❌|
|Relatório Gerencial|	Relatório consolidado com análises|	PostgreSQL (views materializadas)|	✅ 1h|

### Detalhamento das Leituras
##### Consultar Saldo Diário
- Endpoint: GET /consolidado/diario?data=2025-03-31
- Fluxo:

  - Verificar Redis (chave: saldo:2025-03-31)
  - Se cache hit → retornar (tempo < 10ms)
  - Se cache miss → consultar PostgreSQL (tabela saldo_diario)
  - Atualizar cache em background
  - Retornar resultado
##### Obter Extrato Período
- Endpoint: GET /extrato?inicio=2025-03-01&fim=2025-03-31
- Fluxo:
  - Consultar PostgreSQL (tabela lancamentos)
  - Aplicar filtros por data e usuário
  - Ordenar por dataHora decrescente
  - Retornar lista paginada
##### Relatório Gerencial
- Endpoint: GET /relatorios/gerencial?ano=2025&mes=3
- Fluxo:
  - Verificar cache (chave: relatorio:2025-03)
  - Se cache miss → consultar views materializadas
  - Calcular totais, médias, projeções
  - Atualizar cache
  - Retornar relatório


<a id="externos"></a>
### 8. Sistemas Externos
Sistemas externos são dependências de infraestrutura ou serviços de terceiros.

|Sistema|	Tipo|	Finalidade|	Padrão de Resiliência|
|--|--|--|--|
|Cloud Identity|	Gerenciamento de usuários, Access Tokens (para chamar APIs) e ID Tokens| Autenticação| 	Circuit Breaker (3 falhas / 60s)
|Active Directory (AD)|	Autenticação|	Login corporativo LDAP/Kerberos|	Circuit Breaker (3 falhas / 45s)|
|Apache Kafka|	Mensageria|	Publicação/consumo de eventos|	Circuit Breaker (3 falhas / 30s)|
|Redis Cache|	Cache|	Armazenamento temporário de saldos|	Circuit Breaker (3 falhas / 30s)|
|SMTP / SendGrid|	E-mail|	Envio de alertas e notificações|	Circuit Breaker (3 falhas / 60s)|

### Integração com Sistemas Externos
##### Cloud Identity

```
┌─────────────────────────────────────────────────────────────┐
│                     Fluxo de Autenticação                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Comerciante → auth-api → Cloud Identity User Pool          │
│                  │                                          │
│                  ├── Valida credenciais                     │
│                  ├── Verifica MFA (opcional)                │
│                  ├── Retorna tokens (Access, Refresh, ID)   │
│                  └── Sincroniza com AD (se configurado)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```


##### Active Directory
```
┌─────────────────────────────────────────────────────────────┐
│                Fluxo de Autenticação Corporativa            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Comerciante → auth-api → LDAP/Kerberos → AD Server         │
│                  │                                          │
│                  ├── Bind com domínio/usuário/senha         │
│                  ├── Busca sAMAccountName                   │
│                  ├── Extrai email e nome                    │
│                  ├── Mapeia SID para usuário interno        │
│                  └── Retorna JWT com claims do AD group     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

##### Apache Kafka

```
┌─────────────────────────────────────────────────────────────┐
│                    Fluxo de Mensageria                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Producer (lancamentos-api)                                 │
│       │                                                     │
│       ▼                                                     │
│  Topic: lancamentos (3 partições, retenção 7 dias)          │
│       │                                                     │
│       ├──► Consumer Group cg1 (consolidacao-worker)         │
│       ├──► Consumer Group cg2 (auditoria-worker)            │
│       │                                                     │
│       ▼                                                     │
│  DLQ (dead letter queue) para eventos com falha             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

##### Redis Cache
```
┌─────────────────────────────────────────────────────────────┐
│                    Estratégia de Cache                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Padrão: Cache-Aside                                        │
│                                                             │
│  Leitura:                                                   │
│    1. Verificar Redis (GET key)                             │
│    2. Se cache hit → retornar                               │
│    3. Se cache miss → buscar no PostgreSQL                  │
│    4. Atualizar Redis (SET key, TTL 24h)                    │
│                                                             │
│  Escrita:                                                   │
│    1. Atualizar PostgreSQL                                  │
│    2. Invalidar ou atualizar Redis                          │
│    3. TTL = 24 horas                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

```

<a id="fluxo"></a>
### 9. Fluxo Completo do Sistema
##### Fluxo Principal (Registro de Lançamento)

```
1. Comerciante autentica via Cognito/AD
2. Envia comando Registrar Lançamento
3. Sistema valida os dados (Política de Validação)
4. Persiste o agregado Lancamento
5. Dispara evento LancamentoRegistrado
6. Evento é publicado no Kafka (topic: lancamentos)
7. Consumidores processam o evento:
   a. Consolidacao-worker:
      - Atualiza SaldoDiario
      - Dispara SaldoDiarioAtualizado
      - Verifica limite de alerta
      - Atualiza Redis cache
   b. Auditoria-worker:
      - Registra evento no MongoDB
8. Se saldo < 0, envia e-mail via SendGrid
9. Comerciante consulta saldo via GET /consolidado (cache)
```

##### Diagrama de Sequência do Fluxo Principal

```
Comerciante    API Gateway    Lancamentos-api    Kafka    Consolidacao-worker    Redis    PostgreSQL    SendGrid
    │               │               │              │              │               │           │            │
    │──POST /lancamentos───────────►│              │              │               │           │            │
    │               │               │              │              │               │           │            │
    │               │               │──Salva──────►│              │               │           │            │
    │               │               │              │              │               │           │            │
    │               │               │──Publica evento────────────►│               │           │            │
    │               │               │              │              │               │           │            │
    │               │               │              │              │──Consome─────►│           │            │
    │               │               │              │              │               │           │            │
    │               │               │              │              │──Atualiza─────────────────►            │
    │               │               │              │              │               │           │            │
    │               │               │              │              │──Atualiza─────────────────────────────►│
    │               │               │              │              │               │           │            │
    │               │               │              │              │               │           │──Salva────►│
    │               │               │              │              │               │           │            │
    │               │               │              │              │               │──Cache───►│            │
    │               │               │              │              │               │           │            │
    │──GET /consolidado────────────►│              │              │               │           │            │
    │               │               │              │              │               │           │            │
    │               │               │──────────────Busca cache───────────────────►│           │            │
    │               │               │              │              │               │           │            │
    │               │               │◄─────────────Retorna saldo──────────────────│           │            │
    │               │               │              │              │               │           │            │
    │◄──Saldo───────────────────────│              │              │               │           │            │
    │               │               │              │              │               │           │            │

```

##### Fluxo de Alerta de Saldo Baixo

```
1. LancamentoRegistrado é processado
2. Consolidacao-worker calcula novo saldo
3. Política de Verificação de Limite é acionada
4. Se saldo < limite (ex: 0):
   a. Dispara evento SaldoBaixoAlertado
   b. Notificacoes-worker consome o evento
   c. Envia e-mail via SendGrid/SMTP
   d. (Opcional) Envia mensagem no Slack/Teams

```

<a id="glossario"></a>
### 10. Glossário do Event Storming
|Termo|	Definição|
|--|--|
|Comando|	Intenção de realizar uma ação no sistema|
|Evento de Domínio|	Algo que ocorreu e é relevante para o negócio|
|Agregado|	Conjunto de objetos que são tratados como uma unidade|
|Agregado Raiz|	A entidade principal que garante consistência do agregado|
|Política|	Regra de negócio que reage a eventos|
|Ator|	Entidade externa (usuário ou sistema) que interage com o sistema|
|CQRS|	Command Query Responsibility Segregation - separação entre escrita e leitura
|Cache-Aside|	Padrão onde o cache é populado sob demanda|
|Circuit Breaker|	Padrão de resiliência que previne falhas em cascata|
|DLQ (Dead Letter Queue)|	Fila para mensagens que falharam após múltiplas tentativas|
|TTL (Time To Live)|	Tempo de vida de um item no cache|
|View Materializada|	Tabela derivada de consultas para otimização de leitura|
|Saga|	Padrão para gerenciar consistência em transações distribuídas|


##### Resumo do Event Storming em Texto


```
FLUXO COMPLETO:

Comerciante
    │
    ▼
Registrar Lançamento (Comando)
    │
    ▼
LancamentoRegistrado (Evento)
    │
    ├──► Validar Lançamento (Política)
    │       └──► Rejeitar se dados inválidos
    │
    ├──► Calcular Saldo Diário (Política)
    │       │
    │       ▼
    │   SaldoDiarioAtualizado (Evento)
    │       │
    │       ├──► Verificar Limite de Alerta (Política)
    │       │       │
    │       │       └──► SaldoBaixoAlertado (Evento)
    │       │               │
    │       │               └──► Enviar E-mail (SendGrid)
    │       │
    │       └──► Atualizar Cache Redis (Política)
    │
    └──► Registrar Auditoria (Política)
            │
            └──► Salvar no MongoDB

Consultas (CQRS):
    │
    ├──► GET /consolidado → Redis → PostgreSQL
    │
    ├──► GET /extrato → PostgreSQL
    │
    └──► GET /relatorio → View Materializada → Cache

```



### Business Capability Modeling (BCM)

##### 1. Business Capability Modeling (BCM)

##### 2. Mapeamento de Domínios Funcionais

##### 3. Bounded Contexts (Contextos Delimitados)

##### 4. Requisitos Funcionais Detalhados

##### 5. Requisitos Não Funcionais Detalhados

##### 6. Diagrama BCM + Bounded Contexts


### 1. Business Capability Modeling (BCM)
##### 1.1 Hierarquia de Capacidades de Negócio

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         Nível 1: Gestão de Fluxo de Caixa                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                      Nível 2: Capacidades Primárias                         │    │
│  ├─────────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                             │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │    │
│  │  │  Gestão de      │  │  Consolidação   │  │  Relatórios e   │              │    │
│  │  │  Lançamentos    │  │  de Saldos      │  │  Auditoria      │              │    │
│  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘              │    │
│  │           │                    │                    │                       │    │
│  │           ▼                    ▼                    ▼                       │    │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │    │
│  │  │                      Nível 3: Capacidades de Suporte                │    │    │
│  │  ├─────────────────────────────────────────────────────────────────────┤    │    │
│  │  │                                                                     │    │    │
│  │  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────┐        │    │    │
│  │  │  │  Identidade   │  │  Notificações │  │  Infraestrutura   │        │    │    │
│  │  │  │  e Acesso     │  │  e Alertas    │  │  e Observabilidade│        │    │    │
│  │  │  └───────────────┘  └───────────────┘  └───────────────────┘        │    │    │
│  │  │                                                                     │    │    │
│  │  └─────────────────────────────────────────────────────────────────────┘    │    │
│  │                                                                             │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Detalhamento das Capacidades de Negócio (Nível 1)
|Capacidade|	Descrição|	Dono |	Maturidade|	KPIs|
|--|--|--|--|--|
|Gestão de Fluxo de Caixa|	Gerenciar entradas e saídas financeiras do negócio|	Diretor Financeiro|	Otimizado|	Saldo diário, fluxo mensal|

### 1.3 Detalhamento das Capacidades Primárias (Nível 2)
|Capacidade|	Descrição|	Sub-capacidades|	Métricas de Negócio|
|--|--|--|--|
|Gestão de Lançamentos|	Registrar, alterar e cancelar movimentações financeiras| Registro de débitos Registro de créditos Cancelamento Categorização|Tempo médio de registro Taxa de cancelamento Volume por categoria|
|Consolidação de Saldos|	Calcular e manter saldos diários/mensais |	Cálculo diário Cálculo mensal Projeções Histórico | Precisão do saldo Tempo de consolidação Disponibilidade do dado |
|Relatórios e Auditoria| Gerar relatórios gerenciais e manter trilha de auditoria| Extrato periódico Relatório gerencial Trilha de auditoria Exportação de dados| Tempo de geração Conformidade Rastreabilidade|

### 1.4 Detalhamento das Capacidades de Suporte (Nível 3)
|Capacidade|	Descrição|	Sub-capacidades|Métricas|
|--|--|--|--|
|Identidade e Acesso|	Gerenciar autenticação e autorização de usuários| Login (Cloud Identity) Login (AD corporativo) MFA RBAC Tempo de login| Taxa de sucesso Tentativas inválidas |
|Notificações e Alertas|Enviar alertas e notificações aos usuários| Alerta de saldo baixo Confirmação de lançamento Relatório periódico| Tempo de entrega Taxa de abertura Alertas disparados|
|Infraestrutura e Observabilidade|Garantir disponibilidade e visibilidade do sistema|Monitoramento Logging Tracing Backup/DR| Uptime MTTR Latência P95|

### 2. Mapeamento de Domínios Funcionais
##### 2.1 Mapa de Domínios (Core, Supporting, Generic)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              Mapa de Domínios - Fluxo de Caixa                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         CORE DOMAIN (Diferencial Competitivo)               │    │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │    │
│  │  │                                                                     │    │    │
│  │  │  • Gestão de Lançamentos (registro, cancelamento, categorização)    │    │    │
│  │  │  • Consolidação de Saldos (cálculo em tempo real)                   │    │    │
│  │  │  • Relatórios Gerenciais (análises financeiras)                     │    │    │
│  │  │                                                                     │    │    │
│  │  └─────────────────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                       SUPPORTING DOMAIN (Apoio ao Core)                     │    │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │    │
│  │  │                                                                     │    │    │
│  │  │  • Notificações e Alertas (e-mail, Slack, webhook)                  │    │    │
│  │  │  • Exportação de Dados (PDF, Excel, CSV)                            │    │    │
│  │  │  • Auditoria (trilha de eventos)                                    │    │    │
│  │  │                                                                     │    │    │
│  │  └─────────────────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        GENERIC DOMAIN (Comoditizado)                        │    │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │    │
│  │  │                                                                     │    │    │
│  │  │  • Identidade e Autenticação (Cloud Identity + AD)                  │    │    │
│  │  │  • Mensageria (Kafka)                                               │    │    │
│  │  │  • Cache (Redis)                                                    │    │    │
│  │  │  • Persistência (PostgreSQL, MongoDB)                               │    │    │
│  │  │  • Observabilidade (Google Cloud Observability)                     │    │    │
│  │  │                                                                     │    │    │
│  │  └─────────────────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```
<a id="requisitos"></a>
### 3. Requisitos Funcionais Detalhados

##### 4.1 Requisitos por Contexto Delimitado

##### Contexto: Identidade
|ID|	Requisito|	Descrição|	Prioridade |Critério de Aceitação|
|--|--|--|--|--|
|RF-ID-01|Login com Cloud Identity|	Autenticar usuário via Google Cloud Identity|	Must Have|	Login bem sucedido com credenciais válidas|
|RF-ID-02|Login com Active Directory|	Autenticar usuário via LDAP/Kerberos|	Must Have|	Login via domínio corporativo|
|RF-ID-03|Autenticação Multifator (MFA)|	Suporte a MFA via SMS ou TOTP|	Should Have |	Usuário deve fornecer segundo fator|
|RF-ID-04|Registro de Usuário|	Criar nova conta de comerciante|	Must Have |	Dados validados e persistidos |
|RF-ID-05|Refresh Token|Renovar token de acesso|Must Have|	Token expirado pode ser renovado|
|RF-ID-06|Logout|	Invalidar token de acesso|	Must Have	|Token não pode mais ser usado|
|RF-ID-07|RBAC (Role-Based Access Control)|	Controle de acesso baseado em papéis|	Must Have|	Administrador e Comerciante com permissões distintas|
|RF-ID-08|Recuperação de Senha|	Enviar e-mail para redefinição de senha|	Should Have|	Link válido por 1 hora|

##### Contexto: Lançamentos
|ID|	Requisito|	Descrição|	Prioridade |Critério de Aceitação|
|--|--|--|--|--|
|RF-LC-01|	Registrar Débito|	Criar lançamento de saída financeira|	Must Have|	Valor |subtraído do saldo|
|RF-LC-02|	Registrar Crédito|	Criar lançamento de entrada financeira|	Must Have|	Valor |adicionado ao saldo|
|RF-LC-03|	Listar Lançamentos|	Listar lançamentos com filtros|	Must Have|	Paginação, ordenação por data|
|RF-LC-04|	Obter Lançamento por ID|	Buscar detalhes de um lançamento|	Must Have|	Retornar 404 se não existir|
|RF-LC-05|	Cancelar Lançamento|	Estornar lançamento existente|	Must Have|	Status alterado para CANCELADO|
|RF-LC-06|	Categorizar Lançamento|	Associar categoria ao lançamento|	Should Have|	Filtro por categoria|
|RF-LC-07|	Editar Lançamento|	Alterar dados de um lançamento|	Should Have|	Apenas lançamentos não consolidados|
|RF-LC-08|	Lançamento Recorrente|	Criar lançamentos automáticos periódicos|	Could Have|	Configuração de periodicidade|

##### Contexto: Consolidação
|ID|	Requisito|	Descrição|	Prioridade |Critério de Aceitação|
|--|--|--|--|--|
|RF-CS-01|	Consultar Saldo Diário|	Obter saldo consolidado por data	|Must Have|	Retorna saldo calculado|
|RF-CS-02|	Consultar Saldo Mensal|	Obter saldo consolidado por mês	|Must Have|	Retorna saldo inicial e final|
|RF-CS-03|	Histórico de Saldos|	Listar saldos de um período	|Should Have|	Gráfico de evolução|
|RF-CS-04|	Projeção de Saldo|	Prever saldo futuro baseado em padrões	|Could Have|	Baseado em média histórica|
|RF-CS-05|	Recalcular Saldo|	Forçar recálculo de um período	|Should Have|	Administrador apenas|
|RF-CS-06|	Exportar Consolidado|	Exportar saldos para CSV	|Should Have|	Formato estruturado|


##### Contexto: Notificação
|ID|	Requisito|	Descrição|	Prioridade |Critério de Aceitação|
|--|--|--|--|--|
|RF-NT-01|	Alerta de Saldo Baixo	Notificar quando saldo < limite	|Must Have|	E-mail enviado em até 1 minuto|
|RF-NT-02|	Confirmação de Lançamento	Notificar quando lançamento é criado	|Should Have|	E-mail opcional|
|RF-NT-03|	Notificação por Slack	Enviar alertas via webhook do Slack	|Could Have|	Mensagem formatada|
|RF-NT-04|	Relatório Periódico	Enviar relatório por e-mail agendado	|Could Have|	Diário, semanal ou mensal|
|RF-NT-05|	Configurar Limite de Alerta	Usuário define limite personalizado	|Should Have|	Persistir preferência|

##### Contexto: Relatórios
|ID|	Requisito|	Descrição|	Prioridade |Critério de Aceitação|
|--|--|--|--|--|
|RF-RL-01|	Extrato Período|	Gerar extrato financeiro	|Must Have|	PDF ou Excel|
|RF-RL-02|	Relatório por Categoria|	Agrupar lançamentos por categoria	|Should Have|	Gráficos e totais|
|RF-RL-03|	Relatório Comparativo|	Comparar períodos distintos	|Could Have|	Ano anterior vs atual|
|RF-RL-04|	Download Assíncrono|	Gerar relatório em background	|Should Have|	Notificar quando pronto|


##### Contexto: Auditoria
|ID|	Requisito|	Descrição|	Prioridade |Critério de Aceitação|
|--|--|--|--|--|
|RF-AU-01|	Registrar Eventos|	Todos os eventos de domínio são registrados	|Must Have|	Log persistido em MongoDB|
|RF-AU-02|	Consultar Trilha|	Buscar eventos por usuário/período	|Should Have|	Filtros e paginação|
|RF-AU-03|	Exportar Auditoria|	Exportar logs para CSV/JSON	|Could Have|	Formato estruturado|
|RF-AU-04|	Retenção Configurável|	Configurar tempo de retenção dos logs|Should Have|	Política por tipo de evento|













