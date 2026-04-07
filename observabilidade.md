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



### 5.4 Métricas de Segurança de Dados
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|security_secret_access_total|	Counter|	Acessos a secrets|	secret_name, accessed_by_type (human/service)|	Monitorar|	1min
|security_encryption_keys_rotated_total|	Counter|	Rotações de chaves de criptografia|	key_name,status|	Monitorar|	1min
|security_backup_encrypted_total|	Counter|	Backups criptografados|	backup_type,size|	Monitorar|	1dia
|security_data_exfiltration_detected|	Counter|	Tentativas de exfiltração de dados|	source_ip,volume,destination|	0|	15s


# 6. Métricas Detalhadas dos Bancos de Dados
### 6.1 Métricas do PostgreSQL (Cloud SQL)
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|cloudsql_cpu_utilization|	Gauge|	Uso de CPU|	instance,environment|	< 80%|	1min
|cloudsql_memory_utilization|	Gauge|	Uso de memória|	instance|	< 85%|	1min
|cloudsql_disk_utilization|	Gauge|	Uso de disco|	instance|	< 80%|	1min
|cloudsql_connections|	Gauge|	Conexões ativas|	instance,database|	< 80% do limite|	15s
|cloudsql_replica_lag_seconds|	Gauge|	Atraso da réplica|	instance|	< 10s|	1min
|postgresql_transactions_total|	Counter|	Transações por segundo|	instance,database|	Monitorar|	15s
|postgresql_slow_queries_total|	Counter|	Queries lentas (> 1s)|	instance,database,query_hash|	< 10/hora|	1min
|postgresql_deadlocks_total|	Counter|	Deadlocks detectados|	instance,database|	0|	1min
|postgresql_cache_hit_ratio|	Gauge|	Taxa de acerto do cache|	instance,cache_type (shared, effective)|	> 95%|	1min
|postgresql_temp_files_total|	Counter|	Arquivos temporários criados|	instance,database|	< 100/hora|	1min


### 6.2 Métricas do Redis (Memorystore)
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|redis_cpu_utilization|	Gauge|	Uso de CPU|	instance|	< 70%	|1min
|redis_memory_usage_ratio|	Gauge|	Uso de memória|	instance|	< 80%|	1min
|redis_connected_clients|	Gauge|	Clientes conectados|	instance|	Monitorar|	15s
|redis_cache_hit_ratio|	Gauge|	Taxa de acerto de cache|	instance|	> 90%|	1min
|redis_evicted_keys_total|	Counter|	Chaves removidas por evicção|	instance,reason (volatile, allkeys)|	< 1000/hora|	1min
|redis_expired_keys_total|	Counter|	Chaves expiradas|	instance|	Monitorar|	1min
|redis_keyspace_hits_total|	Counter|	Acessos com sucesso|	instance,database|	Monitorar|	15s
|redis_keyspace_misses_total|	Counter|	Acessos sem sucesso|	instance,database|	< 10%|	15s
|redis_commands_processed_total|	Counter|	Comandos processados|	instance,command|	Monitorar|	15s
|redis_replication_lag_seconds|	Gauge|	Atraso da replicação|	instance|	< 5s|	1min


### 6.3 Métricas do MongoDB (Atlas)
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|mongodb_cpu_utilization| Gauge|	Uso de CPU|	cluster,node|	< 80%|	1min
|mongodb_memory_utilization|	Gauge|	Uso de memória|	cluster|	< 85%|	1min
|mongodb_disk_utilization|	Gauge|	Uso de disco|	cluster|	< 80%|	1min
|mongodb_connections_current|	Gauge|	Conexões atuais|	cluster|	< 80% do limite|	15s
|mongodb_operation_latencies|	Histogram|	Latência de operações|	cluster,operation (reads, writes, commands)|	reads < 100ms, writes < 200ms|	15s
|mongodb_operations_total|	Counter|	Total de operações|	cluster,operation,database|	Monitorar|	15s
|mongodb_documents_count|	Gauge|	Total de documentos|	cluster,database,collection|	Monitorar|	5min
|mongodb_index_size_bytes|	Gauge|	Tamanho dos índices|	cluster,database,collection|	Monitorar|	5min
|mongodb_oplog_utilization|	Gauge|	Uso do oplog|	cluster|	< 80%|	1min
|mongodb_cache_utilization|	Gaug|e	Uso do cache WiredTiger|	cluster|	< 80%|	1min

