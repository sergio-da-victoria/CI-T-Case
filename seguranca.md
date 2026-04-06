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


### 1. Visão Geral da Arquitetura de Segurança
__A camada de segurança do ecossistema Fluxo de Caixa foi projetada seguindo o princípio de Defesa em Profundidade (Defense in Depth), onde múltiplas camadas de controle protegem o sistema contra falhas ou vulnerabilidades em qualquer camada individual.__

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
yaml


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
hcl

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

```
hcl

# DDoS Protection Configuration
resource "google_compute_security_policy" "ddos_protection" {
  name = "fluxo-caixa-ddos-protection"

  rule {
    action   = "throttle"
    priority = 500
    match {
      expr {
        expression = "request.path.matches('/api/.*')"
      }
    }
    rate_limit_options {
      rate_limit_threshold = 1000
      rate_limit_window_sec = 10
      conform_action = "allow"
      exceed_action = "throttle"
      enforce_on_key = "IP"
    }
    description = "Throttle API requests during DDoS"
  }
}

```


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
hcl

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
hcl

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
hcl

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
|--|--|--|--|
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
Json

{
  "alg": "RS256",              // Algoritmo de assinatura
  "kid": "1",                  // Key ID (para rotação de chaves)
  "typ": "JWT",                // Tipo do token
  "iss": "fluxo-caixa-auth"    // Issuer
}
```


### Payload (Claims) - Detalhado para o Ecossistema

```
json

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
__A assinatura é criada usando o algoritmo especificado no header:__


```
RSASHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  private_key
)

```

### 5.3 Estrutura do Token no Ecossistema

```

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ESTRUTURA DO TOKEN NO ECOSSISTEMA                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         ACCESS TOKEN (JWT)                                  │    │
│  │  • Validade: 1 hora                                                         │    │
│  │  • Uso: Autorização de requisições API                                      │    │
│  │  • Armazenamento: Memória (cliente) / Redis (server-side)                   │    │
│  │  • Transmissão: Bearer Token no Header Authorization                        │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         REFRESH TOKEN                                       │    │
│  │  • Validade: 7 dias                                                         │    │
│  │  • Uso: Renovação do Access Token                                           │    │
│  │  • Armazenamento: HTTP Only Cookie / Redis (server-side)                    │    │
│  │  • Revogação: Possível via API                                              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         ID TOKEN (OIDC)                                     │    │
│  │  • Validade: 1 hora                                                         │    │
│  │  • Uso: Informações do usuário para o frontend                              │    │
│  │  • Armazenamento: Memória (cliente)                                         │    │
│  │  • Conteúdo: Dados do usuário (nome, email, avatar)                         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```
### 5.4 Fluxo de Validação do Token

```
csharp

// Validação de JWT no API Gateway
public class JwtValidationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ITokenRevocationService _revocationService;
    private readonly ILogger<JwtValidationMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        var token = ExtractToken(context.Request);
        
        if (token == null)
        {
            context.Response.StatusCode = 401;
            return;
        }
        
        // 1. Validar assinatura
        if (!ValidateSignature(token))
        {
            _logger.LogWarning("Invalid JWT signature");
            context.Response.StatusCode = 401;
            return;
        }
        
        // 2. Validar expiração
        if (IsTokenExpired(token))
        {
            _logger.LogWarning("Expired JWT token");
            context.Response.StatusCode = 401;
            return;
        }
        
        // 3. Validar issuer e audience
        if (!ValidateIssuerAndAudience(token))
        {
            _logger.LogWarning("Invalid issuer or audience");
            context.Response.StatusCode = 401;
            return;
        }
        
        // 4. Verificar revogação
        if (await _revocationService.IsRevoked(token))
        {
            _logger.LogWarning("Revoked JWT token");
            context.Response.StatusCode = 401;
            return;
        }
        
        // 5. Verificar permissões (RBAC)
        var userRoles = ExtractRoles(token);
        var requiredRole = GetRequiredRole(context.Request);
        
        if (!userRoles.Contains(requiredRole))
        {
            _logger.LogWarning($"Insufficient permissions: {string.Join(",", userRoles)}");
            context.Response.StatusCode = 403;
            return;
        }
        
        // 6. Adicionar claims ao contexto
        context.User = CreatePrincipal(token);
        
        await _next(context);
    }
    
    private bool ValidateSignature(string token)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var validationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSecret)),
            ValidateIssuer = false,
            ValidateAudience = false,
            ClockSkew = TimeSpan.Zero
        };
        
        try
        {
            tokenHandler.ValidateToken(token, validationParameters, out _);
            return true;
        }
        catch
        {
            return false;
        }
    }
    
    private bool IsTokenExpired(string token)
    {
        var jwt = new JwtSecurityTokenHandler().ReadJwtToken(token);
        return jwt.ValidTo < DateTime.UtcNow;
    }
}
```

### 5.5 Refresh Token e Revogação

```

csharp

// Serviço de Refresh Token
public class TokenService
{
    private readonly ICacheService _cache;
    private readonly ILogger<TokenService> _logger;

    public async Task<TokenResponse> RefreshTokenAsync(string refreshToken)
    {
        // 1. Validar refresh token no cache
        var userId = await _cache.GetAsync<string>($"refresh_token:{refreshToken}");
        
        if (userId == null)
        {
            _logger.LogWarning("Invalid or expired refresh token");
            throw new UnauthorizedAccessException();
        }
        
        // 2. Revogar token antigo
        await RevokeTokenAsync(refreshToken);
        
        // 3. Gerar novo access token
        var newAccessToken = GenerateAccessToken(userId);
        
        // 4. Gerar novo refresh token
        var newRefreshToken = GenerateRefreshToken();
        
        // 5. Armazenar novo refresh token
        await _cache.SetAsync($"refresh_token:{newRefreshToken}", userId, TimeSpan.FromDays(7));
        
        return new TokenResponse
        {
            AccessToken = newAccessToken,
            RefreshToken = newRefreshToken,
            ExpiresIn = 3600
        };
    }
    
    public async Task RevokeTokenAsync(string refreshToken)
    {
        // Remover do cache
        await _cache.RemoveAsync($"refresh_token:{refreshToken}");
        
        // Adicionar à blacklist (opcional)
        await _cache.SetAsync($"revoked:{refreshToken}", "true", TimeSpan.FromDays(7));
        
        _logger.LogInformation($"Token revoked: {refreshToken}");
    }
}
```

