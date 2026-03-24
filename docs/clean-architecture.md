# Clean Architecture para APIs .NET Core 10

## Visão Geral

Clean Architecture organiza o código em círculos concêntricos de dependência. A regra fundamental é: **as dependências sempre apontam para dentro** — camadas externas dependem de camadas internas, nunca o contrário.

```
┌─────────────────────────────────────┐
│           Presentation / API        │  ← HTTP, Controllers, Middlewares
│  ┌───────────────────────────────┐  │
│  │       Infrastructure         │  │  ← BD, HTTP clients, mensageria, arquivos
│  │  ┌─────────────────────────┐  │  │
│  │  │      Application        │  │  │  ← Use Cases, DTOs, Interfaces
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │      Domain       │  │  │  │  ← Entidades, Value Objects, Regras
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

---

## Estrutura de Pastas Recomendada

```
src/
├── MyApi.Domain/
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Enums/
│   ├── Exceptions/
│   └── Interfaces/           ← Interfaces de repositório (apenas contratos)
│
├── MyApi.Application/
│   ├── UseCases/
│   │   └── Orders/
│   │       ├── CreateOrder/
│   │       │   ├── CreateOrderCommand.cs
│   │       │   ├── CreateOrderCommandHandler.cs
│   │       │   └── CreateOrderCommandValidator.cs
│   │       └── GetOrderById/
│   │           ├── GetOrderByIdQuery.cs
│   │           └── GetOrderByIdQueryHandler.cs
│   ├── DTOs/
│   ├── Interfaces/           ← Interfaces de serviços externos (email, storage…)
│   ├── Mappings/             ← AutoMapper ou mapeamento manual
│   └── DependencyInjection.cs
│
├── MyApi.Infrastructure/
│   ├── Persistence/
│   │   ├── Repositories/     ← Implementações dos repositórios
│   │   ├── Configurations/   ← EntityTypeConfiguration ou queries Dapper
│   │   └── AppDbContext.cs
│   ├── ExternalServices/     ← Clientes HTTP, Pub/Sub, Storage
│   └── DependencyInjection.cs
│
└── MyApi.Api/
    ├── Controllers/
    ├── Middlewares/
    ├── Filters/
    ├── Program.cs
    └── appsettings.json

tests/
├── MyApi.UnitTests/
├── MyApi.IntegrationTests/
└── MyApi.ArchitectureTests/  ← Testes de aderência arquitetural (NetArchTest)
```

---

## Camadas em Detalhe

### Domain

A camada mais interna. **Não possui dependência de nenhuma outra camada do projeto nem de frameworks externos** (exceto primitivas do .NET).

```csharp
// Entidade rica — regras de negócio residem no domínio
public sealed class Order
{
    private readonly List<OrderItem> _items = [];

    public Guid Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    private Order() { } // EF / Dapper

    public static Order Create(CustomerId customerId)
    {
        ArgumentNullException.ThrowIfNull(customerId);
        return new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Draft
        };
    }

    public void AddItem(ProductId productId, int quantity, decimal unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Não é possível adicionar itens a um pedido confirmado.");

        _items.Add(new OrderItem(productId, quantity, unitPrice));
    }

    public void Confirm()
    {
        if (_items.Count == 0)
            throw new DomainException("O pedido não possui itens.");

        Status = OrderStatus.Confirmed;
    }
}
```

```csharp
// Value Object imutável
public sealed record CustomerId(Guid Value)
{
    public static CustomerId New() => new(Guid.NewGuid());
    public static CustomerId From(Guid value) => new(value);
}
```

```csharp
// Interface do repositório — definida no Domain, implementada na Infrastructure
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task AddAsync(Order order, CancellationToken cancellationToken = default);
    Task UpdateAsync(Order order, CancellationToken cancellationToken = default);
}
```

---

### Application

Orquestra os casos de uso. Contém lógica de aplicação (fluxo), não de domínio.

Use o padrão **CQRS** (Command / Query Responsibility Segregation) com **MediatR**:

```csharp
// Command
public sealed record CreateOrderCommand(Guid CustomerId) : IRequest<Guid>;

// Handler
public sealed class CreateOrderCommandHandler(
    IOrderRepository orderRepository,
    IUnitOfWork unitOfWork)
    : IRequestHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(
        CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        var order = Order.Create(CustomerId.From(request.CustomerId));
        await orderRepository.AddAsync(order, cancellationToken);
        await unitOfWork.SaveChangesAsync(cancellationToken);
        return order.Id;
    }
}

// Validator (FluentValidation)
public sealed class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
    }
}
```

---

### Infrastructure

Implementa as interfaces definidas no Domain e na Application.

```csharp
// Implementação do repositório com Entity Framework
public sealed class OrderRepository(AppDbContext dbContext) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
        => await dbContext.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);

    public async Task AddAsync(Order order, CancellationToken cancellationToken = default)
        => await dbContext.Orders.AddAsync(order, cancellationToken);

    public Task UpdateAsync(Order order, CancellationToken cancellationToken = default)
    {
        dbContext.Orders.Update(order);
        return Task.CompletedTask;
    }
}
```

---

### API (Presentation)

Controllers finos — apenas recebem a requisição HTTP, invocam o MediatR e retornam a resposta.

```csharp
[ApiController]
[Route("v1/orders")]
public sealed class OrdersController(ISender sender) : ControllerBase
{
    [HttpPost]
    [ProducesResponseType<Guid>(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        var command = new CreateOrderCommand(request.CustomerId);
        var orderId = await sender.Send(command, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id = orderId }, orderId);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        var query = new GetOrderByIdQuery(id);
        var result = await sender.Send(query, cancellationToken);
        return result is null ? NotFound() : Ok(result);
    }
}
```

---

## Registro de Dependências

Cada camada expõe um método de extensão para registro no DI container:

```csharp
// MyApi.Application/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(DependencyInjection).Assembly));
        services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);
        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
        return services;
    }
}

// MyApi.Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseNpgsql(configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        return services;
    }
}

// MyApi.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration);
```

---

## Testes de Aderência Arquitetural

Use **NetArchTest** para garantir que as regras arquiteturais são respeitadas:

```csharp
[Fact]
public void Domain_Should_Not_HaveDependencyOn_Application()
{
    var result = Types.InAssembly(DomainAssembly)
        .Should().NotHaveDependencyOn(ApplicationAssembly.GetName().Name)
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}

[Fact]
public void Domain_Should_Not_HaveDependencyOn_Infrastructure()
{
    var result = Types.InAssembly(DomainAssembly)
        .Should().NotHaveDependencyOn(InfrastructureAssembly.GetName().Name)
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}
```
