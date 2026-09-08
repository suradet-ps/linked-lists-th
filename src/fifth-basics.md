# พื้นฐาน

> **ผู้บรรยาย:** ส่วนนี้มีข้อผิดพลาดระดับรากฐานที่กำลังรอปะทุอยู่ เพราะนั่นคือจุดประสงค์หลักของหนังสือเล่มนี้ ทว่า ทันทีที่เราเริ่มก้าวเข้าสู่ดินแดนของ `unsafe` มันเป็นไปได้ที่เราจะเขียนโค้ดผิดแต่คอมไพเลอร์ยังยอมให้ผ่าน แถมโค้ดยัง*ดูเหมือนจะ*ทำงานได้ถูกต้องอีกต่างหาก ข้อผิดพลาดร้ายแรงนี้จะถูกเฉลยในหัวข้อถัดไป อย่าได้ริอ่านนำเนื้อหาในส่วนนี้ไปใช้บนโค้ด production เป็นอันขาด!

เอาล่ะ กลับสู่พื้นฐานกันก่อน เราจะสร้างลิสต์ของเราขึ้นมาอย่างไร?

ก่อนหน้านี้เราเคยเขียนแบบนี้:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

แต่ตอนนี้เราไม่ได้ใช้ `Option` สำหรับ `tail` อีกต่อไปแล้ว:

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

เรา*จะ*ใช้ `Option` ก็ได้นะ แต่ต่างจาก `Box` ตรงที่ตัว `*mut` เองมัน*สามารถเป็น null ได้*ในตัวอยู่แล้ว นั่นแปลว่ามันจะไม่ได้รับอานิสงส์จาก null pointer optimization ดังนั้น เราจึงจะใช้ค่า `null` ในการแทนสถานะ `None` ไปเลย

แล้วเราจะเสก null pointer ขึ้นมาได้อย่างไร? มีอยู่สองสามวิธีครับ แต่ผมชอบใช้ `std::ptr::null_mut()` มากที่สุด ถ้าคุณอยากจะอินดี้ คุณจะใช้ `0 as *mut _` ก็ได้เหมือนกัน แต่มันดู*เละเทะ*ไปหน่อย

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

*เงียบไปเลย* เจ้าคอมไพเลอร์ เดี๋ยวเราก็ได้ใช้พวกมันแล้วน่า!

เอาล่ะ มาลองเขียน `push` กันใหม่อีกรอบ คราวนี้แทนที่เราจะดึง `Option<&mut Node<T>>` ออกมาหลังจากที่ insert เสร็จ เราจะคว้าเอา `*mut Node<T>` ที่ชี้ตรงไปยังเนื้อในของ `Box` ทันที เรารู้ว่าเราทำแบบนี้ได้เพราะข้อมูลที่อยู่ใน `Box` จะมี memory address ที่เสถียรเสมอ แม้ว่าเราจะย้ายกล่อง `Box` นั้นไปไว้ที่อื่นก็ตาม แน่นอนว่าวิธีนี้*ไม่ปลอดภัย (unsafe)* เพราะถ้าเกิดเราเผลอ drop กล่อง `Box` นั้นทิ้งไป เราก็จะมีพอยน์เตอร์ที่ชี้ไปยังหน่วยความจำที่ถูก free ไปแล้วทันที

แล้วเราจะแปลงจากเรเฟอเรนซ์ปกติให้กลายเป็น raw pointer ได้อย่างไรล่ะ? การแปลงชนิดข้อมูลอัตโนมัติ (Coercion) ไงล่ะ! ถ้าเราประกาศชนิดของตัวแปรปลายทางไว้ชัดเจนว่าเป็น raw pointer ตัวแปร reference ปกติจะยอม coerce กลายร่างเป็น raw pointer ให้เราเองทันที:

```rust ,ignore
let raw_tail: *mut _ = &mut *new_tail;
```

ตอนนี้เรามีข้อมูลทุกอย่างที่ต้องการครบแล้ว เราสามารถแปลงโค้ดของเราให้กลับมาใกล้เคียงกับเวอร์ชันที่ใช้ reference ก่อนหน้านี้ได้เลย:

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

หือ? เราก็มีพอยน์เตอร์ชี้ไปยัง Node อยู่แล้วนี่ ทำไมถึงเข้าไปแตะฟิลด์ `next` ไม่ได้ล่ะ?

Rust มักจะทำตัวกวนประสาทเสมอเวลาเราใช้ raw pointer พอเราจะเข้าถึงข้อมูลที่ raw pointer ชี้อยู่ มันจะบังคับให้เราต้อง dereference เองแบบ manual เสมอ เพราะมันถือว่านี่คือ operation ที่อันตรายมาก งั้นก็จัดไปตามนั้น:

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