## 5.6 IAM Roles e Políticas

```
hcl

# IAM Roles para o ecossistema
resource "google_project_iam_custom_role" "comerciante_role" {
  role_id     = "comercianteRole"
  title       = "Comerciante Role"
  description = "Role for comerciante users"
  
  permissions = [
    "lancamentos.create",
    "lancamentos.read",
    "lancamentos.update",
    "lancamentos.delete",
    "consolidado.read",
    "relatorio.generate"
  ]
}

resource "google_project_iam_custom_role" "administrador_role" {
  role_id     = "administradorRole"
  title       = "Administrador Role"
  description = "Role for administrators"
  
  permissions = [
    "lancamentos.*",
    "consolidado.*",
    "relatorio.*",
    "usuarios.*",
    "configuracoes.*"
  ]
}

resource "google_project_iam_custom_role" "auditor_role" {
  role_id     = "auditorRole"
  title       = "Auditor Role"
  description = "Role for auditors"
  
  permissions = [
    "auditoria.read",
    "relatorio.read",
    "logs.read"
  ]
}

# IAM Policy Bindings
resource "google_project_iam_binding" "comerciante_binding" {
  project = var.project_id
  role    = "projects/${var.project_id}/roles/comercianteRole"
  
  members = [
    "group:comerciantes@fluxocaixa.com",
  ]
}
```

### 5.7 Workload Identity

__Workload Identity permite que aplicações em execução no GKE autentiquem-se como service accounts do Google Cloud sem usar chaves de API.__

```
hcl

# Workload Identity Configuration
resource "google_service_account" "workload_identity_sa" {
  account_id   = "fluxo-caixa-wi-sa"
  display_name = "Workload Identity Service Account"
}

resource "google_service_account_iam_binding" "workload_identity_binding" {
  service_account_id = google_service_account.workload_identity_sa.name
  role               = "roles/iam.workloadIdentityUser"
  
  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[fluxo-caixa/default]"
  ]
}
```


```
yaml

# Kubernetes Service Account com Workload Identity

apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluxo-caixa-sa
  namespace: fluxo-caixa
  annotations:
    iam.gke.io/gcp-service-account: fluxo-caixa-wi-sa@${PROJECT_ID}.iam.gserviceaccount.com
```

### 5.8 Alertas da Camada de Identidade

|Alerta|	Severidade|	Gatilho|	Ação|	Canal
|--|--|--|--|--
|Login Suspeito|	🟠 ALTA|	Login de localização diferente|	MFA + Notificar usuário	|Slack + E-mail
|MFA Falhou Múltiplas Vezes|	🟡 MÉDIA|	> 5 falhas MFA em 5min|	Temporariamente bloquear	|Slack
|Privilégio Elevado Atribuído|	🟠 ALTA|	Role admin atribuída|	Revisar e aprovar	|Slack + E-mail
|Token JWT Inválido|	🟡 MÉDIA|	Token inválido/recebido|	Log + Monitorar|	Log only
|Refresh Token Revogado|	🟡 MÉDIA|	Revogação manual de token|	Log	|Slack
|Acesso Não Autorizado|	🔴 CRÍTICA|	Tentativa de acesso sem permissão|	Bloquear + Investigar	|PagerDuty
|Service Account Key Criada|	🟠 ALTA|	Criação de chave de SA|	Revisar necessidade	|Slack
|IAM Policy Alterada|	🟡 MÉDIA|	Mudança em política IAM|	Revisar alteração	|E-mail


### 6. Camada 4: Segurança de Aplicação (App Security)

### 6.1 Arquitetura de Segurança de Aplicação
```

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    SEGURANÇA DE APLICAÇÃO COM KAFKA                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         API GATEWAY (Yarp)                                   │   │
│  │  • JWT Validation  • Rate Limiting  • Input Validation  • Security Headers  │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                       │                                             │
│                                       ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         KAFKA TOPICS - SEGURANÇA                             │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │   │
│  │  │ api-events  │ │auth-events  │ │security-    │ │threat-      │           │   │
│  │  │             │ │             │ │logs         │ │detection    │           │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                       │                                             │
│         ┌─────────────────────────────┼─────────────────────────────┐              │
│         ▼                             ▼                             ▼              │
│  ┌─────────────┐              ┌─────────────┐              ┌─────────────┐        │
│  │ Security    │              │ Threat      │              │ Audit       │        │
│  │ Analyzer    │              │ Detector    │              │ Logger      │        │
│  │ (Consumer)  │              │ (Consumer)  │              │ (Consumer)  │        │
│  └─────────────┘              └─────────────┘              └─────────────┘        │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 API Gateway Security

```
sharp


// API Gateway Security Middleware
public class ApiGatewaySecurityMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRateLimiter _rateLimiter;
    private readonly IInputValidator _inputValidator;
    
    public async Task InvokeAsync(HttpContext context)
    {
        // 1. Validate JWT
        if (!await ValidateJwt(context))
        {
            context.Response.StatusCode = 401;
            return;
        }
        
        // 2. Check Rate Limit
        var clientId = GetClientId(context);
        if (!_rateLimiter.IsAllowed(clientId))
        {
            context.Response.StatusCode = 429;
            context.Response.Headers.Add("Retry-After", "60");
            return;
        }
        
        // 3. Validate Input (Anti-XSS, SQL Injection)
        if (!_inputValidator.ValidateRequest(context.Request))
        {
            context.Response.StatusCode = 400;
            return;
        }
        
        // 4. Add Security Headers
        context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
        context.Response.Headers.Add("X-Frame-Options", "DENY");
        context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
        context.Response.Headers.Add("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
        context.Response.Headers.Add("Content-Security-Policy", "default-src 'self'");
        
        await _next(context);
    }
}
```

### 6.3 SAST/DAST e Container Scanning

```
yaml

