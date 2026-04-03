### Relatório FinOps - Ecossistema Fluxo de Caixa em Google Cloud
##### Sumário

[1. Visão Geral do FinOps](#visao)\
[2. Arquitetura do Ecossistema](#arquitetura)\
[3. Detalhamento de Custos por Serviço](#custo)\
[4. Estimativa de Custo Total Mensal](#mes)\
[5. Estratégias de Otimização de Custos](#estrategia)\
[6. Ferramentas FinOps do Google Cloud](#ferramenta)\
[7. Monitoramento e Alertas Financeiros](#monitoramento)\
[8. Recomendações e Melhores Práticas](#recomendacao)



<a id="arquitetura"></a>
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


<a id="mes"></a>
### 4. Estimativa de Custo Total Mensal
##### 4.1 Resumo por Categoria

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

**TOTAL (sem connectors)		$17.366	100%**\
**TOTAL (com connectors)		$20.116	100%**


### 4.2 Custo Total Mensal por Ambiente
|Ambiente|	Descrição|	Custo Mensal (sem connectors)|
|--|--|--|
|Desenvolvimento|	Ambiente reduzido (20% da produção)|	~$3.473
|Homologação|	Ambiente médio (40% da produção)	|~$6.946
|Produção|	Ambiente completo	| ~$17.366

__TOTAL (3 ambientes)		~$27.785__

### 4.3 Custo Anual Estimado
|Cenário|	Custo Mensal|	Custo Anual|
|--|--|--|
|Apenas Produção|	$17.366|	~$208.392
|Produção + Suporte (3 ambientes)|	$27.785|	~$333.420
|Com Compromisso de 1 ano (CUD)|	$13.893 (20% desconto)|	~$166.714

__Nota: Compromissos de uso (Committed Use Discounts) podem reduzir custos de computação em até 57%.__


### 4.4 Comparativo: Cloud Pub/Sub vs Confluent Kafka
|Critério|	Cloud Pub/Sub|	Confluent Kafka (GCP)|	Diferença
|--|--|--|--|
|Custo Mensal (Produção)|	$500|	$1.352|	+$852 (+170%)
|Garantia de Ordem|	❌ Não|	✅ Sim|	Kafka vence
|Retenção de Mensagens|	7 dias|	Ilimitada (configurável)|	Kafka vence
|Replay de Mensagens|	Limitado|	Completo|	Kafka vence
|Ecossistema|	Nativo GCP|	Multi-cloud|	Pub/Sub| vence
|Operação|	Zero|	Zero (Confluent)|	Empate

__Decisão: Kafka escolhido pela garantia de ordem e capacidade de replay, essenciais para consistência financeira.__


<a id="estrategia"></a>
### 5. Estratégias de Otimização de Custos

##### 5.1 Compute Engine / GKE
|Estratégia|	Impacto|	Implementação|
|--|--|--|
|Committed Use Discounts (CUD)|	20-57%|	Adquirir compromisso de 1 ou 3 anos
|Right Sizing|	20-40%|	Usar Recommender API para ajustar recursos
|Spot VMs para Workers|	60-90%|	Workers não críticos podem usar Spot VMs
|Apagar recursos ociosos|	10-15%|	Automatizar desligamento em horários de baixa

##### 5.2 Cloud SQL
|Estratégia|	Impacto|	Implementação|
|--|--|--|
|CUD para Cloud SQL|	20-50%|	Compromisso de uso para instâncias
|Read Replicas sob demanda|	30%|	Ativar apenas quando necessário
|Otimização de queries|	20-40%|	Cloud SQL Insights para identificar queries lenta


##### 5.3 Confluent Kafka
|Estratégia|	Impacto|	Implementação|
|--|--|--|
|Compromisso anual Confluent|	15-25%|	Contrato enterprise
|Cluster Basic vs Standard|	40-60%|	Se ordem não for crítica, usar Basic
|Retenção reduzida|	20-30%|	Reduzir retenção de 7 para 3 dias
|Compressão de mensagens|	30-50%|	Habilitar compressão Snappy ou LZ4

### 5.4 Observabilidade
|Estratégia|	Impacto|	Implementação|
|--|--|--|
|Amostragem de logs|	50-70%|	Coletar apenas 10-20% dos logs de debug
|Retenção reduzida|	30%|	Reduzir retenção para 15 dias em não críticos
|Exclusão de logs desnecessários|	20-40%|	Filtrar health checks e requests de monitoring

##### 5.5 Cloud Identity
|Estratégia|	Impacto|	Implementação|
|--|--|--|
|Cloud Identity Free|	100%|	Para usuários básicos (sem MFA)
|Licenciamento misto|	40-60%|	Premium apenas para admins e power users

##### 5.6 Networking
|Estratégia|	Impacto|	Implementação|
|--|--|--|
|Standard Tier vs Premium Tier|	20-30%|	Usar Standard Tier para tráfego interno
|Cloud CDN|	40-60%|	Reduz egress para conteúdo estático
|Data Transfer otimizado|	20%|	Manter dados na mesma região

##### 5.7 Resumo de Economia Potencial
|Categoria|	Custo Original|	Otimização|	Custo Otimizado|	Economia
|--|--|--|--|--|
|GKE|	$3.030|	CUD 1 ano (40%) + Right Sizing|	$1.818|	$1.212
|Cloud SQL|	$2.345|	CUD + Read Replicas otimizados|	$1.640|	$705
|Confluent Kafka|	$1.352|	Compromisso + Compressão|	$1.014|	$338
|Observabilidade|	$3.500|	Amostragem + Retenção reduzida|	$1.750|	$1.750
|Cloud Identity|	$5.000|	Licenciamento misto|	$3.000|	$2.000
|Cloud Armor|	$720|	Políticas consolidadas|	$540|	$180
|Cloud NAT|	$230|	Otimização de egress|	$160|	$70

___TOTAL	$17.366		$9.922	$7.444 (43%)__

<a id="ferramenta"></a>
### 6. Ferramentas FinOps do Google Cloud
##### 6.1 Ferramentas Nativas

|Ferramenta	|Função|	Link|	Uso no Projeto|
|--|--|--|--|
|Pricing Calculator|	Estimativa de custos|	cloud.google.com/products/calculator|	Pré-projeto e planejamento
|Cloud Billing Reports|	Análise detalhada de gastos|	Console do GCP|	Monitoramento diário
|FinOps Hub|	Painel unificado de FinOps|	Cloud Console|	Governança centralizada
|Cost Management|	Recomendações de otimização|	Recommender API|	Rightsizing contínuo
|Cloud Asset Inventory|	Inventário de recursos|	Cloud Asset API|	Rastreamento de ativos
|AI-Powered Cost Anomaly Detection|	Detecção de anomalias|	Cloud Billing|	Alertas proativos


### 6.2 FinOps Hub - Funcionalidades
##### O Google Cloud FinOps Hub oferece:

|Funcionalidade|	Benefício|
|--|--|
|Painéis unificados|	Visibilidade centralizada de custos com gráficos interativos
|Sugestões otimizadas|	Recomendações baseadas em padrões de uso
|FinOps Maturity Score|	Métrica de maturidade em práticas FinOps
|Colaboração interdepartamental|	Alinhamento entre times financeiros e técnicos


### 6.3 Configuração de Orçamentos e Alertas


__Exemplo: Criar orçamento via gcloud CLI__
```
gcloud billing budgets create \
    --billing-account=XXXXXX-XXXXXX-XXXXXX \
    --display-name="fluxo-caixa-producao" \
    --budget-amount=17000 \
    --threshold-rule=percent=0.5 \
    --threshold-rule=percent=0.75 \
    --threshold-rule=percent=0.9 \
    --threshold-rule=percent=1.0
```

### 6.4 Tags e Labeling Strategy
|Tag|	Exemplo|	Uso|
|--|--|--|
|environment|	production, staging, dev|	Separar custos por ambiente
|team|	backend, frontend, infra|	Responsabilidade financeira
|cost-center|	financeiro, comercial|	Alocação de custos
|project|	fluxo-caixa, auth, lancamentos|	Custo por microsserviço
|data-classification|	critical, sensitive, public|	Governança de dados
|messaging|	kafka, confluent|	Rastreamento de custos de mensageria


<a id="monitoramento"></a>
### 7. Monitoramento e Alertas Financeiros
##### 7.1 Cloud Functions para Alertas Personalizados


# Cloud Function para monitoramento de gastos

```
python

from google.cloud import billing_v1

def check_billing_cost(event, context):
    """Verifica custos e envia alerta se exceder limite"""
    
    billing_client = billing_v1.CloudBillingClient()
    
    # Configurar limites por ambiente
    limits = {
        'production': 17000,
        'staging': 6800,
        'development': 3400
    }
    
    # Obter custo atual
    current_cost = get_current_month_cost()
    environment = get_environment_tag()
    
    if current_cost > limits.get(environment, 0):
        send_alert(
            subject=f"Alerta de Custo - {environment}",
            message=f"Custo atual: ${current_cost} / Limite: ${limits[environment]}"
        )

```

### 7.2 Dashboards de Custo no Looker Studio
##### Conecte o BigQuery (exportação de billing) ao Looker Studio para criar dashboards personalizados:

|Métrica|	Visualização|	Frequência
|--|--|--|
|Custo por serviço|	Gráfico de barras|	Diário
|Custo Kafka vs outros|	Gráfico de rosca|	Diário
|Tendência mensal|	Gráfico de linha|	Semanal
|Top 10 recursos mais caros|	Tabela classificada|	Diário
|Projeção de fim de mês|	Gauge/Medidor|	Diário


### 7.3 Métricas FinOps a Monitorar
|Métrica|	Descrição|	Alvo|
|--|--|--|
|Precisão de alocação de custos|	% de recursos com tags|	> 95%
|Cobertura de CUD	% de recursos| cobertos por compromisso|	> 70%
|Taxa de recursos ociosos|	% de recursos não utilizados|	< 10%
|Eficiência de custo|	Custo por transação|	Redução mensal de 5%
|MTTR financeiro|	Tempo para responder a anomalias|< 4 horas


<a id="recomendacao"></a>
### 8. Recomendações e Melhores Práticas
##### 8.1 Recomendações Imediatas (Curto Prazo)

|Ação|	Impacto|	Prazo|
|--|--|--|
|Implementar tagging completo|	Visibilidade|	1 semana
|Configurar budgets e alertas|	Prevenção|	1 semana
|Ativar FinOps Hub|	Governança|	1 dia
|Revisar direitos de acesso|	Segurança|	1 semana



### 8.2 Recomendações de Médio Prazo
|Ação|	Impacto|	Prazo|
|--|--|--|
|Adquirir CUD para GKE e Cloud SQL|	20-40%|	1 mês
Implementar right sizing automático|	20-30%|	2 meses
Configurar amostragem de logs|	50-70%|	1 mês
Revisar licenciamento Cloud Identity|	40-60%|	1 mês


### 8.3 Recomendações de Longo Prazo
|Ação|	Impacto|	Prazo|
|--|--|--|
|Migrar workloads para Spot VMs|	60-90%|	3 meses
|Implementar arquitetura serverless|	30-50%|	6 meses
|Negociar contrato enterprise|	15-25%|	12 meses


### 8.4 Checklist de Boas Práticas FinOps
- __Informar__
   - Exportar dados de billing para BigQuery
   - Criar dashboards de custo no Looker Studio
   - Implementar tagging obrigatório por política
   - Configurar relatórios automáticos de chargeback/showback
- __Otimizar__
   - Analisar recomendações do Recommender API semanalmente
   - Revisar e ajustar compromissos de uso trimestralmente
   - Automatizar desligamento de recursos não produtivos
   - Implementar políticas de retenção de dados
- __Operar__
   - Revisões mensais de custo com stakeholders
   - Estabelecer ownership de recursos
   - Criar budget alerts para todos os projetos
   - Documentar decisões de otimizaçã

### 9. Resumo Executivo
##### 9.1 Estimativa de Custo Base

|Cenário|	Custo Mensal|	Custo Anual
|--|--|--|
|Produção|	$15.677|	$188.124
|Desenvolvimento + Homologação + Produção|	$25.083|	$301.000
|Otimizado (com CUD + práticas FinOps)|	$8.908|	$106.896


### 9.2 Maiores Fatores de Custo
|Serviço|	% do Total| Estratégia de Redução
|--|--|--|
|Cloud Identity|	31,5%|	Licenciamento misto
|Observabilidade|	22,0%|	Amostragem + retenção reduzida
|GKE|	19,1%|	CUD + Right sizing
|Cloud SQL|	14,8%|	CUD + Read replicas otimizadas


### 9.3 Próximos Passos
1. __Semana 1:__ Implementar tagging e configurar budgets
2. __Mês 1:__ Ativar FinOps Hub e configurar dashboards
3. __Mês 2:__ Adquirir CUD para workloads estáveis
4. __Mês 3:__ Implementar right sizing automático
5. __Trimestre 2:__ Revisar e ajustar estratégia de otimização

## Nota Final: Custos Baseado Google Cloud na região us-central1