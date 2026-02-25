# Entity Framework Troubleshooting

Diagnostic flows for common entity framework issues.

---

## Quick Diagnosis

| Symptom | Most Likely Cause | Check First |
|---------|-------------------|-------------|
| Board shows no data | Missing strategy registration | TenantEntityDataStrategyCatalog |
| Form fields not showing | Missing metadata | @MetadataField decorators |
| Save not working | Strategy validation | Data strategy save() method |
| Views dropdown empty | Missing views | registerViews() call |
| 404 on entity API | Missing schema | MongooseJmsModule registration |

---

## Symptom: Entity Board Not Loading Data

### Diagnostic Flow

```
1. ActivityEntityName
   └─ Is entity in enum?
   └─ File: jms-model/entities/shared/activity.entity.name.ts

2. JmsSchemaTokens
   └─ Is token defined?
   └─ File: jms-api/libs/providers/provider.mongoose/constants/schema.tokens.ts

3. MongooseJmsModule
   └─ Is schema registered with correct entityName?
   └─ File: jms-api/libs/providers/provider.mongoose/modules/mongoose.jms.module.ts

4. Data Strategy
   └─ Does strategy exist?
   └─ File: jms-api/workbuddy/data/tenant/strategies/tenant.<entity>.entity.data.strategy.ts

5. TenantEntityDataModule
   └─ Is strategy in providers array?
   └─ File: jms-api/workbuddy/data/tenant/tenant.entity.data.module.ts

6. TenantEntityDataStrategyCatalog
   └─ Is strategy in constructor injection?
   └─ Is strategy in strategies map?
   └─ File: jms-api/workbuddy/data/tenant/tenant.entity.data.strategy.catalog.ts

7. System Views
   └─ Are views registered via registerViews()?
   └─ File: jms-api/workbuddy/metadata/<entity>/<entity>.system.views.ts
```

### Common Fixes

**Missing from catalog:**
```typescript
// In TenantEntityDataStrategyCatalog constructor
constructor(
    private myEntity: TenantMyEntityDataStrategy,  // Add this
) {
    this.strategies = Object.freeze({
        [ActivityEntityName.MyEntity]: this.myEntity,  // Add this
    });
}
```

**Missing entityName in mongoose registration:**
```typescript
{
    schema: MyEntitySchema,
    name: 'myentity',
    token: JmsSchemaTokens.MyEntity,
    entityName: ActivityEntityName.MyEntity,  // THIS IS REQUIRED
}
```

---

## Symptom: Entity Form Not Showing Fields

### Diagnostic Flow

```
1. Entity Metadata
   └─ Does metadata file exist?
   └─ Is @MetadataField on each visible field?
   └─ File: jms-api/workbuddy/metadata/<entity>/<entity>.metadata.ts

2. Form Definition
   └─ Is registerTenantForm() called?
   └─ Are fields referenced in summary/tabs/sections?
   └─ File: jms-api/workbuddy/metadata/<entity>/<entity>.definition.ts

3. EntityFormService
   └─ Does service exist?
   └─ Is it registered in factory?
   └─ File: jms-web/workbuddy/modules/managers/<entity>/services/

4. Service Factory
   └─ Is lazy import added?
   └─ File: jms-web/workbuddy/ui/blocks/entity.form/services/entity.form.manage.service.factory.ts

5. Field Types
   └─ Do field types match metadata types?
   └─ Is typeData correct for lookups/parents?
```

### Common Fixes

**Field not in form definition:**
```typescript
registerTenantForm<MyEntity>(ActivityEntityName.MyEntity, {
    tabs: [{
        sections: [{
            fields: [
                { _id: 'myField' },  // Field ID must match metadata field name
            ]
        }]
    }]
});
```

**Wrong typeData for lookup:**
```typescript
@MetadataField({
    type: MetadataEntityFieldType.Lookup,
    displayName: 'Status',
    typeData: {
        type: ActivityEntityName.EntityStatus,
        data: ActivityEntityName.MyEntity  // Must match your entity
    } as MetadataLookupTypeData,
})
status: any;
```

---

## Symptom: Entity Not Saving

### Diagnostic Flow

```
1. Data Strategy save()
   └─ Is validation throwing errors?
   └─ Check console/network for error messages

2. MongoDB Schema
   └─ Are required fields defined?
   └─ Is @Pre('save') generating _id?

3. EntityFormService
   └─ Is form validation passing?
   └─ Check $form().valid

4. API Response
   └─ Check network tab for error response
   └─ Check API logs for exceptions
```

### Common Fixes

**Missing _id generation:**
```typescript
@Pre('save')
public onPreSave() {
    this._id = this._id ? this._id : v7();  // Ensure this exists
    this.createdAt = this.createdAt ? this.createdAt : new Date();
    this.updatedAt = new Date();
}
```

**Strategy validation too strict:**
```typescript
public override async save(item: MyEntity): Promise<MyEntity> {
    // Check what validation is failing
    console.log('Saving item:', item);

    if (!item.name) {
        throw new Error('Name is required');  // Is this being triggered?
    }

    return super.save(item);
}
```

