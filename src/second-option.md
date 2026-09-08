# การใช้ Option

ผู้สังเกตการณ์ที่ช่างสังเกตอาจสังเกตเห็นว่าเราจริง ๆ แล้วได้สร้าง Option เวอร์ชันที่แย่มาก
ขึ้นมาใหม่:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Link ก็แค่ `Option<Box<Node>>` การไม่ต้องเขียน `Option<Box<Node>>` ทุกที่ก็ดีอยู่
และต่างจาก `pop` เราไม่ได้เปิดเผยมันสู่โลกภายนอก ดังนั้นมันอาจไม่เป็นไร
อย่างไรก็ตาม Option มีเมธอด*ที่ดีมาก*หลายตัวที่เรา implement เองมาตลอด
เรามา*ไม่*ทำแบบนั้น แล้วแทนที่ทุกอย่างด้วย Option กัน ขั้นแรกเราจะทำแบบง่าย ๆ
ด้วยการเปลี่ยนชื่อทุกอย่างเป็น Some และ None:

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// yay type aliases!
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

นี่ดีขึ้นเล็กน้อย แต่ชัยชนะครั้งใหญ่จะมาจากเมธอดของ Option

ประการแรก `mem::replace(&mut option, None)` เป็นสำนวนที่ใช้บ่อยอย่างไม่น่าเชื่อ
จน Option ได้ทำมันเป็นเมธอดไปเลย: `take`

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

ประการที่สอง `match option { None => None, Some(x) => Some(y) }` เป็นสำนวนที่ใช้บ่อย
อย่างไม่น่าเชื่อจนถูกเรียกว่า `map` `map` รับฟังก์ชันหนึ่งเพื่อดำเนินการกับ `x` ใน `Some(x)`
เพื่อสร้าง `y` ใน `Some(y)` เราอาจเขียน `fn` ที่ถูกต้องแล้วส่งให้ `map`
แต่เราอยากเขียนสิ่งที่จะทำแบบ*อินไลน์*มากกว่า

วิธีทำแบบนี้คือผ่าน*คลอ저 (closure)* คลอ저คือฟังก์ชันนิรนามที่มี
พลังวิเศษเพิ่มเติม: มันสามารถอ้างถึงตัวแปรท้องถิ่น*ภายนอก*คลอ저!
สิ่งนี้ทำให้มันมีประโยชน์มากสำหรับตรรกะทุกชนิด ที่เดียวที่เราใช้ `match` คือใน
`pop` งั้นเรามาเขียนมันใหม่:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

อะ ดีกว่าเยอะ ให้แน่ใจว่าเราไม่ได้ทำอะไรพัง:

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

เยี่ยม! มาต่อกับการปรับปรุง*พฤติกรรม*ของโค้ดจริง ๆ กันเถอะ