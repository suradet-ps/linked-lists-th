# IterMut

ผมจะพูดตรง ๆ IterMut นั้นบ้าคลั่ง ซึ่งตัวมันเองก็ดูเหมือนคำพูดที่บ้าคลั่ง
แน่นอนว่ามันต้องเหมือนกับ Iter!

ในแง่ของความหมาย ใช่ แต่ธรรมชาติของเรเฟอเรนซ์ร่วมและเรเฟอเรนซ์ที่เปลี่ยนแปลงได้
หมายความว่า Iter เป็น "เรื่องง่าย" แต่ IterMut เป็นเวทย์มนตร์ของพ่อมดจริงจัง

ข้อคิดสำคัญมาจากการ implement Iterator สำหรับ Iter ของเรา:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> { /* stuff */ }
}
```

ซึ่งสามารถลดรูปเป็น:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next<'b>(&'b mut self) -> Option<&'a T> { /* stuff */ }
}
```

ลายเซ็นของ `next` ไม่ได้สร้างข้อจำกัด*ใด ๆ* ระหว่างไลฟ์ไทม์ของ
อินพุตและเอาต์พุต! ทำไมเราถึงแคร์? มันหมายความว่าเราสามารถเรียก `next`
ซ้ำไปเรื่อย ๆ ได้อย่างไม่มีเงื่อนไข!

```rust ,ignore
let mut list = List::new();
list.push(1); list.push(2); list.push(3);

let mut iter = list.iter();
let x = iter.next().unwrap();
let y = iter.next().unwrap();
let z = iter.next().unwrap();
```

เจ๋ง!

สิ่งนี้*ไม่เป็นไรแน่นอน*สำหรับเรเฟอเรนซ์ร่วมเพราะจุดประสงค์ทั้งหมดคือ
คุณสามารถมีจำนวนมากพร้อมกันได้ อย่างไรก็ตามเรเฟอเรนซ์ที่เปลี่ยนแปลงได้
*ไม่สามารถ*อยู่ร่วมกันได้ จุดประสงค์ทั้งหมดคือมันเป็นเอกสิทธิ์

ผลลัพธ์สุดท้ายคือมันยากขึ้นอย่างเห็นได้ชัดในการเขียน IterMut ด้วยโค้ดแบบ
ปลอดภัย (และเรายังไม่ได้ลงลึกว่ามันหมายความว่าอะไร...) อย่างน่าตกใจ
IterMut สามารถ implement ได้สำหรับโครงสร้างหลายชนิดอย่างสมบูรณ์แบบ
ปลอดภัย!

เราจะเริ่มด้วยการนำโค้ด Iter มาแล้วเปลี่ยนทุกอย่างเป็นแบบที่เปลี่ยนแปลงได้:

```rust ,ignore
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}
```

```text
> cargo build
error[E0596]: cannot borrow `self.head` as mutable, as it is behind a `&` reference
  --> src/second.rs:95:25
   |
94 |     pub fn iter_mut(&self) -> IterMut<'_, T> {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
95 |         IterMut { next: self.head.as_deref_mut() }
   |                         ^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error[E0507]: cannot move out of borrowed content
   --> src/second.rs:103:9
    |
103 |         self.next.map(|node| {
    |         ^^^^^^^^^ cannot move out of borrowed content
```

โอเค ดูเหมือนเรามีสองข้อผิดพลาดต่างกันที่นี่ ข้อแรกชัดเจนมาก
มันยังบอกเราด้วยว่าต้องแก้ยังไง! คุณไม่สามารถอัพเกรดเรเฟอเรนซ์ร่วมเป็น
เรเฟอเรนซ์ที่เปลี่ยนแปลงได้ ดังนั้น `iter_mut` ต้องรับ `&mut self`
แค่ข้อผิดพลาดจากการก๊อปแปะที่โง่ ๆ

```rust ,ignore
pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut { next: self.head.as_deref_mut() }
}
```

แล้วอีกอันล่ะ?

โอ๊ะ! จริง ๆ แล้วผมทำผิดพลาดโดยบังเอิญตอนเขียน implement ของ `iter`
ในส่วนก่อนหน้า และเราแค่โชคดีที่มันทำงานได้!

เราเพิ่งได้เจอกับเวทมนตร์ของ Copy ครั้งแรก ตอนที่เราแนะนำ
[ความเป็นเจ้าของ][ownership] เราบอกว่าเมื่อคุณย้ายของ คุณจะใช้มันไม่ได้อีกต่อไป
สำหรับบางชนิด สิ่งนี้สมเหตุสมผล เพื่อนที่ดีของเรา Box จัดการการจัดสรร
บนฮีปสำหรับเรา และเราไม่ต้องการให้โค้ดสองชิ้นคิดว่าพวกเขาต้อง
ปลดปล่อยหน่วยความจำของมัน

