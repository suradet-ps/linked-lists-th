# Peek (การมอง)

สิ่งหนึ่งที่เราไม่ได้ bother implement ในครั้งก่อนคือการมอง (peek) มาลองทำกันเถอะ
สิ่งเดียวที่เราต้องทำคือคืนเรเฟอเรนซ์ไปยังองค์ประกอบในหัวของลิสต์ ถ้ามีอยู่จริง
ฟังดูง่าย ลองกัน:

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

*ถอนหายใจ* อีกแล้ว Rust?

Map รับ `self` แบบค่า ซึ่งจะย้าย Option ออกจากสิ่งที่มันอยู่
ก่อนหน้านี้ไม่เป็นไรเพราะเราเพิ่ง `take` มันออกมา แต่ตอนนี้เราจริง ๆ แล้ว
อยากปล่อยมันไว้ที่เดิม วิธีที่*ถูกต้อง*ในการจัดการเรื่องนี้คือผ่านเมธอด
`as_ref` ของ Option ซึ่งมีนิยามดังนี้:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

มันลดรูป `Option<T>` เป็น Option ของเรเฟอเรนซ์ไปยังอินเทอร์นัลของมัน เราสามารถ
ทำเองได้ด้วย match ที่ชัดเจนแต่ *อืม ไม่เอา* มันหมายความว่าเราต้อง dereference
เพิ่มเติมเพื่อตัดผ่านการอ้อมเพิ่มเติม แต่โชคดีที่โอเปอเรเตอร์ `.` จัดการให้เรา

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

สำเร็จ!

เรายังสามารถสร้างเวอร์ชัน*ที่เปลี่ยนแปลงได้*ของเมธอดนี้ด้วย `as_mut`:

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
> cargo build

```

ง่ายสุด!

อย่าลืมทดสอบมัน:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

ดี แต่เราไม่ได้ทดสอบจริง ๆ ว่าเราสามารถกลายพันธุ์ค่าที่ `peek_mut` คืนมาได้หรือไม่
ถ้าเรเฟอเรนซ์เป็นแบบที่เปลี่ยนแปลงได้แต่ไม่มีใครกลายพันธุ์มัน เราได้ทดสอบความเปลี่ยนแปลงได้จริง ๆ หรือเปล่า?
มาลองใช้ `map` กับ `Option<&mut T>` นี้เพื่อใส่ค่าที่ลึกซึ้ง:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

คอมไพเลอร์บ่นว่า `value` เป็นแบบไม่เปลี่ยนแปลงได้ แต่เราเขียน `&mut value` ชัด ๆ
มันกลับกลายเป็นว่าการเขียนอาร์กิวเมนต์ของคลอ저แบบนั้นไม่ได้ระบุว่า `value`
เป็นเรเฟอเรนซ์ที่เปลี่ยนแปลงได้ แต่มันสร้างแพตเทิร์นที่จะจับคู่กับอาร์กิวเมนต์
ของคลอ저 แทน ถ้าเราใช้ `|value|` ชนิดของ `value` จะเป็น `&mut i32`
และเราสามารถกลายพันธุ์หัวได้จริง ๆ:

```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

ดีกว่าเยอะ!