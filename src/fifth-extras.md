# ของแถมเพิ่มเติม

ตอนนี้ `push` และ `pop` เขียนเสร็จแล้ว ทุกอย่างที่เหลือก็เหมือนกันทุกประการกับกรณีสแต็กอย่างน่าประหลาดใจ มีเพียงโอเปอเรชันที่เปลี่ยนความยาวของลิสต์เท่านั้นที่ต้องแตะที่ตัวชี้หาง (tail pointer)

แต่แน่นอน ตอนนี้ทุกอย่างเป็นตัวชี้ดิบ (raw pointer) แล้ว เราจึงต้องเขียนโค้ดใหม่เพื่อใช้สิ่งเหล่านั้น! และถ้าเราจะแตะทุกส่วนของโค้ดอยู่แล้ว เราก็ควรใช้โอกาสนี้ตรวจสอบให้แน่ใจว่าเราไม่ได้พลาดอะไรไป

อย่างไรก็ตาม มาเริ่มคัดลอก-วางโค้ดจากสแต็กกัน:

```rust ,ignore
// ...

pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}
```

IntoIter ดูดีแล้ว แต่ `Iter` และ `IterMut` กำลังละเมิดกฎง่ายๆ ของเราคือไม่ใช้ตัวชี้ที่ปลอดภัย (safe pointer) ในโครงสร้างข้อมูลอีกต่อไป มาทำให้ปลอดภัยโดยเปลี่ยนไปใช้ตัวชี้ดิบ (raw pointer):

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: *mut Node<T>,
}

pub struct IterMut<'a, T> {
    next: *mut Node<T>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head }
    }
}
```

ดูดีแล้ว!

```text
error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:17:17
   |
17 | pub struct Iter<'a, T> {
   |                 ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`

error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:21:20
   |
21 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`
```

ไม่ดีเลย! [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html) ที่พวกเขากล่าวถึงคืออะไร?

> ชนิดขนาดศูนย์ (ZST) ที่ใช้ทำเครื่องหมายสิ่งที่ "ทำตัวเหมือน" เป็นเจ้าของ `T`
>
> การเพิ่มฟิลด์ `PhantomData<T>` ลงในโครงสร้างข้อมูลของคุณจะบอกคอมไพเลอร์ว่าโครงสร้างข้อมูลของคุณทำตัวเหมือนกับว่ามันเก็บค่าชนิด `T` แม้ว่าจริงๆ แล้วจะไม่ได้เก็บก็ตาม ข้อมูลนี้ถูกใช้ในการคำนวณคุณสมบัติความปลอดภัยบางอย่าง
>
> หากต้องการคำอธิบายเชิงลึกเกี่ยวกับวิธีการใช้ `PhantomData<T>` โปรดดู [the Nomicon](https://doc.rust-lang.org/nightly/nomicon/)

เฮ้ อย่ารีบร้อนเลย เรากำลังอ่านหนังสือที่ *ฉัน* เขียน ไม่ใช่หนังสือเล่มอื่นที่ *เนิร์ด* บางคนคงเขียน! ฉันพนันเลยว่าถ้าพวกเขาเขียนโครงสร้างข้อมูลในนั้น มันคงเป็นอะไรที่น่าเบื่อเหมือน Array Stack และ *ไม่* ใช่ Linked List

> ไลฟ์ไทม์ที่ไม่ได้ใช้งาน
>
> กรณีการใช้งานที่พบบ่อยที่สุดสำหรับ PhantomData คือโครงสร้างข้อมูลที่มีพารามิเตอร์ไลฟ์ไทม์ที่ไม่ได้ใช้ โดยทั่วไปเป็นส่วนหนึ่งของโค้ด unsafe

อ้อ แสดงว่าเรากำลังตั้งชื่อไลฟ์ไทม์ในโครงสร้างข้อมูลแต่ไม่ได้ใช้มันจริงๆ เรา *สามารถ* เดินไปทาง PhantomData ได้ แต่ฉันต้องการเก็บไว้สำหรับลิงก์ลิสต์แบบ two-way linked list ในบทถัดไปที่จะ *ต้องการ* จริงๆ

เรากำลังอยู่ในสถานการณ์ที่น่าสนใจเพราะเราจริงๆ แล้วไม่ต้องการ PhantomData *ฉันคิดว่านะ* ฉันจะอ้างแค่ว่ามันเป็นอย่างนั้นและเชื่อว่ามันจริง ถ้า miri ตะโกนใส่เราตอนท้าย ฉันจะยอมรับและเราจะทำ PhantomData

สิ่งที่เราจะทำจริงๆ คือใส่เรเฟอเรนซ์กลับไปในโครงสร้าง Iterator เหล่านี้และมีความสุขที่ยังใช้เรเฟอเรนซ์ได้ในบางที่ ฉันคิดว่ามันสมเหตุสมผลเพราะยังมีการซ้อนทับที่ถูกต้องเมื่อคุณใช้อิเทอเรเตอร์: คุณสร้างอิเทอเรเตอร์ ใช้เรเฟอเรนซ์ที่ปลอดภัยอยู่พักหนึ่ง แล้วก็ทิ้งอิเทอเรเตอร์นั้นไป

เฉพาะเมื่ออิเทอเรเตอร์หายไปแล้วเท่านั้นคุณจึงจะเข้าถึงลิสต์และเรียกเมธอดอย่าง `push` และ `pop` ที่ต้องจัดการกับตัวชี้หางและ Boxes ได้ ในระหว่างการวนซ้ำ เราจะ dereference ตัวชี้ดิบจำนวนมาก ดังนั้นจึงมีการผสมผสานอยู่บ้าง แต่เราควรคิดว่าเรเฟอเรนซ์เหล่านั้นเป็น reborrows ของตัวชี้ดิบได้

*ฉัน* ยังไม่เชื่อ 100% แต่อยากลองดู!

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            Iter { next: self.head.as_ref() }
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe {
            IterMut { next: self.head.as_mut() }
        }
    }
}
```

