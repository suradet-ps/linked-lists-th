# ของแถมเพิ่มเติม

ตอนนี้ `push` และ `pop` ก็เขียนเสร็จเรียบร้อยแล้ว ส่วนฟังก์ชันอื่นๆ ที่เหลือ แปลกดีที่มันเหมือนกับกรณีของสแต็กแทบทุกกระเบียดนิ้ว เพราะมีเพียง operation ที่ทำให้ความยาวของลิสต์เปลี่ยนไปเท่านั้นที่จำเป็นต้องแตะต้องตัว tail pointer

แต่แน่นอนว่า ในเมื่อตอนนี้ทุกสิ่งทุกอย่างกลายเป็น unsafe pointer ไปหมดแล้ว เราจึงต้องมารื้อเขียนโค้ดใหม่ให้สอดคล้องกัน! และไหนๆ เราก็ต้องมารื้อแตะโค้ดทุกส่วนกันอยู่แล้ว เราก็ถือโอกาสนี้ตรวจเช็กให้ชัวร์ไปเลยว่าไม่ได้ทำอะไรตกหล่นไป

เอาล่ะ มาเริ่มก็อปปี้โค้ดจากเวอร์ชันสแต็กมาแปะกันเลย:

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

`IntoIter` ดูไม่มีปัญหาอะไร แต่ตัว `Iter` กับ `IterMut` ดันแอบแหกกฎง่ายๆ ของเราที่ว่าจะไม่ยอมใช้ safe pointer ใน struct อีกต่อไป เพื่อความปลอดภัย เรามาเปลี่ยนพวกมันให้ใช้ raw pointer กันดีกว่า:

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

ดูเข้าท่าดีนี่!

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

ไม่รอดแฮะ! แล้วไอ้ [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html) ที่คอมไพเลอร์แนะนำมานี่มันคืออะไรกันล่ะ?

