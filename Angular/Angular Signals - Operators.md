  #sgnal/operator

Operators ใน Angular Signals เป็นเครื่องมือที่ใช้ในการจัดการและแปลงค่าของ signals แบบต่อเนื่อง (reactive streams) เพื่อให้สามารถดำเนินการกับข้อมูลที่เปลี่ยนแปลงได้อย่างมีประสิทธิภาพ

## ตารางเนื้อหา

- [Operators พื้นฐานของ Signal](https://claude.ai/chat/83f50173-b880-4ec7-9561-794f31635354#operators-%E0%B8%9E%E0%B8%B7%E0%B9%89%E0%B8%99%E0%B8%90%E0%B8%B2%E0%B8%99%E0%B8%82%E0%B8%AD%E0%B8%87-signal)
- [RxJS Interop - การใช้งานร่วมกับ RxJS](https://claude.ai/chat/83f50173-b880-4ec7-9561-794f31635354#rxjs-interop---%E0%B8%81%E0%B8%B2%E0%B8%A3%E0%B9%83%E0%B8%8A%E0%B9%89%E0%B8%87%E0%B8%B2%E0%B8%99%E0%B8%A3%E0%B9%88%E0%B8%A7%E0%B8%A1%E0%B8%81%E0%B8%B1%E0%B8%9A-rxjs)
- [Signal Input Operators](https://claude.ai/chat/83f50173-b880-4ec7-9561-794f31635354#signal-input-operators)
- [การสร้าง Custom Operators](https://claude.ai/chat/83f50173-b880-4ec7-9561-794f31635354#%E0%B8%81%E0%B8%B2%E0%B8%A3%E0%B8%AA%E0%B8%A3%E0%B9%89%E0%B8%B2%E0%B8%87-custom-operators)
- [ตัวอย่างการใช้งาน Operators](https://claude.ai/chat/83f50173-b880-4ec7-9561-794f31635354#%E0%B8%95%E0%B8%B1%E0%B8%A7%E0%B8%AD%E0%B8%A2%E0%B9%88%E0%B8%B2%E0%B8%87%E0%B8%81%E0%B8%B2%E0%B8%A3%E0%B9%83%E0%B8%8A%E0%B9%89%E0%B8%87%E0%B8%B2%E0%B8%99-operators)

## Operators พื้นฐานของ Signal

Signal ใน Angular มี operators พื้นฐานที่ใช้งานบ่อย ดังนี้

### 1. `set()`

ใช้สำหรับกำหนดค่าใหม่ให้กับ signal โดยตรง

```typescript
const counter = signal(0);
counter.set(5); // กำหนดค่าเป็น 5
```

### 2. `update()`

ใช้สำหรับอัพเดตค่าของ signal โดยอิงจากค่าปัจจุบัน

```typescript
const counter = signal(0);
counter.update(current => current + 1); // เพิ่มค่าขึ้น 1
```

### 3. `mutate()`

ใช้สำหรับแก้ไขค่าของ signal แบบ mutable โดยตรงโดยไม่ต้องสร้างอ็อบเจกต์ใหม่ **(เหมาะสำหรับ objects หรือ arrays)**

```typescript
const user = signal({ name: 'John', age: 30 });
user.mutate(value => {
  value.age = 31; // แก้ไขค่าโดยตรงโดยไม่ต้องสร้างอ็อบเจกต์ใหม่
});
```

### 4. `asReadonly()`

แปลง signal เป็น read-only signal ที่สามารถอ่านค่าได้อย่างเดียว แต่ไม่สามารถแก้ไขได้

```typescript
const counter = signal(0);
const readonlyCounter = counter.asReadonly();

// สามารถอ่านค่าได้
console.log(readonlyCounter());

// ไม่สามารถแก้ไขค่าได้
// readonlyCounter.set(1); // Error: Property 'set' does not exist
```

## RxJS Interop - การใช้งานร่วมกับ RxJS

Angular มี operators สำหรับแปลงระหว่าง Signal และ Observable ของ RxJS

### 1. `toSignal()`

แปลง Observable เป็น Signal

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';
import { map } from 'rxjs/operators';

// สร้าง Observable ที่นับเพิ่มทุก 1 วินาที
const counter$ = interval(1000).pipe(
  map(count => count + 1)
);

// แปลงเป็น Signal
const counter = toSignal(counter$, { initialValue: 0 });

// ใช้งาน Signal ในเทมเพลต
// {{ counter() }}
```

### 2. `toObservable()`

แปลง Signal เป็น Observable

```typescript
import { toObservable } from '@angular/core/rxjs-interop';

const count = signal(0);
const count$ = toObservable(count);

// สามารถใช้ operators ของ RxJS กับ Observable ได้
count$.pipe(
  map(value => value * 2),
  filter(value => value > 5)
).subscribe(value => {
  console.log('Filtered and doubled value:', value);
});
```

### 3. `fromEvent()`

สร้าง Signal จาก DOM event

```typescript
import { fromEvent } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-mouse-tracker',
  template: `
    <div>
      Mouse position: {{ mouseX() }}, {{ mouseY() }}
    </div>
  `
})
export class MouseTrackerComponent {
  private readonly mouseMove$ = fromEvent<MouseEvent>(document, 'mousemove');
  
  readonly mouseX = toSignal(
    this.mouseMove$.pipe(map(event => event.clientX)),
    { initialValue: 0 }
  );
  
  readonly mouseY = toSignal(
    this.mouseMove$.pipe(map(event => event.clientY)),
    { initialValue: 0 }
  );
}
```

## Signal Input Operators

Angular 17 แนะนำ Signal Inputs ซึ่งมี operators เฉพาะดังนี้

### 1. `input()`

ใช้สร้าง Signal Input ที่สามารถรับค่าจาก parent component ได้

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: `<h1>Hello, {{ name() }}!</h1>`
})
export class GreetingComponent {
  name = input<string>(); // ไม่มีค่าเริ่มต้น, ต้องกำหนดค่าจาก parent
}
```

### 2. `input()` พร้อมค่าเริ่มต้น

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: `<h1>Hello, {{ name() }}!</h1>`
})
export class GreetingComponent {
  name = input<string>('Guest'); // มีค่าเริ่มต้นเป็น 'Guest'
}
```

### 3. `input()` พร้อม transform function

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: `<h1>Hello, {{ name() }}!</h1>`
})
export class GreetingComponent {
  // แปลงข้อความเป็นตัวพิมพ์ใหญ่โดยอัตโนมัติ
  name = input<string, string>('guest', {
    transform: (value: string) => value.toUpperCase()
  });
}
```

### 4. `required` input

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-profile',
  template: `<div>User ID: {{ userId() }}</div>`
})
export class UserProfileComponent {
  // input ที่จำเป็นต้องระบุค่า
  userId = input.required<string>();
}
```

### 5. `model()`

ใช้สำหรับสร้าง two-way binding แบบ Signal

```typescript
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div>
      <button (click)="decrement()">-</button>
      <span>{{ count() }}</span>
      <button (click)="increment()">+</button>
    </div>
  `
})
export class CounterComponent {
  count = model<number>(0); // เริ่มต้นที่ 0 และสามารถอัพเดตได้ 2 ทาง
  
  increment() {
    this.count.update(value => value + 1);
  }
  
  decrement() {
    this.count.update(value => value - 1);
  }
}
```

