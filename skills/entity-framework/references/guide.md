# Complete Entity Framework Guide: Board & Detail Implementation

This guide covers the complete process for creating a new entity with Board (list/grid view) and Detail (form view) functionality.

## Overview

The entity framework follows a layered architecture:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    1. jms-model (TypeScript Interfaces)              │
│   - Entity interface definition                                       │
│   - ActivityEntityName enum                                          │
│   - Exports in index.ts                                              │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│                    2. jms-api (Backend Implementation)               │
│   a. MongoDB Document Schema                                         │
│   b. Schema Token Registration                                       │
│   c. Mongoose Module Registration                                    │
│   d. Repository Implementation                                       │
│   e. Entity Metadata (@MetadataEntity, @MetadataField)              │
│   f. Entity Form Definition (registerTenantForm)                     │
│   g. Entity Views (registerViews)                                    │
│   h. Entity Data Strategy                                            │
│   i. Strategy Registration in Catalog                                │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│                    3. jms-web (Frontend Implementation)              │
│   a. Dashboard Component (Board)                                     │
│   b. Form Manage Service (Detail)                                    │
│   c. Service Factory Registration                                    │
│   d. Routes Configuration                                            │
└──────────────────────────────────────────────────────────────────────┘
```

---

## PART 1: jms-model (Shared TypeScript)

### Step 1.1: Define Entity Interface

**Location**: `jms-model/entities/<entity-folder>/<entity>.ts`

```typescript
// jms-model/entities/project/project.ts

import { Entity } from '../entity';
import { EntitySubStatus } from '../entity.status';
import { ContactEntity } from '../contact';
import { SiteTiny } from '../shared';
import { Attachment } from '../shared/attachment';

export interface ProjectTypeEntity extends Entity {
    sortOrder: number;
}

export interface Project extends Entity {
    projectNumber?: string;
    projectType: ProjectTypeEntity;
    site?: SiteTiny;
    customer: ContactEntity;
    parent?: Entity;
    description?: string;
    budget?: number;
    startDate?: Date;
    dueDate?: Date;
    completedDate?: Date;
    customFields?: any;
    attachments?: Attachment[];
    createdAt?: Date;
    createdBy?: Entity;
    updatedAt?: Date;
    updatedBy?: Entity;
    status?: EntitySubStatus;
    tags?: Entity[];
}

// Command types for executeCommand
export type ProjectCommands =
    | { commandId: 'Archive', data: { id: string } }
    | { commandId: 'Clone', data: { id: string } };
```

### Step 1.2: Add to ActivityEntityName Enum

**Location**: `jms-model/entities/shared/activity.entity.name.ts`

```typescript
export enum ActivityEntityName {
    // ... existing entries

    Project = 'Project',
    ProjectType = 'ProjectType',
}
```

### Step 1.3: Export from Index Files

**Location**: `jms-model/entities/<entity-folder>/index.ts`

```typescript
// jms-model/entities/project/index.ts
export * from './project';
```

**Location**: `jms-model/entities/index.ts`

```typescript
// Add to existing exports
export * from './project';
```

---

## PART 2: jms-api (Backend)

### Step 2.1: Create MongoDB Document Schema

**Location**: `jms-api/libs/providers/provider.mongoose/schema/<entity>.document.ts`

```typescript
// jms-api/libs/providers/provider.mongoose/schema/project.document.ts

import { Project, ProjectTypeEntity, ContactEntity, Entity, EntitySubStatus, SiteTiny, Attachment } from '@jms/model';
import { Schema, Field, getMongooseSchema, Pre } from '../decorators/index';
import { EntitySchema } from './entity.document';
import { SchemaTypes } from 'mongoose';
import { AttachmentSchema } from './attachment.document';
import { SiteProTinySchema } from './site.pro.document';
import { ContactEntitySchema } from './contact.document';
import { EntitySubStatusSchema } from './entity.status.document';
import { v7 } from 'uuid';

@Schema('project',
    { timestamps: true },
    [
        // Indexes for common queries
        { fields: { _id: 1, active: 1 } },
        { fields: { active: 1, 'customer._id': 1 } },
        { fields: { active: 1, 'status.statusType': 1, name: 1 } },
        { fields: { active: 1, projectNumber: 1, name: 1 } },
        { fields: { updatedAt: 1 } },
    ]
)
class ProjectModel implements Required<Project> {

