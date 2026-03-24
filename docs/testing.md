# Estratégia de Testes

## Pirâmide de Testes

```
          /\
         /  \
        / E2E \          ← Poucos (lentos, caros)
       /--------\
      /          \
     / Integration \     ← Alguns (velocidade média)
    /--------------\
   /                \
  /   Unit Tests     \   ← Muitos (rápidos, baratos)
 /--------------------\
```

A plataforma segue a **Pirâmide de Testes** com:

- **Testes Unitários**: rápidos, isolados, sem dependências externas — a maior parte dos testes
- **Testes de Integração**: verificam a integração entre camadas ou com o banco de dados real
- **Testes E2E** (opcionais): validam o fluxo completo via API HTTP

---

## Stack de Testes

| Biblioteca | Finalidade |
|------------|-----------|
| **xUnit** | Framework de testes |
| **Moq** | Mocking de dependências |
| **FluentAssertions** | Asserções legíveis |
| **Microsoft.AspNetCore.Mvc.Testing** | Testes de integração da Web API |
| **Testcontainers** | Containers Docker para testes de integração (banco real) |
| **Bogus** | Geração de dados falsos/fixtures |

```bash
# Instalar pacotes de teste
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Moq
dotnet add package FluentAssertions
dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package Testcontainers.PostgreSql
dotnet add package Bogus
```

---

## Testes Unitários

### Estrutura e Nomenclatura

Siga o padrão **AAA** (Arrange, Act, Assert) e nomeie os testes com:

```
[Método/Comportamento]_[Cenário/Contexto]_[ResultadoEsperado]
```

```csharp
// tests/MyApp.UnitTests/Domain/OrderTests.cs
public sealed class OrderTests
{
    [Fact]
    public void AddItem_WhenOrderIsPending_ShouldAddItemAndUpdateTotal()
    {
        // Arrange
        var order = Order.Create(new CustomerId(Guid.NewGuid()));
        var product = new Product(Guid.NewGuid(), "Produto A", new Money(100m, "BRL"));

        // Act
        order.AddItem(product, 2);

        // Assert
        order.Items.Should().HaveCount(1);
        order.TotalAmount.Amount.Should().Be(200m);
    }

    [Fact]
    public void AddItem_WhenOrderIsConfirmed_ShouldThrowDomainException()
    {
        // Arrange
        var order = Order.Create(new CustomerId(Guid.NewGuid()));
        var product = new Product(Guid.NewGuid(), "Produto A", new Money(100m, "BRL"));
        order.AddItem(product, 1);
        order.Confirm();

        // Act
        var act = () => order.AddItem(product, 1);

        // Assert
        act.Should().Throw<DomainException>()
            .WithMessage("*pedidos pendentes*");
    }

    [Fact]
    public void Confirm_WhenOrderHasNoItems_ShouldThrowDomainException()
    {
        // Arrange
        var order = Order.Create(new CustomerId(Guid.NewGuid()));

        // Act
        var act = () => order.Confirm();

        // Assert
        act.Should().Throw<DomainException>()
            .WithMessage("*sem itens*");
    }

    [Fact]
    public void Confirm_WhenOrderHasItems_ShouldChangeStatusToConfirmed()
    {
        // Arrange
        var order = Order.Create(new CustomerId(Guid.NewGuid()));
        var product = new Product(Guid.NewGuid(), "Produto A", new Money(50m, "BRL"));
        order.AddItem(product, 1);

        // Act
        order.Confirm();

        // Assert
        order.Status.Should().Be(OrderStatus.Confirmed);
        order.DomainEvents.Should().ContainSingle()
            .Which.Should().BeOfType<OrderConfirmedEvent>();
    }
}
```

### Testando Application Handlers

