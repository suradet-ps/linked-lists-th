# Drop (การปล่อย)

เหมือนลิสต์ที่กลายพันธุ์ได้ เรามีปัญหาดีสตรัคเตอร์แบบรีเคอร์ซีฟ
ยอมรับว่ามันไม่ได้แย่เท่าสำหรับลิสต์ที่ไม่เปลี่ยนแปลงได้: ถ้าเราเจอ
โหนดอีกตัวที่เป็นหัวของลิสต์อีกตัว*ที่ไหนสักแห่ง* เราจะไม่ drop
มันแบบรีเคอร์ซีฟ อย่างไรก็ตามมันยังเป็นสิ่งที่เราควรแคร์ และวิธีจัดการ
ไม่ได้ชัดเจนเท่าไหร่ นี่คือวิธีที่เราแก้ไขมาก่อน:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

ปัญหาอยู่ที่บอดี้ของลูป:

```rust ,ignore
cur_link = boxed_node.next.take();
```

นี่คือการกลายพันธุ์ Node ภายใน Box แต่เราทำไม่ได้กับ Rc มันให้เราแค่
การเข้าถึงแบบร่วม เพราะ Rc หลายตัวอาจชี้ไปถึงมัน

แต่ถ้าเรารู้ว่าเราเป็นลิสต์สุดท้ายที่รู้จักโหนดนี้ มัน*จะ*ไม่เป็นไรจริง ๆ ที่จะย้าย
Node ออกจาก Rc แล้วเราก็จะรู้ว่าต้องหยุดเมื่อไหร่: เมื่อเรา*ไม่สามารถ*ย้าย
Node ออกมาได้

และดูสิ Rc มีเมธอดที่ทำสิ่งนี้: `try_unwrap`:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}
```

```text
cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/too-many-lists/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.10s
     Running /Users/ADesires/dev/too-many-lists/lists/target/debug/deps/lists-86544f1d97438f1f

running 8 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

เยี่ยม!
ดี