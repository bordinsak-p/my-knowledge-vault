#angular-outline/template

### ใน Angular การแสดงผลหรือโต้ตอบกับข้อมูลใน **Template** จะใช้ 3 รูปแบบหลัก ๆ คือ

#### 1. **Interpolation** — `{{ }}`  

#template/interpolation

ใช้เพื่อแสดงข้อมูลจาก **Component ไปยัง HTML (View)**

html 
```html
<p>Hello {{ username }}</p>
```

ts
```ts
username = 'ChatGPT';
```

- ใช้ได้เฉพาะ **ค่าที่เป็น string หรือ expression ที่เป็น read-only**
- เช่น `{{ 1 + 1 }}` หรือ `{{ user?.name }}` ก็ใช้ได้


---

#### 2. **Property Binding** — `[property]`

#template/property

ใช้เพื่อส่งค่าจาก **Component ไปยัง HTML element/property**

html
```html
<img [src]="imageUrl">
<button [disabled]="isDisabled">Submit</button>
```

ts
```ts
imageUrl = 'https://example.com/image.jpg';
isDisabled = true;
```

- ใช้กับ **attribute/property** ของ element เช่น `src`, `value`, `disabled`, `class`, `style` เป็นต้น
- หากเขียนผิด จะไม่มี error แต่ไม่ทำงาน


---

#### 3.**Event Binding** — `(event)`

ใช้เพื่อส่งค่าจาก **HTML กลับไปยัง Component** เมื่อมี event เกิดขึ้น

html
```html
<button (click)="onClick()">Click me</button>
<input (input)="onInputChange($event)">
```

ts
```ts
onClick() {
  console.log('Button clicked!');
}

onInputChange(event: Event) {
  const input = event.target as HTMLInputElement;
  console.log(input.value);
}
```

- ใช้กับ event ต่าง ๆ เช่น `click`, `change`, `input`, `keydown` เป็นต้น
- รับค่ากลับผ่าน `$event` ซึ่งเป็น native DOM event


---

