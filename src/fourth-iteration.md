# การวนซ้ำ (Iteration)

มาลองลงมือเขียนตัววนซ้ำ (iterator) ให้กับเจ้าสัตว์ประหลาดตัวนี้กันหน่อย

## IntoIter

`IntoIter` ก็เหมือนเช่นเคย มันจะเป็นส่วนที่เขียนง่ายที่สุดเสมอ แค่ห่อหุ้มตัวสแต็กเอาไว้ แล้วสั่งเรียก `pop`:

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

แต่ตอนนี้เรามีประเด็นใหม่ที่น่าสนใจผุดขึ้นมา: ในขณะที่ลิสต์ตัวก่อน ๆ มีลำดับการวนลูปที่เป็น "ธรรมชาติ" เพียงทิศทางเดียว แต่ Deque นั้นเป็นโครงสร้างแบบสองทิศทาง (bi-directional) ในตัวเองโดยกำเนิด แล้วทำไมจะต้องวนจากหน้าไปหลัง (front-to-back) เสมอไปด้วยล่ะ? ถ้ามีใครสักคนอยากวนลูปย้อนศรจากหลังมาหน้าบ้างล่ะ?

Rust มีคำตอบที่เตรียมไว้สำหรับเรื่องนี้อยู่แล้ว: นั่นคือ trait `DoubleEndedIterator`! เจ้า `DoubleEndedIterator` จะ *สืบทอด (subtrait)* มาจาก `Iterator` (แปลว่าทุกตัวที่เป็น DoubleEndedIterator จะต้องเป็น Iterator ด้วยเสมอ) และต้องการเมธอดใหม่เพิ่มขึ้นมาเพียงเมธอดเดียว: นั่นคือ `next_back` ซึ่งมีรูปแบบ signature เหมือนกับ `next` ทุกประการ เพียงแต่มันจะดึงอิลิเมนต์ออกมาจากปลายอีกฝั่งหนึ่งแทน ความหมายในทางพฤติกรรม (semantics) ของ `DoubleEndedIterator` นั้นช่างเข้าทางเราเหลือเกิน: ตัว iterator จะทำตัวเสมือน deque ไปในตัว คุณสามารถทยอยดึงข้อมูลจากหัวและท้ายสลับไปมาได้เรื่อย ๆ จนกระทั่งปลายทั้งสองฝั่งวิ่งมาชนกัน ซึ่งเมื่อถึงจุดนั้น iterator ก็จะหมดลงและกลายเป็นค่าว่าง

และเช่นเดียวกับ `Iterator` และ `next` ในความเป็นจริง ผู้ใช้งานทั่วไปมักไม่ได้มานั่งเรียก `next_back` กันตรง ๆ สักเท่าไหร่หรอก แต่ความเจ๋งที่สุดของอินเทอร์เฟซนี้ก็คือ มันจะปลดล็อกเมธอด `.rev()` ขึ้นมาให้เราใช้งานได้ฟรี ๆ ซึ่งจะช่วยห่อหุ้ม iterator เดิมเพื่อสร้างเป็น iterator ตัวใหม่ที่จะคายอิลิเมนต์ออกมาในลำดับย้อนศรให้เสร็จสรรพ โดยการทำงานของมันตรงไปตรงมามาก: ทุกครั้งที่เรียก `next` บน iterator ที่ถูก reversed มันก็จะแอบไปเรียก `next_back` ให้นั่นเอง

และเนื่องจากโครงสร้างเราเป็นเดกอยู่แล้ว การจัดเตรียม API นี้ให้จึงง่ายดายมาก:

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

งั้นมาลองเขียนเทสต์กันดูหน่อย:

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

แจ่มเลย

## Iter

ส่วน `Iter` นั้นจะไม่ปรานีเราเท่าไหร่นัก เพราะเราจำเป็นต้องกลับไปรับมือกับไอ้พวกวัตถุ `Ref` เจ้าปัญหานั่นอีกรอบ! และเพราะติดเรื่อง `Ref` เราจึงไม่สามารถเก็บ `&Node` ตรง ๆ ได้เหมือนในบทก่อน ๆ อีกต่อไป งั้นเราลองเปลี่ยนมาเก็บเป็น `Ref<Node>` ดูแทนดีกว่า:

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

