# Push (การเพิ่มเข้าหัว)

งั้นมาเขียนการเพิ่มค่าลงในลิสต์กัน `push` *กลายพันธุ์*ลิสต์
เราจึงต้องรับ `&mut self` เรายังต้องรับ i32 ตัวหนึ่งเพื่อเพิ่มเข้าไปด้วย:

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

ก่อนอื่นเลย เราต้องสร้างโหนดเพื่อเก็บองค์ประกอบของเรา:

```rust ,ignore
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

อะไรจะไปอยู่ที่ `next`? ก็ลิสต์เก่าทั้งหมดไง! เราทำแบบนั้น... ได้เลยมั้ย?

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

ม่ายยยย. Rust บอกสิ่งที่ถูกต้อง แต่แน่นอนว่ามันไม่ชัดเจนว่ามันหมายความว่าอะไรกันแน่
หรือต้องทำยังไงกับมัน:

> cannot move out of borrowed content

เรากำลังพยายามย้ายฟิลด์ `self.head` ออกไปไว้ที่ `next` แต่ Rust ไม่ต้องการให้เราทำแบบนั้น
นี่จะทำให้ `self` เหลือการเริ่มต้นเพียงบางส่วนเมื่อเราสิ้นสุดการยืมและ "คืนให้"
เจ้าของที่ถูกต้องของมัน อย่างที่เราพูดไว้ก่อนหน้านี้ นั่นคือ*สิ่งเดียว*ที่คุณทำไม่ได้กับ `&mut`:
มันจะหยาบคายมาก และ Rust เองก็สุภาพมาก (มันจะอันตรายสุดขีดด้วย แต่แน่นอนว่า
*นั่น*คงไม่ใช่เหตุผลที่มันแคร์)

แล้วถ้าเราเอาอะไรกลับเข้าไปแทนล่ะ? อย่างเช่น โหนดที่เรากำลังสร้างอยู่นี้:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

ไม่ได้ผล ในหลักการ นี่เป็นสิ่งที่ Rust ยอมรับได้จริง ๆ แต่มันไม่ยอม (ด้วยเหตุผลหลายประการ
-- ที่ร้ายแรงที่สุดคือ[ความปลอดภัยของข้อยกเว้น (exception safety)][exception safety])
เราต้องการวิธีเอาหัวออกมาโดยที่ Rust ไม่สังเกตว่ามันหายไป สำหรับคำแนะนำ
เราหันไปหาฮีโร่แฮ็กเกอร์ Rust ผู้โด่งดัง อินเดียน่า โจนส์:

![Indy Prepares to mem::replace](img/indy.gif)

อา ใช่แล้ว Indy แนะนำกลยุทธ์ `mem::replace` ฟังก์ชันที่มีประโยชน์อย่างไม่น่าเชื่อนี้
ให้เราขโมยค่าออกจาก borrow ได้โดยการ*แทนที่*มันด้วยค่าอื่น
มาลองดึง `std::mem` เข้ามาที่ด้านบนของไฟล์ เพื่อให้ `mem` อยู่ในสโคปท้องถิ่น:

```rust ,ignore
use std::mem;
```

และใช้มันอย่างเหมาะสม:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

ตรงนี้เราทำการ `replace` self.head ด้วย Link::Empty ชั่วคราว ก่อนที่จะแทนที่มัน
ด้วยหัวใหม่ของลิสต์ ผมจะไม่โกหก: นี่เป็นสิ่งที่ต้องทำค่อนข้างน่าเศร้า โชคร้ายที่เราจำเป็นต้องทำ
(ตอนนี้)

แต่เฮ้ `push` เสร็จแล้ว! น่าจะเสร็จ ตอนนี้วิธีที่ง่ายที่สุดที่จะทดสอบมัน
ก็คือเขียน `pop` และตรวจสอบว่ามันให้ผลลัพธ์ที่ถูกต้อง

[exception safety]: https://doc.rust-lang.org/nightly/nomicon/exception-safety.html