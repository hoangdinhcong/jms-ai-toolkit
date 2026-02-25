---
description: Scaffold CQRS components (commands, queries, events, handlers, sagas) following WorkBuddy patterns
disable-model-invocation: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Scaffold CQRS Command

Generate CQRS components following the exact patterns used in the WorkBuddy/JMS platform.

## Usage

```bash
/scaffold-cqrs [domain] [type]
```

**Examples**:
- `/scaffold-cqrs invoice crud` - Generate complete CRUD for Invoice
- `/scaffold-cqrs job create` - Generate only Create command + handler
- `/scaffold-cqrs bill saga` - Generate saga for Bill domain
- `/scaffold-cqrs` - Interactive mode (asks questions)

## Workflow

### Step 1: Parse Arguments

If arguments provided:
- Domain: invoice, bill, job, etc.
- Type: crud, create, update, delete, query, event, saga

If no arguments, enter **interactive mode**:
```
What domain are you working with? (e.g., invoice, job, bill): _
What do you want to create?
  [ ] Complete CRUD (Create, Read, Update, Delete)
  [ ] Command (specify: create, update, delete, or custom)
  [ ] Query (specify: getById, list, or custom)
  [ ] Event handler
  [ ] Saga workflow
```

### Step 2: Analyze Existing Structure

Before generating, ALWAYS:
1. Check if domain directory exists: `/jms-api/workbuddy/domains/tenant/{domain}/`
2. Check existing actions: `actions/{domain}.commands.ts`, `actions/{domain}.queries.ts`, `actions/{domain}.events.ts`
3. Check existing handlers: `handlers/handler.*.ts` and `external.handlers/handler.*.ts`
4. Check if domain module exists: `{domain}.domain.module.ts`
5. Identify what already exists to avoid duplication

**Important**: If domain directory doesn't exist, create the full structure including the domain module.

### Step 3: Determine What to Generate

Based on user input and existing code:

**For CRUD**:
- [ ] Commands: Create, Update, Delete (extends EntityCreateCommand, EntityUpdateCommand, EntityDeleteCommand)
- [ ] Queries: GetById, List
- [ ] Events: Created, Updated, Deleted, StatusChanged
- [ ] Event Handlers: Activity logging (for each event)
- [ ] Saga: Processing workflow (e.g., PDF generation, email)

**For Individual Components**:
- Generate only requested component
- Check dependencies (e.g., event handler needs event definition)

### Step 4: Gather Context

Ask user for critical information (only if not obvious):

1. **Security Rights**: What permission is required?
   - `FinanceSecurityRight.ManageInvoice`
   - `JobSecurityRight.ManageJob`
   - Custom rights

2. **Entity Type**: ActivityEntityName?
   - `ActivityEntityName.Invoice`
   - `ActivityEntityName.Bill`
   - `ActivityEntityName.Job`

3. **Saga Steps** (if generating saga): What workflow steps?
   - Example: "Generate PDF → Upload to storage → Update record → Send email"

4. **Repository Token**: What repository?
   - `RepositoryTokens.Invoice`
   - `RepositoryTokens.Bill`

5. **Service Dependencies**: Any special services needed?
   - `ServiceTokens.Pdf`
   - `ServiceTokens.Email`
   - `ServiceTokens.BlobStorage`

### Step 5: Generate Files

Generate files following these exact patterns:

#### Pattern 1: Command (EntityCreateCommand)

**File**: `actions/{domain}.commands.ts`

```typescript
import { ActivityEntityName, {Entity}, {SecurityRight} } from '@jms/model';
import { EntityCreateCommand, EntityUpdateCommand, EntityDeleteCommand } from '@jms/providers/provider.entities/data';

export class Create{Entity}Command extends EntityCreateCommand<{Entity}, {Entity}> {
  constructor(request: Partial<{Entity}>) {
    super(
      ActivityEntityName.{Entity},
      request,
      { rights: [{SecurityRight}.Manage{Entity}] }
    );
  }
}

export class Update{Entity}Command extends EntityUpdateCommand<{Entity}> {
  constructor(id: string, request: Partial<{Entity}>) {
    super(
      ActivityEntityName.{Entity},
      id,
      request,
      { rights: [{SecurityRight}.Manage{Entity}] }
    );
  }
}

export class Delete{Entity}Command extends EntityDeleteCommand<{Entity}> {
  constructor(public id: string) {
    super(
      ActivityEntityName.{Entity},
      id,
      { rights: [{SecurityRight}.Manage{Entity}] }
    );
  }
}
```