จนถึงตรงนี้ยังดูไปได้สวย... การ implement `next` อาจจะชวนปวดเศียรเวียนเกล้าอยู่สักหน่อย แต่ผมคิดว่าลอจิกพื้นฐานมันก็น่าจะเหมือนเดิมกับ `IterMut` ของสแต็กเก่า เพียงแค่ผสมโรงความวิปลาสของ `RefCell` เพิ่มเข้าไป:

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

พับผ่าสิ

อายุขัยของ `node_ref` มันสั้นเกินไป! ต่างจากเรเฟอเรนซ์ปกติทั่วไป Rust ไม่ยอมให้เราตัดแบ่งชิ้นส่วน `Ref` แยกออกจากกันแบบนั้นได้ ค่า `Ref` ที่เราได้ออกมาจาก `head.borrow()` ถูกกำหนดให้อยู่ได้นานเท่ากับตัว `node_ref` เท่านั้น แต่เรากลับเผลอทำลายมันทิ้งไปในจังหวะที่เรียก `Ref::map` เสียแล้ว

ทว่าฟังก์ชันที่เราต้องการจริง ๆ มีอยู่แล้ว และมันมีชื่อว่า *[map_split][]*:

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

เฮือกกก... (Woof.) งั้นมาลองเสี่ยงกันดูสักตั้ง...

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

อืม... เราต้องสั่ง `Ref::map` อีกรอบเพื่อปรับแต่งไลฟ์ไทม์ให้ถูกต้อง แต่ `Ref::map` มันดันคืน `Ref` ออกมา ในขณะที่เราต้องการ `Option<Ref>` แต่เราก็ต้องมุดทะลุ `Ref` เข้าไปเพื่อ map ตัว `Option` ข้างในอีกที...

**(เหม่อมองออกไปในความเวิ้งว้างอันไกลโพ้นอยู่นานสองนาน)**

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

อ้อ... จริงด้วยสิ มันมี `RefCell` อยู่เต็มไปหมดเลยนี่หว่า! ยิ่งเราเดินลึกเข้าไปในลิสต์มากเท่าไหร่ ข้อมูลของเราก็ยิ่งถูกซ้อนทับอยู่ภายใต้เงาของ `RefCell` แต่ละตัวลึกขึ้นเรื่อย ๆ นั่นแปลว่าเราคงต้องเขียนระบบเพื่อคอยถือสแต็กของ `Ref` ทั้งหมดเอาไว้ เพื่อเป็นหลักฐานบันทึก "ยอดการยืมค้างชำระ (outstanding loans)" ทั้งหมดที่เราถืออยู่ เพราะเมื่อไหร่ก็ตามที่เราเลิกดูอิลิเมนต์สักตัว เราจำเป็นต้องตามไปลดตัวนับการยืม (borrow-count) บนทุก ๆ `RefCell` ที่เดินผ่านมาทั้งหมดก่อนหน้านี้ด้วย.........

ผมว่าตรงนี้ไม่มีทางไปต่อได้แล้วล่ะครับ มันคือทางตันสนิทอย่างแท้จริง! งั้นเราถอยทัพออกมาจากดง `RefCell` กันดีกว่า

แล้วถ้าเราหันมาใช้ `Rc` แทนล่ะ? ใครเป็นคนบอกกันนะว่า iterator จำเป็นต้องเก็บเป็นเรเฟอเรนซ์เสมอไป? ทำไมเราไม่สั่ง Clone ตัว `Rc` ทั้งก้อนออกมาเลยล่ะ เพื่อให้ได้ handle ที่ถือสิทธิ์ความเป็นเจ้าของชี้ตรงเข้าไปที่กลางลิสต์ได้แบบสบายใจเฉิบ?

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

