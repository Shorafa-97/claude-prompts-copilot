# Prompt 03 — Application Layer (CQRS with MediatR)

## Role
You are continuing the `ContactsX` .NET 9 backend. Core and Infrastructure are done. Now implement the **Application layer** (`ContactsX.Application`) using CQRS with MediatR, FluentValidation, and AutoMapper.

---

## Few-Shot Examples (Chain-of-Thought)

### Example A — Command with side effects (audit log + completeness)

**Input:** "How do I handle creating a contact so that the audit log is written and completeness is recalculated automatically?"

**Reasoning (CoT):**
1. `CreateContactCommand` carries the raw DTO fields.
2. The handler: (a) maps DTO → entity, (b) calls `ProfileCompletenessCalculator.Calculate(contact)`, (c) persists via `IContactRepository`, (d) writes an `AuditLog` entry, (e) commits via `IUnitOfWork`.
3. I don't put audit logic in the repository — that would couple persistence to audit. Instead a MediatR `IPipelineBehavior<TRequest, TResponse>` could generalize this, but for clarity in this system I handle it inline in the command handler.
4. The handler returns a `ContactDto` — never the raw entity — to prevent leaking domain internals to the API layer.

**Output (command + handler skeleton):**
```csharp
// Application/Contacts/Commands/CreateContact/CreateContactCommand.cs
public record CreateContactCommand(
    string FirstName, string LastName,
    string? FirstNameAr, string? LastNameAr,
    ContactType ContactType,
    Gender? Gender,
    List<string> Emails, List<string> Phones,
    Guid? CurrentEntityId, string? CurrentEntityName
) : IRequest<ContactDto>;

// Application/Contacts/Commands/CreateContact/CreateContactHandler.cs
public class CreateContactHandler(
    IContactRepository contacts,
    IAuditLogRepository auditLogs,
    IUnitOfWork uow,
    IMapper mapper) : IRequestHandler<CreateContactCommand, ContactDto>
{
    public async Task<ContactDto> Handle(CreateContactCommand cmd, CancellationToken ct)
    {
        var contact = mapper.Map<Contact>(cmd);
        contact.ProfileCompleteness = ProfileCompletenessCalculator.Calculate(contact);

        await contacts.AddAsync(contact, ct);
        await auditLogs.AddAsync(new AuditLog
        {
            EntityType = "contact", EntityId = contact.Id,
            Action = AuditAction.Create,
            Changes = JsonSerializer.Serialize(contact)
        }, ct);

        await uow.SaveChangesAsync(ct);
        return mapper.Map<ContactDto>(contact);
    }
}
```

---

### Example B — Query with pagination

**Input:** "How do I implement `GetContactsQuery` with search, type filter, and pagination?"

**Reasoning (CoT):**
1. The query carries optional `Search`, `Type`, `Status`, `Page`, `Limit` parameters.
2. The handler calls a repository method that builds the `IQueryable` and applies filters — keeping EF Core inside Infrastructure.
3. The result is a `PagedResult<ContactDto>` — a generic envelope that the API can serialize directly.
4. I never return `IQueryable` from the repository — that breaks the dependency rule.

**Output:**
```csharp
public record GetContactsQuery(
    string? Search, ContactType? Type, bool? IsActive,
    int Page = 1, int Limit = 20
) : IRequest<PagedResult<ContactDto>>;
```

---

## Your Task

Generate the complete **Application layer** for `ContactsX`.

### Step 1 — Shared DTOs and envelopes
Create in `Application/Common/`:
- `ContactDto` — all publicly visible contact fields (no raw entity exposure)
- `OrganizationDto` — entity fields
- `RelationDto` — relation fields with nested `ContactDto` or `OrganizationDto`
- `AuditLogDto`
- `DuplicateCandidateDto` — includes populated `Record1` and `Record2` as `ContactDto` or `OrganizationDto`
- `PagedResult<T>` — `{ Items: List<T>, Total: int, Page: int, Limit: int }`
- `ImportResultDto` — `{ Imported: int, Errors: List<string> }`

### Step 2 — AutoMapper profiles
Create `Application/Common/Mappings/MappingProfile.cs`:
- Map `CreateContactCommand → Contact`
- Map `Contact → ContactDto`
- Map `CreateOrganizationCommand → Organization`
- Map `Organization → OrganizationDto`
- Handle nested JSON list fields correctly

### Step 3 — Contact CQRS
Generate all commands and queries for contacts:

| Handler | MediatR type | Description |
|---------|-------------|-------------|
| `GetContactsHandler` | Query | Paged list with search + type + status filters |
| `GetContactByIdHandler` | Query | Single contact by ID; throws `NotFoundException` if missing |
| `CreateContactHandler` | Command | Create + recalculate completeness + audit log |
| `UpdateContactHandler` | Command | Patch update + recalculate completeness + audit log |
| `DeleteContactHandler` | Command | Soft or hard delete + audit log |
| `GetContactRelationsHandler` | Query | All entity relations for a contact |

### Step 4 — Organization CQRS
Mirror the Contact CQRS for organizations, adding:
- `GetOrganizationChildrenHandler` — returns child entities for a given parent ID

### Step 5 — Relations, Duplicates, Search CQRS

| Handler | Description |
|---------|-------------|
| `CreateRelationHandler` | Create contact–entity relation; validate both IDs exist |
| `DeleteRelationHandler` | Remove relation by ID |
| `GetDuplicatesHandler` | Paged list filtered by entityType and status |
| `MergeDuplicateHandler` | Keep master record, delete other, update candidate status |
| `DismissDuplicateHandler` | Update candidate status to dismissed |
| `UnifiedSearchHandler` | Query contacts + entities in parallel, return `{ Contacts, Entities }` |

### Step 6 — Import CQRS
Create `ImportContactsCommand` and `ImportOrganizationsCommand`:
- Accept `List<Dictionary<string, object>> Records`
- Apply column name normalization from BRD FR-IMP-02 / FR-IMP-05
- Apply gender normalization from BRD FR-IMP-03
- Apply entity name matching from BRD FR-IMP-04
- Return `ImportResultDto`

### Step 7 — FluentValidation validators
Create a validator for `CreateContactCommand` and `UpdateContactCommand`:
- `FirstName` and `LastName`: required, max 200 chars
- `ContactType`: must be valid enum value
- `Gender`: optional but must be valid if provided
- `Emails`: each item must be a valid email format
- Register a MediatR `ValidationBehavior<TRequest, TResponse>` pipeline behavior that runs validators automatically

### Step 8 — DI Registration
Create `Application/DependencyInjection.cs` with `AddApplication(this IServiceCollection services)` that registers:
- MediatR (`AddMediatR`) scanning `Application` assembly
- AutoMapper (`AddAutoMapper`) scanning `Application` assembly
- FluentValidation validators from `Application` assembly
- `ValidationBehavior` pipeline behavior

Think step by step. For each handler, explicitly state: what it reads, what it writes, and what exception it throws on failure.
