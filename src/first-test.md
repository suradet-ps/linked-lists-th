# การทดสอบ

เอาล่ะ เรามี `push` และ `pop` แล้ว ตอนนี้เราสามารถทดสอบสแต็กของเราได้จริง ๆ!
Rust และ cargo รองรับการทดสอบเป็นฟีเจอร์ระดับเฟิร์สคลาส ดังนั้นมันจะง่ายสุด ๆ
ทั้งหมดที่เราต้องทำคือเขียนฟังก์ชันหนึ่ง และเติมแอตทริบิวต์ `#[test]` ให้มัน

โดยทั่วไปแล้ว ในคอมมูนิตี้ Rust เราพยายามวางเทสต์ไว้ข้างโค้ดที่มันทดสอบ
อย่างไรก็ตาม เรามักจะสร้างเนมสเปซใหม่สำหรับเทสต์ เพื่อหลีกเลี่ยงการขัดแย้งกับโค้ด "จริง"
เช่นเดียวกับที่เราใช้ `mod` เพื่อระบุว่า `first.rs` ควรรวมอยู่ใน `lib.rs`
เราสามารถใช้ `mod` เพื่อสร้างไฟล์ใหม่ทั้งไฟล์แบบ*อินไลน์*:

```rust ,ignore
// in first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```

และเราเรียกมันด้วย `cargo test`

```text
> cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

เย่ เทสต์ที่ไม่ทำอะไรเลยของเราผ่าน! มาทำให้มันไม่ใช่เทสต์ที่ไม่ทำอะไรเลยกัน
เราจะทำแบบนั้นด้วยมาโคร `assert_eq!` นี่ไม่ใช่เวทมนตร์การทดสอบพิเศษอะไร
ทั้งหมดที่มันทำคือเปรียบเทียบสองสิ่งที่คุณให้มัน แล้วทำให้โปรแกรมแพนิกถ้าพวกมันไม่ตรงกัน
ใช่แล้ว คุณบ่งชี้ความล้มเหลวให้กับโครงสร้างการทดสอบด้วยการทำตัวบ้าระเบิด!

```rust ,ignore
mod test {
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`


```

โอ๊ะ! เพราะเราสร้างโมดูลใหม่ เราจึงต้องดึง List เข้ามาแบบชัดเจนเพื่อใช้มัน

```rust ,ignore
mod test {
    use super::List;
    // everything else the same
}
```

```text
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

เย่!

แต่คำเตือนนั้นมันเรื่องอะไรกัน...? เราใช้ List ในเทสต์ของเราชัด ๆ!

...แต่ใช้ตอนทดสอบเท่านั้น! เพื่อเอาใจคอมไพเลอร์ (และเพื่อเป็นมิตรกับผู้ใช้ของเรา)
เราควรระบุว่าโมดูล `test` ทั้งหมดควรถูกคอมไพล์เฉพาะเมื่อเรารันเทสต์เท่านั้น

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
}
```

และนั่นคือทั้งหมดสำหรับการทดสอบ!