---

## Symptom: Custom Commands Not Working

### Diagnostic Flow

```
1. Command Types
   └─ Is command defined in <Entity>Commands type?
   └─ File: jms-model/entities/<entity>/<entity>.ts

2. Data Strategy executeCommand()
   └─ Is command handled in switch statement?
   └─ File: jms-api/workbuddy/data/tenant/strategies/tenant.<entity>.entity.data.strategy.ts

3. Frontend Call
   └─ Is executeCommand() called with correct parameters?
   └─ Check: entityDataService.executeCommand(entity, commandId, data)
```

### Common Fixes

**Missing command in type:**
```typescript
export type MyEntityCommands =
    | { commandId: 'Archive', data: { id: string } }
    | { commandId: 'Clone', data: { id: string } }
    | { commandId: 'MyNewCommand', data: { id: string } };  // Add new command
```

**Missing command handler:**
```typescript
public override async executeCommand<T>(command: MyEntityCommands): Promise<any> {
    switch (command.commandId) {
        case 'Archive':
            return this.archive(command.data.id);
        case 'MyNewCommand':  // Add handler
            return this.handleMyNewCommand(command.data.id);
        default:
            return null;
    }
}
```

---

## Symptom: Views Dropdown Empty

### Diagnostic Flow

```
1. System Views File
   └─ Does file exist?
   └─ Is registerViews() called?
   └─ File: jms-api/workbuddy/metadata/<entity>/<entity>.system.views.ts

2. View Registration
   └─ Are views using correct entity name?
   └─ Check: ActivityEntityName.MyEntity in templateView()

3. Metadata Import
   └─ Is views file imported in metadata index?
   └─ Check: import './myentity.system.views';
```

### Common Fixes

**Views file not imported:**
```typescript
// In jms-api/workbuddy/metadata/<entity>/index.ts
import './myentity.metadata';
import './myentity.definition';
import './myentity.system.views';  // Must be imported!
```

**Wrong entity in view:**
```typescript
registerViews(
    templateView<MyEntity>(
        'All',
        ActivityEntityName.MyEntity,  // Must match your entity exactly
        'All Items',
        // ...
    )
);
```

---

## Symptom: Dashboard Navigation Not Working

### Diagnostic Flow

```
1. Routes Configuration
   └─ Is :id route defined?
   └─ File: jms-web/workbuddy/modules/boards/<entity>/<entity>.routes.ts

2. EntityDashboardContext navigate
   └─ Is navigate function defined?
   └─ Is it using correct router.navigate path?

3. EntityFormManageComponent
   └─ Is entity data attribute passed?
```

### Common Fixes

**Missing detail route:**
```typescript
export const myEntityRoutes: Routes = [
    { path: '', loadComponent: () => import('./dashboard').then(m => m.MyEntityDashboardComponent) },
    { path: ':id', loadComponent: () => import('@workbuddy/ui/blocks/entity.form').then(m => m.EntityFormManageComponent), data: { entity: ActivityEntityName.MyEntity } }  // THIS ROUTE
];
```

**Navigate function issue:**
```typescript
context.navigate = (entity, router, route) => {
    if (context?.itemSelection) return;  // Don't navigate in selection mode
    router.navigate([`${entity._id}`], { relativeTo: route });
};
```

---

## Symptom: API Returns 404 for Entity

### Diagnostic Flow

```
1. Schema Registration
   └─ Is schema in MongooseJmsModule?
   └─ Is collection name correct?

2. API Endpoint
   └─ Check /api/entity/<entityName>
   └─ EntityName must match ActivityEntityName exactly

3. Entity Controller
   └─ Is entity included in EntityController entities?
```

### Common Fixes

**Schema not registered:**
```typescript
// In MongooseJmsModule.getModule()
const schemas = [
    {
        schema: MyEntitySchema,
        name: 'myentity',
        token: JmsSchemaTokens.MyEntity,
        entityName: ActivityEntityName.MyEntity,
        collectionName: 'myentities'  // Optional: explicit collection
    },
];
```

---

## Debug Checklist

Use this checklist to systematically debug entity issues:

- [ ] Entity exists in ActivityEntityName enum
- [ ] Token exists in JmsSchemaTokens enum
- [ ] Schema registered in MongooseJmsModule
- [ ] Metadata file exists with @MetadataEntity decorator
- [ ] All visible fields have @MetadataField decorator
- [ ] Form definition exists with registerTenantForm()
- [ ] Views registered with registerViews()
- [ ] Data strategy extends TenantEntityDataStrategy
- [ ] Strategy registered in TenantEntityDataModule providers
- [ ] Strategy added to TenantEntityDataStrategyCatalog
- [ ] Dashboard component exists
- [ ] Form service exists
- [ ] Form service registered in factory
- [ ] Routes configured for both list and detail

---

## Getting More Help

1. Check API logs: `docker logs jms-api -f`
2. Check browser console for errors
3. Check network tab for API responses
4. Reference existing entities: Asset, Work, Equipment
5. Use **Debug Skill** for cross-layer issues
