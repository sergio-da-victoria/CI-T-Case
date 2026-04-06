# Camada de Segurança - Ecossistema Fluxo de Caixa (Detalhamento Completo)

### Sumário

[1. Visão Geral da Arquitetura de Segurança](#visao)\
[2. Modelo de Responsabilidade Compartilhada](#model)\
[3. Camada 1: Segurança de Perímetro (Edge Security)](#ed)\
[4. Camada 2: Segurança de Rede (Network Security)](#netw)\
[5. Camada 3: Segurança de Identidade e Acesso (IAM)](#iam)\
[6. Camada 4: Segurança de Aplicação (App Security)](#app)\
[7. Camada 5: Segurança de Dados (Data Security)](#dados)\
[8. Camada 6: Monitoramento e Resposta a Incidentes](#incidentes)\
[9. Matriz de Responsabilidades por Camada](#matriz)\
[10. Checklist de Implementação](#implementacao)


# 1. Visão Geral da Arquitetura de Segurança
### __A camada de segurança do ecossistema Fluxo de Caixa foi projetada seguindo o princípio de Defesa em Profundidade (Defense in Depth), onde múltiplas camadas de controle protegem o sistema contra falhas ou vulnerabilidades em qualquer camada individual.__

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ESTRATÉGIA DE SEGURANÇA EM CAMADAS - FLUXO DE CAIXA              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        CAMADA 1: SEGURANÇA DE PERÍMETRO                     │    │
│  │  • WAF (Cloud Armor)  • DDoS Protection  • Rate Limiting  • IP Allowlist    │    │
│  │  🔔 Alertas: Tentativas de ataque, Rate Limit excedido, IP bloqueado        │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        CAMADA 2: SEGURANÇA DE REDE                          │    │
│  │  • VPC Firewall Rules  • Private Subnets  • Cloud NAT  • VPC Peering        │    │
│  │  • Service Controls  • Access Context Manager  • VPC Service Perimeter      │    │
│  │  🔔 Alertas: Tráfego não autorizado, Port scanning, VPC peering alterado    │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        CAMADA 3: SEGURANÇA DE IDENTIDADE                    │    │
│  │  • Cloud Identity  • IAM  • Workload Identity  • MFA  • SSO                 │    │
│  │  • Conditional Access  • Zero Trust  • Just-In-Time Access                  │    │
│  │  🔔 Alertas: Login suspeito, MFA falhou, Privilégio elevado, Acesso anômalo │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        CAMADA 4: SEGURANÇA DE APLICAÇÃO                     │    │
│  │  • API Security  • JWT Validation  • Input Validation  • CSP                │    │
│  │  • Secret Scanning  • SAST/DAST  • Dependency Scanning                      │    │
│  │  🔔 Alertas: SQL Injection, XSS, JWT inválido, Vulnerabilidade detectada    │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        CAMADA 5: SEGURANÇA DE DADOS                         │    │
│  │  • Encryption at Rest  • Encryption in Transit  • Key Management (KMS)      │    │
│  │  • Data Masking  • Tokenization  • Backup Encryption                        │    │
│  │  🔔 Alertas: Acesso não autorizado a dados, Chave comprometida, Exfiltração │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                    CAMADA 6: MONITORAMENTO E RESPOSTA (SIEM/SOAR)           │    │
│  │  • SIEM (Chronicle)  • SOAR  • Cloud Logging  • Security Command Center     │    │
│  │  • Threat Intelligence  • Anomaly Detection  • Incident Response            │    │
│  │  🔔 Alertas: Correlação de eventos, Comportamento anômalo, IOC detectado    │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ARQUITETURA DE SEGURANÇA - 6 CAMADAS                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  CAMADA 1: SEGURANÇA DE PERÍMETRO (Edge Security)                           │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │    │
│  │  │ Cloud Armor │  │  Cloud CDN  │  │ Rate Limit  │  │  DDoS       │         │    │
│  │  │   (WAF)     │  │             │  │             │  │ Protection  │         │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                          │
│                                          ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  CAMADA 2: SEGURANÇA DE REDE (Network Security)                             │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │    │
│  │  │ VPC Network │  │  Firewall   │  │  Cloud NAT  │  │   VPC       │         │    │
│  │  │  Peering    │  │   Rules     │  │             │  │  Service    │         │    │
│  │  │             │  │             │  │             │  │  Controls   │         │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                          │
│                                          ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  CAMADA 3: SEGURANÇA DE IDENTIDADE E ACESSO (IAM)                           │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │    │
│  │  │ Cloud       │  │  Workload   │  │  IAM Roles  │  │  MFA / SSO  │         │    │
│  │  │ Identity    │  │  Identity   │  │  & Policies │  │             │         │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                          │
│                                          ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  CAMADA 4: SEGURANÇA DE APLICAÇÃO (Application Security)                    │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │    │
│  │  │ API Gateway │  │  JWT/SAML   │  │  SAST/DAST  │  │  Container  │         │    │
│  │  │  Security   │  │  Validation │  │             │  │  Scanning   │         │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                          │
│                                          ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  CAMADA 5: SEGURANÇA DE DADOS (Data Security)                               │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │    │
│  │  │ Encryption  │  │  KMS Key    │  │  Data       │  │  Backup     │         │    │
│  │  │ at Rest     │  │  Management │  │  Masking    │  │  Encryption │         │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                          │
│                                          ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  CAMADA 6: MONITORAMENTO E RESPOSTA (SIEM/SOAR)                             │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │    │
│  │  │ Security    │  │  Chronicle  │  │  Cloud      │  │  Incident   │         │    │
│  │  │ Command     │  │  SIEM       │  │  Logging    │  │  Response   │         │    │
│  │  │ Center      │  │             │  │             │  │             │         │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 2. Modelo de Responsabilidade Compartilhada
|Camada|	Responsabilidade Google Cloud|	Responsabilidade Cliente (Fluxo Caixa)
|--|--|--
|Perímetro| Infraestrutura de rede global, DDoS protection base|	Configuração WAF, Regras customizadas, Rate limiting
|Rede|	VPC base, Cloud NAT, Load balancing|	Firewall rules, Network policies, VPC peering
|Identidade|	Cloud Identity base, IAM infrastructure|	MFA enforcement, Role definitions, JWT validation
|Aplicação|	N/A (Plataforma)|	API security, SAST/DAST, Secret management
|Dados|	Encryption at rest (base), KMS infrastructure|	Key rotation, Data masking, Backup encryption
|Monitoramento|	Logging infrastructure, SCC base|	Alert configuration, Incident response plan


### 3. Camada 1: Segurança de Perímetro (Edge Security)

__3.1 WAF (Cloud Armor) - Detalhamento Completo__
__O Google Cloud Armor é o Web Application Firewall (WAF) que protege o ecossistema contra ataques na camada de aplicação (OWASP Top 10).__

### __Regras| Predefinidas (OWASP Top 10)__

|Regra|	Descrição|	Ação|	Severidade
|--|--|--|--
|owasp-crs-v030001-id942100-sqli|	SQL Injection Attack|	DENY|	🔴 CRÍTICA
|owasp-crs-v030001-id941100-xss|	Cross-Site Scripting (XSS)|	DENY|	🔴 CRÍTICA
|owasp-crs-v030001-id931100-rce|	Remote Code Execution (RCE)|	DENY|	🔴 CRÍTICA
|owasp-crs-v030001-id933100-php-injection|	PHP Injection|	DENY|	🟠 ALTA
|owasp-crs-v030001-id943100-session-fixation|	Session Fixation|	DENY|	🟠 ALTA
|owasp-crs-v030001-id944100-java-injection|	Java Injection|	DENY|	🟠 ALTA
|owasp-crs-v030001-id920100-method-enforcement|	HTTP Method Enforcement|	ALLOW|	🟡 MÉDIA


__Regras Customizadas para o Ecossistema__

# Regras customizadas do Cloud Armor

```
custom_rules:
  - name: "block-known-malicious-ips"
    description: "Bloquear IPs maliciosos conhecidos"
    priority: 1000
    action: DENY
    match:
      src_ip_ranges:
        - "192.0.2.0/24"
        - "198.51.100.0/24"

  - name: "rate-limit-login"
    description: "Limitar tentativas de login por IP"
    priority: 2000
    action: RATE_BASED_BLOCK
    rate_limit_options:
      rate_limit_threshold: 10
      rate_limit_window_sec: 60
      conform_action: ALLOW
      exceed_action: DENY
    match:
      expr:
        expression: "request.path.startsWith('/api/auth/login')"

  - name: "geo-blocking"
    description: "Bloquear tráfego de países não autorizados"
    priority: 3000
    action: DENY
    match:
      geo_match:
        region_codes:
          - "XX" # Códigos de países bloqueados

  - name: "allow-health-checks"
    description: "Permitir health checks"
    priority: 10
    action: ALLOW
    match:
      expr:
        expression: "request.headers['User-Agent'].contains('GoogleHC')"

  - name: "block-api-abuse"
    description: "Bloquear padrões de abuso de API"
    priority: 2500
    action: DENY
    match:
      expr:
        expression: "request.path.matches('/api/.*') && request.method == 'POST' && size(request.body) > 1000000"
```

__Configuração Terraform do Cloud Armor__

```
# Cloud Armor Security Policy
resource "google_compute_security_policy" "waf_policy" {
  name        = "fluxo-caixa-waf-policy"
  description = "WAF policy for Fluxo Caixa"

  # OWASP Predefined Rules
  rule {
    action   = "deny(403)"
    priority = 1000
    match {
      expr {
        expression = "evaluatePreconfiguredWaf('xss-v33-stable')"
      }
    }
    description = "XSS detection"
  }

  rule {
    action   = "deny(403)"
    priority = 1100
    match {
      expr {
        expression = "evaluatePreconfiguredWaf('sqli-v33-stable')"
      }
    }
    description = "SQL Injection detection"
  }

  rule {
    action   = "deny(403)"
    priority = 1200
    match {
      expr {
        expression = "evaluatePreconfiguredWaf('rce-v33-stable')"
      }
    }
    description = "RCE detection"
  }

  # Rate Limiting for Login
  rule {
    action   = "rate_based_ban"
    priority = 2000
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    rate_limit_options {
      rate_limit_threshold = 10
      rate_limit_window_sec = 60
      conform_action = "allow"
      exceed_action = "deny(429)"
      enforce_on_key = "IP"
    }
    description = "Rate limiting for all requests"
  }

  # Geo-blocking (optional)
  rule {
    action   = "deny(403)"
    priority = 3000
    match {
      expr {
        expression = "origin.region_code in ['XX', 'YY']"
      }
    }
    description = "Block traffic from unauthorized countries"
  }

  # Default allow rule
  rule {
    action   = "allow"
    priority = 2147483647
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    description = "Default allow rule"
  }
}
```
### 3.2 DDoS Protection

|Camada|	Proteção|	Descrição
|--|--|--
|Network DDoS|	Automática|	Proteção contra ataques volumétricos (UDP flood, ICMP flood)
|Protocol DDoS|	Automática|	Proteção contra SYN flood, ACK flood
|Application DDoS|	Configurável|	Proteção contra HTTP flood via Cloud Armor
|Advanced Protection|	Cloud Armor Managed|	Proteção contra ataques complexos (L7 DDoS)


### 3.3 Rate Limiting


```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         RATE LIMITING EM MÚLTIPLAS CAMADAS                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  CAMADA 1: Cloud Armor (Edge)                                                       │
│  • 100 req/min por IP                                                               │
│  • Ação: Bloquear com HTTP 429                                                      │
│                                                                                     │
│  CAMADA 2: API Gateway (Application)                                                │
│  • 1000 req/min por usuário (JWT)                                                   │
│  • Ação: Retornar HTTP 429 com Retry-After                                          │
│                                                                                     │
│  CAMADA 3: Aplicação (Service)                                                      │
│  • 5000 req/min por endpoint                                                        │
│  • Ação: Queue ou degrade response                                                  │
│                                                                                     │
│  CAMADA 4: Banco de Dados                                                           │
│  • 1000 conexões simultâneas                                                        │
│  • Ação: Connection pooling + timeout                                               │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

```

### 3.4 Alertas da Camada de Perímetro
|Alerta|	Severidade|	Gatilho	Ação|	Canal
|--|--|--|--
|SQL Injection Detectado|	🔴 CRÍTICA|	WAF bloqueia SQLi|	Isolar IP + Notificar|	PagerDuty + Slack
|XSS Detectado|	🔴 CRÍTICA|	WAF bloqueia XSS|	Isolar IP + Notificar|	PagerDuty + Slack
|Rate Limit Excedido|	🟠 ALTA|	> 100 req/min por IP|	Alertar + Bloquear temporariamente|	Slack
|DDoS Attack Detectado|	🔴 CRÍTICA|	Tráfego > 3x normal|	Ativar proteção avançada|	PagerDuty
|IP Malicioso Bloqueado|	🟡 MÉDIA|	IP em blacklist|	Log + Monitorar|	Slack
|Regra WAF Modificada|	🟡 MÉDIA|	Mudança em regras|	Revisar alteração|	E-mail
|Geo-blocking Acionado|	🟡 MÉDIA|	Tráfego de país bloqueado|	Log + Monitorar|	Slack


### 4. Camada 2: Segurança de Rede (Network Security)

### 4.1 VPC Architecture

### __A arquitetura de VPCs segmentadas isola diferentes camadas da aplicação:__

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         ARQUITETURA DE VPCS SEGMENTADAS                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                    VPC PÚBLICA (DMZ) - 10.0.0.0/20                          │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │   │
│  │  │  Subnet: public-lb (10.0.0.0/24)                                    │    │   │
│  │  │  • Cloud Load Balancer  • Cloud CDN  • Cloud Armor (WAF)            │    │   │
│  │  └─────────────────────────────────────────────────────────────────────┘    │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │   │
│  │  │  Subnet: public-gateway (10.0.1.0/24)                               │    │   │
│  │  │  • API Gateway (Yarp)  • Rate Limiting                              │    │   │
│  │  └─────────────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                             │
│                                      │ (VPC Peering + Firewall)                    │
│                                      ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                    VPC DE APLICAÇÃO - 10.0.16.0/20                          │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │   │
│  │  │  Subnet: app-services (10.0.16.0/24)                                │    │   │
│  │  │  • GKE Cluster  • Auth API  • Lancamentos API                       │    │   │
│  │  │  • Consolidacao API  • Relatorios API                               │    │   │
│  │  └─────────────────────────────────────────────────────────────────────┘    │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │   │
│  │  │  Subnet: app-workers (10.0.17.0/24)                                 │    │   │
│  │  │  • Consolidacao Worker  • Notificacoes Worker  • Auditoria Worker   │    │   │
│  │  └─────────────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                             │
│                                      │ (Private Google Access)                     │
│                                      ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                    VPC DE DADOS - 10.0.32.0/20                              │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │   │
│  │  │  Subnet: data-sql (10.0.32.0/24)                                    │    │   │
│  │  │  • Cloud SQL (PostgreSQL)  • Read Replica                           │    │   │
│  │  └─────────────────────────────────────────────────────────────────────┘    │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │   │
│  │  │  Subnet: data-cache (10.0.33.0/24)                                  │    │   │
│  │  │  • Memorystore (Redis)  • Redis Cluster                             │    │   │
│  │  └─────────────────────────────────────────────────────────────────────┘    │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │   │
│  │  │  Subnet: data-storage (10.0.34.0/24)                                │    │   │
│  │  │  • Cloud Storage  • MongoDB                                         │    │   │
│  │  └─────────────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Firewall Rules

```
# Deny all ingress by default
resource "google_compute_firewall" "deny_all_ingress" {
  name    = "deny-all-ingress"
  network = google_compute_network.app_vpc.name

  deny {
    protocol = "all"
    ports    = []
  }

  source_ranges = ["0.0.0.0/0"]
  priority      = 65535
  
  log_config {
    metadata = "INCLUDE_ALL_METADATA"
  }
}

# Allow only from API Gateway to GKE
resource "google_compute_firewall" "allow_gateway_to_gke" {
  name    = "allow-gateway-to-gke"
  network = google_compute_network.app_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["8080-8090"]
  }

  source_tags = ["api-gateway"]
  target_tags = ["gke-node"]
  priority    = 1000
  
  log_config {
    metadata = "INCLUDE_ALL_METADATA"
  }
}

# Allow GKE to Cloud SQL
resource "google_compute_firewall" "allow_gke_to_cloudsql" {
  name    = "allow-gke-to-cloudsql"
  network = google_compute_network.app_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["5432"]
  }

  source_tags = ["gke-node"]
  target_tags = ["cloud-sql"]
  priority    = 1000
  
  log_config {
    metadata = "INCLUDE_ALL_METADATA"
  }
}

# Allow GKE to Redis
resource "google_compute_firewall" "allow_gke_to_redis" {
  name    = "allow-gke-to-redis"
  network = google_compute_network.app_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["6379"]
  }

  source_tags = ["gke-node"]
  target_tags = ["redis"]
  priority    = 1000
  
  log_config {
    metadata = "INCLUDE_ALL_METADATA"
  }
}

# Allow internal communication between GKE pods
resource "google_compute_firewall" "allow_internal_gke" {
  name    = "allow-internal-gke"
  network = google_compute_network.app_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["1-65535"]
  }

  allow {
    protocol = "udp"
    ports    = ["1-65535"]
  }

  source_tags = ["gke-node"]
  target_tags = ["gke-node"]
  priority    = 2000
}
```
## 4.3 VPC Service Controls

___O VPC Service Controls cria um perímetro de segurança ao redor dos serviços gerenciados:__

```
# VPC Service Controls Perimeter
resource "google_access_context_manager_service_perimeter" "data_perimeter" {
  parent = "accessPolicies/${google_access_context_manager_access_policy.fluxo_caixa.name}"
  name   = "accessPolicies/${google_access_context_manager_access_policy.fluxo_caixa.name}/servicePerimeters/data_perimeter"
  title  = "Data Perimeter"
  
  status {
    resources = [
      "projects/${var.project_id}",
    ]

    restricted_services = [
      "storage.googleapis.com",
      "sqladmin.googleapis.com",
      "redis.googleapis.com",
      "pubsub.googleapis.com",
      "bigquery.googleapis.com"
    ]

    vpc_accessible_services {
      enable_restriction = true
      allowed_services   = [
        "restricted.googleapis.com",
        "storage.googleapis.com",
        "bigquery.googleapis.com"
      ]
    }
  }
}
```

### 4.4 Cloud NAT e Private Google Access

```
# Cloud NAT for outbound internet (no inbound)
resource "google_compute_router_nat" "cloud_nat" {
  name                               = "fluxo-caixa-nat"
  router                            = google_compute_router.router.name
  region                            = var.region
  nat_ip_allocate_option            = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "LIST_OF_SUBNETWORKS"
  
  subnetworks {
    name                     = google_compute_subnetwork.app_subnet.id
    source_ip_ranges_to_nat  = ["PRIMARY_IP_RANGE"]
  }

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}

# Private Google Access for subnets
resource "google_compute_subnetwork" "private_subnet" {
  name          = "private-subnet"
  ip_cidr_range = "10.0.32.0/24"
  region        = var.region
  network       = google_compute_network.data_vpc.id

  private_ip_google_access = true
}
```

### 4.5 Alertas da Camada de Rede

|Alerta|	Severidade|	Gatilho	Ação|	Canal
|Port Scanning Detectado|	🟠 ALTA|	> 10 portas escaneadas|	Bloquear IP + Notificar	|PagerDuty + Slack
|Tráfego não Autorizado|	🔴 CRÍTICA|	Firewall bloqueia tráfego|	Investigar origem	|PagerDuty
|VPC Peering Alterado|	🟡 MÉDIA|	Mudança em peering|	Revisar alteração	|E-mail + Slack
|Firewall Rule Alterada|	🟡 MÉDIA|	Mudança em regra de firewall|	Revisar alteração	|Slack
|Cloud NAT Log Error|	🟠 ALTA|	Erros no NAT|	Verificar conectividade	|Slack
|Subnet IP Exaustão|	🟡 MÉDIA|	Uso de IP > 80%|	Expandir subnet	|E-mail
|VPC Flow Log Anomaly|	🟠 ALTA|	Tráfego anômalo detectado|	Investigar origem	|Slack


### 5. Camada 3: Segurança de Identidade e Acesso (IAM)
__5.1 Cloud Identity + Active Directory__
__O ecossistema suporta dois modos de autenticação:__

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    FLUXO DE AUTENTICAÇÃO - CLOUD IDENTITY + AD                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                       OPÇÃO 1: CLOUD IDENTITY                               │    │
│  │                                                                             │    │
│  │  Comerciante → Cloud Identity User Pool → MFA (TOTP/SMS) → JWT Token        │    │
│  │                                                                             │    │
│  │  • Suporte a Social Login (Google, Apple, Microsoft)                        │    │
│  │  • Federated Identities                                                     │    │
│  │  • SAML 2.0 / OIDC                                                          │    │
│  │                                                                             │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                       OPÇÃO 2: ACTIVE DIRECTORY                             │    │
│  │                                                                             │    │
│  │  Comerciante → LDAP Bind → Kerberos Ticket → Sincronização → JWT Token      │    │
│  │                                                                             │    │
│  │  • LDAP Authentication (port 389/636)                                       │    │
│  │  • Kerberos Ticket Validation                                               │    │
│  │  • Group-based Authorization                                                │    │
│  │  • On-Premises or Azure AD                                                  │    │
│  │                                                                             │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 JWT (JSON Web Token) - Detalhamento Completo

__O JWT (JSON Web Token) é o padrão utilizado para autenticação e autorização no ecossistema.__
__Estrutura do JWT__


__Um JWT é composto por três partes codificadas em Base64Url, separadas por pontos (.):__


```

eyJhbGciOiJSUzI1NiIsImtpZCI6IjEifQ.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvYW8gQ29tZXJjaWFudGUiLCJpYXQiOjE1MTYyMzkwMjIsInJvbGVzIjpbImNvbWVyY2lhbnRlIl19.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
         │                             │                                         │
         │                             │                                         │
         ▼                             ▼                                         ▼
      HEADER                        PAYLOAD                                   SIGNATURE


```

### Header (Cabeçalho)

```

{
  "alg": "RS256",              // Algoritmo de assinatura
  "kid": "1",                  // Key ID (para rotação de chaves)
  "typ": "JWT",                // Tipo do token
  "iss": "fluxo-caixa-auth"    // Issuer
}
```


### Payload (Claims) - Detalhado para o Ecossistema

```

{
  // Standard Claims (RFC 7519)
  "sub": "user_1234567890",           // Subject (Identificador do usuário)
  "iss": "fluxo-caixa-auth",          // Issuer (Quem emitiu)
  "aud": "fluxo-caixa-api",           // Audience (Para quem é destinado)
  "exp": 1743897600,                  // Expiration (Expiração - Unix timestamp)
  "iat": 1743811200,                  // Issued At (Emissão)
  "nbf": 1743811200,                  // Not Before (Não antes de)
  "jti": "jwt_abc123def456",          // JWT ID (Identificador único)
  
  // Custom Claims do Ecossistema
  "user_id": "usr_12345678",          // ID interno do usuário
  "email": "joao@fluxocaixa.com",     // E-mail do usuário
  "name": "João Comerciante",         // Nome completo
  "roles": [                          // Papéis do usuário
    "comerciante",
    "premium"
  ],
  "permissions": [                    // Permissões específicas
    "lancamento:write",
    "lancamento:read",
    "consolidado:read",
    "relatorio:generate"
  ],
  "auth_provider": "cloud_identity",  // Provedor de autenticação
  "auth_provider_id": "google_123456",// ID no provedor
  "mfa_verified": true,               // MFA foi verificado
  "device_id": "device_abcdef",       // Dispositivo autenticado
  "session_id": "sess_123456",        // ID da sessão
  "tenant_id": "tenant_001",          // Tenant (multi-tenant)
  "environment": "production"         // Ambiente
}

```

### Signature (Assinatura)

```

```