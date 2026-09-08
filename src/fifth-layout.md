# โครงสร้างในหน่วยความจำ

แล้วคิวแบบซิงกลีลิงก์หน้าตาเป็นอย่างไร? ตอนที่เรามีสแต็กแบบซิงกลีลิงก์
เรา push ที่ปลายด้านหนึ่งของลิสต์ แล้ว pop จากปลายเดียวกัน ความแตกต่าง
ระหว่างสแต็กกับคิวคือคิว pop จาก*ปลายอีกด้าน*หนึ่ง จากการ implement
สแต็กของเรามี:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

stack push X:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

stack pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

เพื่อสร้างคิว เราแค่ต้องตัดสินใจว่าจะย้ายการดำเนินงานไหนไปที่
ปลายของลิสต์: push หรือ pop? เพราะลิสต์ของเราเป็นแบบซิงกลีลิงก์ เราสามารถ
ย้าย*การดำเนินงานใดก็ได้*ไปที่ปลายด้วยความพยายามเท่ากัน

เพื่อย้าย `push` ไปที่ปลาย เราแค่เดินไปที่ `None` แล้วตั้งเป็น Some ด้วย
องค์ประกอบใหม่

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

flipped push X:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)
```

เพื่อย้าย `pop` ไปที่ปลาย เราแค่เดินไปที่โหนด*ก่อน* None แล้ว `take`:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)

flipped pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

เราสามารถทำแบบนี้วันนี้แล้วเรียกว่าจบ แต่นั่นจะแย่มาก! การดำเนินงานทั้งสอง
เดินข้ามลิสต์*ทั้งหมด* บางคนอาจโต้แย้งว่าการ implement คิวแบบนั้น
เป็นคิวจริงๆ เพราะมันเปิดเผย interface ที่ถูกต้อง อย่างไรก็ตาม
ผมเชื่อว่าการรับประกันประสิทธิภาพเป็นส่วนหนึ่งของ interface ผมไม่สน
เกี่ยวกับ boundary แบบ asymptotic ที่แม่นยำ แค่ "เร็ว" กับ "ช้า" คิวรับประกัน
ว่า push และ pop เร็ว และการเดินข้ามลิสต์ทั้งหมดนั้น*ไม่*เร็วอย่างแน่นอน

การสังเกตที่สำคัญอย่างหนึ่งคือเราเสียงานจำนวนมากไปทำ*สิ่งเดียวกัน*
ซ้ำแล้วซ้ำเล่า เราสามารถ "แคช" งานเหล่านั้นและใช้ซ้ำได้ไหม? ได้! เราสามารถเก็บตัวชี้ไปยัง
ปลายของลิสต์ แล้วกระโดดตรงไปที่นั่น!

ปรากฏว่ามีแค่การกลับกันอย่างหนึ่งของ `push` และ `pop` ที่ใช้ได้กับสิ่งนี้
เพื่อกลับกัน `pop` เราจะต้องย้ายตัวชี้ "tail" ถอยหลัง แต่
เพราะลิสต์ของเราเป็นแบบซิงกลีลิงก์ เราทำไม่ได้อย่างมีประสิทธิภาพ
ถ้าเราเปลี่ยนไปกลับกัน `push` เราแค่ต้องย้ายตัวชี้ "head"
ไปข้างหน้า ซึ่งง่าย

มาลองดู:

```rust ,ignore
use std::mem;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // swap the old tail to point to the new tail
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        match old_tail {
            Some(mut old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
            }
        }
    }
}
```

ผมกำลังไปเร็วขึ้นกับรายละเอียด impl ตอนนี้เพราะเราควรจะ
คุ้นเคยกับสิ่งเหล่านี้ดีแล้ว ไม่ใช่ว่าคุณควรคาดหวัง
จะสร้างโค้ดนี้ในการลองครั้งแรก ผมแค่ข้ามส่วนหนึ่งของ
การลองผิดลองถูกที่เราเคยเจอมา ผมจริงๆ แล้วทำผิดพลาดจำนวนมาก
เขียนโค้ดนี้ที่ผมไม่ได้แสดง แต่คุณจะเห็นแค่ผมทิ้ง `mut` หรือ
`;` หลายครั้งก่อนที่มันจะหยุดให้บทเรียน ไม่ต้องห่วง เราจะเจอ
ข้อความ*อื่นๆ* มากมาย!

```text
> cargo build