การใช้งานใน parent component:

```typescript
@Component({
  selector: 'app-parent',
  template: `
    <app-counter [(count)]="parentCount" />
    <div>Parent count: {{ parentCount() }}</div>
  `
})
export class ParentComponent {
  parentCount = signal(5);
}
```

## การสร้าง Custom Operators

คุณสามารถสร้าง operators ของตัวเองเพื่อใช้งานกับ signals ได้

### ตัวอย่าง Custom Operator: `debounce`

```typescript
import { signal, type WritableSignal, type Signal, type SettableSignal } from '@angular/core';

// Custom operator สำหรับ debounce signal
export function debounce<T>(
  source: Signal<T>, 
  delay: number
): Signal<T> {
  // สร้าง signal ใหม่ที่จะ debounce ค่า
  const debouncedSignal = signal<T>(source());
  let timeoutId: any = null;
  
  // สร้าง effect ที่จะติดตามการเปลี่ยนแปลงของ source signal
  effect(() => {
    // อ่านค่าปัจจุบันของ source signal
    const currentValue = source();
    
    // ยกเลิก timeout เดิม (ถ้ามี)
    if (timeoutId !== null) {
      clearTimeout(timeoutId);
    }
    
    // ตั้ง timeout ใหม่
    timeoutId = setTimeout(() => {
      // อัพเดต debounced signal หลังจากรอ delay
      debouncedSignal.set(currentValue);
      timeoutId = null;
    }, delay);
  });
  
  return debouncedSignal;
}

// ตัวอย่างการใช้งาน
const searchQuery = signal('');
const debouncedSearch = debounce(searchQuery, 300);

// เมื่อมีการเปลี่ยนแปลง searchQuery, debouncedSearch จะอัพเดตหลังจากรอ 300ms
searchQuery.set('Angular');
// debouncedSearch() จะอัพเดตเป็น 'Angular' หลังจาก 300ms
```

