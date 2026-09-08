# Peek (การแอบดูข้อมูล)

สิ่งหนึ่งที่เราไม่ได้สนใจจะ implement เลยในบทที่แล้วก็คือการแอบดูข้อมูลที่หัวลิสต์ (peeking) งั้นเรามาทำเรื่องนี้กันเลยดีกว่า ทั้งหมดที่เราต้องทำก็เพียงแค่ส่งเรเฟอเรนซ์ของข้อมูลที่อยู่ในหัวลิสต์กลับออกไป (ถ้ามันมีข้อมูลอยู่นะ) ฟังดูเหมือนง่ายใช่ไหมล่ะ? งั้นมาลองเขียนกันเลย:

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

*(ถอนหายใจยาวววว...)* คราวนี้มีปัญหาอะไรอีกล่ะจ๊ะ Rust?

สาเหตุก็เพราะว่า `map` จะรับค่า `self` เข้าไปแบบส่งผ่านด้วยค่า (by value) ซึ่งมันจะพยายามย้าย (move) ตัว `Option` ออกมาจากเจ้าของของมัน ที่ผ่านมาโค้ดเราไม่พังก็เพราะเราสั่ง `take()` ขโมยมันออกมาก่อน แต่ในกรณีนี้เราแค่อยากแอบดูข้อมูลเฉย ๆ และอยากปล่อยให้มันอยู่ที่เดิม! ซึ่งวิธีที่ *ถูกต้อง* ในการรับมือกับเรื่องนี้ก็คือการเรียกใช้เมธอด `as_ref` ของ `Option` ซึ่งมีนิยามดังนี้:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

เมธอดนี้จะแปลง `Option<T>` ให้กลายเป็น Option ที่เก็บเรเฟอเรนซ์ชี้เข้าไปยังข้อมูลข้างในแทน นั่นคือ `Option<&T>` จริงอยู่ว่าเราจะเขียน `match` เพื่อแกะกล่องออกมาเองก็ได้นะ แต่ *แหวะ... อย่าหาทำเลย* การทำแบบนี้หมายความว่าเราต้องทำการดีเรเฟอเรนซ์ (dereference) เพิ่มอีกหนึ่งชั้นเพื่อทะลวงผ่านการชี้ทางอ้อม แต่โชคดีมากที่ตัวดำเนินการจุด (`.`) ของ Rust ช่วยจัดการดีเรฟให้เราโดยอัตโนมัติ

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

สำเร็จเป๊ะตามเป้า!

นอกจากนี้ เรายังสามารถเขียนเวอร์ชัน *ที่แก้ไขค่าได้* ของเมธอดนี้ได้ง่าย ๆ ด้วยการใช้ `as_mut`:

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

ง่ายจนแทบไม่ต้องออกแรง!

และอย่าลืมเขียนเทสต์ทดสอบมันด้วยนะ:

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

ดูดีเลยทีเดียว แต่เดี๋ยวก่อนนะ... เรายังไม่ได้ทดสอบจริง ๆ จัง ๆ เลยนี่นาว่าเราสามารถแก้ไขเปลี่ยนแปลงค่าที่ `peek_mut` ส่งกลับออกมาได้จริงหรือเปล่า? เพราะถ้าเรเฟอเรนซ์เป็นแบบ mutable แต่ไม่เคยมีใครเข้าไปแก้ค่ามันเลย เราจะพูดได้เต็มปากจริง ๆ เหรอว่าเราได้ทดสอบความ mutability แล้ว? งั้นเรามาลองใช้ `map` บน `Option<&mut T>` ตัวนี้เพื่อยัดค่าที่มีความหมายลึกซึ้ง (อย่างเลข 42) ลงไปดูซิ:

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

อ้าว คอมไพเลอร์ฟ้อง error เฉยเลยว่า `value` เป็นตัวแปรที่แก้ไขค่าไม่ได้ (immutable) ทั้ง ๆ ที่เราก็พิมพ์ว่า `&mut value` อยู่ชัด ๆ นี่นา มันเกิดอะไรขึ้นกันแน่? ความจริงก็คือ การเขียนพารามิเตอร์ของโคลเชอร์แบบนั้น ไม่ได้แปลว่า `value` เป็นเรเฟอเรนซ์ที่แก้ไขได้ครับ แต่มันเป็นการสร้างแพตเทิร์นที่จะนำไปจับคู่ (pattern match) กับอาร์กิวเมนต์ที่ส่งเข้ามาในโคลเชอร์ต่างหาก โดย `|&mut value|` จะมีความหมายว่า "อาร์กิวเมนต์ที่ส่งเข้ามาเป็นเรเฟอเรนซ์ที่แก้ไขได้นะ แต่ช่วยก็อปปี้ค่าที่มันชี้อยู่มาใส่ไว้ในตัวแปร `value` ให้หน่อย" ดังนั้น ถ้าเราเปลี่ยนมาเขียนเป็น `|value|` ธรรมดา ชนิดข้อมูลของ `value` ก็จะเป็น `&mut i32` ตัวจริงเสียงจริง และเราก็สามารถแก้ไขค่าที่หัวลิสต์ได้จริง ๆ เสียที:

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

แจ่มกว่าเดิมเยอะเลย!