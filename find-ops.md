### Relatório FinOps - Ecossistema Fluxo de Caixa em Google Cloud
##### Sumário

[1. Visão Geral do FinOps](#visao)\
[2. Arquitetura do Ecossistema](#arquitetura)\
[3. Detalhamento de Custos por Serviço](#custo)\
[4. Estimativa de Custo Total Mensal](#mes)\
[5. Estratégias de Otimização de Custos](#visao)\
[6. Ferramentas FinOps do Google Cloud](#visao)\
[7. Monitoramento e Alertas Financeiros](#visao)\
[8. Recomendações e Melhores Práticas](#visao)




<a id="visao"></a>
### 2. Arquitetura do Ecossistema
##### 2.1 Componentes do Ecossistema

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          Ecossistema Fluxo de Caixa - GCP                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         Camada de Segurança                                 │    │
│  │  • Cloud Armor (WAF)              • Cloud Identity                          │    │
│  │  • IAM + Cloud NAT                • Secret Manager                          │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         Camada de Gateway e Rede                            │    │
│  │  • Cloud Load Balancing (HTTP/S)   • Cloud CDN (opcional)                   │    │
│  │  • Cloud DNS                       • Cloud NAT                              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                      Camada de Computação (GKE)                             │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐      │    │
│  │  │ auth-api  │ │lancamentos│ │consolidacao││relatorios │ │ workers   │      │    │
│  │  │   Pod     │ │   Pod     │ │    Pod    │ │   Pod     │ │   Pods    │      │    │
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘ └───────────┘      │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│         ┌─────────────────────────────┼─────────────────────────────┐               │
│         ▼                             ▼                             ▼               │
│  ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────────┐  │
│  │  Cloud SQL      │          │  Memorystore    │          │  Confluent Cloud    │  │ 
│  │  (PostgreSQL)   │          │  (Redis)        │          │  (Kafka Gerenciado) │  │
│  │  • Command DB   │          │  • Cache        │          │  • Mensageria       │  │
│  │  • Read DB      │          │  • Sessions     │          │  • Eventos          │  │
│  └─────────────────┘          └─────────────────┘          └─────────────────────┘  │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                      Camada de Observabilidade                              │    │
│  │  • Cloud Monitoring    • Cloud Logging    • Cloud Trace    • Error Reporting│    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

<a id="custo"></a>
### 3. Detalhamento de Custos por Serviço
##### 3.1 Google Kubernetes Engine (GKE)

Configuração: GKE Autopilot (recomendado para redução de overhead operacional)

|Recurso| Especificação| Quantidade| Preço Unitário| Custo Mensal|
|--|--|--|--|--|
|auth-api|	2 vCPU, 4 GB RAM|	3 pods|	$0.08/vCPU/h + $0.011/GB/h|	~$380
lancamentos-api|	4 vCPU, 8 GB RAM|	3 pods|	$0.08/vCPU/h + $0.011/GB/h|	~$760|
consolidacao-api|	4 vCPU, 8 GB RAM|	3 pods|	$0.08/vCPU/h + $0.011/GB/h|	~$760|
relatorios-api|	2 vCPU, 4 GB RAM|	2 pods|	$0.08/vCPU/h + $0.011/GB/h|	~$250|
consolidacao-worker|	4 vCPU, 8 GB RAM|	2 pods|	$0.08/vCPU/h + $0.011/GB/h|	~$500|
notificacoes-worker|	1 vCPU, 2 GB RAM|	2 pods|	$0.08/vCPU/h + $0.011/GB/h|	~$130|
auditoria-worker|	2 vCPU, 4 GB RAM|	2 pods|	$0.08/vCPU/h + $0.011/GB/h|	~$250|

__Subtotal GKE: ~$3.030/mês__
__Otimização: GKE Autopilot gerencia automaticamente os nodes, pagando apenas pelos recursos solicitados pelos pods, sem custo de gerenciamento de nodes.__

### 3.2 Cloud SQL (PostgreSQL)
##### Configuração: Alta disponibilidade (Primary + Regional Failover)

|Componente|	Especificação|	Custo Mensal|
|--|--|--|
|Command DB (Primary)|	8 vCPU, 32 GB RAM, 500 GB SSD|	~$650|
Command DB (Failover)|	8 vCPU, 32 GB RAM, 500 GB SSD|	~$650|
Read DB (Primary)|	4 vCPU, 16 GB RAM, 1 TB SSD|	~$450|
Read DB (Failover)|	4 vCPU, 16 GB RAM, 1 TB SSD|	~$450|
Backup Storage|	500 GB (retention 30 dias)|	~$25|
Network Egress|	1 TB/mês|	~$120|

__Subtotal Cloud SQL: ~$2.345/mês__
__Otimização: Utilizar Cloud SQL Insights para monitorar queries pesadas e otimizar índices.__

### 3.3 Memorystore (Redis Cache)

|Especificação|	Capacidade|	Preço|	Custo Mensal|
|--|--|--|--|
|Standard Tier (HA)|	10 GB|	$0.007/GB/h + $0.015/GB/h (HA)|	~$160|

__Subtotal Memorystore: ~$160/mês__

### 3.4 Confluent Cloud (Kafka Gerenciado)
##### Configuração: Kafka como serviço gerenciado (Confluent Cloud no Google Cloud Marketplace)

|Componente|	Especificação|	Custo Mensal|
|--|--|--|
|Cluster Basic|	3 brokers (small)|	~$0.15/hora (~$108/mês)|
|Cluster Standard|	3 brokers (medium) - recomendado|	~$0.35/hora (~$252/mês)|
|Dados Ingress|	5 TB/mês|	$0.11/GB	~$550|
|Dados Egress|	5 TB/mês|	$0.11/GB	~$550|
|Partições|	100 partições|	Incluído no Standard	$0|
|Retenção de Mensagens|	7 dias (padrão)|	Incluído	$0|


**Subtotal Confluent Cloud (Kafka): ~$1.352/mês**

**Alternativa Auto-gerenciada: Kafka em VMs no GCE custaria ~$500/mês, mas exige equipe dedicada para operação.**

### 3.5 Cloud Load Balancing
##### Configuração: External HTTP(S) Load Balancer (Premium Tier)

|Componente|	Quantidade|	Preço|	Custo Mensal|
|--|--|--|--|
|Forwarding Rules|	5 regras|	$0.025/hora (primeiras 5) + $0.01/hora (adicionais)|	~$38|
|Processamento de Dados|	5 TB/mês|	Variável por região|	~$50|

__Subtotal Load Balancing: ~$88/mês__
__Otimização: Para redução de custos, considerar Standard Tier para workloads não críticas.__

### 3.6 Cloud Armor (WAF)

|Componente|	Especificação|	Custo Mensal|
|--|--|--|
|Cloud Armor Standard|	Regras OWASP predefinidas + políticas customizadas|	~$180|
|Taxa por política|	$0.75/hora por política|	~$540|
|Processamento de Requisições|	10 milhões de requisições/mês|	~$0 (incluído)|

__Subtotal Cloud Armor: ~$720/mês__


### 3.7 Cloud Identity
|Componente|	Especificação|	Custo Mensal|
|--|--|--|
|Cloud Identity Premium|	10.000 usuários|	$6/usuario/ano (~$0.50/mês)	~$5.000|

__Subtotal Cloud Identity: ~$5.000/mês__
__Nota: Este é um dos maiores custos do projeto. Para 10.000 usuários ativos, Cloud Identity Premium inclui MFA, SSO e recursos avançados de segurança.__

### 3.8 Observabilidade (Cloud Monitoring, Logging, Trace)
##### Cloud Logging

|Componente|	Volume Mensal|	Preço|	Custo|
|--|--|--|---|
|Armazenamento de Logs|	5 TB (5.120 GiB)|	$0.50/GiB (após 50 GiB free)|	~$2.535|
|Logs de Rede (VPC Flow)|	1 TB|	$0.25/GiB|	~$256|
|Retenção (30+ dias)|	5 TB|	$0.01/GiB/mês|	~$51|

__Subtotal Cloud Logging: ~$2.842/mês__

__Cloud Monitoring__


|Componente|	Volume|	Preço|	Custo|
|--|--|--|--|
|Métricas (Prometheus)|	500 milhões de amostras|	$0.06/milhão (primeiros 50B)|	~$30|
|Métricas Customizadas|	500 MiB|	$0.258/MiB (primeiros 100k MiB)|	~$129|
|Uptime Checks|	1 milhão de execuções|	$0.30/1.000 execuções|	~$300|
|API Calls de Leitura|	10 milhões|	$0.01/1.000 chamadas|	~$100|

__Subtotal Cloud Monitoring: ~$559/mês__

### Cloud Trace

|Componente|	Volume|	Preço|	Custo|
|--|--|--|--|
|Spans|	500 milhões|	$0.20/milhão (após 2.5M free)|	~$99|

__Subtotal Cloud Trace: ~$99/mês__

__Total Observabilidade: ~$3.500/mês__


### 3.9 Cloud NAT
|Componente|	Especificação|	Custo| Mensal|
|--|---|--|--|
|Cloud NAT Gateway|	5 TB de dados processados|	$0.045/GB|	~$230|

__Subtotal Cloud NAT: ~$230/mês__


### 3.10 Cloud DNS
|Componente|	Especificação|	Custo| Mensal|
|--|--|--|--|
|Zonas Hospedadas|	2 zonas|	$0.20/zona/mês|	~$0.40|
|Consultas|	10 milhões|	$0.40/1 milhão|	~$4|

__Subtotal Cloud DNS: ~$5/mês__



### 3.11 Secret Manager
|Componente|	Especificação|	Custo| Mensal|
|--|--|--|--|
|Segredos Ativos|	20 secrets|	$0.06/secret/mês|	~$1.20|
|Operações de Acesso|	100.000|	$0.03/10.000|	~$0.30|

__Subtotal Secret Manager: ~$2/mês__

### 3.12 Cloud CDN (Opcional)
|Componente|	Especificação|	Custo Mensal|
|--|--|--|
|Cache Hit|	1 TB|	$0.075/GB (Américas)|	~$75|
|Cache Miss/Fill|	100 GB|	$0.075/GB|	~$7.50|

__Subtotal Cloud CDN: ~$83/mês (opcional)__


### 3.13 Confluent Cloud Connectors (Opcional - Integração com GCP)
|Componente|	Especificação|	Custo Mensal|
|--|--|--|
|BigQuery Sink| Connector	5 TB processados|	$0.50/GB|	~$2.500|
|Cloud Storage Sink Connector|	1 TB|	$0.25/GB|	~$250|


<a id="visao"></a>
### 4. Estimativa de Custo Total Mensal
#####4.1 Resumo por Categoria

|Categoria|	Serviço|	Custo Mensal (USD)|	% do Total|
|--|--|--|--|
|Identidade|	Cloud Identity Premium|	$5.000|	28,8%|
|Observabilidade|	Logging + Monitoring + Trace|	$3.500|	20,2%|
|Computação|	GKE Autopilot|	$3.030|	17,5%|
|Banco de Dados|	Cloud SQL (HA)|	$2.345|	13,5%|
|Mensageria|	Confluent Cloud (Kafka)|	$1.352|	7,8%|
|Segurança|	Cloud Armor (WAF)|	$720|	4,2%|
|Rede|	Cloud NAT + Load Balancing + DNS|	$323|	1,9%|
|Cache|	Memorystore (Redis)	|$160|	0,9%|
|Armazenamento|	Cloud Storage|	$14|	0,1%|
|Secrets|	Secret Manager|	$2|	0,0%|
|CDN (Opcional)|	Cloud CDN|	$83|	0,5%|
|Connectors (Opcional)|	Confluent Connectors|	$2.750|	15,8%|

__TOTAL (sem connectors)		$17.366	100%__
__TOTAL (com connectors)		$20.116	100%__






