# Pop (การนำออกจากหัว)

เหมือน `push`, `pop` ต้องการกลายพันธุ์ลิสต์ ต่างจาก `push` เราอยากจะคืนค่า
บางอย่างกลับไปจริง ๆ แต่ `pop` ก็ต้องจัดการกับกรณีมุมแหลมที่ยุ่งยากด้วย:
ถ้าลิสต์ว่างเปล่าล่ะ? เพื่อแทนกรณีนี้ เราจะใช้ชนิด `Option` ที่ไว้ใจได้:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
}
```

`Option<T>` คือ enum ที่แทนค่าที่อาจมีอยู่จริง มันสามารถเป็น `Some(T)` หรือ `None`
ก็ได้ เราสามารถสร้าง enum ของเราเองสำหรับสิ่งนี้เหมือนที่เราทำกับ Link
แต่เราต้องการให้ผู้ใช้ของเราเข้าใจว่าชนิดที่เราคืนคืออะไรกันแน่ และ Option ก็เป็น
ที่แพร่หลายจน*ทุกคน*รู้จักมัน อันที่จริงมันเป็นพื้นฐานขนาดที่มันถูก import เข้าสู่สโคป
โดยปริยายในทุกไฟล์ พร้อมกับวาเรียนต์ `Some` และ `None` ของมันด้วย
(ดังนั้นเราไม่ต้องเขียน `Option::None`)

ส่วนที่แหลม ๆ บน `Option<T>` บอกว่า Option นั้น*เจเนอริก*เหนือ T
นั่นหมายความว่าคุณสามารถสร้าง Option สำหรับ*ชนิดใดก็ได้*!

เอ่อ เราเจอเจ้า `Link` นี้แล้ว เราจะรู้ได้ยังไงว่ามันเป็น Empty หรือมี More?
ด้วยการจับคู่แพตเทิร์นกับ `match`!

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
}
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

โอ๊ะ `pop` ต้องคืนค่า และเรายังไม่ได้ทำแบบนั้น เราสามารถ return `None` ได้
แต่ในกรณีนี้มันอาจจะดีกว่าถ้า return `unimplemented!()` เพื่อระบุว่าเรายังไม่ได้
ทำให้ฟังก์ชันเสร็จ `unimplemented!()` เป็นมาโคร (`!` บอกว่ามันคือมาโคร)
ที่ทำให้โปรแกรมแพนิกเมื่อเราไปถึงมัน (≈ทำให้มันพังแบบควบคุมได้)

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

การแพนิกแบบไม่มีเงื่อนไขเป็นตัวอย่างของ[ฟังก์ชันแบบไดเวอร์จิง (diverging function)][diverging]
ฟังก์ชันแบบไดเวอร์จิงไม่เคยคืนค่ากลับไปยังผู้เรียก ดังนั้นมันจึงถูกใช้ได้ในที่ที่
คาดหวังค่าของชนิดใดก็ได้ ที่นี่ `unimplemented!()` ถูกใช้แทนค่าของชนิด `Option<T>`

โปรดสังเกตว่าเราไม่จำเป็นต้องเขียน `return` ในโปรแกรมของเรา นิพจน์สุดท้าย
(โดยพื้นฐานคือบรรทัดสุดท้าย) ในฟังก์ชันจะถูก return เป็นค่าของมันโดยปริยาย
สิ่งนี้ทำให้เราสามารถเขียนสิ่งที่ง่ายมาก ๆ ได้กระชับขึ้นเล็กน้อย คุณสามารถ
return อย่างชัดเจนก่อนเวลาด้วย `return` ได้เสมอเหมือนภาษาในตระกูล C อื่น ๆ

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

ไม่เอาน่า Rust อย่ามาตอแยเราเลย! ตามปกติ Rust โมโหมากที่เรา โชคดีที่ครั้งนี้
มันยังให้ข้อมูลครบถ้วนด้วย! โดยค่าเริ่มต้น การจับคู่แพตเทิร์นจะพยายามย้ายเนื้อหา
เข้าสู่แบรนช์ใหม่ แต่เราทำแบบนี้ไม่ได้เพราะเราไม่ได้เป็นเจ้าของ self แบบโดยค่า (by-value) ตรงนี้

```text
help: consider borrowing here: `&self.head`
```

Rust บอกว่าเราควรเพิ่มเรเฟอเรนซ์เข้าไปใน `match` ของเราเพื่อแก้ปัญหานี้ 🤷‍♀️ มาลองกัน:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

```text
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

