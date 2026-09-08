# การทำลายโครงสร้าง

`pop_front` ควรมีตรรกะเดียวกับ `push_front` แต่กลับกัน ลองมาดู:

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    // need to take the old head, ensuring it's -2
    self.head.take().map(|old_head| {                         // -1 old
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {                               // -1 new
                // not emptying list
                new_head.borrow_mut().prev.take();            // -1 old
                self.head = Some(new_head);                   // +1 new
                // total: -2 old, +0 new
            }
            None => {
                // emptying list
                self.tail.take();                             // -1 old
                // total: -2 old, (no new)
            }
        }
        old_head.elem
    })
}
```

```text
> cargo build

error[E0609]: no field `elem` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:64:22
   |
64 |             old_head.elem
   |                      ^^^^ unknown field
```

อัค *RefCells* ต้อง `borrow_mut` อีกแล้วสินะ...

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {
                new_head.borrow_mut().prev.take();
                self.head = Some(new_head);
            }
            None => {
                self.tail.take();
            }
        }
        old_head.borrow_mut().elem
    })
}
```

```text
cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fourth.rs:64:13
   |
64 |             old_head.borrow_mut().elem
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot move out of borrowed content
```

*ถอนหายใจ*

> cannot move out of borrowed content

ฮึม... ดูเหมือนว่า Box จะ*ดีเกินไป*สำหรับเรา `borrow_mut` ได้แค่
`&Node<T>` แต่เราไม่สามารถย้ายออกจากมันได้!

เราต้องการบางอย่างที่รับ `RefCell<T>` แล้วคืน `T` มาให้ มาตรวจสอบ
[เอกสาร][refcell] สำหรับบางอย่างแบบนั้น:

> `fn into_inner(self) -> T`
>
> Consumes the RefCell, returning the wrapped value.

ดูน่าสนใจ!

```rust ,ignore
old_head.into_inner().elem
```

```text
> cargo build

error[E0507]: cannot move out of an `Rc`
  --> src/fourth.rs:64:13
   |
64 |             old_head.into_inner().elem
   |             ^^^^^^^^ cannot move out of an `Rc`
```

อะ 🥠 `into_inner` ต้องการย้าย RefCell ออกมา แต่เราทำไม่ได้เพราะมันอยู่ใน
`Rc` อย่างที่เราเห็นในบทก่อนหน้า `Rc<T>` อนุญาตให้เราได้แค่เรเฟอเรนซ์ร่วม
เข้าไปยังส่วนภายใน นั่นสมเหตุสมผลเพราะนั่นคือ*จุดประสงค์ทั้งหมด*ของ
ตัวชี้แบบนับเรเฟอเรนซ์: มันใช้ร่วมกัน!

นี่เป็นปัญหาสำหรับเราตอนที่เราต้องการใช้ Drop สำหรับลิสต์แบบนับเรเฟอเรนซ์
และวิธีแก้ก็เหมือนกัน: `Rc::try_unwrap` ที่ย้ายเนื้อหาของ Rc ออกมา
ถ้า refcount เท่ากับ 1

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

`Rc::try_unwrap` คืนค่า `Result<T, Rc<T>` Result โดยพื้นฐานแล้วเป็น
`Option` ที่ขยายไปแล้ว โดย case `None` มีข้อมูลที่เกี่ยวข้อง ในกรณีนี้
คือ `Rc` ที่คุณพยายาม unwrap เพราะเราไม่สนใจกรณีที่ล้มเหลว
(ถ้าเราเขียนโปรแกรมถูกต้อง มัน*ต้อง*สำเร็จ) เราแค่เรียก `unwrap` บนมัน

อย่างไรก็ตาม มาดูว่าเราจะได้ข้อผิดพลาดคอมไพเลอร์อะไรต่อไป
(ยอมรับเถอะ มันต้องมีสักอย่าง)

```text
> cargo build

error[E0599]: no method named `unwrap` found for type `std::result::Result<std::cell::RefCell<fourth::Node<T>>, std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>>` in the current scope
  --> src/fourth.rs:64:38
   |
64 |             Rc::try_unwrap(old_head).unwrap().into_inner().elem
   |                                      ^^^^^^
   |
   = note: the method `unwrap` exists but the following trait bounds were not satisfied:
           `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>> : std::fmt::Debug`
```

อุ๊ก `unwrap` บน Result ต้องการให้คุณ debug-print กรณีผิดพลาดได้
`RefCell<T>` จะ implement `Debug) ก็ต่อเมื่อ `T` implement
`Node` ไม่ได้ implement Debug

แทนที่จะทำแบบนั้น มาแก้ปัญหาโดยแปลง Result เป็น Option ด้วย `ok`:

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

PLEASE.

```text
cargo build

```

YES.

*ฟิว*

เราทำได้แล้ว

เรา implement `push` และ `pop` ได้แล้ว

มาทดสอบโดยยืมเทสต์พื้นฐานจากสแต็กเก่า (เพราะนั่นคือทั้งหมดที่
เรา implement ไปแล้ว):

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop_front(), None);

        // Populate list
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);
    }
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 9 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::basics ... ok
test fifth::test::iter_mut ... ok
test third::test::basics ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured

```

*เจ๋งมาก*

ตอนนี้เราสามารถลบสิ่งต่างๆ ออกจากลิสต์ได้อย่างถูกต้องแล้ว
เราสามารถ implement Drop ได้ Drop ค่อนข้างน่าสนใจกว่ารอบนี้
ตรงที่ก่อนหน้าเราทำ Drop สำหรับสแต็กแค่เพื่อหลีกเลี่ยง
การเรียกซ้ำแบบไม่จำกัด ตอนนี้เราต้อง implement Drop เพื่อให้
เกิด*อะไรขึ้นก็ได้*เลย

`Rc` ไม่สามารถจัดการกับวัฏจักรได้ ถ้ามีวัฏจักร ทุกอย่างจะทำให้
ทุกอย่างอื่นยังมีชีวิตอยู่ ลิงก์ลิสต์แบบลิงก์คู่ ปรากฏว่า เป็นแค่
ห่วงโซ่ของวัฏจักรเล็กๆ! ดังนั้นตอนที่เรา drop ลิสต์ โหนดสองปลายจะมี
refcount ลดลงเหลือ 1... แล้วจะไม่มีอะไรเกิดขึ้นอีก ถ้าลิสต์ของเรามี
Exactly one node เราก็โอเค แต่โดยทั่วไปลิสต์ควรทำงานถูกต้อง
ถ้ามีหลายองค์ประกอบ บางทีนั่นอาจเป็นแค่ผม

อย่างที่เราเห็น การลบองค์ประกอบค่อนข้างเจ็บปวด ดังนั้นสิ่งที่ง่ายที่สุด
สำหรับเราคือแค่ `pop` จนกว่าจะได้ None:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

```text
cargo build

```

(จริงๆ เราสามารถทำแบบนี้กับ mutable stacks ได้ แต่ทางลัดสำหรับ
คนที่เข้าใจ!)

เราสามารถดูการ implement เวอร์ชัน `_back` ของ `push` และ `pop` ได้
แต่มันเป็นแค่การ copy-paste ที่เราจะเลื่อนไปทีหลังในบท ตอนนี้
มาดูสิ่งที่น่าสนใจกว่า!


[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html
