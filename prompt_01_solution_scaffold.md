# Prompt 01 — Solution Scaffold & Core Layer

## Role
You are a senior .NET 9 backend engineer applying **Clean Architecture** with CQRS, MediatR, and Entity Framework Core on PostgreSQL. You will now build the **ContactsX** backend step by step.

---

## Few-Shot Examples (Chain-of-Thought)

### Example A — Thinking about layers first

**Input:** "I need to model a `Contact` entity for a government contacts registry."

**Reasoning (CoT):**
1. `Contact` has no dependencies on infrastructure — it belongs in `Core`.
2. It contains domain rules (e.g. gender can only be male/female), so I put validation logic in a domain value object or enforce via enum.
3. JSON columns (emails, phones, addresses) are persisted as `jsonb` in PostgreSQL — I model them as C# `List<T>` backed by a value converter.
4. Profile completeness is a calculated field — I put the weight table and formula in a static `ProfileCompletenessCalculator` service in `Core`, keeping the entity clean.

**Output:**
```csharp
// Core/Entities/Contact.cs
public class Contact
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public string FirstName { get; set; } = default!;
    public string LastName { get; set; } = default!;
    public string? FirstNameAr { get; set; }
    public string? LastNameAr { get; set; }
    public Gender? Gender { get; set; }
    public ContactType ContactType { get; set; } = ContactType.Citizen;
    public List<string> Emails { get; set; } = [];
    public List<string> Phones { get; set; } = [];
    public int ProfileCompleteness { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
}
```

---

### Example B — Interfaces belong in Core, not Infrastructure

**Input:** "Where do I define `IContactRepository`?"

**Reasoning (CoT):**
1. `IContactRepository` is used by Application layer handlers — it must be resolvable without knowing about EF Core.
2. I define the interface in `Core/Interfaces/` so Application can reference it.
3. The concrete `ContactRepository` lives in `Infrastructure/Persistence/Repositories/` and implements the interface.
4. This keeps the dependency rule: Infrastructure → Core, never Core → Infrastructure.

**Output (interface):**
```csharp
// Core/Interfaces/IContactRepository.cs
public interface IContactRepository
{
    Task<Contact?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<List<Contact>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(Contact contact, CancellationToken ct = default);
    Task UpdateAsync(Contact contact, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}
```

---

## Your Task

Generate the full **solution scaffold and Core layer** for `ContactsX`:

### Step 1 — CLI scaffold
Produce the exact `dotnet` CLI commands to:
- Create the solution `ContactsX.sln`
- Add four projects: `ContactsX.API`, `ContactsX.Application`, `ContactsX.Core`, `ContactsX.Infrastructure`
- Add all necessary project references (following the dependency rule above)
- Add NuGet packages per project:
  - **API**: `Microsoft.AspNetCore.Authentication.JwtBearer`, `Swashbuckle.AspNetCore`
  - **Application**: `MediatR`, `FluentValidation`, `AutoMapper`
  - **Infrastructure**: `Npgsql.EntityFrameworkCore.PostgreSQL`, `Microsoft.EntityFrameworkCore.Design`
  - **Core**: no external packages

### Step 2 — Core layer: all entities
Generate the full C# entity classes in `Core/Entities/` for:
- `Contact` — all fields from BRD section 5.1 (FR-CON-01)
- `Entity` (organization) — all fields from BRD section 5.2 (FR-ENT-01)
- `ContactEntityRelation` — fields from BRD section 5.3 (FR-REL-01)
- `AuditLog` — fields from BRD section 5.9 (FR-AUD-01)
- `DuplicateCandidate` — fields from BRD section 5.5 (FR-DUP-01)

Rules:
- Use private setters where domain invariants must be protected.
- Model JSON array fields as `List<string>` or a typed class (e.g. `ContactAddress`).
- Include all enums in `Core/Enums/`: `ContactType`, `EntityType`, `Gender`, `DuplicateStatus`, `AuditAction`.

### Step 3 — Core layer: interfaces
Generate these interfaces in `Core/Interfaces/`:
- `IContactRepository`
- `IEntityRepository`
- `IRelationRepository`
- `IAuditLogRepository`
- `IDuplicateRepository`
- `IUnitOfWork`

### Step 4 — Domain service
Generate `Core/Services/ProfileCompletenessCalculator.cs` with:
- A static `Calculate(Contact contact)` method implementing the weight table from BRD FR-PCS-01 (total weight = 110).
- A static `Calculate(Organization entity)` method implementing the weight table from BRD FR-PCS-02 (total weight = 100).
- Formula: `(filledWeight / totalWeight) * 100`, rounded to nearest integer.

Think step by step before writing each class. Show your reasoning as inline comments where the logic is non-obvious.