ไชโย คอมไพล์ผ่านอีกครั้ง! ตอนนี้มาคิดตรรกะกัน เราต้องการสร้าง Option
งั้นมาสร้างตัวแปรสำหรับมัน ในกรณี Empty เราต้อง return None
ในกรณี More เราต้อง return `Some(i32)` และเปลี่ยนหัวของลิสต์
งั้นมาลองทำแบบนั้นดู?

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match &self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content

```

*หัว*

*โต๊ะ*

เรากำลังพยายามย้ายออกจาก `node` ทั้งที่เรามีแค่เรเฟอเรนซ์ร่วมกับมัน

เราควรถอยออกมาแล้วคิดว่าเรากำลังพยายามทำอะไร เราต้องการ:

* ตรวจสอบว่าลิสต์ว่างเปล่า
* ถ้าว่างเปล่า ก็แค่ return None
* ถ้า*ไม่*ว่างเปล่า
    * เอาหัวของลิสต์ออก
    * เอา `elem` ของมันออก
    * แทนที่หัวของลิสต์ด้วย `next` ของมัน
    * return `Some(elem)`

ข้อคิดสำคัญคือเราต้องการ*เอาออก* ซึ่งหมายความว่าเราต้องการหัวของลิสต์
*แบบโดยค่า (by value)* แน่นอนว่าเราทำแบบนั้นผ่านเรเฟอเรนซ์ร่วมที่เราได้จาก
`&self.head` ไม่ได้ เราก็มีแค่เรเฟอเรนซ์ที่เปลี่ยนแปลงได้กับ `self` เท่านั้น
ดังนั้นวิธีเดียวที่เราจะย้ายของได้คือการ*แทนที่มัน* ดูเหมือนเราจะต้องเต้น
Empty แดนซ์กันอีกรอบ!

มาลองแบบนั้น:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

โอ้ พระเจ้า

มันคอมไพล์ผ่านโดย*ไม่มีคำเตือนเลย*!!!!!

จริง ๆ แล้วผมจะใช้ลินต์ส่วนตัวของผมตรงนี้: เราสร้างค่า `result` นี้เพื่อคืนกลับ
แต่จริง ๆ แล้วเราไม่จำเป็นต้องทำแบบนั้นเลย! เช่นเดียวกับที่ฟังก์ชันประเมินผลเป็น
นิพจน์สุดท้าย ทุกบล็อกก็ประเมินผลเป็นนิพจน์สุดท้ายของมันเช่นกัน โดยปกติเราระงับ
พฤติกรรมนี้ด้วยเครื่องหมายเซมิโคลอน ซึ่งจะทำให้บล็อกประเมินผลเป็นทูเพิลว่าง `()`
นี่คือค่าที่ฟังก์ชันที่ไม่ได้ประกาศชนิดคืนค่า -- อย่าง `push` -- คืนกลับมา

ดังนั้น เราสามารถเขียน `pop` เป็นแบบนี้แทน:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

ซึ่งกระชับและเป็นธรรมชาติกว่าเล็กน้อย สังเกตว่าแบรนช์ Link::Empty
เสียปีกกาไปเลย เพราะเรามีนิพจน์เดียวให้ประเมินผล เป็นแค่ช็อตคัทที่สวยงามสำหรับกรณีง่าย ๆ

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

เยี่ยม ยังทำงานได้!

[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns