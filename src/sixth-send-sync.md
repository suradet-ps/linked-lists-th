# Send, Sync และการทดสอบการคอมไพล์

เอาจริง ๆ เรายังมี trait อีกคู่หนึ่งที่ต้องคำนึงถึง แต่สองตัวนี้พิเศษกว่าเพื่อน เรากำลังจะต้องรับมือกับ "จักรวรรดิโรมันอันศักดิ์สิทธิ์" (Holy Roman Empire) แห่งโลก Rust: นั่นคือ Unsafe Opt-In Built-In Traits (OIBITs): [Send และ Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html) ซึ่งในความเป็นจริงแล้ว มันดันเป็น opt-out และ built-out ซะงั้น (ถูกต้อง 1 ใน 3 ข้อก็ถือว่าบุญโขแล้วล่ะ!)

เฉกเช่นเดียวกับ `Copy` เจ้า trait สองตัวนี้ไม่มีโค้ดการทำงานผูกติดอยู่เลยแม้แต่น้อย มันทำหน้าที่เป็นเพียงตัวมาร์ก (marker) บ่งบอกว่า type ของคุณมีคุณสมบัติเฉพาะตัวบางประการ โดย `Send` จะบอกว่า type ของคุณปลอดภัยที่จะส่งข้ามไปยัง thread อื่น ส่วน `Sync` จะบอกว่า type ของคุณปลอดภัยที่จะแชร์ข้าม thread (`&Self: Send`)

เหตุผลเดียวกับที่ทำให้ `LinkedList` เป็น covariant ก็สามารถนำมาใช้กับตรงนี้ได้เช่นกัน: โดยทั่วไปแล้วคอลเลกชันธรรมดา ๆ ที่ไม่ได้ใช้ลูกเล่น interior mutability พิสดาร ย่อมปลอดภัยที่จะกำหนดให้เป็น `Send` และ `Sync`

แต่ผมเพิ่งบอกไปว่าพวกมันเป็น *opt-out* (มีให้ตั้งแต่เกิด ถ้าไม่ต้องการต้องสั่งยกเว้นเอาเอง) ถ้างั้นจริง ๆ แล้วตอนนี้โค้ดเรามีคุณสมบัติพวกนี้อยู่แล้วหรือยังล่ะ? แล้วเราจะรู้ได้ยังไง?

ลองมาเพิ่มเวทมนตร์เล็ก ๆ น้อย ๆ ลงไปในโค้ดของเรากันครับ: เป็นฟังก์ชันขยะส่วนตัวแบบ private ที่จะคอมไพล์ไม่ผ่านเด็ดขาด เว้นแต่ว่าบรรดา type ของเราจะมีคุณสมบัติครบถ้วนตามที่เราคาดหวังไว้:

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

โอ้โห ให้ตายสิ อะไรกันเนี่ย! อุตส่าห์เล่นมุกจักรวรรดิโรมันอันศักดิ์สิทธิ์ซะดิบดี!

ก็นะ ผมแอบโกหกคุณไปตอนที่บอกว่า raw pointer มีเกราะป้องกันความปลอดภัยแค่อันเดียว: เพราะนี่คืออีกอันหนึ่งครับ ทั้ง `*const` และ `*mut` ต่าง opt-out ไม่รับ `Send` และ `Sync` โดยชัดแจ้งเพื่อความปลอดภัย ดังนั้นเราจึงต้อง opt-in กลับเข้ามาใหม่ *จริง ๆ*:

```rust ,ignore
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```

สังเกตว่าเราต้องเขียน *unsafe impl* ตรงนี้: เพราะพวกมันคือ *unsafe trait*! โค้ด unsafe อื่น ๆ (เช่น ไลบรารีการประมวลผลพร้อมกัน / concurrency) จะต้องพึ่งพาและเชื่อใจว่าเราอิมพลีเมนต์ trait เหล่านี้อย่างถูกต้องเท่านั้น! และเนื่องจากมันไม่มีเนื้อโค้ดจริง ๆ สิ่งที่เรารับประกันก็มีเพียงแค่ว่า ใช่ครับ โค้ดของเราปลอดภัยที่จะส่ง (`Send`) หรือแชร์ (`Sync`) ข้าม thread ได้อย่างแน่นอน!

อย่าเที่ยวเอาไปแปะใส่มั่วซั่วส่งเดชนะครับ แต่ในฐานะที่ผมเป็นมืออาชีพที่ได้รับใบรับรอง (Certified Professional) ขอยืนยันตรงนี้เลยว่า: ใช่ครับ ใส่แบบนี้ปลอดภัยร้อยเปอร์เซ็นต์ สังเกตว่าเราไม่จำเป็นต้องอิมพลีเมนต์ `Send` และ `Sync` ให้กับ `IntoIter` เลย: เพราะมันห่อหุ้มแค่ `LinkedList` ตัวเดียว ดังนั้นมันจึง auto-derive ทั้ง `Send` และ `Sync` ให้อัตโนมัติ — ผมบอกแล้วไงว่าจริง ๆ แล้วพวกมันเป็น opt-out! (คุณสามารถ opt-out ออกมาได้ด้วยไวยากรณ์สุดฮาอย่าง `impl !Send for MyType {}`)

```text
cargo build
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
```

โอเค เยี่ยมเลย!

...เดี๋ยวนะ เอาจริง ๆ มันจะอันตรายมากถ้าหากสิ่งที่ *ไม่ควร* มีคุณสมบัติพวกนี้ ดันทะลึ่งมีขึ้นมา โดยเฉพาะอย่างยิ่ง `IterMut` ที่ *ไม่ควร* เป็น covariant อย่างเด็ดขาด เพราะมัน "ประหนึ่งว่า" เป็น `&mut T` แต่เราจะเช็กเรื่องนี้ได้ยังไงล่ะ?

