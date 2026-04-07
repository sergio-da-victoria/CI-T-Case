# Observabilidade, Monitoração, Métricas e Logs

### Sumário


[1. Visão Geral da Observabilidade](#visao)\
[2. Ferramentas do Google Cloud Observability](#ferramenta)\
[3. Métricas Detalhadas das APIs](#api)\
[4. Métricas Detalhadas dos Workers](#work)\
[5. Métricas Detalhadas de Segurança](#seguranca)\
[6. Métricas Detalhadas dos Bancos de Dados](#dado)\
[7. Métricas Detalhadas do Kafka](#kafka)\
[8. Métricas Detalhadas de Infraestrutura (GKE)](#gke)\
[9. Dashboards no Cloud Monitoring](#monitoring)\
[10. Alertas no Cloud Monitoring](#alerta)\
[11. Instrumentação em C# .NET 10](#stack)\
[12. Terraform para Observabilidade](#terraform)\
[13. Tabela Resumo de Métricas por Serviço](#resumo)

# 1. Visão Geral da Observabilidade

### 1.1 Os Três Pilares da Observabilidade

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    OS TRÊS PILARES DA OBSERVABILIDADE                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                              MÉTRICAS                                       │    │
│  │  • O quê está acontecendo?                                                  │    │
│  │  • Latência P50, P95, P99                                                   │    │
│  │  • Taxa de erros (5xx, 4xx)                                                 │    │
│  │  • Throughput (req/seg)                                                     │    │
│  │  • Uso de CPU/Memória                                                       │    │
│  │  • Ferramenta: Cloud Monitoring + Prometheus                                │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                              LOGS                                           │    │
│  │  • Por quê aconteceu?                                                       │    │
│  │  • Logs estruturados (JSON)                                                 │    │
│  │  • Logs de aplicação                                                        │    │
│  │  • Logs de segurança                                                        │    │
│  │  • Logs de auditoria                                                        │    │
│  │  • Ferramenta: Cloud Logging                                                │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                              TRACES                                         │    │
│  │  • Onde está o problema?                                                    │    │
│  │  • Distributed tracing (OpenTelemetry)                                      │    │
│  │  • Latência por span                                                        │    │
│  │  • Mapa de dependências                                                     │    │
│  │  • Root cause analysis                                                      │    │
│  │  • Ferramenta: Cloud Trace                                                  │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Arquitetura de Observabilidade

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ARQUITETURA DE OBSERVABILIDADE                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                          APLICAÇÕES (C# .NET 10)                            │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │    │
│  │  │   Auth API  │  │Lancamentos  │  │Consolidacao │  │   Worker    │         │    │
│  │  │             │  │    API      │  │    API      │  │             │         │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │    │
│  │                              │                                              │    │
│  │                    OpenTelemetry SDK                                        │    │
│  │              (Metrics, Traces, Logs)                                        │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                    GOOGLE CLOUD OBSERVABILITY                               │    │
│  │                                                                             │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │    │
│  │  │  Cloud Metrics  │  │  Cloud Logging  │  │   Cloud Trace   │              │    │
│  │  │                 │  │                 │  │                 │              │    │
│  │  │ • Métricas de   │  │ • Logs de       │  │ • Distributed   │              │    │
│  │  │   aplicação     │  │   aplicação     │  │   tracing       │              │    │
│  │  │ • Métricas de   │  │ • Logs de       │  │ • Análise de    │              │    │
│  │  │   infraestrutura│  │   segurança     │  │   latência      │              │    │
│  │  │ • Métricas de   │  │ • Logs de       │  │ • Mapa de       │              │    │
│  │  │   banco de dados│  │   auditoria     │  │   dependências  │              │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘              │    │
│  │                                                                             │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │    │
│  │  │  Cloud Profiler │  │  Error Reporting│  │  Cloud Debugger │              │    │
│  │  │                 │  │                 │  │                 │              │    │
│  │  │ • Análise de    │  │ • Agregação de  │  │ • Debug remoto  │              │    │
│  │  │   CPU/Memória   │  │   exceções      │  │ • Breakpoints   │              │    │
│  │  │ • Flame graphs  │  │ • Stack traces  │  │ • Variáveis     │              │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                    DASHBOARDS E ALERTAS                                     │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │    │
│  │  │  Cloud Monitor  │  │    Looker       │  │    Grafana      │              │    │
│  │  │   Dashboards    │  │    Studio       │  │   (Custom)      │              │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘              │    │
│  │                                                                             │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │    │
│  │  │   PagerDuty     │  │     Slack       │  │     E-mail      │              │    │
│  │  │   (Crítico)     │  │   (Alta/Média)  │  │   (Média/Baixa) │              │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

# 2. Ferramentas do Google Cloud Observability

### 2.1 Cloud Monitoring

|Funcionalidade|	Descrição|	Uso no Projeto|
|--|--|--
|Métricas|	Coleta de métricas de infraestrutura e aplicação|	Métricas de API, Workers, Banco de Dados
|Alertas|	Definição de condições e notificações|	Alertas de latência, erro, segurança
|Dashboards|	Visualização personalizada de métricas|	Dashboards por serviço
|Uptime Checks|	Verificação de disponibilidade de endpoints|	Health checks das APIs
|Service Monitoring|	Monitoramento de SLIs e SLOs|	SLIs de disponibilidade e latência



### 2.2 Cloud Logging

|Funcionalidade|	Descrição|	Uso no Projeto|
|--|--|--
|Logs Explorer|	Consulta e análise de logs|	Debugging e investigação
|Logs-based Metrics|	Criação de métricas a partir de logs|	Contagem de erros por tipo
|Log Sinks|	Exportação de logs para destinos externos|	BigQuery para análise de longo prazo
|Log Router|	Roteamento de logs para diferentes destinos|	Separação por severidade

### 2.3 Cloud Trace

|Funcionalidade|	Descrição|	Uso no Projeto|
|--|--|--
|Distributed Tracing|	Rastreamento de requisições entre serviços|	Latência por endpoint
|Trace Explorer|	Análise detalhada de traces|	Root cause analysis
|Latency Analysis|	Identificação de gargalos|	Otimização de performance
|Service Map|	Visualização de dependências|	Mapa de comunicação entre APIs

### 2.4 Cloud Profiler
|Funcionalidade|	Descrição|	Uso no Projeto|
|--|--|--
|CPU Profiling|	Análise de uso de CPU|	Otimização de código
|Heap Profiling|	Análise de uso de memória|	Detecção de memory leaks
|Flame Graphs|	Visualização de consumo de recursos|	Identificação de hotspots

### 2.5 Error Reporting
|Funcionalidade|	Descrição|	Uso no Projeto|
|--|--|--
|Funcionalidade|	Descrição|	Uso no Projeto
|Error Aggregation|	Agrupamento de erros similares|	Identificação de padrões de falha
|Stack Traces|	Captura de stack traces|	Debugging
|Notifications|	Alertas de novos erros|	Notificações para equipe

# 3. Métricas Detalhadas das APIs

### 3.1 Métricas Obrigatórias (RED Method)

|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|api_requests_total|	Counter|	Requisições por segundo|	endpoint, method, status, version|	Monitorar|	15s
|api_request_duration_seconds|	Histogram|	Latência das requisições|	endpoint, method, quantile|	P95 < 500ms|	15s
|api_errors_total|	Counter|	Taxa de erros (5xx, 4xx)|	endpoint, error_type, status_code|	< 0.1%|	15s







