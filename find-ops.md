### Relatório FinOps - Ecossistema Fluxo de Caixa em Google Cloud
##### Sumário

[1. Visão Geral do FinOps](#visao)\
[2. Arquitetura do Ecossistema](#arquitetura)\
[3. Detalhamento de Custos por Serviço](#visao)\
[4. Estimativa de Custo Total Mensal](#visao)\
[5. Estratégias de Otimização de Custos](#visao)\
[6. Ferramentas FinOps do Google Cloud](#visao)\
[7. Monitoramento e Alertas Financeiros](#visao)\
[8. Recomendações e Melhores Práticas](#visao)




<a id="visao"></a>
###  3. Comandos
Comandos representam ações ou intenções iniciadas por um ator.




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

### 2. Detalhamento de Custos por Serviço
##### 2.1 Google Kubernetes Engine (GKE)

Configuração: GKE Autopilot (recomendado para redução de overhead operacional)

|Recurso| Especificação| Quantidade| Preço| Unitário| Custo Mensal|
|--|--|--|--|--|-|
|auth-api|	2 vCPU, 4 GB RAM	3 pods	$0.08/vCPU/h + $0.011/GB/h	~$380
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

|Especificação|	Capacidade|	Preço	Custo Mensal|
|--|--|--|
|Standard Tier (HA)|	10 GB|	$0.007/GB/h + $0.015/GB/h (HA)|	~$160|

__Subtotal Memorystore: ~$160/mês__

