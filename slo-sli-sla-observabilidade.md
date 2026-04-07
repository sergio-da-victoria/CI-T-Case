# Observabilidade com SLI & SLO & SLA - Ecossistema Fluxo de Caixa

### Sumário


[1. O que são SLI e SLO?](#sao_sli)\
[2. SLIs para o Ecossistema Fluxo de Caixa](#sli)\
[3. SLOs para o Ecossistema Fluxo de Caixa](#slo)\
[4. Implementação no Google Cloud Monitoring](#google)\
[5. Error Budget e Alertas](#error)\
[6. Dashboard de SLOs](#dasboard)\
[7. Políticas de Alerta Baseadas em SLO](#politica)

---
<a id="sao_sli"></a>
### 1. Observabilidade (O "Porquê")
Diferente do monitoramento clássico (que apenas avisa se o servidor caiu), a observabilidade em TI é sobre entender estados internos complexos. Em arquiteturas de **microserviços** e **nuvem (Cloud)**, é impossível prever todas as formas de falha.

*   **Na prática:** É o que permite a um desenvolvedor ou engenheiro de infraestrutura rastrear uma requisição que passou por 10 serviços diferentes e descobrir que ela falhou por causa de uma configuração de timeout no banco de dados.
*   **Pilares Técnicos:**
    *   **Métricas:** Dados agregados (ex: Prometheus).
    *   **Logs:** Eventos granulares (ex: ELK Stack, Splunk).
    *   **Traces:** Rastro da requisição (ex: Jaeger, Honeycomb).

---

### 2. SLI - Service Level Indicator (O "O quê")
O SLI é a métrica técnica específica que define o que é sucesso para o serviço. Na TI, não medimos tudo, medimos o que importa para a experiência do usuário.

*   **Exemplos Técnicos de SLIs:**
    *   **Disponibilidade:** Proporção de tempo que o serviço está up ou proporção de requisições com sucesso (status HTTP 200).
    *   **Latência:** Tempo que leva para processar uma requisição (ex: 95% das requisições devem ser < 200ms).
    *   **Vazão (Throughput):** Quantas requisições o sistema aguenta por segundo.
    *   **Erro:** Porcentagem de erros (status 5xx) em relação ao total.

---

### 3. SLO - Service Level Objective (O "Quanto")
O SLO é a meta numérica para o SLI. É o "ponto de equilíbrio" entre a velocidade de desenvolvimento e a estabilidade do sistema. **Em TI, o SLO é uma meta interna.**

*   **Por que não 100%?** Em TI, buscar 100% de disponibilidade é proibitivamente caro e impede a inovação. Se você quer 100% de uptime, você não pode nunca atualizar o software.
*   **Exemplo de SLO:** "O serviço de login deve ter 99,9% de sucesso (SLI) ao longo de um mês."
*   **Error Budget (Orçamento de Erro):** Se o seu SLO é 99,9%, você tem 0,1% de "permissão" para falhar. Esse tempo pode ser usado para lançar atualizações arriscadas. Se o orçamento acaba, o time para de lançar features e foca em correções de bugs.

---

### 4. SLA - Service Level Agreement (O "E se...")
O SLA é o contrato formal entre a empresa de TI (provedor) e o cliente final. Ele é redigido por advogados e gestores de conta com base nos SLOs, mas geralmente é **menos rigoroso** que o SLO.

*   **Exemplo:**
    *   **SLO (Interno):** 99,9% (O time de engenharia se esforça para isso).
    *   **SLA (Contrato):** 99,5% (Se cair abaixo disso, a empresa paga multa ou devolve créditos).
*   **A lógica:** Você define um SLO interno mais alto para ter uma "margem de segurança" antes de quebrar o SLA e sofrer prejuízos financeiros.

---

### Resumo Visual para TI

| Conceito | Pergunta-chave | Exemplo de TI |
| :--- | :--- | :--- |
| **Observabilidade** | O que está acontecendo lá dentro? | "O trace mostra que o gargalo é na API de frete." |
| **SLI** | O que estamos medindo? | "Taxa de erro das requisições HTTP." |
| **SLO** | Qual a nossa meta para a equipe? | "Menos de 0,5% de erro por semana." |
| **SLA** | Qual o limite antes da multa? | "Disponibilidade mínima de 99% em contrato." |

---

### Os "Quatro Sinais de Ouro" (Golden Signals)
Se você trabalha com TI, ao definir seus SLIs e SLOs, você deve sempre olhar para estes 4 pontos:

1.  **Latência:** O tempo que leva para processar um pedido.
2.  **Tráfego:** A demanda colocada sobre o sistema (ex: requisições HTTP/seg).
3.  **Erros:** A taxa de requisições que falham explicitamente ou implicitamente.
4.  **Saturação:** O quão "cheio" está o seu serviço (ex: uso de memória ou CPU em 90%).

### Conclusão
Para um profissional de TI, a **Observabilidade** fornece os dados, o **SLI** seleciona o dado importante, o **SLO** define a meta de qualidade para o time técnico, e o **SLA** protege o negócio legalmente caso as falhas ultrapassem o limite aceitável.


# 1.1 Definições

```

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SLI e SLO - Conceitos Fundamentais                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                              SLI (Service Level Indicator)                  │    │
│  │                                                                             │    │
│  │  • É uma MÉTRICA que mede a qualidade do serviço                            │    │
│  │  • Responde à pergunta: "Quão bem estamos atendendo?"                       │    │
│  │  • Exemplos: Latência, Disponibilidade, Taxa de erro                        │    │
│  │  • Fórmula: Eventos bons / Eventos totais                                   │    │
│  │                                                                             │    │
│  │  Exemplo: Disponibilidade = Requisições com status 200 / Total requisições  │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                              SLO (Service Level Objective)                  │    │
│  │                                                                             │    │
│  │  • É uma META para o SLI                                                    │    │
│  │  • Responde à pergunta: "Qual é o nível de serviço que prometemos?"         │    │
│  │  • Exemplos: 99.9% de disponibilidade, 95% das requisições < 500ms          │    │
│  │  • Define o limite aceitável para o SLI                                     │    │
│  │                                                                             │    │
│  │  Exemplo: 99.9% de disponibilidade nos últimos 30 dias                      │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                          SLA (Service Level Agreement)                      │    │
│  │                                                                             │    │
│  │  • É um CONTRATO LEGAL com o cliente                                        │    │
│  │  • Consequências financeiras se não cumprido                                │    │
│  │  • Baseado nos SLOs internos (geralmente mais rigoroso)                     │    │
│  │                                                                             │    │
│  │  Exemplo: Se disponibilidade < 99.9%, crédito de 10% na mensalidade         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Relação entre SLI, SLO e SLA

```

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    RELAÇÃO SLI → SLO → SLA                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│                                                                                     │
│     ┌─────────────┐         ┌─────────────┐         ┌─────────────┐                 │
│     │    SLI      │─────────│    SLO      │───────▶│   SLA       │                 │
│     │             │         │             │         │             │                 │
│     │ "O que      │         │ "O que      │         │ "O que      │                 │
│     │  estamos    │         │ prometemos" │         │ contratamos"│                 │
│     │ entregando?"│         │             │         │             │                 │
│     └─────────────┘         └─────────────┘         └─────────────┘                 │
│           │                        │                        │                       │
│           ▼                        ▼                        ▼                       │
│     ┌─────────────┐         ┌─────────────┐         ┌─────────────┐                 │
│     │ Métrica     │         │ Meta        │         │ Acordo      │                 │
│     │ Interna     │         │ Interna     │         │ Legal       │                 │
│     │(time series)│         │ (99.9%)     │         │+ Compensação│                 │
│     └─────────────┘         └─────────────┘         └─────────────┘                 │
│                                                                                     │
│                                                                                     │
│  Exemplo Prático:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │ SLI:  =   (Requisições com status 200) / (Total de requisições)             │    │
│  │ SLO:  =   99.9% de disponibilidade nos últimos 30 dias                      │    │
│  │ SLA:  =   99.5% de disponibilidade (com compensação financeira)             │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Error Budget (Orçamento de Erros)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         ERROR BUDGET                                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  O Error Budget é o quanto o serviço PODE falhar dentro do SLO definido.            │
│                                                                                     │
│  Fórmula: Error Budget = 1 - SLO                                                    │
│                                                                                     │
│  Exemplo:                                                                           │
│  • SLO de disponibilidade = 99.9%                                                   │
│  • Error Budget = 0.1% (ou 43 minutos por mês)                                      │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                                                                             │    │
│  │  Uso do Error Budget:                                                       │    │
│  │                                                                             │    │
│  │  ✅ Budget disponível → Pode lançar novas features (risco controlado)       │    │
│  │  ⚠️ Budget consumido 80% → Congelar lançamentos (foco em estabilidade)      │    │
│  │  🔴 Budget esgotado → Parar deploys (apenas correções críticas)             │    │
│  │                                                                             │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

```

<a id="sli"></a>
# 2. SLIs para o Ecossistema Fluxo de Caixa

### 2.1 SLIs de Disponibilidade
|SLI|	Descrição|	Métrica|	Fórmula
|--|--|--|--
|Disponibilidade da API|	APIs estão respondendo?|	api_requests_total|	(requests - errors_5xx) / requests
Disponibilidade do Worker|	Workers estão processando?|	worker_uptime_seconds|	worker_uptime > 0
Disponibilidade do Banco|	Banco está acessível?|	cloudsql_database_up|	database_up = 1
Disponibilidade do Kafka|	Kafka está acessível?|	kafka_cluster_up|	cluster_up = 1

### 2.2 SLIs de Latência
|SLI|	Descrição|	Métrica|	Fórmula
|--|--|--|--
Latência da API (P95)|	95% das requisições|	api_request_duration_seconds|	< 500ms
Latência da API (P99)|	99% das requisições|	api_request_duration_seconds|	< 1000ms
Latência do Worker|	Processamento de mensagens|	worker_processing_duration_seconds|	< 100ms
Latência do Banco|	Consultas SQL|	db_query_duration_seconds|	< 50ms
Latência do Cache|	Acessos ao Redis|	redis_command_duration_seconds|	< 10ms
Latência do Kafka|	Publicação de mensagens|	kafka_produce_latency_seconds|	< 50ms

### 2.3 SLIs de Qualidade
|SLI|	Descrição|	Métrica|	Fórmula
|--|--|--|--
|Taxa de Erro da API|	Requisições com erro	|api_errors_total / api_requests_total|	< 0.1%
|Taxa de Sucesso do Worker|	Mensagens processadas|	worker_messages_processed / worker_messages_consumed|	> 99.9%
|Freshness dos Dados|	Atraso do Read Database|	replica_lag_seconds|	< 10s
|Cache Hit Ratio|	Eficiência do cache|	cache_hits / (cache_hits + cache_misses)|	> 90%

# 3. SLOs para o Ecossistema Fluxo de Caixa

<a id="slo"></a>
### 3.1 SLOs por Serviço
|Serviço|	SLO	|Descrição|	Janela|	Error Budget
|--|--|--|--|--
|API Gateway|	99.99%|	Disponibilidade do gateway|	30 dias|	4.32 minutos/mês
|Auth API|	99.95%|	Disponibilidade da autenticação|	30 dias|	21.6 minutos/mês
|Lancamentos API|	99.95%|	Disponibilidade do registro|	30 dias|	21.6 minutos/mês
|Consolidacao API|	99.99%|	Disponibilidade da consulta|	30 dias|	4.32 minutos/mês
|Relatorios API|	99.90%|	Disponibilidade dos relatórios|	30 dias|	43.2 minutos/mês
|Workers|	99.95%|	Disponibilidade dos workers|	30 dias|	21.6 minutos/mês
|PostgreSQL|	99.95%|	Disponibilidade do banco|	30 dias|	21.6 minutos/mês
|Redis|	99.99%|	Disponibilidade do cache|	30 dias	|4.32 minutos/mês
|Kafka|	99.99%|	Disponibilidade da mensageria|	30 dias|	4.32 minutos/mês


### 3.2 SLOs de Performance
|Serviço|	SLI|	SLO|	Descrição
|--|--|--|--
|API Gateway|	Latência P95|	< 100ms	|95% das requisições
|Auth API|	Latência P95|	< 200ms|	Login/registro
|Lancamentos API|	Latência P95|	< 150ms|	Registro de lançamento
|Consolidacao API|	Latência P95|	< 100ms|	Consulta de saldo (cache)
|Consolidacao API|	Latência P95|	< 300ms|	Consulta de saldo (DB)
|Workers|	Processamento P95|	< 100ms|	Processamento de mensagem
|PostgreSQL|	Query P95|	< 50ms|	Consultas otimizadas
|Redis|	Command P95|	< 10ms|	Operações de cache


### 3.3 SLOs de Qualidade
|Serviço|	SLI|	SLO|	Descrição
|--|--|--|--
|APIs|	Taxa de erro|	< 0.1%|	Erros 5xx
|Workers|	Taxa de sucesso|	> 99.9%|	Mensagens processadas
|Kafka|	Consumer lag|	< 1000|	Atraso do consumidor
|PostgreSQL|	Replica lag|	< 10s|	Atraso da réplica
|Redis|	Cache hit ratio|	> 90%|	Eficiência do cache


<a id="google"></a>
# 4. Implementação no Google Cloud Monitoring
### 4.1 Definindo SLIs no Terraform

```
hcl


# sli-definitions.tf

# SLI de Disponibilidade da API
resource "google_monitoring_slo" "api_availability_slo" {
  service = google_monitoring_service.api_service.service_id
  slo_id  = "api-availability-slo"

  basic_sli {
    availability {
      enabled = true
    }
  }

  goal        = 0.9995  # 99.95%
  rolling_period_days = 30
  
  calendar_period = "DAY"
}

# SLI de Latência da API (P95)
resource "google_monitoring_slo" "api_latency_slo" {
  service = google_monitoring_service.api_service.service_id
  slo_id  = "api-latency-slo"

  basic_sli {
    latency {
      threshold = "0.5s"  # 500ms
    }
  }

  goal        = 0.95  # 95% das requisições
  rolling_period_days = 30
}

# SLI de Taxa de Erro da API
resource "google_monitoring_slo" "api_error_rate_slo" {
  service = google_monitoring_service.api_service.service_id
  slo_id  = "api-error-rate-slo"

  basic_sli {
    availability {
      enabled = true
    }
  }

  goal        = 0.999  # 99.9%
  rolling_period_days = 30
}

# SLI de Disponibilidade do Worker
resource "google_monitoring_slo" "worker_availability_slo" {
  service = google_monitoring_service.worker_service.service_id
  slo_id  = "worker-availability-slo"

  basic_sli {
    availability {
      enabled = true
    }
  }

  goal        = 0.9995
  rolling_period_days = 30
}

# SLI de Processamento do Worker
resource "google_monitoring_slo" "worker_processing_slo" {
  service = google_monitoring_service.worker_service.service_id
  slo_id  = "worker-processing-slo"

  basic_sli {
    latency {
      threshold = "0.1s"  # 100ms
    }
  }

  goal        = 0.95
  rolling_period_days = 30
}

# SLI de Disponibilidade do Banco de Dados
resource "google_monitoring_slo" "database_availability_slo" {
  service = google_monitoring_service.database_service.service_id
  slo_id  = "database-availability-slo"

  basic_sli {
    availability {
      enabled = true
    }
  }

  goal        = 0.9995
  rolling_period_days = 30
}

# SLI de Latência do Banco de Dados
resource "google_monitoring_slo" "database_latency_slo" {
  service = google_monitoring_service.database_service.service_id
  slo_id  = "database-latency-slo"

  basic_sli {
    latency {
      threshold = "0.05s"  # 50ms
    }
  }

  goal        = 0.95
  rolling_period_days = 30
}

# SLI de Disponibilidade do Kafka
resource "google_monitoring_slo" "kafka_availability_slo" {
  service = google_monitoring_service.kafka_service.service_id
  slo_id  = "kafka-availability-slo"

  basic_sli {
    availability {
      enabled = true
    }
  }

  goal        = 0.9999
  rolling_period_days = 30
}

# SLI de Consumer Lag do Kafka
resource "google_monitoring_slo" "kafka_lag_slo" {
  service = google_monitoring_service.kafka_service.service_id
  slo_id  = "kafka-lag-slo"

  basic_sli {
    latency {
      threshold = "1000"  # 1000 mensagens
    }
  }

  goal        = 0.95
  rolling_period_days = 30
}

# SLI de Cache Hit Ratio do Redis
resource "google_monitoring_slo" "redis_cache_slo" {
  service = google_monitoring_service.redis_service.service_id
  slo_id  = "redis-cache-slo"

  basic_sli {
    performance {
      threshold = "0.9"  # 90%
    }
  }

  goal        = 0.95
  rolling_period_days = 30
}
```
# 4.2 Registrando os Serviços no Cloud Monitoring

```

hcl

# services.tf

# API Service
resource "google_monitoring_service" "api_service" {
  service_id = "fluxo-caixa-api"
  display_name = "Fluxo Caixa API"

  basic_service {
    service_labels = {
      service_name = "fluxo-caixa-api"
      namespace    = "fluxo-caixa"
      environment  = var.environment
    }
  }

  user_labels = {
    team = "backend"
    criticality = "high"
  }
}

# Worker Service
resource "google_monitoring_service" "worker_service" {
  service_id = "fluxo-caixa-workers"
  display_name = "Fluxo Caixa Workers"

  basic_service {
    service_labels = {
      service_name = "fluxo-caixa-workers"
      namespace    = "fluxo-caixa"
      environment  = var.environment
    }
  }
}

# Database Service
resource "google_monitoring_service" "database_service" {
  service_id = "fluxo-caixa-database"
  display_name = "Fluxo Caixa Database"

  basic_service {
    service_labels = {
      service_name = "cloud-sql"
      environment  = var.environment
    }
  }
}

# Kafka Service
resource "google_monitoring_service" "kafka_service" {
  service_id = "fluxo-caixa-kafka"
  display_name = "Fluxo Caixa Kafka"

  basic_service {
    service_labels = {
      service_name = "confluent-kafka"
      environment  = var.environment
    }
  }
}

# Redis Service
resource "google_monitoring_service" "redis_service" {
  service_id = "fluxo-caixa-redis"
  display_name = "Fluxo Caixa Redis"

  basic_service {
    service_labels = {
      service_name = "memorystore-redis"
      environment  = var.environment
    }
  }
}

```
<a id="error"></a>
# 5. Error Budget e Alertas

### 5.1 Error Budget Burn Rate Alerts


```

hcl


# error-budget-alerts.tf

# Alerta de consumo rápido do Error Budget (Burn Rate)
resource "google_monitoring_alert_policy" "slo_burn_rate_fast" {
  display_name = "SLO - High Burn Rate (Fast)"
  combiner     = "OR"
  
  conditions {
    display_name = "Error budget burn rate > 10 in 1 hour"
    condition_threshold {
      filter     = "select_slo_health('projects/${var.project_id}/services/fluxo-caixa-api/serviceLevelObjectives/api-availability-slo')"
      duration   = "3600s"
      comparison = "COMPARISON_LT"
      threshold_value = 0.9  # 90% do SLO consumido em 1 hora
      
      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }
  
  notification_channels = [
    google_monitoring_notification_channel.pagerduty.id,
    google_monitoring_notification_channel.slack.id
  ]
}

# Alerta de consumo médio do Error Budget
resource "google_monitoring_alert_policy" "slo_burn_rate_slow" {
  display_name = "SLO - High Burn Rate (Slow)"
  combiner     = "OR"
  
  conditions {
    display_name = "Error budget burn rate > 2 in 6 hours"
    condition_threshold {
      filter     = "select_slo_health('projects/${var.project_id}/services/fluxo-caixa-api/serviceLevelObjectives/api-availability-slo')"
      duration   = "21600s"
      comparison = "COMPARISON_LT"
      threshold_value = 0.5  # 50% do SLO consumido em 6 horas
      
      aggregations {
        alignment_period   = "300s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }
  
  notification_channels = [
    google_monitoring_notification_channel.slack.id
  ]
}

# Alerta de SLO sendo violado
resource "google_monitoring_alert_policy" "slo_violation" {
  display_name = "SLO - Being Violated"
  combiner     = "OR"
  
  conditions {
    display_name = "SLO being violated"
    condition_threshold {
      filter     = "select_slo_health('projects/${var.project_id}/services/fluxo-caixa-api/serviceLevelObjectives/api-availability-slo')"
      duration   = "300s"
      comparison = "COMPARISON_LT"
      threshold_value = 0.99
      
      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }
  
  notification_channels = [
    google_monitoring_notification_channel.pagerduty.id,
    google_monitoring_notification_channel.slack.id
  ]
}


```

# 5.2 SLO Dashboard com Error Budget

```

hcl

resource "google_monitoring_dashboard" "slo_dashboard" {
  dashboard_json = <<EOF
{
  "displayName": "Fluxo Caixa - SLO Dashboard",
  "gridLayout": {
    "columns": 2,
    "widgets": [
      {
        "title": "🎯 API Availability SLO",
        "scorecard": {
          "dataSets": [
            {
              "timeSeriesQuery": "select_slo_health('projects/${var.project_id}/services/fluxo-caixa-api/serviceLevelObjectives/api-availability-slo')"
            }
          ],
          "thresholds": [
            {"label": "OK", "value": 0.999, "color": "GREEN"},
            {"label": "WARNING", "value": 0.99, "color": "YELLOW"},
            {"label": "CRITICAL", "value": 0.95, "color": "RED"}
          ]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "📊 API Latency SLO (P95 < 500ms)",
        "scorecard": {
          "dataSets": [
            {
              "timeSeriesQuery": "select_slo_health('projects/${var.project_id}/services/fluxo-caixa-api/serviceLevelObjectives/api-latency-slo')"
            }
          ]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "💰 Error Budget Remaining",
        "scorecard": {
          "dataSets": [
            {
              "timeSeriesQuery": "select_slo_compliance('projects/${var.project_id}/services/fluxo-caixa-api/serviceLevelObjectives/api-availability-slo')"
            }
          ]
        },
        "height": 4,
        "width": 1
      },
      {
        "title": "⚠️ Error Budget Burn Rate",
        "xyChart": {
          "dataSets": [
            {
              "timeSeriesQuery": "select_slo_health('projects/${var.project_id}/services/fluxo-caixa-api/serviceLevelObjectives/api-availability-slo') | rate(1h)"
            }
          ]
        },
        "height": 4,
        "width": 1
      }
    ]
  }
}
EOF
}

```

<a id="dasboard"></a> 
# 6. Dashboard de SLOs

### 6.1 Dashboard Completo de SLOs

```

json

{
  "dashboard": {
    "displayName": "Fluxo Caixa - SLO Dashboard",
    "gridLayout": {
      "widgets": [
        {
          "title": "API Availability SLO (99.95%)",
          "scorecard": {
            "metric": "slo/availability",
            "thresholds": [
              {"value": 99.95, "color": "GREEN"},
              {"value": 99.9, "color": "YELLOW"},
              {"value": 99.5, "color": "RED"}
            ]
          }
        },
        {
          "title": "API Latency SLO (P95 < 500ms)",
          "scorecard": {
            "metric": "slo/latency",
            "thresholds": [
              {"value": 95, "color": "GREEN"},
              {"value": 90, "color": "YELLOW"},
              {"value": 80, "color": "RED"}
            ]
          }
        },
        {
          "title": "Worker Processing SLO (P95 < 100ms)",
          "scorecard": {
            "metric": "slo/worker_processing"
          }
        },
        {
          "title": "Database Latency SLO (P95 < 50ms)",
          "scorecard": {
            "metric": "slo/database_latency"
          }
        },
        {
          "title": "Kafka Availability SLO (99.99%)",
          "scorecard": {
            "metric": "slo/kafka_availability"
          }
        },
        {
          "title": "Redis Cache Hit SLO (> 90%)",
          "scorecard": {
            "metric": "slo/redis_cache_hit"
          }
        }
      ]
    }
  }
}

```

<a id="politica"></a> 
# 7. Políticas de Alerta Baseadas em SLO

### 7.1 Matriz de Decisão por Severidade
|Severidade|	Condição|	Ação
|--|--|--
|🔴 Crítica|	Error Budget < 10% (última hora)|	Congelar deploys, time de plantão acionado
|🟠 Alta|	Error Budget < 30% (últimas 6h)|	Revisão de mudanças, monitoramento intensificado
|🟡 Média|	Error Budget < 50% (últimas 24h)|	Análise de causas, otimização
|🟢 Baixa|	Error Budget > 80%|	Operação normal, pode lançar features

# 7.2 Workflow de Resposta

```
yaml

# slo-response-workflow.yaml
workflow:
  - trigger: error_budget_burn_rate > 10
    severity: CRITICAL
    actions:
      - block_deploys: true
      - oncall_engineer: paged
      - incident_created: true
      - rollback_last_deploy: if_available
  
  - trigger: error_budget_burn_rate > 5
    severity: HIGH
    actions:
      - block_deploys: true
      - team_notification: slack
      - performance_review: started
  
  - trigger: error_budget_burn_rate > 2
    severity: MEDIUM
    actions:
      - team_notification: slack
      - investigation: scheduled
  
  - trigger: slo_violation_projected
    severity: WARNING
    actions:
      - log_analysis: started
      - capacity_review: if_needed

```
# 8. Resumo dos SLOs do Ecossistema      


|Serviço|	SLO Disponibilidade|	SLO Latência|	SLO Qualidade
|--|--|--|--
|API Gateway|	99.99%|	P95 < 100ms|	Erros < 0.1%
|Auth API|	99.95%|	P95 < 200ms|	Erros < 0.1%
|Lancamentos API|	99.95%|	P95 < 150ms|	Erros < 0.1%
|Consolidacao API|	99.99%|	P95 < 100ms| (cache)	Erros < 0.1%
|Workers|	99.95%|	P95 < 100ms|	Sucesso > 99.9%
|PostgreSQL|	99.95%|	P95 < 50ms|	Replica lag < 10s
|Redis|	99.99%|	P95 < 10ms| 	Hit ratio > 90%
|Kafka|	99.99%|	P95 < 50ms|	Consumer lag < 1000

### Error Budget por Serviço (30 dias)

|Serviço|	SLO|	Error Budget|	Minutos de Falha/mês
|--|--|--|--
|API Gateway|	99.99%|	0.01%| 4.32 min
|Auth API|	99.95%|	0.05%|	21.6 min
|Lancamentos API|	99.95%|	0.05%|	21.6 min
|Consolidacao API|	99.99%|	0.01%|	4.32 min
|Workers|	99.95%|	0.05%|	21.6 min
|PostgreSQL|	99.95%|	0.05%|	21.6 min
|Redis|	99.99%|	0.01%|	4.32 min
|Kafka|	99.99%|	0.01%|	4.32 min




