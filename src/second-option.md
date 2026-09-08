# การใช้ Option

ผู้อ่านที่ช่างสังเกตเป็นพิเศษอาจจะเริ่มเอะใจแล้วว่า จริง ๆ แล้วสิ่งที่เราทำไปในบทแรก ก็คือการประดิษฐ์ `Option` เวอร์ชันสุดห่วยขึ้นมาใช้เองใหม่ชัด ๆ:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

เพราะจริง ๆ แล้ว `Link` มันก็คือ `Option<Box<Node>>` ดี ๆ นี่เอง แน่นอนว่าการไม่ต้องคอยมานั่งพิมพ์คำว่า `Option<Box<Node>>` ซ้ำไปซ้ำมามันก็สะดวกดีอยู่หรอก และต่างจาก `pop` ตรงที่เราไม่ได้เปิดเผยไส้ในของมันออกไปสู่โลกภายนอก มันก็เลยดูเหมือนไม่น่าจะมีปัญหาอะไร ทว่า `Option` ดันมีสารพัดเมธอด *ชั้นยอด* ที่ช่วยอำนวยความสะดวกมากมาย ซึ่งที่ผ่านมาเรามัวแต่นั่งเขียนลอจิกพวกนั้นขึ้นมาเองด้วยมือล้วน ๆ งั้นเราอย่ามาเสียเวลาทำแบบนั้นกันเลยดีกว่า เปลี่ยนทุกอย่างหันมาใช้ `Option` กันให้หมดดีกว่า! เริ่มแรก เราจะลองทำแบบตรงไปตรงมาที่สุดก่อน ด้วยการเปลี่ยนชื่อทุกอย่างให้หันมาใช้ `Some` และ `None`:

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

โค้ดแบบนี้ก็ดูดีขึ้นมาอีกนิดหนึ่งแล้ว แต่ประโยชน์เนื้อ ๆ เน้น ๆ ที่แท้จริงจะมาจากบรรดาเมธอดที่มีมาให้ในตัวของ `Option` ต่างหาก

ข้อแรก ท่าอย่าง `mem::replace(&mut option, None)` เป็นท่ามาตรฐานที่เจอบ่อยเสียจนผู้สร้าง `Option` จับมันมาทำเป็นเมธอดสำเร็จรูปให้เราเรียกใช้ได้ทันที นั่นคือเมธอด `take`:

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

ข้อที่สอง แพตเทิร์นอย่าง `match option { None => None, Some(x) => Some(y) }` ก็เป็นอีกหนึ่งท่าที่เจอบ่อยจนเขาตั้งชื่อเมธอดให้ว่า `map` โดย `map` จะรับฟังก์ชันเข้าไปตัวหนึ่ง เพื่อนำไปรันกับค่า `x` ที่อยู่ข้างใน `Some(x)` แล้วแปลงออกมาเป็นผลลัพธ์ `y` เพื่อห่อกลับเป็น `Some(y)` ส่งคืนให้ จริงอยู่ว่าเราจะเขียนฟังก์ชัน `fn` ปกติส่งเข้าไปให้ `map` ก็ได้ แต่เราอยากเขียนคำสั่งที่จะทำแบบ *อินไลน์ (inline)* ณ จุดนั้นไปเลยมากกว่า

และวิธีการทำแบบนั้นก็คือการใช้ *โคลเชอร์ (closure)* นั่นเอง! โคลเชอร์คือฟังก์ชันนิรนาม (anonymous function) ที่มาพร้อมกับพลังวิเศษ: คือมันสามารถหยิบตัวแปรโลคอลที่อยู่ *ภายนอก* ตัวโคลเชอร์มาใช้งานได้ด้วย! คุณสมบัตินี้ทำให้มันมีประโยชน์มหาศาลในการเขียนเงื่อนไขต่าง ๆ และจุดเดียวในโค้ดเราที่ยังต้องใช้ `match` อยู่ก็คือในเมธอด `pop` งั้นเรามาเขียนมันใหม่ด้วย `map` กันเลย:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

อ่า ค่อยดูดีขึ้นเยอะเลย! ทีนี้มาลองรันเทสต์ดูหน่อยว่าเราไม่ได้ทำอะไรพังใช่ไหม:

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

แจ่มมาก! ทีนี้เรามาลุยต่อกับการยกระดับ *พฤติกรรมและความสามารถ* ของโค้ดเราจริง ๆ กันเลยดีกว่า