# 7. Métricas Detalhadas do Kafka
### 7.1 Métricas de Produção
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|kafka_messages_in_per_sec|	Gauge|	Mensagens recebidas por segundo|	topic,partition|	Monitorar|	15s
|kafka_bytes_in_per_sec|	Gauge|	Bytes recebidos por segundo|	topic,partition|	Monitorar|	15s
|kafka_bytes_out_per_sec|	Gauge|	Bytes enviados por segundo|	topic,partition|	Monitorar|	15s
|kafka_produce_requests_total|	Counter|	Requisições de produção|	topic,client_id|	Monitorar|	15s
|kafka_produce_errors_total|	Counter|	Erros na produção|	topic,error_type|	< 0.1%|	1min
|kafka_produce_latency_seconds|	Histogram|	Latência de produção|	topic|	P95 < 50ms|	15s


### 7.2 Métricas de Consumo
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|kafka_consumer_lag|	Gauge|	Atraso do consumidor|	topic,partition,consumer_group|	< 1000|	15s
|kafka_consumer_lag_millis|	Gauge|	Atraso em milissegundos|	topic,consumer_group|	< 10000ms|	15s
|kafka_consumer_commits_total|	Counter|	Commits realizados|	topic,partition,consumer_group|	Monitorar|	1min
|kafka_consumer_errors_total|	Counter|	Erros do consumidor| consumer_group,error_type	|< 10/hora|	1min
|kafka_fetch_requests_total|	Counter|	Requisições de fetch| topic, consumer_group|	Monitorar|	15s
|kafka_fetch_latency_seconds|	Histogram|	Latência de fetch|	topic|	P95 < 100ms|	15


### 7.3 Métricas de Cluster
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|kafka_partitions_under_replicated|	Gauge|	Partições sub-replicadas|	topic|	0|	1min
|kafka_offline_partitions|	Gauge|	Partições offline|	topic|	0|	1min
|kafka_controller_count|	Gauge|	Número de controllers|	cluster|	1|	1min
|kafka_active_controllers|	Gauge|	Controllers ativos|	cluster|	1|	1min
|kafka_isr_shrinks_total|	Counter|	Reduções de ISR|	topic,partition|	Monitorar|	1min
|kafka_isr_expands_total|	Counter|	Expansões de ISR|	topic, partition|	Monitorar|	1min
|kafka_leader_elections_total|	Counter|	Eleições de líder|	topic, partition|	< 10/hora|	1min

### 8.1 Métricas de Rede
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|gke_cluster_cpu_utilization|	Gauge|	Uso de CPU do cluster|	cluster_name,node_pool|	< 80%|	1min
|gke_cluster_memory_utilization|	Gauge|	Uso de memória do cluster|	cluster_name,node_pool|	< 85%|	1min
|gke_cluster_disk_utilization|	Gauge|	Uso de disco dos nós|	cluster_name,node|	< 80%|	1min
|gke_node_count|	Gauge|	Número de nós|	cluster_name,node_pool|	Monitorar|	1min
|gke_pod_count|	Gauge|	Número de pods|	cluster_name,namespace|	Monitorar|	15s
|gke_container_restarts_total|	Counter|	Reinicializações de containers|	pod,container,reason|	< 10/hora|	1min

