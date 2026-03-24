# Comunicação REST — Boas Práticas

## Convenções de Nomenclatura

| Recurso | Endpoint |
|---|---|
| Listar pedidos | `GET /v1/orders` |
| Obter pedido | `GET /v1/orders/{id}` |
| Criar pedido | `POST /v1/orders` |
| Atualizar pedido | `PUT /v1/orders/{id}` |
| Atualizar parcialmente | `PATCH /v1/orders/{id}` |
| Excluir pedido | `DELETE /v1/orders/{id}` |
| Confirmar pedido (ação) | `POST /v1/orders/{id}/confirm` |

**Regras:**
- Utilize **substantivos no plural** para nomes de recursos (`orders`, `customers`).
- Evite verbos nas rotas — use os métodos HTTP para expressar a ação.
- Para ações de negócio que não mapeiam diretamente para CRUD, use sub-recursos de ação (`/confirm`, `/cancel`).
- Sempre versione as APIs: `/v1/`, `/v2/`.

---

## Códigos de Status HTTP

| Situação | Código |
|---|---|
| Recurso criado | `201 Created` + header `Location` |
| Operação bem-sucedida sem corpo | `204 No Content` |
| Recurso não encontrado | `404 Not Found` |
| Erro de validação | `400 Bad Request` |
| Não autenticado | `401 Unauthorized` |
| Sem permissão | `403 Forbidden` |
| Conflito de estado | `409 Conflict` |
| Erro interno | `500 Internal Server Error` |

---

## Formato de Resposta

Use **Problem Details** (RFC 7807) para respostas de erro:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "Validation failed",
  "status": 400,
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "errors": {
    "CustomerId": ["'CustomerId' must not be empty."]
  }
}
```

Configure Problem Details no `Program.cs`:

```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
```

```csharp
// GlobalExceptionHandler.cs
public sealed class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        logger.LogError(exception, "Unhandled exception: {Message}", exception.Message);

        var (statusCode, title) = exception switch
        {
            DomainException => (StatusCodes.Status422UnprocessableEntity, "Domain rule violation"),
            NotFoundException => (StatusCodes.Status404NotFound, "Resource not found"),
            _ => (StatusCodes.Status500InternalServerError, "An unexpected error occurred")
        };

        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = exception.Message
        }, cancellationToken);

        return true;
    }
}
```

---

## Paginação

Utilize paginação baseada em cursor ou offset. Padrão recomendado:

**Request:**
```
GET /v1/orders?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 150,
    "totalPages": 8
  }
}
```

---

## Versionamento

Configure o versionamento de API com `Asp.Versioning`:

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1);
    options.ReportApiVersions = true;
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"));
});
```

---

## Documentação com OpenAPI (Swagger)

```csharp
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((doc, _, _) =>
    {
        doc.Info.Title = "Orders API";
        doc.Info.Version = "v1";
        doc.Info.Contact = new OpenApiContact
        {
            Name = "Team",
            Email = "team@example.com"
        };
        return Task.CompletedTask;
    });
});
```

---

## Resiliência no consumo de APIs (BFF → API Core)

```csharp
// Use Polly via Microsoft.Extensions.Http.Resilience (.NET Core 10)
builder.Services.AddHttpClient<IOrderApiClient, OrderApiClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.MaxRetryAttempts = 3;
        options.CircuitBreaker.FailureRatio = 0.5;
    });
```

---

## Headers obrigatórios

| Header | Obrigatório | Descrição |
|---|---|---|
| `Authorization: Bearer <token>` | Sim | Token JWT de autenticação |
| `X-Correlation-Id` | Sim | ID de correlação para rastreamento distribuído |
| `Content-Type: application/json` | Sim (POST/PUT/PATCH) | Tipo do corpo da requisição |
| `Accept: application/json` | Recomendado | Tipo esperado na resposta |

---

## Segurança

- Sempre utilize **HTTPS** — nunca exponha endpoints HTTP em produção.
- Valide e sanitize toda entrada de dados antes de processá-la.
- Não exponha detalhes internos (stack traces, mensagens de banco) em respostas de produção.
- Configure rate limiting para prevenir abuso:

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 10;
    });
});
```
