#angular-outline/signal
## สิ่งที่คุณจะได้เรียนรู้

- Signals คืออะไรและทำไมจึงมีประโยชน์
- วิธีการสร้างและใช้งาน signals
- วิธีสร้าง computed signals
- การใช้ signals กับ NgFor และ NgIf
- วิธีการอัพเดต signals
- การใช้ effects เพื่อตอบสนองต่อการเปลี่ยนแปลงของ signals

## ภาพรวม

Signals เป็นระบบการเฝ้าติดตามการเปลี่ยนแปลงในแอปพลิเคชัน Angular ที่ช่วยให้คุณสามารถ:

- ติดตามการเปลี่ยนแปลงของค่าอย่างละเอียด
- อัพเดต DOM เฉพาะส่วนที่ได้รับผลกระทบจากการเปลี่ยนแปลงเท่านั้น
- สร้างฟังก์ชันที่ตอบสนองต่อการเปลี่ยนแปลงโดยอัตโนมัติ
- ปรับปรุงประสิทธิภาพของการเรนเดอร์ UI

Signal จะเก็บค่าไว้และมอบวิธีการอ่านและอัพเดตค่านั้นอย่างมีประสิทธิภาพ

## การใช้งาน Signals พื้นฐาน

### การสร้าง Signal

```typescript
import { signal } from '@angular/core';

// สร้าง signal ที่มีค่าเริ่มต้นเป็น 0
const count = signal(0);

// อ่านค่าจาก signal
console.log('ค่าปัจจุบัน:', count());

// อัพเดตค่าของ signal
count.set(1);

// อัพเดตค่าโดยอ้างอิงค่าปัจจุบัน
count.update(current => current + 1);
```

### Computed Signals

Computed signals คือ signals ที่คำนวณค่าจาก signals อื่นๆ โดยจะอัพเดตอัตโนมัติเมื่อ signals ที่อ้างอิงมีการเปลี่ยนแปลง:

```typescript
import { signal, computed } from '@angular/core';

const firstName = signal('John');
const lastName = signal('Doe');

// สร้าง computed signal
const fullName = computed(() => `${firstName()} ${lastName()}`);

console.log(fullName()); // "John Doe"

// เมื่อ firstName เปลี่ยน fullName จะอัพเดตโดยอัตโนมัติ
firstName.set('Jane');
console.log(fullName()); // "Jane Doe"
```

## การใช้ Signals ในเทมเพลต

สามารถใช้ signals ในเทมเพลต Angular ได้โดยตรง:

```html
<p>Count: {{ count() }}</p>
<button (click)="increment()">เพิ่ม</button>
```

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  templateUrl: './counter.component.html',
})
export class CounterComponent {
  count = signal(0);
  
  increment() {
    this.count.update(current => current + 1);
  }
}
```

### การใช้กับ ngFor และ ngIf

```html
<div *ngIf="showList()">
  <ul>
    <li *ngFor="let item of items()">{{ item }}</li>
  </ul>
</div>
```

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-list',
  templateUrl: './list.component.html',
})
export class ListComponent {
  items = signal(['Item 1', 'Item 2', 'Item 3']);
  showList = signal(true);
   
  toggleList() {
    this.showList.update(current => !current);
  }
  
  addItem(newItem: string) {
    this.items.update(currentItems => [...currentItems, newItem]);
  }
}
```

## Effects

Effects ใช้เพื่อดำเนินการเมื่อ signals มีการเปลี่ยนแปลง:

```typescript
import { Component, signal, effect } from '@angular/core';

@Component({
  selector: 'app-logger',
  template: '<button (click)="increment()">เพิ่ม</button>',
})
export class LoggerComponent {
  count = signal(0);
  
  constructor() {
    // สร้าง effect ที่ทำงานทุกครั้งที่ count เปลี่ยนแปลง
    effect(() => {
      console.log(`ค่า count เปลี่ยนเป็น: ${this.count()}`);
    });
  }
  
  increment() {
    this.count.update(c => c + 1);
  }
}
```

