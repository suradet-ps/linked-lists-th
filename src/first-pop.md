# Pop (การนำออกจากหัว)

เช่นเดียวกับ `push` เมธอด `pop` จำเป็นต้องแก้ไขเปลี่ยนแปลงค่าในลิสต์ แต่ต่างจาก `push` ตรงที่เราต้องการให้มันคืนค่า (return) อะไรบางอย่างกลับไปด้วย ทว่า `pop` ยังต้องรับมือกับกรณีพิเศษที่ชวนปวดหัว (corner case) อีกอย่างหนึ่งด้วย: นั่นคือถ้าลิสต์ว่างเปล่าล่ะ จะให้ทำยังไง? เพื่อจัดการกับกรณีนี้ เราจะหยิบชนิดข้อมูลที่ไว้ใจได้อย่าง `Option` มาใช้งาน:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
}
```

`Option<T>` คือ enum ที่ใช้เป็นตัวแทนของค่าที่อาจจะมีอยู่หรือไม่มีก็ได้ โดยมันเป็นไปได้สองรูปแบบคือ `Some(T)` หรือ `None` จริงอยู่ว่าเราจะเขียน enum ขึ้นมาใช้เองเหมือนกับที่เราทำกับ `Link` ก็ได้ แต่เราอยากให้คนที่เอาไลบรารีของเราไปใช้เข้าใจได้ทันทีว่าค่าที่เราส่งกลับไปคืออะไรกันแน่ และ `Option` ก็เป็นของสามัญประจำบ้านที่ *ทุกคนในโลก Rust* รู้จักกันเป็นอย่างดี อันที่จริง มันเป็นชนิดข้อมูลพื้นฐานเสียจนถูก import เข้ามาในสโคปของทุกไฟล์โดยอัตโนมัติ พร้อมกับวาเรียนต์ `Some` และ `None` ของมันด้วย (เราจึงเรียกใช้ `Some` หรือ `None` ได้ตรง ๆ โดยไม่ต้องพิมพ์ `Option::None`)

เครื่องหมายวงเล็บแหลมตรง `<T>` ของ `Option<T>` บ่งบอกว่า `Option` เป็นชนิดข้อมูลแบบ *เจเนอริก (generic)* เหนือ `T` ซึ่งหมายความว่าคุณสามารถสร้าง `Option` สำหรับ *ชนิดข้อมูลใด ๆ ก็ได้ในโลก*!

เอาล่ะ ทีนี้เรามีตัวแปร `Link` อยู่ แล้วเราจะรู้ได้ยังไงว่าข้างในมันเป็น `Empty` หรือมี `More` ซ่อนอยู่กันแน่? คำตอบคือ: การจับคู่แพตเทิร์นด้วย `match` ยังไงล่ะ!

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

อุ๊ปส์... `pop` ต้องส่งค่ากลับออกไป แต่ตอนนี้เรายังไม่ได้ส่งอะไรกลับไปเลย เรา *อาจจะ* ลองส่ง `None` กลับไปแก้ขัดก่อนก็ได้ แต่ในสถานการณ์นี้ การส่ง `unimplemented!()` น่าจะเป็นความคิดที่ดีกว่า เพื่อส่งสัญญาณเตือนว่าเรายังเขียนฟังก์ชันนี้ไม่เสร็จ โดย `unimplemented!()` เป็นมาโคร (เครื่องหมาย `!` บ่งบอกว่าเป็นมาโคร) ที่จะสั่งให้โปรแกรมแพนิก (panic) ทันทีที่ทำงานมาถึงจุดนี้ (อารมณ์ประมาณสั่งให้โปรแกรมแครชแบบควบคุมได้)

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

การสั่งให้แพนิกแบบไม่มีเงื่อนไขถือเป็นตัวอย่างหนึ่งของ [ฟังก์ชันที่ไม่มีวันส่งค่ากลับ (diverging function)][diverging] ฟังก์ชันประเภทนี้จะไม่มีวันส่งการควบคุมกลับไปยังผู้เรียก ดังนั้นมันจึงสามารถนำไปเสียบไว้ในตำแหน่งใดก็ตามที่คอมไพเลอร์คาดหวังค่าของชนิดข้อมูลใด ๆ ก็ได้ ในที่นี้ `unimplemented!()` จึงถูกนำมาใช้แทนที่ค่าของชนิด `Option<T>` ได้อย่างลงตัว

และโปรดสังเกตอีกครั้งว่าเราไม่จำเป็นต้องพิมพ์คำสั่ง `return` ในโปรแกรมเลย เพราะนิพจน์สุดท้าย (บรรทัดสุดท้าย) ในฟังก์ชันจะถือเป็นค่าที่ส่งกลับโดยอัตโนมัติ ซึ่งช่วยให้เราเขียนฟังก์ชันสั้น ๆ ได้อย่างกระชับขึ้นมาก แต่ถ้าอยากจะสั่งให้ออกจากฟังก์ชันก่อนกำหนด คุณก็ยังคงใช้ `return` ได้เสมอเหมือนภาษาตระกูล C ทั่วไป

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

โธ่เอ๊ย Rust เลิกจิกกัดเราสักทีเถอะ! และแน่นอนว่า Rust กำลังหงุดหงิดเราหนักมากเหมือนเคย แต่โชคดีที่คราวนี้มันให้คำอธิบายมาอย่างละเอียดยิบ! โดยตามธรรมชาติแล้ว การจับคู่แพตเทิร์นจะพยายามย้าย (move) ข้อมูลข้างในเข้าไปยังกิ่งก้าน (branch) ใหม่ แต่เราทำแบบนั้นไม่ได้ เพราะตรงนี้เราไม่ได้ถือความเป็นเจ้าของ `self` แบบส่งผ่านด้วยค่า (by-value)

```text
help: consider borrowing here: `&self.head`
```

Rust แนะนำว่าให้เราใส่เรเฟอเรนซ์เข้าไปใน `match` เพื่อแก้ปัญหานี้ 🤷‍♀️ งั้นมาลองทำตามดู:

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

ไชโย! กลับมาคอมไพล์ผ่านอีกครั้งแล้ว! ทีนี้เรามาวางลอจิกข้างในกันต่อ เราต้องการสร้างค่า Option ส่งกลับไป งั้นมาประกาศตัวแปรขึ้นมารับค่า ในกรณี `Empty` เราต้องส่ง `None` กลับไป ส่วนในกรณี `More` เราต้องส่ง `Some(i32)` กลับไป พร้อมกับปรับหัวของลิสต์ให้ชี้ไปยังโหนดถัดไป งั้นมาลองเขียนตามนั้นดูเลย:

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

*(เอาหัวโขกโต๊ะแพล็บ...)*

เรากำลังพยายามย้ายข้อมูลออกจาก `node` ทั้งที่เราถือเพียงแค่เรเฟอเรนซ์แบบแชร์ (shared reference) ไปยังมันเท่านั้น

เราคงต้องถอยหลังกลับมาหนึ่งก้าว แล้วคิดดูให้ดีว่าจริง ๆ แล้วเรากำลังพยายามจะทำอะไรกันแน่ สิ่งที่เราต้องการก็คือ:

* ตรวจสอบดูว่าลิสต์ว่างเปล่าหรือไม่
* ถ้าลิสต์ว่าง ก็ส่ง `None` กลับไป
* ถ้าลิสต์ *ไม่* ว่าง
    * ดึงหัวเดิมของลิสต์ออกมา
    * ดึงค่า `elem` ข้างในนั้นออกมา
    * เอา `next` ของมันมาเสียบแทนที่หัวลิสต์ตัวเดิม
    * ส่ง `Some(elem)` กลับไป

ประเด็นสำคัญที่สุดก็คือ เราต้องการ *ดึงของออกมา* ซึ่งนั่นแปลว่าเราต้องการหัวของลิสต์แบบ *ส่งผ่านด้วยค่า (by value)* แน่นอนว่าเราไม่มีทางทำแบบนั้นได้ผ่านเรเฟอเรนซ์แบบแชร์ที่เราได้มาจาก `&self.head` แถมเรายังมี "แค่" เรเฟอเรนซ์ที่แก้ไขได้ไปยัง `self` เท่านั้น ดังนั้นหนทางเดียวที่เราจะย้ายข้อมูลออกมาได้ ก็คือต้อง *เอาค่าอื่นเข้าไปแทนที่ทันที* ดูทรงแล้วเราคงต้องร่ายรำกระบวนท่าสลับค่าด้วย `Empty` กันอีกรอบแล้วล่ะ!

มาลองเขียนตามนี้ดู:

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

คุณพระช่วย!

มันคอมไพล์ผ่านฉลุยแบบ *ไม่มีคำเตือน (warning) โผล่มาแม้แต่อันเดียว*!!!!!

จริง ๆ แล้วผมขอใช้กฎความสะอาดส่วนตัวตรงนี้นิดหน่อย: ตะกี้เราสร้างตัวแปร `result` ขึ้นมารับค่าเพื่อส่งกลับ แต่เอาจริง ๆ เราไม่จำเป็นต้องทำแบบนั้นเลย! เพราะเช่นเดียวกับที่ฟังก์ชันจะส่งผลลัพธ์เป็นนิพจน์บรรทัดสุดท้าย ทุก ๆ บล็อกโค้ดใน Rust ก็จะประเมินผลลัพธ์ออกมาเป็นนิพจน์บรรทัดสุดท้ายของมันเช่นเดียวกัน โดยปกติเรามักจะระงับพฤติกรรมนี้ไว้ด้วยการใส่เครื่องหมายเซมิโคลอน (`;`) ซึ่งจะทำให้บล็อกนั้นคืนค่าเป็นทูเพิลว่างเปล่าแทน นั่นคือ `()` (ซึ่งจริง ๆ แล้วนี่คือค่าที่ฟังก์ชันที่ไม่ระบุชนิดข้อมูลส่งกลับ -- อย่างเช่น `push` -- ส่งกลับออกมานั่นเอง)

ดังนั้น เราจึงเขียนเมธอด `pop` ใหม่ออกมาได้แบบนี้:

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

ซึ่งโค้ดจะกระชับและเป็นธรรมชาติในแบบของ Rust มากขึ้น สังเกตว่าในกิ่ง `Link::Empty` เราตัดปีกกาทิ้งไปได้เลย เพราะมีนิพจน์ให้ประเมินผลเพียงแค่อันเดียว ถือเป็นชอร์ตคัทที่ช่วยให้อ่านง่ายขึ้นมากสำหรับกรณีสั้น ๆ

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

แจ่มมาก ยังคงทำงานได้ถูกต้องสมบูรณ์!

[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns