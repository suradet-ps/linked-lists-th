# ลิสต์แบบ linking เดี่ยวสองทิศทาง (The Double Singly-Linked List)

เราเคยประสบปัญหากับ linked list แบบสองทิศทาง (doubly-linked list) เพราะมี.semantics ของ ownership ที่ยุ่งเหยิง:
ไม่มี node ไหนที่เป็นเจ้าของ node อื่นอย่างเด็ดขาด อย่างไรก็ตาม เราประสบปัญหานี้เพราะเรา
นำมโนทัศน์ที่มีอยู่เดิมว่า linked list *คืออะไร* เข้ามาใช้ กล่าวคือ เราสมมติว่า
link ทั้งหมดไปในทิศทางเดียวกัน

แทนที่จะเป็นเช่นนั้น เราสามารถแยก list ออกเป็นสองส่วน: ส่วนหนึ่งไปทางซ้าย
และอีกส่วนหนึ่งไปทางขวา:

```rust ,ignore
// lib.rs
// ...
pub mod silly1;     // NEW!
```

```rust ,ignore
// silly1.rs
use crate::second::List as Stack;

struct List<T> {
    left: Stack<T>,
    right: Stack<T>,
}
```

ตอนนี้แทนที่จะมีแค่ stack ที่ปลอดภัย เรามี list อเนกประสงค์ เราสามารถ
ขยาย list ไปทางซ้ายหรือขวาได้โดยการ push ไปยัง stack ใดก็ได้ เราสามารถ
"เดิน" ไปตาม list ได้โดยการ pop ค่าจากปลายด้านหนึ่งและ push ไปยังอีกด้านหนึ่ง
เพื่อหลีกเลี่ยงการจัดสรรหน่วยความจำที่ไม่จำเป็น เราจะคัดลอก source ของ
Stack ที่ปลอดภัยของเราเพื่อเข้าถึงรายละเอียดภายในของมัน:

```rust ,ignore
pub struct Stack<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Stack { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            let node = *node;
            self.head = node.next;
            node.elem
        })
    }

    pub fn peek(&self) -> Option<&T> {
        self.head.as_ref().map(|node| {
            &node.elem
        })
    }

    pub fn peek_mut(&mut self) -> Option<&mut T> {
        self.head.as_mut().map(|node| {
            &mut node.elem
        })
    }
}

impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

และแก้ไข `push` และ `pop` เล็กน้อย:

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let new_node = Box::new(Node {
        elem: elem,
        next: None,
    });

    self.push_node(new_node);
}

fn push_node(&mut self, mut node: Box<Node<T>>) {
    node.next = self.head.take();
    self.head = Some(node);
}

pub fn pop(&mut self) -> Option<T> {
    self.pop_node().map(|node| {
        node.elem
    })
}

fn pop_node(&mut self) -> Option<Box<Node<T>>> {
    self.head.take().map(|mut node| {
        self.head = node.next.take();
        node
    })
}
```

ตอนนี้เราสามารถสร้าง List ของเราได้:

```rust ,ignore
pub struct List<T> {
    left: Stack<T>,
    right: Stack<T>,
}

impl<T> List<T> {
    fn new() -> Self {
        List { left: Stack::new(), right: Stack::new() }
    }
}
```

และเราสามารถทำสิ่งต่างๆ ตามปกติได้:


```rust ,ignore
pub fn push_left(&mut self, elem: T) { self.left.push(elem) }
pub fn push_right(&mut self, elem: T) { self.right.push(elem) }
pub fn pop_left(&mut self) -> Option<T> { self.left.pop() }
pub fn pop_right(&mut self) -> Option<T> { self.right.pop() }
pub fn peek_left(&self) -> Option<&T> { self.left.peek() }
pub fn peek_right(&self) -> Option<&T> { self.right.peek() }
pub fn peek_left_mut(&mut self) -> Option<&mut T> { self.left.peek_mut() }
pub fn peek_right_mut(&mut self) -> Option<&mut T> { self.right.peek_mut() }
```

แต่สิ่งที่น่าสนใจที่สุดคือ เราสามารถเดินไปมาได้!


```rust ,ignore
pub fn go_left(&mut self) -> bool {
    self.left.pop_node().map(|node| {
        self.right.push_node(node);
    }).is_some()
}

pub fn go_right(&mut self) -> bool {
    self.right.pop_node().map(|node| {
        self.left.push_node(node);
    }).is_some()
}
```

เราคืนค่า boolean ที่นี่เพื่อความสะดวกในการระบุว่าเราสามารถย้ายได้จริงหรือไม่
ตอนนี้มาทดสอบกันเถอะ:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn walk_aboot() {
        let mut list = List::new();             // [_]

        list.push_left(0);                      // [0,_]
        list.push_right(1);                     // [0, _, 1]
        assert_eq!(list.peek_left(), Some(&0));
        assert_eq!(list.peek_right(), Some(&1));

        list.push_left(2);                      // [0, 2, _, 1]
        list.push_left(3);                      // [0, 2, 3, _, 1]
        list.push_right(4);                     // [0, 2, 3, _, 4, 1]

        while list.go_left() {}                 // [_, 0, 2, 3, 4, 1]

        assert_eq!(list.pop_left(), None);
        assert_eq!(list.pop_right(), Some(0));  // [_, 2, 3, 4, 1]
        assert_eq!(list.pop_right(), Some(2));  // [_, 3, 4, 1]

        list.push_left(5);                      // [5, _, 3, 4, 1]
        assert_eq!(list.pop_right(), Some(3));  // [5, _, 4, 1]
        assert_eq!(list.pop_left(), Some(5));   // [_, 4, 1]
        assert_eq!(list.pop_right(), Some(4));  // [_, 1]
        assert_eq!(list.pop_right(), Some(1));  // [_]

        assert_eq!(list.pop_right(), None);
        assert_eq!(list.pop_left(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 16 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fourth::test::into_iter ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::basics ... ok
test third::test::iter ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured
```

นี่เป็นตัวอย่างที่สุดโต่งของข้อมูลแบบ *finger* ซึ่งเราเก็บรักษา finger บางประเภทไว้ในโครงสร้าง
และผลที่ตามมาคือเราสามารถสนับสนุน operation ต่างๆ บนตำแหน่งต่างๆ ได้ในเวลาที่
แปรผกผันกับระยะห่างจาก finger

เราสามารถเปลี่ยนแปลง list รอบ finger ได้อย่างรวดเร็วมาก แต่ถ้าเราต้องการ
เปลี่ยนแปลงในระยะไกลจาก finger เราต้องเดินไปที่นั่น เราสามารถเดินไปที่นั่นได้
อย่างถาวรโดยการย้ายองค์ประกอบจาก stack หนึ่งไปยังอีก stack หนึ่ง หรือเราสามารถ
เดินไปตาม link ด้วย `&mut` ชั่วคราวเพื่อทำการเปลี่ยนแปลง อย่างไรก็ตาม `&mut`
ไม่สามารถย้อนกลับขึ้นไปใน list ได้ ในขณะที่ finger ของเราทำได้!
