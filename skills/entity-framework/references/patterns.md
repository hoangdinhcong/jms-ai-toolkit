# Entity Framework Patterns

Quick copy-paste templates for common entity framework patterns.

---

## Key Imports

### jms-model

```typescript
import { ActivityEntityName, Entity, EntitySubStatus } from '@jms/model';
```

### jms-api - Schema

```typescript
import { Schema, Field, getMongooseSchema, Pre } from '../decorators/index';
import { JmsSchemaTokens } from '@jms/providers/provider.mongoose';
import { SchemaTypes } from 'mongoose';
import { v7 } from 'uuid';
```

### jms-api - Metadata

```typescript
import { MetadataEntity, MetadataField, MetadataEntityFieldType, MetadataEntityFieldEditorType, getMetadata, getEnumMetadata, SecurityModuleName, LabelType, MetadataLookupTypeData, EntityStatusType } from '@jms/model';
import { registerTenantForm } from '@jms/providers/provider.entities/core';
import { registerViews, templateView, templateViewCard, templateViewList, templateViewFromOptions } from '@jms/providers/provider.entities/views/model';
```

### jms-api - Strategy

```typescript
import { TenantEntityDataStrategy, getEntityItems, EntityQueryBuilderOptions } from '@jms/providers/provider.entities/data';
import { Injectable } from '@nestjs/common';
```

### jms-web - Dashboard

```typescript
import { EntityDashboardContext, ENTITY_DASHBOARD_COMPONENTS, EntityDashboardTopLevelDirective } from '@workbuddy/ui/blocks/entity.dashboard';
import { EntityOutletComponent } from '@workbuddy/ui/blocks/entity.outlet';
import { EntityCreatorService } from '@workbuddy/modules/creators';
import { EntityDataService } from '@workbuddy/services/entities';
```

### jms-web - Form Service

```typescript
import { EntityFormService } from '@workbuddy/ui/blocks/entity.form/services';
import { EntityReactiveForm } from '@workbuddy/ui/blocks/entity.form/services/entity.form.reactive.transformer';
import { ENTITY_PROVIDERS } from '@workbuddy/ui/blocks/entity.form/decorators';
```

---

## Entity Interface Template

```typescript
// jms-model/entities/<entity>/<entity>.ts

import { Entity } from '../entity';
import { EntitySubStatus } from '../entity.status';
import { ContactEntity } from '../contact';
import { Attachment } from '../shared/attachment';

export interface MyEntity extends Entity {
    entityNumber?: string;
    description?: string;
    customer: ContactEntity;
    status?: EntitySubStatus;
    customFields?: any;
    attachments?: Attachment[];
    tags?: Entity[];
    createdAt?: Date;
    createdBy?: Entity;
    updatedAt?: Date;
    updatedBy?: Entity;
}

export type MyEntityCommands =
    | { commandId: 'Archive', data: { id: string } }
    | { commandId: 'Clone', data: { id: string } };
```

---

## MongoDB Schema Template

```typescript
@Schema('myentity',
    { timestamps: true },
    [
        { fields: { _id: 1, active: 1 } },
        { fields: { active: 1, 'customer._id': 1 } },
        { fields: { active: 1, 'status.statusType': 1, name: 1 } },
        { fields: { updatedAt: 1 } },
    ]
)
class MyEntityModel implements Required<MyEntity> {
    @Field({ type: String, default: v7, required: true }) _id: string;
    @Field(String) name: string;
    @Field({ type: Boolean, default: true }) active: boolean;
    @Field(String) entityNumber: string;
    @Field(String) description: string;
    @Field(ContactEntitySchema) customer: ContactEntity;
    @Field(EntitySubStatusSchema) status: EntitySubStatus;
    @Field(SchemaTypes.Mixed) customFields: any;
    @Field([AttachmentSchema]) attachments: Attachment[];
    @Field([EntitySchema]) tags: Entity[];
    @Field(EntitySchema) createdBy: Entity;
    @Field(Date) createdAt: Date;
    @Field(EntitySchema) updatedBy: Entity;
    @Field(Date) updatedAt: Date;

    // Non-persisted
    color: string;
    data: string;
    system: boolean;

    @Pre('save')
    public onPreSave() {
        this._id = this._id ? this._id : v7();
        this.customFields = this?.customFields ?? {};
        this.createdAt = this.createdAt ? this.createdAt : new Date();
        this.updatedAt = new Date();
    }
}

export const MyEntitySchema = getMongooseSchema(MyEntityModel);
```

