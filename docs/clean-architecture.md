# Clean Architecture – Padrão de Estrutura dos Projetos .NET

## Visão Geral

Todos os projetos .NET Core da plataforma (BFF e Core API) devem seguir o padrão **Clean Architecture** (Arquitetura Limpa), proposto por Robert C. Martin (Uncle Bob). Este padrão garante:

- **Independência de frameworks**: o domínio não depende de nenhum framework externo
- **Testabilidade**: regras de negócio podem ser testadas sem UI, banco de dados ou qualquer agente externo
- **Independência de UI**: a interface pode mudar sem impactar o domínio
- **Independência de banco de dados**: o domínio não conhece o banco usado
- **Independência de serviços externos**: as regras de negócio não conhecem detalhes externos

---

## Regra de Dependência

> As dependências no código-fonte devem apontar **somente para dentro** — em direção às políticas de alto nível.

```
┌─────────────────────────────────────────────────────────────┐
│                     Frameworks & Drivers                    │
│  (Web API, EF Core, Dapper, GCP SDK, HTTP Clients, etc.)   │
│                                                             │
│    ┌─────────────────────────────────────────────────┐     │
│    │              Interface Adapters                  │     │
│    │  (Controllers, Presenters, Repositories Impl.)  │     │
│    │                                                  │     │
│    │    ┌───────────────────────────────────────┐    │     │
│    │    │          Application Layer             │    │     │
│    │    │   (Use Cases / Application Services)  │    │     │
│    │    │                                        │    │     │
│    │    │    ┌─────────────────────────────┐    │    │     │
│    │    │    │        Domain Layer          │    │    │     │
│    │    │    │  (Entities, Value Objects,   │    │    │     │
│    │    │    │   Domain Services, Events)   │    │    │     │
│    │    │    └─────────────────────────────┘    │    │     │
│    │    └───────────────────────────────────────┘    │     │
│    └─────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘

         ◄── Dependências sempre apontam para dentro ──►
```

---

## Estrutura de Pastas

### Core API

```
src/
├── MyApp.Domain/                    # Camada de Domínio (sem dependências externas)
│   ├── Entities/
│   │   ├── Order.cs
│   │   └── Customer.cs
│   ├── ValueObjects/
│   │   ├── Money.cs
│   │   └── Email.cs
│   ├── Enums/
│   │   └── OrderStatus.cs
│   ├── Events/                      # Domain Events
│   │   └── OrderCreatedEvent.cs
│   ├── Exceptions/
│   │   └── DomainException.cs
│   └── Interfaces/                  # Portas (contratos que o domínio define)
│       ├── Repositories/
│       │   └── IOrderRepository.cs
│       └── Services/
│           └── IPaymentService.cs
│
├── MyApp.Application/               # Camada de Aplicação
│   ├── UseCases/
│   │   ├── CreateOrder/
│   │   │   ├── CreateOrderCommand.cs
│   │   │   ├── CreateOrderHandler.cs
│   │   │   └── CreateOrderResponse.cs
│   │   └── GetOrder/
│   │       ├── GetOrderQuery.cs
│   │       ├── GetOrderHandler.cs
│   │       └── GetOrderResponse.cs
│   ├── DTOs/
│   │   └── OrderDto.cs
│   ├── Mappings/
│   │   └── OrderMappingProfile.cs
│   └── Interfaces/                  # Interfaces de serviços de aplicação
│       └── IOrderApplicationService.cs
│
├── MyApp.Infrastructure/            # Camada de Infraestrutura
│   ├── Persistence/
│   │   ├── Repositories/
│   │   │   └── OrderRepository.cs   # Implementa IOrderRepository
│   │   ├── Context/
│   │   │   └── AppDbContext.cs       # EF Core DbContext
│   │   └── Migrations/
│   ├── ExternalServices/
│   │   ├── PaymentServiceClient.cs  # Implementa IPaymentService
│   │   └── GcpPubSubPublisher.cs
│   ├── Messaging/
│   │   └── EventPublisher.cs
│   └── DependencyInjection/
│       └── InfrastructureExtensions.cs
│
└── MyApp.Api/                       # Camada de Apresentação (Web API)
    ├── Controllers/
    │   └── OrdersController.cs
    ├── Middlewares/
    │   ├── ExceptionHandlingMiddleware.cs
    │   └── CorrelationIdMiddleware.cs
    ├── Filters/
    │   └── ValidationFilter.cs
    ├── Program.cs
    └── appsettings.json

tests/
├── MyApp.UnitTests/
│   ├── Domain/
│   │   └── OrderTests.cs
│   └── Application/
│       └── CreateOrderHandlerTests.cs
└── MyApp.IntegrationTests/
    ├── Api/
    │   └── OrdersControllerTests.cs
    └── Infrastructure/
        └── OrderRepositoryTests.cs
```

