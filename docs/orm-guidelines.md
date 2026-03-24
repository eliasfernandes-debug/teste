# ORM — Dapper e Entity Framework Core

As APIs Core podem utilizar **Dapper** ou **Entity Framework Core** dependendo do cenário. Este guia apresenta boas práticas para cada um.

---

## Quando usar cada ORM?

| Cenário | ORM recomendado |
|---|---|
| Consultas complexas, relatórios, alto volume de leitura | **Dapper** |
| Operações CRUD simples, rico mapeamento de domínio | **Entity Framework Core** |
| APIs que misturam consultas simples e complexas | **Hybrid** (EF para escrita + Dapper para leitura) |

O padrão **Hybrid** é bastante comum em Clean Architecture com CQRS: Entity Framework para os **Commands** (escrita) e Dapper para as **Queries** (leitura).

---

## Entity Framework Core

### Configuração

```csharp
// Infrastructure/DependencyInjection.cs
services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(
        configuration.GetConnectionString("DefaultConnection"),
        npgsql => npgsql.MigrationsAssembly(typeof(AppDbContext).Assembly.FullName)));
```

### DbContext

```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### EntityTypeConfiguration

```csharp
internal sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");

        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id).ValueGeneratedNever();

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(50)
            .IsRequired();

        // Value Object como Owned Entity
        builder.OwnsOne(o => o.CustomerId, ownedBuilder =>
        {
            ownedBuilder.Property(c => c.Value)
                .HasColumnName("customer_id")
                .IsRequired();
        });

        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey("order_id")
            .OnDelete(DeleteBehavior.Cascade);

        // Concorrência otimista
        builder.Property<uint>("RowVersion").IsRowVersion();
    }
}
```

### Migrations

```bash
# Adicionar migration
dotnet ef migrations add InitialCreate \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api

# Aplicar migrations
dotnet ef database update \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api
```

**Boas práticas com EF Core:**
- Utilize **migrations** em código, nunca atualize o banco manualmente.
- Configure **soft delete** com `IsDeleted` + query filter global quando necessário.
- Evite lazy loading em APIs — sempre use `Include` explícito.
- Não exponha o `DbContext` fora da camada Infrastructure.
- Utilize `AsNoTracking()` em consultas somente-leitura.

```csharp
// Consulta somente-leitura com AsNoTracking
public async Task<IReadOnlyList<OrderSummary>> GetSummariesAsync(CancellationToken ct)
    => await dbContext.Orders
        .AsNoTracking()
        .Where(o => o.Status == OrderStatus.Confirmed)
        .Select(o => new OrderSummary(o.Id, o.CustomerId.Value, o.Status))
        .ToListAsync(ct);
```

---

## Dapper

### Configuração

```csharp
// Infrastructure/DependencyInjection.cs
services.AddScoped<IDbConnection>(_ =>
    new NpgsqlConnection(configuration.GetConnectionString("DefaultConnection")));
```

### Repositório de Leitura com Dapper

```csharp
public sealed class OrderReadRepository(IDbConnection connection) : IOrderReadRepository
{
    public async Task<OrderDetailDto?> GetDetailAsync(Guid orderId, CancellationToken ct)
    {
        const string sql = """
            SELECT
                o.id          AS Id,
                o.customer_id AS CustomerId,
                o.status      AS Status,
                oi.id         AS ItemId,
                oi.product_id AS ProductId,
                oi.quantity   AS Quantity,
                oi.unit_price AS UnitPrice
            FROM orders o
            LEFT JOIN order_items oi ON oi.order_id = o.id
            WHERE o.id = @OrderId
            """;

        var orderDict = new Dictionary<Guid, OrderDetailDto>();

        await connection.QueryAsync<OrderDetailDto, OrderItemDto, OrderDetailDto>(
            sql,
            (order, item) =>
            {
                if (!orderDict.TryGetValue(order.Id, out var entry))
                {
                    entry = order;
                    orderDict[order.Id] = entry;
                }
                if (item is not null)
                    entry.Items.Add(item);
                return entry;
            },
            new { OrderId = orderId },
            splitOn: "ItemId");

        return orderDict.GetValueOrDefault(orderId);
    }
}
```

**Boas práticas com Dapper:**
- Sempre use **parâmetros** nas queries — nunca concatene strings (prevenção de SQL Injection).
- Utilize `using var connection = ...` ou injete `IDbConnection` com escopo `Scoped`.
- Prefira `QueryAsync` / `ExecuteAsync` para operações assíncronas.
- Para queries complexas, prefira `sql` como string verbatim (`"""..."""`) para legibilidade.
- Documente queries complexas com comentários explicando o propósito.

---

## Padrão Hybrid (EF + Dapper)

```
┌─────────────────────────────────┐
│         CQRS + Hybrid ORM       │
│                                 │
│  Command → EF Core → Banco      │  ← Escrita com rastreamento de entidade
│  Query   → Dapper  → Banco      │  ← Leitura direta, sem overhead do EF
└─────────────────────────────────┘
```

Registre ambos no DI:

```csharp
services.AddDbContext<AppDbContext>(...);                  // EF — Commands
services.AddScoped<IDbConnection>(_ => new NpgsqlConnection(...)); // Dapper — Queries
```

---

## Unit of Work

Gerencie transações com o padrão **Unit of Work**:

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

public sealed class UnitOfWork(AppDbContext dbContext) : IUnitOfWork
{
    public Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        => dbContext.SaveChangesAsync(cancellationToken);
}
```

---

## Segurança

- **Nunca** armazene connection strings em `appsettings.json` em ambientes produtivos. Use o **GCP Secret Manager**.
- Use o **Workload Identity** do GKE para autenticação no Cloud SQL sem senhas.
- Utilize Cloud SQL Auth Proxy ou Cloud SQL Connector para conexões seguras.
