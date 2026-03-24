# Boas Práticas – SOLID, Clean Code e Princípios Gerais

## SOLID

Os cinco princípios SOLID são fundamentais para código orientado a objetos de qualidade. Todos os projetos da plataforma devem segui-los.

---

### S – Single Responsibility Principle (Princípio da Responsabilidade Única)

> Uma classe deve ter apenas **um motivo para mudar**.

```csharp
// ❌ Errado – classe com múltiplas responsabilidades
public class OrderService
{
    public void CreateOrder(Order order) { /* regra de negócio */ }
    public void SendEmail(Order order) { /* envio de e-mail */ }
    public void SaveToDatabase(Order order) { /* persistência */ }
    public string GenerateOrderPdf(Order order) { /* geração de PDF */ }
}

// ✅ Correto – cada classe tem uma responsabilidade
public class CreateOrderHandler { /* apenas orquestra a criação do pedido */ }
public class EmailNotificationService { /* apenas envia e-mails */ }
public class OrderRepository { /* apenas persiste pedidos */ }
public class OrderPdfGenerator { /* apenas gera PDFs */ }
```

---

### O – Open/Closed Principle (Aberto/Fechado)

> Classes devem estar **abertas para extensão** e **fechadas para modificação**.

```csharp
// ❌ Errado – modificar a classe para cada novo tipo de desconto
public class DiscountCalculator
{
    public decimal Calculate(Order order, string discountType)
    {
        if (discountType == "student") return order.Total * 0.10m;
        if (discountType == "senior") return order.Total * 0.15m;
        if (discountType == "vip") return order.Total * 0.20m; // precisa modificar
        return 0;
    }
}

// ✅ Correto – extensível sem modificar código existente
public interface IDiscountStrategy
{
    decimal Calculate(Order order);
}

public sealed class StudentDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.10m;
}

public sealed class SeniorDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.15m;
}

public sealed class VipDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.20m;
}

// Adicionar novo desconto = criar nova classe, sem modificar existentes
public sealed class NewYearDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.25m;
}
```

---

### L – Liskov Substitution Principle (Substituição de Liskov)

> Objetos de uma subclasse devem poder substituir objetos da superclasse **sem alterar o comportamento correto** do programa.

```csharp
// ❌ Errado – subclasse viola o contrato da interface
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(Guid id);
    Task AddAsync(T entity);
    Task UpdateAsync(T entity); // ReadOnlyRepository não pode implementar isso
}

public class ReadOnlyOrderRepository : IRepository<Order>
{
    public Task<Order?> GetByIdAsync(Guid id) => /* implementa */;
    public Task AddAsync(Order order) => throw new NotSupportedException(); // Viola LSP!
    public Task UpdateAsync(Order order) => throw new NotSupportedException(); // Viola LSP!
}

// ✅ Correto – interfaces separadas por capacidade
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(Guid id);
}

public interface IWriteRepository<T>
{
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
}

public interface IRepository<T> : IReadRepository<T>, IWriteRepository<T> { }

public class ReadOnlyOrderRepository : IReadRepository<Order>
{
    public Task<Order?> GetByIdAsync(Guid id) => /* implementa corretamente */;
}
```

---

### I – Interface Segregation Principle (Segregação de Interfaces)

> Os clientes **não devem ser forçados** a depender de interfaces que não utilizam.

```csharp
// ❌ Errado – interface "gorda" força implementações desnecessárias
public interface IOrderService
{
    Task<Order> GetByIdAsync(Guid id);
    Task CreateAsync(Order order);
    Task DeleteAsync(Guid id);
    Task<byte[]> GeneratePdfAsync(Guid id);  // Nem todos precisam disso
    Task SendEmailConfirmationAsync(Guid id); // Nem todos precisam disso
}

// ✅ Correto – interfaces coesas e pequenas
public interface IOrderReader
{
    Task<Order?> GetByIdAsync(Guid id);
}

public interface IOrderWriter
{
    Task CreateAsync(Order order);
    Task DeleteAsync(Guid id);
}

public interface IOrderDocumentGenerator
{
    Task<byte[]> GeneratePdfAsync(Guid id);
}

public interface IOrderNotifier
{
    Task SendEmailConfirmationAsync(Guid id);
}
```

