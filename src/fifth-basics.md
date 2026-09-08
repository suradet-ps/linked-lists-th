# พื้นฐาน

> **ผู้บรรยาย:** ส่วนนี้มีข้อผิดพลาดพื้นฐานที่กำลังจะเกิดขึ้น เพราะนั่นคือ
> จุดประสงค์ทั้งหมดของหนังสือ อย่างไรก็ตามเมื่อเราเริ่มใช้ `unsafe` มันเป็นไปได้ที่จะทำผิด
> และยังคงคอมไพล์และ*ดูเหมือน*ทำงานได้ ข้อผิดพลาดพื้นฐานจะถูกระบุในส่วนถัดไป
> อย่าใช้เนื้อหาของส่วนนี้ในโค้ด production!

เอาล่ะ กลับสู่พื้นฐาน เราจะสร้างลิสต์อย่างไร?

ก่อนหน้าเราแค่ทำ:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

แต่เราไม่ได้ใช้ Option สำหรับ `tail` อีกแล้ว:

```text
> cargo build

error[E0308]: mismatched types
  --> src/fifth.rs:15:34
   |
15 |         List { head: None, tail: None }
   |                                  ^^^^ expected *-ptr, found 
   |                                       enum `std::option::Option`
   |
   = note: expected type `*mut fifth::Node<T>`
              found type `std::option::Option<_>`
```

เรา*สามารถ*ใช้ Option แต่ต่างจาก Box `*mut` *เป็น* null ได้ นั่นหมายความว่า
มันไม่ได้รับประโยชน์จากการเพิ่มประสิทธิภาพด้วยพอยน์เตอร์ null แทนที่เราจะใช้ `null`
เพื่อเป็นตัวแทน None

แล้วเราจะได้ null pointer อย่างไร? มีหลายวิธี แต่ผมชอบใช้
`std::ptr::null_mut()` ถ้าคุณต้องการ คุณยังใช้ `0 as *mut _` ได้ แต่
ดูเหมือน*ยุ่งเหยิง*เกินไป

```rust ,ignore
use std::ptr;

// defns...

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: ptr::null_mut() }
    }
}
```

```text
cargo build

warning: field is never used: `head`
 --> src/fifth.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fifth.rs:5:5
  |
5 |     tail: *mut Node<T>,
  |     ^^^^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `head`
  --> src/fifth.rs:12:5
   |
12 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
```

*เงียบ*คอมไพเลอร์ เราจะใช้พวกมันเร็วๆ นี้

เอาล่ะ มาเขียน `push` อีกครั้ง คราวนี้แทนที่จะคว้า
`Option<&mut Node<T>>` หลังจาก insert เราจะคว้า
`*mut Node<T>` ไปยังส่วนภายในของ Box ทันที เรารู้ว่าเราสามารถทำได้
เพราะเนื้อหาของ Box มีที่อยู่ที่เสถียร แม้ว่าเราจะย้าย
Box ไปรอบๆ แน่นอน นี่ไม่*ปลอดภัย* เพราะถ้าเราแค่ drop Box เราจะ
มีตัวชี้ไปยังหน่วยความจำที่ถูกปลดปล่อย

เราจะสร้างตัวชี้ดิบจากตัวชี้ปกติอย่างไร? Coercions! ถ้าตัวแปร
ถูกประกาศเป็นตัวชี้ดิบ เรเฟอเรนซ์ปกติจะ coerce เข้าไปในนั้น:

```rust ,ignore
let raw_tail: *mut _ = &mut *new_tail;
```

