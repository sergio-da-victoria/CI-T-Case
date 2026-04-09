# Fluxo de Caixa - POC CI&T
---
### Vale ressaltar que esta entrega constitui uma Prova de Conceito (PoC) voltada à validação funcional. A arquitetura definitiva foi rigorosamente projetada para o ambiente Cloud Goole, priorizando pilares críticos como resiliência, robustez estrutural e segurança da informação. É fundamental a leitura detalhada da documentação do 'Fluxo de Caixa', onde os recursos e protocolos de alta disponibilidade estão especificados. Para dúvidas técnicas ou esclarecimentos sobre a arquitetura, estou à disposição em: sergiodavictoria@hotmail.com 
---


### Procedimento para Instalação

git clone https://github.com/sergio-da-victoria/FluxoCaixa.git


    
##### O que é necessário para Instalação  ?

__1 Ter o Nginx ou Python para web__
__2 Instalação docker docker compose para gerar a image - container__
__3 Obter o Cliente do GIT__



- 1. git clone https://github.com/sergio-da-victoria/FluxoCaixa.git
- 2. após o git clone vá para FluxoCaixa 
cd FluxoCaixa
- 2. executar o (docker compose) - docker compose up -d  para API
- 3. executar para Web 
cd src/FluxoCaixa.Web
    npx http-server -p 7070 ou
    python3 -m http.server 7070

A Porta fica a seu Critério. - Lembre que a API está usando a porta 8080    


- 4. Para acesso o Swagger 
http://localhost:8080/swagger/index.html
![Swagger](/swagger.jpg)

- 5. Apenas Json
http://localhost:8080/swagger/v1/swagger.json
![Swagger Json](/swagger-json.jpg)
    

### Portas utilizadas
|Serviço|	Porta|	URL
|--|--|-
|Frontend Web|	7070|	http://localhost
|API Backend|	 8080|	http://localhost:8080
|Swagger UI|	7070|	http://localhost:8080/swagger



### 4. Resumo dos Endpoints
|Método|	Endpoint|	Descrição
|--|--|--
|POST|	/api/lancamentos|	Registrar novo lançamento
|GET|	/api/lancamentos|	Listar todos os lançamentos
|GET|	/api/lancamentos?inicio=&fim=|	Listar lançamentos por período
|GET|	/api/lancamentos/{id}|	Obter lançamento por ID
|DELETE|	/api/lancamentos/{id}|	Cancelar lançamento
|GET|	/api/lancamentos/estatisticas|	Obter estatísticas
|GET|	/api/consolidado/diario?data=|	Saldo diário
|GET|	/api/consolidado/semanal?ano=&semana=|	Saldo semanal
|GET|	/api/consolidado/mensal?ano=&mes=|	Saldo mensal
|GET|	/api/consolidado/periodo?inicio=&fim=|	Saldo por período
|GET|	/api/consolidado/resumo|	Resumo completo


### 5. Pagina Web
http://localhost:8080/swagger/v1/swagger.json
![Web Page](/pg-web.jpg)


### Poc - Construida em

```
.NET SDK:
 Version:           10.0.104
 Commit:            80d3e14f5e
 Workload version:  10.0.100-manifests.c7707153
 MSBuild version:   18.0.11+80d3e14f5

Runtime Environment:
 OS Name:     fedora
 OS Version:  43
 OS Platform: Linux
 RID:         fedora.43-x64
 Base Path:   /usr/lib64/dotnet/sdk/10.0.104/

.NET workloads installed:
There are no installed workloads to display.
Configured to use workload sets when installing new manifests.
No workload sets are installed. Run "dotnet workload restore" to install a workload set.

Host:
  Version:      10.0.4
  Architecture: x64
  Commit:       80d3e14f5e

.NET SDKs installed:
  10.0.104 [/usr/lib64/dotnet/sdk]

.NET runtimes installed:
  Microsoft.AspNetCore.App 10.0.4 [/usr/lib64/dotnet/shared/Microsoft.AspNetCore.App]
  Microsoft.NETCore.App 10.0.4 [/usr/lib64/dotnet/shared/Microsoft.NETCore.App]

Other architectures found:
  None

Environment variables:
  DOTNET_BUNDLE_EXTRACT_BASE_DIR           [/home/sergio/.cache/dotnet_bundle_extract]
  DOTNET_ROOT                              [/usr/lib64/dotnet]

global.json file:
  Not found

Learn more:
  https://aka.ms/dotnet/info

```