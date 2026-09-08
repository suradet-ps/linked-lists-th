# การทดสอบ

เอาล่ะ ในเมื่อเราเขียนทั้ง `push` และ `pop` เสร็จเรียบร้อยแล้ว ตอนนี้เราก็พร้อมที่จะทดสอบการทำงานของสแต็กของเราจริง ๆ เสียที! ทั้ง Rust และ Cargo รองรับการเขียนเทสต์เป็นฟีเจอร์ระดับเฟิร์สคลาส (first-class feature) อยู่แล้ว ดังนั้นเรื่องนี้จึงง่ายสุด ๆ ทั้งหมดที่เราต้องทำก็เพียงแค่เขียนฟังก์ชันขึ้นมาตัวหนึ่ง แล้วแปะแอตทริบิวต์ `#[test]` ไว้เหนือฟังก์ชันนั้น

โดยปกติในคอมมูนิตี้ Rust เรานิยมวางโค้ดทดสอบไว้ข้าง ๆ โค้ดจริงที่มันกำลังทดสอบอยู่เสมอ แต่เรามักจะแยกเนมสเปซใหม่ออกมาสำหรับเทสต์โดยเฉพาะ เพื่อไม่ให้ชื่อฟังก์ชันไปชนหรือปนกับโค้ด "จริง" และเช่นเดียวกับที่เราใช้คำสั่ง `mod` เพื่อระบุว่าไฟล์ `first.rs` ควรรวมอยู่ใน `lib.rs` เราก็สามารถใช้ `mod` เพื่อสร้างโมดูลย่อยขึ้นมาข้างในไฟล์เดียวกันแบบ *อินไลน์ (inline)* ได้เลย:

```rust ,ignore
// in first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```

และเราสั่งรันเทสต์ได้ง่าย ๆ ด้วยคำสั่ง `cargo test`:

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

ไชโย! เทสต์แบบ "ไม่ทำอะไรเลย" ของเราผ่านฉลุย! ทีนี้มาทำให้มันเริ่มทำงานจริงจังกันดีกว่า เราจะใช้มาโคร `assert_eq!` เข้ามาช่วยตรวจสอบ ซึ่งจริง ๆ นี่ไม่ได้มีเวทมนตร์อะไรซับซ้อนเลย ทั้งหมดที่มันทำก็แค่จับของสองชิ้นที่คุณส่งเข้าไปมาเทียบกัน และถ้ามันไม่ตรงกัน มันก็จะสั่งให้โปรแกรมแพนิก (panic) ทันที ใช่แล้ว! วิธีการที่คุณจะบอกระบบรันเทสต์ว่าเทสต์พัง ก็คือการสั่งให้โปรแกรมเกิดอาการสติแตก (panic) นั่นเอง!

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

อุ๊ปส์! เพราะเราสร้างโมดูลย่อยขึ้นมาใหม่ เราจึงต้องดึง `List` จากโมดูลแม่เข้ามาใช้งานอย่างชัดเจนก่อน:

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

เยี่ยมไปเลย!

แต่เดี๋ยวนะ แล้วข้อความ warning นั่นมันอะไรกัน...? เราก็เรียกใช้ `List` ในเทสต์ของเราอยู่เห็น ๆ นี่นา!

...แต่เราเรียกใช้มัน *เฉพาะตอนรันเทสต์* เท่านั้นไงล่ะ! ดังนั้น เพื่อเอาใจคอมไพเลอร์ (และเพื่อไม่ให้โค้ดส่วนนี้ไปรบกวนผู้ที่นำไลบรารีของเราไปใช้งาน) เราจึงควรระบุให้ชัดเจนว่า โมดูล `test` ทั้งก้อนนี้ จะต้องถูกคอมไพล์ก็ต่อเมื่อเรากำลังสั่งรันเทสต์เท่านั้น:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
}
```

และนั่นคือทั้งหมดของการเขียนเทสต์ครับ!