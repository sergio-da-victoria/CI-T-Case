# CI/CD Pipeline - Google Cloud Build, Cloud Deploy, Artifact Registry com TDD e DBB


# Sumário

[1. Visão Geral da Pipeline](#geral)\
[2. Estrutura do Repositório](#repositorio)\
[3. Testes TDD (Test-Driven Development)](#teste)\
[4. Database Build (DBB)](#database)\
[5. Google Cloud Build - Configuração](#build)\
[6. Artifact Registry - Configuração](#registry)\
[7. Google Cloud Deploy - Configuração](#deply)\
[8. Scripts de Deploy](#script)\
[9. Pipeline Completa](#pipline)\
[10. Monitoramento e Alertas](#monitoramento)


<a id="geral"></a>
### 1. Visão Geral da Pipeline

```

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         CI/CD PIPELINE - FLUXO DE CAIXA                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │  Commit  │───▶│  Build  │──▶│  Test    │──▶│  DBB     │──▶│  Package │       │
│  │  GitHub  │    │  .NET    │    │  TDD     │    │  Migrate │    │  Docker  │       │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘       │
│                                                                   │                 │
│                                                                   ▼                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │  Deploy  │◄───│  Approve │◄───│  Render  │◄───│  Push    │◄───│  Store   │       │
│  │  Cloud   │    │  Manual  │    │ Manifests│    │  AR      │    │  Test    │       │
│  │  Deploy  │    │          │    │          │    │          │    │          │       │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘       │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Etapas da Pipeline

|Fase|	Responsabilidade|	Ferramenta
|--|--|--
|Build|	Compilar código C#|	.NET 10 SDK|
|Test TDD|	Executar testes unitários e integração|	xUnit, NSubstitute
|DBB|	Migrações de banco de dados|	Flyway/EF Core Migrations
|Package|	Criar imagem Docker|	Docker Build
|Artifact Registry|	Armazenar imagens|	Google Artifact Registry
|Render|	Gerar manifests K8s|	kustomize/helm
|Approve|	Aprovação manual|	Cloud Deploy
|Deploy|	Implantar no GKE|	Google Cloud Deploy


<a id="repositorio"></a>
### 2. Estrutura do Repositório

```
fluxo-caixa/
├── .github/
│   └── workflows/
│       └── ci-cd.yaml
├── src/
│   ├── FluxoCaixa.AuthApi/
│   ├── FluxoCaixa.LancamentosApi/
│   ├── FluxoCaixa.ConsolidacaoApi/
│   ├── FluxoCaixa.RelatoriosApi/
│   ├── FluxoCaixa.ConsolidacaoWorker/
│   ├── FluxoCaixa.NotificacoesWorker/
│   └── FluxoCaixa.AuditoriaWorker/
├── tests/
│   ├── FluxoCaixa.Tests.Unit/
│   ├── FluxoCaixa.Tests.Integration/
│   └── FluxoCaixa.Tests.E2E/
├── database/
│   ├── migrations/
│   │   ├── V1__Create_usuarios_table.sql
│   │   ├── V2__Create_lancamentos_table.sql
│   │   └── V3__Create_saldo_diario_table.sql
│   └── seed/
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   ├── overlays/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   └── skaffold.yaml
├── cloudbuild/
│   ├── cloudbuild.yaml
│   ├── cloudbuild-test.yaml
│   ├── cloudbuild-db.yaml
│   └── cloudbuild-deploy.yaml
├── scripts/
│   ├── build.sh
│   ├── test.sh
│   ├── db-migrate.sh
│   └── deploy.sh
├── Dockerfile
├── .dockerignore
└── README.md
```

<a id="teste"></a>
### 3. Testes TDD (Test-Driven Development)
### 3.1 Projeto de Testes Unitários

```
<!-- tests/FluxoCaixa.Tests.Unit/FluxoCaixa.Tests.Unit.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="coverlet.collector" Version="6.0.2" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.12.0" />
    <PackageReference Include="xunit" Version="2.9.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageReference Include="NSubstitute" Version="5.3.0" />
    <PackageReference Include="FluentAssertions" Version="6.12.2" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="../../src/FluxoCaixa.LancamentosApi/FluxoCaixa.LancamentosApi.csproj" />
  </ItemGroup>

</Project>

```

### 3.2 Teste Unitário - LancamentoService (TDD)

```
// tests/FluxoCaixa.Tests.Unit/Services/LancamentoServiceTests.cs
using FluentAssertions;
using NSubstitute;
using NSubstitute.ExceptionExtensions;
using Polly;
using Xunit;

namespace FluxoCaixa.Tests.Unit.Services;

public class LancamentoServiceTests
{
    private readonly LancamentoService _sut;
    private readonly NpgsqlDataSource _dataSource;
    private readonly IKafkaProducer<string, LancamentoRegistradoEvent> _kafkaProducer;
    private readonly IAsyncPolicy _dbCircuitBreaker;
    private readonly IAsyncPolicy _kafkaCircuitBreaker;
    private readonly ILogger<LancamentoService> _logger;

    public LancamentoServiceTests()
    {
        _dataSource = Substitute.For<NpgsqlDataSource>();
        _kafkaProducer = Substitute.For<IKafkaProducer<string, LancamentoRegistradoEvent>>();
        _dbCircuitBreaker = Policy.NoOpAsync();
        _kafkaCircuitBreaker = Policy.NoOpAsync();
        _logger = Substitute.For<ILogger<LancamentoService>>();
        
        _sut = new LancamentoService(
            _dataSource, 
            _kafkaProducer, 
            _dbCircuitBreaker, 
            _kafkaCircuitBreaker, 
            _logger);
    }

    [Fact]
    public async Task RegistrarLancamentoAsync_DeveRegistrarComSucesso_QuandoDadosValidos()
    {
        // Arrange - TDD: Preparar os dados de entrada
        var request = new LancamentoRequestDto
        {
            Valor = 500.00m,
            Tipo = "CREDITO",
            Descricao = "Venda de produtos",
            Categoria = "VENDAS",
            Estabelecimento = "Loja Matriz"
        };
        var usuarioId = Guid.NewGuid();
        
        var expectedResponse = new LancamentoResponseDto
        {
            Id = Guid.NewGuid(),
            Valor = 500.00m,
            Tipo = "CREDITO",
            Status = "CONFIRMADO"
        };
        
        // Mock do banco de dados
        var mockConnection = Substitute.For<NpgsqlConnection>();
        var mockCommand = Substitute.For<NpgsqlCommand>();
        _dataSource.OpenConnectionAsync(Arg.Any<CancellationToken>())
            .Returns(Task.FromResult(mockConnection));
        
        // Act - Executar o método testado
        var result = await _sut.RegistrarLancamentoAsync(request, usuarioId);
        
        // Assert - Verificar o resultado
        result.Should().NotBeNull();
        result.Tipo.Should().Be("CREDITO");
        result.Status.Should().Be("CONFIRMADO");
        result.Valor.Should().Be(500.00m);
        
        // Verificar se o evento foi publicado
        await _kafkaProducer.Received(1).ProduceAsync(
            Arg.Any<string>(),
            Arg.Any<string>(),
            Arg.Any<LancamentoRegistradoEvent>(),
            Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task RegistrarLancamentoAsync_DeveLancarExcecao_QuandoValorInvalido()
    {
        // Arrange - Valor negativo
        var request = new LancamentoRequestDto
        {
            Valor = -100.00m,
            Tipo = "CREDITO",
            Descricao = "Valor inválido",
            Categoria = "TESTE"
        };
        var usuarioId = Guid.NewGuid();
        
        // Act & Assert
        await Assert.ThrowsAsync<ArgumentException>(
            () => _sut.RegistrarLancamentoAsync(request, usuarioId));
    }

    [Fact]
    public async Task RegistrarLancamentoAsync_DeveLancarExcecao_QuandoTipoInvalido()
    {
        // Arrange - Tipo inválido
        var request = new LancamentoRequestDto
        {
            Valor = 100.00m,
            Tipo = "INVALIDO",
            Descricao = "Tipo inválido",
            Categoria = "TESTE"
        };
        var usuarioId = Guid.NewGuid();
        
        // Act & Assert
        await Assert.ThrowsAsync<ArgumentException>(
            () => _sut.RegistrarLancamentoAsync(request, usuarioId));
    }

    [Fact]
    public async Task RegistrarLancamentoAsync_DeveAplicarCircuitBreaker_QuandoBancoIndisponivel()
    {
        // Arrange
        var request = new LancamentoRequestDto
        {
            Valor = 500.00m,
            Tipo = "CREDITO",
            Descricao = "Teste circuit breaker",
            Categoria = "TESTE"
        };
        var usuarioId = Guid.NewGuid();
        
        // Mock para simular falha no banco
        _dataSource.OpenConnectionAsync(Arg.Any<CancellationToken>())
            .ThrowsAsync(new NpgsqlException("Connection failed"));
        
        // Act & Assert
        await Assert.ThrowsAsync<NpgsqlException>(
            () => _sut.RegistrarLancamentoAsync(request, usuarioId));
    }

    [Theory]
    [InlineData(100, "DEBITO")]
    [InlineData(250.50, "DEBITO")]
    [InlineData(1000, "CREDITO")]
    public async Task RegistrarLancamentoAsync_DeveAceitarMultiplosValores_QuandoValido(decimal valor, string tipo)
    {
        // Arrange
        var request = new LancamentoRequestDto
        {
            Valor = valor,
            Tipo = tipo,
            Descricao = "Teste parametrizado",
            Categoria = "TESTE"
        };
        var usuarioId = Guid.NewGuid();
        
        // Mock do banco de dados
        var mockConnection = Substitute.For<NpgsqlConnection>();
        _dataSource.OpenConnectionAsync(Arg.Any<CancellationToken>())
            .Returns(Task.FromResult(mockConnection));
        
        // Act
        var result = await _sut.RegistrarLancamentoAsync(request, usuarioId);
        
        // Assert
        result.Should().NotBeNull();
        result.Valor.Should().Be(valor);
        result.Tipo.Should().Be(tipo);
    }
}
```

### 3.3 Teste de Integração

```
// tests/FluxoCaixa.Tests.Integration/DatabaseTests.cs
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.DependencyInjection;
using Npgsql;
using Respawn;
using Xunit;

namespace FluxoCaixa.Tests.Integration;

public class DatabaseTests : IClassFixture<WebApplicationFactory<Program>>, IAsyncLifetime
{
    private readonly WebApplicationFactory<Program> _factory;
    private Respawner _respawner;
    private NpgsqlConnection _connection;

    public DatabaseTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    public async Task InitializeAsync()
    {
        _connection = new NpgsqlConnection(
            "Host=localhost;Database=fluxocaixa_test;Username=postgres;Password=postgres");
        await _connection.OpenAsync();
        
        _respawner = await Respawner.CreateAsync(_connection, new RespawnerOptions
        {
            TablesToIgnore = new Respawn.Graph.Table[] { "__EFMigrationsHistory" }
        });
    }

    public async Task DisposeAsync()
    {
        await _respawner.ResetAsync(_connection);
        await _connection.CloseAsync();
    }

    [Fact]
    public async Task SaldoDiario_DeveSerAtualizado_AposRegistroDeLancamento()
    {
        // Arrange
        using var scope = _factory.Services.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        
        var lancamento = new Lancamento
        {
            Id = Guid.NewGuid(),
            Valor = 500.00m,
            Tipo = "CREDITO",
            DataHora = DateTime.UtcNow,
            Descricao = "Teste integração",
            UsuarioId = Guid.NewGuid(),
            Status = "CONFIRMADO"
        };
        
        // Act
        dbContext.Lancamentos.Add(lancamento);
        await dbContext.SaveChangesAsync();
        
        // Assert
        var salvo = await dbContext.Lancamentos.FindAsync(lancamento.Id);
        salvo.Should().NotBeNull();
        salvo.Valor.Should().Be(500.00m);
        salvo.Status.Should().Be("CONFIRMADO");
    }
}

```

### 3.4 Teste de Contrato de API

```
// tests/FluxoCaixa.Tests.E2E/ApiContractTests.cs
using System.Net;
using System.Net.Http.Json;
using FluentAssertions;
using Xunit;

namespace FluxoCaixa.Tests.E2E;

[Collection("ApiTests")]
public class ApiContractTests : IClassFixture<ApiTestFixture>
{
    private readonly HttpClient _client;
    private readonly ApiTestFixture _fixture;

    public ApiContractTests(ApiTestFixture fixture)
    {
        _fixture = fixture;
        _client = fixture.CreateClient();
    }

    [Fact]
    public async Task PostLancamentos_DeveRetornar201_QuandoDadosValidos()
    {
        // Arrange
        var request = new
        {
            valor = 500.00m,
            tipo = "CREDITO",
            descricao = "Venda de produtos",
            categoria = "VENDAS",
            estabelecimento = "Loja Matriz"
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/lancamentos", request);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        
        var content = await response.Content.ReadFromJsonAsync<LancamentoResponseDto>();
        content.Should().NotBeNull();
        content.Valor.Should().Be(500.00m);
    }

    [Fact]
    public async Task GetConsolidadoDiario_DeveRetornar200_QuandoDataValida()
    {
        // Arrange
        var data = DateOnly.FromDateTime(DateTime.Today);
        
        // Act
        var response = await _client.GetAsync($"/api/consolidado/diario?data={data:yyyy-MM-dd}");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var content = await response.Content.ReadFromJsonAsync<SaldoDiarioResponseDto>();
        content.Should().NotBeNull();
    }

    [Fact]
    public async Task DeleteLancamento_DeveRetornar200_QuandoIdValido()
    {
        // Arrange - Primeiro criar um lançamento
        var createRequest = new
        {
            valor = 100.00m,
            tipo = "DEBITO",
            descricao = "Para cancelar",
            categoria = "TESTE"
        };
        
        var createResponse = await _client.PostAsJsonAsync("/api/lancamentos", createRequest);
        var created = await createResponse.Content.ReadFromJsonAsync<LancamentoResponseDto>();
        
        // Act
        var deleteResponse = await _client.DeleteAsync($"/api/lancamentos/{created.Id}");
        
        // Assert
        deleteResponse.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}

```

### 3.5 Script de Execução de Testes


```
#!/bin/bash
# scripts/test.sh

set -e

echo "========================================="
echo "Executando Testes TDD - Fluxo de Caixa"
echo "========================================="

# Cores para output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Função para executar testes com cobertura
run_tests_with_coverage() {
    local project_path=$1
    local test_name=$2
    
    echo -e "${YELLOW}Executando testes: $test_name${NC}"
    
    dotnet test $project_path \
        --logger:"console;verbosity=detailed" \
        --collect:"XPlat Code Coverage" \
        --settings coverlet.runsettings \
        --results-directory ./TestResults
    
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}✓ $test_name passou${NC}"
    else
        echo -e "${RED}✗ $test_name falhou${NC}"
        exit 1
    fi
}

# Executar testes unitários
echo -e "\n${YELLOW}[1/4] Testes Unitários${NC}"
run_tests_with_coverage "tests/FluxoCaixa.Tests.Unit" "Unit Tests"

# Executar testes de integração
echo -e "\n${YELLOW}[2/4] Testes de Integração${NC}"
run_tests_with_coverage "tests/FluxoCaixa.Tests.Integration" "Integration Tests"

# Executar testes E2E
echo -e "\n${YELLOW}[3/4] Testes E2E${NC}"
run_tests_with_coverage "tests/FluxoCaixa.Tests.E2E" "E2E Tests"

# Gerar relatório de cobertura
echo -e "\n${YELLOW}[4/4] Gerando relatório de cobertura${NC}"
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator \
    -reports:./TestResults/**/coverage.cobertura.xml \
    -targetdir:./CoverageReport \
    -reporttypes:Html;Badges

echo -e "\n${GREEN}=========================================${NC}"
echo -e "${GREEN}✓ Todos os testes passaram com sucesso!${NC}"
echo -e "${GREEN}Relatório de cobertura: ./CoverageReport/index.html${NC}"
echo -e "${GREEN}=========================================${NC}"

```

<a id="database"></a>
### 4. Database Build (DBB)

### 4.1 Migrações Flyway

__Usuários__

```
-- database/migrations/V1__Create_usuarios_table.sql
CREATE TABLE IF NOT EXISTS usuarios (
    id UUID PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    senha_hash VARCHAR(255),
    cognito_sub VARCHAR(255) UNIQUE,
    ad_sid VARCHAR(255) UNIQUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP
);

CREATE INDEX idx_usuarios_email ON usuarios(email);
CREATE INDEX idx_usuarios_cognito_sub ON usuarios(cognito_sub);

```

__Lançamentos__
```
-- database/migrations/V2__Create_lancamentos_table.sql
CREATE TABLE IF NOT EXISTS lancamentos (
    id UUID PRIMARY KEY,
    valor DECIMAL(15,2) NOT NULL,
    tipo VARCHAR(10) NOT NULL CHECK (tipo IN ('DEBITO', 'CREDITO')),
    data_hora TIMESTAMP NOT NULL,
    descricao TEXT,
    categoria VARCHAR(50),
    usuario_id UUID NOT NULL REFERENCES usuarios(id),
    estabelecimento VARCHAR(100),
    status VARCHAR(20) DEFAULT 'CONFIRMADO',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    cancelled_at TIMESTAMP
);

CREATE INDEX idx_lancamentos_usuario_data ON lancamentos(usuario_id, data_hora);
CREATE INDEX idx_lancamentos_status ON lancamentos(status);
CREATE INDEX idx_lancamentos_categoria ON lancamentos(categoria);
```

__Saldo__
```
-- database/migrations/V3__Create_saldo_diario_table.sql
CREATE TABLE IF NOT EXISTS saldo_diario (
    data DATE PRIMARY KEY,
    total_creditos DECIMAL(15,2) NOT NULL DEFAULT 0,
    total_debitos DECIMAL(15,2) NOT NULL DEFAULT 0,
    saldo DECIMAL(15,2) NOT NULL DEFAULT 0,
    quantidade_transacoes INTEGER NOT NULL DEFAULT 0,
    ultima_atualizacao TIMESTAMP NOT NULL DEFAULT NOW(),
    versao INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_saldo_diario_data ON saldo_diario(data DESC);
CREATE INDEX idx_saldo_diario_saldo ON saldo_diario(saldo);

```

### 4.2 Script de Migração do Banco

```

#!/bin/bash
# scripts/db-migrate.sh

set -e

echo "========================================="
echo "Database Build (DBB) - Migrações"
echo "========================================="

# Configurações
DB_HOST=${DB_HOST:-localhost}
DB_PORT=${DB_PORT:-5432}
DB_NAME=${DB_NAME:-fluxocaixa}
DB_USER=${DB_USER:-postgres}
DB_PASSWORD=${DB_PASSWORD:-postgres}

# Função para executar migração
run_migration() {
    local environment=$1
    
    echo -e "\nExecutando migrações para ambiente: $environment"
    
    # Usar Flyway para migrações
    flyway \
        -url="jdbc:postgresql://$DB_HOST:$DB_PORT/$DB_NAME" \
        -user="$DB_USER" \
        -password="$DB_PASSWORD" \
        -locations="filesystem:./database/migrations" \
        -table="flyway_schema_history" \
        migrate
    
    if [ $? -eq 0 ]; then
        echo "✓ Migrações aplicadas com sucesso para $environment"
    else
        echo "✗ Falha ao aplicar migrações para $environment"
        exit 1
    fi
}

# Executar migrações por ambiente
case "${1}" in
    dev)
        export DB_NAME="fluxocaixa_dev"
        run_migration "dev"
        ;;
    staging)
        export DB_NAME="fluxocaixa_staging"
        run_migration "staging"
        ;;
    prod)
        echo "Aplicando migrações em PRODUÇÃO..."
        read -p "Confirmar? (yes/no): " confirm
        if [ "$confirm" != "yes" ]; then
            echo "Migração cancelada"
            exit 0
        fi
        export DB_NAME="fluxocaixa_prod"
        run_migration "prod"
        ;;
    *)
        echo "Uso: $0 {dev|staging|prod}"
        exit 1
        ;;
esac

echo -e "\n✓ Database Build concluído com sucesso!"
```

<a id="build"></a>
### 5. Google Cloud Build - Configuração

### 5.1 cloudbuild.yaml (Pipeline Principal)

```
# cloudbuild/cloudbuild.yaml
steps:
  # Step 1: Setup .NET SDK
  - name: 'mcr.microsoft.com/dotnet/sdk:10.0'
    id: 'setup-dotnet'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        echo "✅ .NET 10 SDK configurado"
        dotnet --version

  # Step 2: Restore dependencies
  - name: 'mcr.microsoft.com/dotnet/sdk:10.0'
    id: 'dotnet-restore'
    entrypoint: 'dotnet'
    args: ['restore', 'FluxoCaixa.sln']

  # Step 3: Build solution
  - name: 'mcr.microsoft.com/dotnet/sdk:10.0'
    id: 'dotnet-build'
    entrypoint: 'dotnet'
    args: ['build', 'FluxoCaixa.sln', '--configuration', 'Release', '--no-restore']

  # Step 4: Run Unit Tests (TDD)
  - name: 'mcr.microsoft.com/dotnet/sdk:10.0'
    id: 'dotnet-test-unit'
    entrypoint: 'dotnet'
    args: [
      'test', 'tests/FluxoCaixa.Tests.Unit/FluxoCaixa.Tests.Unit.csproj',
      '--configuration', 'Release',
      '--logger', 'trx',
      '--results-directory', '/workspace/test-results/unit'
    ]

  # Step 5: Run Integration Tests
  - name: 'mcr.microsoft.com/dotnet/sdk:10.0'
    id: 'dotnet-test-integration'
    entrypoint: 'dotnet'
    args: [
      'test', 'tests/FluxoCaixa.Tests.Integration/FluxoCaixa.Tests.Integration.csproj',
      '--configuration', 'Release',
      '--logger', 'trx',
      '--results-directory', '/workspace/test-results/integration'
    ]
    env:
      - 'ConnectionStrings__ReadDb=Host=cloudsql;Database=fluxocaixa_test;Username=postgres;Password=${_DB_PASSWORD}'

  # Step 6: Run E2E Tests
  - name: 'mcr.microsoft.com/dotnet/sdk:10.0'
    id: 'dotnet-test-e2e'
    entrypoint: 'dotnet'
    args: [
      'test', 'tests/FluxoCaixa.Tests.E2E/FluxoCaixa.Tests.E2E.csproj',
      '--configuration', 'Release',
      '--logger', 'trx',
      '--results-directory', '/workspace/test-results/e2e'
    ]

  # Step 7: Build Docker images
  - name: 'gcr.io/cloud-builders/docker'
    id: 'docker-build-auth'
    args: [
      'build', 
      '-t', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/auth-api:${SHORT_SHA}',
      '-t', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/auth-api:latest',
      '-f', 'src/FluxoCaixa.AuthApi/Dockerfile',
      '.'
    ]

  - name: 'gcr.io/cloud-builders/docker'
    id: 'docker-build-lancamentos'
    args: [
      'build', 
      '-t', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/lancamentos-api:${SHORT_SHA}',
      '-t', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/lancamentos-api:latest',
      '-f', 'src/FluxoCaixa.LancamentosApi/Dockerfile',
      '.'
    ]

  - name: 'gcr.io/cloud-builders/docker'
    id: 'docker-build-consolidacao'
    args: [
      'build', 
      '-t', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-api:${SHORT_SHA}',
      '-t', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-api:latest',
      '-f', 'src/FluxoCaixa.ConsolidacaoApi/Dockerfile',
      '.'
    ]

  - name: 'gcr.io/cloud-builders/docker'
    id: 'docker-build-worker'
    args: [
      'build', 
      '-t', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-worker:${SHORT_SHA}',
      '-t', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-worker:latest',
      '-f', 'src/FluxoCaixa.ConsolidacaoWorker/Dockerfile',
      '.'
    ]

  # Step 8: Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    id: 'docker-push'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        docker push ${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/auth-api:${SHORT_SHA}
        docker push ${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/lancamentos-api:${SHORT_SHA}
        docker push ${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-api:${SHORT_SHA}
        docker push ${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-worker:${SHORT_SHA}

  # Step 9: Database Migration (DBB)
  - name: 'gcr.io/cloud-builders/gcloud'
    id: 'database-migrate'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        gcloud sql connect ${_CLOUD_SQL_INSTANCE} --user=postgres --database=${_DB_NAME} \
          < database/migrations/V1__Create_usuarios_table.sql
        gcloud sql connect ${_CLOUD_SQL_INSTANCE} --user=postgres --database=${_DB_NAME} \
          < database/migrations/V2__Create_lancamentos_table.sql
        gcloud sql connect ${_CLOUD_SQL_INSTANCE} --user=postgres --database=${_DB_NAME} \
          < database/migrations/V3__Create_saldo_diario_table.sql

  # Step 10: Render Kubernetes manifests
  - name: 'gcr.io/k8s-skaffold/skaffold:latest'
    id: 'render-manifests'
    entrypoint: 'skaffold'
    args: ['render', '--output', 'rendered.yaml']

  # Step 11: Deploy to Cloud Deploy
  - name: 'gcr.io/google.com/cloudsdktool/google-cloud-cli'
    id: 'cloud-deploy'
    entrypoint: 'gcloud'
    args: [
      'deploy', 'releases', 'create', 'rel-${SHORT_SHA}',
      '--delivery-pipeline', 'fluxo-caixa-pipeline',
      '--region', '${_REGION}',
      '--source', '.',
      '--images', 'auth-api=${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/auth-api:${SHORT_SHA}',
      '--images', 'lancamentos-api=${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/lancamentos-api:${SHORT_SHA}',
      '--images', 'consolidacao-api=${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-api:${SHORT_SHA}',
      '--images', 'consolidacao-worker=${_REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-worker:${SHORT_SHA}'
    ]

# Substituições
substitutions:
  _REGION: us-central1
  _CLOUD_SQL_INSTANCE: fluxo-caixa-db
  _DB_NAME: fluxocaixa_prod

# Artifacts a serem salvos
artifacts:
  objects:
    location: 'gs://${PROJECT_ID}-build-artifacts/${SHORT_SHA}/'
    paths:
      - 'test-results/**'
      - 'rendered.yaml'

# Logs
logsBucket: 'gs://${PROJECT_ID}-build-logs'

# Timeout
timeout: 3600s

# Options
options:
  machineType: 'E2_HIGHCPU_8'
  diskSizeGb: 100
  
  
### 6. Artifact Registry - Configuração
### 6.1 Dockerfile Multi-Stage

```

<a id="registry"></a>
### 6. Artifact Registry - Configuração

### 6.1 Dockerfile Multi-Stage


```
# Dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy solution and project files
COPY FluxoCaixa.sln .
COPY src/FluxoCaixa.AuthApi/*.csproj src/FluxoCaixa.AuthApi/
COPY src/FluxoCaixa.LancamentosApi/*.csproj src/FluxoCaixa.LancamentosApi/
COPY src/FluxoCaixa.ConsolidacaoApi/*.csproj src/FluxoCaixa.ConsolidacaoApi/
COPY src/FluxoCaixa.ConsolidacaoWorker/*.csproj src/FluxoCaixa.ConsolidacaoWorker/
COPY tests/FluxoCaixa.Tests.Unit/*.csproj tests/FluxoCaixa.Tests.Unit/
COPY tests/FluxoCaixa.Tests.Integration/*.csproj tests/FluxoCaixa.Tests.Integration/

# Restore dependencies
RUN dotnet restore FluxoCaixa.sln

# Copy all source code
COPY . .

# Build and publish
RUN dotnet publish src/FluxoCaixa.AuthApi/FluxoCaixa.AuthApi.csproj \
    -c Release -o /app/auth-api --no-restore

RUN dotnet publish src/FluxoCaixa.LancamentosApi/FluxoCaixa.LancamentosApi.csproj \
    -c Release -o /app/lancamentos-api --no-restore

RUN dotnet publish src/FluxoCaixa.ConsolidacaoApi/FluxoCaixa.ConsolidacaoApi.csproj \
    -c Release -o /app/consolidacao-api --no-restore

RUN dotnet publish src/FluxoCaixa.ConsolidacaoWorker/FluxoCaixa.ConsolidacaoWorker.csproj \
    -c Release -o /app/consolidacao-worker --no-restore

# Stage 2: Runtime - Auth API
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS auth-api
WORKDIR /app
COPY --from=build /app/auth-api .
EXPOSE 8081
ENTRYPOINT ["dotnet", "FluxoCaixa.AuthApi.dll"]

# Stage 2: Runtime - Lancamentos API
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS lancamentos-api
WORKDIR /app
COPY --from=build /app/lancamentos-api .
EXPOSE 8082
ENTRYPOINT ["dotnet", "FluxoCaixa.LancamentosApi.dll"]

# Stage 2: Runtime - Consolidacao API
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS consolidacao-api
WORKDIR /app
COPY --from=build /app/consolidacao-api .
EXPOSE 8083
ENTRYPOINT ["dotnet", "FluxoCaixa.ConsolidacaoApi.dll"]

# Stage 2: Runtime - Worker
FROM mcr.microsoft.com/dotnet/runtime:10.0 AS consolidacao-worker
WORKDIR /app
COPY --from=build /app/consolidacao-worker .
ENTRYPOINT ["dotnet", "FluxoCaixa.ConsolidacaoWorker.dll"]


```

### 6.2 Script de Criação do Artifact Registry

```
#!/bin/bash
# scripts/create-artifact-registry.sh

set -e

PROJECT_ID=${PROJECT_ID:-$(gcloud config get-value project)}
REGION=${REGION:-us-central1}
REPO_NAME=${REPO_NAME:-fluxo-caixa}

echo "========================================="
echo "Criando Artifact Registry"
echo "========================================="

# Criar repositório
gcloud artifacts repositories create $REPO_NAME \
    --repository-format=docker \
    --location=$REGION \
    --description="Fluxo Caixa Docker Images" \
    --labels=environment=production,team=backend

# Configurar políticas de retenção
gcloud artifacts repositories set-iam-policy $REPO_NAME \
    --location=$REGION \
    policy.yaml

# Criar política de limpeza
cat << EOF | gcloud artifacts repositories add-iam-policy-binding $REPO_NAME --location=$REGION --member='user:team@example.com' --role='roles/artifactregistry.writer'
EOF

echo "✅ Artifact Registry criado: ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}"

# Listar imagens
gcloud artifacts docker images list ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}

```
<a id="deploy"></a>
### 7. Google Cloud Deploy - Configuração


### 7.1 Delivery Pipeline

```
# clouddeploy/delivery-pipeline.yaml
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: fluxo-caixa-pipeline
description: Delivery pipeline for Fluxo Caixa application
serialPipeline:
  stages:
  - targetId: dev
    profiles: [dev]
    strategy:
      standard:
        verify: true
  - targetId: staging
    profiles: [staging]
    strategy:
      standard:
        verify: true
    deployParameters:
    - values:
        minReplicas: "2"
        maxReplicas: "5"
  - targetId: prod
    profiles: [prod]
    strategy:
      canary:
        runtimeConfig:
          kubernetes:
            serviceNetworking:
              service: fluxo-caixa-svc
              deployment: fluxo-caixa
        canaryDeployment:
          steps:
          - percent: 5
          - percent: 25
          - percent: 50
          - percent: 100
    deployParameters:
    - values:
        minReplicas: "3"
        maxReplicas: "10"

```

### 7.2 Targets

```
clouddeploy/targets.yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: dev
description: Development environment
gke:
  cluster: projects/${PROJECT_ID}/locations/us-central1/clusters/fluxo-caixa-dev
  internalIp: true
---

apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: staging
description: Staging environment
gke:
  cluster: projects/${PROJECT_ID}/locations/us-central1/clusters/fluxo-caixa-staging
  internalIp: true
---

apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
description: Production environment
gke:
  cluster: projects/${PROJECT_ID}/locations/us-central1/clusters/fluxo-caixa-prod
  internalIp: true
requireApproval: true
```

### 7.3 Skaffold Configuration

```
# k8s/skaffold.yaml
apiVersion: skaffold/v4beta11
kind: Config
metadata:
  name: fluxo-caixa
build:
  artifacts:
  - image: auth-api
    context: .
    docker:
      dockerfile: src/FluxoCaixa.AuthApi/Dockerfile
    sync:
      manual:
      - src: 'src/FluxoCaixa.AuthApi/**/*.cs'
        dest: .
  - image: lancamentos-api
    context: .
    docker:
      dockerfile: src/FluxoCaixa.LancamentosApi/Dockerfile
  - image: consolidacao-api
    context: .
    docker:
      dockerfile: src/FluxoCaixa.ConsolidacaoApi/Dockerfile
  - image: consolidacao-worker
    context: .
    docker:
      dockerfile: src/FluxoCaixa.ConsolidacaoWorker/Dockerfile
  tagPolicy:
    gitCommit: {}
  local:
    push: false
deploy:
  kubectl:
    manifests:
    - k8s/base/deployment.yaml
    - k8s/base/service.yaml
    - k8s/base/configmap.yaml
profiles:
- name: dev
  manifests:
    rawYaml:
    - k8s/overlays/dev/deployment-patch.yaml
- name: staging
  manifests:
    rawYaml:
    - k8s/overlays/staging/deployment-patch.yaml
- name: prod
  manifests:
    rawYaml:
    - k8s/overlays/prod/deployment-patch.yaml

```
<a id="script"></a>
### 8. Scripts de Deploy

### 8.1 Deploy Principal
```
#!/bin/bash
# scripts/deploy.sh

set -e

ENVIRONMENT=${1:-dev}
PROJECT_ID=$(gcloud config get-value project)
REGION=${REGION:-us-central1}
SHORT_SHA=$(git rev-parse --short HEAD)

echo "========================================="
echo "Deploying Fluxo Caixa - Environment: $ENVIRONMENT"
echo "========================================="

# Função para deploy
deploy_to_environment() {
    local env=$1
    local approve=${2:-false}
    
    echo "Deploying to $env..."
    
    # Criar release no Cloud Deploy
    RELEASE_ID="rel-${SHORT_SHA}-${env}"
    
    gcloud deploy releases create $RELEASE_ID \
        --delivery-pipeline=fluxo-caixa-pipeline \
        --region=$REGION \
        --target=$env \
        --images=auth-api=${REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/auth-api:${SHORT_SHA},lancamentos-api=${REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/lancamentos-api:${SHORT_SHA},consolidacao-api=${REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-api:${SHORT_SHA},consolidacao-worker=${REGION}-docker.pkg.dev/${PROJECT_ID}/fluxo-caixa/consolidacao-worker:${SHORT_SHA}
    
    if [ "$approve" = true ]; then
        gcloud deploy releases approve $RELEASE_ID \
            --delivery-pipeline=fluxo-caixa-pipeline \
            --region=$REGION \
            --target=$env
    fi
    
    echo "✅ Deploy to $env completed"
}

# Deploy baseado no ambiente
case $ENVIRONMENT in
    dev)
        deploy_to_environment "dev" false
        ;;
    staging)
        deploy_to_environment "staging" false
        ;;
    prod)
        echo "🚨 DEPLOY PARA PRODUÇÃO 🚨"
        read -p "Confirmar deploy para PRODUÇÃO? (yes/no): " confirm
        if [ "$confirm" = "yes" ]; then
            deploy_to_environment "prod" true
        else
            echo "Deploy cancelado"
            exit 0
        fi
        ;;
    all)
        deploy_to_environment "dev" false
        deploy_to_environment "staging" false
        echo "Aguardando aprovação para produção..."
        ;;
    *)
        echo "Uso: $0 {dev|staging|prod|all}"
        exit 1
        ;;
esac

echo -e "\n✅ Deploy concluído com sucesso!"
```

### 8.2 Rollback
```
#!/bin/bash
# scripts/rollback.sh

set -e

ENVIRONMENT=${1:-dev}
REGION=${REGION:-us-central1}

echo "========================================="
echo "Rollback Fluxo Caixa - Environment: $ENVIRONMENT"
echo "========================================="

# Listar releases anteriores
gcloud deploy releases list \
    --delivery-pipeline=fluxo-caixa-pipeline \
    --region=$REGION \
    --filter="targetId=$ENVIRONMENT" \
    --sort-by=~createTime \
    --limit=5

# Solicitar release para rollback
read -p "Enter release ID to rollback to: " RELEASE_ID

# Executar rollback
gcloud deploy releases rollback $RELEASE_ID \
    --delivery-pipeline=fluxo-caixa-pipeline \
    --region=$REGION \
    --target=$ENVIRONMENT

echo "✅ Rollback to $RELEASE_ID completed"
```

<a id="pipelene"></a>
### 9. Pipeline Completa

### 9.1 Workflow GitHub Actions (Alternativo)

```
# .github/workflows/ci-cd.yaml
name: CI/CD Pipeline - Fluxo Caixa

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '10.0.x'
  PROJECT_ID: 'fluxo-caixa-project'
  REGION: 'us-central1'
  REPO_NAME: 'fluxo-caixa'

jobs:
  test:
    name: Testes TDD
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      
      - name: Restore dependencies
        run: dotnet restore FluxoCaixa.sln
      
      - name: Build
        run: dotnet build FluxoCaixa.sln --configuration Release --no-restore
      
      - name: Run Unit Tests
        run: dotnet test tests/FluxoCaixa.Tests.Unit --configuration Release --logger trx --collect:"XPlat Code Coverage"
      
      - name: Run Integration Tests
        run: dotnet test tests/FluxoCaixa.Tests.Integration --configuration Release --logger trx
      
      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: |
            **/TestResults/**/*.trx
            **/CoverageReport/**
  
  build-and-push:
    name: Build e Push para Artifact Registry
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      
      - id: auth
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      
      - name: Configure Docker
        run: |
          gcloud auth configure-docker ${REGION}-docker.pkg.dev
      
      - name: Build and Push Images
        run: |
          SHORT_SHA=$(git rev-parse --short HEAD)
          
          docker build --target auth-api -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/auth-api:${SHORT_SHA} -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/auth-api:latest .
          docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/auth-api --all-tags
          
          docker build --target lancamentos-api -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/lancamentos-api:${SHORT_SHA} -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/lancamentos-api:latest .
          docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/lancamentos-api --all-tags
          
          docker build --target consolidacao-api -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/consolidacao-api:${SHORT_SHA} -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/consolidacao-api:latest .
          docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/consolidacao-api --all-tags
          
          docker build --target consolidacao-worker -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/consolidacao-worker:${SHORT_SHA} -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/consolidacao-worker:latest .
          docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/consolidacao-worker --all-tags

  database-migrate:
    name: Database Build (DBB)
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - uses: actions/checkout@v4
      
      - id: auth
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      
      - name: Run Migrations
        run: |
          gcloud sql connect fluxo-caixa-db --user=postgres --database=fluxocaixa_prod < database/migrations/V1__Create_usuarios_table.sql
          gcloud sql connect fluxo-caixa-db --user=postgres --database=fluxocaixa_prod < database/migrations/V2__Create_lancamentos_table.sql
          gcloud sql connect fluxo-caixa-db --user=postgres --database=fluxocaixa_prod < database/migrations/V3__Create_saldo_diario_table.sql

  deploy:
    name: Deploy com Cloud Deploy
    runs-on: ubuntu-latest
    needs: [build-and-push, database-migrate]
    environment:
      name: production
    steps:
      - uses: actions/checkout@v4
      
      - id: auth
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      
      - name: Deploy to GKE
        run: |
          SHORT_SHA=$(git rev-parse --short HEAD)
          
          gcloud deploy releases create rel-${SHORT_SHA} \
            --delivery-pipeline=fluxo-caixa-pipeline \
            --region=us-central1 \
            --target=prod \
            --images=auth-api=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/auth-api:${SHORT_SHA},lancamentos-api=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/lancamentos-api:${SHORT_SHA},consolidacao-api=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/consolidacao-api:${SHORT_SHA},consolidacao-worker=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/consolidacao-worker:${SHORT_SHA}
          
          gcloud deploy releases approve rel-${SHORT_SHA} \
            --delivery-pipeline=fluxo-caixa-pipeline \
            --region=us-central1 \
            --target=prod

```

<a id="monitoramento"></a>
### 10. Monitoramento e Alertas

### 10.1 Cloud Monitoring Dashboard
```
# monitoring/dashboard.yaml
apiVersion: monitoring.googleapis.com/v1
kind: Dashboard
metadata:
  name: fluxo-caixa-cicd-dashboard
  labels:
    team: backend
spec:
  displayName: "CI/CD Pipeline - Fluxo Caixa"
  gridLayout:
    widgets:
    - title: "Build Status"
      xyChart:
        dataSets:
        - timeSeriesQuery: |
            fetch cloud_build
            | metric 'build.googleapis.com/build/duration'
            | filter (status = 'SUCCESS')
            | align rate(1m)
        targetAxis: Y1
    - title: "Test Coverage"
      scorecard:
        gauge:
          field: coverage_percentage
          critical: 80
          upper: 100
    - title: "Deploy Status"
      xyChart:
        dataSets:
        - timeSeriesQuery: |
            fetch cloud_deploy
            | metric 'deploy.googleapis.com/deployment/duration'
        targetAxis: Y1
    - title: "Deployment Frequency"
      xyChart:
        dataSets:
        - timeSeriesQuery: |
            fetch cloud_deploy
            | metric 'deploy.googleapis.com/deployment/count'
            | align rate(1d)

```

### 10.2 Alertas
```
# monitoring/alerts.yaml
apiVersion: monitoring.googleapis.com/v1
kind: AlertPolicy
metadata:
  name: build-failure-alert
spec:
  displayName: "Build Failure Alert"
  conditions:
  - displayName: "Build failed for more than 5 minutes"
    conditionThreshold:
      filter: |
        resource.type = "build"
        metric.type = "build.googleapis.com/build/duration"
        metric.labels.status = "FAILURE"
      aggregations:
      - alignmentPeriod: 300s
        perSeriesAligner: ALIGN_COUNT
      duration: 300s
      comparison: COMPARISON_GT
      thresholdValue: 0
  alertStrategy:
    autoClose: 1800s
  notificationChannels:
  - projects/${PROJECT_ID}/notificationChannels/slack
  - projects/${PROJECT_ID}/notificationChannels/email

```

### Resumo da Pipeline
|Fase|	Ferramenta|	Responsabilidade|	Gatilho
|--|--|--|--
|Test TDD|	xUnit, NSubstitute|	Testes unitários, integração, E2E	|Push any branch
|Build|	.NET 10 SDK|	Compilar código	|Push to main
|DBB|	Flyway + Cloud SQL|	Migrações de banco	|Push to main
|Package|	Docker|	Criar imagens	|Push to main
|Artifact Registry|	GAR|	Armazenar imagens	|Push to main
|Cloud Deploy|	Google Cloud Deploy|	Implantar no GKE	|Push to main
|Monitoramento|	Cloud Monitoring|	Métricas e alertas|	Contínuo

__Este CI/CD pipeline garante que todo o código seja testado (TDD), as migrações de banco de dados sejam aplicadas (DBB), e as aplicações sejam implantadas de forma segura e automatizada no Google Cloud Platform.__