#### Pattern 2: Query

**File**: `actions/{domain}.queries.ts`

```typescript
import { CqrsQuery } from '@jms/providers/provider.cqrs';
import { {Entity} } from '@jms/model';

export class Get{Entity}ByIdQuery extends CqrsQuery<{Entity}> {
  constructor(public readonly {entity}Id: string) {
    super();
  }
}

export class List{Entity}sQuery extends CqrsQuery<{Entity}[]> {
  constructor(
    public readonly filters?: {Entity}Filters,
    public readonly pagination?: { page: number; pageSize: number }
  ) {
    super();
  }
}
```

#### Pattern 3: Event

**File**: `actions/{domain}.events.ts`

```typescript
import { CqrsEvent } from '@jms/providers/provider.cqrs';
import { {Entity} } from '@jms/model';

export class {Entity}CreatedEvent extends CqrsEvent {
  constructor(public readonly {entity}: {Entity}) {
    super();
  }
}

export class {Entity}UpdatedEvent extends CqrsEvent {
  constructor(
    public readonly before: {Entity},
    public readonly after: {Entity},
    public readonly patch: Partial<{Entity}>
  ) {
    super();
  }
}

export class {Entity}DeletedEvent extends CqrsEvent {
  constructor(public readonly {entity}: {Entity}) {
    super();
  }
}
```

#### Pattern 4: Command Handler

**File**: `handlers/handler.command.{domain}.{action}.ts`

```typescript
import { commandHandler } from '@jms/providers/provider.cqrs';
import { {Entity}sRepository, RepositoryTokens } from '@jms/common';
import { tryCatch } from '@jms/model';
import { Create{Entity}Command } from '../actions';
import { {Entity}CreatedEvent } from '../actions/{domain}.events';

export const create{Entity}CommandHandler = commandHandler(
  Create{Entity}Command,
  async (ctx, command) => {
    const {entity}s = await ctx.resolve<{Entity}sRepository>(
      RepositoryTokens.{Entity}
    );
    const { request } = command;

    // Create {entity}
    const [error, {entity}] = await tryCatch(
      {entity}s.create({
        ...request,
        createdBy: ctx.user._id,
        createdAt: new Date()
      })
    );

    if (error) {
      ctx.logger.trace('Unable to create {entity}');
      ctx.logger.exception(error);
      throw error;
    }

    ctx.logger.trace(`{Entity} created: ${{entity}._id}`);

    // Dispatch event
    await ctx.dispatch(new {Entity}CreatedEvent({entity}));

    return {entity};
  }
);
```

#### Pattern 5: Query Handler

**File**: `handlers/handler.query.{domain}.get.by.id.ts`

```typescript
import { queryHandler } from '@jms/providers/provider.cqrs';
import { {Entity}sRepository, RepositoryTokens } from '@jms/common';
import { NotFoundException } from '@nestjs/common';
import { Get{Entity}ByIdQuery } from '../actions';

export const get{Entity}ByIdQueryHandler = queryHandler(
  Get{Entity}ByIdQuery,
  async (ctx, query) => {
    const {entity}s = await ctx.resolve<{Entity}sRepository>(
      RepositoryTokens.{Entity}
    );

    const {entity} = await {entity}s.getById(query.{entity}Id);

    if (!{entity}) {
      throw new NotFoundException(`{Entity} ${query.{entity}Id} not found`);
    }

    return {entity};
  }
);
```

#### Pattern 6: Event Handler

**File**: `handlers/handler.event.{domain}.created.activity.ts`

