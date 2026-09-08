# การวนซ้ำ (Iteration)

มาลองวนซ้ำสิ่งนี้กัน

## IntoIter

IntoIter เหมือนเคย จะเป็นส่วนที่ง่ายที่สุด แค่ครอบสแต็กและ
เรียก `pop`:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop_front()
    }
}
```

แต่เรามีการพัฒนาใหม่ที่น่าสนใจ ตรงที่ก่อนหน้ามีแค่
"ธรรมชาติ" การวนซ้ำเดียวสำหรับลิสต์ เดกมีลักษณะ
two-way โดยธรรมชาติ อะไรพิเศษเกี่ยวกับ front-to-back?
ถ้ามีคนต้องการวนซ้ำในทิศทางตรงกันข้ามล่ะ?

Rust มีคำตอบสำหรับสิ่งนี้: `DoubleEndedIterator` DoubleEndedIterator
* inherits * จาก Iterator (หมายความว่าทุก DoubleEndedIterator เป็น Iterator)
และต้องการเมธอดใหม่หนึ่งอย่าง: `next_back` มันมีลายเซ็นเดียวกับ
`next` แต่มันจะคืนองค์ประกอบจากอีกปลายหนึ่ง ความหมายของ
DoubleEndedIterator สะดวกมากสำหรับเรา: อิเทอเรเตอร์กลายเป็น
เดก คุณสามารถบริโภคองค์ประกอบจากด้านหน้าและด้านหลังจนกว่า
สองปลายจะมาบรรจบกัน ซึ่งอิเทอเรเตอร์จะว่าง

เหมือนกับ Iterator และ `next` ปรากฏว่า `next_back` ไม่ใช่สิ่งที่
ผู้บริโภค DoubleEndedIterator สนใจจริงๆ แต่ส่วนที่ดีที่สุดของ
interface นี้คือมันเปิดเผยเมธอด `rev` ที่ครอบ
อิเทอเรเตอร์เพื่อสร้างอันใหม่ที่คืนองค์ประกอบในลำดับกลับกัน
ความหมายของมันค่อนข้างตรงไปตรงมา: การเรียก `next` บน
อิเทอเรเตอร์กลับกันเป็นแค่การเรียก `next_back`

อย่างไรก็ตาม เพราะเราเป็นเดกอยู่แล้ว การให้ API นี้จึงง่าย:

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

และมาทดสอบมัน:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push_front(1); list.push_front(2); list.push_front(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next_back(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next_back(), None);
    assert_eq!(iter.next(), None);
}
```


```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 11 tests
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::iter ... ok
test third::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured

```

ดี

## Iter

Iter จะไม่ให้อภัยมากนัก เราจะต้องจัดการกับ Ref ที่น่ากลัว
อีกครั้ง! เพราะ Refs เราไม่สามารถเก็บ `&Node`s ได้เหมือนที่เคย
แทนที่เราจะลองเก็บ `Ref<Node>`s:

```rust ,ignore
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}
```

```text
> cargo build

```

จนถึงตอนนี้ยังดี การ implement `next` จะค่อนข้างยุ่งยาก แต่ผมคิดว่า
มันเป็นตรรกะเดียวกับ IterMut เก่าแต่มี RefCell ที่บ้าคลั่ง:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

```text
cargo build

error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:155:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   -------- borrow is only valid in the closure body
    |             |
    |             reference to `node_ref` escapes the closure body here

error[E0505]: cannot move out of `node_ref` because it is borrowed
   --> src/fourth.rs:156:22
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- lifetime `'1` appears in the type of `self`
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ------   -------- borrow of `node_ref` occurs here
    |             |
    |             assignment requires that `node_ref` is borrowed for `'1`
156 |             Ref::map(node_ref, |node| &node.elem)
    |                      ^^^^^^^^ move out of `node_ref` occurs here