# Cloud Build SAST Pipeline
steps:
  # SAST with Semgrep
  - name: 'semgrep/semgrep'
    id: 'sast-scan'
    args: ['scan', '--config', 'p/owasp-top-ten', '--json', '--output', 'sast-results.json']
  
  # Dependency Scanning with Snyk
  - name: 'snyk/snyk'
    id: 'dependency-scan'
    args: ['test', '--json-file-output=dep-results.json']
  
  # Container Scanning with Trivy
  - name: 'aquasec/trivy'
    id: 'container-scan'
    args: ['image', '--severity', 'CRITICAL,HIGH', '--format', 'json', '--output', 'container-results.json', '${_IMAGE_NAME}']
  
  # Fail on critical vulnerabilities
  - name: 'gcr.io/cloud-builders/gcloud'
    id: 'check-results'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        if grep -q '"severity":"CRITICAL"' sast-results.json; then
          echo "❌ Critical SAST findings found!"
          exit 1
        fi

```


### 6.4 Secret Management

```
csharp

// Secret Manager Service
public class SecretManagerService
{
    private readonly SecretManagerServiceClient _client;
    private readonly ILogger<SecretManagerService> _logger;
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        var secretVersionName = new SecretVersionName(_projectId, secretName, "latest");
        
        try
        {
            var response = await _client.AccessSecretVersionAsync(secretVersionName);
            var secret = response.Payload.Data.ToStringUtf8();
            
            // Log access (audit)
            _logger.LogInformation($"Secret accessed: {secretName}");
            
            return secret;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Failed to access secret: {secretName}");
            throw;
        }
    }
    
    public async Task<string> CreateSecretAsync(string secretName, string secretValue)
    {
        var secret = new Secret
        {
            SecretName = new SecretName(_projectId, secretName),
            Replication = new Replication
            {
                Automatic = new Automatic()
            }
        };
        
        var response = await _client.CreateSecretAsync(secret);
        
        var secretVersion = new SecretVersion
        {
            SecretVersionName = new SecretVersionName(_projectId, secretName, "1"),
            Payload = new SecretPayload
            {
                Data = ByteString.CopyFromUtf8(secretValue)
            }
        };
        
        await _client.AddSecretVersionAsync(secretVersion);
        
        return response.SecretName.SecretId;
    }
}

```

### 6.5 Alertas da Camada de Aplicação

|Alerta|	Severidade|	Gatilho|	Ação|	Canal
|--|--|--|--|--
|SQL Injection Detectado|	🔴 CRÍTICA|	Padrão SQLi na request|	Bloquear IP	|PagerDuty
XSS Detectado|	🔴 CRÍTICA|	Padrão XSS na request|	Bloquear IP	|PagerDuty
JWT Inválido|	🟡 MÉDIA|	Token inválido/recebido|	Log	|Slack
Vulnerabilidade Crítica|	🔴 CRÍTICA|	CVE crítica detectada|	Patch urgente	|PagerDuty
Secret Acessado por Humano|	🟠 ALTA|	Secret acessado por usuário|	Investigar	|Slack
API Abuse Detectado|	🟠 ALTA|	Padrão de abuso de API|	Rate limit + Alertar	|Slack
Container Vulnerability|	🟠 ALTA|	CVE em container image|	Rebuild image	|Slack


### 7. Camada 5: Segurança de Dados (Data Security)
__7.1 Criptografia em Trânsito (TLS 1.3)__



```
yaml

# TLS 1.3 Configuration for Load Balancer
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: fluxo-caixa-cert
spec:
  domains:
    - api.fluxocaixa.com
    - www.fluxocaixa.com
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "fluxo-caixa-ip"
    networking.gke.io/managed-certificates: "fluxo-caixa-cert"
    kubernetes.io/ingress.allow-http: "false"
spec:
  rules:
  - host: api.fluxocaixa.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-gateway
            port:
              number: 443
```


### 7.2 Criptografia em Repouso (AES-256)


```
hcl

# Cloud SQL with CMEK
resource "google_sql_database_instance" "postgres" {
  name             = "fluxo-caixa-postgres"
  database_version = "POSTGRES_16"
  region          = var.region

  settings {
    tier              = "db-custom-4-16384"
    disk_size         = 100
    disk_type         = "PD_SSD"
    disk_autoresize   = true
    availability_type = "REGIONAL"

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.data_vpc.id
      
      ssl_mode = "ENCRYPTED_ONLY"
    }

    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      
      backup_retention_settings {
        retained_backups = 30
        retention_unit   = "COUNT"
      }
    }

    disk_encryption_configuration {
      kms_key_name = google_kms_crypto_key.database_key.id
    }
  }

  deletion_protection = true
}
```

### 7.3 Cloud KMS (Key Management Service)

```

hcl

# Cloud KMS Key Ring
resource "google_kms_key_ring" "fluxo_caixa" {
  name     = "fluxo-caixa-keyring"
  location = var.region
}

# Database Encryption Key
resource "google_kms_crypto_key" "database_key" {
  name            = "database-encryption-key"
  key_ring        = google_kms_key_ring.fluxo_caixa.id
  rotation_period = "7776000s" # 90 days

  version_template {
    algorithm = "GOOGLE_SYMMETRIC_ENCRYPTION"
  }
}

# Application Secrets Key
resource "google_kms_crypto_key" "app_secrets_key" {
  name            = "app-secrets-key"
  key_ring        = google_kms_key_ring.fluxo_caixa.id
  rotation_period = "7776000s"
}

# Backup Key
resource "google_kms_crypto_key" "backup_key" {
  name            = "backup-encryption-key"
  key_ring        = google_kms_key_ring.fluxo_caixa.id
  rotation_period = "7776000s"
}

# Key Rotation Schedule
resource "google_kms_crypto_key_version" "database_key_version" {
  crypto_key = google_kms_crypto_key.database_key.id
  state      = "ENABLED"
}

