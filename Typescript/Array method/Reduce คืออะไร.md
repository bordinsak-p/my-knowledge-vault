 คือฟังก์ชันใน JavaScript ที่ใช้ **"รวมค่าทุกตัวใน array" ให้เป็นค่าหนึ่งค่าที่เรากำหนดเองได้  
(จะรวมเป็นผลรวม, object, array ใหม่, group ฯลฯ ก็ได้หมด)**

```ts
array.reduce((accumulator, currentValue, index?, array?) => {
  // logic
  return accumulator;
}, initialValue);
```

- `accumulator` → ค่าที่สะสมไว้
- `currentValue` → ค่าแต่ละตัวใน array
- `initialValue` → ค่าเริ่มต้นของ `accumulator` อะไรก็ได้เช่น 0, [] (array ว่าง), { }

### ตัวอย่าง 1: รวมเลข

```ts
const numbers = [1, 2, 3, 4, 5];

const sum = numbers.reduce((acc, num) => {
  return acc + num;
}, 0);

console.log(sum); // 👉 15
```

- เริ่มต้น `acc = 0`
- วนทีละตัว → บวกเข้าไป