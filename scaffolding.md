# Scaffolding - API's e Workers em C# 14 com Circuit Breaker + Kubernetes Deployments 

### Sumário 



[1. Estrutura do Projeto](#estrutura)\
[2. buildingblocks - Projeto Compartilhado](#buildingblocks)\
[3. Auth API](#authapi)\
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


### 2. BuildingBlocks - Projeto Compartilhado


__2.1 Events / LancamentoRegistradoEvent.cs__

```
namespace FluxoCaixa.BuildingBlocks.Events;

public record LancamentoRegistradoEvent
{
    public Guid Id { get; init; }
    public decimal Valor { get; init; }
    public string Tipo { get; init; } = string.Empty;
    public DateTime DataHora { get; init; }
    public string Descricao { get; init; } = string.Empty;
    public string Categoria { get; init; } = string.Empty;
    public Guid UsuarioId { get; init; }
    public string Estabelecimento { get; init; } = string.Empty;
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
    
    public static LancamentoRegistradoEvent Create(Guid id, decimal valor, string tipo, 
        DateTime dataHora, string descricao, string categoria, Guid usuarioId, string estabelecimento) =>
        new()
        {
            Id = id,
            Valor = valor,
            Tipo = tipo,
            DataHora = dataHora,
            Descricao = descricao,
            Categoria = categoria,
            UsuarioId = usuarioId,
            Estabelecimento = estabelecimento,
            OccurredAt = DateTime.UtcNow
        };
}
```

 
 __2.2 Events / LancamentoCanceladoEvent.cs__

 ```
namespace FluxoCaixa.BuildingBlocks.Events;

public record LancamentoCanceladoEvent
{
    public Guid Id { get; init; }
    public Guid UsuarioId { get; init; }
    public decimal Valor { get; init; }
    public DateTime DataHora { get; init; }
    public DateTime CanceladoEm { get; init; } = DateTime.UtcNow;
    
    public static LancamentoCanceladoEvent Create(Guid id, Guid usuarioId, decimal valor, DateTime dataHora) =>
        new()
        {
            Id = id,
            UsuarioId = usuarioId,
            Valor = valor,
            DataHora = dataHora,
            CanceladoEm = DateTime.UtcNow
        };
}

 ```

__2.3 Events / SaldoDiarioAtualizadoEvent.cs__

```
namespace FluxoCaixa.BuildingBlocks.Events;

public record SaldoDiarioAtualizadoEvent
{
    public DateOnly Data { get; init; }
    public decimal TotalCreditos { get; init; }
    public decimal TotalDebitos { get; init; }
    public decimal Saldo { get; init; }
    public int QuantidadeTransacoes { get; init; }
    public DateTime AtualizadoEm { get; init; }
    
    public static SaldoDiarioAtualizadoEvent Create(DateOnly data, decimal totalCreditos, 
        decimal totalDebitos, decimal saldo, int quantidadeTransacoes) =>
        new()
        {
            Data = data,
            TotalCreditos = totalCreditos,
            TotalDebitos = totalDebitos,
            Saldo = saldo,
            QuantidadeTransacoes = quantidadeTransacoes,
            AtualizadoEm = DateTime.UtcNow
        };
}

```

__2.4 Events / SaldoBaixoAlertadoEvent.cs__

```
namespace FluxoCaixa.BuildingBlocks.Events;

public record SaldoBaixoAlertadoEvent
{
    public Guid UsuarioId { get; init; }
    public DateOnly Data { get; init; }
    public decimal SaldoAtual { get; init; }
    public decimal LimiteAlerta { get; init; }
    public string EmailUsuario { get; init; } = string.Empty;
    
    public static SaldoBaixoAlertadoEvent Create(Guid usuarioId, DateOnly data, 
        decimal saldoAtual, decimal limiteAlerta, string emailUsuario) =>
        new()
        {
            UsuarioId = usuarioId,
            Data = data,
            SaldoAtual = saldoAtual,
            LimiteAlerta = limiteAlerta,
            EmailUsuario = emailUsuario
        };
}

```


__2.5 Interfaces / IKafkaProducer.cs__

```
namespace FluxoCaixa.BuildingBlocks.Interfaces;

public interface IKafkaProducer<TKey, TValue>
{
    Task ProduceAsync(string topic, TKey key, TValue value, CancellationToken cancellationToken = default);
    Task ProduceAsync(string topic, TValue value, CancellationToken cancellationToken = default);
    Task FlushAsync(CancellationToken cancellationToken = default);
}

```

__2.6 Interfaces / IKafkaConsumer.cs__

```
namespace FluxoCaixa.BuildingBlocks.Interfaces;

public interface IKafkaConsumer<TKey, TValue>
{
    Task StartConsumingAsync(string topic, string groupId, Func<TValue, Task> handleAsync, 
        CancellationToken cancellationToken = default);
    void StopConsuming();
    Task CommitAsync(CancellationToken cancellationToken = default);
}

```

__2.7 Interfaces / ICacheService.cs__


```

namespace FluxoCaixa.BuildingBlocks.Interfaces;

public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken cancellationToken = default);
    Task SetAsync<T>(string key, T value, TimeSpan? expiry = null, CancellationToken cancellationToken = default);
    Task RemoveAsync(string key, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(string key, CancellationToken cancellationToken = default);
}

```

__2.8 DTOs / LancamentoDto.cs__

```
namespace FluxoCaixa.BuildingBlocks.DTOs;

public record LancamentoRequestDto
{
    public decimal Valor { get; init; }
    public string Tipo { get; init; } = string.Empty;
    public string Descricao { get; init; } = string.Empty;
    public string Categoria { get; init; } = string.Empty;
    public string Estabelecimento { get; init; } = string.Empty;
    
    public bool IsValid() => Valor > 0 && (Tipo == "DEBITO" || Tipo == "CREDITO");
}

public record LancamentoResponseDto
{
    public Guid Id { get; init; }
    public decimal Valor { get; init; }
    public string Tipo { get; init; } = string.Empty;
    public DateTime DataHora { get; init; }
    public string Descricao { get; init; } = string.Empty;
    public string Categoria { get; init; } = string.Empty;
    public string Status { get; init; } = string.Empty;
    public string Estabelecimento { get; init; } = string.Empty;
}

public record SaldoDiarioResponseDto
{
    public DateOnly Data { get; init; }
    public decimal TotalCreditos { get; init; }
    public decimal TotalDebitos { get; init; }
    public decimal Saldo { get; init; }
    public int QuantidadeTransacoes { get; init; }
}

public record LoginRequestDto
{
    public string Email { get; init; } = string.Empty;
    public string Password { get; init; } = string.Empty;
    public string? Dominio { get; init; }
}

public record LoginResponseDto
{
    public string Token { get; init; } = string.Empty;
    public string RefreshToken { get; init; } = string.Empty;
    public DateTime ExpiraEm { get; init; }
    public UsuarioDto Usuario { get; init; } = null!;
}

public record UsuarioDto
{
    public Guid Id { get; init; }
    public string Nome { get; init; } = string.Empty;
    public string Email { get; init; } = string.Empty;
    public List<string> Roles { get; init; } = new();
}
```

<a id="authapi"></a>

### 3. Circuit Breaker (C# 14)

```
using System.Diagnostics;
using Polly;
using Polly.CircuitBreaker;

namespace FluxoCaixa.CircuitBreaker.Policies;

public static class CircuitBreakerPolicy
{
    public static IAsyncPolicy<HttpResponseMessage> GetHttpCircuitBreakerPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrResult(msg => msg.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (outcome, breakDelay, context) =>
                {
                    Activity.Current?.AddEvent(new ActivityEvent("CircuitBreakerOpened"));
                    Console.WriteLine($"[Circuit Breaker] HTTP - Aberto por {breakDelay.TotalSeconds}s. " +
                        $"Erro: {outcome.Exception?.Message ?? outcome.Result?.StatusCode.ToString()}");
                },
                onReset: (context) =>
                {
                    Activity.Current?.AddEvent(new ActivityEvent("CircuitBreakerReset"));
                    Console.WriteLine("[Circuit Breaker] HTTP - Resetado.");
                },
                onHalfOpen: (context) =>
                {
                    Console.WriteLine("[Circuit Breaker] HTTP - Meio-aberto, testando.");
                }
            );
    }

    public static IAsyncPolicy GetDatabaseCircuitBreakerPolicy()
    {
        return Policy
            .Handle<Exception>(ex => ex is Npgsql.NpgsqlException or TimeoutException)
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(60),
                onBreak: (ex, breakDelay, context) =>
                {
                    Activity.Current?.AddEvent(new ActivityEvent("DatabaseCircuitBreakerOpened"));
                    Console.WriteLine($"[Circuit Breaker] Database - Aberto: {ex.Message}");
                },
                onReset: (context) =>
                {
                    Console.WriteLine("[Circuit Breaker] Database - Resetado.");
                },
                onHalfOpen: (context) =>
                {
                    Console.WriteLine("[Circuit Breaker] Database - Meio-aberto.");
                }
            );
    }

    public static IAsyncPolicy GetKafkaCircuitBreakerPolicy()
    {
        return Policy
            .Handle<Exception>(ex => ex is Confluent.Kafka.KafkaException or TimeoutException)
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromSeconds(45),
                onBreak: (ex, breakDelay, context) =>
                {
                    Activity.Current?.AddEvent(new ActivityEvent("KafkaCircuitBreakerOpened"));
                    Console.WriteLine($"[Circuit Breaker] Kafka - Aberto: {ex.Message}");
                },
                onReset: (context) =>
                {
                    Console.WriteLine("[Circuit Breaker] Kafka - Resetado.");
                },
                onHalfOpen: (context) =>
                {
                    Console.WriteLine("[Circuit Breaker] Kafka - Meio-aberto.");
                }
            );
    }

    public static IAsyncPolicy GetRedisCircuitBreakerPolicy()
    {
        return Policy
            .Handle<Exception>(ex => ex is StackExchange.Redis.RedisConnectionException or TimeoutException)
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (ex, breakDelay, context) =>
                {
                    Activity.Current?.AddEvent(new ActivityEvent("RedisCircuitBreakerOpened"));
                    Console.WriteLine($"[Circuit Breaker] Redis - Aberto: {ex.Message}");
                },
                onReset: (context) =>
                {
                    Console.WriteLine("[Circuit Breaker] Redis - Resetado.");
                },
                onHalfOpen: (context) =>
                {
                    Console.WriteLine("[Circuit Breaker] Redis - Meio-aberto.");
                }
            );
    }

    public static IAsyncPolicy GetCognitoCircuitBreakerPolicy()
    {
        return Policy
            .Handle<Exception>(ex => ex is HttpRequestException or TimeoutException)
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromSeconds(60),
                onBreak: (ex, breakDelay, context) =>
                {
                    Activity.Current?.AddEvent(new ActivityEvent("CognitoCircuitBreakerOpened"));
                    Console.WriteLine($"[Circuit Breaker] Cognito - Aberto: {ex.Message}");
                },
                onReset: (context) =>
                {
                    Console.WriteLine("[Circuit Breaker] Cognito - Resetado.");
                },
                onHalfOpen: (context) =>
                {
                    Console.WriteLine("[Circuit Breaker] Cognito - Meio-aberto.");
                }
            );
    }
}

```

__3.2 CircuitBreakerRegistry.cs__

```

using Polly;

namespace FluxoCaixa.CircuitBreaker.Configurations;

public static class CircuitBreakerRegistry
{
    private static readonly Dictionary<string, IAsyncPolicy> _policies = new();
    private static readonly Lock _lock = new();

    public static IAsyncPolicy GetPolicy(string name, Func<IAsyncPolicy> policyFactory)
    {
        lock (_lock)
        {
            if (!_policies.TryGetValue(name, out var policy))
            {
                policy = policyFactory();
                _policies[name] = policy;
            }
            return policy;
        }
    }

    public static IAsyncPolicy<T> GetPolicy<T>(string name, Func<IAsyncPolicy<T>> policyFactory)
    {
        lock (_lock)
        {
            if (!_policies.ContainsKey(name))
            {
                var policy = policyFactory();
                _policies[name] = policy;
                return policy;
            }
            return (IAsyncPolicy<T>)_policies[name];
        }
    }

    public static IAsyncPolicy GetHttpPolicy() => 
        GetPolicy("http", CircuitBreakerPolicy.GetHttpCircuitBreakerPolicy);

    public static IAsyncPolicy GetDatabasePolicy() => 
        GetPolicy("database", CircuitBreakerPolicy.GetDatabaseCircuitBreakerPolicy);

    public static IAsyncPolicy GetKafkaPolicy() => 
        GetPolicy("kafka", CircuitBreakerPolicy.GetKafkaCircuitBreakerPolicy);

    public static IAsyncPolicy GetRedisPolicy() => 
        GetPolicy("redis", CircuitBreakerPolicy.GetRedisCircuitBreakerPolicy);

    public static IAsyncPolicy GetCognitoPolicy() => 
        GetPolicy("cognito", CircuitBreakerPolicy.GetCognitoCircuitBreakerPolicy);
}
```