# Key Access Control
resource "google_kms_crypto_key_iam_binding" "database_key_binding" {
  crypto_key_id = google_kms_crypto_key.database_key.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"

  members = [
    "serviceAccount:${google_service_account.cloudsql_sa.email}",
    "serviceAccount:${google_service_account.backup_sa.email}"
  ]
}

```

### 7.4 Data Masking e Tokenização

```
csharp


// Data Masking Service
public class DataMaskingService
{
    // Mask PII data
    public string MaskEmail(string email)
    {
        if (string.IsNullOrEmpty(email)) return email;
        var parts = email.Split('@');
        if (parts.Length != 2) return email;
        
        var localPart = parts[0];
        var masked = localPart.Length <= 2 
            ? "***" 
            : localPart.Substring(0, 2) + new string('*', localPart.Length - 2);
        
        return $"{masked}@{parts[1]}";
    }
    
    public string MaskDocument(string document)
    {
        if (string.IsNullOrEmpty(document)) return document;
        if (document.Length <= 4) return "****";
        
        return "***" + document.Substring(document.Length - 4);
    }
    
    public string MaskCreditCard(string cardNumber)
    {
        if (string.IsNullOrEmpty(cardNumber)) return cardNumber;
        if (cardNumber.Length < 4) return "****";
        
        return new string('*', cardNumber.Length - 4) + cardNumber.Substring(cardNumber.Length - 4);
    }
    
    // Tokenization for sensitive data
    public async Task<string> TokenizeAsync(string sensitiveData)
    {
        var token = Guid.NewGuid().ToString();
        await _cache.SetAsync($"token:{token}", sensitiveData, TimeSpan.FromHours(24));
        return token;
    }
    
    public async Task<string> DetokenizeAsync(string token)
    {
        return await _cache.GetAsync<string>($"token:{token}");
    }
}
```

### 7.5 Backup Encryption


```

hcl


resource "google_storage_bucket" "encrypted_backups" {
  name          = "${var.project_id}-encrypted-backups"
  location      = var.region
  force_destroy = false

  encryption {
    default_kms_key_name = google_kms_crypto_key.backup_key.id
  }

  versioning {
    enabled = true
  }

  retention_policy {
    retention_period = 2592000 # 30 days
    is_locked        = false
  }

  lifecycle_rule {
    condition {
      age = 90
    }
    action {
      type = "Delete"
    }
  }
}


```

### 7.6 Alertas da Camada de Dados

|Alerta|	Severidade|	Gatilho|	Ação|	Canal
|--|--|--|--|--
|Exfiltração de Dados|	🔴 CRÍTICA|	Volume anômalo de dados|	Bloquear + Investigar|	PagerDuty
|Chave KMS Comprometida|	🔴 CRÍTICA|	Rota de chave|	Rotacionar imediatamente|	PagerDuty
|Backup Falhou|	🟠 ALTA|	Backup não completou|	Reexecutar backup|	Slack + E-mail
|Acesso Não Autorizado a Dados|	🔴 CRÍTICA|	Query sem permissão|	Bloquear + Investigar|	PagerDuty
|Data Masking Falhou|	🟡 MÉDIA|	Dados sensíveis expostos|	Corrigir masking|	Slack
|KMS Key Rotation Atrasada|	🟡 MÉDIA|	Key > 90 dias sem rotação|	Rotacionar chave|	Slack
|Criptografia Desabilitada|	🟠 ALTA|	TLS/SSL desabilitado|	Reativar imediatamente|	PagerDuty

### 8. Camada 6: Monitoramento e Resposta a Incidentes
__8.1 Security Command Center (SCC)__

```
hcl

# Security Command Center Configuration
resource "google_scc_source" "fluxo_caixa_source" {
  organization    = var.org_id
  display_name    = "Fluxo Caixa Security Source"
  description     = "Custom security findings for Fluxo Caixa"
}


resource "google_pubsub_topic" "security_alerts" {
  name = "security-alerts"
}

resource "google_pubsub_subscription" "security_alert_processor" {
  name  = "security-alert-processor"
  topic = google_pubsub_topic.security_alerts.name

  push_config {
    push_endpoint = "https://security-alert-processor.fluxocaixa.com/webhook"
    attributes = {
      x-goog-version = "v1"
    }
  }
}

```

### 8.2 Chronicle SIEM

```
python


# Chronicle SIEM Integration
from google.cloud import chronicle

class ChronicleSIEM:
    def __init__(self, project_id, location):
        self.client = chronicle.Client()
        self.project_id = project_id
        self.location = location
    
    def ingest_log(self, log_data, log_type):
        """Ingest log into Chronicle"""
        entry = {
            "log_type": log_type,
            "timestamp": datetime.utcnow().isoformat(),
            "data": log_data
        }
        
        response = self.client.ingest(
            project_id=self.project_id,
            location=self.location,
            log_entry=entry
        )
        
        return response
    
    def create_udm_event(self, event_data):
        """Create Unified Data Model event"""
        udm_event = {
            "metadata": {
                "event_type": "GENERIC_EVENT",
                "product_name": "Fluxo Caixa",
                "vendor_name": "Fluxo Caixa"
            },
            "principal": {
                "user": event_data.get("user"),
                "ip": event_data.get("ip_address")
            },
            "security_result": {
                "severity": event_data.get("severity"),
                "action": event_data.get("action")
            }
        }
        
        return udm_event
    
    def search_ioc(self, ioc_type, ioc_value):
        """Search for Indicator of Compromise"""
        response = self.client.search(
            project_id=self.project_id,
            ioc_type=ioc_type,
            ioc_value=ioc_value,
            time_range={"start_time": "2025-01-01T00:00:00Z"}
        )
        
        return response

```

### 8.3 Cloud Logging e Monitoring

```
hcl


# Security Log Sinks
resource "google_logging_project_sink" "security_logs" {
  name        = "security-logs-sink"
  destination = "bigquery.googleapis.com/projects/${var.project_id}/datasets/security_logs"
  filter      = <<EOF
    (severity >= "WARNING") OR
    (protoPayload.methodName:("firewalls" OR "iam" OR "compute")) OR
    (resource.type = "k8s_cluster") OR
    (logName:"cloudaudit.googleapis.com") OR
    (protoPayload.authenticationInfo.principalEmail:".*")
  EOF

  unique_writer_identity = true
}

