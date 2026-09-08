# การทำลายโครงสร้าง

ลอจิกของ `pop_front` ควรจะมีโครงสร้างเหมือนกับ `push_front` ทุกอย่าง เพียงแค่ย้อนศรกันเท่านั้น งั้นมาลองเขียนกัน:

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

โอ๊ย... เจ้าพวก *RefCell* ทั้งหลาย! สงสัยเราคงต้องเรียก `borrow_mut` อีกรอบสินะ...

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

*เฮ้อ...*

> cannot move out of borrowed content

หืม... ดูเหมือนว่าตอนใช้ `Box` ชีวิตเราจะ *สบายเกินไป* จนเคยตัวแฮะ เพราะการเรียก `borrow_mut` มันส่งคืนกลับมาแค่ `&mut Node<T>` เท่านั้น ซึ่งเราไม่ได้รับอนุญาตให้ย้าย (move) ข้อมูลหลุดออกมาจากเรเฟอเรนซ์แบบนั้นได้!

สิ่งที่เราต้องการจริง ๆ คือฟังก์ชันที่รับ `RefCell<T>` เข้าไป แล้วคาย `T` ตัวจริงออกมาให้เรา งั้นลองแวะไปเปิด [เอกสารคู่มือ][refcell] ดูสิว่ามีอะไรทำนองนั้นไหม:

> `fn into_inner(self) -> T`
>
> Consumes the RefCell, returning the wrapped value.

ฟังดูเข้าท่าแฮะ!

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

พับผ่าสิ... เจ้า `into_inner` มันอยากจะย้าย (move) ตัว RefCell ออกมาจริง ๆ นั่นแหละ แต่เราทำไม่ได้ เพราะมันถูกห่อหุ้มไว้ข้างใน `Rc` อีกที! และอย่างที่เราได้เห็นกันไปแล้วในบทก่อนหน้า เจ้า `Rc<T>` ยินยอมให้เราขอยืมได้แค่ shared reference เพื่อมองเข้าไปดูข้อมูลข้างในของมันเท่านั้น ซึ่งมันก็สมเหตุสมผลของมันดีอยู่หรอก เพราะนั่นคือ *หัวใจสำคัญที่สุด* ของพอยน์เตอร์แบบนับจำนวนอ้างอิง: มันมีไว้ให้ทุกคนแชร์ใช้งานร่วมกัน!

ปัญหานี้เป็นปัญหาเดียวกับตอนที่เราพยายามเขียน `Drop` ให้กับลิสต์แบบ reference-counted เลย และทางออกก็ยังคงเหมือนเดิมไม่เปลี่ยน: นั่นคือเรียกใช้ `Rc::try_unwrap` ซึ่งจะยอมย้ายเอาข้อมูลข้างในของ `Rc` ออกมาให้ หากค่า refcount ของมันเหลืออยู่แค่ 1 ตัวเป๊ะ ๆ

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

เมธอด `Rc::try_unwrap` จะคืนค่ากลับมาเป็น `Result<T, Rc<T>>` ซึ่งโดยพื้นฐานแล้ว `Result` ก็เปรียบเสมือน `Option` ในเวอร์ชันที่กว้างขึ้น โดยฝั่งกรณีของ `None` จะมีข้อมูลความผิดพลาดแนบติดมาด้วยเสมอ ในที่นี้ก็คือตัว `Rc` ที่เราพยายามจะ unwrap มันนั่นเอง และเนื่องจากเราไม่ได้สนใจเคสที่มันจะล้มเหลวอยู่แล้ว (เพราะถ้าเราเขียนโปรแกรมของเรามาถูกต้อง มัน *จะต้อง* ทำงานสำเร็จเสมอ!) เราจึงสามารถสั่ง `.unwrap()` ทับมันไปตรง ๆ ได้เลย

อย่างไรก็ดี มานั่งลุ้นกันต่อดีกว่าว่าเราจะได้เจอ error อะไรจากคอมไพเลอร์เป็นรายต่อไป (ยอมรับความจริงเถอะครับว่า มันต้องมีโผล่มาแน่นอน)

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

โธ่เอ๊ยยย... การสั่ง `unwrap` บน `Result` กำหนดเงื่อนไขไว้ว่าไทป์นั้นจะต้องสามารถพิมพ์ค่า Debug ออกมาได้ยามเกิด error แต่ทว่า `RefCell<T>` จะ implement `Debug` ก็ต่อเมื่อชนิดข้อมูล `T` ข้างใน implement `Debug` เท่านั้น และในตอนนี้ `Node` ของเราไม่ได้ implement `Debug` สักกะหน่อย

