# Send, Sync และ Compile Tests

จริงๆ แล้วเรายังมี trait อีกคู่หนึ่งที่ต้องคิดถึง แต่มันพิเศษ เราต้องจัดการกับจักรวรรดิโรมันอันศักดิ์สิทธิ์ของ Rust: Unsafe Opt-In Built-In Traits (OIBITs): [Send and Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html) ซึ่งจริงๆ แล้วเป็น opt-out และ built-out (ได้ 1 จาก 3 ก็ถือว่าดีแล้ว!)

เช่นเดียวกับ Copy trait เหล่านี้ไม่มีโค้ดเลย แค่เป็นเครื่องหมายบอกว่า type ของคุณมีคุณสมบัติบางอย่าง Send บอกว่า type ของคุณปลอดภัยที่จะส่งไปยัง thread อื่น Sync บอกว่า type ของคุณปลอดภัยที่จะแชร์ระหว่าง thread (&Self: Send)

เหตุผลเดียวกันกับ LinkedList ที่เป็น covariance ใช้ได้ที่นี่: ปกติ collection ทั่วไปที่ไม่ใช้ interior mutability tricks ซับซ้อนจะปลอดภัยที่จะทำให้เป็น Send และ Sync

แต่ผมบอกว่ามันเป็น *opt out* แล้วตอนนี้เราเป็นแล้วหรือยัง? แล้วเราจะรู้ได้อย่างไร?

มาเพิ่ม magic ใหม่ในโค้ดกัน: สุ่มข้อมูลส่วนตัวที่จะ compile ไม่ผ่านถ้า type ของเราไม่มีคุณสมบัติที่คาดหวัง:

```rust ,ignore
#[allow(dead_code)]
fn assert_properties() {
    fn is_send<T: Send>() {}
    fn is_sync<T: Sync>() {}

    is_send::<LinkedList<i32>>();
    is_sync::<LinkedList<i32>>();

    is_send::<IntoIter<i32>>();
    is_sync::<IntoIter<i32>>();

    is_send::<Iter<i32>>();
    is_sync::<Iter<i32>>();

    is_send::<IterMut<i32>>();
    is_sync::<IterMut<i32>>();

    is_send::<Cursor<i32>>();
    is_sync::<Cursor<i32>>();

    fn linked_list_covariant<'a, T>(x: LinkedList<&'static T>) -> LinkedList<&'a T> { x }
    fn iter_covariant<'i, 'a, T>(x: Iter<'i, &'static T>) -> Iter<'i, &'a T> { x }
    fn into_iter_covariant<'a, T>(x: IntoIter<&'static T>) -> IntoIter<&'a T> { x }
}
```

```text
cargo build
   Compiling linked-list v0.0.3 
error[E0277]: `NonNull<Node<i32>>` cannot be sent between threads safely
   --> src\lib.rs:433:5
    |
433 |     is_send::<LinkedList<i32>>();
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ `NonNull<Node<i32>>` cannot be sent between threads safely
    |
    = help: within `LinkedList<i32>`, the trait `Send` is not implemented for `NonNull<Node<i32>>`
    = note: required because it appears within the type `Option<NonNull<Node<i32>>>`
note: required because it appears within the type `LinkedList<i32>`
   --> src\lib.rs:8:12
    |
8   | pub struct LinkedList<T> {
    |            ^^^^^^^^^^
note: required by a bound in `is_send`
   --> src\lib.rs:430:19
    |
430 |     fn is_send<T: Send>() {}
    |                   ^^^^ required by this bound in `is_send`

<a million more errors>
```

โอ้ บ้าแล้ว เป็นไปได้ยังไง! ผมมีมุกจักรวรรดิโรมันอันศักดิ์สิทธิ์ดีๆ อยู่เชียวนะ!

อืม ผมโกหกคุณตอนที่บอกว่า raw pointer มี safety guard แค่อันเดียว: นี่คืออีกอันหนึ่ง `*const` และ `*mut` opt out จาก Send และ Sync เพื่อความปลอดภัย เพราะฉะนั้นเราต้อง opt back in *จริงๆ*:

```rust ,ignore
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```

สังเกตว่าเราต้องเขียน *unsafe impl* ที่นี่: เหล่านี้เป็น *unsafe traits*! Unsafe code (เช่น concurrency libraries) จะต้องพึ่งพาเราที่จะ implement traits เหล่านี้ได้ถูกต้อง! เนื่องจากไม่มีโค้ดจริง การรับประกันที่เราทำก็แค่ ใช่ เราปลอดภัยจริงๆ ที่จะ Send หรือ Share ระหว่าง thread!

อย่าเพิ่ง slapt เหล่านี้อย่างลวกๆ แต่ผมเป็น Certified Professional ที่มาบอกว่า: โอเค อันนี้โอเคจริงๆ สังเกตว่าเราไม่ต้อง implement Send และ Sync สำหรับ IntoIter: มันแค่มี LinkedList อยู่ในนั้น เพราะฉะนั้นมันจะ auto-derive Send และ Sync &mdash; ผมบอกแล้วว่าจริงๆ มันเป็น opt out! (คุณ opt out ด้วย syntax ที่น่าขำคือ `impl !Send for MyType {}`.)

