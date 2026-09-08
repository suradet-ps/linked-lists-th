# พื้นฐาน

เราได้เรียนรู้พื้นฐานของ Rust ไปค่อนข้างมากแล้ว ดังนั้นเราสามารถทำ
สิ่งง่าย ๆ ซ้ำได้อีกครั้ง

สำหรับ constructor เราสามารถก๊อปแปะได้อีก:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

`push` และ `pop` ไม่ค่อยมีความหมายอีกต่อไป เราสามารถให้
`prepend` และ `tail` ซึ่งให้สิ่งที่คล้ายกัน

มาเริ่มกับ prepend มันรับลิสต์และองค์ประกอบหนึ่ง และคืน List
เหมือนกรณีลิสต์ที่กลายพันธุ์ได้ เราต้องการสร้างโหนดใหม่ที่มีลิสต์เก่า
เป็นค่า `next` ของมัน สิ่งใหม่เดียวคือวิธี*ได้*ค่า next นั้น
เพราะเราไม่ได้รับอนุญาตให้กลายพันธุ์อะไร

คำตอบแห่งคำอธิษฐานของเราคือเทรต Clone Clone ถูก implement
โดยแทบทุกชนิด และให้วิธีแบบเจเนอริกในการได้ "อีกตัวหนึ่งที่เหมือนตัวนี้"
ที่แยกจากกันทางตรรกะ โดยมีแค่เรเฟอเรนซ์ร่วม มันเหมือน copy constructor
ใน C++ แต่มันไม่เคยถูกเรียกใช้โดยปริยาย

Rc โดยเฉพาะใช้ Clone เป็นวิธีเพิ่มจำนวนอ้างอิง ดังนั้นแทนที่จะย้าย
Box ไปอยู่ในซับลิสต์ เราแค่ clone หัวของลิสต์เก่า เราไม่ต้อง match
บนหัวด้วยซ้ำ เพราะ Option มีการ implement Clone ที่ทำสิ่งที่เราต้องการ

เอาล่ะ ลองกัน:

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

ว้าว Rust จริงจังมากเรื่องการใช้งานฟิลด์จริง ๆ มันบอกได้ว่าไม่มีผู้บริโภค
คนไหนจะสังเกตเห็นการใช้งานฟิลด์เหล่านี้ได้จริง ๆ! ยังไงก็ตาม ดูเหมือนเราโอเค

`tail` คือกลับด้านตรรกะของโอเปอเรชันนี้ มันรับลิสต์และคืนลิสต์ทั้งหมด
โดยนำองค์ประกอบแรกออก ทั้งหมดคือ clone องค์ประกอบ*ที่สอง*ในลิสต์
(ถ้ามีอยู่) ลองกัน:

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

ฮึม เราทำพัง `map` คาดหวังให้เราคืน Y แต่ที่นี่เราคืน `Option<Y>`
โชคดีที่นี่เป็นอีกแพตเทิร์นหนึ่งของ Option ที่ใช้บ่อย และเราสามารถใช้
`and_then` เพื่อให้เราคืน Option ได้

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```text
> cargo build

```

เยี่ยม

ตอนนี้เรามี `tail` แล้ว เราควรให้ `head` ซึ่งคืนเรเฟอเรนซ์ไปยัง
องค์ประกอบแรก นั่นก็คือ `peek` จากลิสต์ที่กลายพันธุ์ได้:

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

```text
> cargo build

```

ดี

ฟังก์ชันเพียงพอที่เราจะทดสอบมันได้:

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

Iter ก็เหมือนกับที่เคยเป็นสำหรับลิสต์ที่กลายพันธุ์ได้:

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

ใครเคยบอกว่า dynamic typing ง่ายกว่า?

(พวกโง่บอก)

โปรดสังเกตว่าเราไม่สามารถ implement IntoIter หรือ IterMut สำหรับชนิดนี้ได้
เรามีการเข้าถึงองค์ประกอบแบบร่วมเท่านั้น