เรามีข้อมูลทั้งหมดที่ต้องการ เราสามารถแปลงโค้ดของเราเป็น เกือบ
เวอร์ชันเรเฟอเรนซ์ก่อนหน้า:

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // .is_null checks for null, equivalent to checking for None
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        self.tail.next = Some(new_tail);
    } else {
        // Otherwise, update the head to point to it
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             self.tail.next = Some(new_tail);
   |             ----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; 
   |             try dereferencing it: `(*self.tail).next`
```

ฮะ? เรามีตัวชี้ไปยัง Node ทำไมเราถึงเข้าถึงฟิลด์ `next` ไม่ได้?

Rust ค่อนข้างจะ jerk เมื่อคุณใช้ตัวชี้ดิบ เพื่อเข้าถึงเนื้อหาของ
ตัวชี้ดิบ มันยืนยันว่าเราต้อง dereference ด้วยตัวเอง เพราะมันเป็นการดำเนินงาน
ที่ไม่ปลอดภัยมาก ดังนั้นมาทำแบบนั้น:

```rust ,ignore
*self.tail.next = Some(new_tail);
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             *self.tail.next = Some(new_tail);
   |             -----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; 
   |             try dereferencing it: `(*self.tail).next`
```

อุ๊ก operator precedence

```rust ,ignore
(*self.tail).next = Some(new_tail);
```

```text
> cargo build

error[E0133]: dereference of raw pointer is unsafe and requires 
              unsafe function or block

  --> src/fifth.rs:31:13
   |
31 |             (*self.tail).next = Some(new_tail);
   |             ^^^^^^^^^^^^^^^^^ dereference of raw pointer
   |
   = note: raw pointers may be NULL, dangling or unaligned; 
     they can violate aliasing rules and cause data races: 
     all of these are undefined behavior
```

นี่. ไม่ควร. ยาก. ขนาดนี้

จำไหมว่าผมบอกว่า Unsafe Rust เหมือน FFI language สำหรับ Safe Rust?
컴ไพเลอร์ต้องการให้เราบอกอย่างชัดเจนว่าเรากำลังทำ FFI ที่ไหน เรามี
สองทางเลือก ประการแรก เราสามารถระบุฟังก์ชัน*ทั้งหมด*เป็น unsafe ซึ่งจะ
ทำให้มันเป็นฟังก์ชัน Unsafe Rust และสามารถเรียกได้เฉพาะในบริบท `unsafe`
เท่านั้น นี่ไม่ดี เพราะเราต้องการให้ลิสต์ของเราปลอดภัยที่จะใช้ ประการที่สอง
เราสามารถเขียนบล็อก `unsafe` ภายในฟังก์ชัน เพื่อบอกขอบเขต FFI
นี่จะประกาศว่าฟังก์ชันทั้งหมดปลอดภัย มาทำแบบหลัง:


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    if !self.tail.is_null() {
        // Hello Compiler, I Know I Am Doing Something Dangerous And
        // I Promise To Be A Good Programmer Who Never Makes Mistakes.
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build
warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

ย-ย!

ค่อนข้างน่าสนใจว่านี่เป็น*ที่เดียว*ที่เราต้องเขียนบล็อก unsafe
จนถึงตอนนี้ เราทำสิ่งตัวชี้ดิบทุกที่ เป็นอะไร?

ปรากฏว่า Rust เป็นกฎข้อบังคับที่massive เมื่อพูดถึง
`unsafe` เราต้องการเพิ่มประสิทธิภาพชุดโปรแกรม Safe Rust ให้มากที่สุด
เพราะเป็นโปรแกรมที่เรามั่นใจได้มากกว่า เพื่อทำเช่นนั้น
Rust ค่อยๆ ตัดพื้นผิวที่ minimal สำหรับความไม่ปลอดภัย หมายเหตุว่าทุก
สถานที่อื่นที่เราทำงานกับตัวชี้ดิบคือการ *assigning* หรือ
แค่สังเกตว่ามันเป็น null หรือไม่

ถ้าคุณไม่ dereference ตัวชี้ดิบจริงๆ *สิ่งเหล่านั้นปลอดภัยทั้งหมด
ที่จะทำ* คุณแค่อ่านและเขียนจำนวนเต็ม! ครั้งเดียวที่คุณสามารถ
เข้าไปในปัญหาได้จริงๆ กับตัวชี้ดิบคือถ้าคุณ dereference มันจริงๆ
ดังนั้น Rust บอกว่า*เฉพาะ*การกระทำนั้นเท่านั้นที่ไม่ปลอดภัย และทุกอย่างอื่นปลอดภัย

Super. Pedantic. แต่ทางเทคนิคถูกต้อง

> **ผู้บรรยาย:** ที่ไหนสักแห่งอีกฟากของโลก วิศวกรฮาร์ดแวร์
> รู้สึกหนาวสั่น down หลังของเธอ &mdash; บางคนต้องยืนยันว่าตัวชี้
> เป็นแค่จำนวนเต็มอีกครั้ง เธอมองลงมาที่ข้อเสนอของเธอสำหรับฮาร์ดแวร์ใหม่
> การรับรองความถูกต้องของตัวชี้และหลั่งน้ำตาหนึ่งหยด วิศวกรคอมไพเลอร์
> ข้างๆ ไม่รู้สึกอะไร &mdash; พวกเขาเรียนรู้นานแล้วว่าต้องสวมเสื้อกันหนาวหนัก
> เสมอ

มีแค่บางส่วนของการดำเนินงานตัวชี้ที่*จริงๆ* ไม่ปลอดภัยทำให้เกิด
ปัญหาที่น่าสนใจ: แม้ว่าเราควรบอกขอบเขตของความไม่ปลอดภัย
ด้วยบล็อก `unsafe` มันจริงๆ แล้วขึ้นอยู่กับสถานะที่
ตั้งอยู่นอกบล็อก นอกฟังก์ชัน ด้วยซ้ำ!

นี่คือสิ่งที่ผมเรียกว่า unsafe *taint* ทันทีที่คุณใช้ `unsafe` ในโมดูล
โมดูลทั้งหมดจะเปื้อนด้วยความไม่ปลอดภัย ทุกอย่างต้องถูกเขียนอย่างถูกต้อง
เพื่อให้แน่ใจว่า invariant ทั้งหมดถูกปฏิบัติตามสำหรับโค้ด unsafe

taint นี้จัดการได้เพราะ*ความเป็นส่วนตัว* นอกโมดูลของเรา ทุก
ฟิลด์ของ struct เป็นส่วนตัวทั้งหมด ดังนั้นไม่มีใครอื่นสามารถแก้ไขสถานะของเราได้
แบบ arbitrily ตราบเท่าที่ไม่มีการผสมผสานของ API ที่เราเปิดเผยทำให้เกิดสิ่งแย่
เกิดขึ้น เท่าที่ผู้สังเกตภายนอกเป็นห่วง โค้ดทั้งหมดของเราปลอดภัย!
และจริงๆ นี่ไม่ต่างจากกรณี FFI ไม่มีใครต้องสน
ว่าคลังคณิตศาสตร์ Python บางตัว shell ไป C ตราบเท่าที่มันเปิดเผย interface ที่ปลอดภัย

อย่างไรก็ตาม มาต่อด้วย `pop` ซึ่งเกือบจะ verbatim เวอร์ชัน
เรเฟอเรนซ์:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        if self.head.is_none() {
            self.tail = ptr::null_mut();
        }

        head.elem
    })
}
```

อีกครั้งเราเห็นกรณีที่ความปลอดภัยเป็นแบบ stateful ถ้าเราล้มเหลวที่จะ null out
ตัวชี้ tail ในฟังก์ชัน*นี้*เราจะไม่เห็นปัญหาใดๆ อย่างไรก็ตาม
การเรียก `push` ในภายหลังจะเริ่มเขียนไปยัง tail ที่ลอย!

มาทดสอบมัน:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

นี่คือเทสต์สแต็ก แต่ด้วยผลลัพธ์ `pop` ที่คาดหวังกลับกัน
ผมยังเพิ่มขั้นตอนเพิ่มเติมที่ปลายเพื่อให้แน่ใจว่ากรณี tail-pointer
corruption ใน `pop` ไม่เกิดขึ้น

```text
cargo test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

ทองดาว!

> **ผู้บรรยาย:** นี่มันมาแล้ว...