> ชนิดข้อมูลขนาดศูนย์ไบต์ (Zero-Sized Type หรือ ZST) ที่ใช้สำหรับระบุเครื่องหมายว่า struct นี้ "ทำตัวเสมือนกับว่า" เป็นเจ้าของ `T`
>
> การใส่ฟิลด์ `PhantomData<T>` เข้าไปใน struct จะเป็นการบอกให้คอมไพเลอร์ทราบว่า struct ของคุณมีพฤติกรรมเสมือนกับว่ามันเก็บค่าชนิด `T` เอาไว้ข้างใน แม้ว่าในความเป็นจริงจะไม่ได้เก็บก็ตาม ข้อมูลนี้จะถูกนำไปใช้ในการวิเคราะห์คุณสมบัติด้านความปลอดภัยบางประการ
>
> หากต้องการคำอธิบายเชิงลึกเกี่ยวกับวิธีการใช้งาน `PhantomData<T>` สามารถอ่านเพิ่มเติมได้ที่ [The Rustonomicon](https://doc.rust-lang.org/nightly/nomicon/)

เฮ้ยยย อย่าเพิ่งใจร้อนสิครับ! คุณกำลังอ่านหนังสือที่ *ผม* เขียนนะ ไม่ใช่หนังสือเล่มอื่นที่พวก *เนิร์ด* คนไหนก็ไม่รู้เขียนสักหน่อย! ผมพนันได้เลยว่าถ้าในนั้นมีสอนเรื่องโครงสร้างข้อมูล มันคงเป็นอะไรที่น่าเบื่ออย่าง Array Stack แน่ๆ และ*ไม่ใช่* Linked List ชัวร์!

> ไลฟ์ไทม์ที่ไม่ได้ถูกใช้งาน (Unused lifetimes)
>
> กรณีการใช้งานที่พบบ่อยที่สุดสำหรับ PhantomData คือการที่ struct มีการประกาศ generic lifetime parameter เอาไว้ แต่ไม่ได้นำไปใช้งานในฟิลด์ใดๆ เลย ซึ่งมักจะพบได้บ่อยในโค้ดฝั่ง unsafe

อ๋อ... แปลว่าเราดันไปประกาศชื่อ lifetime ไว้ใน struct แต่ไม่ได้หยิบมันมาใช้งานในฟิลด์ไหนเลย เรา*จะ*แก้ด้วย PhantomData ก็ได้นะ แต่ผมอยากเก็บมุกนี้เอาไว้ใช้กับ doubly-linked list ในบทถัดไปมากกว่า เพราะบทนั้นน่ะ*จำเป็นต้องใช้*ของจริง

ตอนนี้เรากำลังตกอยู่ในสถานการณ์ที่น่าสนใจทีเดียว เพราะเอาจริงๆ แล้วเราไม่ได้จำเป็นต้องใช้ PhantomData เลย... *ผมคิดว่างั้นนะ* ผมขอทึกทักเอาเองดื้อๆ แบบนี้แหละ แล้วเชื่อว่ามันเป็นจริงไปก่อน ถ้าสุดท้าย Miri มันแหกปากโวยวายขึ้นมา ผมค่อยยอมจำนนแล้วกลับมาใส่ PhantomData ให้มันทีหลัง

สิ่งที่เราจะทำจริงๆ ก็คือ ยอมใส่ reference กลับเข้าไปใน struct ของ Iterator เหล่านี้ แล้วมีความสุขกับการได้หยิบ reference มาใช้งานบ้างในบางจุด ผมคิดว่ามันก็สมเหตุสมผลดี เพราะตอนใช้งาน iterator มันมีการวางลำดับการยืม (nesting) ที่ชัดเจนและเรียบร้อยอยู่แล้ว: คุณสร้าง iterator ขึ้นมา, ใช้งาน reference ที่ปลอดภัยไปสักพัก, แล้วก็ drop ตัว iterator นั้นทิ้งไป

เมื่อตัว iterator หายไปแล้วเท่านั้น คุณถึงจะกลับมาแตะที่ตัวลิสต์เพื่อเรียกเมธอดอย่าง `push` และ `pop` ซึ่งต้องไปยุ่งเกี่ยวกับ tail pointer และ `Box` ได้ ในระหว่างการ iterate เราจะมีการ dereference raw pointer อยู่หลายจังหวะ จึงอาจจะมีการปนเปกันอยู่บ้าง แต่เราก็น่าจะพอมองได้ว่า reference เหล่านั้นเป็นเสมือน reborrow ของ raw pointer นั่นเอง

*ตัวผมเอง* ก็ยังไม่ได้ปักใจเชื่อ 100% หรอกนะ แต่ขอลองดูหน่อยเถอะ!

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

ถ้าเราจะเก็บ reference ไว้ เราก็ต้องอัปเกรด raw pointer ของเราให้กลายเป็น Option ของ reference เสียก่อน เรา*จะ*เขียนเช็กว่าพอยน์เตอร์เป็น null หรือไม่ก็ได้ แต่นี่เป็นหนึ่งในกรณีแคบๆ ที่หาได้ยากยิ่ง ซึ่ง*ผมคิดว่า*มันพอจะยอมรับได้ที่จะหยิบเอาเมธอดตัวแสบอย่าง [ptr::as_ref](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1) และ [ptr::as_mut](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut) มาใช้งาน

โดยปกติแล้วผมมักจะ*เตือน*ให้หลีกเลี่ยงเมธอดสองตัวนี้ราวกับโรคระบาด เพราะมันชอบทำอะไรพิลึกๆ ชวนเสียวสันหลัง แถมยังเป็นการดึงเอา reference กลับเข้ามาในระบบอีก ทั้งๆ ที่ "กฎง่ายๆ" ของผมคือพยายามหลีกเลี่ยงเรื่องพวกนี้แท้ๆ!

เมธอดพวกนั้นมีคำเตือนแปะไว้เพียบ แต่จุดที่น่าสนใจที่สุดคือตรงนี้ครับ:

> คุณจะต้องเป็นผู้บังคับใช้กฎ aliasing ของ Rust ด้วยตนเอง เนื่องจาก lifetime `'a` ที่ถูกส่งกลับออกมานั้นถูกเลือกขึ้นมาแบบลอยๆ และอาจไม่ได้สะท้อนถึง lifetime ที่แท้จริงของข้อมูล โดยเฉพาะอย่างยิ่ง ตลอดช่วงระยะเวลาของ lifetime นี้ หน่วยความจำที่พอยน์เตอร์ตัวนี้ชี้อยู่จะต้องไม่ถูกเข้าถึง (ไม่ว่าจะอ่านหรือเขียน) ผ่านทางพอยน์เตอร์ตัวอื่นเป็นอันขาด

เฮ้ย ดูสิครับ! นั่นมันเรื่องที่เราเพิ่งจะนั่งคุยกันมาตั้ง 25 หน้าชัดๆ! และในเมื่อผมได้ยืนยันไปแล้วว่าเราสามารถใช้ reference ตรงนี้ได้อย่าง*ปลอดภัยแน่นอน* ปัญหาเรื่อง aliasing ก็เป็นอันปิดจ๊อบ! ส่วนความชั่วร้ายอีกอย่างก็คือ signature ของมัน:

```rust ,ignore
pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T>
```

เห็นไหมว่า lifetime ตัวนั้นไม่ได้มีความเชื่อมโยงกับ input ใดๆ เลย เพราะ `self` ถูกส่งเข้ามาแบบ by-value? ใช่แล้วครับ นั่นคือสิ่งที่เราเรียกว่า "unbounded lifetime" และมันคือตัวอันตรายขั้นสุด เพราะมันพร้อมจะแกล้งทำตัวให้มีอายุยืนยาวเท่าไหร่ก็ได้ตามที่เราขอ แม้กระทั่ง `'static`! วิธีเดียวที่จะ*รับมือ*กับมันได้ คือต้องรีบยัดมันเข้าไปไว้ในจุดที่*ถูกจำกัดขอบเขต* ซึ่งโดยทั่วไปก็แปลว่า "รีบ return มันออกจากฟังก์ชันไปให้เร็วที่สุด เพื่อให้ signature ของฟังก์ชันเป็นตัวคุมขอบเขตของมันเอาไว้"

แหม ชวนให้ใจหายใจคว่ำดีแท้ แต่เราจะลุยต่อไม่ย่อท้อ! มาหยิบเอาโค้ด implementation ของ iterator จากสแต็กมาใช้กัน:

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

ถึงเวลาพิสูจน์ความจริงแล้ว...

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

เยสสสสสสสส!!! สะใจโว้ย **ผู้บรรยาย**! เห็นไหมล่ะว่าบางครั้งผมก็ไม่ได้ทำพลาดเฟ้ย!

> **ผู้บรรยาย:** แต่จุดประสงค์ทั้งหมดของหนังสือเล่มนี้ ก็คือการผิดพลาดเพื่อเป็นบทเรียนให้ผู้อ่านไม่ใช่เรอะ

เออใช่! แต่บางครั้งบทเรียนมันก็คือการที่ผมเป็นฝ่ายถูกไงเล่า! และทุกคนก็ควรจะตั้งใจฟังเวลาผมพูดเรื่อง unsafe code เพราะผมเสียเวลาในชีวิตไปกับการนั่งคิดเรื่องความถูกต้อง (soundness) ของ iterator implementation มากเกินไปแล้วเว้ย?! เคลียร์นะ?! เคลียร์!

เอาล่ะ มาต่อกันที่ `peek` และ `peek_mut`:

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

รอบนี้ผมจะไม่แม้แต่จะเขียนเทสต์มาทดสอบมันด้วยซ้ำ เพราะคนอย่างผมไม่มีวันทำผิดพลาดอีกต่อไปแล้ว

> **ผู้บรรยาย:** `cargo build`

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

เออ ก็ได้วะ!

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

ดูท่าทางผมคงจะ*ขยันทำพลาดต่อไปเรื่อยๆ* สินะ... งั้นเรามารัดกุมกันเป็นพิเศษ แล้วเพิ่มเทสต์ตัวใหม่ที่ผมจะขอเรียกว่า "อาหารสำหรับ Miri (miri food)" ดีกว่า: ซึ่งก็คือการจับเอา API ทั้งหมดของเรามายำมั่วซั่วสลับไปมา เพื่อช่วยให้ Miri ดักจับความผิดพลาดของเราได้ง่ายขึ้น

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

เพอร์เฟกต์!