    @Field({ type: String, default: v7, required: true }) _id: string;
    @Field(String) name: string;
    @Field({ type: Boolean, default: true }) active: boolean;
    @Field(String) projectNumber: string;

    @Field(SchemaTypes.Mixed) projectType: ProjectTypeEntity;
    @Field(String) description: string;
    @Field(Number) budget: number;

    @Field(ContactEntitySchema) customer: ContactEntity;
    @Field(SiteProTinySchema) site: SiteTiny;

    @Field(EntitySchema) parent: Entity;

    @Field(Date) startDate: Date;
    @Field(Date) dueDate: Date;
    @Field(Date) completedDate: Date;

    @Field(SchemaTypes.Mixed) customFields: any;
    @Field([AttachmentSchema]) attachments: Attachment[];

    @Field(EntitySchema) createdBy: Entity;
    @Field(Date) createdAt: Date;

    @Field(EntitySchema) updatedBy: Entity;
    @Field(Date) updatedAt: Date;

    @Field([EntitySchema]) tags: Entity[];
    @Field(EntitySubStatusSchema) status: EntitySubStatus;

    // Non-persisted fields (for runtime only)
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

export const ProjectSchema = getMongooseSchema(ProjectModel);
```

### Step 2.2: Add Schema Token

**Location**: `jms-api/libs/providers/provider.mongoose/constants/schema.tokens.ts`

```typescript
export enum JmsSchemaTokens {
    // ... existing tokens

    Project = 'Project',
    ProjectType = 'ProjectType',
}
```

### Step 2.3: Register in Mongoose Module

**Location**: `jms-api/libs/providers/provider.mongoose/modules/mongoose.jms.module.ts`

```typescript
import { ProjectSchema } from '../schema/project.document';

// Inside MongooseJmsModule.getModule()
const schemas = [
    // ... existing schemas

    {
        schema: ProjectSchema,
        name: 'project',                           // mongoose model name
        token: JmsSchemaTokens.Project,
        entityName: ActivityEntityName.Project,    // links to entity framework
        collectionName: 'projects'                 // optional: explicit collection name
    },
];
```

### Step 2.4: Create Repository (if custom logic needed)

**Location**: `jms-api/libs/providers/provider.mongoose/repositories/jms/<entity>.repository.ts`

```typescript
// Optional: Only if you need custom repository logic
// Otherwise, the generic repository from EntityDataStrategy works

import { Injectable, Inject } from '@nestjs/common';
import { Model } from 'mongoose';
import { Project } from '@jms/model';
import { JmsSchemaTokens } from '../../constants/schema.tokens';

@Injectable()
export class MongoProjectRepository {
    constructor(
        @Inject(JmsSchemaTokens.Project)
        private model: Model<Project>
    ) {}

    // Custom methods here
}
```

### Step 2.5: Create Entity Metadata

**Location**: `jms-api/workbuddy/metadata/<entity>/<entity>.metadata.ts`

```typescript
// jms-api/workbuddy/metadata/project/project.metadata.ts

import {
    MetadataEntity,
    MetadataField,
    MetadataEntityFieldType,
    MetadataEntityFieldEditorType,
    ActivityEntityName,
    SecurityModuleName,
    LabelType,
    MetadataLookupTypeData,
    EntityStatusType,
    getEnumMetadata,
    getMetadata,
    Project
} from '@jms/model';

@MetadataEntity({
    displayName: 'Project',
    type: ActivityEntityName.Project,
    module: SecurityModuleName.Projects,  // Links to security rights
    importable: true,                      // Available in bulk import
    webForms: false,                       // Available in web forms
})
export class ProjectMetadataModel implements Required<Project> {

    @MetadataField({
        type: MetadataEntityFieldType.String,
        displayName: 'ID',
        readonly: true,
        importable: false
    })
    _id: string;

    @MetadataField({
        type: MetadataEntityFieldType.String,
        displayName: 'Name',
        required: true,
        textSearchable: true,
        uiOverrides: {
            sortable: true,
        }
    })
    name: string;