ถ้าเราจะเก็บเรเฟอเรนซ์ เราต้องอัปเกรดตัวชี้ดิบ (raw pointer) ของเราเป็น Option ของเรเฟอเรนซ์ เรา *สามารถ* ตรวจสอบว่าตัวชี้เป็น null ได้ แต่นี่เป็นหนึ่งในกรณีที่แคบมากๆ ที่ *ฉันคิดว่า* ปลอดภัยที่จะใช้เมธอด [ptr::as_ref](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1) และ [ptr::as_mut](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut) ที่น่าเกลียดเหล่านั้น

ฉัน *โดยทั่วไป* แนะนำให้หลีกเลี่ยงเมธอดเหล่านี้เหมือนโรคระบาด เพราะมันทำสิ่งที่น่าประหลาดใจและน่าเกลียด และมันจะนำเรเฟอเรนซ์กลับมาโดยเนื้อแท้ ทั้งที่ "กฎง่ายๆ" ของฉันคือหลีกเลี่ยงการทำอย่างนั้น!

เมธอดเหล่านั้นมีคำเตือนมากมาย แต่สิ่งที่น่าสนใจที่สุดคือ:

> คุณต้องบังคับใช้กฎการแอลิเอสของ Rust เนื่องจากไลฟ์ไทม์ `'a` ที่ถูกส่งกลับถูกเลือกโดยพลการและไม่จำเป็นต้องสะท้อนไลฟ์ไทม์จริงของข้อมูล โดยเฉพาะอย่างยิ่ง ในช่วงระยะเวลาของไลฟ์ไทม์นี้ หน่วยความจำที่ตัวชี้ชี้อยู่จะต้องไม่ถูกเข้าถึง (อ่านหรือเขียน) ผ่านตัวชี้อื่น

เฮ้ ดูสิ มันคือสิ่งที่เราคุยกันมา 25 หน้า! ฉันได้assert แล้วว่าเราจะ *ต้อง* ใช้เรเฟอเรนซ์ที่นี่ได้โดยไม่มีปัญหา ดังนั้นปัญหาการแอลิเอสจึงแก้ไขแล้ว! ส่วนที่น่ากลัวอีกอย่างคือ signature:

```rust ,ignore
pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T>
```