---

### D – Dependency Inversion Principle (Inversão de Dependência)

> Módulos de alto nível não devem depender de módulos de baixo nível. **Ambos devem depender de abstrações**.

```csharp
// ❌ Errado – handler depende diretamente da implementação
public class CreateOrderHandler
{
    private readonly SqlOrderRepository _repository; // Dependência de implementação concreta!

    public CreateOrderHandler()
    {
        _repository = new SqlOrderRepository(); // Acoplamento forte!
    }
}

// ✅ Correto – handler depende de abstração, implementação é injetada
public class CreateOrderHandler
{
    private readonly IOrderRepository _repository; // Depende da abstração

    public CreateOrderHandler(IOrderRepository repository) // Injeção de dependência
    {
        _repository = repository;
    }
}

// No container de DI:
services.AddScoped<IOrderRepository, SqlOrderRepository>(); // Implementação configurada externamente
```

---

## Clean Code

### Nomenclatura Clara e Expressiva

```csharp
// ❌ Evitar
var d = DateTime.UtcNow;
var lst = new List<Order>();
void Proc(Order o) { }
bool chk(Order o) => o.s == 1;

// ✅ Preferir
var createdAt = DateTime.UtcNow;
var pendingOrders = new List<Order>();
void ProcessOrder(Order order) { }
bool IsOrderPending(Order order) => order.Status == OrderStatus.Pending;
```

### Métodos Pequenos e Focados

```csharp
// ❌ Método fazendo tudo
public async Task<Guid> ProcessOrder(CreateOrderRequest request)
{
    // validação
    if (request.CustomerId == Guid.Empty) throw new Exception("Invalid");
    if (!request.Items.Any()) throw new Exception("No items");

    // busca cliente
    var customer = await _db.Customers.FindAsync(request.CustomerId);
    if (customer == null) throw new Exception("Not found");

    // cria pedido
    var order = new Order { Id = Guid.NewGuid(), CustomerId = request.CustomerId };
    foreach (var item in request.Items)
    {
        var product = await _db.Products.FindAsync(item.ProductId);
        order.Items.Add(new OrderItem { ProductId = item.ProductId, Quantity = item.Quantity, Price = product.Price });
        order.Total += product.Price * item.Quantity;
    }

    // persiste
    _db.Orders.Add(order);
    await _db.SaveChangesAsync();

    // notifica
    await _emailService.SendAsync(customer.Email, $"Pedido {order.Id} criado");

    return order.Id;
}

// ✅ Cada método com responsabilidade única e clara
public async Task<Guid> HandleAsync(CreateOrderCommand command, CancellationToken ct)
{
    var order = await BuildOrderAsync(command, ct);
    await PersistOrderAsync(order, ct);
    await NotifyCustomerAsync(order, ct);
    return order.Id;
}

private async Task<Order> BuildOrderAsync(CreateOrderCommand command, CancellationToken ct)
{
    var order = Order.Create(new CustomerId(command.CustomerId));
    foreach (var itemRequest in command.Items)
    {
        var product = await GetProductOrThrowAsync(itemRequest.ProductId, ct);
        order.AddItem(product, itemRequest.Quantity);
    }
    return order;
}
```

### Evite Números Mágicos e Strings Mágicas

```csharp
// ❌ Números e strings sem contexto
if (order.Items.Count > 50) throw new Exception("limit");
var status = "CONFIRMED";
Thread.Sleep(3000);

// ✅ Constantes nomeadas
private const int MaxItemsPerOrder = 50;
private const int RetryDelayMilliseconds = 3_000;

if (order.Items.Count > MaxItemsPerOrder)
    throw new DomainException($"Um pedido não pode ter mais de {MaxItemsPerOrder} itens.");

order.Status = OrderStatus.Confirmed;
await Task.Delay(RetryDelayMilliseconds, cancellationToken);
```

### Tratamento de Erros Expressivo

