# Scaffolding - API's e Workers em C# 14 com Circuit Breaker + Kubernetes Deployments 

### Sumário 



[1. Estrutura do Projeto](#estrutura)\
[2. buildingblocks - Projeto Compartilhado](#buildingblocks)\
[3. Auth API](#visao)\
[4. Lancamentos API](#visao)\
[5. Consolidacao API](#visao)\
[6. Relatorios API](#visao)\
[7. Workers](#visao)\
[8. Kubernetes Deployments (GKE)](#visao)


<a id="estrutura"></a>

### 1. Estrutura do Projeto

```
FluxoCaixa/
├── src/
│   ├── BuildingBlocks/
│   │   ├── FluxoCaixa.BuildingBlocks/
│   │   │   ├── Events/
│   │   │   ├── Interfaces/
│   │   │   ├── DTOs/
│   │   │   └── Extensions/
│   │   └── FluxoCaixa.CircuitBreaker/
│   │       ├── Policies/
│   │       └── Configurations/
│   ├── Services/
│   │   ├── Auth.Api/
│   │   ├── Lancamentos.Api/
│   │   ├── Consolidacao.Api/
│   │   ├── Relatorios.Api/
│   │   ├── Consolidacao.Worker/
│   │   ├── Notificacoes.Worker/
│   │   └── Auditoria.Worker/
│   └── Gateway/
│       └── ApiGateway/
├── deployments/
│   ├── auth-api.yaml
│   ├── lancamentos-api.yaml
│   ├── consolidacao-api.yaml
│   ├── relatorios-api.yaml
│   ├── consolidacao-worker.yaml
│   ├── notificacoes-worker.yaml
│   ├── auditoria-worker.yaml
│   └── api-gateway.yaml
└── Dockerfile
```



 

  

