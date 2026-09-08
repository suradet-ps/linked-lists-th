# พื้นฐาน

ตอนนี้เราเข้าใจพื้นฐานของ Rust ไปเยอะพอตัวแล้ว ดังนั้นเรื่องเบสิกหลาย ๆ อย่างเราก็นำกลับมาเขียนใหม่ได้สบาย ๆ

สำหรับ constructor เราก็แทบจะก็อปแปะโค้ดเดิมมาได้เลย:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

ส่วนเมธอดอย่าง `push` และ `pop` ดูจะไม่ค่อยสมเหตุสมผลสำหรับลิสต์แบบนี้เท่าไหร่นัก แต่เราสามารถเตรียม `prepend` และ `tail` มาให้ใช้งานแทน ซึ่งทำหน้าที่คล้ายคลึงกันอย่างมาก

มาเริ่มจาก `prepend` กันก่อนเลย มันจะรับตัวลิสต์เองกับอิลิเมนต์ตัวใหม่เข้ามา แล้วคืน `List` ตัวใหม่ออกไป เช่นเดียวกับกรณีของลิสต์แบบแก้ไขค่าได้ (mutable list) เราต้องการสร้างโหนดใหม่ที่ถือลิสต์ตัวเก่าเป็นค่า `next` สิ่งแปลกใหม่เพียงอย่างเดียวในที่นี้ก็คือ วิธีการที่เราจะ *ดึง* ค่า `next` นั้นมาใส่ เพราะในลิสต์แบบนี้เราไม่ได้รับอนุญาตให้เข้าไปแก้ไขค่า (mutate) อะไรในตัวเดิมได้เลย

และคำตอบที่สวรรค์ประทานมาให้ก็คือ trait `Clone`! แทบทุกชนิดข้อมูลใน Rust ล้วน implement `Clone` เอาไว้ ซึ่งมันเปิดทางให้เราสร้าง "อีกตัวหนึ่งที่เหมือนตัวนี้ทุกประการ" ขึ้นมาใหม่อย่างเป็นอิสระต่อกันในทางตรรกะ โดยขอเพียงแค่มี shared reference ตัวเดียวก็พอ มันให้ความรู้สึกคล้ายกับ copy constructor ใน C++ แต่จะไม่ถูกเรียกใช้งานโดยปริยาย (implicitly) แบบสุ่มสี่สุ่มห้าเด็ดขาด

โดยเฉพาะอย่างยิ่งสำหรับ `Rc` นั้น การสั่ง `.clone()` จะถูกใช้เป็นกลไกในการเพิ่มตัวนับจำนวนอ้างอิง (reference count) ขึ้นหนึ่งค่า ดังนั้น แทนที่เราจะต้องย้าย (move) `Box` เข้าไปอยู่ในซับลิสต์ เราก็เพียงแค่สั่ง clone ที่ส่วนหัวของลิสต์ตัวเก่าได้เลย แถมเราไม่จำเป็นต้องเขียน `match` เพื่อแกะ `head` ด้วยซ้ำ เพราะตัว `Option` เองก็มี implementation ของ `Clone` ที่จัดการลอจิกทั้งหมดนี้ให้เราอย่างตรงใจเป๊ะ

เอาล่ะ มาลองดูกันสักตั้ง:

```rust ,ignore
pub fn prepend(&self, elem: T) -> List<T> {
    List { head: Some(Rc::new(Node {
        elem: elem,
        next: self.head.clone(),
    }))}
}
```

```text
> cargo build

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

ว้าว Rust นี่เคร่งครัดเอาเรื่องกับการเช็กว่าฟิลด์ถูกใช้งานจริงไหมจริง ๆ มันฉลาดพอที่จะรู้ว่ายังไม่มีโค้ดภายนอกส่วนไหนสามารถมองเห็นหรือหยิบฟิลด์พวกนี้ไปใช้งานได้เลย! แต่ถึงอย่างนั้น เท่าที่ดูตอนนี้โค้ดเราก็ยังไปได้สวย

ส่วน `tail` ก็คือคู่ตรงข้ามในทางตรรกะของโอเปอเรชันนี้ มันจะรับลิสต์เข้ามา แล้วคืนลิสต์ทั้งก้อนกลับไปโดยตัดอิลิเมนต์ตัวแรกทิ้ง ซึ่งทั้งหมดที่มันทำก็แค่ clone อิลิเมนต์ตัวที่ *สอง* ของลิสต์ออกมา (หากมีตัวตนอยู่) งั้นมาลองเขียนกัน:

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().map(|node| node.next.clone()) }
}
```

```text
cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

หืม เราทำพลาดไปหน่อย `map` คาดหวังให้เราคืนค่าชนิด `Y` ออกมา แต่ในโค้ดนี้เราดันคืน `Option<Y>` ออกมาเสียนี่ โชคดีที่นี่เป็นอีกหนึ่งแพตเทิร์นยอดฮิตของ `Option` เราจึงสามารถหยิบใช้ `and_then` เพื่อให้เราสามารถคืนค่า `Option` ออกมาได้โดยตรง

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```text
> cargo build

```

ยอดเยี่ยม

เมื่อเรามี `tail` แล้ว เราก็ควรจัดเตรียม `head` ที่จะคืนค่าเรเฟอเรนซ์ไปยังอิลิเมนต์ตัวแรกด้วยเช่นกัน ซึ่งจริง ๆ แล้วมันก็คือการยกเอาเมธอด `peek` จากลิสต์แบบแก้ไขค่าได้มาใช้นั่นเอง:

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

```text
> cargo build

```

แจ่มเลย

ฟังก์ชันการทำงานแค่นี้ก็เพียงพอแล้วที่เราจะเริ่มเขียนเทสต์ทดสอบมันได้:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.prepend(1).prepend(2).prepend(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // Make sure empty tail works
        let list = list.tail();
        assert_eq!(list.head(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured

```

สมบูรณ์แบบ!

การ implement `Iter` ก็เหมือนกันเป๊ะกับที่เราเคยทำไว้ในลิสต์แบบแก้ไขค่าได้:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

```rust ,ignore
#[test]
fn iter() {
    let list = List::new().prepend(1).prepend(2).prepend(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 7 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured

```

ไหนใครเคยพูดนะว่าไดนามิกไทป์ (Dynamic typing) มันง่ายกว่า?

(พวกมือสมัครเล่นเท่านั้นแหละที่พูดแบบนั้น)

โปรดสังเกตว่า เราไม่สามารถ implement `IntoIter` หรือ `IterMut` ให้กับชนิดข้อมูลนี้ได้ เพราะเราสามารถเข้าถึงอิลิเมนต์ต่าง ๆ ได้ผ่านทาง shared reference เท่านั้น