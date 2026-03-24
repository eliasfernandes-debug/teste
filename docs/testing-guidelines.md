# Testes — Unitários e Integrados

## Pirâmide de Testes

```
         /\
        /  \      ← Testes de arquitetura (NetArchTest)
       /----\
      /      \    ← Testes integrados (WebApplicationFactory, Testcontainers)
     /--------\
    /          \  ← Testes unitários (xUnit, FluentAssertions, NSubstitute)
   /____________\
```

- **Testes unitários:** rápidos, isolados, sem I/O. Focados em Domain e Application.
- **Testes integrados:** verificam o comportamento de ponta a ponta com banco real (Testcontainers) ou in-memory.
- **Testes de arquitetura:** garantem que as regras de dependência arquitetural são respeitadas.

---

## Stack Recomendada

| Lib | Uso |
|---|---|
| `xUnit` | Framework de testes |
| `FluentAssertions` | Asserções expressivas |
| `NSubstitute` | Mocking |
| `Bogus` | Geração de dados falsos |
| `Testcontainers` | Banco real em container para testes integrados |
| `NetArchTest.Rules` | Testes de aderência arquitetural |
| `Microsoft.AspNetCore.Mvc.Testing` | `WebApplicationFactory` para testes integrados de API |

---

## Testes Unitários

### Convenções

- Nomeie os métodos com o padrão: `Método_Cenário_ResultadoEsperado`.
- Organize os testes com o padrão **AAA** (Arrange, Act, Assert).
- Um teste deve validar exatamente **uma** coisa.
- Não use lógica condicional (`if`, `switch`) dentro de testes.

### Testando o Domínio

```csharp
public sealed class OrderTests
{
    [Fact]
    public void Create_WithValidCustomerId_ShouldReturnDraftOrder()
    {
        // Arrange
        var customerId = CustomerId.New();

        // Act
        var order = Order.Create(customerId);

        // Assert
        order.Status.Should().Be(OrderStatus.Draft);
        order.Items.Should().BeEmpty();
        order.CustomerId.Should().Be(customerId);
    }

    [Fact]
    public void Confirm_WithNoItems_ShouldThrowDomainException()
    {
        // Arrange
        var order = Order.Create(CustomerId.New());

        // Act
        var act = () => order.Confirm();

        // Assert
        act.Should().Throw<DomainException>()
            .WithMessage("O pedido não possui itens.");
    }

    [Fact]
    public void AddItem_ToConfirmedOrder_ShouldThrowDomainException()
    {
        // Arrange
        var order = CreateConfirmedOrder();

        // Act
        var act = () => order.AddItem(ProductId.New(), quantity: 1, unitPrice: 10m);

        // Assert
        act.Should().Throw<DomainException>()
            .WithMessage("*confirmado*");
    }

    private static Order CreateConfirmedOrder()
    {
        var order = Order.Create(CustomerId.New());
        order.AddItem(ProductId.New(), 1, 10m);
        order.Confirm();
        return order;
    }
}
```

### Testando Use Cases (Application)

```csharp
public sealed class CreateOrderCommandHandlerTests
{
    private readonly IOrderRepository _orderRepository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly CreateOrderCommandHandler _sut;

    public CreateOrderCommandHandlerTests()
    {
        _orderRepository = Substitute.For<IOrderRepository>();
        _unitOfWork = Substitute.For<IUnitOfWork>();
        _sut = new CreateOrderCommandHandler(_orderRepository, _unitOfWork);
    }

    [Fact]
    public async Task Handle_WithValidCommand_ShouldPersistOrderAndReturnId()
    {
        // Arrange
        var customerId = Guid.NewGuid();
        var command = new CreateOrderCommand(customerId);
        _unitOfWork.SaveChangesAsync(Arg.Any<CancellationToken>()).Returns(1);

        // Act
        var orderId = await _sut.Handle(command, CancellationToken.None);

        // Assert
        orderId.Should().NotBeEmpty();
        await _orderRepository.Received(1).AddAsync(
            Arg.Is<Order>(o => o.CustomerId.Value == customerId),
            Arg.Any<CancellationToken>());
        await _unitOfWork.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Handle_WithEmptyCustomerId_ShouldThrowValidationException()
    {
        // Arrange
        var command = new CreateOrderCommand(Guid.Empty);

        // Act
        var act = async () => await _sut.Handle(command, CancellationToken.None);

        // Assert
        await act.Should().ThrowAsync<ValidationException>();
    }
}
```

