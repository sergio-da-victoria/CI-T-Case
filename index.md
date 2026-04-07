# CI&T Case - Fluxo de Caixa
####  Controle de fluxo de caixa diário com os lançamentos(débitos e créditos), também precisa de um relatório que disponibilize o saldo diário consolidado.


##### Sumário
[1. Visão Geral do Event Storming](#visao)\
[2. Atores / Usuários](#atores)\
[3. Comandos](#comandos)\
[4. Eventos de Domínio](#dominios)\
[5. Agregados](#agregados)\
[6. Políticas / Regras de Negócio](#negocio)\
[7. Leituras / Consultas (CQRS)](#cqrs)\
[8. Sistemas Externos](#externo)\
[9. Fluxo Completo do Sistema](#fluxo)\
[10. Glossário do Event Storming](#glossario)\
[11. Consolidação de Saldo](#consolidado)\
[12. Business Capability Modeling](/business-capability.md)\
[13. Relatório FindOps - Custos](/find-ops.md)\
[14. Trade OFF - Troubleshooting - Recomendações](/trade-off.md)\
[15. Scaffolding - Codigo](/scaffolding.md)\
[16. CI/CD Pipeline - Google Cloud Build, Cloud Deploy, Artifact Registry com TDD e DBB](/deplyment-ci-cd.md)\
[17. Camada de Segurança](/seguranca.md)/
[18. Observabilidade, Monitoração e Métricas e Logs](/observabilidade.md)



### Diagrama Domain Driven Design - Fluxo de Caixa
![DDD](/ddd-case.jpg)


### Diagrama do Event Storming - Fluxo de Caixa
![Storming](/event-storming.jpg)


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

<a id="comandos"></a>
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


<a id="negocio"></a>
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


<a id="externo"></a>
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
mermaid
```

### Diagrama C4 Model Contexto
![Contexto](/Diagrama-de-Contexto.jpg)

### Diagrama C4 Model Container
![Contêineres](/Diagrama-de-Conteineres.jpg)

### Diagrama C4 Model Componentes
![Componentes](/Diagrama-de-Componentes.jpg)

### Diagrama C4 Model Codigo
![Codigo](/Diagrama-de-Codigo.jpg)

### Diagrama Sistema BPMN
![BPMN](/Diagrama-BPMN.jpg)


### Diagrama de Sequência Detalhado
![Sequência](/Diagrama-de-sequencia.jpg)



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




<a id="consolidado"></a>
### 11. Consolidação de Saldo - (core)

__O que é o Consolidado?__
__O Consolidado é o coração do sistema de Fluxo de Caixa. Ele é responsável por calcular, armazenar e disponibilizar o saldo financeiro do comerciante em diferentes períodos (diário, semanal, mensal). O consolidado responde à pergunta fundamental: "Quanto dinheiro eu tenho em um determinado dia/período?"__

__Visão Geral do Fluxo de Funcionamento__

```

┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         FLUXO COMPLETO DO CONSOLIDADO                                   │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │  Comerciante │────▶│  API Gateway │────▶│lancamentos-api│─▶│ Command DB  │        │
│  │  (Registra)  │     │              │     │ (POST /lanc) │     │ (PostgreSQL) │        │
│  └──────────────┘     └──────────────┘     └──────┬───────┘     └──────────────┘        │
│                                                    │                                    │
│                                                    │ Publica Evento                     │
│                                                    ▼                                    │
│                                          ┌──────────────────┐                           │
│                                          │ Kafka - Topic    │                           │
│                                          │ "lancamentos"    │                           │
│                                          └────────┬─────────┘                           │
│                                                    │                                    │
│                                                    │ Consome                            │
│                                                    ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐   │
│  │                         consolidacao-worker                                      │   │
│  │                                                                                  │   │
│  │  1. Lê evento LancamentoRegistrado                                               │   │
│  │  2. Identifica data e tipo (débito/crédito)                                      │   │
│  │  3. Calcula novo saldo diário                                                    │   │
│  │  4. Atualiza Read Database (PostgreSQL)                                          │   │
│  │  5. Atualiza Redis Cache                                                         │   │
│  │  6. Verifica se saldo < 0 → Dispara alerta                                       │   │
│  │                                                                                  │   │
│  └──────────────────────────────────────────────────────────────────────────────────┘   │
│                                                    │                                    │
│                                                    ▼                                    │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────────────────────┐     │
│  │  Comerciante │────▶│  API Gateway │────▶│ consolidacao-api (GET /consolidado)│     │
│  │  (Consulta)  │     │              │     │                                      │     │
│  └──────────────┘     └──────────────┘     │  1. Verifica Redis Cache (rápido)    │     │
│                                            │  2. Se não houver, busca no Read DB  │     │
│                                            │  3. Retorna saldo consolidado        │     │
│                                            └──────────────────────────────────────┘     │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### Componentes Envolvidos no Consolidado
|Componente|	Responsabilidade
|--|--
|consolidacao-api	API| de consulta de saldos (Query Side - CQRS)
|consolidacao-worker	Worker| que processa eventos e atualiza saldos
|Read Database (PostgreSQL)|	Armazena saldos consolidados por data
|Redis Cache	Cache| de alta velocidade para consultas
|Kafka (Topic: lancamentos)|	Fila de eventos para processamento assíncrono

### Consulta do Saldo Consolidado

```
Rquest
// Comerciante consulta o saldo
GET /api/consolidado/diario?data=2025-04-05

// Fluxo da consulta:
// 1. Verifica Redis → Cache Hit (mais rápido)
// 2. Se não houver cache → Busca no PostgreSQL
// 3. Retorna o resultado

Response

{
    "data": "2025-04-05",
    "saldoInicial": 1430.50,
    "totalCreditos": 850.00,
    "totalDebitos": 350.00,
    "saldoFinal": 1930.50,
    "quantidadeTransacoes": 12,
    "mediaTransacao": 100.00
}
```


### Tipos de Consolidado

__Consolidado Diário__
|Campo|	Descrição
|--|--
|data|	Data do consolidado
|saldoInicial|	Saldo do dia anterior
|totalCreditos|	Soma de todas as entradas do dia
|totalDebitos|	Soma de todas as saídas do dia
|saldoFinal|	saldoInicial + creditos - debitos
|quantidadeTransacoes|	Número total de lançamentos


__Consolidado Semanal__
```
Agrupa os 7 dias da semana, mostrando:
Saldo inicial da semana (domingo)
Total de créditos/débitos da semana
Saldo final da semana (sábado)
Detalhamento diário
```

__Consolidado Mensal__
```
Agrupa todos os dias do mês, mostrando:
Saldo inicial do mês
Total de créditos/débitos do mês
Saldo final do mês
Detalhamento por categoria
```


__Consolidado por Período__
```
Permite consultar qualquer intervalo personalizado:
Trimestre
Semestre
Ano
Período customizado
```

### Estrutura dos Dados Consolidados
__Tabela saldo_diario (Read Database)__

|Coluna|	Tipo|	Restrição| Descrição|
|--|--|--|--
|data|	DATE|	PRIMARY| KEY	Data do consolidado (formato YYYY-MM-DD)
|total_creditos|	DECIMAL(15,2)|	NOT NULL DEFAULT 0|	Soma de todas as entradas do dia
|total_debitos|	DECIMAL(15,2)|	NOT NULL DEFAULT 0|	Soma de todas as saídas do dia
|saldo|	DECIMAL(15,2)|	NOT NULL DEFAULT 0|	Saldo final do dia
|quantidade_transacoes|	INTEGER|	NOT NULL DEFAULT 0|	Número total de lançamentos no dia
|ticket_medio|	DECIMAL(15,2)|	GENERATED|	Valor médio por transação (calculado)
|ultima_atualizacao|	TIMESTAMP|	NOT NULL|	Última vez que o registro foi atualizado
|versao|	INTEGER|	NOT NULL DEFAULT 1|	Controle de concorrência otimista
|criado_em|	TIMESTAMP|	NOT NULL DEFAULT NOW()|	Data de criação do registro


### 3.1 Dados de Exemplo
|data|	total_creditos|	total_debitos|	saldo|	qtd_transacoes|	ticket_medio
|--|--|--|--|--|--
|2025-04-01|	1.250,00|	850,00|	1.500,00|	15|	140,00
|2025-04-02|	850,00|	420,00|	1.930,00|	12|	105,83
|2025-04-03|	320,00|	180,00|	2.070,00|	8|	62,50
|2025-04-04|	150,00|	290,00|	1.930,00|	6|	73,33
|2025-04-05|	0,00|	500,00|	1.430,00|	1|	500,00
|2025-04-06|	500,00|	0,00|	1.930,00|	1|	500,00
|2025-04-07|	780,00|	350,00|	2.360,00|	10|	113,00



### Padrões Implementados no Consolidado
__CQRS (Command Query Responsibility Segregation)__

```
Command Side (Escrita)              Query Side (Leitura)
┌─────────────────────┐            ┌─────────────────────┐
│  lancamentos-api    │            │  consolidacao-api   │
│  (escreve dados)    │            │  (lê dados)         │
└──────────┬──────────┘            └──────────┬──────────┘
           │                                  │
           │ Kafka                             │
           ▼                                  ▼
┌─────────────────────┐            ┌─────────────────────┐
│  consolidacao-worker│            │  Redis Cache        │
│  (atualiza leitura) │            │  (cache)            │
└──────────┬──────────┘            └──────────┬──────────┘
           │                                  │
           ▼                                  ▼
┌─────────────────────┐            ┌─────────────────────┐
│  Read Database      │◄───────────│  PostgreSQL         │
│ (dados consolidados)│            │  (fallback)         │
└─────────────────────┘            └─────────────────────┘
```

### Event-Driven
```
Lançamento registrado → Evento publicado no Kafka
Worker consome evento → Atualiza consolidado
Desacoplamento entre escrita e leitura
```


### Cache-Aside Pattern
__Consulta de Saldo:__

```
1. Verifica Redis → Se encontrou → Retorna (10ms)
2. Se não encontrou → Busca no PostgreSQL (100ms)
3. Atualiza Redis para próximas consultas
4. Retorna resultado
```


### Benefícios do Consolidado
__Benefício	Descrição__

```
1. Performance	Cache Redis reduz latência de 100ms para <10ms
3. Escalabilidade	Leitura e escrita separadas (CQRS)
4. Consistência Eventual	Processamento assíncrono via Kafka
5. Disponibilidade	Cache e banco de dados redundantes
6. Rastreabilidade	Histórico completo de saldos
```


### Resumo do Fluxo

```

1. Comerciante → Registra lançamento (débito/crédito)
2. lancamentos-api → Publica evento no Kafka
3. Kafka → Topic "lancamentos"
4. consolidacao-worker → Consome evento e calcula novo saldo
5. Worker → Atualiza PostgreSQL (Read DB)
6. Worker → Atualiza Redis Cache
7. Worker → Verifica se saldo < 0 (alerta)
8. Comerciante → Consulta saldo via consolidacao-api
9. consolidacao-api → Retorna do Redis (cache) ou PostgreSQL
```

__O Consolidado é o componente que transforma dados brutos de lançamentos em informação financeira valiosa para o comerciante, permitindo tomada de decisão em tempo real.__
