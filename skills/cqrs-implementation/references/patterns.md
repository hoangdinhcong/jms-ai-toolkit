# CQRS Full Pattern Reference

Comprehensive handler, saga, and registration patterns for WorkBuddy's custom CQRS implementation.

## Table of Contents

- [Command Handler Pattern](#command-handler-pattern)
- [Query Handler Pattern](#query-handler-pattern)
- [Event Handler Pattern](#event-handler-pattern)
- [Saga Pattern](#saga-pattern)
- [Security Rights Pattern](#security-rights-pattern)
- [Handler Registration Pattern](#handler-registration-pattern)
- [Domain Module Pattern](#domain-module-pattern)

---

## Command Handler Pattern

```typescript
import { commandHandler } from '@jms/providers/provider.cqrs';
import { tryCatch } from '@jms/model';
import { CreateInvoiceCommand } from '../actions';
import { InvoiceCreatedEvent } from '../actions/invoice.events';

export const createInvoiceCommandHandler = commandHandler(
  CreateInvoiceCommand,
  async (ctx, command) => {
    const invoices = await ctx.resolve<InvoicesRepository>(RepositoryTokens.Invoice);
    const { request } = command;

    const [error, invoice] = await tryCatch(
      invoices.create({
        ...request,
        createdBy: ctx.user._id,
        createdAt: new Date()
      })
    );

    if (error) {
      ctx.logger.trace('Unable to create invoice');
      ctx.logger.exception(error);
      throw error;
    }

    ctx.logger.trace(`Invoice created: ${invoice._id}`);
    await ctx.dispatch(new InvoiceCreatedEvent(invoice));
    return invoice;
  }
);
```

---

## Query Handler Pattern

```typescript
import { queryHandler } from '@jms/providers/provider.cqrs';
import { NotFoundException } from '@nestjs/common';
import { GetInvoiceByIdQuery } from '../actions';

export const getInvoiceByIdQueryHandler = queryHandler(
  GetInvoiceByIdQuery,
  async (ctx, query) => {
    const invoices = await ctx.resolve<InvoicesRepository>(RepositoryTokens.Invoice);
    const invoice = await invoices.getById(query.invoiceId);

    if (!invoice) {
      throw new NotFoundException(`Invoice ${query.invoiceId} not found`);
    }

    return invoice;
  }
);
```

---

## Event Handler Pattern

```typescript
import { eventHandler } from '@jms/providers/provider.cqrs';
import { InvoiceCreatedEvent } from '../actions/invoice.events';

export const invoiceCreatedActivityHandler = eventHandler(
  InvoiceCreatedEvent,
  async (ctx, event) => {
    const activities = await ctx.resolve<ActivityService>(ServiceTokens.Activity);

    const [error] = await tryCatch(
      activities.createActivity({
        entityType: 'Invoice',
        entityId: event.invoice._id,
        action: 'Created',
        userId: ctx.user._id
      })
    );

    if (error) {
      ctx.logger.trace('Failed to log activity');
      ctx.logger.exception(error);
      // Don't throw - let other handlers run
    }
  }
);
```

---

## Saga Pattern

```typescript
import { createSaga } from '@jms/providers/provider.cqrs/saga';
import { InvoiceCreatedEvent } from '../actions/invoice.events';

interface InvoiceProcessingData {
  invoiceId: string;
  pdfBuffer?: Buffer;
  pdfUrl?: string;
  blobName?: string;
}

export const invoiceProcessingSaga = createSaga<InvoiceProcessingData>({
  name: 'InvoiceProcessingSaga',
  trigger: InvoiceCreatedEvent,
  steps: [
    // Step 1: Generate PDF
    {
      command: async (ctx, trigger, events, input) => {
        const event = trigger as InvoiceCreatedEvent;
        const pdfService = await ctx.resolve(ServiceTokens.Pdf);

        ctx.logger.trace(`Generating PDF for invoice ${event.invoice._id}`);
        const pdfBuffer = await pdfService.generateInvoicePdf(event.invoice);

        return {
          status: 'SUCCESS',
          data: { invoiceId: event.invoice._id, pdfBuffer }
        };
      }
    },

    // Step 2: Upload to storage (with compensation)
    {
      command: async (ctx, trigger, events, input) => {
        const storage = await ctx.resolve(ServiceTokens.BlobStorage);
        const blobName = `invoices/${input.invoiceId}.pdf`;
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

    // Step 3: Update entity with PDF URL (with compensation)
    {
      command: async (ctx, trigger, events, input) => {
        const invoices = await ctx.resolve(RepositoryTokens.Invoice);
        await invoices.update(input.invoiceId, { pdfUrl: input.pdfUrl });
        ctx.logger.trace(`Updated invoice ${input.invoiceId} with PDF URL`);
        return { status: 'SUCCESS', data: input };
      },
      compensation: async (ctx, trigger, events, input) => {
        const invoices = await ctx.resolve(RepositoryTokens.Invoice);
        await invoices.update(input.invoiceId, { pdfUrl: null });
        ctx.logger.trace(`Removed PDF URL from invoice ${input.invoiceId}`);
        return { status: 'SUCCESS', data: input };
      }
    }
  ]
});
```

---

## Security Rights Pattern

ALWAYS include security rights for write operations:

```typescript
export class CreateInvoiceCommand extends EntityCreateCommand<Invoice, Invoice> {
  constructor(request: Partial<Invoice>) {
    super(
      ActivityEntityName.Invoice,
      request,
      {
        rights: [FinanceSecurityRight.ManageInvoice],
        meterName: 'invoices_created',
        meterType: MeterType.Counter
      }
    );
  }
}
```

---

## Handler Registration Pattern

```typescript
// handlers/index.ts
import { createInvoiceCommandHandler } from './handler.command.invoice.create';
import { updateInvoiceCommandHandler } from './handler.command.invoice.update';
import { deleteInvoiceCommandHandler } from './handler.command.invoice.delete';
import { getInvoiceByIdQueryHandler } from './handler.query.invoice.get.by.id';
import { invoiceCreatedActivityHandler } from './handler.event.invoice.created.activity';
import { invoiceProcessingSaga } from './saga.invoice.processing';

export const INVOICE_HANDLERS = [
  // Sagas first (they subscribe to events)
  invoiceProcessingSaga,

  // Commands
  createInvoiceCommandHandler,
  updateInvoiceCommandHandler,
  deleteInvoiceCommandHandler,

  // Queries
  getInvoiceByIdQueryHandler,

  // Events
  invoiceCreatedActivityHandler
];
```

```typescript
// external.handlers/index.ts
import { invoiceJobCompletedEventHandler } from './handler.event.job.completed';

export const INVOICE_EXTERNAL_HANDLERS = [
  invoiceJobCompletedEventHandler
];
```

---

## Domain Module Pattern

```typescript
import { Module } from '@nestjs/common';
import { registerHandlers } from '@jms/providers/provider.cqrs';
import { INVOICE_HANDLERS } from './handlers';
import { INVOICE_EXTERNAL_HANDLERS } from './external.handlers';

@Module({ imports: [] })
export class InvoiceDomainModule {
  constructor() {
    registerHandlers(
      ...INVOICE_HANDLERS,
      ...INVOICE_EXTERNAL_HANDLERS
    );
  }
}

// In tenant.domain.module.ts
@Global()
@Module({
  imports: [
    InvoiceDomainModule,
    BillDomainModule,
    // ... other domain modules
  ]
})
export class TenantDomainModule {}
```