## ข้อดีของ Signals

1. **ประสิทธิภาพที่ดีขึ้น**: อัพเดตเฉพาะส่วนของ DOM ที่ได้รับผลกระทบจากการเปลี่ยนแปลง
2. **ความละเอียดในการติดตาม**: ติดตามการเปลี่ยนแปลงในระดับที่เล็กกว่าการติดตามแบบเดิม
3. **การบำรุงรักษาที่ง่ายขึ้น**: ความสัมพันธ์ระหว่างข้อมูลและ UI ชัดเจนขึ้น
4. **การ debug ที่ง่ายขึ้น**: เข้าใจได้ง่ายว่าอะไรทำให้เกิดการอัพเดต
5. **ความยืดหยุ่น**: ทำงานได้ดีกับการจัดการสถานะทั้งในและนอก Zone.js

## อัพเดต Objects และ Arrays ใน Signals

เนื่องจาก signals ตรวจจับการเปลี่ยนแปลงโดยเปรียบเทียบการอ้างอิง (reference) คุณต้องสร้างอ็อบเจกต์หรืออาร์เรย์ใหม่เมื่อต้องการอัพเดต:

```typescript
// อ็อบเจกต์ใน signal
const user = signal({ name: 'John', age: 25 });

// วิธีที่ถูกต้อง - สร้างอ็อบเจกต์ใหม่
user.update(currentUser => ({ ...currentUser, age: 26 }));

// อาร์เรย์ใน signal
const numbers = signal([1, 2, 3]);

// วิธีที่ถูกต้อง - สร้างอาร์เรย์ใหม่
numbers.update(currentNumbers => [...currentNumbers, 4]);
```

## ตัวอย่างการใช้งานจริง

ต่อไปนี้เป็นตัวอย่างการใช้งาน Signal ในแอปพลิเคชัน Todo List อย่างง่าย

### 1. โครงสร้างโปรเจค

```
todo-app/
├── src/
│   ├── app/
│   │   ├── todo-list/
│   │   │   ├── todo-list.component.ts
│   │   │   ├── todo-list.component.html
│   │   │   └── todo.model.ts
│   │   ├── app.component.ts
│   │   └── app.module.ts
```

### 2. สร้างโมเดล Todo

```typescript
// todo.model.ts
export interface Todo {
  id: number;
  title: string;
  completed: boolean;
}
```

### 3. สร้าง Todo List Component

