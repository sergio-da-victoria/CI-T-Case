# Scaffolding - API's e Workers em C# 14 com Circuit Breaker + Kubernetes Deployments 

### Sumário 



[1. Estrutura do Projeto](#estrutura)\
[2. buildingblocks - Projeto Compartilhado](#buildingblocks)\
[3. Circuit Breaker](#breaker)\
[4. Auth API](#authapi)\
[5. Lancamentos API](#lancamento)\
[6. Consolidacao API](#consolidacao)\
[7. Workers](#work)\
[8. Relatorios API](#relatorio)\
[9. Kubernetes Deployments (GKE)](#kubernete)


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



<a id="buildingblocks"></a>
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

<a id="breaker"></a>

### 3. Circuit Breaker (C# 14)

__3.1 CircuitBreakerPolicy.cs__

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

<a id="authapi"></a>

### 4. Auth API

__4.1 Program.cs__

```

using System.Text;
using FluxoCaixa.BuildingBlocks.Interfaces;
using FluxoCaixa.CircuitBreaker.Configurations;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// JWT Authentication
var key = Encoding.ASCII.GetBytes(builder.Configuration["Jwt:Secret"] ?? 
    "minha-chave-secreta-com-mais-de-32-caracteres-para-seguranca");
builder.Services.AddAuthentication(x =>
{
    x.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    x.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(x =>
{
    x.RequireHttpsMetadata = false;
    x.SaveToken = true;
    x.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(key),
        ValidateIssuer = false,
        ValidateAudience = false,
        ClockSkew = TimeSpan.Zero
    };
});

builder.Services.AddAuthorization();

// Database
builder.Services.AddNpgsqlDataSource(builder.Configuration.GetConnectionString("AuthDb")!);

// Redis Cache
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration["Redis:ConnectionString"]!));
builder.Services.AddScoped<ICacheService, RedisCacheService>();

// Circuit Breakers
builder.Services.AddSingleton(CircuitBreakerRegistry.GetDatabasePolicy());
builder.Services.AddSingleton(CircuitBreakerRegistry.GetRedisPolicy());
builder.Services.AddSingleton(CircuitBreakerRegistry.GetCognitoPolicy());

// Services
builder.Services.AddScoped<IAuthService, AuthService>();

// Health Checks
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("AuthDb")!)
    .AddRedis(builder.Configuration["Redis:ConnectionString"]!);

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health/ready");
app.MapHealthChecks("/health/live");

app.Run();

```

__4.2 appsettings.json__

```

{
  "Jwt": {
    "Secret": "minha-chave-secreta-com-mais-de-32-caracteres-para-seguranca",
    "ExpirationHours": 8
  },
  "ConnectionStrings": {
    "AuthDb": "Host=postgres-auth;Port=5432;Database=fluxocaixa_auth;Username=postgres;Password=postgres"
  },
  "Redis": {
    "ConnectionString": "redis:6379,password=redispass,abortConnect=false"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}

```

__4.3 AuthService.cs__

```
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using FluxoCaixa.BuildingBlocks.DTOs;
using FluxoCaixa.BuildingBlocks.Interfaces;
using Microsoft.IdentityModel.Tokens;
using Polly;
using StackExchange.Redis;

namespace Auth.Api.Services;

public interface IAuthService
{
    Task<LoginResponseDto?> LoginAsync(LoginRequestDto request);
    Task<LoginResponseDto?> RefreshTokenAsync(string refreshToken);
    Task<bool> RegisterUserAsync(RegisterRequestDto request);
}

public class AuthService : IAuthService
{
    private readonly IConfiguration _configuration;
    private readonly NpgsqlDataSource _dataSource;
    private readonly ICacheService _cache;
    private readonly IAsyncPolicy _dbCircuitBreaker;
    private readonly IAsyncPolicy _redisCircuitBreaker;
    private readonly ILogger<AuthService> _logger;

    public AuthService(
        IConfiguration configuration,
        NpgsqlDataSource dataSource,
        ICacheService cache,
        [FromKeyedServices("DatabaseCircuitBreaker")] IAsyncPolicy dbCircuitBreaker,
        [FromKeyedServices("RedisCircuitBreaker")] IAsyncPolicy redisCircuitBreaker,
        ILogger<AuthService> logger)
    {
        _configuration = configuration;
        _dataSource = dataSource;
        _cache = cache;
        _dbCircuitBreaker = dbCircuitBreaker;
        _redisCircuitBreaker = redisCircuitBreaker;
        _logger = logger;
    }

    public async Task<LoginResponseDto?> LoginAsync(LoginRequestDto request)
    {
        try
        {
            var user = await _dbCircuitBreaker.ExecuteAsync(async () =>
            {
                await using var conn = await _dataSource.OpenConnectionAsync();
                await using var cmd = new NpgsqlCommand(
                    @"SELECT id, nome, email, roles FROM usuarios 
                      WHERE email = @email AND senha_hash = crypt(@senha, senha_hash) 
                      AND deleted_at IS NULL",
                    conn);
                cmd.Parameters.AddWithValue("@email", request.Email);
                cmd.Parameters.AddWithValue("@senha", request.Password);

                await using var reader = await cmd.ExecuteReaderAsync();
                if (await reader.ReadAsync())
                {
                    return new UsuarioDto
                    {
                        Id = reader.GetGuid(0),
                        Nome = reader.GetString(1),
                        Email = reader.GetString(2),
                        Roles = reader.GetString(3).Split(',').ToList()
                    };
                }
                return null;
            });

            if (user == null) return null;

            var token = GenerateJwtToken(user);
            var refreshToken = GenerateRefreshToken();

            // Cache refresh token
            await _redisCircuitBreaker.ExecuteAsync(async () =>
            {
                await _cache.SetAsync($"refresh:{refreshToken}", user.Id.ToString(), 
                    TimeSpan.FromHours(24));
            });

            return new LoginResponseDto
            {
                Token = token,
                RefreshToken = refreshToken,
                ExpiraEm = DateTime.UtcNow.AddHours(8),
                Usuario = user
            };
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Database circuit breaker is OPEN - service unavailable");
            throw new Exception("Serviço temporariamente indisponível. Tente novamente mais tarde.");
        }
    }

    public async Task<LoginResponseDto?> RefreshTokenAsync(string refreshToken)
    {
        try
        {
            var userId = await _redisCircuitBreaker.ExecuteAsync(async () =>
            {
                var userIdStr = await _cache.GetAsync<string>($"refresh:{refreshToken}");
                return userIdStr != null ? Guid.Parse(userIdStr) : (Guid?)null;
            });

            if (userId == null) return null;

            var user = await _dbCircuitBreaker.ExecuteAsync(async () =>
            {
                await using var conn = await _dataSource.OpenConnectionAsync();
                await using var cmd = new NpgsqlCommand(
                    "SELECT id, nome, email, roles FROM usuarios WHERE id = @id AND deleted_at IS NULL",
                    conn);
                cmd.Parameters.AddWithValue("@id", userId.Value);

                await using var reader = await cmd.ExecuteReaderAsync();
                if (await reader.ReadAsync())
                {
                    return new UsuarioDto
                    {
                        Id = reader.GetGuid(0),
                        Nome = reader.GetString(1),
                        Email = reader.GetString(2),
                        Roles = reader.GetString(3).Split(',').ToList()
                    };
                }
                return null;
            });

            if (user == null) return null;

            var newToken = GenerateJwtToken(user);
            var newRefreshToken = GenerateRefreshToken();

            await _redisCircuitBreaker.ExecuteAsync(async () =>
            {
                await _cache.RemoveAsync($"refresh:{refreshToken}");
                await _cache.SetAsync($"refresh:{newRefreshToken}", user.Id.ToString(), 
                    TimeSpan.FromHours(24));
            });

            return new LoginResponseDto
            {
                Token = newToken,
                RefreshToken = newRefreshToken,
                ExpiraEm = DateTime.UtcNow.AddHours(8),
                Usuario = user
            };
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Redis circuit breaker is OPEN");
            throw new Exception("Serviço temporariamente indisponível.");
        }
    }

    public async Task<bool> RegisterUserAsync(RegisterRequestDto request)
    {
        try
        {
            return await _dbCircuitBreaker.ExecuteAsync(async () =>
            {
                await using var conn = await _dataSource.OpenConnectionAsync();
                await using var cmd = new NpgsqlCommand(
                    @"INSERT INTO usuarios (id, nome, email, senha_hash, roles, created_at) 
                      VALUES (@id, @nome, @email, crypt(@senha, gen_salt('bf')), @roles, @createdAt)",
                    conn);

                cmd.Parameters.AddWithValue("@id", Guid.NewGuid());
                cmd.Parameters.AddWithValue("@nome", request.Nome);
                cmd.Parameters.AddWithValue("@email", request.Email);
                cmd.Parameters.AddWithValue("@senha", request.Password);
                cmd.Parameters.AddWithValue("@roles", "comerciante");
                cmd.Parameters.AddWithValue("@createdAt", DateTime.UtcNow);

                var result = await cmd.ExecuteNonQueryAsync();
                return result > 0;
            });
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Database circuit breaker is OPEN");
            throw new Exception("Serviço temporariamente indisponível.");
        }
    }

    private string GenerateJwtToken(UsuarioDto user)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.ASCII.GetBytes(_configuration["Jwt:Secret"]!);
        
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new(ClaimTypes.Email, user.Email),
            new(ClaimTypes.Name, user.Nome)
        };
        
        claims.AddRange(user.Roles.Select(role => new Claim(ClaimTypes.Role, role)));

        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(claims),
            Expires = DateTime.UtcNow.AddHours(8),
            SigningCredentials = new SigningCredentials(
                new SymmetricSecurityKey(key), 
                SecurityAlgorithms.HmacSha256Signature)
        };
        
        var token = tokenHandler.CreateToken(tokenDescriptor);
        return tokenHandler.WriteToken(token);
    }

    private static string GenerateRefreshToken()
    {
        var randomNumber = new byte[32];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomNumber);
        return Convert.ToBase64String(randomNumber);
    }
}

```

__4.4 AuthController.cs__

```

using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using FluxoCaixa.BuildingBlocks.DTOs;

namespace Auth.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;
    private readonly ILogger<AuthController> _logger;

    public AuthController(IAuthService authService, ILogger<AuthController> logger)
    {
        _authService = authService;
        _logger = logger;
    }

    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginRequestDto request)
    {
        try
        {
            var response = await _authService.LoginAsync(request);
            if (response == null)
                return Unauthorized(new { message = "Credenciais inválidas" });
            
            return Ok(response);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro durante login");
            return StatusCode(503, new { message = "Serviço temporariamente indisponível" });
        }
    }

    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterRequestDto request)
    {
        var result = await _authService.RegisterUserAsync(request);
        if (!result)
            return BadRequest(new { message = "Erro ao registrar usuário" });
        
        return Ok(new { message = "Usuário registrado com sucesso" });
    }

    [HttpPost("refresh")]
    public async Task<IActionResult> Refresh([FromBody] RefreshTokenRequest request)
    {
        var response = await _authService.RefreshTokenAsync(request.RefreshToken);
        if (response == null)
            return Unauthorized(new { message = "Refresh token inválido" });
        
        return Ok(response);
    }

    [Authorize]
    [HttpGet("me")]
    public IActionResult GetCurrentUser()
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var email = User.FindFirst(ClaimTypes.Email)?.Value;
        
        return Ok(new { userId, email });
    }
}

public record RegisterRequestDto
{
    public string Nome { get; init; } = string.Empty;
    public string Email { get; init; } = string.Empty;
    public string Password { get; init; } = string.Empty;
}

public record RefreshTokenRequest
{
    public string RefreshToken { get; init; } = string.Empty;
}

```

<a id="lancamento"></a>

### 5. Lancamentos API

__5.1 Program.cs__

```
using System.Text;
using FluxoCaixa.BuildingBlocks.Interfaces;
using FluxoCaixa.CircuitBreaker.Configurations;
using Lancamentos.Api.Services;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// JWT Authentication
var key = Encoding.ASCII.GetBytes(builder.Configuration["Jwt:Secret"]!);
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = false,
            ValidateAudience = false,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key)
        };
    });

builder.Services.AddAuthorization();

// Database
builder.Services.AddNpgsqlDataSource(builder.Configuration.GetConnectionString("LancamentosDb")!);

// Kafka Producer
builder.Services.AddSingleton<IKafkaProducer<string, LancamentoRegistradoEvent>>(sp =>
    new KafkaProducer<string, LancamentoRegistradoEvent>(
        builder.Configuration["Kafka:BootstrapServers"]!));

// Circuit Breakers
builder.Services.AddSingleton(CircuitBreakerRegistry.GetDatabasePolicy());
builder.Services.AddSingleton(CircuitBreakerRegistry.GetKafkaPolicy());

// Services
builder.Services.AddScoped<ILancamentoService, LancamentoService>();

// Health Checks
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("LancamentosDb")!)
    .AddKafka(builder.Configuration["Kafka:BootstrapServers"]!);

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health/ready");
app.MapHealthChecks("/health/live");

app.Run();

```

__5.2 LancamentoService.cs__

```
using System.Text.Json;
using FluxoCaixa.BuildingBlocks.DTOs;
using FluxoCaixa.BuildingBlocks.Events;
using FluxoCaixa.BuildingBlocks.Interfaces;
using Polly;

namespace Lancamentos.Api.Services;

public interface ILancamentoService
{
    Task<LancamentoResponseDto> RegistrarLancamentoAsync(LancamentoRequestDto request, Guid usuarioId);
    Task<LancamentoResponseDto?> CancelarLancamentoAsync(Guid id, Guid usuarioId);
    Task<IEnumerable<LancamentoResponseDto>> ListarLancamentosAsync(Guid usuarioId, DateOnly? inicio, DateOnly? fim);
}

public class LancamentoService : ILancamentoService
{
    private readonly NpgsqlDataSource _dataSource;
    private readonly IKafkaProducer<string, LancamentoRegistradoEvent> _kafkaProducer;
    private readonly IAsyncPolicy _dbCircuitBreaker;
    private readonly IAsyncPolicy _kafkaCircuitBreaker;
    private readonly ILogger<LancamentoService> _logger;

    public LancamentoService(
        NpgsqlDataSource dataSource,
        IKafkaProducer<string, LancamentoRegistradoEvent> kafkaProducer,
        [FromKeyedServices("DatabaseCircuitBreaker")] IAsyncPolicy dbCircuitBreaker,
        [FromKeyedServices("KafkaCircuitBreaker")] IAsyncPolicy kafkaCircuitBreaker,
        ILogger<LancamentoService> logger)
    {
        _dataSource = dataSource;
        _kafkaProducer = kafkaProducer;
        _dbCircuitBreaker = dbCircuitBreaker;
        _kafkaCircuitBreaker = kafkaCircuitBreaker;
        _logger = logger;
    }

    public async Task<LancamentoResponseDto> RegistrarLancamentoAsync(LancamentoRequestDto request, Guid usuarioId)
    {
        if (!request.IsValid())
            throw new ArgumentException("Dados do lançamento inválidos");

        try
        {
            var lancamento = await _dbCircuitBreaker.ExecuteAsync(async () =>
            {
                await using var conn = await _dataSource.OpenConnectionAsync();
                await using var cmd = new NpgsqlCommand(
                    @"INSERT INTO lancamentos (id, valor, tipo, data_hora, descricao, categoria, 
                        usuario_id, estabelecimento, status, created_at) 
                      VALUES (@id, @valor, @tipo, @dataHora, @descricao, @categoria, 
                        @usuarioId, @estabelecimento, 'CONFIRMADO', @createdAt)
                      RETURNING id, data_hora",
                    conn);

                var id = Guid.NewGuid();
                var dataHora = DateTime.UtcNow;

                cmd.Parameters.AddWithValue("@id", id);
                cmd.Parameters.AddWithValue("@valor", request.Valor);
                cmd.Parameters.AddWithValue("@tipo", request.Tipo);
                cmd.Parameters.AddWithValue("@dataHora", dataHora);
                cmd.Parameters.AddWithValue("@descricao", request.Descricao);
                cmd.Parameters.AddWithValue("@categoria", request.Categoria);
                cmd.Parameters.AddWithValue("@usuarioId", usuarioId);
                cmd.Parameters.AddWithValue("@estabelecimento", request.Estabelecimento);
                cmd.Parameters.AddWithValue("@createdAt", dataHora);

                await cmd.ExecuteNonQueryAsync();

                return new LancamentoResponseDto
                {
                    Id = id,
                    Valor = request.Valor,
                    Tipo = request.Tipo,
                    DataHora = dataHora,
                    Descricao = request.Descricao,
                    Categoria = request.Categoria,
                    Status = "CONFIRMADO",
                    Estabelecimento = request.Estabelecimento
                };
            });

            // Publicar evento no Kafka com Circuit Breaker
            await _kafkaCircuitBreaker.ExecuteAsync(async () =>
            {
                var evento = LancamentoRegistradoEvent.Create(
                    lancamento.Id, lancamento.Valor, lancamento.Tipo,
                    lancamento.DataHora, lancamento.Descricao, lancamento.Categoria,
                    usuarioId, request.Estabelecimento);

                await _kafkaProducer.ProduceAsync("lancamentos", evento.Id.ToString(), evento);
                _logger.LogInformation("Evento LancamentoRegistrado publicado para ID {LancamentoId}", evento.Id);
            });

            return lancamento;
        }
        catch (BrokenCircuitException ex)
        {
            _logger.LogError(ex, "Circuit breaker aberto ao tentar registrar lançamento");
            throw new Exception("Serviço temporariamente indisponível. Tente novamente mais tarde.");
        }
    }

    public async Task<LancamentoResponseDto?> CancelarLancamentoAsync(Guid id, Guid usuarioId)
    {
        try
        {
            var lancamento = await _dbCircuitBreaker.ExecuteAsync(async () =>
            {
                await using var conn = await _dataSource.OpenConnectionAsync();
                
                // Buscar lançamento
                await using var selectCmd = new NpgsqlCommand(
                    "SELECT valor, tipo, data_hora FROM lancamentos WHERE id = @id AND usuario_id = @usuarioId AND status = 'CONFIRMADO'",
                    conn);
                selectCmd.Parameters.AddWithValue("@id", id);
                selectCmd.Parameters.AddWithValue("@usuarioId", usuarioId);

                await using var reader = await selectCmd.ExecuteReaderAsync();
                if (!await reader.ReadAsync())
                    return null;

                var valor = reader.GetDecimal(0);
                var tipo = reader.GetString(1);
                var dataHora = reader.GetDateTime(2);
                await reader.Close();

                // Cancelar lançamento
                await using var updateCmd = new NpgsqlCommand(
                    "UPDATE lancamentos SET status = 'CANCELADO', cancelled_at = @cancelledAt WHERE id = @id",
                    conn);
                updateCmd.Parameters.AddWithValue("@cancelledAt", DateTime.UtcNow);
                updateCmd.Parameters.AddWithValue("@id", id);
                await updateCmd.ExecuteNonQueryAsync();

                // Publicar evento de cancelamento
                var cancelEvent = LancamentoCanceladoEvent.Create(id, usuarioId, valor, dataHora);
                await _kafkaProducer.ProduceAsync("lancamentos", cancelEvent.Id.ToString(), cancelEvent);

                return new LancamentoResponseDto
                {
                    Id = id,
                    Valor = valor,
                    Tipo = tipo,
                    DataHora = dataHora,
                    Status = "CANCELADO"
                };
            });

            return lancamento;
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Circuit breaker aberto ao cancelar lançamento");
            throw new Exception("Serviço temporariamente indisponível");
        }
    }

    public async Task<IEnumerable<LancamentoResponseDto>> ListarLancamentosAsync(Guid usuarioId, DateOnly? inicio, DateOnly? fim)
    {
        try
        {
            return await _dbCircuitBreaker.ExecuteAsync(async () =>
            {
                var lancamentos = new List<LancamentoResponseDto>();
                
                await using var conn = await _dataSource.OpenConnectionAsync();
                var sql = @"SELECT id, valor, tipo, data_hora, descricao, categoria, status, estabelecimento 
                           FROM lancamentos WHERE usuario_id = @usuarioId";
                
                if (inicio.HasValue)
                    sql += " AND data_hora >= @inicio";
                if (fim.HasValue)
                    sql += " AND data_hora <= @fim";
                
                sql += " ORDER BY data_hora DESC";

                await using var cmd = new NpgsqlCommand(sql, conn);
                cmd.Parameters.AddWithValue("@usuarioId", usuarioId);
                if (inicio.HasValue)
                    cmd.Parameters.AddWithValue("@inicio", inicio.Value.ToDateTime(TimeOnly.MinValue));
                if (fim.HasValue)
                    cmd.Parameters.AddWithValue("@fim", fim.Value.ToDateTime(TimeOnly.MaxValue));

                await using var reader = await cmd.ExecuteReaderAsync();
                while (await reader.ReadAsync())
                {
                    lancamentos.Add(new LancamentoResponseDto
                    {
                        Id = reader.GetGuid(0),
                        Valor = reader.GetDecimal(1),
                        Tipo = reader.GetString(2),
                        DataHora = reader.GetDateTime(3),
                        Descricao = reader.GetString(4),
                        Categoria = reader.GetString(5),
                        Status = reader.GetString(6),
                        Estabelecimento = reader.GetString(7)
                    });
                }

                return lancamentos;
            });
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Circuit breaker aberto ao listar lançamentos");
            throw new Exception("Serviço temporariamente indisponível");
        }
    }
}

```

__5.3 LancamentosController.cs__

```
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using FluxoCaixa.BuildingBlocks.DTOs;

namespace Lancamentos.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class LancamentosController : ControllerBase
{
    private readonly ILancamentoService _lancamentoService;
    private readonly ILogger<LancamentosController> _logger;

    public LancamentosController(ILancamentoService lancamentoService, ILogger<LancamentosController> logger)
    {
        _lancamentoService = lancamentoService;
        _logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> RegistrarLancamento([FromBody] LancamentoRequestDto request)
    {
        var usuarioId = Guid.Parse(User.FindFirst(ClaimTypes.NameIdentifier)?.Value 
            ?? throw new UnauthorizedAccessException());
        
        var result = await _lancamentoService.RegistrarLancamentoAsync(request, usuarioId);
        return CreatedAtAction(nameof(RegistrarLancamento), new { id = result.Id }, result);
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> CancelarLancamento(Guid id)
    {
        var usuarioId = Guid.Parse(User.FindFirst(ClaimTypes.NameIdentifier)?.Value 
            ?? throw new UnauthorizedAccessException());
        
        var result = await _lancamentoService.CancelarLancamentoAsync(id, usuarioId);
        if (result == null)
            return NotFound();
        
        return Ok(result);
    }

    [HttpGet]
    public async Task<IActionResult> ListarLancamentos([FromQuery] DateOnly? inicio, [FromQuery] DateOnly? fim)
    {
        var usuarioId = Guid.Parse(User.FindFirst(ClaimTypes.NameIdentifier)?.Value 
            ?? throw new UnauthorizedAccessException());
        
        var result = await _lancamentoService.ListarLancamentosAsync(usuarioId, inicio, fim);
        return Ok(result);
    }
}

```

<a id="consolidacao"></a>
### 6. Consolidacao API

__6.1 Program.cs__

```
using System.Text;
using FluxoCaixa.BuildingBlocks.Interfaces;
using FluxoCaixa.CircuitBreaker.Configurations;
using Consolidacao.Api.Services;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// JWT Authentication
var key = Encoding.ASCII.GetBytes(builder.Configuration["Jwt:Secret"]!);
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = false,
            ValidateAudience = false,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key)
        };
    });

builder.Services.AddAuthorization();

// Redis Cache
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration["Redis:ConnectionString"]!));
builder.Services.AddScoped<ICacheService, RedisCacheService>();

// Database Read
builder.Services.AddNpgsqlDataSource(builder.Configuration.GetConnectionString("ReadDb")!);

// Circuit Breakers
builder.Services.AddSingleton(CircuitBreakerRegistry.GetDatabasePolicy());
builder.Services.AddSingleton(CircuitBreakerRegistry.GetRedisPolicy());

// Services
builder.Services.AddScoped<IConsolidacaoService, ConsolidacaoService>();

// Health Checks
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("ReadDb")!)
    .AddRedis(builder.Configuration["Redis:ConnectionString"]!);

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health/ready");
app.MapHealthChecks("/health/live");

app.Run();

```
__6.2 ConsolidacaoService.cs__

```

using System.Text.Json;
using FluxoCaixa.BuildingBlocks.DTOs;
using FluxoCaixa.BuildingBlocks.Interfaces;
using Polly;
using StackExchange.Redis;

namespace Consolidacao.Api.Services;

public interface IConsolidacaoService
{
    Task<SaldoDiarioResponseDto?> ObterSaldoDiarioAsync(DateOnly data);
    Task<SaldoMensalResponseDto?> ObterSaldoMensalAsync(int ano, int mes);
}

public class ConsolidacaoService : IConsolidacaoService
{
    private readonly ICacheService _cache;
    private readonly NpgsqlDataSource _dataSource;
    private readonly IAsyncPolicy _redisCircuitBreaker;
    private readonly IAsyncPolicy _dbCircuitBreaker;
    private readonly ILogger<ConsolidacaoService> _logger;

    public ConsolidacaoService(
        ICacheService cache,
        NpgsqlDataSource dataSource,
        [FromKeyedServices("RedisCircuitBreaker")] IAsyncPolicy redisCircuitBreaker,
        [FromKeyedServices("DatabaseCircuitBreaker")] IAsyncPolicy dbCircuitBreaker,
        ILogger<ConsolidacaoService> logger)
    {
        _cache = cache;
        _dataSource = dataSource;
        _redisCircuitBreaker = redisCircuitBreaker;
        _dbCircuitBreaker = dbCircuitBreaker;
        _logger = logger;
    }

    public async Task<SaldoDiarioResponseDto?> ObterSaldoDiarioAsync(DateOnly data)
    {
        var cacheKey = $"saldo:{data:yyyy-MM-dd}";
        
        try
        {
            // Tentar obter do Redis com Circuit Breaker
            var cached = await _redisCircuitBreaker.ExecuteAsync(async () =>
            {
                return await _cache.GetAsync<SaldoDiarioResponseDto>(cacheKey);
            });
            
            if (cached != null)
            {
                _logger.LogInformation("Cache hit para data {Data}", data);
                return cached;
            }
        }
        catch (BrokenCircuitException)
        {
            _logger.LogWarning("Redis circuit breaker aberto - ignorando cache");
        }
        
        // Cache miss - buscar no Read Database
        try
        {
            var saldo = await _dbCircuitBreaker.ExecuteAsync(async () =>
            {
                await using var conn = await _dataSource.OpenConnectionAsync();
                await using var cmd = new NpgsqlCommand(
                    @"SELECT data, total_creditos, total_debitos, saldo, quantidade_transacoes 
                      FROM saldo_diario WHERE data = @data",
                    conn);
                cmd.Parameters.AddWithValue("@data", data.ToDateTime(TimeOnly.MinValue));
                
                await using var reader = await cmd.ExecuteReaderAsync();
                if (await reader.ReadAsync())
                {
                    return new SaldoDiarioResponseDto
                    {
                        Data = data,
                        TotalCreditos = reader.GetDecimal(1),
                        TotalDebitos = reader.GetDecimal(2),
                        Saldo = reader.GetDecimal(3),
                        QuantidadeTransacoes = reader.GetInt32(4)
                    };
                }
                return null;
            });
            
            if (saldo != null)
            {
                // Atualizar cache em background
                _ = Task.Run(async () =>
                {
                    try
                    {
                        await _redisCircuitBreaker.ExecuteAsync(async () =>
                        {
                            await _cache.SetAsync(cacheKey, saldo, TimeSpan.FromHours(24));
                        });
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Falha ao atualizar cache Redis");
                    }
                });
            }
            
            return saldo;
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Database circuit breaker aberto");
            throw new Exception("Serviço temporariamente indisponível");
        }
    }

    public async Task<SaldoMensalResponseDto?> ObterSaldoMensalAsync(int ano, int mes)
    {
        try
        {
            return await _dbCircuitBreaker.ExecuteAsync(async () =>
            {
                await using var conn = await _dataSource.OpenConnectionAsync();
                
                // Calcular saldo mensal a partir dos saldos diários
                await using var cmd = new NpgsqlCommand(
                    @"SELECT 
                        SUM(total_creditos) as total_creditos,
                        SUM(total_debitos) as total_debitos,
                        SUM(saldo) as saldo_total
                      FROM saldo_diario 
                      WHERE EXTRACT(YEAR FROM data) = @ano AND EXTRACT(MONTH FROM data) = @mes",
                    conn);
                cmd.Parameters.AddWithValue("@ano", ano);
                cmd.Parameters.AddWithValue("@mes", mes);
                
                await using var reader = await cmd.ExecuteReaderAsync();
                if (await reader.ReadAsync())
                {
                    return new SaldoMensalResponseDto
                    {
                        Ano = ano,
                        Mes = mes,
                        TotalCreditos = reader.GetDecimal(0),
                        TotalDebitos = reader.GetDecimal(1),
                        Saldo = reader.GetDecimal(2)
                    };
                }
                return null;
            });
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Database circuit breaker aberto");
            throw new Exception("Serviço temporariamente indisponível");
        }
    }
}

public record SaldoMensalResponseDto
{
    public int Ano { get; init; }
    public int Mes { get; init; }
    public decimal TotalCreditos { get; init; }
    public decimal TotalDebitos { get; init; }
    public decimal Saldo { get; init; }
}

```

__6.3 ConsolidadoController.cs__

```

using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace Consolidacao.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class ConsolidadoController : ControllerBase
{
    private readonly IConsolidacaoService _consolidacaoService;

    public ConsolidadoController(IConsolidacaoService consolidacaoService)
    {
        _consolidacaoService = consolidacaoService;
    }

    [HttpGet("diario")]
    public async Task<IActionResult> ObterSaldoDiario([FromQuery] DateOnly? data)
    {
        var dataConsulta = data ?? DateOnly.FromDateTime(DateTime.Today);
        var resultado = await _consolidacaoService.ObterSaldoDiarioAsync(dataConsulta);
        
        if (resultado == null)
            return NotFound($"Nenhum saldo encontrado para a data {dataConsulta}");
        
        return Ok(resultado);
    }

    [HttpGet("mensal")]
    public async Task<IActionResult> ObterSaldoMensal([FromQuery] int ano, [FromQuery] int mes)
    {
        var resultado = await _consolidacaoService.ObterSaldoMensalAsync(ano, mes);
        
        if (resultado == null)
            return NotFound($"Nenhum saldo encontrado para {mes}/{ano}");
        
        return Ok(resultado);
    }
}

```

__6.4 RedisCacheService.cs__

```

using System.Text.Json;
using FluxoCaixa.BuildingBlocks.Interfaces;
using StackExchange.Redis;

namespace Consolidacao.Api.Services;

public class RedisCacheService : ICacheService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _db;

    public RedisCacheService(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _db = redis.GetDatabase();
    }

    public async Task<T?> GetAsync<T>(string key, CancellationToken cancellationToken = default)
    {
        var value = await _db.StringGetAsync(key);
        if (value.IsNullOrEmpty)
            return default;
        
        return JsonSerializer.Deserialize<T>(value!);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null, CancellationToken cancellationToken = default)
    {
        var json = JsonSerializer.Serialize(value);
        await _db.StringSetAsync(key, json, expiry ?? TimeSpan.FromHours(24));
    }

    public async Task RemoveAsync(string key, CancellationToken cancellationToken = default)
    {
        await _db.KeyDeleteAsync(key);
    }

    public async Task<bool> ExistsAsync(string key, CancellationToken cancellationToken = default)
    {
        return await _db.KeyExistsAsync(key);
    }
}

```

<a id="work"></a>

### 7. Workers

__7.1 Consolidacao.Worker / Program.cs__

```
using FluxoCaixa.CircuitBreaker.Configurations;
using Consolidacao.Worker.Services;
using StackExchange.Redis;

var builder = Host.CreateApplicationBuilder(args);

// Kafka Consumer
builder.Services.AddSingleton<IConsumer<string, string>>(sp =>
{
    var config = new ConsumerConfig
    {
        BootstrapServers = builder.Configuration["Kafka:BootstrapServers"],
        GroupId = "consolidacao-group",
        AutoOffsetReset = AutoOffsetReset.Earliest,
        EnableAutoCommit = false,
        AllowAutoCreateTopics = true
    };
    return new ConsumerBuilder<string, string>(config).Build();
});

// Redis
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration["Redis:ConnectionString"]!));
builder.Services.AddScoped<ICacheService, RedisCacheService>();

// Database
builder.Services.AddNpgsqlDataSource(builder.Configuration.GetConnectionString("ReadDb")!);

// Circuit Breakers
builder.Services.AddSingleton(CircuitBreakerRegistry.GetDatabasePolicy());
builder.Services.AddSingleton(CircuitBreakerRegistry.GetRedisPolicy());
builder.Services.AddSingleton(CircuitBreakerRegistry.GetKafkaPolicy());

// Worker
builder.Services.AddHostedService<ConsolidacaoWorker>();

var host = builder.Build();
host.Run();

```

__7.2 Consolidacao.Worker / ConsolidacaoWorker.cs__

```
using System.Text.Json;
using Confluent.Kafka;
using FluxoCaixa.BuildingBlocks.Events;
using FluxoCaixa.BuildingBlocks.Interfaces;
using Polly;

namespace Consolidacao.Worker.Services;

public class ConsolidacaoWorker : BackgroundService
{
    private readonly IConsumer<string, string> _consumer;
    private readonly NpgsqlDataSource _dataSource;
    private readonly ICacheService _cache;
    private readonly IAsyncPolicy _dbCircuitBreaker;
    private readonly IAsyncPolicy _redisCircuitBreaker;
    private readonly IAsyncPolicy _kafkaCircuitBreaker;
    private readonly ILogger<ConsolidacaoWorker> _logger;

    public ConsolidacaoWorker(
        IConsumer<string, string> consumer,
        NpgsqlDataSource dataSource,
        ICacheService cache,
        [FromKeyedServices("DatabaseCircuitBreaker")] IAsyncPolicy dbCircuitBreaker,
        [FromKeyedServices("RedisCircuitBreaker")] IAsyncPolicy redisCircuitBreaker,
        [FromKeyedServices("KafkaCircuitBreaker")] IAsyncPolicy kafkaCircuitBreaker,
        ILogger<ConsolidacaoWorker> logger)
    {
        _consumer = consumer;
        _dataSource = dataSource;
        _cache = cache;
        _dbCircuitBreaker = dbCircuitBreaker;
        _redisCircuitBreaker = redisCircuitBreaker;
        _kafkaCircuitBreaker = kafkaCircuitBreaker;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _consumer.Subscribe("lancamentos");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var consumeResult = _consumer.Consume(stoppingToken);
                await ProcessEventAsync(consumeResult.Message.Value, stoppingToken);
                _consumer.Commit(consumeResult);
            }
            catch (ConsumeException ex)
            {
                _logger.LogError(ex, "Erro ao consumir mensagem do Kafka");
                await Task.Delay(5000, stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Erro inesperado no worker");
                await Task.Delay(5000, stoppingToken);
            }
        }
        
        _consumer.Close();
    }

    private async Task ProcessEventAsync(string messageValue, CancellationToken cancellationToken)
    {
        var evento = JsonSerializer.Deserialize<LancamentoRegistradoEvent>(messageValue);
        if (evento == null) return;
        
        _logger.LogInformation("Processando lançamento ID {LancamentoId}", evento.Id);
        
        var data = DateOnly.FromDateTime(evento.DataHora);
        var valor = evento.Tipo == "CREDITO" ? evento.Valor : -evento.Valor;
        
        try
        {
            // Atualizar Read Database com Circuit Breaker
            await _dbCircuitBreaker.ExecuteAsync(async () =>
            {
                await using var conn = await _dataSource.OpenConnectionAsync(cancellationToken);
                await using var cmd = new NpgsqlCommand(
                    @"INSERT INTO saldo_diario (data, total_creditos, total_debitos, saldo, quantidade_transacoes, ultima_atualizacao)
                      VALUES (@data, 
                              CASE WHEN @tipo = 'CREDITO' THEN @valor ELSE 0 END,
                              CASE WHEN @tipo = 'DEBITO' THEN @valor ELSE 0 END,
                              @valor,
                              1,
                              @atualizacao)
                      ON CONFLICT (data) DO UPDATE SET
                          total_creditos = saldo_diario.total_creditos + EXCLUDED.total_creditos,
                          total_debitos = saldo_diario.total_debitos + EXCLUDED.total_debitos,
                          saldo = saldo_diario.saldo + EXCLUDED.saldo,
                          quantidade_transacoes = saldo_diario.quantidade_transacoes + 1,
                          ultima_atualizacao = EXCLUDED.ultima_atualizacao",
                    conn);
                
                cmd.Parameters.AddWithValue("@data", data.ToDateTime(TimeOnly.MinValue));
                cmd.Parameters.AddWithValue("@tipo", evento.Tipo);
                cmd.Parameters.AddWithValue("@valor", valor);
                cmd.Parameters.AddWithValue("@atualizacao", DateTime.UtcNow);
                
                await cmd.ExecuteNonQueryAsync(cancellationToken);
            });
            
            // Atualizar Redis Cache com Circuit Breaker
            await _redisCircuitBreaker.ExecuteAsync(async () =>
            {
                var cacheKey = $"saldo:{data:yyyy-MM-dd}";
                var saldoAtual = await _cache.GetAsync<SaldoCacheDto>(cacheKey);
                
                if (saldoAtual != null)
                {
                    saldoAtual.Saldo += valor;
                    await _cache.SetAsync(cacheKey, saldoAtual, TimeSpan.FromHours(24));
                }
            });
            
            _logger.LogInformation("Saldo atualizado para data {Data}. Valor: {Valor}", data, valor);
            
            // Verificar saldo baixo
            var saldoTotal = await ObterSaldoAtual(data, cancellationToken);
            if (saldoTotal < 0)
            {
                _logger.LogWarning("Saldo baixo detectado para data {Data}: {Saldo}", data, saldoTotal);
                // Publicar evento de alerta (será implementado no worker de notificações)
            }
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Circuit breaker aberto - mensagem será reenviada para DLQ");
            throw;
        }
    }

    private async Task<decimal> ObterSaldoAtual(DateOnly data, CancellationToken cancellationToken)
    {
        await using var conn = await _dataSource.OpenConnectionAsync(cancellationToken);
        await using var cmd = new NpgsqlCommand(
            "SELECT saldo FROM saldo_diario WHERE data = @data",
            conn);
        cmd.Parameters.AddWithValue("@data", data.ToDateTime(TimeOnly.MinValue));
        
        var result = await cmd.ExecuteScalarAsync(cancellationToken);
        return result == DBNull.Value ? 0 : Convert.ToDecimal(result);
    }
    
    private record SaldoCacheDto
    {
        public decimal Saldo { get; set; }
        public DateTime AtualizadoEm { get; set; }
    }
}

```

__7.3 Notificacoes.Worker / NotificacoesWorker.cs__

```

using System.Text.Json;
using System.Net.Mail;
using Confluent.Kafka;
using FluxoCaixa.BuildingBlocks.Events;
using Polly;

namespace Notificacoes.Worker;

public class NotificacoesWorker : BackgroundService
{
    private readonly IConsumer<string, string> _consumer;
    private readonly IAsyncPolicy _smtpCircuitBreaker;
    private readonly ILogger<NotificacoesWorker> _logger;
    private readonly IConfiguration _configuration;

    public NotificacoesWorker(
        IConsumer<string, string> consumer,
        [FromKeyedServices("SmtpCircuitBreaker")] IAsyncPolicy smtpCircuitBreaker,
        ILogger<NotificacoesWorker> logger,
        IConfiguration configuration)
    {
        _consumer = consumer;
        _smtpCircuitBreaker = smtpCircuitBreaker;
        _logger = logger;
        _configuration = configuration;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _consumer.Subscribe("consolidacao");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var consumeResult = _consumer.Consume(stoppingToken);
                var evento = JsonSerializer.Deserialize<SaldoBaixoAlertadoEvent>(consumeResult.Message.Value);
                
                if (evento != null)
                {
                    await EnviarAlertaSaldoBaixoAsync(evento, stoppingToken);
                }
                
                _consumer.Commit(consumeResult);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Erro no worker de notificações");
                await Task.Delay(5000, stoppingToken);
            }
        }
        
        _consumer.Close();
    }

    private async Task EnviarAlertaSaldoBaixoAsync(SaldoBaixoAlertadoEvent evento, CancellationToken cancellationToken)
    {
        try
        {
            await _smtpCircuitBreaker.ExecuteAsync(async () =>
            {
                using var client = new SmtpClient(_configuration["Smtp:Host"], int.Parse(_configuration["Smtp:Port"]!));
                client.Credentials = new System.Net.NetworkCredential(
                    _configuration["Smtp:Username"], 
                    _configuration["Smtp:Password"]);
                client.EnableSsl = true;

                var mail = new MailMessage
                {
                    From = new MailAddress(_configuration["Smtp:From"]!, "Fluxo Caixa"),
                    Subject = "⚠️ Alerta: Saldo Diário Baixo",
                    Body = $@"
                        <h2>⚠️ Alerta: Saldo Diário Baixo</h2>
                        <p>Olá,</p>
                        <p>Seu saldo diário está baixo!</p>
                        <ul>
                            <li><strong>Data:</strong> {evento.Data:dd/MM/yyyy}</li>
                            <li><strong>Saldo Atual:</strong> R$ {evento.SaldoAtual:F2}</li>
                            <li><strong>Limite de Alerta:</strong> R$ {evento.LimiteAlerta:F2}</li>
                        </ul>
                        <p>Acesse o sistema para mais detalhes.</p>
                        <p>Atenciosamente,<br/>Equipe Fluxo Caixa</p>",
                    IsBodyHtml = true
                };
                mail.To.Add(evento.EmailUsuario);

                await client.SendMailAsync(mail, cancellationToken);
                _logger.LogInformation("E-mail enviado para {Email}", evento.EmailUsuario);
            });
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("SMTP circuit breaker aberto - e-mail não enviado");
        }
    }
}
```

__7.4 Auditoria.Worker / AuditoriaWorker.cs__

```
using System.Text.Json;
using Confluent.Kafka;
using FluxoCaixa.BuildingBlocks.Events;
using MongoDB.Driver;
using Polly;

namespace Auditoria.Worker;

public class AuditoriaWorker : BackgroundService
{
    private readonly IConsumer<string, string> _consumer;
    private readonly IMongoCollection<AuditoriaEvento> _auditoriaCollection;
    private readonly IAsyncPolicy _mongoCircuitBreaker;
    private readonly ILogger<AuditoriaWorker> _logger;

    public AuditoriaWorker(
        IConsumer<string, string> consumer,
        IMongoDatabase mongoDatabase,
        [FromKeyedServices("MongoCircuitBreaker")] IAsyncPolicy mongoCircuitBreaker,
        ILogger<AuditoriaWorker> logger)
    {
        _consumer = consumer;
        _auditoriaCollection = mongoDatabase.GetCollection<AuditoriaEvento>("auditoria");
        _mongoCircuitBreaker = mongoCircuitBreaker;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _consumer.Subscribe(new[] { "lancamentos", "consolidacao", "notificacoes" });
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var consumeResult = _consumer.Consume(stoppingToken);
                
                await _mongoCircuitBreaker.ExecuteAsync(async () =>
                {
                    var auditoria = new AuditoriaEvento
                    {
                        Id = Guid.NewGuid(),
                        Tipo = consumeResult.Topic,
                        Dados = consumeResult.Message.Value,
                        UsuarioId = ObterUsuarioIdDoEvento(consumeResult.Message.Value),
                        OcorridoEm = DateTime.UtcNow,
                        Origem = consumeResult.Topic
                    };
                    
                    await _auditoriaCollection.InsertOneAsync(auditoria, cancellationToken: stoppingToken);
                    _logger.LogDebug("Evento de auditoria salvo: {Tipo}", consumeResult.Topic);
                });
                
                _consumer.Commit(consumeResult);
            }
            catch (BrokenCircuitException)
            {
                _logger.LogError("MongoDB circuit breaker aberto - eventos não estão sendo auditados");
                await Task.Delay(10000, stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Erro no worker de auditoria");
                await Task.Delay(5000, stoppingToken);
            }
        }
        
        _consumer.Close();
    }

    private static Guid ObterUsuarioIdDoEvento(string messageValue)
    {
        try
        {
            using var doc = JsonDocument.Parse(messageValue);
            if (doc.RootElement.TryGetProperty("UsuarioId", out var usuarioIdElement))
            {
                return Guid.Parse(usuarioIdElement.GetString()!);
            }
        }
        catch
        {
            // Ignorar
        }
        return Guid.Empty;
    }
}

public class AuditoriaEvento
{
    public Guid Id { get; set; }
    public string Tipo { get; set; } = string.Empty;
    public string Dados { get; set; } = string.Empty;
    public Guid UsuarioId { get; set; }
    public DateTime OcorridoEm { get; set; }
    public string Origem { get; set; } = string.Empty;
}

```

<a id="estrutura"></a>

### 8. Kubernetes Deployments (GKE)

__8.1 auth-api.yaml__

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lancamentos-api
  namespace: fluxo-caixa
  labels:
    app: lancamentos-api
    environment: production
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lancamentos-api
  template:
    metadata:
      labels:
        app: lancamentos-api
        environment: production
        tier: backend
    spec:
      containers:
      - name: lancamentos-api
        image: gcr.io/fluxo-caixa/lancamentos-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8082
          name: http
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__LancamentosDb
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: lancamentos-db-connection
        - name: Kafka__BootstrapServers
          valueFrom:
            secretKeyRef:
              name: kafka-secrets
              key: bootstrap-servers
        - name: Jwt__Secret
          valueFrom:
            secretKeyRef:
              name: jwt-secrets
              key: secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8082
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8082
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: lancamentos-api
  namespace: fluxo-caixa
spec:
  selector:
    app: lancamentos-api
  ports:
  - port: 8082
    targetPort: 8082
    name: http
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: lancamentos-api-hpa
  namespace: fluxo-caixa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: lancamentos-api
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80


```


__8.2 lancamentos-api.yaml__

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lancamentos-api
  namespace: fluxo-caixa
  labels:
    app: lancamentos-api
    environment: production
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lancamentos-api
  template:
    metadata:
      labels:
        app: lancamentos-api
        environment: production
        tier: backend
    spec:
      containers:
      - name: lancamentos-api
        image: gcr.io/fluxo-caixa/lancamentos-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8082
          name: http
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__LancamentosDb
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: lancamentos-db-connection
        - name: Kafka__BootstrapServers
          valueFrom:
            secretKeyRef:
              name: kafka-secrets
              key: bootstrap-servers
        - name: Jwt__Secret
          valueFrom:
            secretKeyRef:
              name: jwt-secrets
              key: secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8082
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8082
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: lancamentos-api
  namespace: fluxo-caixa
spec:
  selector:
    app: lancamentos-api
  ports:
  - port: 8082
    targetPort: 8082
    name: http
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: lancamentos-api-hpa
  namespace: fluxo-caixa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: lancamentos-api
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

```

__8.3 consolidacao-api.yaml__

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: consolidacao-api
  namespace: fluxo-caixa
  labels:
    app: consolidacao-api
    environment: production
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: consolidacao-api
  template:
    metadata:
      labels:
        app: consolidacao-api
        environment: production
        tier: backend
    spec:
      containers:
      - name: consolidacao-api
        image: gcr.io/fluxo-caixa/consolidacao-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8083
          name: http
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__ReadDb
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: read-db-connection
        - name: Redis__ConnectionString
          valueFrom:
            secretKeyRef:
              name: redis-secrets
              key: connection-string
        - name: Jwt__Secret
          valueFrom:
            secretKeyRef:
              name: jwt-secrets
              key: secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8083
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8083
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: consolidacao-api
  namespace: fluxo-caixa
spec:
  selector:
    app: consolidacao-api
  ports:
  - port: 8083
    targetPort: 8083
    name: http
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: consolidacao-api-hpa
  namespace: fluxo-caixa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: consolidacao-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

```

__8.4 relatorios-api.yaml__

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: relatorios-api
  namespace: fluxo-caixa
  labels:
    app: relatorios-api
    environment: production
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: relatorios-api
  template:
    metadata:
      labels:
        app: relatorios-api
        environment: production
        tier: backend
    spec:
      containers:
      - name: relatorios-api
        image: gcr.io/fluxo-caixa/relatorios-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8084
          name: http
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__ReadDb
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: read-db-connection
        - name: Jwt__Secret
          valueFrom:
            secretKeyRef:
              name: jwt-secrets
              key: secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8084
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8084
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: relatorios-api
  namespace: fluxo-caixa
spec:
  selector:
    app: relatorios-api
  ports:
  - port: 8084
    targetPort: 8084
    name: http
  type: ClusterIP

```

__8.5 consolidacao-worker.yaml__

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: consolidacao-worker
  namespace: fluxo-caixa
  labels:
    app: consolidacao-worker
    environment: production
    tier: worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: consolidacao-worker
  template:
    metadata:
      labels:
        app: consolidacao-worker
        environment: production
        tier: worker
    spec:
      containers:
      - name: consolidacao-worker
        image: gcr.io/fluxo-caixa/consolidacao-worker:latest
        imagePullPolicy: Always
        env:
        - name: ConnectionStrings__ReadDb
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: read-db-connection
        - name: Redis__ConnectionString
          valueFrom:
            secretKeyRef:
              name: redis-secrets
              key: connection-string
        - name: Kafka__BootstrapServers
          valueFrom:
            secretKeyRef:
              name: kafka-secrets
              key: bootstrap-servers
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - /bin/grpc_health_probe
            - -addr=:5000
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: consolidacao-worker-hpa
  namespace: fluxo-caixa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: consolidacao-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: kafka_consumer_lag
      target:
        type: AverageValue
        averageValue: "1000"

```

__8.6 notificacoes-worker.yaml__

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notificacoes-worker
  namespace: fluxo-caixa
  labels:
    app: notificacoes-worker
    environment: production
    tier: worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notificacoes-worker
  template:
    metadata:
      labels:
        app: notificacoes-worker
        environment: production
        tier: worker
    spec:
      containers:
      - name: notificacoes-worker
        image: gcr.io/fluxo-caixa/notificacoes-worker:latest
        imagePullPolicy: Always
        env:
        - name: Kafka__BootstrapServers
          valueFrom:
            secretKeyRef:
              name: kafka-secrets
              key: bootstrap-servers
        - name: Smtp__Host
          valueFrom:
            secretKeyRef:
              name: smtp-secrets
              key: host
        - name: Smtp__Port
          valueFrom:
            secretKeyRef:
              name: smtp-secrets
              key: port
        - name: Smtp__Username
          valueFrom:
            secretKeyRef:
              name: smtp-secrets
              key: username
        - name: Smtp__Password
          valueFrom:
            secretKeyRef:
              name: smtp-secrets
              key: password
        - name: Smtp__From
          valueFrom:
            secretKeyRef:
              name: smtp-secrets
              key: from
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"

```

__8.7 auditoria-worker.yaml__

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auditoria-worker
  namespace: fluxo-caixa
  labels:
    app: auditoria-worker
    environment: production
    tier: worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auditoria-worker
  template:
    metadata:
      labels:
        app: auditoria-worker
        environment: production
        tier: worker
    spec:
      containers:
      - name: auditoria-worker
        image: gcr.io/fluxo-caixa/auditoria-worker:latest
        imagePullPolicy: Always
        env:
        - name: Kafka__BootstrapServers
          valueFrom:
            secretKeyRef:
              name: kafka-secrets
              key: bootstrap-servers
        - name: MongoDB__ConnectionString
          valueFrom:
            secretKeyRef:
              name: mongodb-secrets
              key: connection-string
        - name: MongoDB__Database
          value: "fluxocaixa_auditoria"
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"

```

__8.8 api-gateway.yaml__

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: fluxo-caixa
  labels:
    app: api-gateway
    environment: production
    tier: gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
        environment: production
        tier: gateway
    spec:
      containers:
      - name: api-gateway
        image: gcr.io/fluxo-caixa/api-gateway:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: Jwt__Secret
          valueFrom:
            secretKeyRef:
              name: jwt-secrets
              key: secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: fluxo-caixa
spec:
  selector:
    app: api-gateway
  ports:
  - port: 80
    targetPort: 8080
    name: http
  type: LoadBalancer
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
  namespace: fluxo-caixa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

```

__8.9 ConfigMap e Secrets__

```
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: fluxo-caixa
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Kafka__Topic__Lancamentos: "lancamentos"
  Kafka__Topic__Consolidacao: "consolidacao"
  Kafka__Topic__Notificacoes: "notificacoes"
  Kafka__Topic__Auditoria: "auditoria"
  Redis__CacheTTLHours: "24"
  Jwt__ExpirationHours: "8"
---
# secrets.yaml (valores devem ser preenchidos com dados reais)
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
  namespace: fluxo-caixa
type: Opaque
stringData:
  auth-db-connection: "Host=postgres-auth;Database=fluxocaixa_auth;Username=postgres;Password=CHANGE_ME"
  lancamentos-db-connection: "Host=postgres-lanc;Database=fluxocaixa_lancamentos;Username=postgres;Password=CHANGE_ME"
  read-db-connection: "Host=postgres-read;Database=fluxocaixa_read;Username=postgres;Password=CHANGE_ME"
---
apiVersion: v1
kind: Secret
metadata:
  name: redis-secrets
  namespace: fluxo-caixa
type: Opaque
stringData:
  connection-string: "redis:6379,password=CHANGE_ME,abortConnect=false"
---
apiVersion: v1
kind: Secret
metadata:
  name: kafka-secrets
  namespace: fluxo-caixa
type: Opaque
stringData:
  bootstrap-servers: "kafka:9092"
---
apiVersion: v1
kind: Secret
metadata:
  name: jwt-secrets
  namespace: fluxo-caixa
type: Opaque
stringData:
  secret: "SUA_CHAVE_SECRETA_AQUI_COM_MAIS_DE_32_CARACTERES"
---
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secrets
  namespace: fluxo-caixa
type: Opaque
stringData:
  connection-string: "mongodb://mongodb:27017"

```

__8.10 Namespace e Network Policy__

```
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fluxo-caixa
---
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-gateway
  namespace: fluxo-caixa
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - port: 8081
    - port: 8082
    - port: 8083
    - port: 8084
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-worker-kafka
  namespace: fluxo-caixa
spec:
  podSelector:
    matchLabels:
      tier: worker
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: kafka
    ports:
    - port: 9092
  policyTypes:
  - Egress

```


__8.11 Dockerfile (Base)__

```
# Dockerfile.base
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy csproj files and restore dependencies
COPY ["BuildingBlocks/FluxoCaixa.BuildingBlocks/FluxoCaixa.BuildingBlocks.csproj", "BuildingBlocks/FluxoCaixa.BuildingBlocks/"]
COPY ["BuildingBlocks/FluxoCaixa.CircuitBreaker/FluxoCaixa.CircuitBreaker.csproj", "BuildingBlocks/FluxoCaixa.CircuitBreaker/"]
RUN dotnet restore "BuildingBlocks/FluxoCaixa.BuildingBlocks/FluxoCaixa.BuildingBlocks.csproj"
RUN dotnet restore "BuildingBlocks/FluxoCaixa.CircuitBreaker/FluxoCaixa.CircuitBreaker.csproj"

# Copy all source code
COPY . .

# Build specific service (parametrizado)
ARG SERVICE_NAME
WORKDIR /src/Services/${SERVICE_NAME}
RUN dotnet publish -c Release -o /app/publish

# Runtime image
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
EXPOSE 8080
EXPOSE 8081
EXPOSE 8082
EXPOSE 8083
EXPOSE 8084

COPY --from=build /app/publish .

ENTRYPOINT ["dotnet", "FluxoCaixa.Service.dll"]

```

__8.12 Dockerfile específico para cada serviço (exemplo: auth-api)__


```
# Dockerfile.auth-api
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY . .
RUN dotnet restore "Services/Auth.Api/Auth.Api.csproj"
RUN dotnet publish "Services/Auth.Api/Auth.Api.csproj" -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
EXPOSE 8081
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Auth.Api.dll"]

```

__8.13 Comandos para Deploy no GKE__

```
# 1. Criar namespace
kubectl create namespace fluxo-caixa

# 2. Aplicar secrets (primeiro)
kubectl apply -f deployments/secrets.yaml

# 3. Aplicar configmap
kubectl apply -f deployments/configmap.yaml

# 4. Aplicar network policies
kubectl apply -f deployments/network-policy.yaml

# 5. Aplicar deployments e services
kubectl apply -f deployments/auth-api.yaml
kubectl apply -f deployments/lancamentos-api.yaml
kubectl apply -f deployments/consolidacao-api.yaml
kubectl apply -f deployments/relatorios-api.yaml
kubectl apply -f deployments/consolidacao-worker.yaml
kubectl apply -f deployments/notificacoes-worker.yaml
kubectl apply -f deployments/auditoria-worker.yaml
kubectl apply -f deployments/api-gateway.yaml

# 6. Verificar status
kubectl get pods -n fluxo-caixa
kubectl get services -n fluxo-caixa
kubectl get hpa -n fluxo-caixa

# 7. Ver logs de um pod específico
kubectl logs -f deployment/auth-api -n fluxo-caixa

# 8. Escalar manualmente (se necessário)
kubectl scale deployment lancamentos-api --replicas=5 -n fluxo-caixa

# 9. Rollback em caso de problema
kubectl rollout undo deployment/lancamentos-api -n fluxo-caixa

# 10. Verificar recursos utilizados
kubectl top pods -n fluxo-caixa
kubectl top nodes

```









