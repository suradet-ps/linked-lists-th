# ดับเบิลซิงกลีลิงก์ลิสต์ (The Double Singly-Linked List)

เราเคยต้องปวดหัวกับ doubly-linked list กันมาแล้ว เพราะมันมี semantics ของ ownership ที่พันกันอีรุงตุงนัง: ไม่มีโหนดไหนเป็นเจ้าของโหนดอื่นได้อย่างเด็ดขาด ทว่า ที่เราต้องมานั่งเจอปัญหานี้ ก็เพราะเราดันพกเอาภาพจำเดิม ๆ ว่า linked list *ควรจะเป็นแบบไหน* ติดตัวเข้ามาด้วย นั่นก็คือ เราไปด่วนสรุปเอาเองว่าทิศทางของลิงก์ทั้งหมดต้องชี้ไปทางเดียวกันเสมอ

แต่จริง ๆ แล้ว เราสามารถผ่าครึ่งลิสต์ของเราออกเป็นสองฝั่งได้ครับ: ฝั่งหนึ่งชี้ไปทางซ้าย และอีกฝั่งหนึ่งชี้ไปทางขวา:

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

คราวนี้ แทนที่เราจะมีแค่สแต็กธรรมดา ๆ ที่ปลอดภัย เราก็จะได้ลิสต์อเนกประสงค์มาใช้งานแทนแล้ว เราสามารถขยายลิสต์ไปทางซ้ายหรือทางขวาได้ตามใจชอบ ด้วยการ push ลงในสแต็กฝั่งใดฝั่งหนึ่ง นอกจากนี้เรายังสามารถ "เดิน" ไปตามแนวลิสต์ได้ โดยการ pop ค่าออกจากปลายฝั่งหนึ่งแล้วเอาไป push ใส่อีกฝั่งหนึ่ง และเพื่อหลีกเลี่ยงการ allocate หน่วยความจำโดยไม่จำเป็น เราจะขอยกเอา source code ของ Stack ที่ปลอดภัยตัวเดิมของเรามาใช้ เพื่อให้สามารถเข้าถึงรายละเอียดภายใน (private fields) ของมันได้:

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

แล้วก็ปรับแต่งโค้ดของ `push` กับ `pop` สักเล็กน้อย:

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

ทีนี้เราก็พร้อมสร้าง List ของเราแล้ว:

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

แล้วเราก็สามารถใส่ฟังก์ชันการทำงานพื้นฐานตามปกติเข้าไป:


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

แต่ส่วนที่น่าสนใจที่สุดก็คือ เราสามารถเดินเล่นไปมาบนลิสต์ได้ครับ!


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

เราคืนค่าเป็น boolean เพื่อความสะดวกในการตรวจสอบว่าเราสามารถเดินขยับตำแหน่งได้จริงหรือไม่ เอาล่ะ มาลองทดสอบเจ้าตัวนี้กันเลยดีกว่า:

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

นี่เป็นตัวอย่างแบบสุดขั้วของโครงสร้างข้อมูลตระกูล *finger* (finger data structure) โดยเราจะคงตำแหน่งชี้ (finger) บางอย่างไว้ในโครงสร้างข้อมูล และส่งผลให้เราสามารถรองรับ operation บนตำแหน่งต่าง ๆ โดยใช้เวลาที่แปรผันตรงกับระยะห่างจาก finger นั้น

เราสามารถแก้ไขเปลี่ยนแปลงข้อมูลในลิสต์รอบ ๆ บริเวณ finger ได้อย่างรวดเร็วมาก แต่ถ้าเราต้องการจะแก้ไขตำแหน่งที่อยู่ห่างออกไปไกลจาก finger เราก็ต้องเดินดุ่ม ๆ ไปยังจุดนั้นให้ถึงเสียก่อน ซึ่งเราสามารถเดินไปยังตำแหน่งนั้นอย่างถาวรได้โดยการย้าย element จากสแต็กฝั่งหนึ่งไปยังอีกฝั่งหนึ่ง หรือเราอาจจะแค่เดินไต่ตามลิงก์ไปด้วย `&mut` แบบชั่วคราวเพื่อทำการแก้ไขก็ได้ ทว่า `&mut` จะไม่มีวันเดินถอยหลังย้อนกลับขึ้นมาบนลิสต์ได้ ในขณะที่ finger ของเราทำได้อย่างสบาย ๆ!
