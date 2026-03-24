# ORM – Dapper e Entity Framework Core

## Quando Usar Cada ORM

| Critério | Dapper | Entity Framework Core |
|----------|---------|-----------------------|
| Queries complexas e otimizadas | ✅ Recomendado | ⚠️ Pode ser verboso |
| CRUD simples | ⚠️ Mais verboso | ✅ Recomendado |
| Rastreamento de entidades | ❌ Não suporta | ✅ Suporta |
| Performance em leitura massiva | ✅ Mais rápido | ⚠️ Overhead de tracking |
| Migrations automáticas | ❌ Manual | ✅ Suporta |
| Mapeamento complexo de domínio | ⚠️ Manual | ✅ Configurável via Fluent API |
| Relatórios / Read Models | ✅ Ideal | ⚠️ Pode gerar N+1 |

**Recomendação da plataforma:**
- **Entity Framework Core** para a camada de **escrita** (Commands) e mapeamento do modelo de domínio.
- **Dapper** para a camada de **leitura** (Queries), relatórios e projeções otimizadas.

Essa combinação é conhecida como **CQRS com ORMs distintos**.

---

## Entity Framework Core

### Configuração

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL   # PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

### DbContext

```csharp
// Infrastructure/Persistence/Context/AppDbContext.cs
public sealed class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Aplica todas as configurações do assembly automaticamente
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### Configuração Fluent API (Entity Type Configuration)

```csharp
// Infrastructure/Persistence/Configurations/OrderConfiguration.cs
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");

        builder.HasKey(o => o.Id);

        builder.Property(o => o.Id)
            .HasColumnName("id")
            .ValueGeneratedNever();

        // Value Object: Money
        builder.OwnsOne(o => o.TotalAmount, money =>
        {
            money.Property(m => m.Amount)
                .HasColumnName("total_amount")
                .HasColumnType("numeric(18,2)")
                .IsRequired();

            money.Property(m => m.Currency)
                .HasColumnName("currency")
                .HasMaxLength(3)
                .IsRequired();
        });

        // Value Object: CustomerId
        builder.Property(o => o.CustomerId)
            .HasConversion(
                customerId => customerId.Value,
                value => new CustomerId(value))
            .HasColumnName("customer_id")
            .IsRequired();

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasColumnName("status")
            .IsRequired();

        builder.Property(o => o.CreatedAt)
            .HasColumnName("created_at")
            .IsRequired();

        // Relacionamento com OrderItems
        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey("order_id")
            .OnDelete(DeleteBehavior.Cascade);

        builder.Navigation(o => o.Items).UsePropertyAccessMode(PropertyAccessMode.Field);
    }
}
```

### Migrations

```bash
# Criar migration
dotnet ef migrations add AddOrderTable --project src/MyApp.Infrastructure --startup-project src/MyApp.Api

# Aplicar migration
dotnet ef database update --project src/MyApp.Infrastructure --startup-project src/MyApp.Api

# Reverter migration
dotnet ef database update PreviousMigrationName
```

### Repository com EF Core

```csharp
// Infrastructure/Persistence/Repositories/OrderRepository.cs
public sealed class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .AsNoTracking()               // Use tracking apenas quando for atualizar
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
    }

    public async Task<Order?> GetByIdForUpdateAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return await _context.Orders     // Com tracking para atualização
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
}
```

### Unit of Work

```csharp
// Domain/Interfaces/IUnitOfWork.cs
public interface IUnitOfWork
{
    Task<int> CommitAsync(CancellationToken cancellationToken = default);
}

// Infrastructure/Persistence/UnitOfWork.cs
public sealed class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public UnitOfWork(AppDbContext context) => _context = context;

    public async Task<int> CommitAsync(CancellationToken cancellationToken = default)
    {
        return await _context.SaveChangesAsync(cancellationToken);
    }
}
```

---

## Dapper

### Configuração

```bash
dotnet add package Dapper
dotnet add package Npgsql  # PostgreSQL
```

### Read Repository com Dapper

```csharp
// Infrastructure/Persistence/ReadRepositories/OrderReadRepository.cs
public sealed class OrderReadRepository : IOrderReadRepository
{
    private readonly IDbConnection _connection;

    public OrderReadRepository(IDbConnection connection) => _connection = connection;