```

แย่

`node_ref` ไม่อยู่นานพอ ต่างจากเรเฟอเรนซ์ปกติ Rust ไม่ยอมให้
เราแยก Refs แบบนั้น Ref ที่เราได้จาก `head.borrow()` อนุญาตให้
อยู่นานเท่า `node_ref` เท่านั้น แต่เราจบลงด้วยการทำลายมันในการ
เรียก `Ref::map`

ฟังก์ชันที่เราต้องการมีอยู่แล้ว และมันชื่อ *[map_split][]*:

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

วูฟ มาลองดู...

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = next.as_ref().map(|head| head.borrow());

        elem
    })
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:159:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
...
159 |             self.0 = next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   ---- borrow is only valid in the closure body
    |             |
    |             reference to `next` escapes the closure body here
```

เอิ่ม เราต้องใช้ `Ref::Map` อีกครั้งเพื่อให้ lifetimes ถูกต้อง แต่ `Ref::Map`
คืน `Ref` แต่เราต้องการ `Option<Ref>` แต่เราต้องผ่าน
Ref เพื่อ map Option...

**จ้องไปไกลนานมาก**

??????

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = if next.is_some() {
            Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
        } else {
            None
        };

        elem
    })
}
```

```text
error[E0308]: mismatched types
   --> src/fourth.rs:162:22
    |
162 |                 Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `fourth::Node`, found struct `std::cell::RefCell`
    |
    = note: expected type `std::cell::Ref<'_, fourth::Node<_>>`
               found type `std::cell::Ref<'_, std::cell::RefCell<fourth::Node<_>>>`
```

โอ้ ถูกต้อง มี RefCells หลายตัว ยิ่งเราเดินลึกเข้าไปในลิสต์ เรายิ่ง
ซ้อนกันภายใต้แต่ละ RefCell เราจะต้องรักษา stack ของ
Refs เพื่อแสดงเงินกู้ที่เรายังถืออยู่ทั้งหมด เพราะถ้าเราหยุด
ดูองค์ประกอบหนึ่ง เราต้องลด borrow-count บนทุก RefCell
ที่มาก่อนมัน.........

ผมไม่คิดว่ามีอะไรที่เราทำได้ที่นี่ มันเป็นทางตัน มาลอง
ออกจาก RefCells

แล้ว Rc ของเราล่ะ? ใครบอกว่าเราต้องเก็บเรเฟอเรนซ์?
ทำไมเราถึงไม่ Clone Rc ทั้งหมดเพื่อให้ได้ตัวชี้ที่ดีไปยังกลาง
ลิสต์?

```rust ,ignore
pub struct Iter<T>(Option<Rc<Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.clone()))
    }
}

