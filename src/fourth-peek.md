# Peek (การมอง)

เอาล่ะ เราผ่าน `push` และ `pop` มาแล้ว ไม่โกหก มันค่อนข้างอารมณ์
ตอนนี้ การคอมไพล์ถูกต้องเป็นยาเสพติดชั้นยอด

มาสงบสติอารมณ์ด้วยการทำอะไรที่ง่ายๆ: มา implement `peek_front`
แค่นั้น มันเคยง่ายมาก ยังคงง่ายอยู่ จริงไหม?

จริงไหม?

จริงๆ แล้ว ผมคิดว่าผมแค่ copy-paste ได้เลย!

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

เดี๋ยว ไม่ใช่ครั้งนี้

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // BORROW!!!!
        &node.borrow().elem
    })
}
```

ฮ่า

```text
cargo build

error[E0515]: cannot return value referencing temporary value
  --> src/fourth.rs:66:13
   |
66 |             &node.borrow().elem
   |             ^   ----------^^^^^
   |             |   |
   |             |   temporary value created here
   |             |
   |             returns a value referencing data owned by the current function
```

โอเค ผมแค่จะเผาคอมพิวเตอร์แล้ว

นี่คือตรรกะเดียวกับสแต็กแบบซิงกลีลิงก์ของเรา ทำไมสิ่งต่างๆ
ถึงต่างกัน ทำไม

คำตอบจริงๆ แล้วคือบทเรียนทั้งหมดของบทนี้: RefCells ทำให้ทุกอย่าง
น่าเศร้า จนถึงตอนนี้ RefCells เป็นแค่สิ่งรบกวน ตอนนี้พวกมันจะกลายเป็น
ฝันร้าย

เกิดอะไรขึ้น? เพื่อเข้าใจสิ่งนี้ เราต้องย้อนกลับไปที่นิยามของ
`borrow`:

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

ในส่วนโครงสร้างเราพูดว่า:

> แทนที่จะบังคับใช้แบบ static, RefCell บังคับใช้ที่รันไทม์
> ถ้าคุณฝ่าฝืนกฎ RefCell จะแค่แพนิกและทำให้โปรแกรมพัง
> ทำไมมันถึงคืนค่า Ref และ RefMut เหล่านี้? พวกมันทำงานเหมือน
> `Rc` แต่สำหรับการยืม นอกจากนี้พวกมันเก็บ RefCell ไว้ในสถานะถูกยืมจนกว่า
> จะออกจากสโคป **เราจะพูดถึงเรื่องนี้ทีหลัง**

ถึงเวลาแล้ว

`Ref` และ `RefMut` implement `Deref` และ `DerefMut` ตามลำดับ
ดังนั้นเพื่อวัตถุประสงค์ส่วนใหญ่พวกมันทำงาน*เหมือน* `&T` และ `&mut T`
แต่เพราะวิธีการทำงานของเทรตเหล่านั้น เรเฟอเรนซ์ที่ถูกคืนออกมาจะเชื่อมต่อ
กับไลฟ์ไทม์ของ Ref ไม่ใช่ RefCell จริงๆ นั่นหมายความว่า Ref
ต้องคงอยู่นานเท่าที่เรายังเก็บเรเฟอเรนซ์ไว้

สิ่งนี้จำเป็นจริงๆ สำหรับความถูกต้อง เมื่อ Ref ถูก drop มันจะบอก
RefCell ว่ามันไม่ได้ถูกยืมอีกต่อไป ดังนั้นถ้าเรา*จัดการ*เก็บเรเฟอเรนซ์
นานกว่าที่ Ref มีอยู่ เราอาจได้ RefMut ในขณะที่เรเฟอเรนซ์
ยังคงอยู่และทำลายระบบชนิดของ Rust ทั้งหมด

ดังนั้นเราอยู่ตรงไหน? เราแค่อยากคืนเรเฟอเรนซ์ แต่เราต้อง
เก็บ Ref นี้ไว้ แต่เมื่อเราคืนเรเฟอเรนซ์จาก `peek` ฟังก์ชันจะจบลง
และ `Ref` จะออกจากสโคป

😖

เท่าที่ผมรู้ เราจริงๆ แล้วตายอย่างสมบูรณ์ในน้ำที่นี่ คุณไม่สามารถ
 encapsulate การใช้ RefCells แบบนั้นได้

แต่... ถ้าเราแค่ยอมแพ้จากการซ่อนรายละเอียดการ implement ทั้งหมด?
ถ้าเราคืน Refs?

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        node.borrow()
    })
}
```

```text
> cargo build

error[E0412]: cannot find type `Ref` in this scope
  --> src/fourth.rs:63:40
   |
63 |     pub fn peek_front(&self) -> Option<Ref<T>> {
   |                                        ^^^ not found in this scope
help: possible candidates are found in other modules, you can import them into scope
   |
1  | use core::cell::Ref;
   |
1  | use std::cell::Ref;
   |
```

เอิ่ม ต้อง import อะไรบางอย่าง


```rust ,ignore
use std::cell::{Ref, RefCell};
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/fourth.rs:64:9
   |
64 | /         self.head.as_ref().map(|node| {
65 | |             node.borrow()
66 | |         })
   | |__________^ expected type parameter, found struct `fourth::Node`
   |
   = note: expected type `std::option::Option<std::cell::Ref<'_, T>>`
              found type `std::option::Option<std::cell::Ref<'_, fourth::Node<T>>>`
```

ฮึม... ถูกต้อง เรามี `Ref<Node<T>>` แต่เราอยากได้ `Ref<T>` เราอาจ
ละทิ้งความหวังทั้งหมดในการ encapsulate และแค่คืนค่านั้น
เราอาจทำให้สิ่งต่างๆ ซับซ้อนขึ้นและครอบ `Ref<Node<T>>` ด้วยชนิดใหม่
เพื่อเปิดเผยการเข้าถึง `&T` เท่านั้น

ทั้งสองตัวเลือก*ค่อนข้าง*แย่

แทนที่เราจะลงไปลึกกว่า มาสนุกกันเถอะ แหล่งความสนุกของเราคือ
*สิ่งนี้*:

```rust ,ignore
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> Make a new Ref for a component of the borrowed data.

ใช่: เหมือนกับที่คุณ map บน Option ได้ คุณก็ map บน Ref ได้

ผมแน่ใจว่ามีคนที่ไหนสักแห่งตื่นเต้นมากเพราะ *monads* หรืออะไรก็ตาม
แต่ผมไม่สนเรื่องเหล่านั้น นอกจากนี้ผมไม่คิดว่ามันเป็น monad ที่สมบูรณ์
เพราะไม่มี case ที่เหมือน None แต่เรื่องนั้นช่างเถอะ

มันเจ๋งและนั่นคือทั้งหมดที่ผมสน *ผมต้องการสิ่งนี้*

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}
```

```text
> cargo build
```

อ้าว ย-ย!

มาตรวจสอบว่ามันทำงานโดยใช้เทสต์จากสแต็กของเรา เราต้อง
แก้ไขเล็กน้อยเพื่อจัดการกับข้อเท็จจริงที่ว่า Refs ไม่ได้ implement การเปรียบเทียบ

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
}
```


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test third::test::basics ... ok
test second::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

เจ๋ง!