โอ๊ยยย... ลำดับความสำคัญของตัวดำเนินการ (operator precedence) นี่มันน่ารำคาญจริงๆ!

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

มัน. ไม่. ควร. จะ. ยาก. ขนาด. นี้. ไหม. วะ!

จำได้ไหมที่ผมเคยบอกว่า Unsafe Rust เปรียบเหมือนภาษา FFI สำหรับ Safe Rust? เอาล่ะ คอมไพเลอร์อยากให้เราขีดเส้นแบ่งให้ชัดเจนว่าเรากำลังทำตัวเป็น FFI อยู่ตรงไหนบ้าง ซึ่งเรามี 2 ทางเลือก ทางแรกคือเราสามารถแปะป้ายกำกับฟังก์ชัน*ทั้งฟังก์ชัน*ว่าเป็น `unsafe` ไปเลย ซึ่งจะทำให้มันกลายเป็นฟังก์ชัน Unsafe Rust ทันที และจะถูกเรียกใช้ได้เฉพาะในบริบทที่เป็น `unsafe` เท่านั้น แต่วิธีนี้ไม่เวิร์ก เพราะเราต้องการให้คนอื่นเอาลิสต์ของเราไปใช้งานได้อย่างปลอดภัย (safe) ส่วนทางเลือกที่สองคือ เราสามารถเปิดบล็อก `unsafe` ข้างในตัวฟังก์ชัน เพื่อขีดเส้นแบ่งเขตแดน FFI เฉพาะจุดเอาไว้ การทำแบบนี้จะเป็นการประกาศว่าภาพรวมของฟังก์ชันนั้นยังคงปลอดภัยอยู่ งั้นเรามาใช้วิธีนี้กัน:


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

เย้!

น่าแปลกใจดีเหมือนกันนะ ที่นั่นเป็นจุด*เพียงจุดเดียว*ที่เราต้องเขียนบล็อก `unsafe` จนถึงตอนนี้ ทั้งๆ ที่เราเอา raw pointer ไปล่อเป้าไว้ทั่วบ้านทั่วเมือง มันเกิดอะไรขึ้นกันแน่?

ปรากฏว่า Rust เป็นพวกยึดถือกฎระเบียบอย่างเคร่งครัดและจู้จี้จุกจิกขั้นสุดเมื่อเป็นเรื่องของ `unsafe` ในมุมมองที่สมเหตุสมผล เราย่อมอยากขยายขอบเขตของโค้ด Safe Rust ให้กว้างที่สุดเท่าที่จะทำได้ เพราะนั่นคือส่วนของโปรแกรมที่เรามั่นใจในความถูกต้องได้มากที่สุด และเพื่อที่จะทำแบบนั้นได้ Rust จึงต้องขีดวงจำกัดพื้นผิวของความไม่ปลอดภัย (unsafe surface area) ให้เหลือน้อยที่สุดเท่าที่จะเป็นไปได้ สังเกตดูสิครับว่าจุดอื่นๆ ทั้งหมดที่เราเล่นกับ raw pointer นั้น มีแค่การ*กำหนดค่า (assign)* ให้มัน หรือไม่ก็แค่คอยส่องดูว่าค่ามันเป็น null หรือเปล่าเท่านั้นเอง

ถ้าคุณไม่ได้ลงมือ dereference เพื่อเปิดดูไส้ในของ raw pointer จริงๆ *การกระทำเหล่านั้นล้วนปลอดภัย 100% ทั้งสิ้น* เพราะคุณก็แค่กำลังอ่านหรือเขียนตัวเลขจำนวนเต็มธรรมดาๆ เท่านั้นเอง! จุดเดียวที่ raw pointer จะสร้างปัญหาหายนะให้คุณได้จริงๆ ก็คือตอนที่คุณ dereference มัน ดังนั้น Rust จึงบอกว่า มี*เฉพาะ* operation นั้นแหละที่ไม่ปลอดภัย ส่วนเรื่องอื่นๆ ถือว่าปลอดภัยหายห่วงทั้งหมด

จู้จี้. หยุมหยิม. สุดๆ. แต่ในทางเทคนิคมันถูกต้องเป๊ะ