### 8.2 Métricas de Rede
|Métrica|	Tipo|	Descrição|	Labels|	Alvo|	Frequência
|--|--|--|--|--|--
|gke_network_received_bytes|	Counter|	Bytes recebidos|	pod,namespace|	Monitorar|	15s
|gke_network_sent_bytes|	Counter|	Bytes enviados|	pod,namespace|	Monitorar|	15s
|gke_network_received_packets|	Counter|	Pacotes recebidos|	pod,namespace| Monitorar|	15s
|gke_network_sent_packets|	Counter|	Pacotes enviados|	pod,namespace	|Monitorar	|15s
|gke_network_errors_total|	Counter|	Erros de rede|	pod,namespace,direction|	< 100/hora|	1min


# 9. Dashboards no Cloud Monitoring
### 9.1 Dashboard da API

```
hcl

resource "google_monitoring_dashboard" "api_dashboard" {
  dashboard_json = <<EOF
{
  "displayName": "Fluxo Caixa - API Dashboard",
  "gridLayout": {
    "columns": 2,
    "widgets": [
      {
        "title": "📊 Request Rate (req/s)",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/api_requests_total' | rate(1m)",
            "plotType": "LINE",
            "minAlignmentPeriod": "60s"
          }],
          "yAxis": {
            "label": "Requests per second"
          }
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "📈 Error Rate (5xx)",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/api_requests_total' | filter (status =~ '5..') | rate(1m)",
            "plotType": "LINE"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "⏱️ Latency (P95)",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/api_request_duration_seconds' | histogram_quantile(0.95)"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "👥 Active Users",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/usuarios_ativos'"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "📦 Total Lancamentos by Type",
        "pieChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/lancamentos_total' | group_by [tipo]"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "💰 Total Value by Category",
        "pieChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/lancamentos_valor_total' | group_by [categoria]"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "🔐 JWT Validation Success Rate",
        "scorecard": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/jwt_validacoes_total' | filter (resultado = 'sucesso') | rate(1m)"
          }]
        },
        "height": 3,
        "width": 1
      },
      {
        "title": "💾 API Memory Usage",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch k8s_container | metric 'kubernetes.io/container/memory/used_bytes' | filter (namespace_name = 'fluxo-caixa')"
          }]
        },
        "height": 3,
        "width": 1
      }
    ]
  }
}
EOF
}
```

### 9.2 Dashboard dos Workers

```
hcl


resource "google_monitoring_dashboard" "workers_dashboard" {
  dashboard_json = <<EOF
{
  "displayName": "Fluxo Caixa - Workers Dashboard",
  "gridLayout": {
    "columns": 2,
    "widgets": [
      {
        "title": "📨 Messages Processed by Worker",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/worker_messages_processed_total' | rate(1m)"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "⚠️ Worker Errors",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/worker_messages_failed_total' | rate(1m)"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "⏱️ Processing Duration (P95)",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/worker_processing_duration_seconds' | histogram_quantile(0.95)"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "📊 Kafka Consumer Lag",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch kafka | metric 'kafka.consumer_lag' | group_by [consumer_group]"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "🔄 Worker Retries",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/worker_retries_total' | rate(1m)"
          }]
        },
        "height": 3,
        "width": 1
      },
      {
        "title": "🗑️ DLQ Messages",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/worker_dlq_messages_total' | rate(1m)"
          }]
        },
        "height": 3,
        "width": 1
      }
    ]
  }
}
EOF
}

```

### 9.3 Dashboard de Segurança

