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

### 3.2 Métricas de Negócio

|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|lancamentos_total|	Counter|	Total de lançamentos|	tipo|, categoria|, usuario_tier|	Monitorar|	1min
|lancamentos_valor_total|	Counter|	Valor total (R$)|	tipo| categoria|	Monitorar|	1min
|usuarios_ativos|	Gauge|	Usuários ativos| (últimos 5min)	|role |plano	> 100	|1min
|saldo_medio|	Gauge|	Saldo médio dos usuários|	-|	Monitorar|	5min
|alertas_disparados|	Counter|	Alertas de saldo baixo|	severity| tipo_alerta|	Monitorar|	1min
|jwt_validacoes_total|	Counter|	Validações de JWT|	resultado (sucesso/falha)|	> 99%|	1min

### 3.3 Métricas de Performance
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|api_cpu_usage_seconds|	Counter|	Tempo de CPU por requisição|	endpoint|	< 0.1s|	1min
|api_memory_bytes|	Gauge|	Memória utilizada|	pod, container|	< 512MB|	15s
|api_connections_active|	Gauge|	Conexões ativas|	tipo (http, db, redis)|	< 1000|	15s
|api_db_query_duration_seconds|	Histogram|	Latência de queries no banco|	query_type|	P95 < 100ms|	15s
|api_cache_hit_ratio|	Gauge|	Taxa de acerto de cache|	cache_name|	> 90%	|1min

### 3.4 Métricas de Disponibilidade
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|api_uptime_seconds|	Gauge|	Tempo desde último restart|	pod	|> 24h|	15s
|api_health_check_success|	Gauge|	Status do health check (/health)|	endpoint|	1|	30s
|api_ready_check_success|	Gauge|	Status do readiness check|	endpoint	1|	30s

# 4. Métricas Detalhadas dos Workers
### 4.1 Métricas de Processamento

|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|worker_messages_consumed_total|	Counter	Mensagens consumidas do Kafka|	topic, partition, worker_name| Monitorar	|15s
|worker_messages_processed_total|	Counter	Mensagens processadas com sucesso|	topic, worker_name|	Monitorar	|15s
|worker_messages_failed_total|	Counter	Mensagens com erro|	topic,error_type,worker_name|	< 0.1%|	15s
|worker_processing_duration_seconds|	Histogram	Tempo de processamento|	topic, worker_name|	P95 < 100ms|	15s
|worker_retries_total|	Counter	Tentativas de reprocessamento|	topic, attempt_number|	Monitorar|	1min

### 4.2 Métricas de Kafka Consumer
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|worker_consumer_lag|	Gauge|	Atraso do consumidor|	topic, partition, consumer_group|	< 1000|	15s
|worker_consumer_lag_millis|	Gauge|	Atraso em milissegundos|	topic, consumer_group|	< 10000ms|	15s
|worker_consumer_commits_total|	Counter|	Commits realizados|	topic, partition|	Monitorar|	1min
|worker_consumer_rebalances_total|	Counter|	Rebalanceamentos do consumer group|	consumer_group|	< 10/hora|	1min
|worker_dlq_messages_total|	Counter|	Mensagens enviadas para DLQ|	topic,dlq_topic,reason|	Monitorar|	1min


4.3 Métricas de Worker Health
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|worker_cpu_usage_seconds|	Counter|	Tempo de CPU do worker|	worker_name|	< 70%|	15s
|worker_memory_bytes|	Gauge|	Memória utilizada|	worker_name|	< 512MB|	15s
|worker_gc_total|	Counter|	Coletas de garbage collector|	generation|	Monitorar|	1min
|worker_threads_active|	Gauge|	Threads ativas|	worker_name|	Monitorar|	15s
|worker_uptime_seconds|	Gauge|	Tempo desde último| restart|	worker_name|	> 24h|	15s

# 5. Métricas Detalhadas de Segurança
### 5.1 Métricas de Ameaças
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|security_waf_blocked_total|	Counter|	Requisições bloqueadas pelo WAF|	rule,ip,country,path|	0|	15s
|security_sql_injection_total|	Counter|	Tentativas de SQL Injection|	source_ip,endpoint,user_agent|	0|	15s
|security_xss_total|	Counter|	Tentativas de XSS|	source_ip,endpoint|	0|	15s
|security_path_traversal_total|	Counter|	Tentativas de path traversal|	source_ip,path|	0|	15s
|security_rce_total|	Counter|	Tentativas de RCE|	source_ip,endpoint|	0|	15s
|security_brute_force_total|	Counter|	Tentativas de força bruta|	source_ip,target_user|	< 10	|1min


### 5.2 Métricas de Autenticação
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|auth_logins_success_total|	Counter|	Logins bem-sucedidos|	provider (CI/AD),user_tier|	Monitorar|	1min
|auth_logins_failed_total|	Counter|	Logins com falha|	reason (invalid_password,user_not_found,mfa_failed)|	< 10/hora|	1min
|auth_mfa_verified_total|	Counter|	MFA verificados com sucesso|	method (sms, totp)|	Monitorar|	1min
|auth_mfa_failed_total|	Counter|	Falhas de MFA|	method,reason| < 5/hora	|1min
|auth_tokens_issued_total|	Counter|	Tokens JWT emitidos|	type (access, refresh)|	Monitorar|	1min
|auth_tokens_revoked_total|	Counter|	Tokens revogados|	reason (logout,expire,manual)|	Monitorar|	1min
|auth_tokens_invalid_total|	Counter|	Tokens inválidos recebidos|	reason (expired,signature,malformed)|	< 100/hora|	1min




### 5.3 Métricas de Autorização
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|auth_unauthorized_total|	Counter|	Tentativas de acesso não autorizado|	endpoint,user,required_role,ip|	0|	15s
|auth_forbidden_total|	Counter|	Acessos negados por permissão|	endpoint,user,missing_permission|	0|	15s
|auth_admin_actions_total|	Counter|	Ações de administrador|	 action,user|	Monitorar|	1min
|auth_impersonation_total|	Counter|	Usuários assumindo identidade|	impersonator,target_user|	Monitorar|	1min





