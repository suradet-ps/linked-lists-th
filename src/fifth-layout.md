# โครงสร้างในหน่วยความจำ

แล้วคิวแบบ singly-linked list หน้าตาเป็นอย่างไรกันนะ? ตอนที่เราทำสแต็ก เรา push ที่ปลายข้างหนึ่งของลิสต์ แล้ว pop ออกจากปลายข้างเดียวกัน แต่ความแตกต่างระหว่างสแต็กกับคิวก็คือ คิวจะ pop ออกจาก*ปลายอีกข้างหนึ่ง* จากโค้ดสแต็กที่เราเคยทำไว้:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

stack push X:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

stack pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

ในการสร้างคิว เราแค่ต้องตัดสินใจว่าจะย้าย operation ไหนไปไว้ที่ปลายหางของลิสต์: `push` หรือ `pop`? และเนื่องจากลิสต์ของเราเป็น singly-linked list เราจึงสามารถย้าย*operation ไหนก็ได้*ไปไว้ที่ปลายหางโดยเหนื่อยเท่าๆ กัน

ถ้าจะย้าย `push` ไปไว้ที่หาง เราก็แค่วิ่งไล่ไปจนถึง `None` แล้วเปลี่ยนให้มันเป็น `Some` ที่บรรจุเอลิเมนต์ใหม่เข้าไป:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

flipped push X:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)
```

ถ้าจะย้าย `pop` ไปไว้ที่หาง เราก็แค่วิ่งไล่ไปที่โหนด*ตัวก่อนหน้า* None แล้วสั่ง `take`:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)

flipped pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

เราจะทำแบบนี้เลยตอนนี้แล้วบอกว่าเสร็จแล้วก็ได้ แต่นั่นมันแย่สุดๆ! เพราะทั้งสองวิธีนี้ต้องเดินไล่ตั้งแต่ต้นจนจบลิสต์*ทั้งหมด* บางคนอาจจะเถียงว่าอิมพลีเมนต์แบบนั้นมันก็ถือเป็นคิวได้เหมือนกัน เพราะมี interface ถูกต้องครบถ้วน แต่ผมมองว่าการรับประกันประสิทธิภาพ (performance guarantees) ก็เป็นส่วนหนึ่งของ interface เช่นกัน ผมไม่ได้สนใจเรื่องขอบเขตเชิง asymptotic (Big-O) ที่เป๊ะเว่อร์อะไรขนาดนั้นหรอก เอาแค่ "เร็ว" กับ "ช้า" พอ คิวควรต้องรับประกันว่าทั้ง push และ pop ต้องเร็ว และการวิ่งวนข้ามทั้งลิสต์นั้นมัน*ไม่เร็ว*อย่างแน่นอน

ข้อสังเกตสำคัญอย่างหนึ่งคือ เราเสียเวลาไปกับการทำ*เรื่องเดิมซ้ำๆ* เรา "แคช" งานพวกนั้นแล้วนำมาใช้ซ้ำได้ไหม? ได้สิ! เราสามารถเก็บพอยน์เตอร์ชี้ไปที่ปลายหางของลิสต์ไว้ แล้วกระโดดตรงไปที่นั่นได้เลย!

ทว่า ปรากฏว่ามีแค่แบบเดียวเท่านั้นที่สลับแล้วเวิร์ก ถ้าเราสลับ `pop` ไปไว้ที่หาง เราจะต้องเลื่อนพอยน์เตอร์ "tail" ถอยหลังกลับมา ซึ่งเนื่องจากลิสต์ของเราเป็น singly-linked list เราจึงไม่สามารถทำแบบนั้นได้อย่างมีประสิทธิภาพ แต่ถ้าเราสลับ `push` ไปไว้ที่หางแทน เราแค่ต้องเลื่อนพอยน์เตอร์ "head" ไปข้างหน้า ซึ่งง่ายกว่ากันเยอะ

มาลองดูกัน:

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

รอบนี้ผมจะไปเร็วขึ้นในส่วนรายละเอียดของการ implement นะครับ เพราะเราน่าจะเริ่มคุ้นเคยกับเรื่องพวกนี้กันดีแล้ว ไม่ใช่ว่าคุณต้องทำโค้ดนี้ออกมาได้ถูกเป๊ะตั้งแต่ครั้งแรกหรอกนะ ผมแค่ตัดขั้นตอนการลองผิดลองถูกบางช่วงที่เราเคยเจอกันมาแล้วออกไป ตอนที่ผมเขียนโค้ดนี้จริงๆ ผมก็ทำพลาดไปตั้งเยอะแยะที่ไม่ได้เอามาโชว์ให้เห็น แต่การที่ต้องมานั่งดูผมลืมใส่ `mut` หรือลืมเติม `;` ซ้ำแล้วซ้ำเล่ามันคงไม่ได้อะไรขึ้นมา ไม่ต้องห่วงครับ เรายังมี error message แปลกๆ *แบบอื่น* ให้ดูกันอีกเพียบ!

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

พังยับ!

> use of moved value: `new_tail`

`Box` ไม่ได้ implement `Copy` ดังนั้นเราจึงไม่สามารถเอาค่าของมันไปใส่ไว้สองที่พร้อมกันได้ และที่สำคัญยิ่งกว่านั้นคือ `Box` เป็น*เจ้าของ (owner)* สิ่งที่มันชี้อยู่ และจะพยายาม deallocate หน่วยความจำทิ้งเมื่อมันถูก drop ถ้าโค้ด `push` ของเราคอมไพล์ผ่านขึ้นมาจริงๆ ล่ะก็ เราจะกลายเป็นการสั่ง free ท้ายลิสต์ซ้ำสองรอบ (double free)! ยิ่งไปกว่านั้น ตามที่เขียนไว้นี้ โค้ดของเราจะเผลอ free `old_tail` ทิ้งทุกครั้งที่มีการ push อีกต่างหาก!
หายนะชัดๆ! 🙀

เอาล่ะ เรารู้วิธีสร้างตัวชี้ที่ไม่ใช่เจ้าของแล้ว นั่นคือเรเฟอเรนซ์!

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

ตรงนี้ไม่มีอะไรซับซ้อน เป็นตรรกะเดิมเหมือนโค้ดก่อนหน้านี้ เพียงแต่เราอาศัยการ return ค่าออกมาแบบเนียนๆ เพื่อดึงเรเฟอเรนซ์ของ tail มาจากจุดที่เราเพิ่งยัด `Box` ลงไป

```text
> cargo build