---

## Testes Integrados

### Configuração com WebApplicationFactory e Testcontainers

```csharp
// Tests/IntegrationTests/ApiFactory.cs
public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .WithDatabase("orders_test")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Substituir a connection string pela do container
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_dbContainer.GetConnectionString()));
        });
    }

    public new async Task DisposeAsync()
    {
        await _dbContainer.StopAsync();
    }
}
```

### Teste Integrado de Endpoint

```csharp
public sealed class OrdersEndpointTests(ApiFactory factory)
    : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task POST_Orders_WithValidPayload_ShouldReturn201()
    {
        // Arrange
        var request = new { CustomerId = Guid.NewGuid() };

        // Act
        var response = await _client.PostAsJsonAsync("/v1/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var orderId = await response.Content.ReadFromJsonAsync<Guid>();
        orderId.Should().NotBeEmpty();
    }

    [Fact]
    public async Task GET_Orders_WithNonExistentId_ShouldReturn404()
    {
        // Arrange
        var nonExistentId = Guid.NewGuid();

        // Act
        var response = await _client.GetAsync($"/v1/orders/{nonExistentId}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

---

## Testes de Arquitetura

```csharp
public sealed class ArchitectureTests
{
    private static readonly Assembly DomainAssembly =
        typeof(MyApi.Domain.AssemblyReference).Assembly;
    private static readonly Assembly ApplicationAssembly =
        typeof(MyApi.Application.AssemblyReference).Assembly;
    private static readonly Assembly InfrastructureAssembly =
        typeof(MyApi.Infrastructure.AssemblyReference).Assembly;

    [Fact]
    public void Domain_ShouldNot_HaveDependencyOn_Application()
    {
        Types.InAssembly(DomainAssembly)
            .Should().NotHaveDependencyOn(ApplicationAssembly.GetName().Name)
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Domain_ShouldNot_HaveDependencyOn_Infrastructure()
    {
        Types.InAssembly(DomainAssembly)
            .Should().NotHaveDependencyOn(InfrastructureAssembly.GetName().Name)
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Application_ShouldNot_HaveDependencyOn_Infrastructure()
    {
        Types.InAssembly(ApplicationAssembly)
            .Should().NotHaveDependencyOn(InfrastructureAssembly.GetName().Name)
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Handlers_ShouldBeSealed()
    {
        Types.InAssembly(ApplicationAssembly)
            .That().ImplementInterface(typeof(IRequestHandler<,>))
            .Should().BeSealed()
            .GetResult().IsSuccessful.Should().BeTrue();
    }
}
```

---

## Cobertura de Código

Configure a coleta de cobertura no projeto:

```xml
<!-- tests/MyApi.UnitTests/MyApi.UnitTests.csproj -->
<PropertyGroup>
  <CollectCoverage>true</CollectCoverage>
  <CoverletOutputFormat>cobertura</CoverletOutputFormat>
  <CoverletOutput>./coverage/</CoverletOutput>
  <Threshold>80</Threshold>
</PropertyGroup>
```

Execute com relatório:

```bash
dotnet test --collect:"XPlat Code Coverage" \
  --results-directory ./coverage \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura
```

**Meta de cobertura mínima:** 80% para camadas Domain e Application.

---

## Boas Práticas Gerais

- Testes devem ser **determinísticos** — o mesmo teste deve sempre produzir o mesmo resultado.
- Testes **não devem depender uns dos outros** — cada teste deve ser independente.
- Utilize **builders** ou **object mothers** para criar objetos de teste complexos:

```csharp
// Padrão Object Mother / Builder
public static class OrderFaker
{
    public static Order CreateDraft() => Order.Create(CustomerId.New());

    public static Order CreateConfirmed()
    {
        var order = CreateDraft();
        order.AddItem(ProductId.New(), quantity: 2, unitPrice: 50m);
        order.Confirm();
        return order;
    }
}
```

- Agrupe testes relacionados em classes separadas por funcionalidade.
- Mantenha os testes próximos ao código que testam (`*.UnitTests` e `*.IntegrationTests` como projetos separados).