### BFF API

```
src/
├── MyBff.Application/
│   ├── UseCases/
│   │   └── GetOrderSummary/
│   │       ├── GetOrderSummaryQuery.cs
│   │       └── GetOrderSummaryHandler.cs
│   └── Interfaces/
│       └── ICoreApiClient.cs
│
├── MyBff.Infrastructure/
│   ├── HttpClients/
│   │   └── CoreApiClient.cs        # Implementa ICoreApiClient
│   └── DependencyInjection/
│       └── InfrastructureExtensions.cs
│
└── MyBff.Api/
    ├── Controllers/
    │   └── OrderSummaryController.cs
    ├── Program.cs
    └── appsettings.json
```

---

## Camadas em Detalhe

### Domain Layer (Domínio)

A camada mais interna. **Não pode ter dependência de nenhum pacote NuGet externo** (exceto abstrações puras).

```csharp
// Domain/Entities/Order.cs
public sealed class Order
{
    public Guid Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public Money TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private Order() { } // Para EF Core

    public static Order Create(CustomerId customerId)
    {
        ArgumentNullException.ThrowIfNull(customerId);

        return new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Pending,
            TotalAmount = Money.Zero
        };
    }

    public void AddItem(Product product, int quantity)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Itens só podem ser adicionados a pedidos pendentes.");

        var item = OrderItem.Create(product, quantity);
        _items.Add(item);
        TotalAmount = TotalAmount.Add(item.Subtotal);
    }

    public void Confirm()
    {
        if (!_items.Any())
            throw new DomainException("Não é possível confirmar um pedido sem itens.");

        Status = OrderStatus.Confirmed;
        AddDomainEvent(new OrderConfirmedEvent(Id));
    }
}
```

```csharp
// Domain/ValueObjects/Money.cs
public sealed record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public static Money Zero => new(0, "BRL");

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Valor não pode ser negativo.");
        if (string.IsNullOrWhiteSpace(currency)) throw new DomainException("Moeda é obrigatória.");

        Amount = amount;
        Currency = currency;
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException("Não é possível somar moedas diferentes.");

        return new Money(Amount + other.Amount, Currency);
    }
}
```

```csharp
// Domain/Interfaces/Repositories/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IEnumerable<Order>> GetByCustomerIdAsync(Guid customerId, CancellationToken cancellationToken = default);
    Task AddAsync(Order order, CancellationToken cancellationToken = default);
    Task UpdateAsync(Order order, CancellationToken cancellationToken = default);
}
```

---

### Application Layer (Aplicação)

Orquestra o fluxo de um caso de uso. Conhece o domínio, mas não conhece infraestrutura.

```csharp
// Application/UseCases/CreateOrder/CreateOrderCommand.cs
public sealed record CreateOrderCommand(Guid CustomerId, IEnumerable<OrderItemRequest> Items);

public sealed record OrderItemRequest(Guid ProductId, int Quantity);
```

