# การสร้างโครงสร้าง

เอาล่ะ เราจะเริ่มต้นจากการประกอบโครงสร้างลิสต์กันก่อน ซึ่งสำหรับระบบใหม่นี้มันค่อนข้างตรงไปตรงมามาก เมธอด `new` ยังคงเรียบง่ายเหมือนเดิม แค่เซ็ตทุกฟิลด์ให้เป็น `None` ให้หมด และเนื่องจากรูปแบบมันเริ่มจะเทอะทะขึ้นเรื่อย ๆ เรามาแยกคอนสตรัคเตอร์สำหรับ `Node` ออกมาต่างหากด้วยเลยดีกว่า:

```rust ,ignore
impl<T> Node<T> {
    fn new(elem: T) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Node {
            elem: elem,
            prev: None,
            next: None,
        }))
    }
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

```text
> cargo build

**A BUNCH OF DEAD CODE WARNINGS BUT IT BUILT**
```

เย้!

คราวนี้มาลองเขียนฟังก์ชันสำหรับ push ข้อมูลเข้าไปที่ด้านหน้าสุดของลิสต์ (push front) กันบ้าง และเนื่องจากลิงก์ลิสต์แบบเชื่อมโยงสองทาง (doubly-linked list) มันซับซ้อนกว่ากันลิบลับ เราจึงจำเป็นต้องลงแรงกันเยอะขึ้นพอสมควร ในขณะที่โอเปอเรชันของลิงก์ลิสต์ทางเดียวมักจะเขียนจบได้ในบรรทัดเดียวง่าย ๆ แต่โอเปอเรชันของลิสต์สองทางกลับยุ่งเหยิงกว่ากันเยอะ

โดยเฉพาะอย่างยิ่ง ตอนนี้เราจำเป็นต้องดูแลกรณีขอบเขต (boundary cases) เวลาที่ลิสต์ว่างเปล่าเป็นพิเศษ โอเปอเรชันส่วนใหญ่มักจะแตะต้องแค่ตัวชี้ `head` หรือไม่ก็ `tail` แต่เมื่อเกิดการเปลี่ยนผ่านเข้าสู่หรือออกจากภาวะลิสต์ว่าง เราจำเป็นต้องแก้ไขค่าของ *ทั้งสองฝั่ง* พร้อม ๆ กัน

วิธีง่าย ๆ ในการตรวจสอบว่าเมธอดที่เราเขียนขึ้นมานั้นถูกต้องสมเหตุสมผลหรือไม่ ก็คือการคอยรักษาเงื่อนไขคงตัว (invariant) ดังต่อไปนี้ไว้ให้ได้: โหนดแต่ละตัวควรจะต้องมีตัวชี้ชี้มาหามันพอดี 2 ตัวเป๊ะ (exactly two pointers) โดยโหนดแต่ละตัวที่อยู่ตรงกลางลิสต์จะถูกชี้โดยโหนดก่อนหน้า (predecessor) และโหนดถัดไป (successor) ส่วนโหนดที่อยู่ปลายสุดทั้งสองฝั่งจะถูกชี้โดยตัวโครงสร้างลิสต์เอง

งั้นมาลองลุยกันดูสักตั้ง:

```rust ,ignore
pub fn push_front(&mut self, elem: T) {
    // new node needs +2 links, everything else should be +0
    let new_head = Node::new(elem);
    match self.head.take() {
        Some(old_head) => {
            // non-empty list, need to connect the old_head
            old_head.prev = Some(new_head.clone()); // +1 new_head
            new_head.next = Some(old_head);         // +1 old_head
            self.head = Some(new_head);             // +1 new_head, -1 old_head
            // total: +2 new_head, +0 old_head -- OK!
        }
        None => {
            // empty list, need to set the tail
            self.tail = Some(new_head.clone());     // +1 new_head
            self.head = Some(new_head);             // +1 new_head
            // total: +2 new_head -- OK!
        }
    }
}
```

```text
cargo build

error[E0609]: no field `prev` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:39:26
   |
39 |                 old_head.prev = Some(new_head.clone()); // +1 new_head
   |                          ^^^^ unknown field

error[E0609]: no field `next` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:40:26
   |
40 |                 new_head.next = Some(old_head);         // +1 old_head
   |                          ^^^^ unknown field
