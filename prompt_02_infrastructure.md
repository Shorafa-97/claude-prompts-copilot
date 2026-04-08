# Prompt 02 — Infrastructure Layer

## Role
You are continuing the `ContactsX` .NET 9 Clean Architecture backend. The Core layer is complete. Now implement the **Infrastructure layer** (`ContactsX.Infrastructure`).

---

## Few-Shot Examples (Chain-of-Thought)

### Example A — EF Core configuration for JSON columns

**Input:** "How do I persist `List<string> Emails` in PostgreSQL using EF Core?"

**Reasoning (CoT):**
1. PostgreSQL supports `jsonb` natively. Npgsql maps `List<string>` to `jsonb` automatically via `HasColumnType("jsonb")`.
2. I define the mapping in an `IEntityTypeConfiguration<Contact>` class, not in `OnModelCreating`, to keep the DbContext clean.
3. For nested objects like `ContactAddress`, I use `OwnsMany` or a custom value converter serializing to JSON.

**Output:**
```csharp
// Infrastructure/Persistence/Configurations/ContactConfiguration.cs
public class ContactConfiguration : IEntityTypeConfiguration<Contact>
{
    public void Configure(EntityTypeBuilder<Contact> builder)
    {
        builder.HasKey(c => c.Id);
        builder.Property(c => c.FirstName).IsRequired().HasMaxLength(200);
        builder.Property(c => c.LastName).IsRequired().HasMaxLength(200);
        builder.Property(c => c.Emails)
               .HasColumnType("jsonb")
               .HasConversion(
                   v => JsonSerializer.Serialize(v, JsonOptions),
                   v => JsonSerializer.Deserialize<List<string>>(v, JsonOptions)!);
        builder.Property(c => c.ContactType)
               .HasConversion<string>();
        builder.Property(c => c.Gender)
               .HasConversion<string>();
    }
    private static readonly JsonSerializerOptions JsonOptions = new(JsonSerializerDefaults.Web);
}
```

---

### Example B — Generic Repository pattern

**Input:** "Should I use a generic repository or specific repositories?"

**Reasoning (CoT):**
1. A generic `IRepository<T>` reduces boilerplate but leaks query concerns into Application.
2. For `ContactsX`, I use **specific repositories** (e.g. `IContactRepository`) with query methods relevant to the domain — this keeps each repository interface slim and meaningful.
3. The concrete implementation inherits from a thin `BaseRepository<T>` providing `GetByIdAsync`, `AddAsync`, etc., to avoid duplication.

**Output:**
```csharp
// Infrastructure/Persistence/Repositories/BaseRepository.cs
public abstract class BaseRepository<T>(AppDbContext db) where T : class
{
    protected readonly AppDbContext Db = db;
    public async Task<T?> GetByIdAsync(Guid id, CancellationToken ct)
        => await Db.Set<T>().FindAsync([id], ct);
    public async Task AddAsync(T entity, CancellationToken ct)
        => await Db.Set<T>().AddAsync(entity, ct);
    public Task DeleteAsync(T entity)
    {
        Db.Set<T>().Remove(entity);
        return Task.CompletedTask;
    }
}
```

---

## Your Task

Generate the complete **Infrastructure layer** for `ContactsX`.

### Step 1 — AppDbContext
Create `Infrastructure/Persistence/AppDbContext.cs`:
- Register all `DbSet<T>` for: `Contact`, `Organization` (entity), `ContactEntityRelation`, `AuditLog`, `DuplicateCandidate`.
- Apply all `IEntityTypeConfiguration<T>` via `modelBuilder.ApplyConfigurationsFromAssembly`.
- Override `SaveChangesAsync` to automatically update `UpdatedAt` on modified entities.

### Step 2 — EF Core configurations
Create one configuration class per entity in `Infrastructure/Persistence/Configurations/`:

| Entity | Key rules |
|--------|-----------|
| `Contact` | `Emails`, `Phones`, `Addresses`, `Classifications` → `jsonb`; `ContactType`, `Gender` → string enum |
| `Organization` | `Addresses`, `ContactPoints` → `jsonb`; `EntityType` → string enum; self-referencing FK `ParentEntityId` |
| `ContactEntityRelation` | FK to `Contact` and `Organization`; unique index on `(ContactId, EntityId, Role)` |
| `AuditLog` | `Changes` → `jsonb`; `Action` → string enum; no cascade delete |
| `DuplicateCandidate` | `MatchReasons` → `jsonb`; `Status` → string enum; index on `Status` |

### Step 3 — Repositories
Implement all five repositories extending `BaseRepository<T>`:
- `ContactRepository` — add `SearchAsync(string query)` using `EF.Functions.ILike` across name fields, national ID
- `EntityRepository` — add `SearchAsync(string query)`, `GetChildrenAsync(Guid parentId)`
- `RelationRepository` — add `GetByContactIdAsync`, `GetByEntityIdAsync`
- `AuditLogRepository` — add `GetPagedAsync(entityType, action, page, limit)`
- `DuplicateRepository` — add `GetPendingAsync()`, `GetByIdWithRecordsAsync(Guid id)`

### Step 4 — Unit of Work
Create `Infrastructure/Persistence/UnitOfWork.cs` implementing `IUnitOfWork`:
```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

### Step 5 — DI Registration
Create `Infrastructure/DependencyInjection.cs` with an extension method `AddInfrastructure(this IServiceCollection services, IConfiguration config)` that:
- Registers `AppDbContext` with `UseNpgsql` (connection string from `config["ConnectionStrings:Postgres"]`)
- Registers all repositories as scoped
- Registers `UnitOfWork` as scoped

Think step by step. Show your reasoning for any non-obvious EF Core decisions (e.g., why `jsonb` over owned entities, why a specific index).
