# ทำให้ทุกอย่างเป็นเจเนอริก

เราได้แตะเจเนอริกมาบ้างแล้วกับ Option และ Box แต่จนถึงตอนนี้เราจัดการ
หลีกเลี่ยงการประกาศชนิดใหม่ที่เจเนอริกเหนือองค์ประกอบแบบสุ่มได้

ปรากฎว่ามันจริง ๆ แล้วง่ายมาก มาทำให้ทุกชนิดของเราเป็นเจเนอริกตอนนี้เลย:

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

คุณแค่ทำให้ทุกอย่างชี้แหลมขึ้นเล็กน้อย แล้วจู่ ๆ โค้ดของคุณก็เป็นเจเนอริก
แน่นอนว่าเราไม่สามารถ*แค่*ทำแบบนี้ได้ ไม่อย่างนั้นคอมไพเลอร์จะโมโหมาก

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

ปัญหาชัดเจนมาก: เราพูดถึง `List` อยู่ แต่มันไม่ใช่ของจริงแล้ว เหมือน Option และ Box
ตอนนี้เราต้องพูดถึง `List<Something>` เสมอ

แต่ Something ที่เราใช้ใน impl เหล่านี้คืออะไร? เหมือนกับ List เราต้องการให้
implement ของเราใช้ได้กับ `T` ทุกตัว ดังนั้น เหมือน List มาทำให้ `impl`
ของเรากลายเป็นรูปแหลมกัน:

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

...และนั่นคือทั้งหมด!

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

โค้ดทั้งหมดของเราตอนนี้เป็นเจเนอริกเหนือค่าสุ่มของ T อย่างสมบูรณ์
ให้ตายเถอะ Rust *ง่าย* ผมขอชื่นชม `new` เป็นพิเศษซึ่งไม่ได้เปลี่ยนแปลงเลย:

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

เพลิดเพลินกับความรุ่งเรืองของ Self ผู้พิทักษ์การรี팩เตอร์และโค้ดแบบก๊อปแปะ
นอกจากนี้ยังน่าสนใจที่เราไม่เขียน `List<T>` เมื่อเราสร้างอินสแตนซ์ของลิสต์
ส่วนนั้นถูก Infer ให้เราโดยอัตโนมัติจากข้อเท็จจริงที่ว่าเรากำลังคืนมันจากฟังก์ชัน
ที่คาดหวัง `List<T>`

เอาล่ะ มาต่อกับ*พฤติกรรม*ใหม่ทั้งหมดกันเถอะ!