error[E0106]: missing lifetime specifier
 --> src/fifth.rs:3:18
  |
3 |     tail: Option<&mut Node<T>>, // NEW!
  |                  ^ expected lifetime parameter
```

อ้อ จริงด้วยสิ! เราต้องระบุ lifetime ให้กับเรเฟอเรนซ์ใน struct เสมอ... หืม... แล้ว lifetime ของเรเฟอเรนซ์ตัวนี้ควรจะเป็นอะไรดีล่ะ? หน้าตามันคล้ายๆ กับ `IterMut` เลยใช่ไหม? งั้นลองทำแบบเดียวกับที่เราเคยทำใน `IterMut` ดู โดยการเพิ่ม generic lifetime `'a` เข้าไป:

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

โอ้โห error message ละเอียดเป็นบ้า ชักจะเริ่มน่าเป็นห่วงแล้วสิ เพราะข้อความแบบนี้แปลว่าเรากำลังทำอะไรที่พิสดารพิลึกกึกกือมากๆ อยู่แน่ๆ ลองดูท่อนที่น่าสนใจตรงนี้ครับ:

> the lifetime must be valid for the lifetime `'a` as defined on the impl

เรากำลังยืมค่ามาจาก `self` แต่คอมไพเลอร์บอกว่าเราต้องมีชีวิตอยู่นานเท่ากับ `'a`... ถ้างั้น ถ้าเราบอกมันไปว่าตัว `self` น่ะ มีอายุอยู่ยาวนานเท่ากับ `'a` จริงๆ ล่ะ จะเป็นยังไงนะ..?

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

เฮ้ย ได้ผลเฉยเลย! ยอดเยี่ยม!

งั้นมาเขียน `pop` ต่อกันเลย:

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

และเขียนเทสต์ทดสอบดูสักหน่อย:

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

🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀

โอ้ คุณพระคุณเจ้าช่วย...

คอมไพเลอร์ไม่ได้ผิดเลยที่พ่น error ใส่หน้าเราแบบนี้ เราเพิ่งจะก่อบาปมหันต์ขั้นร้ายแรงที่สุดในโลกของ Rust ไป: เราดันเก็บเรเฟอเรนซ์ที่ชี้กลับมายังตัวเราเองไว้*ข้างในตัวเราเอง* (self-referential) และไม่รู้ไปกล่อมท่าไหน ถึงได้ทำให้ Rust หลงเชื่อว่าตรรกะแบบนี้มันเมคเซนส์ใน `push` กับ `pop` ของเราได้ (เอาจริงๆ ผมยังช็อกเลยนะที่มันยอมให้ผ่านตอนแรก)