    @MetadataField({
        type: MetadataEntityFieldType.String,
        displayName: 'Project Number',
        importable: false,
        textSearchable: true,
        uiOverrides: {
            sortable: true,
        }
    })
    projectNumber: string;

    @MetadataField({
        type: MetadataEntityFieldType.Lookup,
        displayName: 'Project Type',
        typeData: {
            type: ActivityEntityName.ProjectType,
            data: ActivityEntityName.ProjectType
        },
        required: true,
        filterable: true,
        filterType: MetadataEntityFieldEditorType.EntityPicker,
        uiOverrides: {
            sortable: true,
        }
    })
    projectType: any;

    @MetadataField({
        type: MetadataEntityFieldType.Parent,
        displayName: 'Customer',
        typeData: ActivityEntityName.Customer,
        required: true,
        textSearchable: true,
        filterable: true,
        uiOverrides: {
            readonly: true,
            sortable: true,
        }
    })
    customer: any;

    @MetadataField({
        type: MetadataEntityFieldType.Parent,
        displayName: 'Site',
        typeData: ActivityEntityName.SitePro,
        textSearchable: true,
        filterable: true,
        uiOverrides: {
            readonly: true
        }
    })
    site: any;

    @MetadataField({
        type: MetadataEntityFieldType.Lookup,
        displayName: 'Status',
        typeData: {
            type: ActivityEntityName.EntityStatus,
            data: ActivityEntityName.Project
        } as MetadataLookupTypeData,
        readonly: false,
        filterable: false,
    })
    status: any;

    @MetadataField({
        type: MetadataEntityFieldType.Enum,
        displayName: 'Status Type',
        typeData: getEnumMetadata(EntityStatusType),
        readonly: true,
        filterable: true,
        importable: false
    })
    statusType: EntityStatusType;

    @MetadataField({
        type: MetadataEntityFieldType.String,
        displayName: 'Description',
    })
    description: string;

    @MetadataField({
        type: MetadataEntityFieldType.Currency,
        displayName: 'Budget',
    })
    budget: number;

    @MetadataField({
        type: MetadataEntityFieldType.Date,
        displayName: 'Start Date',
        highlightable: true,
    })
    startDate: Date;

    @MetadataField({
        type: MetadataEntityFieldType.Date,
        displayName: 'Due Date',
        highlightable: true,
    })
    dueDate: Date;

    @MetadataField({
        type: MetadataEntityFieldType.Date,
        displayName: 'Completed Date',
    })
    completedDate: Date;

    @MetadataField({
        type: [MetadataEntityFieldType.Tag],
        displayName: 'Labels',
        typeData: LabelType.Project,  // Add to LabelType enum
        filterable: true,
        filterType: MetadataEntityFieldEditorType.EntityPicker,
        textSearchable: true,
    })
    tags: any[];

    @MetadataField({
        type: MetadataEntityFieldType.CustomFields,
        typeData: ActivityEntityName.Project
    })
    customFields: any;

    @MetadataField({
        type: [MetadataEntityFieldType.Attachment],
        webForm: false,
        importable: false,
    })
    attachments: any[];

    @MetadataField({
        type: MetadataEntityFieldType.DateTime,
        displayName: 'Updated At',
        uiOverrides: {
            sortable: true,
        }
    })
    updatedAt: Date;

