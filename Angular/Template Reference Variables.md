#angular-outline/template-reference-variables
## การใช้เครื่องหมาย # (Template Reference Variables)

เครื่องหมาย `#` ใน Angular ใช้สำหรับสร้าง **Template Reference Variables** ซึ่งเป็นตัวแปรที่อ้างอิงไปยัง DOM element หรือ Angular component ภายใน template

```html
<p-checkbox
  label="ลูกจ้างพิการ"
  #disableFlag
  binary="true"
  formControlName="disableFlag"
  (onChange)="onChangeDisableFlag($event)"
></p-checkbox>
```

ในตัวอย่างนี้ `#disableFlag` สร้างตัวแปรชื่อ `disableFlag` ที่สามารถเข้าถึง `p-checkbox` component ได้

## วิธีการใช้งาน Template Reference Variables

### 1. อ้างอิงใน Template เดียวกัน

```html
<!-- Input field -->
<input #userInput type="text" placeholder="กรอกชื่อ">

<!-- ใช้งานตัวแปร userInput -->
<button (click)="showValue(userInput.value)">แสดงค่า</button>
<p>ค่าที่กรอก: {{ userInput.value }}</p>
```

### 2. ใช้กับ Form Elements

```html
<form #myForm="ngForm">
  <input name="username" ngModel required>
  <input name="email" ngModel required>
  
  <button [disabled]="!myForm.valid">Submit</button>
  <p>Form Status: {{ myForm.valid ? 'Valid' : 'Invalid' }}</p>
</form>
```

### 3. อ้างอิงไปยัง Component

```html
<!-- Component Reference -->
<app-user-profile #userProfile></app-user-profile>

<button (click)="userProfile.saveProfile()">บันทึกโปรไฟล์</button>
<button (click)="userProfile.resetForm()">รีเซ็ต</button>
```

### 4. ใช้ใน Component Class

```typescript
// Component
export class MyComponent {
  @ViewChild('disableFlag') disableFlagRef!: ElementRef;
  @ViewChild('userInput') userInputRef!: ElementRef;

  ngAfterViewInit() {
    // เข้าถึง element ผ่าน ViewChild
    console.log(this.disableFlagRef.nativeElement);
    this.userInputRef.nativeElement.focus();
  }

  clearInput() {
    this.userInputRef.nativeElement.value = '';
  }
}
```

## เทคนิคและทริคการใช้งาน

### 1. การใช้กับ Structural Directives

```html
<div *ngFor="let item of items; let i = index">
  <input #itemInput type="text" [value]="item.name">
  <button (click)="updateItem(i, itemInput.value)">Update</button>
</div>
```

### 2. การส่งผ่าน Reference เป็น Parameter

```html
<input #searchInput type="text">
<button (click)="search(searchInput)">ค้นหา</button>

<!-- หรือส่งเฉพาะ value -->
<button (click)="search(searchInput.value)">ค้นหาเฉพาะค่า</button>
```

### 3. การใช้หลาย References

```html
<form #myForm="ngForm">
  <input #firstInput name="first" ngModel>
  <input #secondInput name="second" ngModel>
  
  <button (click)="compareInputs(firstInput.value, secondInput.value)">
    เปรียบเทียบ
  </button>
</form>
```

### 4. การใช้กับ Conditional Rendering

```html
<div *ngIf="showForm; else noForm">
  <input #conditionalInput type="text">
  <button (click)="processInput(conditionalInput.value)">Process</button>
</div>

<ng-template #noForm>
  <p>ไม่มีฟอร์ม</p>
</ng-template>
```

### 5. การใช้กับ Custom Components

```html
<!-- Parent Component -->
<app-child #childComponent 
           [data]="parentData"
           (onSave)="handleSave($event)">
</app-child>

<button (click)="childComponent.triggerAction()">
  เรียกใช้ Method ของ Child
</button>
```

```typescript
// Child Component
@Component({
  selector: 'app-child',
  template: `<div>Child Component</div>`
})
export class ChildComponent {
  @Input() data: any;
  @Output() onSave = new EventEmitter();

  triggerAction() {
    console.log('Action triggered from parent!');
  }
}
```

## ข้อควรระวัง

### 1. Scope ของ Template Reference

- Template Reference Variables มี scope เฉพาะใน template เท่านั้น
- ไม่สามารถเข้าถึงจาก component class โดยตรง (ต้องใช้ @ViewChild)

### 2. Naming Convention

```html
<!-- ✅ Good -->
<input #userNameInput type="text">
<form #registrationForm="ngForm">

<!-- ❌ Avoid -->
<input #x type="text">
<form #f="ngForm">
```

### 3. การใช้กับ *ngFor

```html
<!-- Template Reference ใน ngFor จะสร้างใหม่ทุกครั้งที่ loop -->
<div *ngFor="let item of items">
  <input #itemRef type="text">
  <!-- itemRef จะแตกต่างกันในแต่ละ iteration -->
</div>
```

## Best Practices

1. **ใช้ชื่อที่มีความหมาย**: `#userInput` ดีกว่า `#input`
2. **รวมกับ @ViewChild**: สำหรับการเข้าถึงใน component class
3. **ระวัง Memory Leaks**: หลีกเลี่ยงการเก็บ reference ไว้นานเกินไป
4. **ใช้กับ Form Validation**: เหมาะสำหรับการตรวจสอบ form แบบ real-time

```html
<!-- Example: Real-time Validation -->
<input #emailInput 
       type="email" 
       (blur)="validateEmail(emailInput.value)"
       [class.invalid]="!isValidEmail(emailInput.value)">
       
<div *ngIf="!isValidEmail(emailInput.value)" class="error">
  กรุณากรอกอีเมลให้ถูกต้อง
</div>
```

Template Reference Variables เป็นเครื่องมือที่ทรงพลังใน Angular ที่ช่วยให้เราสามารถเข้าถึงและควบคุม DOM elements และ components ได้อย่างยืดหยุ่นและมีประสิทธิภาพ