```csharp
// tests/MyApp.UnitTests/Application/CreateOrderHandlerTests.cs
public sealed class CreateOrderHandlerTests
{
    private readonly Mock<IOrderRepository> _orderRepositoryMock;
    private readonly Mock<IProductRepository> _productRepositoryMock;
    private readonly Mock<IUnitOfWork> _unitOfWorkMock;
    private readonly Mock<IEventPublisher> _eventPublisherMock;
    private readonly CreateOrderHandler _handler;

    public CreateOrderHandlerTests()
    {
        _orderRepositoryMock = new Mock<IOrderRepository>();
        _productRepositoryMock = new Mock<IProductRepository>();
        _unitOfWorkMock = new Mock<IUnitOfWork>();
        _eventPublisherMock = new Mock<IEventPublisher>();

        _handler = new CreateOrderHandler(
            _orderRepositoryMock.Object,
            _productRepositoryMock.Object,
            _unitOfWorkMock.Object,
            _eventPublisherMock.Object);
    }

    [Fact]
    public async Task HandleAsync_WhenValidCommand_ShouldCreateOrderAndReturnId()
    {
        // Arrange
        var productId = Guid.NewGuid();
        var customerId = Guid.NewGuid();
        var product = new Product(productId, "Produto A", new Money(100m, "BRL"));

        _productRepositoryMock
            .Setup(r => r.GetByIdAsync(productId, It.IsAny<CancellationToken>()))
            .ReturnsAsync(product);

        _unitOfWorkMock
            .Setup(u => u.CommitAsync(It.IsAny<CancellationToken>()))
            .ReturnsAsync(1);

        var command = new CreateOrderCommand(customerId, new[]
        {
            new OrderItemRequest(productId, 2)
        });

        // Act
        var orderId = await _handler.HandleAsync(command);

        // Assert
        orderId.Should().NotBeEmpty();
        _orderRepositoryMock.Verify(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()), Times.Once);
        _unitOfWorkMock.Verify(u => u.CommitAsync(It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task HandleAsync_WhenProductNotFound_ShouldThrowNotFoundException()
    {
        // Arrange
        _productRepositoryMock
            .Setup(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync((Product?)null);

        var command = new CreateOrderCommand(Guid.NewGuid(), new[]
        {
            new OrderItemRequest(Guid.NewGuid(), 1)
        });

        // Act
        var act = async () => await _handler.HandleAsync(command);

        // Assert
        await act.Should().ThrowAsync<NotFoundException>()
            .WithMessage("*não encontrado*");

        _orderRepositoryMock.Verify(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()), Times.Never);
        _unitOfWorkMock.Verify(u => u.CommitAsync(It.IsAny<CancellationToken>()), Times.Never);
    }
}
```

---

## Testes de Integração

### Configuração com WebApplicationFactory

```csharp
// tests/MyApp.IntegrationTests/Fixtures/ApiFactory.cs
public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("testdb")
        .WithUsername("postgres")
        .WithPassword("postgres")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Remove o DbContext de produção
            var descriptor = services.SingleOrDefault(d =>
                d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null) services.Remove(descriptor);

            // Adiciona DbContext apontando para o container de teste
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_dbContainer.GetConnectionString()));
        });
    }

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();

        // Aplica migrations no banco de testes
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public new async Task DisposeAsync()
    {
        await _dbContainer.StopAsync();
    }
}
```

### Testes de Endpoint HTTP