    // Non-decorated fields
    createdAt: Date;
    createdBy: any;
    updatedBy: any;
    parent: any;
    color: string;
    data: string;
    active: boolean;
    system: boolean;
}

export const ProjectMetadataSchema = getMetadata(ProjectMetadataModel);
```

### Step 2.6: Create Entity Form Definition

**Location**: `jms-api/workbuddy/metadata/<entity>/<entity>.definition.ts`

```typescript
// jms-api/workbuddy/metadata/project/project.definition.ts

import {
    ActivityEntityName,
    EntityFormId,
    EntityFormSectionType,
    EntityFormTabType,
    Project
} from '@jms/model';
import { registerTenantForm } from '@jms/providers/provider.entities/core';

registerTenantForm<Project>(ActivityEntityName.Project, {
    _id: EntityFormId.Manage,
    name: 'Manage',

    // Summary sections (left sidebar)
    summary: [
        {
            _id: 'summary',
            columns: 1,
            name: '',
            type: EntityFormSectionType.Fields,
            fields: [
                { _id: 'customer', name: 'Customer' },
                { _id: 'status' },
                { _id: 'projectType' },
            ]
        },
        {
            _id: 'dates',
            columns: 1,
            name: 'Timeline',
            type: EntityFormSectionType.Fields,
            fields: [
                { _id: 'startDate', name: 'Start' },
                { _id: 'dueDate', name: 'Due' },
            ]
        }
    ],

    // Tabs (main content area)
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
                        { _id: 'projectType' },
                        { _id: 'budget' },
                        { _id: 'tags', colspan: 2 },
                    ]
                },
                {
                    _id: 'dates',
                    columns: 2,
                    name: 'Dates',
                    type: EntityFormSectionType.Fields,
                    fields: [
                        { _id: 'startDate' },
                        { _id: 'dueDate' },
                        { _id: 'completedDate' },
                    ]
                }
            ]
        },
        {
            _id: 'tasks',
            name: 'Tasks',
            type: EntityFormTabType.Relation,
            relation: {
                _id: ActivityEntityName.Task,
                name: 'Tasks',
            }
        },
        {
            _id: 'files',
            name: 'Files',
            type: EntityFormTabType.Relation,
            relation: {
                _id: ActivityEntityName.File,
                name: 'Files',
            }
        },
        {
            _id: 'activities',
            name: 'Activities',
            type: EntityFormTabType.Relation,
            relation: {
                _id: ActivityEntityName.Activity,
                name: 'Activities',
            }
        },
    ]
});
```

### Step 2.7: Create Entity System Views

**Location**: `jms-api/workbuddy/metadata/<entity>/<entity>.system.views.ts`

```typescript
// jms-api/workbuddy/metadata/project/project.system.views.ts

import { ActivityEntityName, EntityViewDisplay, Project } from '@jms/model';
import {
    templateView,
    templateViewCard,
    templateViewList,
    templateViewFromOptions
} from '@jms/providers/provider.entities/views/model';
import { registerViews } from '@jms/providers/provider.entities/core';

// Card view layout
const projectCard = templateViewCard<Project>({
    title: 'name',
    description: 'description',
    status: 'status.name',
    body: ['customer.name', 'startDate', 'dueDate'],
    footer: ['budget', 'tags']
});

// List view columns
const projectList = templateViewList<Project>({
    fixed: ['name'],
    other: [
        'projectNumber',
        'customer.name',
        'status.name',
        'projectType.name',
        'startDate',
        'dueDate',
        'budget',
        'tags',
        'updatedAt'
    ]
});

registerViews(
    templateView<Project>(
        'All',
        ActivityEntityName.Project,
        'All Projects',
        [
            { _id: 'status.statusType', value: 'Active' }
        ],
        ['status', 'projectType', 'tags'],
        projectCard,
        projectList
    ),

    templateView<Project>(
        'Active',
        ActivityEntityName.Project,
        'Active Projects',
        [
            { _id: 'status.statusType', value: 'Active' }
        ],
        ['projectType', 'tags'],
        projectCard,
        projectList
    ),

    templateView<Project>(
        'Completed',
        ActivityEntityName.Project,
        'Completed Projects',
        [
            { _id: 'status.statusType', value: 'Completed' }
        ],
        ['projectType', 'tags'],
        projectCard,
        projectList
    ),

    templateViewFromOptions<Project>({
        _id: 'OverBudget',
        entity: ActivityEntityName.Project,
        name: 'Over Budget',
        display: EntityViewDisplay.List,
        filters: [
            { _id: 'status.statusType', value: 'Active' },
        ],
        quickFilters: ['customer', 'projectType'],
        list: projectList,
    })
);
```

### Step 2.8: Create Entity Data Strategy

**Location**: `jms-api/workbuddy/data/tenant/strategies/tenant.<entity>.entity.data.strategy.ts`

```typescript
// jms-api/workbuddy/data/tenant/strategies/tenant.project.entity.data.strategy.ts

import { Injectable } from '@nestjs/common';
import {
    ActivityEntityName,
    EntityItemsRequest,
    EntityItemsResponse,
    EntityViewMetadata,
    EntityViewSortDirection,
    getEffectiveEntityItemsRequest,
    Project,
    ProjectCommands,
    tryCatch
} from '@jms/model';
import {
    TenantEntityDataStrategy,
    getEntityItems,
    EntityQueryBuilderOptions
} from '@jms/providers/provider.entities/data';