```

hcl


resource "google_monitoring_dashboard" "security_dashboard" {
  dashboard_json = <<EOF
{
  "displayName": "Fluxo Caixa - Security Dashboard",
  "gridLayout": {
    "columns": 2,
    "widgets": [
      {
        "title": "🛡️ Security Incidents by Type",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/security_incidents_total'"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "🚫 WAF Blocked Requests",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/security_waf_blocked_total' | rate(1m)"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "🔑 Authentication Failures",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/auth_logins_failed_total' | rate(1m)"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "🚨 Unauthorized Access Attempts",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/auth_unauthorized_total' | rate(1m)"
          }]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "💉 SQL Injection Attempts",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/security_sql_injection_total' | rate(1m)"
          }]
        },
        "height": 3,
        "width": 1
      },
      {
        "title": "🔐 Secret Access by Type",
        "pieChart": {
          "dataSets": [{
            "timeSeriesQuery": "fetch prometheus_target | metric 'prometheus.googleapis.com/security_secret_access_total' | group_by [accessed_by_type]"
          }]
        },
        "height": 3,
        "width": 1
      }
    ]
  }
}
EOF
}

```

# 10. Alertas no Cloud Monitoring

### 10.1 Alertas da API
|Alerta|	Severidade|	Condição|	Duração|	Canais
|--|--|--|--|--
|API High Latency|	🔴 CRÍTICA|	P95 > 500ms|	5min|	PagerDuty,Slack
|API High Error Rate|	🔴 CRÍTICA|	Erros 5xx > 1%|	2min|	PagerDuty,Slack
|API Low Availability|	🔴 CRÍTICA|	Uptime < 99.9%|	5min|	PagerDuty
|API High CPU|	🟠 ALTA|	CPU > 80%|	10min|	Slack
|API High Memory|	🟠 ALTA|	Memória > 85%|	10min|	Slack
|API Many 4xx Errors|	🟡 MÉDIA|	4xx > 10%|	5min|	Slack,E-mail


### 10.2 Alertas dos Workers
|Alerta|	Severidade|	Condição|	Duração|	Canais
|--|--|--|--|--
Kafka High Lag|	🔴 CRÍTICA|	Consumer lag > 5000|	5min|	PagerDuty,Slack
Worker High Error Rate|	🔴 CRÍTICA|	Erros > 10/min|	2min|	PagerDuty
Worker Processing Slow|	🟠 ALTA|	P95 processing > 500ms|	5min|	Slack
Worker Down|	🔴 CRÍTICA|	Worker offline|	1min|	PagerDuty
DLQ Growing|	🟠 ALTA|	DLQ messages > 1000|	10min|	Slack


### 10.3 Alertas de Segurança
|Alerta|	Severidade|	Condição|	Duração|	Canais
|--|--|--|--|--
|SQL Injection Detected|	🔴 CRÍTICA|	Qualquer tentativa|	0s|	PagerDuty, Slack
|XSS Attack Detected|	🔴 CRÍTICA|	Qualquer tentativa|	0s|	PagerDuty, Slack
|Brute Force Attack|	🟠 ALTA|	> 10 falhas/min|	1min|	PagerDuty,Slack
|Unauthorized Access|	🔴 CRÍTICA|	Qualquer acesso negado|	0s|	PagerDuty,Slack
|WAF Block Spike|	🟠 ALTA|	> 100 bloqueios/min|	1min|	Slack
|Secret Accessed by Human|	🟠 ALTA|	Secret de app acessada|	0s|	Slack|


### 10.4 Alertas de Banco de Dados
|Alerta|	Severidade|	Condição|	Duração|	Canais
|--|--|--|--|--
|PostgreSQL High CPU|	🟠 ALTA	CPU| > 80%|	10min|	Slack
|PostgreSQL Low Storage|	🔴 CRÍTICA|	Disco > 85%|	5min|	PagerDuty
|Redis Low Cache Hit|	🟡 MÉDIA|	Cache hit < 80%|	10min|	Slack
|MongoDB High Latency|	🟠 ALTA|	Reads > 200ms|	5min|	Slack
|Database Connection Spike|	🟡 MÉDIA|	Conexões > 80%|	5min|	Slack


# 11. Instrumentação em C# .NET 10
### 11.1 Program.cs - Configuração OpenTelemetry


