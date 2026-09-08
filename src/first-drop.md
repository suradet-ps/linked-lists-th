# Drop (การปล่อย)

เราสร้างสแต็กได้ ใส่ค่าเข้าไปได้ นำค่าออกได้ และเรายังทดสอบแล้วว่าทุกอย่างทำงานถูกต้อง!

เราจำเป็นต้องกังวลเรื่องการทำความสะอาดลิสต์ของเรามั้ย? ในทางเทคนิคแล้ว ไม่ต้องเลย!
เหมือน C++ Rust ใช้ดีสตรัคเตอร์เพื่อทำความสะอาดทรัพยากรอัตโนมัติเมื่อเลิกใช้
ชนิดหนึ่งจะมีดีสตรัคเตอร์ถ้ามัน implement *เทรต*ที่เรียกว่า Drop
เทรตคือคำหรู ๆ ของ Rust สำหรับอินเทอร์เฟซ เทรต Drop มีอินเทอร์เฟซดังนี้:

```rust ,ignore
pub trait Drop {
    fn drop(&mut self);
}
```

โดยพื้นฐานแล้ว "เมื่อคุณหลุดจากสโคป ฉันจะให้เวลาคุณสักครู่เพื่อเก็บกวาดเรื่องของคุณ"

คุณไม่จำเป็นต้อง implement Drop จริง ๆ ถ้าคุณมีชนิดที่ implement Drop อยู่แล้ว
และสิ่งที่คุณอยากทำก็แค่เรียก*ดีสตรัคเตอร์ของพวกมัน* ในกรณีของ List
ทั้งหมดที่มันอยากทำคือ drop หัวของมัน ซึ่งต่อไปอาจ*บางที*จะพยายาม drop
`Box<Node>` หนึ่งตัว ทั้งหมดนั้นจัดการให้อัตโนมัติ... แต่มีเงื่อนงำหนึ่ง

การจัดการอัตโนมัตินี้จะแย่

ลองพิจารณาลิสต์ง่าย ๆ:

```text
list -> A -> B -> C
```

เมื่อ `list` ถูก drop มันจะพยายาม drop A ซึ่งจะพยายาม drop B
ซึ่งจะพยายาม drop C บางคนอาจเริ่มกังวลอย่างถูกต้อง นี่คือโค้ดแบบรีเคอร์ซีฟ
และโค้ดแบบรีเคอร์ซีฟสามารถทำให้สแต็กแตกได้!

บางคนอาจคิดว่า "นี่ชัดเจนว่าเป็นการเรียกซ้ำแบบท้าย และภาษาไหนก็ตามที่เหมาะสม
จะรับประกันว่าโค้ดแบบนั้นจะไม่ทำให้สแต็กแตก" อันที่จริง นี่*ผิด*! เพื่อดูว่าทำไม
ลองเขียนสิ่งที่คอมไพเลอร์ต้องทำดู โดยการ implement Drop ให้ List ของเราเอง
ในแบบที่คอมไพเลอร์ทำ:

```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        // NOTE: you can't actually explicitly call `drop` in real Rust code;
        // we're pretending to be the compiler!
        self.head.drop(); // tail recursive - good!
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match *self {
            Link::Empty => {} // Done!
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // tail recursive - good!
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // uh oh, not tail recursive!
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```

เรา*ไม่สามารถ* drop เนื้อหาของ Box *หลัง*จากการปลดปล่อยได้ ดังนั้นจึงไม่มีทาง
ที่จะ drop แบบเรียกซ้ำท้าย! แทนที่จะทำแบบนั้น เราจะต้องเขียน drop แบบอีเทอเรทีฟ
ให้ `List` ที่ย้ายโหนดออกจาก box ของพวกมันด้วยตัวเอง

```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == "do this thing until this pattern doesn't match"
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node goes out of scope and gets dropped here;
            // but its Node's `next` field has been set to Link::Empty
            // so no unbounded recursion occurs.
        }
    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

```

เยี่ยม!

---------------------

<span style="float:left">![Bonus](img/profbee.gif)</span>

## ส่วนโบนัสสำหรับการเพิ่มประสิทธิภาพก่อนวัยอันควร!

การ implement drop ของเราจริง ๆ แล้ว*คล้ายมาก*กับ
`while let Some(_) = self.pop() { }` ซึ่งแน่นอนว่าง่ายกว่า มันต่างกันยังไง
และปัญหาด้านประสิทธิภาพอะไรที่อาจเกิดขึ้นจากมัน เมื่อเราเริ่มทำลิสต์ของเรา
เป็นเจเนอริกเพื่อเก็บอย่างอื่นที่ไม่ใช่จำนวนเต็ม?

<details>
  <summary>คลิกเพื่อขยายดูคำตอบ</summary>

Pop คืนค่า `Option<i32>` ในขณะที่การ implement ของเราจัดการแค่ Link (`Box<Node>`)
ดังนั้นการ implement ของเราจึงย้ายแค่ตัวชี้ไปยังโหนด ในขณะที่แบบที่ใช้ pop
จะย้ายค่าที่เราเก็บไว้ในโหนด สิ่งนี้แพงมากได้ ถ้าเราทำลิสต์เป็นเจเนอริก
และมีคนใช้มันเก็บอินสแตนซ์ของ VeryBigThingWithADropImpl (VBTWADI)
Box สามารถรันการ implement ของ drop ของเนื้อหามันในตำแหน่งเดิม (in-place)
ดังนั้นมันจึงไม่เจอปัญหานี้ เพราะ VBTWADI เป็น*ชนิดเดียวกับที่*ทำให้การใช้ลิงก์ลิสต์
น่าดึงดูดใจมากกว่าอาร์เรย์จริง ๆ การทำตัวห่วยต่อกรณีนี้ก็คงน่าผิดหวังไปหน่อย

ถ้าคุณต้องการสิ่งที่ดีที่สุดจากทั้งสองการ implement คุณสามารถเพิ่มเมธอดใหม่
`fn pop_node(&mut self) -> Link` ซึ่ง `pop` และ `drop` ทั้งคู่สามารถสร้างขึ้นมาจากมันได้อย่างสะอาด

</details>