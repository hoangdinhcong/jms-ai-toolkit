---
name: conventions
description: Provides WorkBuddy coding conventions and style rules for TypeScript, Angular, and NestJS. Activates automatically for all work in jms-model, jms-api, jms-web covering imports, exports, naming, formatting, architecture patterns, error handling, and API design.
user-invocable: false
---

# WorkBuddy Coding Conventions

These conventions apply to all code across the WorkBuddy/JMS platform.

## Code Style Guidelines

- **TypeScript**: Use strict types, avoid any when possible
- **Imports**: Order by external, then internal, then relative paths. ALWAYS use single-line imports, even if they become very long. Never use multi-line destructuring for imports as they are automatically managed by the IDE
- **Exports**: ALWAYS use single-line exports, even if they become very long
- **Index Files**: index.ts files must be barrel-only (exports only); no logic or side effects
- **File Responsibility**: Keep one concern per file, avoid monolithic files and mixed responsibilities
- **Barrel Imports**: Prefer importing from barrel exports when available
- **Formatting**: 2-space indentation, single quotes
- **Function Parameters**: ALWAYS keep parameters on a single line, never split across multiple lines. Add a blank line between function declaration and the first line of the function body
- **Naming**: camelCase for variables/methods, PascalCase for classes/interfaces
- **Components**: Directive suffix, component suffix, kebab-case selectors
- **Nullish Coalescing**: Prefer `??`, `??=`, and `?.` over explicit `undefined`/`null` checks and `||` for defaults
- **Error Handling**: Use try/catch with specific error types
- **API Design**: Follow RESTful conventions, use DTOs
- **State Management**: Use NgRx for complex state, services for simpler cases
- **Documentation**: Document public APIs and complex functions

## Architecture Rules

- Microservice-based backend (NestJS) with CQRS pattern
- Angular frontend with domain-driven design
- MongoDB for persistence, Redis for caching
- Tenants are fully isolated by container and database; do not design cross-tenant access or aggregation

## Anti-Patterns

- **Prop Drilling**: If you find yourself passing data through multiple layers (frontend -> backend -> service -> another service), STOP and reconsider the architecture. Look for existing session/context data that can be accessed directly at the point of need instead of threading parameters through the entire call chain
- **ProtectionConfig**: Whenever you use ProtectionConfig you need to comment on the meaning of the protectionconfig and the values set

## Packages

- Use bun package manager
- Do not attempt to lint source code. It is not enabled in our project