```typescript
// todo-list.component.ts
import { Component, signal, computed, effect } from '@angular/core';
import { Todo } from './todo.model';

@Component({
  selector: 'app-todo-list',
  templateUrl: './todo-list.component.html',
})
export class TodoListComponent {
  // สร้าง signal สำหรับเก็บรายการ todos
  todos = signal<Todo[]>([
    { id: 1, title: 'เรียนรู้ Angular Signals', completed: false },
    { id: 2, title: 'สร้างแอป Todo', completed: false },
    { id: 3, title: 'ทำความสะอาดบ้าน', completed: true },
  ]);

  // สร้าง signal สำหรับเก็บข้อความที่กำลังพิมพ์
  newTodoText = signal('');

  // สร้าง computed signal สำหรับนับจำนวน todos ที่ยังไม่เสร็จ
  remainingTodos = computed(() => 
    this.todos().filter(todo => !todo.completed).length
  );

  // สร้าง computed signal สำหรับนับจำนวน todos ทั้งหมด
  totalTodos = computed(() => this.todos().length);

  // สร้าง computed signal สำหรับคำนวณความคืบหน้า
  completionRate = computed(() => {
    const total = this.totalTodos();
    if (total === 0) return 0;
    return Math.round(
      ((total - this.remainingTodos()) / total) * 100
    );
  });

  constructor() {
    // สร้าง effect เพื่อบันทึกลงใน localStorage เมื่อ todos เปลี่ยนแปลง
    effect(() => {
      localStorage.setItem('todos', JSON.stringify(this.todos()));
      console.log('บันทึก todos ลงใน localStorage แล้ว');
    });
  }

  // โหลดข้อมูลจาก localStorage เมื่อ component เริ่มทำงาน
  ngOnInit() {
    const savedTodos = localStorage.getItem('todos');
    if (savedTodos) {
      this.todos.set(JSON.parse(savedTodos));
    }
  }

  // เพิ่ม todo ใหม่
  addTodo() {
    if (!this.newTodoText().trim()) return;
    
    // สร้าง id ใหม่โดยใช้ค่าสูงสุดที่มีอยู่ + 1
    const newId = Math.max(0, ...this.todos().map(t => t.id)) + 1;
    
    // อัพเดต todos โดยสร้างอาร์เรย์ใหม่
    this.todos.update(currentTodos => [
      ...currentTodos,
      {
        id: newId,
        title: this.newTodoText().trim(),
        completed: false
      }
    ]);
    
    // รีเซ็ตข้อความหลังจากเพิ่ม todo แล้ว
    this.newTodoText.set('');
  }

  // ลบ todo
  deleteTodo(id: number) {
    this.todos.update(currentTodos => 
      currentTodos.filter(todo => todo.id !== id)
    );
  }

  // สลับสถานะ completed ของ todo
  toggleTodo(id: number) {
    this.todos.update(currentTodos => 
      currentTodos.map(todo => 
        todo.id === id 
          ? { ...todo, completed: !todo.completed } 
          : todo
      )
    );
  }

  // อัพเดตข้อความที่กำลังพิมพ์
  updateNewTodoText(event: Event) {
    const input = event.target as HTMLInputElement;
    this.newTodoText.set(input.value);
  }

  // เคลียร์ todos ที่เสร็จแล้วทั้งหมด
  clearCompleted() {
    this.todos.update(currentTodos => 
      currentTodos.filter(todo => !todo.completed)
    );
  }
}
```

### 4. สร้าง Template สำหรับ Todo List Component

```html
<!-- todo-list.component.html -->
<div class="todo-container">
  <h1>Todo List ด้วย Angular Signals</h1>
  
  <!-- แสดงความคืบหน้า -->
  <div class="progress">
    <div class="progress-bar" [style.width.%]="completionRate()"></div>
    <span>{{ completionRate() }}% เสร็จสิ้น</span>
  </div>
  
  <!-- ฟอร์มสำหรับเพิ่ม todo ใหม่ -->
  <div class="add-todo">
    <input 
      type="text" 
      placeholder="เพิ่มงานใหม่..." 
      [value]="newTodoText()" 
      (input)="updateNewTodoText($event)"
      (keyup.enter)="addTodo()"
    />
    <button (click)="addTodo()">เพิ่ม</button>
  </div>
  
  <!-- แสดงรายการ todos -->
  <ul class="todo-list">
    <li *ngFor="let todo of todos()" [class.completed]="todo.completed">
      <input 
        type="checkbox" 
        [checked]="todo.completed" 
        (change)="toggleTodo(todo.id)"
      />
      <span class="todo-text">{{ todo.title }}</span>
      <button class="delete" (click)="deleteTodo(todo.id)">ลบ</button>
    </li>
  </ul>
  
  <!-- แสดงข้อมูลสถิติและปุ่มเคลียร์ -->
  <div class="todo-footer" *ngIf="totalTodos() > 0">
    <span>{{ remainingTodos() }} รายการที่เหลือจาก {{ totalTodos() }}</span>
    <button 
      *ngIf="totalTodos() > remainingTodos()" 
      (click)="clearCompleted()"
    >
      เคลียร์รายการที่เสร็จแล้ว
    </button>
  </div>
</div>
```

### 5. สไตล์ CSS (เพิ่มเติม)