impl<T> Iterator for Iter<T> {
    type Item =
```

เอ่อ... เดี๋ยวตอนนี้เราคืนอะไร? `&T`? `Ref<T>`?

ไม่ ไม่มีสิ่งใดทำงาน... Iter ของเราไม่มีไลฟ์ไทม์อีกแล้ว! ทั้ง `&T`
และ `Ref<T>` ต้องการให้เราประกาศไลฟ์ไทม์ล่วงหน้าก่อนเข้าสู่
`next` แต่สิ่งใดก็ตามที่เราจัดการได้จาก Rc จะยืม
Iterators... สมอง... เจ็บ... อาาาาาาาาาาาาา

บางทีเราอาจ... map... Rc... ให้ได้ `Rc<T>`? นั่นมีอยู่จริงไหม?
เอกสารของ Rc ไม่ดูเหมือนจะมีอะไรแบบนั้น จริงๆ มีคนสร้าง [a crate][own-ref]
ที่ให้คุณทำแบบนั้นได้

แต่เดี๋ยว แม้เราจะทำ*แบบนั้น* เราก็มีปัญหาที่ใหญ่กว่า: ผีร้ายของ
iterator invalidation ก่อนหน้าเราได้รับการคุ้มครองจาก
iterator invalidation อย่างสมบูรณ์ เพราะ Iter ยืมลิสต์ ทำให้
ไม่สามารถเปลี่ยนแปลงได้เลย แต่ถ้า Iter ของเรากำลังคืน Rcs
มันจะไม่ยืมลิสต์เลย! นั่นหมายความว่าคนสามารถเริ่มเรียก `push` และ `pop`
บนลิสต์ในขณะที่พวกเขาถือตัวชี้เข้าไป!

โอ้พระเจ้า นั่นจะทำอะไร?!

จริงๆ push ไม่เป็นไร เรามีมุมมองเข้าไปยัง subrange ของ
ลิสต์ และลิสต์จะเติบโตเกินสายตาของเรา ไม่มีปัญหา

แต่ `pop` เป็นอีกเรื่องหนึ่ง ถ้าพวกเขากำลัง pop องค์ประกอบนอก
ช่วงของเรา มัน*ยัง*ไม่เป็นไร เราไม่เห็นโหนดเหล่านั้นจึงไม่มีอะไรเกิดขึ้น
แต่ถ้าพวกเขาพยายาม pop โหนดที่เราชี้อยู่... ทุกอย่างจะระเบิด!
โดยเฉพาะตอนที่พวกเขาไป `unwrap` ผลลัพธ์ของ
`try_unwrap` มันจะล้มเหลวจริงๆ และทั้งโปรแกรมจะแพนิก

นั่นเจ๋งมาก เราสามารถมี interior owning pointers จำนวนมาก
เข้าไปในลิสต์และเปลี่ยนแปลงมันได้ในเวลาเดียวกัน*และมันจะทำงาน*
จนกว่าพวกเขาจะพยายามลบโหนดที่เราชี้อยู่ และแม้ตอนนั้นเราไม่ได้
รับ dangling pointers หรืออะไร โปรแกรมจะแพนิกอย่างแน่นอน!

แต่การจัดการ iterator invalidation บนการ map Rcs
ดูเหมือน... แย่ `Rc<RefCell>` ได้ล้มเหลวเราอย่างแท้จริงในที่สุด
น่าสนใจตรงที่เราประสบกับการกลับกันของ persistent stack ตรงที่
persistent stack ดิ้นรนเพื่อ reclaim ความเป็นเจ้าของของข้อมูลแต่ได้
เรเฟอเรนซ์ตลอดทั้งวัน ลิสต์ของเราไม่มีปัญหาในการได้ความเป็นเจ้าของ
แต่ดิ้นรนมากในการให้ยืมเรเฟอเรนซ์

แม้ว่าจะยุติธรรม ส่วนใหญ่การดิ้นรนของเราเกี่ยวข้องกับการต้องการซ่อน
รายละเอียดการ implement และมี API ที่ดี เรา*สามารถ*ทำทุกอย่างได้ดี
ถ้าเราต้องการแค่ส่ง Nodes ไปมา

เฮ้ เราสามารถสร้าง IterMuts หลายตัวพร้อมกันที่ถูกตรวจสอบรันไทม์
เพื่อไม่ให้เข้าถึงองค์ประกอบเดียวกันแบบ mutable!

จริงๆ การออกแบบนี้เหมาะสมกว่าสำหรับโครงสร้างข้อมูลภายใน
ที่ไม่เคยออกมาสู่ผู้บริโภค API มิวเทบิลิตี้ภายในยอดเยี่ยมสำหรับ
การเขียน *applications* ที่ปลอดภัย ไม่ใช่ *libraries* ที่ปลอดภัย

อย่างไรก็ตาม นั่นคือผมยอมแพ้ Iter และ IterMut เราสามารถทำได้ แต่ *อุ๊ก*

[own-ref]: https://crates.io/crates/owning_ref
[map-split]: https://doc.rust-lang.org/std/cell/struct.Ref.html#method.map_split