เอ่อ... เดี๋ยวนะ แล้วตกลงตอนนี้เราจะคืนค่าอะไรออกไปล่ะ? `&T` เหรอ? หรือ `Ref<T>`?

ไม่เวิร์กสักกะอย่าง... `Iter` ของเราตอนนี้มันไม่มีไลฟ์ไทม์ผูกติดอยู่อีกต่อไปแล้ว! ทั้ง `&T` และ `Ref<T>` ล้วนบังคับให้เราต้องประกาศไลฟ์ไทม์เตรียมไว้ตั้งแต่แรกก่อนที่จะก้าวเท้าเข้าไปในเมธอด `next` แต่ไม่ว่าค่าอะไรก็ตามที่เราจะควักออกมาจาก `Rc` ได้ มันก็จะต้องเป็นการยืมตัว Iterator เองทั้งสิ้น... ปวด... กะโหลก... โว้ยยยย... อ๊ากกกกกกกก!

หรือว่าเราจะลอง... map ตัว `Rc`... เพื่อให้ได้ `Rc<T>` ออกมาแทน? มันมีท่าแบบนั้นอยู่จริงไหมนะ? ในคู่มือของ `Rc` ก็ดูเหมือนจะไม่มีฟังก์ชันทำนองนี้ให้ใช้เลย... แต่จริง ๆ แล้วมีคนใจดีทำ [crate ภายนอก][own-ref] ที่เปิดให้ทำแบบนั้นได้อยู่แฮะ

แต่ช้าก่อน... ต่อให้เราแก้ปัญหานั้นได้จริง เราก็ยังต้องเจอกับปัญหาที่ใหญ่ยิ่งกว่านั้นรออยู่อีก: นั่นคือวิญญาณร้ายตามหลอกหลอนที่เรียกว่า **Iterator Invalidation (การที่ตัววนซ้ำกลายเป็นโมฆะ)**! ที่ผ่านมา เราได้รับความคุ้มครองอย่างเบ็ดเสร็จจากปัญหานี้มาโดยตลอด เพราะตัว `Iter` จะทำการยืมตัวลิสต์เอาไว้ ส่งผลให้ลิสต์ตัวแม่กลายสภาพเป็น immutable โดยสมบูรณ์ แต่ถ้าหาก `Iter` ของเราคายค่าออกมาเป็น `Rc` ตัวมันก็จะไม่แตะต้องยืมตัวลิสต์เลยแม้แต่นิดเดียว! นั่นหมายความว่า ใครก็ตามสามารถสั่งเรียก `push` และ `pop` บนตัวลิสต์ได้ตามใจชอบ ทั้ง ๆ ที่ยังมีคนถือตัวชี้ชี้คาอยู่ข้างในลิสต์นั้น!

คุณพระช่วย... แล้วมันจะเกิดหายนะอะไรขึ้นล่ะนั่น?!

เอาจริง ๆ การเรียก `push` ก็ยังพอรับได้นะ เพราะเราถือแค่มุมมองย่อยของช่วงหนึ่งในลิสต์ แล้วลิสต์มันแค่งอกยาวเลยสายตาเราออกไป ก็ไม่ได้มีปัญหาอะไรคอขาดบาดตาย

แต่ทว่าการเรียก `pop` นี่สิคือหนังคนละม้วน! ถ้าพวกเขา pop อิลิเมนต์ที่อยู่นอกช่วงการมองเห็นของเรา มันก็ *ยังคง* พอถูไถไปได้ เพราะเรามองไม่เห็นโหนดเหล่านั้นอยู่แล้ว จึงไม่มีอะไรกระทบกระเทือน แต่ถ้าหากพวกเขาพยายามสั่ง pop โหนดที่เรากำลังถือตัวชี้ชี้คาตาอยู่ล่ะก็... บึ้มมมม! พังพินาศย่อยยับทันที! โดยเฉพาะอย่างยิ่งตอนที่โค้ดพยายามไปสั่ง `.unwrap()` ผลลัพธ์ที่ได้มาจาก `try_unwrap` มันจะล้มเหลวจริงจัง และส่งผลให้โปรแกรมทั้งตัวสั่ง panic พังพาบลงไปในบัดดล!