@Injectable()
export class TenantProjectEntityDataStrategy extends TenantEntityDataStrategy<Project> {

    constructor() {
        super(ActivityEntityName.Project);
    }

    public override async getItems(request?: EntityItemsRequest): Promise<EntityItemsResponse<any>> {

        const { metadata, view, views } = await super.getHeader(request);

        const result = await getEntityItems(this.repository, {
            view,
            views,
            metadata,
            request,

            // Pre-match filters (always applied)
            buildPreMatch: (options: EntityQueryBuilderOptions) => {
                return [{ active: true }];
            },

            // Request-based filters
            buildRequestMatch: (options: EntityQueryBuilderOptions) => {
                const and = [];

                // Filter by parent (when embedded)
                if (options.request?.parentId) {
                    and.push({ 'customer._id': options.request.parentId });
                }

                return and;
            }
        });

        return result;
    }

    public override async save(item: Project): Promise<Project> {

        // Validation
        if (!item.name) {
            throw new Error('Project name is required');
        }

        if (!item.customer) {
            throw new Error('Customer is required');
        }

        // Generate project number if new
        if (!item._id) {
            item.projectNumber = await this.generateProjectNumber();
        }

        return super.save(item);
    }

    public override async executeCommand<T>(command: ProjectCommands): Promise<any> {

        switch (command.commandId) {
            case 'Archive':
                return this.archiveProject(command.data.id);

            case 'Clone':
                return this.cloneProject(command.data.id);

            default:
                return null;
        }
    }

    private async archiveProject(id: string): Promise<Project> {
        return this.writeRepository.patch(id, {
            active: false,
            updatedAt: new Date()
        }, null);
    }

    private async cloneProject(id: string): Promise<Project> {
        const original = await this.repository.getById(id);
        const clone = {
            ...original,
            _id: undefined,
            projectNumber: await this.generateProjectNumber(),
            name: `${original.name} (Copy)`,
            createdAt: new Date(),
            updatedAt: new Date()
        };
        return this.writeRepository.save(clone, null);
    }

    private async generateProjectNumber(): Promise<string> {
        // Implement your numbering logic
        const count = await this.repository.count({ active: true });
        return `PRJ-${String(count + 1).padStart(6, '0')}`;
    }
}
```

### Step 2.9: Register Strategy in Catalog

**Location**: `jms-api/workbuddy/data/tenant/tenant.entity.data.module.ts`

```typescript
import { TenantProjectEntityDataStrategy } from './strategies/tenant.project.entity.data.strategy';

@Module({
    providers: [
        // ... existing strategies
        TenantProjectEntityDataStrategy,
    ],
})
export class TenantEntityDataModule {}
```

**Location**: `jms-api/workbuddy/data/tenant/tenant.entity.data.strategy.catalog.ts`

```typescript
@Injectable()
export class TenantEntityDataStrategyCatalog {
    constructor(
        // ... existing strategies
        private project: TenantProjectEntityDataStrategy,
    ) {
        this.strategies = Object.freeze({
            // ... existing mappings
            [ActivityEntityName.Project]: this.project,
        });
    }
}
```

---

## PART 3: jms-web (Frontend)

### Step 3.1: Create Dashboard Component (Board View)

**Location**: `jms-web/workbuddy/modules/boards/<entity>/components/dashboard/<entity>.dashboard.component.ts`

```typescript
// jms-web/workbuddy/modules/boards/projects/components/dashboard/project.dashboard.component.ts