```typescript
import { eventHandler } from '@jms/providers/provider.cqrs';
import { ActivityService, ServiceTokens } from '@jms/common';
import { {Entity}CreatedEvent } from '../actions/{domain}.events';

export const {entity}CreatedActivityHandler = eventHandler(
  {Entity}CreatedEvent,
  async (ctx, event) => {
    const activities = await ctx.resolve<ActivityService>(
      ServiceTokens.Activity
    );

    await activities.createActivity({
      entityType: '{Entity}',
      entityId: event.{entity}._id,
      action: 'Created',
      userId: ctx.user._id
    });

    ctx.logger.trace(`Activity logged for {entity} ${event.{entity}._id}`);
  }
);
```

#### Pattern 7: Saga

**File**: `handlers/saga.{domain}.{workflow}.ts`

```typescript
import { createSaga } from '@jms/providers/provider.cqrs/saga';
import { {Entity}CreatedEvent } from '../actions/{domain}.events';
import { ServiceTokens, RepositoryTokens } from '@jms/common';

interface {Entity}ProcessingData {
  {entity}Id: string;
  pdfBuffer?: Buffer;
  pdfUrl?: string;
  blobName?: string;
}

export const {entity}ProcessingSaga = createSaga<{Entity}ProcessingData>({
  name: '{Entity}ProcessingSaga',
  trigger: {Entity}CreatedEvent,
  steps: [
    // Step 1: Generate PDF
    {
      command: async (ctx, trigger, events, input) => {
        const event = trigger as {Entity}CreatedEvent;
        const pdfService = await ctx.resolve(ServiceTokens.Pdf);

        ctx.logger.trace(`Generating PDF for {entity} ${event.{entity}._id}`);

        const pdfBuffer = await pdfService.generate{Entity}Pdf(event.{entity});

        return {
          status: 'SUCCESS',
          data: {
            {entity}Id: event.{entity}._id,
            pdfBuffer
          }
        };
      }
    },

    // Step 2: Upload to storage
    {
      command: async (ctx, trigger, events, input) => {
        const storage = await ctx.resolve(ServiceTokens.BlobStorage);

        ctx.logger.trace(`Uploading PDF for {entity} ${input.{entity}Id}`);

        const blobName = `{entity}s/${input.{entity}Id}.pdf`;
        const url = await storage.upload(blobName, input.pdfBuffer);

        return {
          status: 'SUCCESS',
          data: { ...input, blobName, pdfUrl: url }
        };
      },
      compensation: async (ctx, trigger, events, input) => {
        if (input.blobName) {
          const storage = await ctx.resolve(ServiceTokens.BlobStorage);
          await storage.delete(input.blobName);
          ctx.logger.trace(`Deleted blob: ${input.blobName}`);
        }
        return { status: 'SUCCESS', data: input };
      }
    },

    // Step 3: Update entity with PDF URL
    {
      command: async (ctx, trigger, events, input) => {
        const {entity}s = await ctx.resolve(RepositoryTokens.{Entity});

        await {entity}s.update(input.{entity}Id, {
          pdfUrl: input.pdfUrl
        });

        ctx.logger.trace(`Updated {entity} ${input.{entity}Id} with PDF URL`);

        return { status: 'SUCCESS', data: input };
      },
      compensation: async (ctx, trigger, events, input) => {
        const {entity}s = await ctx.resolve(RepositoryTokens.{Entity});
        await {entity}s.update(input.{entity}Id, { pdfUrl: null });
        ctx.logger.trace(`Removed PDF URL from {entity} ${input.{entity}Id}`);
        return { status: 'SUCCESS', data: input };
      }
    }
  ]
});
```

#### Pattern 8: Handlers Index

**File**: `handlers/index.ts`

```typescript
import { create{Entity}CommandHandler } from './handler.command.{domain}.create';
import { update{Entity}CommandHandler } from './handler.command.{domain}.update';
import { delete{Entity}CommandHandler } from './handler.command.{domain}.delete';
import { get{Entity}ByIdQueryHandler } from './handler.query.{domain}.get.by.id';
import { list{Entity}sQueryHandler } from './handler.query.{domain}.list';
import { {entity}CreatedActivityHandler } from './handler.event.{domain}.created.activity';
import { {entity}ProcessingSaga } from './saga.{domain}.processing';

export const {DOMAIN}_HANDLERS = [
  // Sagas first (they subscribe to events)
  {entity}ProcessingSaga,

  // Commands
  create{Entity}CommandHandler,
  update{Entity}CommandHandler,
  delete{Entity}CommandHandler,

  // Queries
  get{Entity}ByIdQueryHandler,
  list{Entity}sQueryHandler,

  // Events
  {entity}CreatedActivityHandler
];
```