error[E0382]: use of moved value: `new_tail`
  --> src/fifth.rs:38:38
   |
26 |         let new_tail = Box::new(Node {
   |             -------- move occurs because `new_tail` has type `std::boxed::Box<fifth::Node<T>>`, which does not implement the `Copy` trait
...
33 |         let old_tail = mem::replace(&mut self.tail, Some(new_tail));
   |                                                          -------- value moved here
...
38 |                 old_tail.next = Some(new_tail);
   |                                      ^^^^^^^^ value used here after move
```

แย่!

> use of moved value: `new_tail`

Box ไม่ได้ implement Copy ดังนั้นเราจึงไม่สามารถกำหนดให้สองตำแหน่งได้
สิ่งที่สำคัญกว่าคือ Box *เป็นเจ้าของ*สิ่งที่มันชี้ และจะพยายามปลดปล่อยมันเมื่อ
มันถูก drop ถ้า `push` ของเราคอมไพล์ได้ เราจะ free หางของลิสต์สองครั้ง!
จริงๆ แล้วตามที่เขียนไว้ โค้ดของเราจะ free old_tail ทุกครั้งที่ push
แย่! 🙀

เอาล่ะ เรารู้วิธีสร้างตัวชี้ที่ไม่ใช่เจ้าของ นั่นคือเรเฟอเรนซ์!

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: Option<&mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

ไม่มีอะไรซับซ้อนที่นี่ ตรรกะเดียวกับโค้ดก่อนหน้า แต่เราใช้
การ return แบบซ่อนเพื่อดึงเรเฟอเรนซ์หางจาก
ที่ไหนก็ตามที่เราใส่ Box จริงๆ

```text
> cargo build

error[E0106]: missing lifetime specifier
 --> src/fifth.rs:3:18
  |
3 |     tail: Option<&mut Node<T>>, // NEW!
  |                  ^ expected lifetime parameter
```

โอ้ ถูกต้อง เราต้องให้ lifetimes แก่เรเฟอเรนซ์ในชนิด ฮึม... ไลฟ์ไทม์
ของเรเฟอเรนซ์นี้คืออะไร? นี่ดูเหมือน IterMut จริงไหม? มาลอง
สิ่งที่เราทำสำหรับ IterMut และเพิ่มเจเนอริก `'a`:

```rust ,ignore
pub struct List<'a, T> {
    head: Link<T>,
    tail: Option<&'a mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<'a, T> List<'a, T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

```text
cargo build

error[E0495]: cannot infer an appropriate lifetime for autoref due to conflicting requirements
  --> src/fifth.rs:35:27
   |
35 |                 self.head.as_deref_mut()
   |                           ^^^^^^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 18:5...
  --> src/fifth.rs:18:5
   |
18 | /     pub fn push(&mut self, elem: T) {
19 | |         let new_tail = Box::new(Node {
20 | |             elem: elem,
21 | |             // When you push onto the tail, your next is always None
...  |
39 | |         self.tail = new_tail;
40 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/fifth.rs:35:17
   |
35 |                 self.head.as_deref_mut()
   |                 ^^^^^^^^^
note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 13:6...
  --> src/fifth.rs:13:6
   |
13 | impl<'a, T> List<'a, T> {
   |      ^^
   = note: ...so that the expression is assignable:
           expected std::option::Option<&'a mut fifth::Node<T>>
              found std::option::Option<&mut fifth::Node<T>>


```

ว้าว นั่นเป็นข้อความแสดงข้อผิดพลาดที่ละเอียดมาก ค่อนข้างกังวล เพราะมัน
แนะนำว่าเราทำบางอย่างที่พังจริงๆ นี่คือส่วนที่น่าสนใจ:

> the lifetime must be valid for the lifetime `'a` as defined on the impl

เรายืมจาก `self` แต่คอมไพเลอร์ต้องการให้เราอยู่นานเท่า `'a`
ถ้าเราบอกมันว่า `self` *อยู่*นานเท่านั้น...?

```rust ,ignore
    pub fn push(&'a mut self, elem: T) {
```

```text
cargo build

warning: field is never used: `elem`
 --> src/fifth.rs:9:5
  |
9 |     elem: T,
  |     ^^^^^^^
  |
  = note: #[warn(dead_code)] on by default
```

โอ้ ฮิ ได้แล้ว! ยอดเยี่ยม!

มาทำ `pop` ด้วย:

```rust ,ignore
pub fn pop(&'a mut self) -> Option<T> {
    // Grab the list's current head
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        // If we're out of `head`, make sure to set the tail to `None`.
        if self.head.is_none() {
            self.tail = None;
        }

        head.elem
    })
}
```

และเขียนเทสต์อย่างรวดเร็วสำหรับมัน:

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
    }
}
```

```text
cargo test

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:68:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
68 |         list.push(1);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:69:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
69 |         list.push(2);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:70:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
70 |         list.push(3);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here


....

** WAY MORE LINES OF ERRORS **

....

error: aborting due to 11 previous errors
```

🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🌟🌟🌟🌟🌟🌟🌟🌟🌟🌟🌟🌟

โอ้พระเจ้า

คอมไพเลอร์ไม่ได้ผิดที่อาเจียนใส่เรา เราเพิ่งก่ออาชญากรรมร้ายแรงของ Rust:
เราเก็บเรเฟอเรนซ์ไปยังตัวเอง*ภายในตัวเอง* ไม่ทางใดก็ทางหนึ่ง เราจัดการ
บอก Rust ว่ามันสมเหตุสมผลใน `push` และ `pop` ของเรา
(ผมตกใจจริงๆ ที่เราทำได้)

เหตุผลที่มัน*แบบ*ทำงานได้คือ Rust ไม่มีความคิด
ของตัวชี้เข้าไปในตัวเองจริงๆ ส่วนหนึ่งของโค้ด*ทางเทคนิค*ถูกต้อง
แบบโดดเดี่ยว (เรา*สามารถ*เรียก push และ pop*ครั้งเดียว*)
แต่จากนั้นความไร้สาระของสิ่งที่เราสร้างขึ้นจะมีผลและทุกอย่างก็*ล็อก*

ผมแน่ใจว่ามี*ประโยชน์บางอย่าง*สำหรับสิ่งที่เราเขียน แต่เท่าที่*ผม*สนใจ
มันเป็นแค่* gibberish*ที่ถูกต้องตามไวยากรณ์ เราพูดว่าเรามีบางอย่างที่มี
ไลฟ์ไทม์ `'a` และ `push` และ `pop` ยืม*self*สำหรับไลฟ์ไทม์นั้น
มัน*แปลก*แต่ Rust สามารถดูแต่ละส่วนของโค้ด*แต่ละส่วน*และ
มันไม่เห็นกฎที่ถูกฝ่าฝืน

แต่เมื่อเราลอง*ใช้*ลิสต์จริงๆ คอมไพเลอร์จะพูดอย่างรวดเร็ว
"ใช่ คุณยืม `self` แบบ mutable สำหรับ `'a` ดังนั้นคุณไม่สามารถใช้ `self` อีกต่อไป
จนถึงจุดสิ้นสุดของ `'a`" แต่*ยัง* "เพราะคุณมี `'a` มันต้องถูกต้อง
สำหรับการมีอยู่ทั้งหมดของลิสต์"

มัน*ใกล้เคียง*กับความขัดแย้ง แต่*มี*วิธีแก้ไขอย่างหนึ่ง: ทันทีที่คุณ `push`
หรือ `pop` ลิสต์จะ "ตรึง" ตัวเองในที่และไม่สามารถเข้าถึงได้อีกต่อไป มันได้
กลืนหางเชิงเปรียบเทียบของตัวเอง และขึ้นสู่โลกแห่งความฝัน

> **ผู้บรรยาย:** มันไม่มีอยู่เมื่อหนังสือเล่มนี้เขียนครั้งแรก แต่ Rust
> ได้ [ formalize แนวคิดของ *pin* into something useful][pin]!
> นี่อาจเป็นการเพิ่มที่ซับซ้อนที่สุดในภาษาตั้งแต่
> *the borrowchecker* เราไม่*ต้องการ*ให้ลิสต์ของเราถูกตรึง!
>
> Pins *จำเป็น*และมีประโยชน์สำหรับ async-await/futures/coroutines เพราะ
> คอมไพเลอร์ต้องสามารถรวมตัวแปรท้องถิ่นของฟังก์ชัน
> เป็นโครงสร้างบางประเภทและเก็บไว้ที่ไหนสักแห่งจนกว่า
> future/coroutine พร้อมที่จะ resume เพราะตัวแปรท้องถิ่นสามารถอ้างอิง
> ตัวแปรท้องถิ่นอื่นๆ และเราต้องการให้มัน*ทำงานได้* โครงสร้างเหล่านี้อาจ
> มีการอ้างอิงถึงตัวเอง!
>
> ดังนั้นเพื่อ `await` หรือ `yield` Rust ต้องมีวิธีอธิบายและ
> จัดการค่าที่ตรึงอย่างถูกต้อง โชคดีที่สิ่งเหล่านี้ส่วนใหญ่เป็น*อย่างมาก*
> แค่ซ่อนอยู่ในเครื่องจักรคอมไพเลอร์อัตโนมัติและไม่มีใครต้อง
> คิดถึง `Pin` (หรือแม้แต่ *Futures*) ภายใต้สถานการณ์ปกติ ข้อยกเว้นหลัก
> คือสิ่งเหล่านี้สำคัญมากสำหรับผู้ที่สร้างและ
> ออกแบบ async *runtimes* เช่น tokio
>
> เราจะไม่ implement async runtime ในหนังสือเล่มนี้ ผมรู้ว่าเพื่อนของผม
> รู้ "เจ๋ง" (พัง) *กล*ทุกประเภทที่คุณสามารถทำได้กับ `Pin`
> แต่เท่าที่ผมรู้ ผมจะมีความสุขกว่าถ้าไม่รู้จักมัน ผมจะ
> บอกตัวเองต่อไปว่าชนิดที่ตรึงไม่ใช่ของจริงและไม่สามารถทำร้ายผมได้

`pop` ของเราแนะนำว่าทำไมการเก็บเรเฟอเรนซ์ไปยังตัวเอง
*ภายใน*ตัวเองอาจอันตรายมาก:

```rust ,ignore
// ...
if self.head.is_none() {
    self.tail = None;
}
```

ถ้าเราลืมทำแบบนี้? หางของเราจะชี้ไปยังโหนด*ที่ถูกลบออกจากลิสต์* โหนดแบบนั้นจะ
ถูกปลดปล่อยทันที และเราจะมีตัวชี้ลอยที่ Rust ควรจะปกป้องเรา!

และ Rust ก็ปกป้องเราจากอันตรายแบบนั้นจริงๆ ในแบบที่... **อ้อม**มาก

แล้วเราทำอะไรได้? ย้อนกลับไปนรก `Rc<RefCell>>`?

กรุณา ไม่

ไม่ แทนที่เราจะไปใช้ *ตัวชี้ดิบ*
โครงสร้างของเราจะหน้าตาแบบนี้:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // DANGER DANGER
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```


และนั่นคือทั้งหมด ไม่มี nonsense แบบ reference-counted-dynamic-borrow-checking
ที่อ่อนแอ! จริง แข็ง ไม่ได้ตรวจสอบ ตัวชี้

> **ผู้บรรยาย:** การ implement นี้ยังคงอันตรายผิดพลาดจริงๆ แต่ยังไม่ถึงเวลาที่จะเรียนรู้บทเรียนนั้น ส่วนถัดไปจะเรียนรู้แบบยาก ตามปกติ

มาเป็น C กันทุกคน มาเป็น C ตลอดทั้งวัน

ผมอยู่บ้าน ผมพร้อมแล้ว

สวัสดี `unsafe`

> **ผู้บรรยาย:** ว้าว แค่ความเย่อหยิ่งที่น่าเหลือเชื่อจากผู้เขียนที่นี่


[pin]: https://doc.rust-lang.org/std/pin/index.html
