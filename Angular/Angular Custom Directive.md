---
tags:
  - angular
  - typescript
  - directive
type: reference
created: 2026-09-16
---

# 🎯 Angular Custom Directive — Attribute และ Structural

> **Component ก็คือ directive ชนิดหนึ่งที่มี template ติดมาด้วย** — Angular แบ่ง directive เป็น 3 ชนิด: **Component** (มี template), **Attribute Directive** (เปลี่ยนพฤติกรรม/หน้าตาของ element ที่มีอยู่แล้ว เช่น `[ngClass]`), **Structural Directive** (ควบคุมว่า element จะถูกสร้าง/ลบใน DOM ไหม เช่น `*ngIf`/`*ngFor`)

---

## 1. Attribute Directive — เพิ่มพฤติกรรมให้ element เดิม

```typescript
import { Directive, ElementRef, Renderer2, HostListener, input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true,
})
export class HighlightDirective {
  color = input('yellow', { alias: 'appHighlight' });   // [appHighlight]="'red'" ตั้งค่าสีเองได้

  constructor(private el: ElementRef, private renderer: Renderer2) {}

  @HostListener('mouseenter')
  onMouseEnter() {
    this.renderer.setStyle(this.el.nativeElement, 'backgroundColor', this.color());
  }

  @HostListener('mouseleave')
  onMouseLeave() {
    this.renderer.removeStyle(this.el.nativeElement, 'backgroundColor');
  }
}
```

```html
<p appHighlight>ชี้เมาส์แล้วไฮไลต์สีเหลือง</p>
<p [appHighlight]="'lightblue'">กำหนดสีเอง</p>
```

### ⚠️ ใช้ `Renderer2` แทนการแก้ DOM ผ่าน `ElementRef` ตรง ๆ

```typescript
// ❌ เข้าถึง DOM ตรง ๆ — ไม่ปลอดภัยกับ SSR, เสี่ยง XSS
this.el.nativeElement.style.backgroundColor = this.color();

// ✅ ผ่าน Renderer2 — ปลอดภัยกว่า, ทำงานถูกต้องทั้งบน browser และ server-side rendering
this.renderer.setStyle(this.el.nativeElement, 'backgroundColor', this.color());
```

**`Renderer2` เป็นชั้นกลางระหว่างโค้ดกับ DOM จริง** — Angular render บน platform ไหนก็ได้ (browser, server สำหรับ SSR, web worker) โดยไม่ต้องแก้โค้ด directive เลย ถ้าใช้ `nativeElement` ตรง ๆ จะพังทันทีบน environment ที่ไม่มี DOM จริง (เช่น SSR)

---

## 2. Structural Directive — ควบคุมว่า element มีอยู่ใน DOM ไหม

### `*ngIf` จริง ๆ แล้วคืออะไร

```html
<p *ngIf="show">ข้อความ</p>
```

**`*` คือ syntax ย่อ (microsyntax) ที่แปลงเป็นแบบเต็มนี้อัตโนมัติ:**

```html
<ng-template [ngIf]="show">
  <p>ข้อความ</p>
</ng-template>
```

`<ng-template>` ไม่ render อะไรเลยจนกว่า directive จะสั่งให้ — structural directive คือตัวตัดสินใจว่าจะ "แกะ" เนื้อหาใน template นั้นออกมาใส่ DOM จริงหรือไม่

### เขียนเอง — `*appUnless` (ตรงข้ามกับ `*ngIf`)

```typescript
import { Directive, TemplateRef, ViewContainerRef, input, effect } from '@angular/core';

@Directive({
  selector: '[appUnless]',
  standalone: true,
})
export class UnlessDirective {
  private hasView = false;
  condition = input.required<boolean>({ alias: 'appUnless' });

  constructor(
    private templateRef: TemplateRef<unknown>,
    private viewContainer: ViewContainerRef,
  ) {
    effect(() => {
      if (!this.condition() && !this.hasView) {
        this.viewContainer.createEmbeddedView(this.templateRef);   // แสดง
        this.hasView = true;
      } else if (this.condition() && this.hasView) {
        this.viewContainer.clear();                                 // ลบออกจาก DOM จริง ๆ
        this.hasView = false;
      }
    });
  }
}
```

