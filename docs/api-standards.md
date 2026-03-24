# Padrões de API REST

## Princípios Gerais

As APIs da plataforma seguem o padrão **RESTful** com as seguintes convenções:

- Comunicação via **HTTPS** (TLS 1.3 obrigatório)
- Formato de dados: **JSON** (Content-Type: `application/json`)
- Autenticação via **Bearer Token JWT** emitido pelo KeyCloak
- **Versionamento** obrigatório em todos os endpoints
- Respostas de erro no formato **RFC 7807** (Problem Details)

---

## Versionamento

Utilize versionamento na URL:

```
/api/v1/orders
/api/v2/orders
```

```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});
```

```csharp
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public sealed class OrdersController : ControllerBase { }
```

---

## Convenções de Nomenclatura

| Regra | Correto | Incorreto |
|-------|---------|-----------|
| Substantivos no plural | `/orders` | `/getOrders` |
| Letras minúsculas | `/orders` | `/Orders` |
| Separador hífen | `/order-items` | `/orderItems` ou `/order_items` |
| Sem verbos na URL | `GET /orders` | `GET /getOrders` |
| Recursos aninhados | `/orders/{id}/items` | `/getOrderItems?orderId=` |

---

## Métodos HTTP e Respostas

| Operação | Método | URL | Sucesso | Erro |
|----------|--------|-----|---------|------|
| Listar | `GET` | `/orders` | `200 OK` | `400`, `401`, `403` |
| Buscar por ID | `GET` | `/orders/{id}` | `200 OK` | `404` |
| Criar | `POST` | `/orders` | `201 Created` | `400`, `409` |
| Atualizar completo | `PUT` | `/orders/{id}` | `200 OK` | `400`, `404` |
| Atualizar parcial | `PATCH` | `/orders/{id}` | `200 OK` | `400`, `404` |
| Deletar | `DELETE` | `/orders/{id}` | `204 No Content` | `404` |

---

## Contratos de Requisição e Resposta

### Request

```json
POST /api/v1/orders
Content-Type: application/json
Authorization: Bearer <token>
X-Correlation-ID: 3fa85f64-5717-4562-b3fc-2c963f66afa6

{
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "items": [
    {
      "productId": "7fa85f64-5717-4562-b3fc-2c963f66afa1",
      "quantity": 2
    }
  ]
}
```

### Response – Sucesso (201 Created)

```json
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/orders/9fa85f64-5717-4562-b3fc-2c963f66afa9
X-Correlation-ID: 3fa85f64-5717-4562-b3fc-2c963f66afa6

{
  "id": "9fa85f64-5717-4562-b3fc-2c963f66afa9",
  "createdAt": "2026-03-24T12:00:00Z"
}
```

### Response – Erro (RFC 7807 Problem Details)

```json
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://example.com/errors/validation",
  "title": "Erro de validação",
  "status": 422,
  "detail": "Um ou mais campos possuem valores inválidos.",
  "instance": "/api/v1/orders",
  "errors": {
    "customerId": ["O campo customerId é obrigatório."],
    "items": ["O pedido deve ter pelo menos 1 item."]
  }
}
```

---

## Paginação

Use paginação baseada em offset para listagens:

**Request:**
```
GET /api/v1/orders?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 150,
    "totalPages": 8,
    "hasNextPage": true,
    "hasPreviousPage": false
  }
}
```

```csharp
// Application/DTOs/PagedResult.cs
public sealed record PagedResult<T>(
    IEnumerable<T> Data,
    int Page,
    int PageSize,
    int TotalItems)
{
    public int TotalPages => (int)Math.Ceiling(TotalItems / (double)PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}
```

---

## Headers Obrigatórios

| Header | Tipo | Descrição |
|--------|------|-----------|
| `Authorization` | Request | Bearer JWT Token |
| `X-Correlation-ID` | Request / Response | ID para rastreamento distribuído (UUID v4) |
| `Content-Type` | Request (body) | `application/json` |
| `Accept` | Request | `application/json` |

```csharp
// Api/Middlewares/CorrelationIdMiddleware.cs
public sealed class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-ID";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        context.Items[CorrelationIdHeader] = correlationId;
        context.Response.Headers[CorrelationIdHeader] = correlationId;

        using var scope = context.RequestServices
            .GetRequiredService<ILogger<CorrelationIdMiddleware>>()
            .BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId });

        await _next(context);
    }
}
```

---

## Tratamento Global de Erros

```csharp
// Api/Middlewares/ExceptionHandlingMiddleware.cs
public sealed class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (NotFoundException ex)
        {
            _logger.LogWarning(ex, "Recurso não encontrado");
            await WriteProblemDetailsAsync(context, StatusCodes.Status404NotFound, ex.Message);
        }
        catch (DomainException ex)
        {
            _logger.LogWarning(ex, "Erro de domínio");
            await WriteProblemDetailsAsync(context, StatusCodes.Status422UnprocessableEntity, ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro interno inesperado");
            await WriteProblemDetailsAsync(context, StatusCodes.Status500InternalServerError,
                "Ocorreu um erro interno. Por favor, tente novamente mais tarde.");
        }
    }

    private static async Task WriteProblemDetailsAsync(HttpContext context, int statusCode, string detail)
    {
        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/problem+json";

        var problem = new ProblemDetails
        {
            Status = statusCode,
            Detail = detail,
            Instance = context.Request.Path
        };

        await context.Response.WriteAsJsonAsync(problem);
    }
}
```

---

## Validação de Entrada

Use **FluentValidation** para validar requests:

```csharp
// Application/UseCases/CreateOrder/CreateOrderCommandValidator.cs
public sealed class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty().WithMessage("O campo customerId é obrigatório.");

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("O pedido deve ter pelo menos 1 item.")
            .ForEach(item =>
            {
                item.ChildRules(i =>
                {
                    i.RuleFor(x => x.ProductId).NotEmpty();
                    i.RuleFor(x => x.Quantity).GreaterThan(0)
                        .WithMessage("A quantidade deve ser maior que zero.");
                });
            });
    }
}
```

---

## Autenticação e Autorização

```csharp
// Program.cs – Configuração JWT/KeyCloak
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["KeyCloak:Authority"];
        options.Audience = builder.Configuration["KeyCloak:Audience"];
        options.RequireHttpsMetadata = true;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromSeconds(30)
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("OrdersRead", policy =>
        policy.RequireAuthenticatedUser().RequireClaim("scope", "orders:read"));
    options.AddPolicy("OrdersWrite", policy =>
        policy.RequireAuthenticatedUser().RequireClaim("scope", "orders:write"));
});
```

```csharp
// Controller com autorização por política
[HttpGet("{id:guid}")]
[Authorize(Policy = "OrdersRead")]
public async Task<IActionResult> GetOrder(Guid id, CancellationToken cancellationToken) { }

[HttpPost]
[Authorize(Policy = "OrdersWrite")]
public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request, CancellationToken cancellationToken) { }
```

---

## Documentação OpenAPI / Swagger

Toda API deve expor sua documentação via Swagger UI:

```csharp
// Program.cs
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Core API",
        Version = "v1",
        Description = "API principal da plataforma"
    });

    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        Description = "JWT Token do KeyCloak"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
            },
            Array.Empty<string>()
        }
    });

    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    options.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFile));
});
```