import { CommonModule } from '@angular/common';
import {
    ChangeDetectionStrategy,
    Component,
    computed,
    inject,
    input,
    signal,
    DestroyRef
} from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { ActivatedRoute, Router } from '@angular/router';
import {
    ActivityEntityName,
    EntityViewDisplay,
    ItemDeletedResponse
} from '@jms/model';
import { RightsService } from '@jms/utils/util.permissions';
import { EntityDataService } from '@workbuddy/services/entities';
import { ConfirmServicePro } from '@workbuddy/ui/blocks/confirm';
import {
    ENTITY_DASHBOARD_COMPONENTS,
    EntityDashboardTopLevelDirective
} from '@workbuddy/ui/blocks/entity.dashboard';
import { EntityDashboardContext } from '@workbuddy/ui/blocks/entity.dashboard/model';
import { ModuleViewHeaderCommandIds } from '@workbuddy/ui/blocks/module.header';
import { EntityCreatorService } from '@workbuddy/modules/creators';
import { Command } from '@workbuddy/ui/model';
import { EntityOutletComponent } from '@workbuddy/ui/blocks/entity.outlet';
import { GridApi } from 'ag-grid-enterprise';
import { MessageService } from 'primeng/api';

@Component({
    selector: 'w-project-dashboard',
    templateUrl: './project.dashboard.component.html',
    styleUrls: ['./project.dashboard.component.less'],
    standalone: true,
    changeDetection: ChangeDetectionStrategy.OnPush,
    providers: [EntityCreatorService],
    imports: [
        CommonModule,
        EntityOutletComponent,
        ENTITY_DASHBOARD_COMPONENTS,
    ],
    hostDirectives: [EntityDashboardTopLevelDirective]
})
export class ProjectDashboardComponent {

    private router = inject(Router);
    private route = inject(ActivatedRoute);
    private destroyRef = inject(DestroyRef);
    private entityService = inject(EntityDataService);
    private messageService = inject(MessageService);
    private confirmService = inject(ConfirmServicePro);
    private rightService = inject(RightsService);
    private creatorService = inject(EntityCreatorService);

    public $context = input.required<EntityDashboardContext>({ alias: 'context' });

    protected $effectiveContext = computed(() => {

        const context = this.$context() ?? {} as EntityDashboardContext;

        context.entity = context?.entity ?? ActivityEntityName.Project;
        context.title = context.title ?? 'Projects';
        context.supportedDisplayModes = context.supportedDisplayModes ?? [
            EntityViewDisplay.List,
            EntityViewDisplay.Card
        ];
        context.emptyIcon = context.emptyIcon ?? 'fa-light fa-folder-open';
        context.emptyMessage = context.emptyMessage ?? 'No projects found';

        if (context.navigate === undefined) {
            context.navigate = (entity, router, route) => {
                if (context?.itemSelection) return;
                this.router.navigate([`${entity._id}`], { relativeTo: this.route });
            }
        }

        // Security check
        const hasManageRights = this.rightService.hasRights(/* your rights */);
        context.hideAdd = !hasManageRights;

        context.view = {
            showCreate: hasManageRights,
            showEdit: hasManageRights,
            showFavorite: hasManageRights
        };

        context.grid = {
            setApi: (api) => this.$gridApi.set(api)
        };

        return context;
    });

    protected $gridApi = signal<GridApi>(null);

    onCommand(command: Command) {
        switch (command?._id) {
            case ModuleViewHeaderCommandIds.Add:
                this.onAdd();
                break;

            case 'Delete':
                this.onDelete(command.data);
                break;
        }
    }

    private async onAdd() {
        const params = await this.creatorService.show({
            entity: ActivityEntityName.Project,
            config: {
                width: '1200px',
                height: '700px',
                header: 'New Project',
                data: {
                    entity: ActivityEntityName.Project,
                    parentId: this.$effectiveContext().parentId
                }
            }
        });

        if (params?.saved && params?.action === 'save') {
            const context = this.$effectiveContext();
            context.navigate(params.saved, this.router, this.route);
        }
    }

    private async onDelete(data: any) {
        if (!data) return;

        const confirmed = await this.confirmService.confirm(
            'Delete Project',
            `Are you sure you want to delete "${data.name}"?`,
            'danger'
        );

        if (!confirmed) return;

        const deletion = this.entityService.getDelete(ActivityEntityName.Project);

        deletion.result$
            .pipe(takeUntilDestroyed(this.destroyRef))
            .subscribe((result) => {
                if (result?.isSuccess) {
                    this.messageService.add({
                        severity: 'success',
                        summary: 'Deleted',
                        detail: `${data.name} has been deleted`
                    });
                } else if (result?.isError) {
                    this.messageService.add({
                        severity: 'error',
                        summary: 'Delete Failed',
                        detail: result.failureReason.message
                    });
                }
            });

        deletion.mutate({ id: data._id });
    }
}
```

**Template**: `project.dashboard.component.html`

```html
@if ($effectiveContext(); as context) {
    <entity-outlet
        [context]="context"
        (onCommand)="onCommand($event)">
    </entity-outlet>
}
```

### Step 3.2: Create Form Manage Service (Detail View)

**Location**: `jms-web/workbuddy/modules/managers/<entity>/services/<entity>.entity.form.manage.service.ts`

```typescript
// jms-web/workbuddy/modules/managers/projects/services/project.entity.form.manage.service.ts