คุณเห็นไหมว่าไลฟ์ไทม์นั้นไม่ได้เชื่อมต่อกับ input เลย เพราะ `self` เป็นค่า? นั่นคือสิ่งที่เราเรียกว่า "unbounded lifetime" และมันเป็นสิ่งที่น่าเกลียด มันยินดีแกล้งทำเป็นใหญ่เท่าที่เราต้องการ แม้แต่ `'static`! วิธี *จัดการ* กับมันคือใส่ไว้ในที่ที่ *ถูกกำหนด* ซึ่งโดยทั่วไปหมายความว่า "ส่งกลับจากฟังก์ชันโดยเร็วที่สุดเพื่อให้ signature ของฟังก์ชันจำกัดมัน"

โว้ว ฉันกังวลเรื่องนี้มากแต่เราจะเดินหน้าต่อไป! มาขโมยโค้ดจากอิเทอเรเตอร์ของสแต็ก:

```rust ,ignore
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.map(|node| {
                self.next = node.next.as_ref();
                &node.elem
            })
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.take().map(|node| {
                self.next = node.next.as_mut();
                &mut node.elem
            })
        }
    }
}
```

ถึงเวลาแห่งความจริงแล้ว...

```text
cargo test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test third::test::basics ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

ใช่!!! เอาล่ะ **ผู้เล่าเรื่อง**! บางครั้งฉันก็ไม่ได้ทำผิด!

> **ผู้เล่าเรื่อง**: แต่จุดประสงค์ทั้งหมดไม่ใช่หรือว่าความผิดพลาดมีไว้เพื่อสอนผู้อ่าน

ก็ใช่สิ บางครั้งบทเรียนก็คือฉันถูกต้องและทุกคนควรเชื่อฉันเมื่อฉันพูดถึงโค้ด unsafe เพราะฉันใช้เวลาคิดเรื่องความสมเหตุสมผลของการใช้งานอิเทอเรเตอร์นานเกินไปแล้ว?! โอเค?! โอเค

เอาล่ะ นี่คือ `peek` และ `peek_mut`

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref()
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut()
    }
}
```

ฉันจะไม่ทดสอบมันด้วยซ้ำเพราะฉันไม่ทำผิดอีกแล้ว

> **ผู้เล่าเรื่อง**: `cargo build`

```text
error[E0308]: mismatched types
  --> src\fifth.rs:66:13
   |
25 | impl<T> List<T> {
   |      - this type parameter
...
64 |     pub fn peek(&self) -> Option<&T> {
   |                           ---------- expected `Option<&T>` 
   |                                      because of return type
65 |         unsafe {
66 |             self.head.as_ref()
   |             ^^^^^^^^^^^^^^^^^^ expected type parameter `T`, 
   |                                found struct `fifth::Node`
   |
   = note: expected enum `Option<&T>`
              found enum `Option<&fifth::Node<T>>`

```

ก็ได้

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref().map(|node| &node.elem)
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut().map(|node| &mut node.elem)
    }
}
```

ฉันเดาว่าฉันจะ *ยังคง* ทำผิดต่อไป ดังนั้นเราจะระมัดระวังเป็นพิเศษและเพิ่มเทสต์ใหม่ที่ฉันจะเรียกว่า "miri food": บางสิ่งที่แค่เล่นสนุกและใช้งาน API ของเราสลับไปมาเพื่อช่วยให้ miri จับข้อผิดพลาดของเราได้

```rust ,ignore
#[test]
fn miri_food() {
    let mut list = List::new();

    list.push(1);
    list.push(2);
    list.push(3);

    assert!(list.pop() == Some(1));
    list.push(4);
    assert!(list.pop() == Some(2));
    list.push(5);

    assert!(list.peek() == Some(&3));
    list.push(6);
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&30));
    assert!(list.pop() == Some(30));

    for elem in list.iter_mut() {
        *elem *= 100;
    }

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&400));
    assert_eq!(iter.next(), Some(&500));
    assert_eq!(iter.next(), Some(&600));
    assert_eq!(iter.next(), None);
    assert_eq!(iter.next(), None);

    assert!(list.pop() == Some(400));
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&5000));
    list.push(7);

    // Drop it on the ground and let the dtor exercise itself
}
```


```text
cargo test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out



MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

สมบูรณ์แบบ