### Step 6: Update Module Registration

**Create or update domain module**: `/jms-api/workbuddy/domains/tenant/{domain}/{domain}.domain.module.ts`

If file doesn't exist, create it:
```typescript
import { Module } from '@nestjs/common';
import { registerHandlers } from '@jms/providers/provider.cqrs';
import { {DOMAIN}_HANDLERS } from './handlers';
import { {DOMAIN}_EXTERNAL_HANDLERS } from './external.handlers';

@Module({
  imports: []
})
export class {Domain}DomainModule {
  constructor() {
    registerHandlers(
      ...{DOMAIN}_HANDLERS,
      ...{DOMAIN}_EXTERNAL_HANDLERS
    );
  }
}
```

**Update parent module**: `/jms-api/workbuddy/domains/tenant.domain.module.ts`

Add import (if new domain):
```typescript
import { {Domain}DomainModule } from './tenant/{domain}/{domain}.domain.module';
```

Add to @Module imports array:
```typescript
@Global()
@Module({
  imports: [
    // ... existing domain modules
    {Domain}DomainModule,
    // ...
  ]
})
export class TenantDomainModule {}
```

### Step 7: Summary Report

After generation, provide a summary:

```
✅ CQRS Scaffolding Complete!

Generated Files (in /domains/tenant/{domain}/):
  ✓ actions/{domain}.commands.ts (3 commands)
  ✓ actions/{domain}.queries.ts (2 queries)
  ✓ actions/{domain}.events.ts (4 events)
  ✓ actions/index.ts
  ✓ handlers/handler.command.{domain}.create.ts
  ✓ handlers/handler.command.{domain}.update.ts
  ✓ handlers/handler.command.{domain}.delete.ts
  ✓ handlers/handler.query.{domain}.get.by.id.ts
  ✓ handlers/handler.query.{domain}.list.ts
  ✓ handlers/index.ts
  ✓ external.handlers/index.ts (empty, for future external event handlers)
  ✓ {domain}.domain.module.ts

Updated Files:
  ✓ tenant.domain.module.ts (imported {Domain}DomainModule)

Location: /jms-api/workbuddy/domains/tenant/{domain}/

Next Steps:
1. Review generated files for domain-specific logic
2. Implement repository methods if needed: RepositoryTokens.{Entity}
3. Add custom business logic in handlers
4. For cross-domain event handlers, add them to external.handlers/
5. Test with: `/review-code`
6. Update frontend to dispatch commands

Documentation:
- Reference examples: @../../../jms-docs/cqrs/EXAMPLES.md
- Reference best practices: @../../../jms-docs/cqrs/BEST_PRACTICES.md
```

## Important Notes

1. **Always check existing files** before generating to avoid duplication
2. **Follow exact naming conventions**: `handler.command.{domain}.{action}.ts`
3. **Use EntityCreateCommand/UpdateCommand/DeleteCommand** for entity operations
4. **Include security rights** on all write commands
5. **Use tryCatch** for all async operations
6. **Dispatch events** after successful operations
7. **Register handlers** in tenant.cqrs.module.ts
8. **Create sagas** for multi-step workflows

## Error Handling

If errors occur:
- Check domain directory exists: `/jms-api/workbuddy/domains/tenant/{domain}/`
- Verify entity exists in @jms/model
- Verify repository token exists in @jms/common
- Check security rights are defined
- Ensure parent directories exist before writing
- Verify domain module is imported in tenant.domain.module.ts

## Validation

After generation, automatically:
- Verify all imports are valid
- Check file naming follows conventions
- Ensure handlers are exported in index.ts
- Verify module registration is updated

## See Also

- CQRS Implementation Skill for patterns
- `/wb:review-code` to validate generated code
- Comprehensive examples: @../../../jms-docs/cqrs/EXAMPLES.md