```csharp
// Application/UseCases/CreateOrder/CreateOrderHandler.cs
public sealed class CreateOrderHandler
{
    private readonly IOrderRepository _orderRepository;
    private readonly IProductRepository _productRepository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IEventPublisher _eventPublisher;

    public CreateOrderHandler(
        IOrderRepository orderRepository,
        IProductRepository productRepository,
        IUnitOfWork unitOfWork,
        IEventPublisher eventPublisher)
    {
        _orderRepository = orderRepository;
        _productRepository = productRepository;
        _unitOfWork = unitOfWork;
        _eventPublisher = eventPublisher;
    }

    public async Task<Guid> HandleAsync(CreateOrderCommand command, CancellationToken cancellationToken = default)
    {
        var order = Order.Create(new CustomerId(command.CustomerId));

        foreach (var itemRequest in command.Items)
        {
            var product = await _productRepository.GetByIdAsync(itemRequest.ProductId, cancellationToken)
                ?? throw new NotFoundException($"Produto {itemRequest.ProductId} não encontrado.");

            order.AddItem(product, itemRequest.Quantity);
        }

        await _orderRepository.AddAsync(order, cancellationToken);
        await _unitOfWork.CommitAsync(cancellationToken);
        await _eventPublisher.PublishAsync(order.DomainEvents, cancellationToken);

        return order.Id;
    }
}
```

---

### Infrastructure Layer (Infraestrutura)

Implementa as interfaces definidas no domínio. Conhece bancos de dados, APIs externas e frameworks.

```csharp
// Infrastructure/Persistence/Repositories/OrderRepository.cs
public sealed class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
    }

    public async Task AddAsync(Order order, CancellationToken cancellationToken = default)
    {
        await _context.Orders.AddAsync(order, cancellationToken);
    }

    public async Task UpdateAsync(Order order, CancellationToken cancellationToken = default)
    {
        _context.Orders.Update(order);
        await Task.CompletedTask;
    }

    public async Task<IEnumerable<Order>> GetByCustomerIdAsync(Guid customerId, CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .Where(o => o.CustomerId == new CustomerId(customerId))
            .ToListAsync(cancellationToken);
    }
}
```

---

### API Layer (Apresentação)

Responsável por receber e responder requisições HTTP. Não contém lógica de negócio.

```csharp
// Api/Controllers/OrdersController.cs
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
public sealed class OrdersController : ControllerBase
{
    private readonly CreateOrderHandler _createOrderHandler;
    private readonly GetOrderHandler _getOrderHandler;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(
        CreateOrderHandler createOrderHandler,
        GetOrderHandler getOrderHandler,
        ILogger<OrdersController> logger)
    {
        _createOrderHandler = createOrderHandler;
        _getOrderHandler = getOrderHandler;
        _logger = logger;
    }

    [HttpPost]
    [ProducesResponseType(typeof(CreateOrderResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateOrder(
        [FromBody] CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation("Criando pedido para o cliente {CustomerId}", request.CustomerId);

        var orderId = await _createOrderHandler.HandleAsync(
            new CreateOrderCommand(request.CustomerId, request.Items), cancellationToken);

        return CreatedAtAction(nameof(GetOrder), new { id = orderId }, new CreateOrderResponse(orderId));
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetOrder(Guid id, CancellationToken cancellationToken)
    {
        var order = await _getOrderHandler.HandleAsync(new GetOrderQuery(id), cancellationToken);
        return Ok(order);
    }
}
```

---

## Injeção de Dependência

```csharp
// Infrastructure/DependencyInjection/InfrastructureExtensions.cs
public static class InfrastructureExtensions
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseNpgsql(configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IEventPublisher, GcpPubSubPublisher>();

        return services;
    }
}
```

```csharp
// Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddScoped<CreateOrderHandler>();
builder.Services.AddScoped<GetOrderHandler>();
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();
app.UseMiddleware<CorrelationIdMiddleware>();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready");

app.Run();
```
