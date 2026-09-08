# Peek (การมอง)

เอาล่ะ เราฝ่าด่าน `push` กับ `pop` มาได้สำเร็จแล้ว จะไม่โกหกเลยนะว่า เมื่อกี้มันชวนให้อารมณ์แปรปรวนอยู่เหมือนกัน ความถูกต้องตอนคอมไพล์ไทม์ (compile-time correctness) นี่มันช่างเสพติดจนถอนตัวไม่ขึ้นจริง ๆ

มาพักสูดหายใจคลายเครียดด้วยการทำอะไรเบสิก ๆ กันดีกว่า: มา implement `peek_front` กันเถอะ ปกติไอ้เมธอดตัวนี้มันเคยเขียนง่ายสุด ๆ มาตลอด คราวนี้มันก็ยังต้องง่ายเหมือนเดิมสิ จริงไหม?

ใช่มะ?

จริง ๆ แล้ว ผมคิดว่ารอบนี้ผมก็แค่ก็อปแปะโค้ดเดิมมาได้เลยด้วยซ้ำ!

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

เดี๋ยวก่อน... รอบนี้ไม่ได้แฮะ

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // BORROW!!!!
        &node.borrow().elem
    })
}
```

ฮ่า!

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

โอเค... เผาคอมพิวเตอร์ทิ้งไปเลยดีกว่า (Ok I'm just burning my computer.)

นี่มันคือลอจิกเดียวกันเป๊ะกับที่เราเคยใช้ในสแต็กเชื่อมโยงทางเดียวเลยนะเว้ย! แล้วทำไมรอบนี้มันถึงพังล่ะ? ทำไมมมมมม?!

และคำตอบของคำถามนี้ก็คือคติธรรมประจำใจของบทนี้ครับ: เจ้า `RefCell` ทำให้ทุกสรรพสิ่งจมดิ่งสู่ความโศกเศร้า! จนถึงตอนนี้ `RefCell` เป็นเพียงแค่ตัวน่ารำคาญ แต่หลังจากนี้เป็นต้นไป มันกำลังจะกลายสภาพเป็นฝันร้ายอย่างแท้จริง

แล้วตกลงมันเกิดอะไรขึ้นกันแน่? เพื่อที่จะเข้าใจเรื่องนี้ เราจำเป็นต้องย้อนกลับไปดูนิยามตั้งต้นของเมธอด `borrow`:

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

ในหัวข้อโครงสร้างหน่วยความจำ เราเคยกล่าวไว้ว่า:

> แต่แทนที่จะตรวจสอบและบังคับใช้กฎนี้แบบ static ตอนคอมไพล์ เจ้า RefCell กลับเลือกที่จะตรวจสอบกฎเหล่านี้ตอนรันไทม์แทน!
> และถ้าคุณแอบฝ่าฝืนกฎ RefCell ก็จะสั่ง panic แล้วแครชโปรแกรมทิ้งทันที!
> แล้วทำไมมันต้องคืนค่าเป็น Ref กับ RefMut ออกมาด้วยล่ะ? ก็เพราะว่าจริง ๆ แล้วชนิดข้อมูลพวกนี้ทำหน้าที่คล้ายกับ `Rc` แต่ใช้สำหรับการยืม
> และพวกมันยังช่วยล็อกให้ RefCell คงสถานะการถูกยืมเอาไว้จนกว่าตัวมันเองจะหลุดออกจากสโคป **ซึ่งเดี๋ยวเราจะมาเจาะลึกเรื่องนี้กันต่อ**

และตอนนี้... ถึงเวลานั้นแล้วครับ

ทั้ง `Ref` และ `RefMut` ต่าง implement `Deref` และ `DerefMut` ตามลำดับ ทำให้ในทางปฏิบัติตัวมันมีพฤติกรรม *แทบจะเหมือนกับ* `&T` และ `&mut T` ทุกประการ แต่ทว่า ด้วยกลไกการทำงานของ trait เหล่านั้น เรเฟอเรนซ์ที่ถูกส่งคืนออกมาจะถูกผูกโยงเข้ากับไลฟ์ไทม์ของตัว `Ref` เอง ไม่ใช่ผูกกับตัว `RefCell` จริง ๆ! นั่นแปลว่าตัว `Ref` จะต้องยังมีชีวิตอยู่เคียงข้างตราบเท่าที่เรายังถือเรเฟอเรนซ์นั้นอยู่

ซึ่งในความเป็นจริง นี่คือสิ่งจำเป็นอย่างยิ่งยวดเพื่อความถูกต้องสมบูรณ์ เพราะเมื่อตัว `Ref` ถูก drop ทิ้งไป มันจะคอยส่งสัญญาณไปบอก `RefCell` ว่าตอนนี้ไม่มีใครขอยืมข้อมูลนี้แล้วนะ ดังนั้นถ้าหากเราดันทะลึ่งถือเรเฟอเรนซ์ค้างไว้ได้นานเกินกว่าอายุขัยของ `Ref` เราก็อาจจะแอบไปขอ `RefMut` ในขณะที่ยังมีคนอื่นถือเรเฟอเรนซ์ค้างคาอยู่ได้ ซึ่งนั่นจะหักคอระบบความปลอดภัยของ Type System ใน Rust พังยับเยินเป็นสองท่อนทันที!

แล้วเราจะติดหล่มอยู่ตรงไหนล่ะ? สิ่งที่เราอยากคืนค่าออกไปมีแค่เรเฟอเรนซ์ธรรมดา ๆ แต่เรากลับจำเป็นต้องถือเจ้าวัตถุ `Ref` นี้ค้างไว้ด้วย ทว่าทันทีที่เราคืนเรเฟอเรนซ์ออกจากฟังก์ชัน `peek` ฟังก์ชันก็ทำงานเสร็จสิ้น และส่งผลให้ตัว `Ref` หลุดออกจากสโคปแล้วตายจากไปทันที!

😖

เท่าที่ผมรู้ ตอนนี้เรือของเราอับปางจมสนิทอยู่กลางทะเลเป็นที่เรียบร้อย (dead in the water) เพราะคุณไม่สามารถทำการแคปซูลห่อหุ้ม (encapsulate) การใช้งาน `RefCell` แบบมิดชิดขนาดนั้นได้เลย

แต่... ถ้าเรายอมยกธงขาว เลิกคิดที่จะซ่อนรายละเอียดเบื้องหลัง (implementation details) ให้มิดชิดดูล่ะ? จะเกิดอะไรขึ้นถ้าเรายอมคืนค่าเป็น `Ref` ออกไปตรง ๆ เลย?

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

แป่ว... สงสัยต้อง import ของเพิ่มสักหน่อยแล้ว

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

หืม... จริงด้วยแฮะ ตอนนี้สิ่งที่เรามีในมือคือ `Ref<Node<T>>` แต่สิ่งที่เราอยากส่งออกไปจริง ๆ คือ `Ref<T>` ต่างหาก เราอาจจะยอมทิ้งความหวังเรื่อง Encapsulation ทั้งหมดแล้วโยนไอ้ค่านั้นออกไปตรง ๆ เลยก็ได้ หรือเราอาจจะทำให้เรื่องมันยุ่งยากซับซ้อนขึ้นไปอีกขั้น ด้วยการสร้างชนิดข้อมูลใหม่มาครอบ `Ref<Node<T>>` เพื่อเปิดให้เข้าถึงได้เฉพาะ `&T`... แต่บอกตรง ๆ ว่าทั้งสองทางเลือกนี้มัน *ค่อนข้างจะ* ปัญญาอ่อนสิ้นดี

แต่แทนที่จะทำแบบนั้น เราจะดำดิ่งลงไปให้ลึกกว่าเดิม! มาหาเรื่องสนุก ๆ ทำกันดีกว่า และขุมพลังความสนุกของเราในรอบนี้ก็คือ *ไอ้สัตว์ประหลาดตัวนี้ (this beast)*:

```rust ,ignore
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> Make a new Ref for a component of the borrowed data.

