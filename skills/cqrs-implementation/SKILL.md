---
name: cqrs-implementation
description: Provides deep knowledge of WorkBuddy's custom CQRS architecture patterns. Activates when working with files in /workbuddy/domains/ or /workbuddy/cqrs/, or when mentioning CQRS, command, query, event, saga, domain module, handler registration, or event-driven architecture.
---

# CQRS Implementation Skill

Enables automatic application of WorkBuddy's custom CQRS patterns when working with commands, queries, events, and sagas.

## When This Skill Loads

Automatically activates when:
- Working with files in `/workbuddy/domains/` or `/workbuddy/cqrs/`
- User mentions "CQRS", "command", "query", "event", "saga", "domain module"
- Creating or modifying handlers
- Discussing event-driven architecture
- Working with `<Entity>DomainModule` files

## Reference Files (Load On-Demand)

| Task | Load This File |
|------|----------------|
| Full handler/saga patterns + examples | [references/patterns.md](references/patterns.md) |

## Quick Pattern Reference

### Project-Specific Patterns

WorkBuddy/JMS uses a **custom CQRS implementation**, NOT NestJS CQRS or MediatR.

#### Handler Functions
```typescript
import { commandHandler, queryHandler, eventHandler } from '@jms/providers/provider.cqrs';
import { createSaga } from '@jms/providers/provider.cqrs/saga';
```

#### Base Command Classes
```typescript
import { EntityCreateCommand, EntityUpdateCommand, EntityDeleteCommand } from '@jms/providers/provider.entities/data';

export class CreateInvoiceCommand extends EntityCreateCommand<Invoice, Invoice> {
  constructor(request: Partial<Invoice>) {
    super(ActivityEntityName.Invoice, request, { rights: [FinanceSecurityRight.ManageInvoice] });
  }
}
```

#### Error Handling - ALWAYS use tryCatch
```typescript
import { tryCatch } from '@jms/model';
const [error, result] = await tryCatch(someAsyncOperation());
if (error) {
  ctx.logger.trace('Operation failed');
  ctx.logger.exception(error);
  throw error;
}
```

#### Event Dispatching - AFTER successful operations
```typescript
const invoice = await invoiceRepo.create(data);
await ctx.dispatch(new InvoiceCreatedEvent(invoice));
return invoice;
```

### File Structure Conventions

**Entity-Specific Handlers** (business entities):
```
/workbuddy/domains/tenant/<entity>/
├── actions/
│   ├── {entity}.commands.ts
│   ├── {entity}.queries.ts
│   ├── {entity}.events.ts
│   └── index.ts
├── handlers/
│   ├── handler.command.{entity}.{action}.ts
│   ├── handler.query.{entity}.{action}.ts
│   ├── handler.event.{entity}.{action}.ts
│   ├── saga.{entity}.{workflow}.ts
│   └── index.ts  # Export {ENTITY}_HANDLERS array
├── external.handlers/
│   ├── handler.event.{other-entity}.{action}.ts
│   └── index.ts  # Export {ENTITY}_EXTERNAL_HANDLERS array
├── {entity}.domain.module.ts
└── index.ts
```

**Cross-Cutting Handlers** (system-wide concerns only):
```
/workbuddy/cqrs/tenant/
├── activity/
├── pdf/
└── ...
```

### Naming Conventions

| Type | Pattern | File Pattern |
|------|---------|-------------|
| Commands | `Create{Entity}Command` (Imperative) | `handler.command.{entity}.{verb}.ts` |
| Queries | `Get{Entity}ByIdQuery` (Get/List/Find) | `handler.query.{entity}.{action}.ts` |
| Events | `{Entity}CreatedEvent` (Past Tense) | `handler.event.{entity}.{action}.ts` |
| Sagas | `{entity}ProcessingSaga` (Workflow) | `saga.{entity}.{workflow}.ts` |

### Handler Registration

Handlers MUST be registered in the entity's domain module constructor:

```typescript
// handlers/index.ts
export const INVOICE_HANDLERS = [
  createInvoiceCommandHandler,
  updateInvoiceCommandHandler,
  getInvoiceByIdQueryHandler,
  invoiceCreatedActivityHandler
];

// invoice.domain.module.ts
@Module({ imports: [] })
export class InvoiceDomainModule {
  constructor() {
    registerHandlers(...INVOICE_HANDLERS, ...INVOICE_EXTERNAL_HANDLERS);
  }
}
```

### Context API

```typescript
export const exampleHandler = commandHandler(ExampleCommand, async (ctx, command) => {
  const user = ctx.user;
  const pdfService = await ctx.resolve<PdfService>(ServiceTokens.Pdf);
  const writeRepo = await ctx.getWriteRepository<Invoice>(JmsSchemaTokens.Invoice);
  const readRepo = await ctx.getReadRepository<Contact>(JmsSchemaTokens.Contact);
  ctx.logger.trace('Operation started');
  await ctx.dispatch(new InvoiceCreatedEvent(invoice));
});
```

### Common Imports

```typescript
// CQRS Core
import { commandHandler, queryHandler, eventHandler } from '@jms/providers/provider.cqrs';
import { CqrsCommand, CqrsQuery, CqrsEvent, CqrsContext } from '@jms/providers/provider.cqrs';
import { createSaga } from '@jms/providers/provider.cqrs/saga';

// Entity Commands
import { EntityCreateCommand, EntityUpdateCommand, EntityDeleteCommand } from '@jms/providers/provider.entities/data';

// Common
import { tryCatch, ActivityEntityName } from '@jms/model';
import { RepositoryTokens, ServiceTokens } from '@jms/common';

// Security
import { FinanceSecurityRight, JobSecurityRight } from '@jms/model';

// NestJS
import { NotFoundException, BadRequestException, ForbiddenException } from '@nestjs/common';
```

## Critical Reminders

1. **Never use generic CQRS patterns** - This project uses a custom implementation
2. **Always use `tryCatch`** for error handling
3. **Events are past-tense** - `JobCreated`, not `CreateJob`
4. **Dispatch events AFTER operations** succeed
5. **Register handlers in module constructors**
6. **Include security rights** on write commands
7. **Use `ctx` for everything** - no global state
8. **File naming is strict** - follow the conventions
9. **Sagas for multi-step workflows** - with compensation
10. **Reference existing examples** in `jms-api/workbuddy/domains/tenant/` (Bill, Job, Invoice)

## Integration with Other Skills

- **Entity Framework Skill**: For repository patterns and data strategies
- **Angular Skill**: For frontend command dispatching

## When User Asks About CQRS

1. **Always reference this project's patterns**, not generic CQRS
2. **Point to existing examples** in the codebase (Bill, PurchaseOrder, Invoice)
3. **Suggest using `/wb:cqrs-scaffold`** for generating new handlers
4. **Recommend `/wb:review-code`** for validation

## Remember

This is a **custom CQRS implementation**. Don't suggest:
- NestJS @CommandHandler decorators
- MediatR patterns from .NET
- Generic CQRS libraries
- Event sourcing patterns (not used here)

Always suggest:
- Project's commandHandler() function
- EntityCreateCommand base classes
- tryCatch error handling
- ctx.dispatch() for events
- Existing codebase patterns