import { Injectable, inject } from '@angular/core';
import { ActivityEntityName, Project } from '@jms/model';
import { EntityFormService } from '@workbuddy/ui/blocks/entity.form/services';
import { EntityReactiveForm } from '@workbuddy/ui/blocks/entity.form/services/entity.form.reactive.transformer';
import { Command } from '@workbuddy/ui/model';
import { EntityDataService } from '@workbuddy/services/entities';
import { ENTITY_PROVIDERS } from '@workbuddy/ui/blocks/entity.form/decorators';

@Injectable()
export class ProjectEntityFormManageService extends EntityFormService<Project> {

    private entityDataService = inject(EntityDataService);

    static [ENTITY_PROVIDERS] = [
        ProjectEntityFormManageService,
    ];

    constructor(entity: ActivityEntityName, entityId: string) {
        super(entity, entityId);
        this.initialize();
    }

    private initialize() {
        // Register commands
        this.$vm.setAdditionalCommands(
            new Command({
                _id: 'archive',
                label: 'Archive',
                icon: 'fa-light fa-archive',
                action: () => this.archive()
            })
        );
    }

    protected override onConfigureForm(form: EntityReactiveForm) {
        // Configure form based on data
        const data = this.$vm.entityData();

        if (data?.status?.statusType === 'Completed') {
            form.getField('budget')?.$readonly.set(true);
        }
    }

    protected override onFieldModelChanged(path: string, value: any) {
        // Handle field changes
        if (path === 'startDate' && value) {
            const form = this.$form();
            const dueDate = form?.getValue('dueDate');
            if (!dueDate) {
                const defaultDue = new Date(value);
                defaultDue.setDate(defaultDue.getDate() + 30);
                form?.setValue('dueDate', defaultDue);
            }
        }
    }

    public override async onCommand(command: Command): Promise<void> {
        switch (command._id) {
            case 'archive':
                await this.archive();
                break;
        }
    }

    private async archive() {
        const data = this.$vm.entityData();
        if (!data) return;

        await this.entityDataService.executeCommand(
            this.$entity(),
            'Archive',
            { id: data._id }
        );

        this.toastService.success('Project archived');
        this.messenger.close();
    }
}
```

### Step 3.3: Register in Service Factory

**Location**: `jms-web/workbuddy/ui/blocks/entity.form/services/entity.form.manage.service.factory.ts`

```typescript
@Injectable()
export class EntityFormManageServiceFactory {

    private readonly ENTITY_SERVICES: Record<string, () => Promise<InstanceType<any>>> = {
        // ... existing services

        [ActivityEntityName.Project]: () =>
            import('@workbuddy/modules/managers/projects')
                .then(m => m.ProjectEntityFormManageService),
    }
}
```

### Step 3.4: Configure Routes

**Location**: `jms-web/workbuddy/modules/boards/<entity>/<entity>.routes.ts`

```typescript
// jms-web/workbuddy/modules/boards/projects/project.routes.ts

import { Routes } from '@angular/router';
import { ActivityEntityName } from '@jms/model';

export const projectRoutes: Routes = [
    {
        path: '',
        loadComponent: () =>
            import('./components/dashboard/project.dashboard.component')
                .then(m => m.ProjectDashboardComponent),
        data: {
            context: {
                entity: ActivityEntityName.Project,
                title: 'Projects'
            }
        }
    },
    {
        path: ':id',
        loadComponent: () =>
            import('@workbuddy/ui/blocks/entity.form')
                .then(m => m.EntityFormManageComponent),
        data: {
            entity: ActivityEntityName.Project
        }
    }
];
```

---

**Note**: See [../SKILL.md](../SKILL.md) for validation checklist.