```

### 8.4 Cloud Monitoring Alert Policies

```
hcl


# Cloud Monitoring Alert Policies
resource "google_monitoring_alert_policy" "security_alert" {
  display_name = "Security Alert"
  combiner     = "OR"
  
  conditions {
    display_name = "High severity log entry"
    condition_threshold {
      filter     = "resource.type = \"global\" AND severity >= \"WARNING\""
      duration   = "60s"
      comparison = "COMPARISON_GT"
      threshold_value = 0
      
      trigger {
        count = 1
      }
    }
  }
  
  notification_channels = [
    google_monitoring_notification_channel.slack.id,
    google_monitoring_notification_channel.pagerduty.id
  }
  
  alert_strategy {
    auto_close = "1800s"
  }
}

```


### 8.5 Tabela Completa de Alertas


|#|Categoria	|Alerta	|Severidade	|Gatilho	|Ação	|Canal	|SLA
|--|--|--|--|--|--|--|--
|1|	Perímetro|	SQL Injection|	🔴 CRÍTICA|	WAF bloqueia SQLi|	Isolar IP + Notificar	|PagerDuty + Slack	|5min
|2|	Perímetro|	XSS Detectado|	🔴 CRÍTICA|	WAF bloqueia XSS|	Isolar IP + Notificar	|PagerDuty + Slack	|5min
|3|	Perímetro|	Rate Limit Excedido|	🟠 ALTA|	> 100 req/min por IP|	Alertar + Bloquear	|Slack	|15min
|4|	Perímetro|	DDoS Attack|	🔴 CRÍTICA|	Tráfego > 3x normal|	Ativar proteção	|PagerDuty	|2min
|5|	Perímetro|	IP Malicioso|	🟡 MÉDIA|	IP em blacklist|	Log + Monitorar	|Slack	|30min
|6|	Rede|	Port Scanning|	🟠 ALTA|	> 10 portas escaneadas	|Bloquear IP	|PagerDuty + Slack|	10min
|7|	Rede|	Tráfego não Autorizado|	🔴 CRÍTICA|	Firewall bloqueia	|Investigar	|PagerDuty	|5min
|8|	Rede|	VPC Peering Alterado|	🟡 MÉDIA|	Mudança em peering|	Revisar|	E-mail + Slack| 	1h
|9|	Rede|	Firewall Rule Alterada|	🟡 MÉDIA|	Mudança em regra|	Revisar|	Slack|	1h
|10|	Identidade|	Login Suspeito|	🟠 ALTA|	Localização diferente|	MFA + Notificar|	Slack + E-mail|	15min
|11|	Identidade|	MFA Falhou|	🟡 MÉDIA|	> 5 falhas em 5min|	Bloquear temp	|Slack	|30min
|12|	Identidade|	Privilégio Elevado|	🟠 ALTA|	Role admin atribuída|	Revisar|	Slack + E-mail|	15min
|13|	Identidade|	Token JWT Inválido|	🟡 MÉDIA|	Token inválido|	Log|	Log only|	1h
|14|	Identidade|	Acesso Não Autorizado|	🔴 CRÍTICA|	Sem permissão|	Bloquear|	PagerDuty|	5min
|15|	Aplicação|	SQL Injection|	🔴 CRÍTICA|	Padrão SQLi|	Bloquear IP|	PagerDuty|	5min
|16|	Aplicação|	Vulnerabilidade Crítica|	🔴 CRÍTICA|	CVE crítica|	Patch urgente|	PagerDuty|	2h
|17|	Aplicação|	Secret Acessado|	🟠 ALTA|	Acesso por humano|	Investigar|	Slack|	30min
|18|	Aplicação|	API Abuse|	🟠 ALTA|	Padrão de abuso|	Rate limit	|Slack|	15min
|19|	Dados|	Exfiltração de Dados|	🔴 CRÍTICA|	Volume anômalo|	Bloquear|	PagerDuty|	2min
|20|	Dados|	Chave KMS Comprometida|	🔴 CRÍTICA|	Rota|	Rotacionar	|PagerDuty|	2min
|21|	Dados|	Backup Falhou|	🟠 ALTA|	Backup falhou|	Reexecutar|	Slack + E-mail|	1h
|22|	Dados|	Criptografia Desabilitada|	🟠 ALTA|	TLS desabilitado|	Reativar|	PagerDuty|	10min
|23|	SIEM|	IOC Detectado|	🔴 CRÍTICA|	Threat intel match|	Isolar + Investigar|	PagerDuty|	5min
|24|	SIEM|	Comportamento Anômalo|	🟠 ALTA|	UBA detecta anomalia|	Investigar|	Slack|	30min


### 9. Matriz de Responsabilidades por Camada
|Camada|	Componente|	Google Cloud|	Time Security|	Time DevOps|	Time Aplicação
|--|--|--|--|--|--|
|Perímetro|	Cloud Armor (WAF)|	|✅ Infra	|✅ Configuração	|✅ Monitoramento	|❌
|Perímetro|	DDoS Protection|	|✅ Automático	|✅ Configuração	|❌	|❌
|Perímetro|	Rate Limiting|	❌	|✅ Definição	|✅ Configuração	|✅ Ajuste
|Rede|	VPC Firewall|	✅ Base|	✅ Regras|	✅ Configuração|	❌
|Rede|	VPC Service Controls|	✅ Infra|	✅ Perímetro|	✅ Configuração|	❌
|Rede|	Cloud NAT|	✅ Infra	|❌|	✅ Configuração|	❌
|Identidade|	Cloud Identity|	✅ Infra	|✅ Configuração	|✅ Integração|	❌
|Identidade|	JWT Validation|	❌|	✅ Padrão|	✅ Implementação|	✅ Uso
|Identidade|	IAM Roles|	✅ Base|	✅ Definição|	✅ Configuração|	✅ Uso
|Identidade|	Workload Identity|	✅ Infra|	❌|	✅ Configuração|	✅ Uso
|Aplicação|	API Security|	❌|	✅ Padrão|	✅ Configuração|	✅ Implementação
|Aplicação|	SAST/DAST|	❌|	✅ Ferramentas|	✅ Pipeline|	✅ Correção
|Aplicação|	Secret Management|	✅ Infra|	✅ Políticas|	✅ Configuração	|✅ Uso
|Dados|	Encryption at Rest|	✅ Base|	✅ KMS|	✅ Configuração	|❌
|Dados|	Data Masking|	❌|	✅ Regras|	❌|	✅ Implementação
					

# Checklist de Implementação de Segurança - Ecossistema Fluxo de Caixa
# Instruções de Uso

### Marque cada item como ✅ (concluído), 🔄 (em andamento), ⏸️ (pausado) ou ❌ (não iniciado)

-   __Registre a data de conclusão e o responsável__
-   __Itens com 🔴 são críticos e devem ser priorizados__
-   __Revise este checklist mensalmente para garantir conformidade contínua__

### 10. CAMADA 1: SEGURANÇA DE PERÍMETRO (Edge Security)

|#|	Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Ativar Cloud Armor (WAF)|	🔴 CRÍTICO|	❌			
|2|	Configurar regras OWASP Top 10|	🔴 CRÍTICO|	❌			
|3|	Configurar regras anti-SQL Injection|	🔴 CRÍTICO|	❌			
|4|	Configurar regras anti-XSS|	🔴 CRÍTICO|	❌			
|5|	Configurar regras anti-RCE|	🟠 ALTO|	❌			
|6|	Configurar Rate Limiting (100 req/min/IP)|	🔴 CRÍTICO|	❌			
|7|	Configurar Rate Limiting (1000 req/min/user)|	🟠 ALTO|	❌			
|8|	Configurar Geo-blocking (países não autorizados)|	🟡 MÉDIO|	❌			
|9|	Configurar IP Allowlist/Blocklist|	🟡 MÉDIO|	❌			
|10|	Ativar DDoS Protection (Standard)|	🔴 CRÍTICO|	❌			
|11|	Configurar alertas de DDoS|	🟠 ALTO|	❌			
|12|	Configurar Cloud CDN com TLS 10.3|	🟠 ALTO|	❌			
|13|	Configurar logging do WAF|	🟡 MÉDIO|	❌			
|14|	Criar dashboard de monitoramento do perímetro|	🟡 MÉDIO|	❌

### . CAMADA 2: SEGURANÇA DE REDE (Network Security)

|#|	Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Criar VPC DMZ (10.0.0.0/20)|	🔴 CRÍTICO|	❌			
|2|	Criar VPC Aplicação (10.0.16.0/20)|	🔴 CRÍTICO|	❌			
|3|	Criar VPC Dados (10.0.32.0/20)|	🔴 CRÍTICO|	❌			
|4|	Configurar VPC Peering entre camadas|	🔴 CRÍTICO|	❌			
|5|	Configurar regra "Deny All Ingress" (priority 65535)|	🔴 CRÍTICO|	❌			
|6|	Configurar regra "Allow Gateway to GKE"|	🔴 CRÍTICO|	❌			
|7|	Configurar regra "Allow GKE to Cloud SQL"|	🔴 CRÍTICO|	❌			
|8|	Configurar regra "Allow GKE to Redis"|	🟠 ALTO|	❌			
|9|	Configurar regra "Allow Internal GKE Communication"|	🟠 ALTO|	❌			
|10|	Ativar VPC Flow Logs|	🟠 ALTO|	❌			
|11|	Configurar VPC Service Controls|	🔴 CRÍTICO|	❌			
|12|	Configurar Access Context Manager|	🟠 ALTO|	❌			
|13|	Configurar Cloud NAT (outbound only)|	🟠 ALTO|	❌			
|14|	Ativar Private Google Access nas subnets|	🟠 ALTO|	❌			
|15|	Configurar Network Policies no Kubernetes|	🟠 ALTO|	❌			
|16|	Configurar alertas de port scanning|	🟡 MÉDIO|	❌			
|17|	Configurar alertas de alteração de firewall rules|	🟡 MÉDIO|	❌			
|18|	Configurar alertas de alteração de VPC peering|	🟡 MÉDIO|	❌

#### 3. CAMADA 3: SEGURANÇA DE IDENTIDADE E ACESSO (IAM)

|#|Item|	Prioridade|	Status|	Data|	Responsável| Observações
|--|--|--|--|--|--|--
|1|	Configurar Cloud Identity|	🔴 CRÍTICO|	❌			
|2|	Configurar integração com Active Directory|	🟠 ALTO|	❌			
|3|	Configurar MFA obrigatório para todos os usuários|	🔴 CRÍTICO|	❌			
|4|	Configurar SSO (SAML/OIDC)|	🟠 ALTO|	❌			
|5|	Configurar JWT com algoritmo RS256|	🔴 CRÍTICO|	❌			
|6|	Configurar expiração de token (1 hora)|	🔴 CRÍTICO|	❌			
|7|	Configurar Refresh Token (7 dias)|	🟠 ALTO|	❌			
|8|	Implementar validação de JWT no API Gateway|	🔴 CRÍTICO|	❌			
|9|	Implementar validação de assinatura|	🔴 CRÍTICO|	❌			
|10|	Implementar validação de issuer/audience|	🔴 CRÍTICO|	❌			
|11|	Implementar token revocation (blacklist)|	🟠 ALTO|	❌			
|12|	Criar IAM Role: Comerciante|	🔴 CRÍTICO|	❌			
|13|	Criar IAM Role: Administrador|	🔴 CRÍTICO|	❌			
|14|	Criar IAM Role: Auditor|	🟠 ALTO|	❌			
|15|	Configurar Workload Identity para GKE|	🔴 CRÍTICO|	❌			
|16|	Remover Service Account Keys (usar WI)|	🟠 ALTO|	❌			
|17|	Configurar IAM Conditions (ex: horário, IP)|	🟡 MÉDIO|	❌			
|18|	Configurar alertas de login suspeito|	🟠 ALTO|	❌			
|19|	Configurar alertas de MFA falhas múltiplas|	🟡 MÉDIO|	❌			
|20|	Configurar alertas de privilégio elevado|	🟠 ALTO|	❌			
|21|	Configurar alertas de acesso não autorizado|	🔴 CRÍTICO|	❌			
|22|	Configurar alertas de criação de Service Account keys|	🟠 ALTO|	❌



### 4. CAMADA 4: SEGURANÇA DE APLICAÇÃO (Application Security)
|#|Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Implementar validação de input (anti-SQLi, anti-XSS)|	🔴 CRÍTICO|	❌			
|2|	Implementar sanitização de outputs|	🔴 CRÍTICO|	❌			
|3|	Configurar Security Headers (CSP, HSTS, X-Frame-Options)|	🟠 ALTO|	❌			
|4|	Configurar Content Security Policy (CSP)|	🟡 MÉDIO|	❌			
|5|	Integrar SAST no pipeline (Semgrep)|	🟠 ALTO|	❌			
|6|	Integrar Dependency Scanning (Snyk)|	🟠 ALTO|	❌			
|7|	Integrar Container Scanning (Trivy)|	🟠 ALTO|	❌			
|8|	Configurar Binary Authorization|	🟠 ALTO|	❌			
|9|	Configurar Secret Manager|	🔴 CRÍTICO|	❌			
|10|	Migrar todas as secrets para Secret Manager|	🔴 CRÍTICO|	❌			
|11|	Configurar rotação automática de secrets|	🟡 MÉDIO|	❌			
|12|	Configurar audit logging de acesso a secrets|	🟠 ALTO|	❌			
|13|	Configurar Pod Security Standards (restricted)|	🔴 CRÍTICO|	❌			
|14|	Configurar Security Context (non-root, read-only fs)|	🟠 ALTO|	❌			
|15|	Configurar seccomp profiles|	🟡 MÉDIO|	❌			
|16|	Configurar alertas de SQL Injection	|🔴 CRÍTICO|	❌			
|17|	Configurar alertas de XSS|	🔴 CRÍTICO|	❌			
|18|	Configurar alertas de vulnerabilidade crítica|	🟠 ALTO|	❌			
|19|	Configurar alertas de API abuse|	🟡 MÉDIO|	❌			
|20|	Configurar alertas de secret acessada por humano|	🟠 ALTO|	❌

### 5. CAMADA 5: SEGURANÇA DE DADOS (Data Security)
|#|Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Ativar encryption at rest no Cloud SQL|	🔴 CRÍTICO|	❌			
|2|	Ativar encryption at rest no Cloud Storage|	🔴 CRÍTICO|	❌			
|3|	Ativar encryption at rest no Memorystore|	🔴 CRÍTICO|	❌			
|4|	Ativar encryption at rest no MongoDB|	🔴 CRÍTICO|	❌			
|5|	Configurar TLS 1.3 no Load Balancer|	🔴 CRÍTICO|	❌			
|6|	Configurar mTLS para comunicação interna|	🟠 ALTO|	❌			
|7|	Criar Cloud KMS Key Ring|	🔴 CRÍTICO|	❌			
|8|	Criar chave de criptografia para banco de dados|	🔴 CRÍTICO|	❌			
|9|	Criar chave de criptografia para backups|	🔴 CRÍTICO|	❌			
|10|	Criar chave de criptografia para aplicação|	🟠 ALTO|	❌			
|11|	Configurar rotação de chaves (90 dias)|	🟠 ALTO|	❌			
|12|	Configurar IAM para acesso às chaves|	🔴 CRÍTICO|	❌			
|13|	Implementar Data Masking para PII|	🟠 ALTO|	❌			
|14|	Implementar Tokenização para dados sensíveis|	🟡 MÉDIO|	❌			
|15|	Configurar backup criptografado|	🔴 CRÍTICO|	❌			
|16|	Configurar Point-in-Time Recovery (PITR)|	🟠 ALTO|	❌			
|17|	Configurar retenção de backups (30 dias)|	🟠 ALTO|	❌			
|18|	Configurar alertas de exfiltração de dados|	🔴 CRÍTICO|	❌			
|19|	Configurar alertas de chave KMS comprometida|	🔴 CRÍTICO|	❌			
|20|	Configurar alertas de backup falhou|	🟠 ALTO|	❌			
|21|	Configurar alertas de criptografia desabilitada|	🟠 ALTO|	❌			
|22|	Configurar alertas de acesso não autorizado a dados|	🔴 CRÍTICO|	❌

### 6. CAMADA 6: MONITORAMENTO E RESPOSTA (SIEM/SOAR)
|#|	Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Ativar Security Command Center (Premium)|	🔴 CRÍTICO|	❌			
|2|	Configurar Threat Detection no SCC|	🔴 CRÍTICO|	❌			
|3|	Configurar Vulnerability Scan no SCC|	🟠 ALTO	|❌			
|4|	Configurar Kafka para logs de segurança|	🔴 CRÍTICO|	❌			
|5|	Criar tópicos Kafka de segurança|	🟠 ALTO|	❌			
|6|	Configurar log sinks para Kafka|	🟠 ALTO|	❌			
|7|	Configurar Chronicle SIEM|	🟠 ALTO|	❌			
|8|	Configurar retenção de logs (90+ dias)|	🟠 ALTO	|❌			
|9|	Configurar Cloud Monitoring alert policies|	🔴 CRÍTICO|	❌			
|10|	Configurar uptime checks|	🟡 MÉDIO|	❌			
|11|	Configurar dashboards de segurança|	🟡 MÉDIO|	❌			
|12|	Configurar integração com PagerDuty|	🔴 CRÍTICO|	❌			
|13|	Configurar integração com Slack|	🟠 ALTO	|❌			
|14|	Configurar integração com E-mail|	🟡 MÉDIO|	❌			
|15|	Criar Plano de Resposta a Incidentes|	🔴 CRÍTICO|	❌			
|16|	Definir matriz de escalonamento|	🟠 ALTO|	❌			
|17|	Criar playbooks para cada tipo de incidente|	🟠 ALTO|	❌			
|18|	Realizar simulação de incidente|	🟡 MÉDIO|	❌			
|19|	Configurar alertas de IOC detectado|	🔴 CRÍTICO|	❌			
|20|	Configurar alertas de comportamento anômalo|	🟠 ALTO	|❌


### 7. CRIPTOGRAFIA E GESTÃO DE CHAVES

|#|	Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Gerar chaves RSA para JWT (2048 bits)|	🔴 CRÍTICO|	❌|			
|2|	Configurar rotação de chaves JWT|	🟠 ALTO|	❌|			
|3|	Configurar KMS para chaves mestras|	🔴 CRÍTICO	|❌			
|4|	Configurar envelope encryption|	🟡 MÉDIO|	❌			
|5|	Configurar CMEK (Customer-Managed Encryption Keys)|	🟠 ALTO|	❌			
|6|	Configurar rotação automática de chaves KMS|	🟠 ALTO|	❌			
|7|	Configurar backup das chaves mestras|	🔴 CRÍTICO|	❌			
|8|	Configurar auditoria de acesso a chaves|	🟠 ALTO|	❌

### 8. COMPLIANCE E GOVERNANÇA
|#|	Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Documentar política de segurança|	🔴 CRÍTICO|	❌			
|2|	Documentar política de retenção de dados|	🔴 CRÍTICO|	❌			
|3|	Documentar política de resposta a incidentes|	🔴 CRÍTICO|	❌			
|4|	Documentar política de backup e DR|	🟠 ALTO|	❌			
|5|	Realizar análise de impacto LGPD|	🔴 CRÍTICO|	❌			
|6|	Configurar retenção de dados conforme LGPD|	🔴 CRÍTICO|	❌			
|7|	Implementar direito ao esquecimento|	🟠 ALTO|	❌			
|8|	Configurar logs de consentimento|	🟡 MÉDIO|	❌			
|9|	Realizar auditoria de segurança trimestral|	🟠 ALTO|	❌			
|10|	Realizar pentest anual|	🟠 ALTO	|❌			
|11|	Configurar revisão de acessos mensal|	🟡 MÉDIO|	❌

### 9. TREINAMENTO E CONSCIENTIZAÇÃO
|#|	Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Treinar equipe em práticas de segurança|	🟠 ALTO|	❌			
|2|	Treinar equipe em resposta a incidentes|	🟠 ALTO|	❌			
|3|	Treinar desenvolvedores em OWASP Top 10|	🟡 MÉDIO|	❌			
|4|	Treinar equipe de operações em hardening|	🟡 MÉDIO|	❌			
|5|	Criar programa de conscientização de segurança|	🟡 MÉDIO|	❌			
|6|	Realizar simulação de phishing|	🟢 BAIXO|	❌


### 10. VALIDAÇÃO E TESTES
|#|	Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Testar regras do WAF (SQLi, XSS)|	🔴 CRÍTICO|	❌			
|2|	Testar Rate Limiting|	🟠 ALTO|	❌			
|3|	Testar autenticação MFA|	🔴 CRÍTICO|	❌			
|4|	Testar validação de JWT|	🔴 CRÍTICO|	❌			
|5|	Testar revogação de token|	🟠 ALTO|	❌			
|6|	Testar firewall rules|	🔴 CRÍTICO|	❌			
|7|	Testar VPC Service Controls|	🟠 ALTO|	❌			
|8|	Testar criptografia de dados|	🔴 CRÍTICO|	❌			
|9|	Testar backup e restore|	🟠 ALTO|	❌			
|10|	Testar alertas de segurança|	🟠 ALTO|	❌			
|11|	Realizar vulnerability scan|	🟠 ALTO|	❌			
|12|	Realizar pentest completo|	🔴 CRÍTICO|	❌


### 11. MANUTENÇÃO CONTÍNUA
#|	Item|	Prioridade|	Status|	Data|	Responsável|	Observações
|--|--|--|--|--|--|--
|1|	Revisar regras do WAF|	Mensal|	❌			
|2|	Revisar firewall rules|	Mensal|	❌			
|3|	Revisar IAM roles e permissões|	Mensal|	❌			
|4|	Revisar logs de segurança|	Semanal|	❌			
|5|	Rotacionar chaves KMS|	90 dias|	❌			
|6|	Rotacionar chaves JWT|	90 dias|	❌			
|7|	Atualizar dependências|	Semanal|	❌			
|8|	Revisar vulnerabilidades (SCC)|	Diário|	❌			
|9|	Revisar alertas de segurança|	Diário|	❌			
|10|	Atualizar playbooks de incidente|	Trimestral|	❌			
|11|	Realizar disaster recovery drill|	Trimestral|	❌			
|12|	Revisar políticas de segurança|	Semestral|	❌
 

### Resumo de Progresso
|Camada|	Total Itens|	Concluídos|	Em Andamento|	Pendentes|	% Concluído
|--|--|--|--|--|--
|1. Perímetro|	14|	0|	0|	14|	0%|
|2.Rede|	18|	0|	0|	18|	0%|
|3.Identidade|	22|	0|	0|	22|	0%|
|4.Aplicação|	20|	0|	0|	20|	0%|
|5.Dados|	22|	0|	0|	22|	0%|
|6.Monitoramento|	20|	0|	0|	20|	0%
|7. Criptografia|	8|	0|	0|	8|	0%
|8.Compliance|	11|	0|	0|	11|	0%
|9.Treinamento|	6|	0|	0|	6|	0%
|10.Validação|	12|	0|	0|	12|	0%
|11.Manutenção|	12|	0|	0|	12|	0%
|TOTAL|	165|	0|	0|	165|	0%


# Aprovações
|Papel	Nome|	  Assinatura|	Data
|--|--|--
__CISO__  
__Arquiteto de Segurança__
__Tech Lead__
__DevOps Lead__