```

csharp

// Program.cs - Configuração completa de observabilidade
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;
using OpenTelemetry.Resources;
using OpenTelemetry.Exporter;

var builder = WebApplication.CreateBuilder(args);

// Configurar OpenTelemetry
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(
            serviceName: "fluxo-caixa-api",
            serviceVersion: "1.0.0",
            serviceInstanceId: Environment.MachineName)
        .AddAttributes(new Dictionary<string, object>
        {
            ["deployment.environment"] = builder.Environment.EnvironmentName,
            ["service.namespace"] = "fluxo-caixa",
            ["cloud.provider"] = "gcp",
            ["cloud.region"] = "us-central1"
        }))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation(options =>
        {
            options.RecordException = true;
            options.Filter = (httpContext) =>
                !httpContext.Request.Path.StartsWithSegments("/health") &&
                !httpContext.Request.Path.StartsWithSegments("/metrics");
            options.EnrichWithHttpRequest = (activity, request) =>
            {
                activity.SetTag("http.user_agent", request.Headers.UserAgent.ToString());
                activity.SetTag("http.client_ip", request.HttpContext.Connection.RemoteIpAddress?.ToString());
            };
            options.EnrichWithHttpResponse = (activity, response) =>
            {
                activity.SetTag("http.status_code", response.StatusCode);
            };
        })
        .AddHttpClientInstrumentation(options =>
        {
            options.RecordException = true;
            options.EnrichWithHttpRequestMessage = (activity, request) =>
            {
                activity.SetTag("http.request.method", request.Method.Method);
            };
        })
        .AddSqlClientInstrumentation(options =>
        {
            options.RecordException = true;
            options.SetDbStatementForText = true;
            options.EnableConnectionLevelAttributes = true;
        })
        .AddRedisInstrumentation()
        .AddKafkaInstrumentation()
        .AddSource("FluxoCaixa.*")
        .SetSampler(new AlwaysOnSampler())
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(builder.Configuration["OTLP:Endpoint"] ?? "http://localhost:4317");
            options.Protocol = OtlpExportProtocol.Grpc;
        }))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddProcessInstrumentation()
        .AddEventCountersInstrumentation()
        .AddMeter("FluxoCaixa.*")
        .AddPrometheusExporter());

// Métricas customizadas
builder.Services.AddSingleton<MetricsService>();
builder.Services.AddSingleton<SecurityMetricsService>();

// Cloud Logging
builder.Logging.ClearProviders();
builder.Logging.AddConsole();
builder.Logging.AddGoogleCloudLogging(options =>
{
    options.ProjectId = builder.Configuration["GoogleCloud:ProjectId"];
    options.LogName = "fluxo-caixa-api-logs";
    options.IncludeEventId = true;
    options.IncludeScopes = true;
    options.ResourceLabels = new Dictionary<string, string>
    {
        ["service"] = "fluxo-caixa-api",
        ["environment"] = builder.Environment.EnvironmentName
    };
});

// Cloud Error Reporting
builder.Services.AddGoogleErrorReporting(options =>
{
    options.ProjectId = builder.Configuration["GoogleCloud:ProjectId"];
    options.ServiceName = "fluxo-caixa-api";
    options.Version = "1.0.0";
});

// Health Checks
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("ReadDb")!)
    .AddRedis(builder.Configuration["Redis:ConnectionString"]!)
    .AddKafka(builder.Configuration["Kafka:BootstrapServers"]!);

var app = builder.Build();

// Middleware para métricas
app.UseMiddleware<MetricsMiddleware>();
app.UseMiddleware<TracingMiddleware>();
app.UseMiddleware<LoggingMiddleware>();

// Endpoints de observabilidade
app.MapPrometheusScrapingEndpoint();
app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = _ => false
});

app.Run();

```

11.2 Métricas Customizadas

