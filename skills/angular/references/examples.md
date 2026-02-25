# WorkBuddy Frontend Code Examples

Simple, practical examples following WorkBuddy patterns.

## Table of Contents

1. [Basic Standalone Component](#1-basic-standalone-component)
2. [Form Field with ControlValueAccessor](#2-form-field-with-controlvalueaccessor)
3. [Simple Directive](#3-simple-directive)
4. [Service with API Integration](#4-service-with-api-integration)
5. [Template with Native Control Flow](#5-template-with-native-control-flow)
6. [Host Binding Styling](#6-host-binding-styling)
7. [Entity Picker Usage](#7-entity-picker-usage)
8. [Signal Forms - Complete Example](#8-signal-forms---complete-example)

---

## 1. Basic Standalone Component

**user-badge.component.ts**

```typescript
import { Component, ChangeDetectionStrategy, ViewEncapsulation, signal, computed, inject, input, output } from '@angular/core';
import { CommonModule } from '@angular/common';

interface User {
  _id: string;
  name: string;
  email: string;
  avatar?: string;
  isOnline?: boolean;
  firstName?: string;
  lastName?: string;
}

@Component({
  selector: 'user-badge',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  encapsulation: ViewEncapsulation.None,
  imports: [CommonModule],
  template: `
    <div class="user-badge">
      @if ($user(); as user) {
        <img [src]="user.avatar" [alt]="user.name" />
        <span>{{ user.name }}</span>
        @if ($showStatus()) {
          <span class="status" [class.online]="user.isOnline">
            {{ user.isOnline ? 'Online' : 'Offline' }}
          </span>
        }
      } @else {
        <span class="placeholder">No user</span>
      }
    </div>
  `,
  styleUrl: './user-badge.component.less',
  host: {
    '[class.user-badge-large]': '$size() === "large"',
    '[class.user-badge-small]': '$size() === "small"',
    '(click)': 'handleClick($event)'
  }
})
export class UserBadgeComponent {
  // Inputs
  public $user = input<User>(null, { alias: 'user' });
  public $size = input<'small' | 'medium' | 'large'>('medium', { alias: 'size' });
  public $showStatus = input(true, { alias: 'showStatus' });

  // Outputs
  public onUserClick = output<User>();

  // Computed
  protected $displayName = computed(() => {
    const user = this.$user();
    return user ? `${user.firstName} ${user.lastName}` : 'Unknown';
  });

  // Methods
  protected handleClick(event: Event): void {
    const user = this.$user();
    if (user) {
      this.onUserClick.emit(user);
    }
  }
}
```

**user-badge.component.less**

```less
.user-badge {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  border-radius: 4px;
  background: var(--surface-100);
  cursor: pointer;

  &:hover {
    background: var(--surface-200);
  }

  img {
    width: 32px;
    height: 32px;
    border-radius: 50%;
  }

  .status {
    font-size: 12px;
    color: var(--text-secondary);

    &.online {
      color: var(--green-500);
    }
  }

  &-large img {
    width: 48px;
    height: 48px;
  }

  &-small img {
    width: 24px;
    height: 24px;
  }
}
```

---

## 2. Form Field with ControlValueAccessor

**custom-input.component.ts**

```typescript
import { Component, ChangeDetectionStrategy, effect, input, model } from '@angular/core';
import { NG_VALUE_ACCESSOR, ControlValueAccessor } from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'custom-input',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule],
  template: `
    <div class="custom-input">
      <label [for]="$id()">{{ $label() }}</label>
      <input
        [id]="$id()"
        [type]="$type()"
        [value]="$value()"
        [disabled]="$disabled()"
        [placeholder]="$placeholder()"
        (input)="handleInput($event)"
        (blur)="handleBlur()"
      />
      @if ($error()) {
        <span class="error">{{ $error() }}</span>
      }
    </div>
  `,
  styleUrl: './custom-input.component.less',
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: CustomInputComponent,
      multi: true
    }
  ]
})
export class CustomInputComponent implements ControlValueAccessor {
  // Inputs
  public $label = input<string>('', { alias: 'label' });
  public $placeholder = input<string>('', { alias: 'placeholder' });
  public $type = input<'text' | 'email' | 'password'>('text', { alias: 'type' });
  public $error = input<string>(null, { alias: 'error' });
  public $id = input<string>(`input-${Math.random()}`, { alias: 'id' });

  // Model for value
  public $value = model<string>('', { alias: 'value' });
  public $disabled = model(false, { alias: 'disabled' });

  // ControlValueAccessor implementation
  private changeFn: (value: any) => void;
  private touchedFn: () => void;
  private isWrite = false;

  writeValue(value: string): void {
    this.isWrite = true;
    this.$value.set(value ?? '');
  }

  registerOnChange(fn: any): void {
    this.changeFn = fn;
  }

  registerOnTouched(fn: any): void {
    this.touchedFn = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.$disabled.set(isDisabled);
  }

  constructor() {
    effect(() => {
      const value = this.$value();
      if (this.isWrite) {
        this.isWrite = false;
        return;
      }
      if (this.changeFn) this.changeFn(value);
    });
  }

  // Event handlers
  protected handleInput(event: Event): void {
    const target = event.target as HTMLInputElement;
    this.$value.set(target.value);
  }

  protected handleBlur(): void {
    if (this.touchedFn) this.touchedFn();
  }
}
```

---

## 3. Simple Directive

**highlight.directive.ts**

```typescript
import { Directive, ElementRef, inject, input, effect } from '@angular/core';

@Directive({
  selector: '[highlight]',
  standalone: true
})
export class HighlightDirective {
  private el = inject(ElementRef);

  // Inputs
  public $color = input<string>('yellow', { alias: 'highlight' });
  public $enabled = input(true, { alias: 'highlightEnabled' });

  constructor() {
    effect(() => {
      const color = this.$color();
      const enabled = this.$enabled();

      if (enabled) {
        this.el.nativeElement.style.backgroundColor = color;
      } else {
        this.el.nativeElement.style.backgroundColor = '';
      }
    });
  }
}
```

**Usage:**

```html
<p highlight="yellow" [highlightEnabled]="true">This text is highlighted</p>
<p [highlight]="customColor()">Dynamic color highlight</p>
```

---

## 4. Service with API Integration

**user.service.ts**

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface User {
  _id: string;
  name: string;
  email: string;
  avatar?: string;
  isOnline?: boolean;
}

@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private api = 'api/users';

  public getAll(): Observable<User[]> {
    return this.http.get<User[]>(this.api);
  }

  public getById(id: string): Observable<User> {
    return this.http.get<User>(`${this.api}/${id}`);
  }

  public create(user: Partial<User>): Observable<User> {
    return this.http.post<User>(this.api, user);
  }

  public update(id: string, user: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.api}/${id}`, user);
  }

  public delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.api}/${id}`);
  }
}
```

**Component using service:**

```typescript
import { Component, signal, inject, OnInit } from '@angular/core';
import { UserService, User } from './user.service';

@Component({
  selector: 'user-list',
  template: `
    @if ($loading()) {
      <p>Loading...</p>
    } @else if ($error()) {
      <p class="error">{{ $error() }}</p>
    } @else {
      @for (user of $users(); track user._id) {
        <user-badge [user]="user" (onUserClick)="handleUserClick($event)" />
      } @empty {
        <p>No users found</p>
      }
    }
  `
})
export class UserListComponent implements OnInit {
  private userService = inject(UserService);

  protected $users = signal<User[]>([]);
  protected $loading = signal(true);
  protected $error = signal<string>(null);

  ngOnInit(): void {
    this.loadUsers();
  }

  private loadUsers(): void {
    this.$loading.set(true);
    this.$error.set(null);

    this.userService.getAll().subscribe({
      next: (users) => {
        this.$users.set(users);
        this.$loading.set(false);
      },
      error: (error) => {
        this.$error.set(error.message);
        this.$loading.set(false);
      }
    });
  }

  protected handleUserClick(user: User): void {
    console.log('User clicked:', user);
  }
}
```

---

## 5. Template with Native Control Flow

**dashboard.component.html**

```html
<!-- Local variables with @let -->
@let currentUser = $currentUser();
@let items = $items();
@let total = items.length;

<div class="dashboard">
  <!-- Conditional rendering with @if -->
  @if (currentUser) {
    <header>
      <h1>Welcome, {{ currentUser.name }}</h1>
      @if (currentUser.isAdmin) {
        <span class="badge">Admin</span>
      }
    </header>
  }

  <!-- Switch statement with @switch -->
  @switch ($viewMode()) {
    @case ('grid') {
      <div class="grid">
        @for (item of items; track item._id) {
          <div class="card">
            <h3>{{ item.title }}</h3>
            <p>{{ item.description }}</p>
          </div>
        }
      </div>
    }
    @case ('list') {
      <ul class="list">
        @for (item of items; track item._id) {
          <li>
            <span>{{ $index + 1 }}. {{ item.title }}</span>
            @if ($first) {
              <span class="badge">First</span>
            }
          </li>
        }
      </ul>
    }
    @default {
      <p>Select a view mode</p>
    }
  }

  <!-- Loop with contextual variables -->
  @for (item of items; track item._id) {
    <div class="item">
      <span>{{ item.title }}</span>
      <span class="meta">
        Item {{ $index + 1 }} of {{ $count }}
        @if ($first) { (First) }
        @if ($last) { (Last) }
        @if ($even) { (Even) }
      </span>
    </div>
  } @empty {
    <div class="empty">
      <p>No items to display</p>
      <button (click)="createItem()">Create First Item</button>
    </div>
  }

  <!-- Nested conditions -->
  @if ($isLoading()) {
    <loading-spinner />
  } @else {
    @if (items.length > 0) {
      <p>Showing {{ items.length }} of {{ total }} items</p>
    } @else if ($hasError()) {
      <error-message [error]="$error()" />
    } @else {
      <empty-state />
    }
  }
</div>
```

---

## 6. Host Binding Styling

**button.component.ts**

```typescript
import { Component, ChangeDetectionStrategy, computed, signal, input } from '@angular/core';

export enum ButtonSize {
  Small = 'small',
  Medium = 'medium',
  Large = 'large'
}

export enum ButtonVariant {
  Primary = 'primary',
  Secondary = 'secondary',
  Tertiary = 'tertiary'
}

@Component({
  selector: 'app-button',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <button [disabled]="$disabled()">
      @if ($iconLeft()) {
        <i [class]="$iconLeft()"></i>
      }
      <span>{{ $label() }}</span>
      @if ($iconRight()) {
        <i [class]="$iconRight()"></i>
      }
    </button>
  `,
  styleUrl: './button.component.less',
  host: {
    '[class]': '$hostClass()',
    '[class.btn-disabled]': '$disabled()',
    '[class.btn-loading]': '$loading()',
    '[class.btn-small]': '$size() === "small"',
    '[class.btn-medium]': '$size() === "medium"',
    '[class.btn-large]': '$size() === "large"',
    '[attr.aria-disabled]': '$disabled()',
    '[attr.aria-busy]': '$loading()',
    '(click)': 'handleClick($event)',
    '(keydown.enter)': 'handleEnter($event)',
    '(keydown.space)': 'handleSpace($event)'
  }
})
export class ButtonComponent {
  public $label = input<string>('', { alias: 'label' });
  public $size = input<ButtonSize>(ButtonSize.Medium, { alias: 'size' });
  public $variant = input<ButtonVariant>(ButtonVariant.Primary, { alias: 'variant' });
  public $disabled = input(false, { alias: 'disabled' });
  public $loading = input(false, { alias: 'loading' });
  public $iconLeft = input<string>(null, { alias: 'iconLeft' });
  public $iconRight = input<string>(null, { alias: 'iconRight' });

  protected $hostClass = computed(() => {
    const variant = this.$variant();
    const size = this.$size();

    return [
      'app-button',
      `btn-${variant}`,
      `btn-${size}`
    ].join(' ');
  });

  protected handleClick(event: Event): void {
    if (this.$disabled() || this.$loading()) {
      event.preventDefault();
      event.stopPropagation();
    }
  }

  protected handleEnter(event: KeyboardEvent): void {
    this.handleClick(event);
  }

  protected handleSpace(event: KeyboardEvent): void {
    event.preventDefault();
    this.handleClick(event);
  }
}
```

---

## 7. Entity Picker Usage

**contact-form.component.ts**

```typescript
import { Component, signal, inject } from '@angular/core';
import { FormBuilder, FormGroup, ReactiveFormsModule } from '@angular/forms';
import { SelectComponent } from '@workbuddy/ui/fields/select';
import { EntityPickerDirective } from '@workbuddy/ui/directives/entity.picker.directive';

@Component({
  selector: 'contact-form',
  standalone: true,
  imports: [ReactiveFormsModule, SelectComponent, EntityPickerDirective],
  template: `
    <form [formGroup]="form">
      <!-- Entity picker for contacts -->
      <w-select
        formControlName="contactId"
        entityPicker="contact"
        [entityParentId]="$accountId()"
        [entityOptions]="{
          addText: 'Select Contact',
          loadType: 'lazy'
        }"
        placeholder="Choose a contact"
      />

      <!-- Entity picker for jobs -->
      <w-select
        formControlName="jobId"
        entityPicker="job"
        [entityParentId]="$contactId()"
        [entityOptions]="{
          addText: 'Select Job',
          loadType: 'eager',
          filters: [
            { propertyName: 'status', operator: 'equals', value: 'active' }
          ]
        }"
        placeholder="Choose a job"
      />

      <!-- Multiple selection -->
      <w-select
        formControlName="assigneeIds"
        entityPicker="user"
        [entityOptions]="{
          addText: 'Select Assignees',
          multiple: true
        }"
        placeholder="Choose assignees"
      />
    </form>
  `
})
export class ContactFormComponent {
  private fb = inject(FormBuilder);

  protected $accountId = signal<string>('account-123');
  protected $contactId = signal<string>(null);

  protected form: FormGroup = this.fb.group({
    contactId: [null],
    jobId: [null],
    assigneeIds: [[]]
  });

  constructor() {
    // Watch contact changes to update job filter
    this.form.get('contactId').valueChanges.subscribe(contactId => {
      this.$contactId.set(contactId);
      this.form.patchValue({ jobId: null }); // Reset job when contact changes
    });
  }
}
```

---

## 8. Signal Forms - Complete Example

Demonstrates: basic form, validators, cross-field validation, nested objects, arrays, and custom controls.

**order-form.component.ts**

```typescript
import { Component, ChangeDetectionStrategy, signal, computed, model, input, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { form, FormField, FormValueControl, ValidationError, required, email, min, minLength, schema, apply, applyEach, validate, validateHttp, customError, submit } from '@angular/forms/signals';

// Interfaces
interface Address { street: string; city: string; zip: string; }
interface OrderItem { name: string; quantity: number; price: number; }
interface OrderData {
  customerEmail: string;
  password: string;
  confirmPassword: string;
  rating: number;
  shipping: Address;
  items: OrderItem[];
}

// Reusable schemas
const addressSchema = schema<Address>((a) => {
  required(a.street); required(a.city); required(a.zip);
});

const itemSchema = schema<OrderItem>((i) => {
  required(i.name); min(i.quantity, 1); min(i.price, 0.01);
});

// Custom FormValueControl (replaces ControlValueAccessor)
@Component({
  selector: 'star-rating',
  standalone: true,
  template: `
    @for (i of [1,2,3,4,5]; track i) {
      <button type="button" (click)="value.set(i)" [disabled]="disabled()">{{ i <= value() ? '★' : '☆' }}</button>
    }
  `
})
export class StarRatingComponent implements FormValueControl<number> {
  readonly value = model(0);
  readonly disabled = input(false);
  readonly errors = input<readonly ValidationError[]>([]);
}

@Component({
  selector: 'order-form',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, FormField, StarRatingComponent],
  template: `
    <form (submit)="onSubmit($event)">
      <!-- Basic fields with validation -->
      <input type="email" [formField]="orderForm.customerEmail" placeholder="Email" />
      <input type="password" [formField]="orderForm.password" placeholder="Password" />
      <input type="password" [formField]="orderForm.confirmPassword" placeholder="Confirm" />

      <!-- Show errors -->
      @if (orderForm.confirmPassword().touched() && orderForm.confirmPassword().invalid()) {
        @for (err of orderForm.confirmPassword().errors(); track err.kind) {
          <span class="error">{{ err.message }}</span>
        }
      }

      <!-- Custom control -->
      <star-rating [field]="orderForm.rating" />

      <!-- Nested object -->
      <input [formField]="orderForm.shipping.street" placeholder="Street" />
      <input [formField]="orderForm.shipping.city" placeholder="City" />
      <input [formField]="orderForm.shipping.zip" placeholder="ZIP" />

      <!-- Array items -->
      @for (item of orderForm.items; track $index; let i = $index) {
        <div class="item-row">
          <input [formField]="item.name" placeholder="Item name" />
          <input type="number" [formField]="item.quantity" />
          <input type="number" [formField]="item.price" step="0.01" />
          <button type="button" (click)="removeItem(i)">X</button>
        </div>
      }
      <button type="button" (click)="addItem()">+ Add Item</button>

      <p>Total: {{ $total() | currency }}</p>

      <button type="submit" [disabled]="orderForm().invalid() || orderForm().submitting()">
        {{ orderForm().submitting() ? 'Saving...' : 'Submit' }}
      </button>
    </form>
  `
})
export class OrderFormComponent {
  protected model = signal<OrderData>({
    customerEmail: '',
    password: '',
    confirmPassword: '',
    rating: 0,
    shipping: { street: '', city: '', zip: '' },
    items: [{ name: '', quantity: 1, price: 0 }]
  });

  protected orderForm = form(this.model, (s) => {
    // Built-in validators
    required(s.customerEmail); email(s.customerEmail);
    required(s.password); minLength(s.password, 8);

    // Cross-field validation (auto-reactive!)
    validate(s.confirmPassword, ({ value, valueOf }) =>
      value() !== valueOf(s.password) ? customError({ kind: 'mismatch', message: 'Passwords must match' }) : null
    );

    // Custom validator
    validate(s.rating, ({ value }) =>
      value() < 1 ? customError({ kind: 'required', message: 'Rating required' }) : null
    );

    // Apply schemas
    apply(s.shipping, addressSchema);
    applyEach(s.items, itemSchema);
  });

  protected $total = computed(() => this.model().items.reduce((sum, i) => sum + i.quantity * i.price, 0));

  addItem(): void { this.orderForm.items()().value.update(items => [...items, { name: '', quantity: 1, price: 0 }]); }
  removeItem(i: number): void { this.orderForm.items()().value.update(items => items.filter((_, idx) => idx !== i)); }

  async onSubmit(event: SubmitEvent): Promise<void> {
    event.preventDefault();
    await submit(this.orderForm, async (form) => {
      console.log('Submit:', form().value());
      return null;
    });
  }
}
```

---

## Summary

These examples demonstrate:

1. ✅ Standalone components with OnPush
2. ✅ Signal-based state (signal, computed, effect)
3. ✅ input()/output()/model() functions
4. ✅ inject() for DI
5. ✅ Native control flow (@if/@for/@switch)
6. ✅ ControlValueAccessor for forms (legacy)
7. ✅ Host bindings for styling
8. ✅ Single-line imports and parameters
9. ✅ Proper naming ($signals, observables$)
10. ✅ LESS styling
11. ✅ Signal Forms: form(), validators, cross-field, nested, arrays, FormValueControl
