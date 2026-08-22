#template/dom-event

`$event` เป็น **ตัวแปรพิเศษใน Angular Template** ที่แทน **DOM event object** ที่ส่งมาจาก element นั้น ๆ

#### รูปแบบพื้นฐาน

html
```html
<input (input)="onInputChange($event)">
```

ts
```ts
onInputChange(event: Event) {
  const input = event.target as HTMLInputElement;
  console.log(input.value);
}
```

#### ตัวอย่าง $event กับ Event ต่าง ๆ

- **(click)**

html
```html
<button (click)="onClick($event)">Click me</button>
```

ts
```ts
onClick(event: MouseEvent) {
  console.log('X:', event.clientX, 'Y:', event.clientY);
}
```

- **(keyup.enter)**

html
```html
<input (keyup.enter)="onEnter($event)">
```

ts
```ts
onEnter(event: KeyboardEvent) {
  console.log('Enter pressed with value:', (event.target as HTMLInputElement).value);
}
```

- **(change) กับ select**

html
```html
<select (change)="onSelectChange($event)">
  <option value="a">A</option>
  <option value="b">B</option>
</select>
```

ts
```ts
onSelectChange(event: Event) {
  const value = (event.target as HTMLSelectElement).value;
  console.log('Selected:', value);
}
```

#### 🔥 เทคนิคเทพๆ ในการใช้ `$event`

#template/tricks

1. **ลด logic ใน component ด้วย template expression**

html
```html
<input (input)="value = $event.target.value">
<p>You typed: {{ value }}</p>
```
- ไม่ต้องมี method ใน component ก็ได้
- ใช้งานได้กับ logic ง่าย ๆ
---

2. **รับหลายค่าพร้อมกัน**

html
```html
<button (click)="doSomething(id, $event)">Click</button>
```

ts
```ts
doSomething(id: number, event: MouseEvent) {
  console.log('ID:', id, 'Mouse X:', event.clientX);
}
```
---

3. **ใช้ร่วมกับ `$event.target.name` สำหรับ form หลายช่อง**

html
```html
<input name="firstName" (input)="updateField($event)">
<input name="lastName" (input)="updateField($event)">
```

ts
```ts
updateField(event: Event) {
  const target = event.target as HTMLInputElement;
  this[target.name] = target.value;
}
```
- ใช้แบบ dynamic property name ได้
---

4. **ใช้กับ directive และ custom components**

html
```html
<app-my-input (customChange)="handleCustom($event)"></app-my-input>
```

ts
```ts
handleCustom(data: any) {
  console.log('Received from custom component:', data);
}
```
- `$event` ในกรณีนี้คือค่าที่ emit จาก `@Output()` ของ child component
---

5. **ใช้กับ animation event หรือ advanced event**

html
```html
<div (@fade.done)="onFadeDone($event)">...</div>
```
- ใช้ `$event` เพื่อจับ event ตอน animation จบ