```csharp
// tests/MyApp.IntegrationTests/Api/OrdersControllerTests.cs
public sealed class OrdersControllerTests : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client;

    public OrdersControllerTests(ApiFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task PostOrder_WithValidPayload_ShouldReturn201WithLocation()
    {
        // Arrange
        var request = new
        {
            customerId = Guid.NewGuid(),
            items = new[] { new { productId = Guid.NewGuid(), quantity = 2 } }
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/v1/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var body = await response.Content.ReadFromJsonAsync<CreateOrderResponse>();
        body!.Id.Should().NotBeEmpty();
    }

    [Fact]
    public async Task PostOrder_WithInvalidPayload_ShouldReturn400()
    {
        // Arrange
        var request = new { customerId = Guid.Empty, items = Array.Empty<object>() };

        // Act
        var response = await _client.PostAsJsonAsync("/api/v1/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task GetOrder_WhenOrderExists_ShouldReturn200WithOrderData()
    {
        // Arrange – criar pedido primeiro
        var createRequest = new
        {
            customerId = Guid.NewGuid(),
            items = new[] { new { productId = Guid.NewGuid(), quantity = 1 } }
        };
        var createResponse = await _client.PostAsJsonAsync("/api/v1/orders", createRequest);
        var created = await createResponse.Content.ReadFromJsonAsync<CreateOrderResponse>();

        // Act
        var response = await _client.GetAsync($"/api/v1/orders/{created!.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var order = await response.Content.ReadFromJsonAsync<OrderResponse>();
        order!.Id.Should().Be(created.Id);
    }

    [Fact]
    public async Task GetOrder_WhenOrderDoesNotExist_ShouldReturn404()
    {
        // Act
        var response = await _client.GetAsync($"/api/v1/orders/{Guid.NewGuid()}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

### Testes de Repositório

```csharp
// tests/MyApp.IntegrationTests/Infrastructure/OrderRepositoryTests.cs
public sealed class OrderRepositoryTests : IClassFixture<ApiFactory>
{
    private readonly IServiceScopeFactory _scopeFactory;

    public OrderRepositoryTests(ApiFactory factory)
    {
        _scopeFactory = factory.Services.GetRequiredService<IServiceScopeFactory>();
    }

    [Fact]
    public async Task AddAsync_ShouldPersistOrder()
    {
        using var scope = _scopeFactory.CreateScope();
        var repository = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        var unitOfWork = scope.ServiceProvider.GetRequiredService<IUnitOfWork>();

        // Arrange
        var order = Order.Create(new CustomerId(Guid.NewGuid()));

        // Act
        await repository.AddAsync(order);
        await unitOfWork.CommitAsync();

        // Assert
        var saved = await repository.GetByIdAsync(order.Id);
        saved.Should().NotBeNull();
        saved!.Id.Should().Be(order.Id);
    }
}
```

---

## Geração de Dados de Teste com Bogus

```csharp
// tests/MyApp.UnitTests/Fixtures/OrderFaker.cs
public sealed class OrderFaker : Faker<Order>
{
    public OrderFaker()
    {
        CustomInstantiator(f => Order.Create(new CustomerId(f.Random.Guid())));
    }
}

public sealed class ProductFaker : Faker<Product>
{
    public ProductFaker()
    {
        CustomInstantiator(f => new Product(
            f.Random.Guid(),
            f.Commerce.ProductName(),
            new Money(f.Finance.Amount(1, 999), "BRL")));
    }
}

// Uso nos testes
[Fact]
public void AddItem_WhenValid_ShouldUpdateTotal()
{
    var order = new OrderFaker().Generate();
    var product = new ProductFaker().Generate();

    order.AddItem(product, 3);

    order.Items.Should().HaveCount(1);
    order.TotalAmount.Amount.Should().Be(product.Price.Amount * 3);
}
```

---

## Execução dos Testes

```bash
# Executar todos os testes
dotnet test

# Executar apenas testes unitários
dotnet test --filter "FullyQualifiedName~UnitTests"

# Executar apenas testes de integração
dotnet test --filter "FullyQualifiedName~IntegrationTests"

# Executar com cobertura de código
dotnet test --collect:"XPlat Code Coverage"

# Relatório de cobertura (requer ReportGenerator)
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coverage-report" -reporttypes:Html
```

---

## Cobertura Mínima Esperada

| Camada | Cobertura Mínima |
|--------|-----------------|
| Domain | **90%** |
| Application | **85%** |
| Infrastructure | **70%** |
| API (Controllers) | **80%** |

Configure no `codecoverage.runsettings`:

```xml
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat Code Coverage">
        <Configuration>
          <Format>cobertura</Format>
          <Exclude>[*.Tests]*,[*.IntegrationTests]*</Exclude>
          <ExcludeByAttribute>GeneratedCodeAttribute</ExcludeByAttribute>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```