เหตุผลที่มัน*เหมือนจะ*ทำงานได้ก็เพราะ Rust ไม่ได้มีแนวคิดเรื่อง pointer ที่ชี้เข้าหาตัวเองอยู่เลย โค้ดแต่ละส่วน*ในทางเทคนิค*นั้นถูกต้องเมื่อมองแยกกันเดี่ยวๆ (เรา*สามารถ*เรียก push และ pop ได้*หนึ่งครั้ง*) แต่จากนั้น ความพิลึกพิลั่นของสิ่งที่เราสร้างขึ้นมาก็จะเริ่มออกฤทธิ์ และทุกอย่างก็*ล็อกตายทันที*

ผมมั่นใจว่าโค้ดที่เราเขียนขึ้นมาเนี่ยมันคงมีประโยชน์กับ*อะไรบางอย่าง*ในโลกนี้แหละ แต่เท่าที่*ผม*เห็น มันเป็นแค่ *ภาษาต่างดาวอันเพ้อเจ้อ (gibberish)* ที่บังเอิญถูกหลักไวยากรณ์เท่านั้นเอง เราบอกว่าเรามีสิ่งที่มีอายุขัยตาม lifetime `'a` และ `push` กับ `pop` ยืม *self* ตลอดช่วงอายุขัยนั้น มัน*พิลึกมาก* แต่ Rust ก้มดูโค้ดทีละท่อน*แยกกัน*แล้วมันไม่เห็นว่ามีกฎข้อไหนถูกละเมิดเลย

แต่ทันทีที่เราพยายามจะหยิบลิสต์ตัวนี้มา*ใช้งาน*จริงๆ คอมไพเลอร์จะรีบทักทันควันว่า:
"ใช่แล้ว คุณยืม `self` แบบ mutable สำหรับ `'a` ไปแล้ว ดังนั้นคุณจะแตะต้อง `self` ไม่ได้อีกจนกว่าจะหมดช่วง `'a`" แต่*ในขณะเดียวกัน*มันก็บอกว่า "เพราะข้างในตัวคุณมี `'a` อยู่ มันจึงต้องมีอายุยืนยาวเท่ากับการคงอยู่ทั้งหมดของลิสต์ตัวนี้"

มัน*เกือบจะ*เป็นข้อขัดแย้งที่ไม่มีทางออก แต่จริงๆ แล้วมัน*มี*ทางออกอยู่ทางหนึ่ง นั่นคือ: ทันทีที่คุณเรียก `push` หรือ `pop` ลิสต์ตัวนั้นจะ "ตรึง (pin)" ตัวเองอยู่กับที่และไม่มีใครเข้าถึงมันได้อีกเลย ราวกับมันได้กลืนกินหางของตัวเองตามสุภาษิตโบราณ แล้วล่องลอยขึ้นสู่สรวงสวรรค์แห่งความฝันไปเรียบร้อยแล้ว