```

csharp

// MetricsService.cs - Métricas de negócio
public class MetricsService
{
    private readonly Counter<long> _lancamentosCounter;
    private readonly Counter<double> _lancamentosValorCounter;
    private readonly Histogram<double> _requestDurationHistogram;
    private readonly UpDownCounter<long> _usuariosAtivosGauge;
    private readonly Counter<long> _jwtValidacoesCounter;
    private readonly Histogram<double> _dbQueryDurationHistogram;
    private readonly Counter<long> _cacheHitsCounter;
    private readonly Counter<long> _cacheMissesCounter;

    public MetricsService(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("FluxoCaixa.API");

        _lancamentosCounter = meter.CreateCounter<long>(
            "lancamentos_total",
            description: "Total number of lancamentos");

        _lancamentosValorCounter = meter.CreateCounter<double>(
            "lancamentos_valor_total",
            unit: "BRL",
            description: "Total value of lancamentos");

        _requestDurationHistogram = meter.CreateHistogram<double>(
            "request_duration_seconds",
            unit: "s",
            description: "Request duration in seconds");

        _usuariosAtivosGauge = meter.CreateUpDownCounter<long>(
            "usuarios_ativos",
            description: "Number of active users");

        _jwtValidacoesCounter = meter.CreateCounter<long>(
            "jwt_validacoes_total",
            description: "Total JWT validations");

        _dbQueryDurationHistogram = meter.CreateHistogram<double>(
            "db_query_duration_seconds",
            unit: "s",
            description: "Database query duration");

        _cacheHitsCounter = meter.CreateCounter<long>(
            "cache_hits_total",
            description: "Total cache hits");

        _cacheMissesCounter = meter.CreateCounter<long>(
            "cache_misses_total",
            description: "Total cache misses");
    }

    public void RecordLancamento(string tipo, string categoria, string usuarioTier)
    {
        _lancamentosCounter.Add(1, new KeyValuePair<string, object>[]
        {
            new("tipo", tipo),
            new("categoria", categoria),
            new("usuario_tier", usuarioTier)
        });
    }

    public void RecordLancamentoValor(double valor, string tipo, string categoria)
    {
        _lancamentosValorCounter.Add(valor, new KeyValuePair<string, object>[]
        {
            new("tipo", tipo),
            new("categoria", categoria)
        });
    }

    public void RecordRequestDuration(double durationSeconds, string endpoint, string method, string statusCode)
    {
        _requestDurationHistogram.Record(durationSeconds, new KeyValuePair<string, object>[]
        {
            new("endpoint", endpoint),
            new("method", method),
            new("status_code", statusCode)
        });
    }

    public void RecordActiveUsers(long count)
    {
        _usuariosAtivosGauge.Add(count - GetCurrentActiveUsers());
    }

    public void RecordJwtValidation(string resultado, string motivo = null)
    {
        var tags = new List<KeyValuePair<string, object>> { new("resultado", resultado) };
        if (motivo != null) tags.Add(new("motivo", motivo));
        
        _jwtValidacoesCounter.Add(1, tags);
    }

    public void RecordDbQueryDuration(double durationSeconds, string queryType, string database)
    {
        _dbQueryDurationHistogram.Record(durationSeconds, new KeyValuePair<string, object>[]
        {
            new("query_type", queryType),
            new("database", database)
        });
    }

    public void RecordCacheHit(string cacheName)
    {
        _cacheHitsCounter.Add(1, new KeyValuePair<string, object>[] { new("cache", cacheName) });
    }

    public void RecordCacheMiss(string cacheName)
    {
        _cacheMissesCounter.Add(1, new KeyValuePair<string, object>[] { new("cache", cacheName) });
    }

    public double GetCacheHitRatio(string cacheName)
    {
        // Implementar cálculo da taxa de acerto
        return 0.95;
    }

    private long GetCurrentActiveUsers()
    {
        // Implementar lógica para obter usuários ativos
        return 0;
    }
}

```

# 12. Terraform para Observabilidade