    public async Task<OrderSummaryDto?> GetSummaryByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        const string sql = """
            SELECT
                o.id              AS Id,
                o.status          AS Status,
                o.total_amount    AS TotalAmount,
                o.currency        AS Currency,
                o.created_at      AS CreatedAt,
                c.name            AS CustomerName,
                c.email           AS CustomerEmail
            FROM orders o
            INNER JOIN customers c ON c.id = o.customer_id
            WHERE o.id = @Id
            """;

        var command = new CommandDefinition(sql, new { Id = id },
            cancellationToken: cancellationToken);

        return await _connection.QueryFirstOrDefaultAsync<OrderSummaryDto>(command);
    }

    public async Task<IEnumerable<OrderListItemDto>> GetPagedByCustomerAsync(
        Guid customerId,
        int page,
        int pageSize,
        CancellationToken cancellationToken = default)
    {
        const string sql = """
            SELECT
                o.id          AS Id,
                o.status      AS Status,
                o.total_amount AS TotalAmount,
                o.created_at  AS CreatedAt
            FROM orders o
            WHERE o.customer_id = @CustomerId
            ORDER BY o.created_at DESC
            LIMIT @PageSize OFFSET @Offset
            """;

        var command = new CommandDefinition(sql,
            new { CustomerId = customerId, PageSize = pageSize, Offset = (page - 1) * pageSize },
            cancellationToken: cancellationToken);

        return await _connection.QueryAsync<OrderListItemDto>(command);
    }

    public async Task<OrderWithItemsDto?> GetWithItemsAsync(Guid id, CancellationToken cancellationToken = default)
    {
        const string sql = """
            SELECT
                o.id           AS Id,
                o.status       AS Status,
                o.total_amount AS TotalAmount,
                i.product_name AS ProductName,
                i.quantity     AS Quantity,
                i.unit_price   AS UnitPrice
            FROM orders o
            INNER JOIN order_items i ON i.order_id = o.id
            WHERE o.id = @Id
            """;

        var command = new CommandDefinition(sql, new { Id = id },
            cancellationToken: cancellationToken);

        OrderWithItemsDto? result = null;

        await _connection.QueryAsync<OrderWithItemsDto, OrderItemDto, OrderWithItemsDto>(
            command,
            (order, item) =>
            {
                result ??= order;
                result.Items.Add(item);
                return result;
            },
            splitOn: "ProductName");

        return result;
    }
}
```

### Registro de Conexão

```csharp
// Infrastructure/DependencyInjection/InfrastructureExtensions.cs
services.AddTransient<IDbConnection>(_ =>
    new NpgsqlConnection(configuration.GetConnectionString("ReadConnection")));

services.AddScoped<IOrderReadRepository, OrderReadRepository>();
```

---

## Boas Práticas Comuns

### Nunca exponha o DbContext fora da camada de infraestrutura

```csharp
// ❌ Errado – Controller acessando DbContext diretamente
public class OrdersController : ControllerBase
{
    private readonly AppDbContext _context; // Não faça isso!
}

// ✅ Correto – Controller usa caso de uso da camada de aplicação
public class OrdersController : ControllerBase
{
    private readonly CreateOrderHandler _createOrderHandler;
}
```

### Nunca use `SaveChanges` diretamente no repositório

```csharp
// ❌ Errado – repositório commita a transação
public async Task AddAsync(Order order)
{
    await _context.Orders.AddAsync(order);
    await _context.SaveChangesAsync(); // Não faça isso!
}

// ✅ Correto – UnitOfWork controla a transação
public async Task AddAsync(Order order)
{
    await _context.Orders.AddAsync(order);
    // O handler chama _unitOfWork.CommitAsync() no final
}
```

### Use `AsNoTracking` para leituras

```csharp
// ✅ Leitura sem rastreamento (mais performático)
var orders = await _context.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(cancellationToken);
```

### Evite N+1 com `Include`

```csharp
// ❌ Errado – N+1 queries
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
{
    var items = await _context.OrderItems // Query extra por cada order!
        .Where(i => i.OrderId == order.Id).ToListAsync();
}

// ✅ Correto – JOIN único com Include
var orders = await _context.Orders
    .Include(o => o.Items)
    .ToListAsync(cancellationToken);
```

### Use parâmetros nomeados no Dapper (prevenção de SQL Injection)

```csharp
// ❌ Errado – concatenação de string (SQL Injection!)
var sql = $"SELECT * FROM orders WHERE id = '{id}'";

// ✅ Correto – parâmetros nomeados
var sql = "SELECT * FROM orders WHERE id = @Id";
var result = await _connection.QueryFirstOrDefaultAsync(sql, new { Id = id });
```