> **ผู้บรรยาย:** ตอนที่หนังสือเล่มนี้เขียนขึ้นครั้งแรกมันยังไม่มีหรอกนะ แต่ภายหลัง Rust
> ได้ [บรรจุแนวคิดเรื่อง *Pin* เข้ามาเป็นฟีเจอร์ทางการจนใช้การได้จริง][pin]!
> นี่น่าจะเป็นส่วนต่อขยายภาษาที่ซับซ้อนที่สุดนับตั้งแต่มี *borrow checker* มาเลยทีเดียว แต่เราไม่ได้*อยาก*ให้ลิสต์ของเราถูก pin เสียหน่อย!
>
> จริงอยู่ที่ Pin *จำเป็น*และมีประโยชน์มหาศาลสำหรับ async-await/futures/coroutines เพราะคอมไพเลอร์ต้องสามารถแพ็กตัวแปร local ทั้งหมดของฟังก์ชันรวมเป็น struct สักตัว แล้วนำไปเก็บไว้ที่ไหนสักแห่งจนกว่า future/coroutine ตัวนั้นจะพร้อม resume ต่อได้ และเนื่องจากตัวแปร local สามารถอ้างอิงถึงตัวแปร local ตัวอื่นได้ (ซึ่งเราก็ต้องการให้มัน*ทำงานได้*) struct เหล่านี้ก็เลยลงเอยด้วยการมี reference ชี้เข้าหาตัวเอง!
>
> ดังนั้น ในการจะ `await` หรือ `yield` ได้ Rust จึงต้องมีวิธีอธิบายและจัดการค่าที่ถูกตรึง (pinned values) ให้อย่างถูกต้อง โชคดีเหลือเกินที่เรื่องพวกนี้*ส่วนใหญ่*ถูกซ่อนไว้ใต้กลไกอัตโนมัติอันซับซ้อนของคอมไพเลอร์ ทำให้ไม่มีใครต้องมานั่งปวดหัวกับ `Pin` (หรือแม้กระทั่ง *Futures*) ในการเขียนโค้ดทั่วไปเลย เว้นแต่ว่าคุณจะเป็นคนกลุ่มที่ต้องพัฒนาหรือออกแบบ async *runtime* อย่าง tokio
>
> แต่หนังสือเล่มนี้เราจะไม่มานั่งเขียน async runtime กันหรอกนะ ผมรู้ว่าเพื่อนๆ ของผมรู้จัก *ลูกเล่นแพรวพราว* (ที่วิปริตพิสดาร) สารพัดอย่างที่ทำได้ด้วย `Pin` แต่จากที่ผมเห็นแล้วล่ะก็ ผมขอแกล้งทำเป็นไม่รู้ไม่เห็นต่อไปน่าจะมีความสุขในชีวิตมากกว่า ผมจะยังคงหลอกตัวเองต่อไปว่า Pinned types ไม่มีอยู่จริง และมันทำร้ายผมไม่ได้

โค้ด `pop` ของเราชี้ให้เห็นอย่างชัดเจนว่าทำไมการเก็บเรเฟอเรนซ์ชี้เข้าหาตัวเองไว้*ข้างใน*ตัวเราเองถึงได้อันตรายขนาดนี้:

```rust ,ignore
// ...
if self.head.is_none() {
    self.tail = None;
}
```

ถ้าเกิดเราลืมเขียนตรงนี้ขึ้นมาล่ะ? ตัว tail ของเราก็จะชี้ไปยังโหนด*ที่เพิ่งถูกปลดออกจากลิสต์ไปแล้ว* โหนดตัวนั้นจะถูก free ทันที และเราก็จะได้ dangling pointer มาครอบครอง ซึ่งนี่คือสิ่งที่ Rust อุตส่าห์สาบานว่าจะปกป้องเราแท้ๆ!

และจริงๆ แล้ว Rust ก็ปกป้องเราจากอันตรายพรรค์นั้นอยู่จริงๆ นั่นแหละ... แค่ในแบบที่**อ้อมโลกสุดกู่**ไปหน่อยเท่านั้นเอง

แล้วเราจะทำยังไงกันดีล่ะ? ต้องยอมจำนนถอยกลับไปลงนรก `Rc<RefCell>>` อีกรอบงั้นเหรอ?

ขอร้องล่ะ... ม่ายยยยยยย

ไม่ครับ แทนที่จะทำแบบนั้น เราจะหลุดโลกฉีกกรอบเดิมๆ แล้วหันไปใช้ *raw pointers (พอยน์เตอร์ดิบ)* กันเลยดีกว่า!
Layout ของเราจะหน้าตาเป็นแบบนี้:

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


และนั่นก็คือทั้งหมดครับ! ลาก่อนไอ้ระบบ reference-counted-dynamic-borrow-checking หยุมหยิมไม่ได้เรื่อง!
ตัวชี้ของจริง ดิบ เถื่อน ไร้การตรวจสอบ!

> **ผู้บรรยาย:** การอิมพลีเมนต์นี้ จริงๆ แล้วยังคงผิดพลาดอย่างมหันต์และอันตรายสุดๆ แต่ยังไม่ถึงเวลาที่ผู้เขียนจะได้เรียนรู้บทเรียนนั้น เดี๋ยวหัวข้อถัดไปก็จะได้เจ็บแล้วจำตามระเบียบ

มาเขียนโค้ดสไตล์ภาษา C กันเถอะพวกเรา! เขียนภาษา C มันทั้งวี่ทั้งวันไปเลย!

ได้เวลากลับคืนสู่เหย้าแล้ว พร้อมลุย!

ยินดีต้อนรับสู่ `unsafe`!

> **ผู้บรรยาย:** ว้าว ช่างเป็นความมั่นหน้าที่น่าทึ่งจากผู้เขียนจริงๆ


[pin]: https://doc.rust-lang.org/std/pin/index.html