```
hcl

# observability.tf - Configuração completa de observabilidade

# Notification Channels
resource "google_monitoring_notification_channel" "pagerduty" {
  display_name = "PagerDuty - Critical Alerts"
  type         = "pagerduty"
  
  labels = {
    service_key = var.pagerduty_service_key
  }
}

resource "google_monitoring_notification_channel" "slack" {
  display_name = "Slack - Security Alerts"
  type         = "slack"
  
  labels = {
    channel_name = "#security-alerts"
  }
}

resource "google_monitoring_notification_channel" "slack_ops" {
  display_name = "Slack - Operations Alerts"
  type         = "slack"
  
  labels = {
    channel_name = "#ops-alerts"
  }
}

resource "google_monitoring_notification_channel" "email" {
  display_name = "Security Team Email"
  type         = "email"
  
  labels = {
    email_address = "security-team@fluxocaixa.com"
  }
}

# Uptime Checks
resource "google_monitoring_uptime_check_config" "api_uptime" {
  display_name = "API Uptime Check"
  timeout      = "10s"
  period       = "60s"

  http_check {
    path         = "/health/live"
    port         = 8080
    request_method = "GET"
    validate_ssl = true
  }

  monitored_resource {
    type = "uptime_url"
    labels = {
      project_id = var.project_id
      host       = "api.fluxocaixa.com"
    }
  }

  content_matchers {
    content = "Healthy"
    matcher = "CONTAINS_STRING"
  }
}

# Service Monitoring (SLO)
resource "google_monitoring_slo" "api_availability" {
  service = google_monitoring_service.api_service.service_id
  slo_id  = "api-availability-slo"

  basic_sli {
    availability {
      enabled = true
    }
  }

  goal        = 0.999
  rolling_period_days = 30
}

resource "google_monitoring_slo" "api_latency" {
  service = google_monitoring_service.api_service.service_id
  slo_id  = "api-latency-slo"

  basic_sli {
    latency {
      threshold = "0.5s"
    }
  }

  goal        = 0.95
  rolling_period_days = 30
}

resource "google_monitoring_service" "api_service" {
  service_id = "fluxo-caixa-api"
  display_name = "Fluxo Caixa API"

  basic_service {
    service_labels = {
      service_name = "fluxo-caixa-api"
      namespace    = "fluxo-caixa"
    }
  }
}


```

# 13. Tabela Resumo de Métricas por Serviço
|Serviço|	Quantidade Métricas|	Quantidade Alertas|	Dashboard
|--|--|--|--
APIs (Auth,Lancamentos,Consolidacao,Relatorios)|	25|	8|	1
|Workers (4 workers)| 20|	6|	1
|Segurança|	18|	8|	1
|PostgreSQL|	12|	4|	1
|Redis|	10|	3|	1
|MongoDB|	10|	3|	1
|Kafka|	15|	4|	1
|GKE Infraestrutura|	10|	3|	1
|TOTAL|	120|	39|	8


# Canais de Notificação por Severidade
|Severidade|	Canais|	Tempo de Resposta
|--|--|--
|🔴 CRÍTICA|	PagerDuty+Slack+ SMS|	5 minutos
|🟠 ALTA|	PagerDuty+Slack|	15| minutos
|🟡 MÉDIA|	Slack+E-mail|	1| hora
|🟢 BAIXA|	Log only|	24| horas


# Ferramentas Utilizadas
|Ferramenta	Propósito|	Métricas|	Logs|	Traces|	Alertas
|--|--|--|--|--
|Cloud Monitoring|	Métricas e Alertas|	✅|	❌|	❌|	✅
|Cloud Logging|	Logs Centralizados|	✅|	✅|	❌|	✅
|Cloud Trace|	Distributed Tracing	❌|	❌|	✅|	❌
|Cloud Profiler|	Performance Analysis|	✅|	❌|	❌|	❌
|Error Reporting|	Error Aggregation|	✅|	✅|	✅|	✅
|Prometheus|	Métricas Customizadas|	✅|	❌|	❌|	✅
|OpenTelemetry|	Instrumentação|	✅|	✅|	✅|	❌