```html
<p *appUnless="isLoading">แสดงเมื่อ isLoading เป็น false</p>
```

| | ทำหน้าที่อะไร |
|---|---|
| `TemplateRef` | อ้างอิงถึงเนื้อหาใน `<ng-template>` ที่ directive แปะอยู่ — ยังไม่ render จนกว่าจะสั่ง |
| `ViewContainerRef` | ตำแหน่งใน DOM ที่ directive ใช้ "แปะ" เนื้อหาลงไปจริง |
| `createEmbeddedView()` | สั่ง render template นั้นลง DOM |
| `clear()` | ลบ view ที่เคย render ออกจาก DOM (ไม่ใช่แค่ซ่อนด้วย CSS — ลบทิ้งจริง ต่างจาก `[hidden]`) |

**ข้อแตกต่างสำคัญจาก attribute directive:** structural directive ควบคุม **การมีอยู่** ของ element ใน DOM (สร้าง/ทำลาย) ส่วน attribute directive แค่ปรับพฤติกรรมของ element ที่**มีอยู่แล้ว**เสมอ

---

## 3. กับดัก

- **แก้ DOM ผ่าน `ElementRef.nativeElement` ตรง ๆ แทน `Renderer2`** — พังบน SSR และเสี่ยง XSS มากกว่า (ข้อ 1)
- **ลืม `standalone: true`** — directive ใช้กับ standalone component ไม่ได้ถ้าไม่ได้ mark หรือไม่ได้ import
- **structural directive ไม่เช็ค `hasView` ก่อนเรียก `createEmbeddedView()` ซ้ำ** — ได้ element ซ้ำกันหลายชุดใน DOM โดยไม่ตั้งใจ
- **ใช้ `[hidden]`/`display: none` แทน structural directive ทั้งที่ต้องการเอา element ออกจาก DOM จริง ๆ** — `[hidden]` แค่ซ่อนด้วย CSS แต่ยัง render อยู่ (component lifecycle ยังทำงาน, DOM ยังหนักอยู่) ถ้าต้องการลบออกจริงต้องใช้ structural directive
- **ลืมว่า `*` เป็นแค่ syntax sugar** — เขียน `*ngIf` ควบคู่กับ directive ตัวอื่นบน element เดียวกันแล้วงงว่าทำไม syntax ผิด (ใช้ `<ng-container>` ห่อแยกชั้นแทน)

---

## 4. Cheat sheet

```typescript
// Attribute directive
@Directive({ selector: '[appX]', standalone: true })
export class XDirective {
  @HostListener('event') onEvent() { ... }
  constructor(private el: ElementRef, private renderer: Renderer2) {}
}

// Structural directive
@Directive({ selector: '[appX]', standalone: true })
export class XDirective {
  constructor(private tpl: TemplateRef<unknown>, private vcr: ViewContainerRef) {}
  // vcr.createEmbeddedView(tpl) / vcr.clear()
}
```

| อาการ | สาเหตุ |
|---|---|
| directive พังตอน SSR | ใช้ `nativeElement` ตรง ๆ แทน `Renderer2` |
| directive ใช้กับ standalone component ไม่ได้ | ลืม `standalone: true`/import |
| element render ซ้ำหลายชุด | structural directive ไม่เช็ค `hasView` ก่อนสร้าง view ใหม่ |
| component ข้างใน `*ngIf`/directive เอง ยังทำงานทั้งที่ "ซ่อน" อยู่ | ใช้ `[hidden]` แทนที่จะใช้ structural directive จริง ๆ |

## 🔗 เกี่ยวข้อง

- [[Angular Signal Primitives]] — `effect()` ที่ใช้เชื่อม signal เข้ากับ `ViewContainerRef` ในตัวอย่าง structural directive
- [[Angular Component Communication]] — `input()` ที่ใช้ตั้งค่า directive จากภายนอก

## 📖 อ่านต่อ

- [Angular — Attribute directives](https://angular.dev/guide/directives/attribute-directives)
- [Angular — Structural directives](https://angular.dev/guide/directives/structural-directives)