ซึ่งจริง ๆ มันก็โคตรเจ๋งเลยนะ: เราสามารถมีพอยน์เตอร์ที่ถือความเป็นเจ้าของอยู่ภายในลิสต์เพียบไปหมด แถมยังแก้ไขข้อมูลไปพร้อม ๆ กันได้ด้วย *และมันก็ยังทำงานได้ราบรื่น* จนกระทั่งมีคนพยายามจะลบโหนดที่เราถืออยู่ทิ้ง และถึงจะเป็นแบบนั้น เราก็ยังไม่เจอพวก dangling pointer เลยด้วยซ้ำ เพราะโปรแกรมจะสั่ง panic ตายอย่างเด็ดขาดและคาดเดาได้เป๊ะ ๆ (deterministically panic)!

แต่การต้องมานั่งรับมือกับปัญหา Iterator Invalidation บวกกับความปวดหัวในการ map `Rc` มันดูจะ... ทรมานสังขารเกินไปหน่อย ในที่สุดคู่หู `Rc<RefCell>` ก็ทำให้เราผิดหวังเข้าจนได้จริง ๆ และที่น่าสนใจก็คือ เรากำลังเผชิญกับสถานการณ์ที่เป็นขั้วตรงข้ามกับสแต็กแบบเพอร์ซิสเทนต์อย่างสิ้นเชิง: ในขณะที่สแต็กแบบเพอร์ซิสเทนต์ต้องดิ้นรนแทบตายในการทวงคืนสิทธิ์ความเป็นเจ้าของ (ownership) แต่กลับแจกจ่ายเรเฟอเรนซ์ให้ทุกคนได้ทั้งวันทั้งคืน ทว่าในเดกตัวนี้ของเรา เรากลับไม่มีปัญหาเรื่องการแย่งสิทธิ์ความเป็นเจ้าของเลย แต่กลับต้องดิ้นรนเลือดตากระเด็นเพียงเพื่อจะขอยืมเรเฟอเรนซ์สักตัวออกมา!

ถึงแม้ว่าถ้าจะพูดให้ยุติธรรม ความยากลำบากส่วนใหญ่ของเรามันเกิดจากการที่เราดื้อดึงอยากจะซ่อนรายละเอียดเบื้องหลัง (encapsulation) และอยากได้ API ที่สวยงามน่าใช้ต่างหาก เพราะถ้าเรายอมหน้าด้านโยนตัว `Node` วิ่งไปวิ่งมาให้ผู้ใช้เห็นตรง ๆ เราก็คงจะเขียนทุกอย่างผ่านฉลุยไปได้นานแล้ว

ให้ตายสิ เราสามารถสร้าง `IterMut` ขึ้นมาพร้อมกันหลาย ๆ ตัวที่คอยตรวจสอบตอนรันไทม์ว่าไม่มีตัวไหนกำลังแก้ไขข้อมูลที่โหนดเดียวกันได้เลยด้วยซ้ำ!

เอาเข้าจริง การออกแบบสถาปัตยกรรมแบบนี้มันเหมาะกับโครงสร้างข้อมูลภายใน (internal data structure) ที่ไม่ถูกเปิดเผยออกมาให้ผู้ใช้ API ภายนอกมองเห็นเสียมากกว่า กลไก Interior Mutability นั้นยอดเยี่ยมมากสำหรับการเขียน *Application* ให้ปลอดภัย แต่กลับไม่ค่อยเหมาะกับการเขียน *Library* ให้ปลอดภัยสักเท่าไหร่นัก

เอาเป็นว่า บทนี้ผมขอยกธงขาวขอยอมแพ้ให้กับ `Iter` และ `IterMut` ก็แล้วกันครับ จะฝืนเขียนต่อก็คงทำได้แหละ... แต่มันไม่ไหวจะเคลียร์จริง ๆ!

[own-ref]: https://crates.io/crates/owning_ref
[map-split]: https://doc.rust-lang.org/std/cell/struct.Ref.html#method.map_split
