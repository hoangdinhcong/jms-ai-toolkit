---
name: angular
description: Guides Angular 20+ development in jms-web following WorkBuddy patterns. Activates when working with Angular components, directives, pipes, signals, NgRx, PrimeNG, standalone components, ChangeDetectionStrategy, inject(), input(), output(), computed(), effect(), Signal Forms, or .component.ts/.directive.ts files.
---

# WorkBuddy Angular Development Guide

This skill ensures you write proper Angular code following WorkBuddy's workspace patterns and conventions.

## When to Use This Skill

Activate this skill when:
- Working with Angular components (.component.ts, .component.html)
- Creating directives or pipes
- Using signals (signal, computed, effect, linkedSignal)
- Implementing ControlValueAccessor
- Working with NgRx or state management
- Using PrimeNG components
- Any work in jms-web project

**Keywords**: Angular, component, directive, pipe, signal, computed, effect, inject, input, output, model, NgRx, PrimeNG, standalone, OnPush, ControlValueAccessor, jms-web

---

## Table of Contents

- [Component Reuse Priority](#-component-reuse-priority-critical)
- [Pre-Implementation Checklist](#-pre-implementation-checklist)
- [Component Structure Pattern](#-component-structure-pattern)
- [Signal-Based State Management](#-signal-based-state-management)
- [Template Syntax](#-template-syntax-native-control-flow)
- [Styling Patterns](#-styling-patterns)
- [Dependency Injection](#-dependency-injection)
- [Forms Integration](#-forms-integration)
- [Signal Forms (Angular 21+)](#-signal-forms-angular-21-experimental)
- [Naming Conventions](#-naming-conventions)
- [Import Organization](#-import-organization)
- [Function Parameter Formatting](#-function-parameter-formatting)
- [WorkBuddy UI Component Reference](#-workbuddy-ui-component-reference)
- [Common Mistakes to Avoid](#-common-mistakes-to-avoid)

---

## 🎯 Component Reuse Priority (CRITICAL!)

**ALWAYS follow this order:**

1. **WorkBuddy UI Components FIRST**
   - Location: `/jms-web/workbuddy/ui/`
   - Check: `components/`, `blocks/`, `fields/`, `directives/`
   - Examples: `w-button`, `w-select`, `w-checkbox`, `entity-dashboard`

2. **PrimeNG Components SECOND**
   - Documentation: https://v17.primeng.org/
   - Examples: `p-button`, `p-dropdown`, `p-calendar`, `p-dialog`

3. **Create Custom Component LAST RESORT**
   - Only if WorkBuddy UI and PrimeNG don't have what you need
   - Follow patterns below when creating

---

## ✅ Pre-Implementation Checklist

Before writing any component:

- [ ] Search WorkBuddy UI components (`/jms-web/workbuddy/ui/`)
- [ ] Check PrimeNG documentation (v17.primeng.org)
- [ ] If creating custom, ensure it follows patterns below
- [ ] Use standalone component (default in Angular 20+)
- [ ] Apply `ChangeDetectionStrategy.OnPush`
- [ ] Use signal-based state (not traditional properties)
- [ ] Use `inject()` for DI (not constructor injection)
- [ ] Use native control flow (@if/@for/@switch, NOT *ngIf/*ngFor)
- [ ] Single-line imports (no multi-line destructuring)
- [ ] Single-line function parameters
- [ ] Proper naming conventions ($signals, observables$)

---

## 📋 Component Structure Pattern

**Quick reference - see [references/examples.md](references/examples.md) for full working examples**

```typescript
@Component({
  selector: 'app-example',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule],
  templateUrl: './example.component.html',
  styleUrl: './example.component.less',
  host: { '[class]': '$hostClass()' }
})
export class ExampleComponent {
  // DI with inject()
  protected readonly service = inject(MyService);

  // Inputs/Outputs/Model
  public $label = input<string>('', { alias: 'label' });
  public onValueChanged = output<string>();
  public $checked = model<boolean>(false, { alias: 'checked' });

  // Local state + computed
  protected $count = signal(0);
  protected $display = computed(() => `${this.$label()} (${this.$count()})`);

  // Effects
  constructor() {
    effect(() => { /* side effects */ });
  }

  // Methods
  protected handleClick(): void {
    this.$count.update(c => c + 1);
  }
}
```

---

## 🔄 Signal-Based State Management

### Signal Types

```typescript
// Simple signal
protected $value = signal<string>('initial');
this.$value.set('new value');
this.$value.update(prev => prev + ' updated');

// Computed (derived state)
protected $displayValue = computed(() => this.$value().toUpperCase());

// Model (two-way binding)
public $checked = model<boolean>(false, { alias: 'checked' });

// LinkedSignal (reactive watcher)
protected $viewService = linkedSignal(() => {
  const context = this.$context();
  return new EntityDashboardService(context, this.entityService);
});
```

### Signal Equality

```typescript
protected $filters = signal<EntityViewFilter[]>([], {
  equal: (a, b) => JSON.stringify(a) === JSON.stringify(b)
});
```

### Effects

```typescript
constructor() {
  // Basic effect
  effect(() => {
    const viewId = this.$viewId();
    if (!viewId) return;
    this.loadData(viewId);
  });

  // Untracked reads
  effect(() => {
    const primary = this.$primary();
    untracked(() => {
      const secondary = this.$secondary(); // Won't trigger
      this.doSomething(primary, secondary);
    });
  });

  // Effect cleanup
  effect((onCleanup) => {
    const sub = this.service.stream$.subscribe(d => this.$data.set(d));
    onCleanup(() => sub.unsubscribe());
  });
}
```

---

## 📝 Template Syntax (Native Control Flow)

**ALWAYS use native control flow, NOT structural directives**

### Conditional Rendering

```html
@if (isVisible()) {
  <div>Visible content</div>
} @else if (isLoading()) {
  <loading-spinner />
} @else {
  <empty-state />
}
```

### Loops

```html
@for (item of items(); track item.id) {
  <div>{{ item.name }}</div>
  <span>Index: {{ $index }}, Count: {{ $count }}</span>
} @empty {
  <p>No items found</p>
}
```

### Switch Statements

```html
@switch (displayMode()) {
  @case ('grid') { <grid-view /> }
  @case ('list') { <list-view /> }
  @default { <default-view /> }
}
```

### Local Variables

```html
@let viewService = $viewService();
@let userName = user().profile.name;
@let greeting = 'Hello, ' + userName;

<h1>{{ greeting }}</h1>
<div>{{ viewService.getData() }}</div>
```

---

## 🎨 Styling Patterns

### LESS Preprocessor

All styles use LESS (`.less` files). See [references/examples.md](references/examples.md) for full stylesheet examples.

### Host Bindings

**Use `host` object, NOT @HostBinding/@HostListener**

```typescript
@Component({
  selector: 'app-example',
  host: {
    '[class]': '$hostClass()',
    '[class.disabled]': '$disabled()',
    '(click)': 'handleClick($event)'
  }
})
```

### Computed Class Bindings

**Use computed signals, NOT ngClass**

```typescript
protected $hostClass = computed(() => {
  const classes = {
    'btn-small': this.$size() === ButtonSize.Small,
    'btn-primary': this.$variant() === 'primary'
  };
  return Object.keys(classes).filter(k => classes[k]).join(' ');
});
```

### Style Bindings

```html
<div [class.active]="isActive()"></div>
<div [style.height.px]="height()"></div>
```

### ViewEncapsulation

```typescript
encapsulation: ViewEncapsulation.None // For global styles
encapsulation: ViewEncapsulation.Emulated // Default, scoped
```

---

## 💉 Dependency Injection

**ALWAYS use `inject()` function, NOT constructor injection**

```typescript
export class ExampleComponent {
  protected readonly router = inject(Router);
  protected readonly service = inject(MyService);
  protected readonly destroyRef = inject(DestroyRef);
  protected readonly optional = inject(OptionalService, { optional: true });
}
```

---

## 📝 Forms Integration

### ControlValueAccessor Pattern

**Quick reference - see [references/examples.md](references/examples.md) for full working examples**

```typescript
import { NG_VALUE_ACCESSOR, ControlValueAccessor } from '@angular/forms';

@Component({
  selector: 'custom-input',
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: CustomInputComponent,
    multi: true
  }]
})
export class CustomInputComponent implements ControlValueAccessor {
  public $value = model<string>('', { alias: 'value' });
  public $disabled = model(false, { alias: 'disabled' });
  private changeFn: (value: any) => void;
  private touchedFn: () => void;
  private isWrite = false;

  writeValue(value: string): void {
    this.isWrite = true;
    this.$value.set(value);
  }

  registerOnChange(fn: any): void { this.changeFn = fn; }
  registerOnTouched(fn: any): void { this.touchedFn = fn; }
  setDisabledState(disabled: boolean): void { this.$disabled.set(disabled); }

  constructor() {
    effect(() => {
      const value = this.$value();
      if (this.isWrite) { this.isWrite = false; return; }
      if (this.changeFn) this.changeFn(value);
    });
  }
}
```

---

## 🆕 Signal Forms (Angular 21+) [EXPERIMENTAL]

Import from `@angular/forms/signals`. Model-first, signal-based forms with automatic sync.

### Forms Priority: Signal Forms → Reactive Forms → Template-Driven

### Quick Reference

```typescript
// 1. Model as signal
const model = signal({ email: '', password: '' });

// 2. Create form with validators
const myForm = form(model, (s) => {
  required(s.email); email(s.email);
  required(s.password); minLength(s.password, 8);
});

// 3. Template: <input [formField]="myForm.email" />
```

### Validators

| Validator | Usage |
|-----------|-------|
| `required(field)` | Field must have value |
| `email(field)` | Valid email format |
| `min(field, n)` / `max(field, n)` | Numeric bounds |
| `minLength(field, n)` / `maxLength(field, n)` | String length |
| `pattern(field, regex)` | Regex validation |
| `validate(field, fn)` | Custom sync validator |
| `validateHttp(field, config)` | Async HTTP validation |

### Custom & Cross-Field Validation

```typescript
// Custom validator
validate(s.website, ({ value }) => !value().startsWith('https://') ? customError({ kind: 'https', message: 'Must be HTTPS' }) : null);

// Cross-field (auto-reactive to both fields!)
validate(s.confirmPassword, ({ value, valueOf }) => value() !== valueOf(s.password) ? customError({ kind: 'mismatch', message: 'Must match' }) : null);
```

### Field State

```typescript
myForm.email().value();      // Get value
myForm.email().value.set();  // Set value
myForm.email().valid();      // Validation passed
myForm.email().errors();     // Error array [{kind, message}]
myForm.email().touched();    // User interacted
myForm.email().pending();    // Async in progress
```

### Nested & Arrays

```typescript
// Nested: myForm.address.city
// Arrays: myForm.items[0].name
// Reusable schemas
const addressSchema = schema<Address>((a) => { required(a.street); required(a.city); });
apply(s.billing, addressSchema);      // Nested object
applyEach(s.items, itemSchema);       // Array items
```

### Form Submission

```typescript
await submit(myForm, async (form) => {
  await api.save(form().value());
  return null; // or return errors array
});
```

### Custom Controls (FormValueControl)

```typescript
@Component({ selector: 'my-input', template: `<input [value]="value()" (input)="value.set($event.target.value)" />` })
export class MyInput implements FormValueControl<string> {
  readonly value = model('');
}
// Usage: <my-input [field]="myForm.name" />
```

---

## 📛 Naming Conventions

```typescript
// Signals: $ prefix
protected $count = signal(0);
protected $displayLabel = computed(() => '...');

// Observables: $ suffix
protected items$ = this.service.getItems();

// Methods: camelCase
protected handleClick(event: Event): void {}

// Classes/Services: PascalCase
export class ExampleComponent {}

// Interfaces: PascalCase with I prefix
export interface ICommand {}

// Types/Enums: PascalCase
export type ButtonMode = 'button' | 'menu';
export enum ButtonSize { Small, Large }
```

### Visibility Modifiers

```typescript
public $label = input<string>(''); // Template + external
protected $items = signal([]); // Template only
private changeFn: (v: any) => void; // Internal only
```

---

## 📦 Import Organization

**ALWAYS use single-line imports (no multi-line destructuring)**

```typescript
import { Component, ChangeDetectionStrategy, ViewEncapsulation, signal, computed, effect, inject, input, output, model } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ActivatedRoute, Router } from '@angular/router';

// Organize by:
// 1. External libraries (Angular, PrimeNG, RxJS, etc.)
// 2. Internal @jms imports
// 3. @workbuddy imports
// 4. Relative imports (./file)
```

---

## 🔧 Function Parameter Formatting

**ALWAYS keep parameters on a single line**

```typescript
public getDashboard(viewId?: string, page?: number, search?: string, filters?: EntityViewFilter[], pageSize: number = DEFAULT_PAGE_SIZE, sort: EntityViewSort = null): void {

  // Blank line after function declaration
  const data = this.loadData(viewId, page);
}
```

---

## 🗂️ WorkBuddy UI Component Reference

**Location**: `/jms-web/workbuddy/ui/`

**Structure**: `components/`, `fields/`, `blocks/`, `directives/`, `cell.renderers/`

**Common Components**:
- w-button, w-select, w-checkbox, w-calendar, w-field
- entity-dashboard, module-header

**Full directory listing**: Browse `/jms-web/workbuddy/ui/` in codebase

---

## 📚 Additional Resources

For full working code samples:
- **[references/examples.md](references/examples.md)**: Real-world component examples with templates and styling

For existing components:
- **WorkBuddy UI**: `/jms-web/workbuddy/ui/`
- **PrimeNG v17**: https://v17.primeng.org/

---

## 🚨 Common Mistakes to Avoid

❌ **Don't create custom components without checking WorkBuddy UI/PrimeNG first**
❌ **Don't use structural directives** (*ngIf, *ngFor, *ngSwitch)
❌ **Don't use @Input/@Output decorators** (use input()/output() functions)
❌ **Don't use constructor injection** (use inject() function)
❌ **Don't use ngClass/ngStyle** (use class/style bindings)
❌ **Don't use multi-line imports**
❌ **Don't use multi-line function parameters**
❌ **Don't forget $ prefix for signals**
❌ **Don't forget explicit visibility modifiers**
❌ **Don't forget OnPush change detection**

---

## Summary

1. **Check WorkBuddy UI → PrimeNG → Create Custom**
2. **Use standalone components with OnPush**
3. **Use signals for state** (signal, computed, effect)
4. **Use inject() for DI**
5. **Use native control flow** (@if/@for/@switch)
6. **Use single-line imports and parameters**
7. **Use proper naming** ($signals, observables$)
8. **Use host bindings for styling**
9. **Use LESS for styles**
10. **Forms: Signal Forms → Reactive Forms → Template-Driven**
11. **Use FormValueControl for custom Signal Form controls**
