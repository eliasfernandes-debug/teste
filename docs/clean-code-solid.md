# Clean Code e SOLID

## Clean Code

Clean Code é um conjunto de práticas que tornam o código legível, manutenível e testável. Abaixo estão as principais diretrizes aplicadas ao contexto .NET Core.

---

### Nomenclatura

```csharp
// ❌ Ruim
public async Task<object> Proc(Guid i, int q, decimal p)
{
    var o = new Order(i);
    o.AddItem(i, q, p);
    return o;
}

// ✅ Bom
public async Task<Order> CreateOrderWithItemAsync(
    Guid customerId,
    Guid productId,
    int quantity,
    decimal unitPrice,
    CancellationToken cancellationToken)
{
    var order = Order.Create(CustomerId.From(customerId));
    order.AddItem(ProductId.From(productId), quantity, unitPrice);
    await orderRepository.AddAsync(order, cancellationToken);
    return order;
}
```

**Regras:**
- Nomes de variáveis e métodos devem revelar sua intenção.
- Evite abreviações desnecessárias (`qty` → `quantity`, `dt` → `date`).
- Booleanos devem ter prefixo de verbo: `isActive`, `hasPermission`, `canConfirm`.
- Classes devem ter nomes de substantivos: `OrderRepository`, `CustomerService`.
- Métodos devem ter nomes de verbos: `CreateOrder`, `GetById`, `SendNotification`.

---

### Funções e Métodos

```csharp
// ❌ Ruim — faz muitas coisas
public async Task ProcessOrder(CreateOrderRequest request)
{
    // Valida
    if (request.CustomerId == Guid.Empty) throw new Exception("Invalid");

    // Cria pedido
    var order = new Order { CustomerId = request.CustomerId, Status = "Draft" };

    // Persiste
    await dbContext.Orders.AddAsync(order);
    await dbContext.SaveChangesAsync();

    // Envia email
    var email = await httpClient.GetAsync($"/customers/{request.CustomerId}/email");
    await emailService.Send(email, "Pedido criado");
}

// ✅ Bom — separação de responsabilidades
public async Task<Guid> Handle(CreateOrderCommand command, CancellationToken ct)
{
    var order = Order.Create(CustomerId.From(command.CustomerId));
    await orderRepository.AddAsync(order, ct);
    await unitOfWork.SaveChangesAsync(ct);
    await domainEventDispatcher.DispatchAsync(order.DomainEvents, ct);
    return order.Id;
}
```

**Regras:**
- Uma função deve fazer **apenas uma coisa**.
- Funções com mais de 20-30 linhas são candidatas a refatoração.
- Prefira funções sem efeitos colaterais ocultos.
- Limite o número de parâmetros — se precisar de muitos, use um objeto de comando/request.

---

### Comentários

```csharp
// ❌ Ruim — comentário que repete o código
// Incrementa i em 1
i++;

// ❌ Ruim — código comentado (use git para isso)
// var oldMethod = GetOrder(id);

// ✅ Bom — explica o "porquê", não o "quê"
// O GCP Pub/Sub garante entrega at-least-once, portanto
// o handler deve ser idempotente usando o messageId como chave.
if (await processedMessages.ContainsAsync(messageId))
    return;
```

**Regras:**
- Comentários devem explicar o **porquê**, não o **quê** (o código já diz o quê).
- Nunca deixe código comentado — use controle de versão.
- XMLDoc (`///`) é bem-vindo em interfaces públicas e métodos públicos de domínio.

---

### Evite Magic Numbers e Strings

```csharp
// ❌ Ruim
if (order.Items.Count > 50) throw new Exception("400");

// ✅ Bom
private const int MaxItemsPerOrder = 50;

if (order.Items.Count > MaxItemsPerOrder)
    throw new DomainException($"O pedido não pode ter mais de {MaxItemsPerOrder} itens.");
```

---

### Tratamento de Erros

```csharp
// ❌ Ruim — captura Exception genérica e silencia o erro
try
{
    await ProcessAsync();
}
catch (Exception)
{
    // ignore
}

// ✅ Bom — trata exceções específicas e registra o contexto
try
{
    await ProcessAsync(orderId, cancellationToken);
}
catch (NotFoundException ex)
{
    logger.LogWarning(ex, "Pedido {OrderId} não encontrado durante processamento.", orderId);
    throw;
}
catch (DomainException ex)
{
    logger.LogError(ex, "Violação de regra de negócio ao processar pedido {OrderId}.", orderId);
    throw;
}
```

---

## SOLID

### S — Single Responsibility Principle (SRP)

> Uma classe deve ter apenas **uma razão para mudar**.