อย่างไรก็ตาม สำหรับชนิดอื่น สิ่งนี้*ไร้สาระ* จำนวนเต็มไม่มี
ความหมายเรื่องความเป็นเจ้าของ มันแค่ตัวเลขที่ไม่มีความหมาย!
นี่คือเหตุผลที่จำนวนเต็มถูกทำเครื่องหมายว่าเป็น Copy ชนิด Copy เป็นที่รู้กันว่า
สามารถคัดลอกได้อย่างสมบูรณ์แบบด้วยการคัดลอกแบบไบต์ต่อไบต์
ดังนั้นมันมีพลังวิเศษ: เมื่อย้าย ค่าเก่ายัง*ใช้ได้อยู่*
ผลที่ตามมา คุณยังสามารถย้ายชนิด Copy ออกจากเรเฟอเรนซ์ได้โดยไม่ต้อง
แทนที่ด้วยอะไร!

จำนวนเต็มทุกชนิดใน Rust (i32, u64, bool, f32, char ฯลฯ) เป็น Copy
คุณยังสามารถประกาศชนิดที่ผู้ใช้กำหนดเองให้เป็น Copy ได้เช่นกัน ตราบใดที่
ส่วนประกอบทั้งหมดของมันเป็น Copy

สำคัญอย่างยิ่งต่อเหตุผลที่โค้ดนี้ทำงานได้ เรเฟอเรนซ์ร่วมก็เป็น Copy เช่นกัน!
เพราะ `&` เป็น copy `Option<&>` *ก็*เป็น Copy ด้วย ดังนั้นเมื่อเราทำ `self.next.map`
มันไม่เป็นไรเพราะ Option ถูกคัดลอก ตอนนี้เราทำไม่ได้แล้ว เพราะ
`&mut` ไม่ใช่ Copy (ถ้าคุณ copy &mut คุณจะมี &mut สองตัวชี้ไปยังตำแหน่งเดียวกัน
ในหน่วยความจำ ซึ่งห้าม) แทนที่เราจะต้อง `take` Option ออกมาอย่างถูกต้อง:

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
        self.next = node.next.as_deref_mut();
        &mut node.elem
    })
}
```

```text
> cargo build

```

เอ่อ... ว้าว พระเจ้า! IterMut ใช้ได้จริง!

มาทดสอบกัน:

```rust ,ignore
#[test]
fn iter_mut() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter_mut();
    assert_eq!(iter.next(), Some(&mut 3));
    assert_eq!(iter.next(), Some(&mut 2));
    assert_eq!(iter.next(), Some(&mut 1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 6 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

```

ใช่ มันใช้ได้

พระเจ้า

อะไรนะ

โอเค ผมหมายความว่ามันจริง ๆ แล้ว*ควร*จะใช้ได้ แต่มักจะมีอะไรโง่ ๆ
ขวางอยู่! ให้ชัดเจนตรงนี้:

เราเพิ่ง implement โค้ดชิ้นหนึ่งที่รับลิงก์ลิสต์แบบซิงกลี
และคืนเรเฟอเรนซ์ที่เปลี่ยนแปลงได้ไปยังทุกองค์ประกอบในลิสต์อย่างน้อยหนึ่งครั้ง
และมันถูกตรวจสอบแบบ static ว่าทำแบบนั้น และมันปลอดภัยอย่างสมบูรณ์
และเราไม่ต้องทำอะไรบ้าคลั่งเลย

นี่เป็นเรื่องใหญ่ ถ้าถามผม มีเหตุผลหลายประการที่ทำไมมันถึงทำงานได้:

* เรา `take` `Option<&mut>` ออกมเพื่อให้เราเข้าถึงเรเฟอเรนซ์ที่เปลี่ยนแปลงได้แบบเอกสิทธิ์
  ไม่ต้องกังวลว่าจะมีคนมาดูมันอีก
* Rust เข้าใจว่ามันโอเคที่จะแบ่งเรเฟอเรนซ์ที่เปลี่ยนแปลงได้เป็นฟิลด์ย่อยของ
  struct ที่ชี้ถึง เพราะไม่มีทาง "ย้อนกลับ" และมันแน่นอนว่าไม่ทับกัน

ปรากฎว่าคุณสามารถใช้ตรรกะพื้นฐานนี้เพื่อให้ IterMut แบบปลอดภัยสำหรับ
อาร์เรย์หรือต้นไม้ได้เช่นกัน! คุณยังสามารถทำให้อิเทอเรเตอร์เป็น DoubleEnded
ได้ เพื่อให้คุณสามารถใช้อิเทอเรเตอร์จากด้านหน้า*และ*ด้านหลังพร้อมกัน! ว้าว!

[ownership]: first-ownership.md