---

## Metadata Template

```typescript
@MetadataEntity({
    displayName: 'My Entity',
    type: ActivityEntityName.MyEntity,
    module: SecurityModuleName.MyModule,
    importable: true,
    webForms: false,
})
export class MyEntityMetadataModel implements Required<MyEntity> {
    @MetadataField({ type: MetadataEntityFieldType.String, displayName: 'ID', readonly: true, importable: false })
    _id: string;

    @MetadataField({ type: MetadataEntityFieldType.String, displayName: 'Name', required: true, textSearchable: true, uiOverrides: { sortable: true } })
    name: string;

    @MetadataField({ type: MetadataEntityFieldType.Parent, displayName: 'Customer', typeData: ActivityEntityName.Customer, required: true, filterable: true })
    customer: any;

    @MetadataField({ type: MetadataEntityFieldType.Lookup, displayName: 'Status', typeData: { type: ActivityEntityName.EntityStatus, data: ActivityEntityName.MyEntity } as MetadataLookupTypeData })
    status: any;

    @MetadataField({ type: MetadataEntityFieldType.CustomFields, typeData: ActivityEntityName.MyEntity })
    customFields: any;

    @MetadataField({ type: [MetadataEntityFieldType.Tag], displayName: 'Labels', typeData: LabelType.MyEntity, filterable: true })
    tags: any[];

    // Non-decorated
    active: boolean;
    createdAt: Date;
    createdBy: any;
    updatedAt: Date;
    updatedBy: any;
}

export const MyEntityMetadataSchema = getMetadata(MyEntityMetadataModel);
```

---

## Form Definition Template

```typescript
registerTenantForm<MyEntity>(ActivityEntityName.MyEntity, {
    _id: EntityFormId.Manage,
    name: 'Manage',

    summary: [
        {
            _id: 'summary',
            columns: 1,
            name: '',
            type: EntityFormSectionType.Fields,
            fields: [
                { _id: 'customer', name: 'Customer' },
                { _id: 'status' },
            ]
        }
    ],

    tabs: [
        {
            _id: 'details',
            name: 'Details',
            type: EntityFormTabType.Sections,
            sections: [
                {
                    _id: 'general',
                    columns: 2,
                    name: 'General',
                    type: EntityFormSectionType.Fields,
                    fields: [
                        { _id: 'name', colspan: 2 },
                        { _id: 'description', colspan: 2 },
                        { _id: 'tags', colspan: 2 },
                    ]
                }
            ]
        },
        {
            _id: 'files',
            name: 'Files',
            type: EntityFormTabType.Relation,
            relation: { _id: ActivityEntityName.File, name: 'Files' }
        },
        {
            _id: 'activities',
            name: 'Activities',
            type: EntityFormTabType.Relation,
            relation: { _id: ActivityEntityName.Activity, name: 'Activities' }
        }
    ]
});
```

---

## System Views Template

```typescript
const myEntityCard = templateViewCard<MyEntity>({
    title: 'name',
    description: 'description',
    status: 'status.name',
    body: ['customer.name'],
    footer: ['tags']
});

const myEntityList = templateViewList<MyEntity>({
    fixed: ['name'],
    other: ['customer.name', 'status.name', 'tags', 'updatedAt']
});

registerViews(
    templateView<MyEntity>(
        'All',
        ActivityEntityName.MyEntity,
        'All Items',
        [{ _id: 'status.statusType', value: 'Active' }],
        ['status', 'tags'],
        myEntityCard,
        myEntityList
    ),
    templateView<MyEntity>(
        'Active',
        ActivityEntityName.MyEntity,
        'Active Items',
        [{ _id: 'status.statusType', value: 'Active' }],
        ['tags'],
        myEntityCard,
        myEntityList
    )
);
```

---

## Data Strategy Template