ถูกต้องครับ! เช่นเดียวกับที่คุณสามารถเรียก `.map()` บน `Option` ได้ คุณก็สามารถสั่ง map บน `Ref` ได้เช่นกัน!

ผมมั่นใจเลยว่าคงมีใครสักคนตรงไหนสักแห่งกำลังตื่นเต้นเนื้อเต้นกับคำว่า *Monad* หรืออะไรทำนองนั้นอยู่แน่ ๆ แต่บอกเลยว่าผมไม่แคร์เรื่องพวกนั้นสักนิด! อีกอย่าง ผมว่ามันก็ไม่ใช่ Monad แบบสมบูรณ์อะไรด้วยซ้ำเพราะมันไม่มีเคสว่างเปล่าแบบ None ให้ใช้... แต่เอาเถอะ ช่างหัวมันปะไร! ประเด็นคือมันโคตรเท่ และนั่นคือสิ่งเดียวที่ผมสนใจ *ผมต้องได้สิ่งนี้!*

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

อู้วววววววววววว เย็ดเข้! (Awww yissss)

มาตรวจเช็กให้ชัวร์ว่ามันทำงานได้จริง โดยการดัดแปลงเทสต์จากสแต็กบทก่อนมาใช้ ซึ่งเราจำเป็นต้องปรับโค้ดนิดหน่อยเพื่อรับมือกับความจริงที่ว่า `Ref` มันไม่ได้ implement การเปรียบเทียบค่า (`PartialEq`) ให้เราโดยอัตโนมัติ:

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

ยอดเยี่ยมกระเทียมเจียว!
