# Drop (การปล่อย)

เช่นเดียวกับลิสต์แบบแก้ไขค่าได้ เราต้องเผชิญหน้ากับปัญหา destructor ทำงานแบบเรียกซ้ำ (recursive destructor) จนอาจทำให้สแต็กพังทะลักได้อีกแล้ว
ต้องยอมรับว่าปัญหานี้ไม่ได้ร้ายแรงเท่าไหร่สำหรับลิสต์แบบเปลี่ยนแปลงค่าไม่ได้ (immutable list) เพราะหากเราไล่ไปเจอโหนดที่เป็นส่วนหัวของลิสต์ตัวอื่นเข้า *ณ จุดใดจุดหนึ่ง* เราก็จะไม่ต้อง drop มันลงไปลึก ๆ แบบ recursive อยู่แล้ว อย่างไรก็ตาม นี่ก็ยังคงเป็นประเด็นที่เราควรใส่ใจ และวิธีการรับมือกับมันก็ไม่ได้ตรงไปตรงมาเสียทีเดียว นี่คือท่าที่เราเคยใช้แก้ปัญหามาก่อนหน้านี้:

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

ปัญหาจะเกิดขึ้นที่ไส้ในของลูป:

```rust ,ignore
cur_link = boxed_node.next.take();
```

โค้ดตรงนี้เป็นการเข้าไปแก้ไขค่า (mutate) ข้อมูล `Node` ที่อยู่ข้างใน `Box` แต่เราไม่สามารถทำแบบนั้นกับ `Rc` ได้เลย เพราะมันให้สิทธิ์เราแค่การเข้าถึงแบบแชร์ (shared access) เท่านั้น เนื่องจากอาจมี `Rc` ตัวอื่นอีกกี่ตัวก็ได้กำลังชี้มาที่โหนดเดียวกันนี้อยู่

แต่ถ้าเรารู้ได้อย่างแน่ชัดว่า เราเป็นลิสต์ตัวสุดท้ายที่ยังคงถือครองโหนดนี้อยู่ มันก็ *ย่อมปลอดภัย* ที่จะย้าย (move) เอา `Node` หลุดออกมาจาก `Rc` ได้จริง ๆ และนั่นยังทำให้เรารู้อีกด้วยว่าจะต้องหยุดลูปเมื่อไหร่: นั่นคือเมื่อไหร่ก็ตามที่เรา *ไม่สามารถ* ดึงตัว `Node` ออกมาได้อีกแล้ว (เพราะยังมีลิสต์อื่นถือมันอยู่)

และน่าอัศจรรย์จริง ๆ ที่ `Rc` มีเมธอดที่ทำสิ่งนี้ให้เราพอดิบพอดี: นั่นคือ `try_unwrap`:

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

ยอดเยี่ยม! แจ่มไปเลย