```csharp
// ❌ Exceções genéricas
throw new Exception("error");
throw new Exception("not found");

// ✅ Exceções de domínio específicas
throw new DomainException("O estoque do produto é insuficiente para a quantidade solicitada.");
throw new NotFoundException($"Pedido com ID {id} não foi encontrado.");
throw new ConflictException($"Já existe um pedido em andamento para o cliente {customerId}.");
```

### Não Use Comentários para Explicar Código Ruim

```csharp
// ❌ Código confuso com comentário explicativo
// Incrementa o contador de tentativas e verifica se excedeu o máximo
if (++rtCnt > maxRtCnt) break;

// ✅ Código autoexplicativo, sem necessidade de comentário
retryCount++;
var maxRetriesExceeded = retryCount > maxRetryAttempts;
if (maxRetriesExceeded) break;
```

### Use `CancellationToken` em Operações Assíncronas

```csharp
// ❌ Sem suporte a cancelamento
public async Task<Order?> GetByIdAsync(Guid id)
{
    return await _context.Orders.FindAsync(id);
}

// ✅ Com suporte a cancelamento
public async Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
{
    return await _context.Orders
        .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
}
```

---

## Princípios Gerais da Plataforma

### Imutabilidade

Prefira objetos imutáveis, especialmente nos Value Objects do domínio:

```csharp
// ✅ Value Object imutável com record
public sealed record Money(decimal Amount, string Currency);
public sealed record Email(string Value);
public sealed record CustomerId(Guid Value);
```

### Fail Fast

Valide entradas o quanto antes e falhe rapidamente:

```csharp
public static Order Create(CustomerId customerId)
{
    ArgumentNullException.ThrowIfNull(customerId);                         // .NET 6+
    if (customerId.Value == Guid.Empty)
        throw new DomainException("O ID do cliente não pode ser vazio.");

    return new Order { CustomerId = customerId, /* ... */ };
}
```

### Logging Estruturado

```csharp
// ❌ Não use interpolação de string no logging
_logger.LogInformation($"Pedido {orderId} criado para o cliente {customerId}");

// ✅ Use templates estruturados (permite indexação e busca em ferramentas como Kibana/Cloud Logging)
_logger.LogInformation("Pedido {OrderId} criado para o cliente {CustomerId}", orderId, customerId);
```

### Nunca Retorne `null` de Coleções

```csharp
// ❌ Retornar null obriga o chamador a verificar null
public IEnumerable<Order>? GetOrders() => null;

// ✅ Retornar coleção vazia
public IEnumerable<Order> GetOrders() => Enumerable.Empty<Order>();
```

### Use `sealed` em Classes Não Projetadas para Herança

```csharp
// ✅ Classes concretas devem ser sealed por padrão
public sealed class CreateOrderHandler { }
public sealed class OrderRepository : IOrderRepository { }
public sealed class CorrelationIdMiddleware { }
```

### Evite Herança em Favor de Composição

```csharp
// ❌ Herança cria acoplamento
public class AuditableRepository : OrderRepository
{
    // estende repositório com auditoria — acoplamento forte
}

// ✅ Composição é mais flexível
public class AuditedOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _inner;
    private readonly IAuditLogger _auditLogger;

    public async Task AddAsync(Order order, CancellationToken ct)
    {
        await _inner.AddAsync(order, ct);
        await _auditLogger.LogAsync("Order.Add", order.Id);
    }
}
```

---

## Checklist de Code Review

Antes de aprovar um Pull Request, verifique:

- [ ] Os nomes de classes, métodos e variáveis são claros e expressivos?
- [ ] Os métodos têm responsabilidade única e são pequenos?
- [ ] Não há lógica de negócio nos Controllers ou Infrastructure?
- [ ] As interfaces de domínio estão no projeto de domínio?
- [ ] Os testes cobrem cenários de sucesso e falha?
- [ ] `CancellationToken` está sendo propagado nas operações assíncronas?
- [ ] O logging usa templates estruturados?
- [ ] Não há strings mágicas ou números mágicos?
- [ ] As exceções são específicas e expressivas?
- [ ] O código segue os princípios SOLID?
- [ ] Não há dependências circulares entre projetos?
- [ ] A camada de domínio não possui dependências de infraestrutura?