```csharp
// ❌ Ruim — OrderService faz tudo
public class OrderService
{
    public void CreateOrder(...) { }
    public void SendConfirmationEmail(...) { }
    public void GenerateInvoicePdf(...) { }
    public void UpdateInventory(...) { }
}

// ✅ Bom — responsabilidades separadas
public class CreateOrderCommandHandler { ... }
public class OrderConfirmationEmailSender { ... }
public class InvoicePdfGenerator { ... }
public class InventoryUpdateHandler { ... }
```

---

### O — Open/Closed Principle (OCP)

> Classes devem ser **abertas para extensão** e **fechadas para modificação**.

```csharp
// ❌ Ruim — adicionar novo tipo de desconto exige modificar a classe
public class DiscountCalculator
{
    public decimal Calculate(Order order, string discountType)
    {
        if (discountType == "seasonal") return order.Total * 0.1m;
        if (discountType == "loyalty") return order.Total * 0.15m;
        return 0;
    }
}

// ✅ Bom — cada desconto é uma implementação separada
public interface IDiscountStrategy
{
    decimal Calculate(Order order);
}

public sealed class SeasonalDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.1m;
}

public sealed class LoyaltyDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.15m;
}

// Adicionando novo desconto sem modificar código existente:
public sealed class FlashSaleDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.25m;
}
```

---

### L — Liskov Substitution Principle (LSP)

> Objetos de uma subclasse devem poder substituir objetos da superclasse **sem alterar o comportamento**.

```csharp
// ❌ Ruim — viola LSP ao lançar exceção em método herdado
public class ReadOnlyRepository : IOrderRepository
{
    public Task AddAsync(Order order, CancellationToken ct)
        => throw new NotSupportedException(); // quebra o contrato
}

// ✅ Bom — interfaces com granularidade adequada
public interface IOrderReadRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
}

public interface IOrderWriteRepository
{
    Task AddAsync(Order order, CancellationToken ct);
    Task UpdateAsync(Order order, CancellationToken ct);
}
```

---

### I — Interface Segregation Principle (ISP)

> Clientes não devem ser forçados a depender de interfaces que **não utilizam**.

```csharp
// ❌ Ruim — interface gorda
public interface IOrderService
{
    Task CreateAsync(CreateOrderCommand command);
    Task ConfirmAsync(Guid orderId);
    Task CancelAsync(Guid orderId);
    Task<byte[]> GenerateInvoicePdfAsync(Guid orderId);
    Task SendConfirmationEmailAsync(Guid orderId);
}

// ✅ Bom — interfaces focadas
public interface IOrderWriter
{
    Task CreateAsync(CreateOrderCommand command);
    Task ConfirmAsync(Guid orderId);
    Task CancelAsync(Guid orderId);
}

public interface IInvoiceGenerator
{
    Task<byte[]> GeneratePdfAsync(Guid orderId);
}

public interface IOrderNotifier
{
    Task SendConfirmationEmailAsync(Guid orderId);
}
```

---

### D — Dependency Inversion Principle (DIP)

> Módulos de alto nível não devem depender de módulos de baixo nível. Ambos devem depender de **abstrações**.

```csharp
// ❌ Ruim — Application depende diretamente da Infrastructure
public class CreateOrderCommandHandler
{
    private readonly SqlOrderRepository _repository; // dependência concreta

    public CreateOrderCommandHandler()
    {
        _repository = new SqlOrderRepository("connection string aqui");
    }
}

// ✅ Bom — Application depende da abstração (interface)
public class CreateOrderCommandHandler(IOrderRepository orderRepository, IUnitOfWork unitOfWork)
    : IRequestHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(CustomerId.From(request.CustomerId));
        await orderRepository.AddAsync(order, ct);
        await unitOfWork.SaveChangesAsync(ct);
        return order.Id;
    }
}
// A implementação concreta (SqlOrderRepository) é injetada pela Infrastructure via DI container
```

---

## Checklist de Revisão de Código

Antes de aprovar um PR, verifique:

- [ ] Os nomes revelam claramente a intenção?
- [ ] Cada método/classe tem uma única responsabilidade?
- [ ] O código é testável sem precisar de infraestrutura real?
- [ ] Existem testes para os novos comportamentos adicionados?
- [ ] As interfaces do domínio foram violadas?
- [ ] Existem magic numbers ou strings sem constante?
- [ ] Exceções são tratadas de forma adequada (sem silenciar erros)?
- [ ] Há logging suficiente para diagnosticar problemas em produção?
- [ ] O código novo introduz dependências não necessárias entre camadas?
- [ ] Secrets ou dados sensíveis foram expostos acidentalmente?
