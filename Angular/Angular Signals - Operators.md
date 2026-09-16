  #sgnal/operator

Operators ใน Angular Signals เป็นเครื่องมือที่ใช้ในการจัดการและแปลงค่าของ signals แบบต่อเนื่อง (reactive streams) เพื่อให้สามารถดำเนินการกับข้อมูลที่เปลี่ยนแปลงได้อย่างมีประสิทธิภาพ

## ตารางเนื้อหา

- [Operators พื้นฐานของ Signal](https://angular.dev/guide/signals)
- [RxJS Interop - การใช้งานร่วมกับ RxJS](https://angular.dev/ecosystem/rxjs-interop)
- [Signal Input Operators](https://angular.dev/guide/signals/inputs)
- [การสร้าง Custom Operators](https://angular.dev/guide/signals)
- [ตัวอย่างการใช้งาน Operators](https://angular.dev/guide/signals)

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

### 3. ~~`mutate()`~~ — ถูกถอดออกจาก API แล้ว ใช้ `update()` แทน

> ⚠️ **`mutate()` ไม่มีอยู่ใน public API ของ Angular Signals อีกต่อไป** (ถูกถอดออกก่อน Angular 18 ออกตัวจริง) โค้ดที่เรียก `.mutate()` จะ **compile ไม่ผ่าน** — ใช้ `.update()` แทนเสมอ แม้กับ object/array ก็ทำแบบเดียวกัน แค่คืน**ค่าใหม่**แทนที่จะแก้ของเดิมตรง ๆ

```typescript
const user = signal({ name: 'John', age: 30 });

// ✅ update() — สร้าง object ใหม่ด้วย spread แทนการแก้ของเดิม
user.update(value => ({ ...value, age: 31 }));

// array ก็หลักการเดียวกัน
const items = signal<string[]>(['a', 'b']);
items.update(list => [...list, 'c']);          // เพิ่มสมาชิก
items.update(list => list.filter(x => x !== 'a'));  // ลบสมาชิก
```

**เหตุผลที่ถอดออก:** การแก้ object/array เดิมตรง ๆ (mutation) ตรวจจับการเปลี่ยนแปลงได้ยากกว่า โดยเฉพาะภายใต้ zoneless change detection — `update()` ที่คืนค่าใหม่เสมอทำให้ signal รู้แน่นอนว่าเปลี่ยนแล้วจริง (ตรวจด้วย `Object.is()` เทียบ reference)

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

> ⚠️ **แก้ไขแล้ว:** ตัวอย่างเดิมเรียก `effect()` ข้างในฟังก์ชันแยกที่ไม่ใช่ constructor/field initializer ของ class — ทำแบบนั้น `effect()` จะ**ไม่ได้อยู่ใน injection context** และ throw `NG0203` ทันที ต้องรับ `Injector` เข้ามาเอง (ผ่าน `inject(Injector)` ตอนถูกเรียกจาก injection context ของ caller หรือรับเป็นพารามิเตอร์ให้ส่งเองได้) แล้วส่งต่อให้ `effect()` ผ่าน option — รายละเอียดเต็ม ๆ ที่ [[Angular Signal Primitives]] ข้อ 4

```typescript
import { signal, effect, inject, Injector, type Signal } from '@angular/core';

// Custom operator สำหรับ debounce signal
// ต้องเรียกจาก field initializer/constructor ของ component/service เสมอ
// (หรือส่ง injector เข้ามาเองถ้าเรียกจากที่อื่น)
export function debounce<T>(source: Signal<T>, delay: number, injector?: Injector): Signal<T> {
  const activeInjector = injector ?? inject(Injector);   // ใช้ injection context ปัจจุบันถ้าไม่ได้ส่งมาเอง
  const debouncedSignal = signal<T>(source());

  effect((onCleanup) => {
    const currentValue = source();
    const timeoutId = setTimeout(() => debouncedSignal.set(currentValue), delay);
    onCleanup(() => clearTimeout(timeoutId));   // ยกเลิก timeout เดิมให้เองทุกรอบ ไม่ต้องเก็บตัวแปรนอก effect
  }, { injector: activeInjector });

  return debouncedSignal;
}
```

```typescript
// ✅ ตัวอย่างการใช้งานที่ถูกต้อง — เรียกตอน field initializer ของ component (อยู่ใน injection context)
export class SearchComponent {
  searchQuery = signal('');
  debouncedSearch = debounce(this.searchQuery, 300);

  onInput(value: string) {
    this.searchQuery.set(value);
    // debouncedSearch() จะอัพเดตเป็นค่าล่าสุดหลังจากหยุดพิมพ์ 300ms
  }
}
```

### ตัวอย่าง Custom Operator: `distinct`

> ⚠️ **แก้ไขแล้ว:** ปัญหาเดียวกับ `debounce` ข้างบน — `effect()` ต้องได้ `injector` ที่ถูกต้อง

```typescript
import { signal, effect, inject, Injector, type Signal } from '@angular/core';

// Custom operator สำหรับกรองค่าที่ซ้ำ
export function distinct<T>(source: Signal<T>, injector?: Injector): Signal<T> {
  const activeInjector = injector ?? inject(Injector);
  const result = signal<T>(source());
  let lastValue = source();

  effect(() => {
    const currentValue = source();
    // อัพเดตเฉพาะเมื่อค่าเปลี่ยนแปลง
    if (currentValue !== lastValue) {
      result.set(currentValue);
      lastValue = currentValue;
    }
  }, { injector: activeInjector });

  return result;
}
```

```typescript
// ✅ ตัวอย่างการใช้งานที่ถูกต้อง — เรียกตอน field initializer ของ component
export class CounterComponent {
  counter = signal(0);
  distinctCounter = distinct(this.counter);

  set(value: number) {
    this.counter.set(value);
    // set ค่าเดิมซ้ำ (เช่น 0 → 0) จะไม่ทำให้ distinctCounter อัพเดต
    // set ค่าใหม่ (เช่น 0 → 1) distinctCounter ถึงจะอัพเดตตาม
  }
}
```

> **หมายเหตุ:** `distinct` ใช้ `effect()` เพราะต้องจำค่าก่อนหน้า (`lastValue`) ข้ามการอ่านแต่ละรอบ ซึ่งเป็น state ที่ `computed()` (ต้อง pure ไม่มี state ของตัวเอง) ทำไม่ได้ — เป็นหนึ่งในไม่กี่กรณีที่ใช้ `effect()` เพื่อเชื่อม signal สองตัวได้อย่างสมเหตุสมผล ต่างจาก use case ทั่วไปที่ควรใช้ `computed()` แทน (ดู [[Angular Signal Primitives]] ข้อ 5)

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

1. **Operators พื้นฐาน**: `set()`, `update()`, `asReadonly()` (~~`mutate()`~~ ถูกถอดออกจาก API แล้ว)
2. **RxJS Interop**: `toSignal()`, `toObservable()`, `fromEvent()`
3. **Signal Inputs**: `input()`, `model()`
4. **Custom Operators**: สร้าง operators เองเพื่อการใช้งานเฉพาะทาง

การใช้ operators เหล่านี้ช่วยให้การสร้าง reactive applications ด้วย Angular มีความยืดหยุ่นและมีประสิทธิภาพมากขึ้น โดยเฉพาะเมื่อต้องการจัดการกับสถานะที่ซับซ้อนหรือต้องการการอัพเดต UI ที่ละเอียดและรวดเร็ว