> **ผู้บรรยาย:** ณ มุมหนึ่งของโลก วิศวกรฮาร์ดแวร์สาวคนหนึ่งรู้สึกเสียวสันหลังวาบขึ้นมา &mdash; ต้องมีใครสักคนในโลกกำลังทึกทักเอาเองว่าพอยน์เตอร์เป็นแค่ตัวเลขจำนวนเต็มอีกแล้วแน่ๆ เธอก้มมองเอกสารข้อเสนอสถาปัตยกรรมฮาร์ดแวร์ใหม่สำหรับการยืนยันความถูกต้องของพอยน์เตอร์ (pointer authentication) แล้วแอบปาดน้ำตาไปหนึ่งหยด ส่วนวิศวกรคอมไพเลอร์ที่นั่งข้างๆ กลับไม่สะทกสะท้านอะไรเลย &mdash; เพราะพวกเขาเรียนรู้มานานแล้วว่าจงใส่เสื้อกันหนาวหนาๆ ไว้เสมอ

การที่มี operation ของพอยน์เตอร์เพียงบางตัวเท่านั้นที่ถือว่าไม่ปลอดภัย*อย่างแท้จริง* ก่อให้เกิดปัญหาที่น่าคิดขึ้นมาข้อหนึ่ง: แม้ว่าเราจะพยายามจำกัดขอบเขตของความไม่ปลอดภัยไว้ด้วยบล็อก `unsafe` แล้วก็ตาม แต่ในความเป็นจริงแล้ว ความปลอดภัยของมันกลับต้องพึ่งพาสถานะที่ถูกตระเตรียมไว้จากนอกบล็อกนั้น... เผลอๆ อยู่นอกฟังก์ชันนั้นด้วยซ้ำไป!

นี่คือสิ่งที่ผมเรียกว่า *มลทินแห่งความไม่ปลอดภัย (unsafe taint)* ทันทีที่คุณเริ่มใช้ `unsafe` ในโมดูลใดโมดูลหนึ่ง ทั้งโมดูลนั้นจะแปดเปื้อนไปด้วยความไม่ปลอดภัยทันที ทุกสิ่งทุกอย่างในโมดูลนั้นจะต้องถูกเขียนขึ้นมาอย่างถูกต้องรัดกุมที่สุด เพื่อให้มั่นใจได้ว่า invariants (เงื่อนไขความถูกต้อง) ทั้งหมดจะยังคงอยู่ครบถ้วนสำหรับโค้ด unsafe นั้น

แต่เรายังพอบริหารจัดการมลทินนี้ได้อยู่ ต้องขอบคุณระบบ *privacy (การจำกัดสิทธิ์การเข้าถึง)* เพราะเมื่อมองจากภายนอกโมดูลของเรา ฟิลด์ทั้งหมดใน struct ล้วนเป็น private ทั้งสิ้น ทำให้ไม่มีใครหน้าไหนสามารถแอบมายุ่มย่ามแก้ไขสถานะภายในของเราตามอำเภอใจได้ ตราบใดที่ API ภายนอกที่เราเปิดให้เรียกใช้ไม่ได้พาไปสู่เรื่องพินาศ ในมุมมองของผู้ใช้งานภายนอก โค้ดทั้งหมดของเราก็ยังคงปลอดภัยสมบูรณ์แบบ! และเอาเข้าจริง เรื่องนี้ก็ไม่ได้ต่างอะไรกับเคสของ FFI เลย ไม่มีผู้ใช้คนไหนต้องมานั่งสนใจหรอกว่าไลบรารีคณิตศาสตร์ใน Python ตัวนั้นจะแอบมุดไปเรียกภาษา C เบื้องหลังหรือไม่ ตราบใดที่มันเปิด interface ที่ปลอดภัยให้เราใช้งาน

เอาล่ะ มาต่อกันที่ `pop` ซึ่งแทบจะถอดแบบมาจากเวอร์ชัน reference แทบทุกกระเบียดนิ้ว:

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

นี่เป็นอีกตัวอย่างหนึ่งที่แสดงให้เห็นว่าความปลอดภัยมีลักษณะขึ้นกับสถานะ (stateful) ถ้าเราลืมเซ็ต tail pointer ให้เป็น null ในฟังก์ชัน*นี้* ตัวฟังก์ชันนี้เองจะไม่มีปัญหาอะไรเลยแม้แต่น้อย ทว่า การเรียก `push` ในครั้งต่อๆ ไปต่างหากที่จะเริ่มไปเขียนทับลงบน dangling tail!

มาทดสอบกันดูดีกว่า:

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

นี่ก็คือเทสต์ชุดเดิมจากสแต็ก แต่สลับผลลัพธ์ของ `pop` ที่คาดหวังให้ถูกต้องตามลำดับของคิว นอกจากนี้ผมยังเพิ่มขั้นตอนต่อท้ายอีกนิดหน่อย เพื่อให้มั่นใจว่ากรณี tail-pointer corruption ใน `pop` จะไม่เกิดขึ้น

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

ได้ดาวทองไปครอง!

> **ผู้บรรยาย:** หายนะกำลังคืบคลานเข้ามาแล้ว...