แทนที่จะต้องไปเสียเวลา implement อะไรพวกนั้น มาแก้ขัดแบบง่าย ๆ ด้วยการแปลง `Result` ให้กลายเป็น `Option` ผ่านเมธอด `.ok()` กันดีกว่า:

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

ขอร้องล่ะรอบนี้...

```text
cargo build

```

เยสสสสสสสสส!

*ฟู่... (ถอนหายใจยาว)*

เราทำสำเร็จแล้ว

ในที่สุดเราก็ implement ทั้ง `push` และ `pop` ได้เสียที!

งั้นมาลองเขียนเทสต์กัน โดยขอยืมเทสต์เบสิกของ `stack` จากบทเก่ามาใช้หน่อย (เพราะตอนนี้เราเพิ่งจะเขียนเสร็จไปแค่นั้นจริง ๆ):

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

*ทำได้เป๊ะเว่อร์ (Nailed it)*

ทีนี้เมื่อเราสามารถดึงข้อมูลออกจากลิสต์ได้อย่างถูกต้องแล้ว เราก็พร้อมที่จะลงมือ implement `Drop` กันต่อ ซึ่งในรอบนี้ เจ้า `Drop` จะมีความน่าสนใจในแง่ของคอนเซ็ปต์ขึ้นมาอีกระดับหนึ่ง ในขณะที่ก่อนหน้านี้เรายอมเหนื่อย implement `Drop` ให้กับสแต็กเพียงเพื่อป้องกันปัญหาการเรียกซ้ำลึกเกินไป (unbounded recursion) จนสแต็กทะลัก แต่ในรอบนี้ เราจำเป็นต้องเขียน `Drop` ขึ้นมา เพื่อให้ตัวลิสต์ *ยอมคืนหน่วยความจำทิ้งจริง ๆ* สักทีต่างหาก!

เพราะเจ้า `Rc` มันไม่สามารถจัดการกับปัญหาการอ้างอิงเป็นวงรอบ (cycles) ได้เลย! ถ้าหากมี cycle เกิดขึ้น ทุกสิ่งทุกอย่างในนั้นก็จะช่วยกันเหนี่ยวรั้งให้ทุกสิ่งทุกอย่างยังมีชีวิตอยู่ต่อไปชั่วกาลนาน และลิงก์ลิสต์แบบสองทาง แท้จริงแล้วมันก็คือสายโซ่ยาวเหยียดที่ร้อยเรียงวัฏจักรจิ๋ว ๆ (tiny cycles) ต่อกันไปเรื่อย ๆ นั่นเอง! ดังนั้น เมื่อเราสั่ง drop ลิสต์ของเรา โหนดปลายสุดทั้งสองฝั่งจะมีค่า refcount ลดลงเหลือ 1... แล้วหลังจากนั้นก็จะไม่มีอะไรถูกคืนเมโมรีอีกเลย... ใช่ครับ ถ้าลิสต์ของเรามีโหนดอยู่เพียงแค่โหนดเดียว มันก็คงไม่มีปัญหาอะไรหรอก แต่ในโลกความเป็นจริง โครงสร้างลิสต์ที่ดีมันควรจะทำงานถูกต้องได้เวลาที่มีหลาย ๆ ข้อมูลสิ... หรือว่าผมคิดไปเองคนเดียวก็ไม่รู้สินะ

และอย่างที่เราได้เห็นกันไปแล้ว การไล่แคะเอาอิลิเมนต์ออกจากโหนดมันช่างเจ็บปวดกระดองใจเหลือเกิน ดังนั้นวิธีที่ง่ายและสบายใจที่สุดสำหรับเราตอนนี้ ก็แค่สั่งวนลูป `pop_front` ไปเรื่อย ๆ จนกว่าจะได้ค่า `None` ออกมานั่นแหละ จบ!

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

(จริง ๆ แล้วเราสามารถใช้ท่านี้กับสแต็กแบบแก้ไขค่าได้ในบทก่อน ๆ ได้เหมือนกันนะ แต่ทางลัดง่าย ๆ แบบนั้นเก็บไว้ให้คนที่เข้าใจมันอย่างถ่องแท้ดีกว่า!)

จริง ๆ เราจะแวะไปเขียนเมธอดเวอร์ชัน `_back` ให้กับทั้ง `push` และ `pop` ตอนนี้เลยก็ได้ แต่เนื่องจากโค้ดพวกนั้นมันก็แค่งานก็อปแปะสลับด้าน เราจึงขอผัดผ่อนยกยอดไปเขียนทีเดียวในช่วงท้ายของบท ตอนนี้เราข้ามไปดูอะไรที่น่าตื่นเต้นกว่านั้นกันก่อนดีกว่า!


[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html