```text
cargo build
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
```

โอ้ ดีมาก!

...เดี๋ยวก่อน จริงๆ แล้วมันอันตรายมากถ้าสิ่งที่ *ไม่ควร* เป็นสิ่งเหล่านี้กลับไม่เป็น โดยเฉพาะ IterMut *ไม่ควร* เป็น covariance อย่างเด็ดขาด เพราะมัน "เหมือน" `&mut T` แต่เราจะตรวจสอบได้อย่างไร?

ด้วย Magic! เอ้ย จริงๆ แล้วด้วย rustdoc! โอเค เราไม่จำเป็นต้องใช้ rustdoc สำหรับเรื่องนี้ แต่มันเป็นวิธีที่ตลกที่สุด ดูสิ ถ้าคุณเขียน doc comment และใส่ code block rustdoc จะพยายาม compile และ run มัน เพราะฉะนั้นเราสามารถใช้มันเพื่อสร้าง "program" ใหม่ๆ ที่ไม่มีผลกับ program หลัก:

```rust ,ignore
    /// ```
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
error[E0308]: mismatched types
 --> src\lib.rs:461:86
  |
6 | fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
  |                                                                                      ^ lifetime mismatch
  |
  = note: expected struct `linked_list::IterMut<'_, &'a T>`
             found struct `linked_list::IterMut<'_, &'static T>`
```

โอ้ เจ๋งเลย เราพิสูจน์แล้วว่ามันเป็น invariant แต่อืม ตอนนี้ test ของเรา fail ไม่ต้องห่วง rustdoc ให้คุณบอกว่ามัน expected โดย annotate fence ด้วย compile_fail!

(จริงๆ เราพิสูจน์แค่ว่ามัน "ไม่ใช่ covariance" แต่จริงๆ ถ้าคุณจัดการกับ type ให้เป็น "contravariance แบบไม่ตั้งใจและไม่ถูกต้อง" ได้ ขอแสดงความยินดีด้วย!)

```rust ,ignore
    /// ```compile_fail
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.49s
     Running unittests src\lib.rs

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

เย้! ผมแนะนำให้ทำ test โดยไม่มี compile_fail เสมอเพื่อให้คุณยืนยันว่ามัน fail ที่จะ compile *ด้วยเหตุผลที่ถูกต้อง* ตัวอย่างเช่น test นั้นก็จะ fail (และ pass ด้วย) ถ้าคุณลืม `use` ซึ่งไม่ใช่สิ่งที่เราต้องการ! แม้ว่ามันจะน่าดึงดูดที่จะ "กำหนด" error ที่เฉพาะจาก compiler แต่จะเป็นฝันร้ายที่จะทำให้มันเป็น breaking change *สำหรับ compiler ที่จะ produce error ที่ดีขึ้น* เราต้องการให้ compiler ดีขึ้น เพราะฉะนั้น ไม่ คุณไม่สามารถทำแบบนั้นได้

(โอ้ เดี๋ยว เราสามารถระบุ error code ที่ต้องการข้าง compile_fail **แต่มันใช้ได้แค่บน nightly และเป็นความคิดที่ไม่ดีที่จะพึ่งพาเพราะเหตุผลที่ระบุไว้ข้างต้น มันจะถูก ignore โดยไม่เตือนบนไม่ใช่ nightly**)

```rust ,ignore
    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

...นอกจากนี้ คุณสังเกตเห็นส่วนที่เราทำให้ IterMut เป็น invariant จริงๆ ไหม? มันง่ายที่จะพลาด เพราะผม "แค่" copy-paste Iter มาแล้ว put ไว้ท้าย มันคือบรรทัดสุดท้ายที่นี่:

```rust ,ignore
pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}
```

มาลองลบ PhantomData ออก:

```text
 cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
error[E0392]: parameter `'a` is never used
  --> src\lib.rs:30:20
   |
30 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, or using a marker such as `PhantomData`
```

ฮา! Compiler คอยดูแลเราและไม่ปล่อยให้เรา *ไม่ใช้* lifetime มาลองใช้ตัวอย่าง *ผิด* แทน:

```rust ,ignore
    _boo: PhantomData<&'a T>,
```

```text
cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
```

มัน compile ผ่าน! Test ของเราจับปัญหาได้ไหมตอนนี้?

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
Test compiled successfully, but it's marked `compile_fail`.

failures:
    src\lib.rs - assert_properties::iter_mut_invariant (line 458)

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.15s
```

เย้ย!!! ระบบทำงานได้! ผมชอบที่มี test ที่ทำงานได้จริง ดังนั้นผมไม่ต้องกลัวความผิดพลาดที่กำลังจะเกิด!