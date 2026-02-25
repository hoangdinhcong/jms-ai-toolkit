---
name: entity-framework
description: Guides implementation and debugging of the WorkBuddy Entity Framework across jms-model, jms-api, jms-web. Activates when working with entities, boards, dashboards, forms, metadata, data strategies, EntityDashboardContext, EntityFormService, registerTenantForm, registerViews, ActivityEntityName, or entity.dashboard/entity.form files.
---

# WorkBuddy Entity Framework Skill

Guides implementation and debugging of the Entity Framework - a layered system spanning jms-model, jms-api, and jms-web.

## When to Use

- Creating a new entity with Board (list/grid) and Detail (form) views
- Debugging entity-related issues
- Working with entity metadata, forms, views, or data strategies

---

## Reference Files (Load On-Demand)

| Task | Load This File |
|------|----------------|
| Full entity implementation | [references/guide.md](references/guide.md) |
| Quick copy-paste templates | [references/patterns.md](references/patterns.md) |
| Debugging entity issues | [references/debug.md](references/debug.md) |

---

## Implementation Workflow

Follow this exact sequence when creating a new entity:

### Phase 1: jms-model (3 steps)

1. Create entity interface at `jms-model/entities/<entity>/<entity>.ts`
2. Add to `ActivityEntityName` enum
3. Export from index.ts files

### Phase 2: jms-api (8 steps)

4. Create MongoDB schema at `provider.mongoose/schema/<entity>.document.ts`
5. Add token to `JmsSchemaTokens` enum
6. Register in `MongooseJmsModule.getModule()`
7. Create metadata at `workbuddy/metadata/<entity>/<entity>.metadata.ts`
8. Create form definition at `workbuddy/metadata/<entity>/<entity>.definition.ts`
9. Create system views at `workbuddy/metadata/<entity>/<entity>.system.views.ts`
10. Create data strategy at `workbuddy/data/tenant/strategies/tenant.<entity>.entity.data.strategy.ts`
11. Register strategy in module and catalog

### Phase 3: jms-web (4 steps)

12. Create dashboard component at `workbuddy/modules/boards/<entity>/`
13. Create form service at `workbuddy/modules/managers/<entity>/`
14. Register in `EntityFormManageServiceFactory`
15. Configure routes

---

## File Locations Quick Reference

| Component | Location |
|-----------|----------|
| Entity Interface | `jms-model/entities/<entity>/<entity>.ts` |
| ActivityEntityName | `jms-model/entities/shared/activity.entity.name.ts` |
| MongoDB Schema | `jms-api/libs/providers/provider.mongoose/schema/<entity>.document.ts` |
| Schema Tokens | `jms-api/libs/providers/provider.mongoose/constants/schema.tokens.ts` |
| Mongoose Module | `jms-api/libs/providers/provider.mongoose/modules/mongoose.jms.module.ts` |
| Entity Metadata | `jms-api/workbuddy/metadata/<entity>/<entity>.metadata.ts` |
| Form Definition | `jms-api/workbuddy/metadata/<entity>/<entity>.definition.ts` |
| System Views | `jms-api/workbuddy/metadata/<entity>/<entity>.system.views.ts` |
| Data Strategy | `jms-api/workbuddy/data/tenant/strategies/tenant.<entity>.entity.data.strategy.ts` |
| Strategy Catalog | `jms-api/workbuddy/data/tenant/tenant.entity.data.strategy.catalog.ts` |
| Dashboard | `jms-web/workbuddy/modules/boards/<entity>/components/dashboard/` |
| Form Service | `jms-web/workbuddy/modules/managers/<entity>/services/` |
| Service Factory | `jms-web/workbuddy/ui/blocks/entity.form/services/entity.form.manage.service.factory.ts` |

---

## Validation Checklist

### jms-model
- [ ] Entity interface extends `Entity` or appropriate base
- [ ] Added to `ActivityEntityName` enum
- [ ] Exported from all index.ts files

### jms-api
- [ ] MongoDB schema has `@Schema`, `@Field` decorators
- [ ] Schema has appropriate indexes
- [ ] Token added to `JmsSchemaTokens`
- [ ] Registered in `MongooseJmsModule.getModule()`
- [ ] Metadata has `@MetadataEntity`, `@MetadataField` decorators
- [ ] Form definition uses `registerTenantForm()`
- [ ] Views use `registerViews()`
- [ ] Data strategy extends `TenantEntityDataStrategy`
- [ ] Strategy registered in module and catalog

### jms-web
- [ ] Dashboard component uses `EntityDashboardContext`
- [ ] Form service extends `EntityFormService`
- [ ] Service registered in `EntityFormManageServiceFactory`
- [ ] Routes configured for dashboard and detail

---

## Behavioral Rules

1. **Follow the exact sequence** - Dependencies matter (model -> api -> web)
2. **Use existing patterns** - Copy from Asset, Work, or Equipment
3. **Register everything** - Missing registrations are the #1 cause of issues
4. **Check the catalog** - Strategy must be in TenantEntityDataStrategyCatalog
5. **Validate at each step** - Don't skip to frontend without backend working

---

## Existing Examples

Reference these implementations:
- **Asset**: `workbuddy/metadata/asset/` (api), `workbuddy/modules/boards/assets/` (web)
- **Work**: `workbuddy/metadata/work/` (api), `workbuddy/modules/boards/work/` (web)
- **Equipment**: `workbuddy/metadata/equipment/` (api), `workbuddy/modules/boards/equipment/` (web)

---

## Related Skills

- **Angular Skill**: For dashboard/form component patterns
- **CQRS Skill**: For command/query handlers related to entity
- **Debug Skill**: For cross-layer debugging
