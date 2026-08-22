คือการ **ผูกข้อมูลจาก component ↔ template แบบสองทาง**  เมื่อค่าบนหน้าจอเปลี่ยน → ข้อมูลใน component ก็เปลี่ยน  เมื่อค่าจาก component เปลี่ยน → ข้อมูลบนหน้าจอก็เปลี่ยน

#### 1. Two-way Binding ด้วย `[(ngModel)]` (ใช้กับ Form/Input)

html
```html
<input [(ngModel)]="userName">
<p>ชื่อผู้ใช้: {{ userName }}</p>
```

ts
```ts
userName = 'สมชาย';
```
- ถ้าแก้ไขใน `<input>` → `userName` ใน component เปลี่ยนทันที
- ถ้าเปลี่ยนค่า `userName` → input ก็แสดงค่าที่เปลี่ยน

#### 2. Custom Two-way Binding (ระดับโปร)

ถ้าสร้าง **component ของตัวเอง** และอยากให้มันใช้ `[(value)]` ได้เหมือน `ngModel`

**Step-1: สร้าง component ลูก**

`custom-input` component

html
```html
<!-- custom-input.component.html -->
<input [value]="value" (input)="onInputChange($event.target.value)">
```

ts
```ts
// custom-input.component.ts
@Input() value: string = '';
@Output() valueChange = new EventEmitter<string>();

onInputChange(newValue: string) {
  this.value = newValue;
  this.valueChange.emit(this.value); // ใช้ $event เพื่อทำ two-way
}
```


**Step-2: ใช้งานใน Parent**

html
```html
<app-custom-input [(value)]="userName"></app-custom-input>
```

ts
```ts
userName = 'สมศรี';
```