### ตัวอย่าง Custom Operator: `distinct`

```typescript
import { signal, type Signal } from '@angular/core';

// Custom operator สำหรับกรองค่าที่ซ้ำ
export function distinct<T>(source: Signal<T>): Signal<T> {
  const result = signal<T>(source());
  let lastValue = source();
  
  effect(() => {
    const currentValue = source();
    // อัพเดตเฉพาะเมื่อค่าเปลี่ยนแปลง
    if (currentValue !== lastValue) {
      result.set(currentValue);
      lastValue = currentValue;
    }
  });
  
  return result;
}

// ตัวอย่างการใช้งาน
const counter = signal(0);
const distinctCounter = distinct(counter);

// เมื่อ set ค่าเดิม จะไม่เกิดการอัพเดตใน distinctCounter
counter.set(0); // ไม่มีการอัพเดต distinctCounter
counter.set(1); // distinctCounter อัพเดตเป็น 1
counter.set(1); // ไม่มีการอัพเดต distinctCounter 
```

## ตัวอย่างการใช้งาน Operators

### ตัวอย่าง: การใช้ `mutate` กับอาร์เรย์

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-todo-list',
  template: `
    <ul>
      <li *ngFor="let todo of todos()">
        <input
          type="checkbox"
          [checked]="todo.completed"
          (change)="toggleTodo(todo.id)"
        />
        {{ todo.title }}
      </li>
    </ul>
  `
})
export class TodoListComponent {
  todos = signal([
    { id: 1, title: 'Learn Angular', completed: false },
    { id: 2, title: 'Build an app', completed: false }
  ]);
  
  toggleTodo(id: number) {
    // ใช้ mutate เพื่อแก้ไขค่าโดยตรง
    this.todos.mutate(todos => {
      const todo = todos.find(t => t.id === id);
      if (todo) {
        todo.completed = !todo.completed;
      }
    });
  }
}
```

### ตัวอย่าง: การใช้ `toSignal` และ `toObservable` ร่วมกับ HTTP

```typescript
import { Component, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { switchMap, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-details',
  template: `
    <div>
      <h2>User Details</h2>
      <input
        type="number"
        [value]="userId()"
        (input)="onUserIdChange($event)"
        placeholder="Enter user ID"
      />
      
      <div *ngIf="userLoading()">Loading...</div>
      <div *ngIf="userError()">{{ userError() }}</div>
      
      <div *ngIf="user() && !userLoading()">
        <h3>{{ user()?.name }}</h3>
        <p>Email: {{ user()?.email }}</p>
      </div>
    </div>
  `
})
export class UserDetailsComponent {
  private http = inject(HttpClient);
  
  // ใช้ Signal เก็บค่า userId ที่ผู้ใช้ป้อน
  userId = signal<number>(1);
  
  // แปลง Signal เป็น Observable
  private userId$ = toObservable(this.userId);
  ด
  // ใช้ userId$ เพื่อดึงข้อมูลผู้ใช้
  private user$ = this.userId$.pipe(
    switchMap(id => this.http.get<User>(`https://jsonplaceholder.typicode.com/users/${id}`).pipe(
      catchError(error => {
        console.error('Error fetching user:', error);
        return of(null);
      })
    ))
  );
  
