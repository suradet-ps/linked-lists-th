# ทำให้ทุกอย่างเป็นเจเนอริก

ที่ผ่านมาเราได้สัมผัสเจเนอริก (generics) กันมาบ้างแล้วผ่านทาง `Option` และ `Box` แต่จนถึงตอนนี้ เราก็ยังคงหลีกเลี่ยงการประกาศชนิดข้อมูลใหม่ที่รองรับข้อมูลชนิดใด ๆ ก็ตาม (arbitrary types) อยู่ดี

แต่เอาเข้าจริง เรื่องนี้มันง่ายนิดเดียวเองครับ งั้นเรามาปรับชนิดข้อมูลทั้งหมดของเราให้เป็นเจเนอริกกันตอนนี้เลยดีกว่า:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

คุณก็แค่เติมวงเล็บแหลม `<...>` เพิ่มเข้าไปอีกนิดหน่อย โค้ดของคุณก็กลายร่างเป็นเจเนอริกขึ้นมาทันที แน่นอนว่าเราจะทำ *แค่นี้* แล้วชิ่งเลยไม่ได้ ไม่อย่างนั้นคอมไพเลอร์คงได้หงุดหงิดใส่เราชุดใหญ่ไฟกระพริบแน่ ๆ:

```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

ปัญหาตรงนี้ชัดเจนมาก: เรากำลังอ้างถึง `List` โดด ๆ แต่มันไม่มีตัวตนอยู่จริงอีกต่อไปแล้ว เพราะเหมือนกับ `Option` และ `Box` ตอนนี้เราจำเป็นต้องระบุเป็น `List<Something>` เสมอ

แต่แล้ว "Something" ที่เราควรนำมาใช้ในบล็อก `impl` พวกนี้คืออะไรกันล่ะ? ก็เหมือนกับตัว `List` นั่นแหละ คือเราอยากให้โค้ดการทำงานของเราใช้งานได้กับ *ทุก ๆ* ชนิดข้อมูล `T` ในโลก ดังนั้น เราก็แค่จับบล็อก `impl` มาเติมวงเล็บแหลมให้มีพารามิเตอร์เจเนอริกด้วยเช่นกัน:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
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
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

...และนั่นคือทั้งหมดที่ต้องทำครับ!

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

ตอนนี้โค้ดทั้งหมดของเราสามารถรองรับค่าชนิดข้อมูล `T` ใด ๆ ก็ตามได้อย่างสมบูรณ์แบบแล้ว ให้ตายสิ Rust นี่มัน *ง่ายจริง ๆ* และผมขอปรบมือให้เกียรติเป็นพิเศษกับเมธอด `new` ซึ่งเราไม่ต้องไปแตะต้องหรือแก้โค้ดข้างในมันเลยแม้แต่ตัวอักษรเดียว:

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

ขอสรรเสริญในความเกรียงไกรของ `Self` ผู้พิทักษ์แห่งการรีแฟกเตอร์ (refactoring) และการเขียนโค้ดสไตล์ก็อปแปะ! อีกจุดหนึ่งที่น่าสนใจก็คือ เราไม่ต้องพิมพ์ว่า `List<T>` ตอนที่สร้างอินสแตนซ์ของลิสต์เลย เพราะคอมไพเลอร์ช่วยอนุมานชนิดข้อมูล (type inference) ให้เราโดยอัตโนมัติ จากการที่มันรู้ว่าเรากำลังส่งค่านี้กลับออกจากฟังก์ชันที่คาดหวัง `List<T>`

เอาล่ะ ทีนี้เรามาเริ่มใส่ *พฤติกรรมและความสามารถใหม่เอี่ยม* ให้กับลิสต์ของเรากันเลย!