```css
/* styles.css หรือใน component styles */
.todo-container {
  max-width: 500px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

.progress {
  height: 20px;
  background-color: #f0f0f0;
  border-radius: 10px;
  margin-bottom: 20px;
  position: relative;
}

.progress-bar {
  height: 100%;
  background-color: #4caf50;
  border-radius: 10px;
  transition: width 0.3s;
}

.progress span {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  font-size: 12px;
}

.add-todo {
  display: flex;
  margin-bottom: 20px;
}

.add-todo input {
  flex: 1;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px 0 0 4px;
}

.add-todo button {
  padding: 8px 15px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 0 4px 4px 0;
  cursor: pointer;
}

.todo-list {
  list-style-type: none;
  padding: 0;
}

.todo-list li {
  display: flex;
  align-items: center;
  padding: 10px;
  border-bottom: 1px solid #eee;
}

.todo-list li.completed .todo-text {
  text-decoration: line-through;
  color: #999;
}

.todo-text {
  flex: 1;
  margin: 0 10px;
}

.delete {
  background-color: #f44336;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 5px 10px;
  cursor: pointer;
}

.todo-footer {
  margin-top: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 14px;
}

.todo-footer button {
  background-color: #f0f0f0;
  border: none;
  padding: 5px 10px;
  border-radius: 4px;
  cursor: pointer;
}
```

## ตัวอย่างการใช้งาน Signal กับ HTTP

ต่อไปนี้เป็นตัวอย่างการใช้ Signals ร่วมกับ HTTP Client เพื่อดึงข้อมูลจาก API:

```typescript
import { Component, signal, computed, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-list',
  template: `
    <div>
      <h2>รายชื่อผู้ใช้</h2>
      
      <!-- แสดงสถานะการโหลด -->
      <div *ngIf="loading()">กำลังโหลดข้อมูล...</div>
      
      <!-- แสดงข้อผิดพลาด -->
      <div *ngIf="error()" class="error">
        เกิดข้อผิดพลาด: {{ error() }}
      </div>
      
      <!-- แสดงรายชื่อผู้ใช้ -->
      <ul *ngIf="users().length && !loading()">
        <li *ngFor="let user of users()">
          <strong>{{ user.name }}</strong>
          <p>{{ user.email }}</p>
        </li>
      </ul>
      
      <button (click)="loadUsers()" [disabled]="loading()">
        {{ users().length ? 'รีเฟรช' : 'โหลดข้อมูลผู้ใช้' }}
      </button>
    </div>
  `,
})
export class UserListComponent {
  private http = inject(HttpClient);
  
  // สร้าง signals
  users = signal<User[]>([]);
  loading = signal(false);
  error = signal<string | null>(null);
  
  // สร้าง computed signal
  hasUsers = computed(() => this.users().length > 0);
  
  constructor() {
    // โหลดข้อมูลผู้ใช้เมื่อ component เริ่มทำงาน
    this.loadUsers();
  }
  
  loadUsers() {
    // รีเซ็ตสถานะ
    this.loading.set(true);
    this.error.set(null);
    
    // เรียก API
    this.http.get<User[]>('https://jsonplaceholder.typicode.com/users')
      .subscribe({
        next: (data) => {
          // อัพเดต signal เมื่อได้รับข้อมูล
          this.users.set(data);
          this.loading.set(false);
        },
        error: (err) => {
          // อัพเดต signal เมื่อเกิดข้อผิดพลาด
          this.error.set(err.message || 'เกิดข้อผิดพลาดในการโหลดข้อมูล');
          this.loading.set(false);
        }
      });
  }
}
```

## สรุป

Signals เป็นวิธีใหม่ในการจัดการสถานะและการเชื่อมโยงข้อมูลใน Angular ที่ให้ประสิทธิภาพที่ดีขึ้นและรูปแบบการเขียนโค้ดที่ชัดเจน ด้วย signals คุณสามารถ:

- จัดการสถานะของแอปพลิเคชันได้อย่างมีประสิทธิภาพ
- สร้างความสัมพันธ์ระหว่างข้อมูลแบบอัตโนมัติด้วย computed signals
- ตอบสนองต่อการเปลี่ยนแปลงของข้อมูลด้วย effects
- ปรับปรุงประสิทธิภาพการเรนเดอร์โดยอัปเดตเฉพาะส่วนของ UI ที่ได้รับผลกระทบ