```typescript
@Injectable()
export class TenantMyEntityDataStrategy extends TenantEntityDataStrategy<MyEntity> {

    constructor() {
        super(ActivityEntityName.MyEntity);
    }

    public override async getItems(request?: EntityItemsRequest): Promise<EntityItemsResponse<any>> {
        const { metadata, view, views } = await super.getHeader(request);

        return await getEntityItems(this.repository, {
            view,
            views,
            metadata,
            request,
            buildPreMatch: () => [{ active: true }],
            buildRequestMatch: (options) => {
                const and = [];
                if (options.request?.parentId) {
                    and.push({ 'customer._id': options.request.parentId });
                }
                return and;
            }
        });
    }

    public override async save(item: MyEntity): Promise<MyEntity> {
        if (!item.name) throw new Error('Name is required');
        return super.save(item);
    }

    public override async executeCommand<T>(command: MyEntityCommands): Promise<any> {
        switch (command.commandId) {
            case 'Archive':
                return this.writeRepository.patch(command.data.id, { active: false }, null);
            default:
                return null;
        }
    }
}
```

---

## EntityDashboardContext Template

```typescript
const context: EntityDashboardContext = {
    title: 'My Entities',
    entity: ActivityEntityName.MyEntity,
    supportedDisplayModes: [EntityViewDisplay.List, EntityViewDisplay.Card],
    emptyIcon: 'fa-light fa-folder-open',
    emptyMessage: 'No items found',

    navigate: (entity, router, route) => {
        router.navigate([`${entity._id}`], { relativeTo: route });
    },

    view: {
        showCreate: true,
        showEdit: true,
        showFavorite: true
    },

    grid: {
        setApi: (api) => this.$gridApi.set(api)
    }
};
```

---

## Dashboard Component Template

```typescript
@Component({
    selector: 'w-myentity-dashboard',
    template: `
        @if ($effectiveContext(); as context) {
            <entity-outlet [context]="context" (onCommand)="onCommand($event)" />
        }
    `,
    standalone: true,
    changeDetection: ChangeDetectionStrategy.OnPush,
    providers: [EntityCreatorService],
    imports: [EntityOutletComponent, ENTITY_DASHBOARD_COMPONENTS],
    hostDirectives: [EntityDashboardTopLevelDirective]
})
export class MyEntityDashboardComponent {
    private router = inject(Router);
    private route = inject(ActivatedRoute);

    public $context = input.required<EntityDashboardContext>({ alias: 'context' });

    protected $effectiveContext = computed(() => {
        const context = this.$context() ?? {} as EntityDashboardContext;
        context.entity = context.entity ?? ActivityEntityName.MyEntity;
        context.title = context.title ?? 'My Entities';
        // ... configure context
        return context;
    });

    onCommand(command: Command) {
        // Handle commands
    }
}
```

---

## Form Service Template

```typescript
@Injectable()
export class MyEntityFormManageService extends EntityFormService<MyEntity> {

    static [ENTITY_PROVIDERS] = [MyEntityFormManageService];

    constructor(entity: ActivityEntityName, entityId: string) {
        super(entity, entityId);
    }

    protected override onConfigureForm(form: EntityReactiveForm) {
        // Configure form fields based on data
    }

    protected override onFieldModelChanged(path: string, value: any) {
        // Handle field value changes
    }

    public override async onCommand(command: Command): Promise<void> {
        // Handle custom commands
    }
}
```

---

## Routes Template

```typescript
export const myEntityRoutes: Routes = [
    {
        path: '',
        loadComponent: () => import('./components/dashboard/myentity.dashboard.component').then(m => m.MyEntityDashboardComponent),
        data: { context: { entity: ActivityEntityName.MyEntity, title: 'My Entities' } }
    },
    {
        path: ':id',
        loadComponent: () => import('@workbuddy/ui/blocks/entity.form').then(m => m.EntityFormManageComponent),
        data: { entity: ActivityEntityName.MyEntity }
    }
];
```

---

## Service Factory Registration

```typescript
// In EntityFormManageServiceFactory
private readonly ENTITY_SERVICES: Record<string, () => Promise<InstanceType<any>>> = {
    [ActivityEntityName.MyEntity]: () =>
        import('@workbuddy/modules/managers/myentity').then(m => m.MyEntityFormManageService),
};
```