ด้วยพลังเวทมนตร์ไงครับ! อ่า จริง ๆ คือด้วย `rustdoc` ต่างหาก! โอเค เราไม่ได้จำเป็นต้องใช้ `rustdoc` สำหรับเรื่องนี้หรอก แต่มันเป็นวิธีที่ปั่นประสาทและฮาที่สุด ลองดูสิครับ ถ้าคุณเขียน doc-comment แล้วใส่บล็อกโค้ดลงไป เจ้า `rustdoc` จะพยายามคอมไพล์และรันมันขึ้นมา ดังนั้นเราจึงสามารถใช้ประโยชน์จากตรงนี้ในการสร้าง "โปรแกรมอิสระแบบไม่ระบุชื่อ" ขึ้นมาใหม่โดยไม่กระทบกับโปรแกรมหลักของเราได้เลย:

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

โอเค เจ๋งเป้ง เราพิสูจน์ได้แล้วว่ามันเป็น invariant แต่เอ่อ... ตอนนี้เทสต์ของเราพังยับเลยแฮะ ไม่ต้องห่วงครับ `rustdoc` ยอมให้เราบอกมันได้ว่าอาการคอมไพล์พังนี้เป็นสิ่งที่เราตั้งใจไว้ โดยการใส่คำกำกับ `compile_fail` ไว้ที่หัวบล็อกโค้ด!

(จริง ๆ แล้วสิ่งที่เราเพิ่งพิสูจน์ไปคือมัน "ไม่เป็น covariant" เท่านั้นแหละครับ แต่เอาจริง ถ้าคุณสามารถทำให้ type กลายเป็น "contravariant โดยบังเอิญและผิดพลาด" ได้เนี่ย... ผมก็คงต้องขอยกมือไหว้แสดงความยินดีด้วยแล้วล่ะ?)

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

เย้! ผมแนะนำว่าให้ลองเขียนเทสต์แบบไม่ใส่ `compile_fail` ดูก่อนเสมอ เพื่อที่คุณจะได้มั่นใจว่ามันคอมไพล์พัง *ด้วยเหตุผลที่ถูกต้องจริง ๆ* ตัวอย่างเช่น เทสต์นั้นจะคอมไพล์พังเหมือนกัน (และทำให้เทสต์ผ่านเฉยเลย) ถ้าคุณดันลืมเขียน `use` ซึ่งนั่นไม่ใช่สิ่งที่เราต้องการจะทดสอบเลยสักนิด! แม้ในทางทฤษฎี การที่เราสามารถ "ระบุบังคับ" ข้อความ error เจาะจงจากคอมไพเลอร์ได้จะดูเย้ายวนใจมากแค่ไหน แต่มันจะกลายเป็นฝันร้ายทันที เพราะมันจะเท่ากับว่า *การที่คอมไพเลอร์พัฒนาให้แสดง error ได้ดีขึ้นและชัดเจนขึ้นจะกลายเป็น breaking change ไปโดยปริยาย* เราอยากให้คอมไพเลอร์ฉลาดขึ้นเรื่อย ๆ ดังนั้น เสียใจด้วยครับ คุณจะมาทำแบบนั้นไม่ได้

(อ๊ะ เดี๋ยวก่อน จริง ๆ เราสามารถระบุรหัส error code ที่ต้องการไว้ข้างหลัง `compile_fail` ได้เลยนะ **แต่มันใช้ได้เฉพาะบน Rust nightly เท่านั้น และมันเป็นความคิดที่แย่มากที่จะไปพึ่งพามันด้วยเหตุผลที่เราเพิ่งว่าไปข้างต้น แถมมันจะถูกเพิกเฉยทิ้งไปเงียบ ๆ บนเวอร์ชันที่ไม่ใช่ nightly อีกต่างหาก**)

```rust ,ignore
    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

...อีกอย่าง คุณสังเกตเห็นตรงจุดที่เราทำให้ `IterMut` กลายเป็น invariant จริง ๆ หรือเปล่าครับ? มันมองข้ามได้ง่ายมาก เพราะผม "แค่" ก็อปแปะ `Iter` แล้วเอามาโยนทิ้งไว้ท้ายไฟล์ บรรทัดนั้นคือบรรทัดสุดท้ายตรงนี้ครับ:

```rust ,ignore
pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}
```

มาลองลบ `PhantomData` ออกดู:

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

ฮ่า! คอมไพเลอร์คอยหนุนหลังเราอยู่เสมอ และไม่ยอมปล่อยให้เรา *ไม่ใช้งาน* lifetime ลอย ๆ แน่นอน งั้นลองมาใส่ตัวอย่างที่ *ผิด* ดูแทนดีกว่า:

```rust ,ignore
    _boo: PhantomData<&'a T>,
```

```text
cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
```

คอมไพล์ผ่านเฉยเลย! แล้วคราวนี้เทสต์ของเราจะตรวจจับความผิดปกตินี้ได้ไหมนะ?

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

เอ้าาาา!!! ระบบมันใช้งานได้จริงโว้ย! ผมชอบจริง ๆ เวลาที่มีเทสต์ที่ทำหน้าที่ของมันได้อย่างถูกต้องสมบูรณ์แบบ เราจะได้ไม่ต้องมานั่งประสาทหลอนกับความผิดพลาดที่จ้องจะโผล่มาตลบหลังเราในภายหลัง!