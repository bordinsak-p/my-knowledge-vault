const { firstNameTh: fname = '', lastNameTh: lname = '', idCard = '' } = this.extraUserInfo ?? {}; หมายความว่ายังไง

มาดูทีละจุดเลยนะ อันนี้คือ JavaScript/TypeScript syntax แบบ **"destructuring + default value"** ใช้บ่อยมากเวลาเราทำกับ object ที่อาจจะว่าง (null/undefined)

```ts
const { firstNameTh: fname = '', lastNameTh: lname = '', idCard = '' } = this.extraUserInfo ?? {};
```

ดึงค่า `firstNameTh`, `lastNameTh`, และ `idCard` จาก `this.extraUserInfo`  
ถ้า `this.extraUserInfo` เป็น null/undefined ให้ใช้ `{}` แทน  
จากนั้น map ค่า `firstNameTh` → ตัวแปร `fname`  
และกำหนด default = '' ถ้าไม่มีค่า"

สรุปสั้น ๆ:

| ตัวแปร   | ได้จาก field                | ถ้าไม่มีค่า   |
| -------- | --------------------------- | ------------- |
| `fname`  | `extraUserInfo.firstNameTh` | กลายเป็น `''` |
| `lname`  | `extraUserInfo.lastNameTh`  | กลายเป็น `''` |
| `idCard` | `extraUserInfo.idCard`      | กลายเป็น `''` |
## ✅ ทำไมถึงดี?

1. 🔒 **ปลอดภัยจาก null/undefined**  
    ใช้ `?? {}` กันไว้ก่อนว่าถ้า `extraUserInfo` ไม่มี ก็อย่าให้ code พัง
    
2. 🧹 **Assign ชื่อใหม่ให้อ่านง่าย**  
    ไม่ต้องใช้ `this.extraUserInfo.firstNameTh` ยาว ๆ
    
3. 🪄 **มี default ให้เสมอ**  
    ใช้ `' '` กัน error เวลาจะ render บนหน้าจอ หรือส่งค่าไป backend


```ts
this.extraUserInfo = {
  firstNameTh: 'Mean',
  lastNameTh: 'Dev',
  // idCard: ไม่มี
};

const { firstNameTh: fname = '', lastNameTh: lname = '', idCard = '' } = this.extraUserInfo ?? {};

console.log(fname); // "Mean"
console.log(lname); // "Dev"
console.log(idCard); // ""
```


## 🧠 Tip เสริม:

- ถ้าไม่อยากเปลี่ยนชื่อ ก็เขียนสั้นลงแบบนี้ได้:
```ts
const { firstNameTh = '', lastNameTh = '', idCard = '' } = this.extraUserInfo ?? {};
```