```

เอาล่ะ คอมไพเลอร์ฟ้อง error เริ่มต้นได้สวย... เริ่มต้นได้สวยจริง ๆ (ประชด)

ทำไมเราถึงไม่สามารถเข้าถึงฟิลด์ `prev` กับ `next` บนโหนดของเราได้ล่ะ? ทีคราวก่อนตอนที่เรามีแค่ `Rc<Node>` มันยังเข้าถึงได้สบาย ๆ อยู่เลยแท้ ๆ ดูเหมือนว่าเจ้า `RefCell` จะเข้ามาเกะกะขวางทางเข้าให้แล้ว

เราคงต้องแวะไปเปิดคู่มืออ่านกันหน่อยแล้วล่ะ

*เสิร์ชกูเกิลคำว่า "rust refcell"*

*[คลิกลิงก์แรกสุด](https://doc.rust-lang.org/std/cell/struct.RefCell.html)*

> A mutable memory location with dynamically checked borrow rules
>
> See the [module-level documentation](https://doc.rust-lang.org/std/cell/index.html) for more.

*คลิกลิงก์ต่อไป*

> Shareable mutable containers.
>
> Values of the `Cell<T>` and `RefCell<T>` types may be mutated through shared references (i.e.
> the common `&T` type), whereas most Rust types can only be mutated through unique (`&mut T`)
> references. We say that `Cell<T>` and `RefCell<T>` provide 'interior mutability', in contrast
> with typical Rust types that exhibit 'inherited mutability'.
>
> Cell types come in two flavors: `Cell<T>` and `RefCell<T>`. `Cell<T>` provides `get` and `set`
> methods that change the interior value with a single method call. `Cell<T>` though is only
> compatible with types that implement `Copy`. For other types, one must use the `RefCell<T>`
> type, acquiring a write lock before mutating.
>
> `RefCell<T>` uses Rust's lifetimes to implement 'dynamic borrowing', a process whereby one can
> claim temporary, exclusive, mutable access to the inner value. Borrows for `RefCell<T>`s are
> tracked 'at runtime', unlike Rust's native reference types which are entirely tracked
> statically, at compile time. Because `RefCell<T>` borrows are dynamic it is possible to attempt
> to borrow a value that is already mutably borrowed; when this happens it results in thread
> panic.
>
> # When to choose interior mutability
>
> The more common inherited mutability, where one must have unique access to mutate a value, is
> one of the key language elements that enables Rust to reason strongly about pointer aliasing,
> statically preventing crash bugs. Because of that, inherited mutability is preferred, and
> interior mutability is something of a last resort. Since cell types enable mutation where it
> would otherwise be disallowed though, there are occasions when interior mutability might be
> appropriate, or even *must* be used, e.g.
>
> * Introducing inherited mutability roots to shared types.
> * Implementation details of logically-immutable methods.
> * Mutating implementations of `Clone`.
>
> ## Introducing inherited mutability roots to shared types
>
> Shared smart pointer types, including `Rc<T>` and `Arc<T>`, provide containers that can be
> cloned and shared between multiple parties. Because the contained values may be
> multiply-aliased, they can only be borrowed as shared references, not mutable references.
> Without cells it would be impossible to mutate data inside of shared boxes at all!
>
> It's very common then to put a `RefCell<T>` inside shared pointer types to reintroduce
> mutability:
>
> ```rust ,ignore
> use std::collections::HashMap;
> use std::cell::RefCell;
> use std::rc::Rc;
>
> fn main() {
>     let shared_map: Rc<RefCell<_>> = Rc::new(RefCell::new(HashMap::new()));
>     shared_map.borrow_mut().insert("africa", 92388);
>     shared_map.borrow_mut().insert("kyoto", 11837);
>     shared_map.borrow_mut().insert("piccadilly", 11826);
>     shared_map.borrow_mut().insert("marbles", 38);
> }
> ```
>
> Note that this example uses `Rc<T>` and not `Arc<T>`. `RefCell<T>`s are for single-threaded
> scenarios. Consider using `Mutex<T>` if you need shared mutability in a multi-threaded
> situation.

เฮ้ เอกสารคู่มือของ Rust นี่มันยังคงยอดเยี่ยมกระเทียมดองไม่เปลี่ยนเลยแฮะ

ท่อนเนื้อหาสำคัญจริง ๆ ที่เราต้องสนใจก็คือบรรทัดนี้:

```rust ,ignore
shared_map.borrow_mut().insert("africa", 92388);
```

โดยเฉพาะตรงเจ้า `borrow_mut` นั่น ดูเหมือนว่าเราจำเป็นต้องสั่งยืม `RefCell` ออกมาอย่างโจ่งแจ้ง (explicitly) เพราะตัวดำเนินการ `.` (dot operator) จะไม่แอบสั่งยืมให้อัตโนมัติ แปลกดีแฮะ... งั้นมาลองปรับโค้ดกันดู:

```rust ,ignore
pub fn push_front(&mut self, elem: T) {
    let new_head = Node::new(elem);
    match self.head.take() {
        Some(old_head) => {
            old_head.borrow_mut().prev = Some(new_head.clone());
            new_head.borrow_mut().next = Some(old_head);
            self.head = Some(new_head);
        }
        None => {
            self.tail = Some(new_head.clone());
            self.head = Some(new_head);
        }
    }
}
```


```text
> cargo build

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

เฮ้ คอมไพล์ผ่านฉลุยแล้ว! ชัยชนะเป็นของคู่มืออย่างเป็นเอกฉันท์อีกครา