  // สร้าง Signal สำหรับสถานะต่างๆ
  userLoading = signal(false);
  userError = signal<string | null>(null);
  
  // แปลง Observable กลับเป็น Signal
  user = toSignal<User | null>(this.user$, { initialValue: null });
  
  // ติดตามการเปลี่ยนแปลงของ userId
  constructor() {
    effect(() => {
      this.userLoading.set(true);
      this.userError.set(null);
      
      // userId เปลี่ยน -> user$ จะทำงาน -> user signal จะอัพเดต
    });
    
    // ติดตามผลลัพธ์ของ user
    effect(() => {
      const currentUser = this.user();
      this.userLoading.set(false);
      
      if (currentUser === null) {
        this.userError.set(`ไม่พบผู้ใช้ที่มี ID: ${this.userId()}`);
      }
    });
  }
  
  onUserIdChange(event: Event) {
    const input = event.target as HTMLInputElement;
    const id = parseInt(input.value, 10) || 1;
    this.userId.set(id);
  }
}
```

### ตัวอย่าง: การใช้ Signal Input Operators ใน Form Component

```typescript
import { Component, input, model } from '@angular/core';

@Component({
  selector: 'app-input-field',
  template: `
    <div>
      <label [for]="id()">{{ label() }}</label>
      <input
        [id]="id()"
        [type]="type()"
        [value]="value()"
        [placeholder]="placeholder()"
        (input)="onInput($event)"
        [required]="required()"
      />
      <small *ngIf="error()">{{ error() }}</small>
    </div>
  `
})
export class InputFieldComponent {
  // Signal Inputs
  id = input.required<string>(); // required input
  label = input<string>(''); // optional input
  type = input<string>('text'); // input with default value
  placeholder = input<string>(''); // optional input
  required = input<boolean>(false); // boolean input with default value
  
  // Error message input with transform
  error = input<string | null, string | null>(null, {
    transform: (value: string | null) => value ? `Error: ${value}` : null
  });
  
  // Two-way binding with model
  value = model<string>('');
  
  onInput(event: Event) {
    const input = event.target as HTMLInputElement;
    this.value.set(input.value);
  }
}
```

### การใช้งานของ Component ข้างต้น:

```typescript
@Component({
  selector: 'app-user-form',
  template: `
    <form>
      <app-input-field
        id="username"
        label="Username"
        placeholder="Enter your username"
        [(value)]="username"
        [required]="true"
      ></app-input-field>
      
      <app-input-field
        id="email"
        label="Email"
        type="email"
        placeholder="Enter your email"
        [(value)]="email"
        [required]="true"
        [error]="emailError()"
      ></app-input-field>
      
      <p>Form values: {{ username() }}, {{ email() }}</p>
    </form>
  `
})
export class UserFormComponent {
  username = signal('');
  email = signal('');
  emailError = computed(() => {
    // ตรวจสอบรูปแบบอีเมล
    const emailValue = this.email();
    if (emailValue && !emailValue.includes('@')) {
      return 'Invalid email format';
    }
    return null;
  });
}
```

## สรุป

Operators ของ Angular Signals ช่วยให้เราสามารถจัดการกับ reactive state ได้อย่างมีประสิทธิภาพ ไม่ว่าจะเป็น:

1. **Operators พื้นฐาน**: `set()`, `update()`, `mutate()`, `asReadonly()`
2. **RxJS Interop**: `toSignal()`, `toObservable()`, `fromEvent()`
3. **Signal Inputs**: `input()`, `model()`
4. **Custom Operators**: สร้าง operators เองเพื่อการใช้งานเฉพาะทาง

การใช้ operators เหล่านี้ช่วยให้การสร้าง reactive applications ด้วย Angular มีความยืดหยุ่นและมีประสิทธิภาพมากขึ้น โดยเฉพาะเมื่อต้องการจัดการกับสถานะที่ซับซ้อนหรือต้องการการอัพเดต UI ที่ละเอียดและรวดเร็ว