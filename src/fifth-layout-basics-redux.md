# โครงสร้างและพื้นฐาน 2: เข้าสู่โหมดดิบ (Getting Raw)

> สรุปย่อจาก 3 หัวข้อที่ผ่านมา: การนำ safe pointer อย่าง `&`, `&mut`, และ `Box` มาผสมปนเปกับ unsafe pointer อย่าง `*mut` และ `*const` ตามอำเภอใจ คือสูตรสำเร็จของหายนะสู่ Undefined Behaviour เพราะตัว safe pointer จะแอบตั้งข้อสมมติและข้อจำกัดเพิ่มเติมที่ raw pointer ไม่ได้ปฏิบัติตาม

คุณพระช่วย... ผมต้องกลับมาเขียน linked list อีกแล้วเหรอเนี่ย? ได้ ได้เลย! ไม่เป็นไร เราไม่เป็นไร ชิลๆ สบายมาก

เราจะรีบเคลียร์เนื้อหาในหัวข้อนี้ให้เสร็จอย่างรวดเร็ว เพราะเราได้คุยเรื่องดีไซน์กันไปหมดแล้วในรอบแรก และทุกสิ่งทุกอย่างที่เราเคยทำไว้มันก็*เกือบจะ*ถูกต้องทั้งหมดแล้ว ยกเว้นแค่เรื่องที่เราเอา safe กับ unsafe pointer มาผสมกันมั่วซั่วนั่นแหละ


# โครงสร้างหน่วยความจำ (Layout)

ดังนั้นใน layout ตัวใหม่นี้ เราจะใช้แต่ raw pointer ล้วนๆ แล้วทุกอย่างก็จะสมบูรณ์แบบ และเราจะไม่มีวันทำผิดพลาดอีกต่อไป

นี่คือ layout ตัวเก่าที่พังยับของเรา:

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // INNOCENT AND KIND
}

type Link<T> = Option<Box<Node<T>>>; // THE REAL EVIL

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

และนี่คือ layout ตัวใหม่แกะกล่องของเรา:

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>,
}

type Link<T> = *mut Node<T>; // MUCH BETTER

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

จำไว้นะครับ: `Option` ไม่ได้น่ารักหรือมีประโยชน์เท่าไหร่เวลาเราทำงานกับ raw pointer ดังนั้นเราจึงจะไม่ใช้มันอีกต่อไป ในหัวข้อถัดๆ ไปเราจะไปทำความรู้จักกับ type `NonNull` กัน แต่ตอนนี้ช่างมันไปก่อน



# พื้นฐาน (Basics)

`List::new` แทบจะเหมือนเดิมทุกประการ:

```rust ,ignore
use ptr;

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: ptr::null_mut(), tail: ptr::null_mut() }
    }
}
```

ส่วน `push` ก็แทบจะเห-


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(
```

เดี๋ยวนะ เราไม่ใช้ `Box` แล้วนี่นา แล้วเราจะจองหน่วยความจำ (allocate) ได้อย่างไรถ้าไม่มี `Box`?

คือเรา*จะ*ใช้ `std::alloc::alloc` ก็ได้แหละ แต่นั่นเหมือนเอามีดดาบซามูไรคาตานะมาหั่นผักในครัว มันทำงานได้ก็จริง แต่มันเว่อร์เกินเหตุและเกะกะเทอะทะมาก

สิ่งที่เราต้องการคือ *มี* Box แต่ก็*ไม่อยากมี* Box ทางเลือกหนึ่งที่หลุดโลกมากแต่*อาจจะ*พอถูไถไปได้ คือเขียนแบบนี้:

```
struct Node<T> {
    elem: T,
    real_next: Option<Box<Node<T>>>,
    next: *mut Node<T>,
}
```

ด้วยไอเดียที่ว่า เราสร้างกล่อง `Box` ขึ้นมาเก็บไว้ในโหนด แต่จากนั้นเราก็หยิบเอา raw pointer ชี้เข้าไปข้างใน และใช้งานเฉพาะ raw pointer ตัวนั้นไปเรื่อยๆ จนกว่าเราจะใช้งานโหนดนั้นเสร็จและต้องการทำลายมันทิ้ง จากนั้นเราค่อย `take` เอา `Box` ออกมาจาก `real_next` แล้วสั่ง drop มันทิ้ง ผม*คิดว่า*วิธีนี้น่าจะสอดคล้องกับโมเดล Stacked Borrows ฉบับย่อของเรานะ?

ถ้าคุณอยากลองทำแบบนั้นดูก็ขอให้ "สนุก" นะครับ แต่มันดูน่าเกลียดน่ากลัวพิลึกเลยใช่ไหมล่ะ? นี่ไม่ใช่บทของ `Rc` กับ `RefCell` อีกต่อไปแล้ว เราจะไม่ยอมเล่นเกมปาหี่พรรค์นี้อีก! เราจะทำอะไรที่มันเรียบง่ายและสะอาดตา

ดังนั้น เราจึงจะหยิบฟังก์ชันแสนวิเศษอย่าง [Box::into_raw][] มาใช้งานแทน:

> ```rust ,ignore
>   pub fn into_raw(b: Box<T>) -> *mut T
> ```
>
> ทำการบริโภค (consume) `Box` และคืนค่า raw pointer ที่ห่อหุ้มอยู่ออกมา
>
> ตัวพอยน์เตอร์จะได้รับการจัด alignment อย่างถูกต้องและไม่มีทางเป็น null
>
> หลังจากเรียกใช้ฟังก์ชันนี้ ผู้เรียกจะต้องเป็นผู้รับผิดชอบดูแลหน่วยความจำที่เคยบริหารจัดการโดย `Box` ด้วยตนเอง โดยเฉพาะอย่างยิ่ง ผู้เรียกจะต้องทำลาย `T` และปลดปล่อยหน่วยความจำ (deallocate) อย่างถูกต้องตามรูปแบบ memory layout ที่ `Box` ใช้งาน ซึ่งวิธีที่ง่ายที่สุดคือการแปลง raw pointer ตัวนี้กลับไปเป็น `Box` อีกครั้งด้วยฟังก์ชัน `Box::from_raw` เพื่อให้ destructor ของ `Box` ทำหน้าที่เก็บกวาดให้โดยอัตโนมัติ
>
> หมายเหตุ: ฟังก์ชันนี้เป็น associated function ซึ่งหมายความว่าคุณจะต้องเรียกใช้งานในรูปแบบ `Box::into_raw(b)` แทนที่จะเป็น `b.into_raw()` ทั้งนี้เพื่อไม่ให้ชื่อไปชนกับเมธอดของ inner type ที่อยู่ข้างใน
>
> **ตัวอย่าง**
>
> การแปลง raw pointer กลับไปเป็น `Box` ด้วย `Box::from_raw` เพื่อให้เก็บกวาดหน่วยความจำอัตโนมัติ:
>
> ```
>  let x = Box::new(String::from("Hello"));
>  let ptr = Box::into_raw(x);
>  let x = unsafe { Box::from_raw(ptr) };
> ```

เยี่ยมเลย ฟังก์ชันนี้ดูเหมือนสร้างมาเพื่อกรณีการใช้งานของเรา*อย่างแท้จริง* แถมยังตรงกับกฎเหล็กที่เราพยายามจะยึดถืออีกด้วย: เริ่มต้นด้วยของที่ปลอดภัย (safe), แปลงมันให้กลายเป็น raw pointer, แล้วค่อยแปลงกลับไปเป็น safe pointer ในตอนท้ายสุด (เมื่อเราต้องการจะ Drop มันทิ้ง)

วิธีนี้ให้ผลลัพธ์เหมือนกับการทำท่าพิสดารอย่าง `real_next` เป๊ะๆ แต่ไม่ต้องมาเสียเวลาเก็บกล่อง `Box` ให้เปลืองพื้นที่ ทั้งๆ ที่มันก็ชี้ไปที่ memory address เดียวกันกับ raw pointer อยู่แล้ว

และในเมื่อตอนนี้เราหันมาใช้ raw pointer กันทั่วทุกหนแห่งแล้ว เราก็ไม่ต้องมามัวพะวงเรื่องการจำกัดบล็อก `unsafe` ให้แคบๆ อีกต่อไป: ตอนนี้ทุกอย่างมัน unsafe หมดแล้วครับ! (เอาจริงๆ มันก็ไม่ปลอดภัยมาตั้งแต่แรกแล้วแหละ แต่บางครั้งการหลอกตัวเองมันก็ทำให้สบายใจดีนะ)


```rust ,ignore
pub fn push(&mut self, elem: T) {
    unsafe {
        // Immediately convert the Box into a raw pointer
        let new_tail = Box::into_raw(Box::new(Node {
            elem: elem,
            next: ptr::null_mut(),
        }));

        if !self.tail.is_null() {
            (*self.tail).next = new_tail;
        } else {
            self.head = new_tail;
        }

        self.tail = new_tail;
    }
}
```


เฮ้ย โค้ดดูสะอาดตาขึ้นเยอะเลยพอเราปักหลักใช้ raw pointer ล้วนๆ แบบนี้!

มาต่อกันที่ `pop` ซึ่งก็คล้ายกับเวอร์ชันที่เราเคยทำไว้มาก เพียงแต่เราต้องไม่ลืมใช้ `Box::from_raw` เพื่อคืนหน่วยความจำกลับสู่ระบบด้วย:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    unsafe {
        if self.head.is_null() {
            None
        } else {
            // RISE FROM THE GRAVE
            let head = Box::from_raw(self.head);
            self.head = head.next;

            if self.head.is_null() {
                self.tail = ptr::null_mut();
            }

            Some(head.elem)
        }
    }
}
```

`take` กับ `map` อันแสนน่ารักของเราลาโลกไปแล้ว ตอนนี้ต้องมานั่งเช็กและเซ็ตค่า `null` เองแบบ manual ลูกทุ่งๆ

และไหนๆ ก็มาถึงตรงนี้แล้ว ขอแถม destructor ลงไปด้วยเลย รอบนี้เราจะ implement มันด้วยการวนลูปเรียก `pop` ซ้ำๆ เพราะมันทั้งน่ารักและเรียบง่ายดี:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() { }
    }
}
```


คราวนี้ ถึงเวลาพิสูจน์ความจริง:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
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
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

```text
cargo test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

ผ่านฉลุย แต่ Miri จะว่ายังไงล่ะ?

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

เฮ้ยยยยยยยยยยยยยย!!!!!

มันทำงานได้แล้วโว้ยยยยยยย!

...น่าจะนะ!

การที่ตรวจไม่พบ Undefined Behaviour ไม่ได้แปลว่ามันไม่ได้แอบซุ่มรอสร้างปัญหาอยู่ แต่ความอดทนในการพิสูจน์ความถูกต้องตามหลักวิชาการสำหรับหนังสือมุกตลกเกี่ยวกับ linked list เล่มนี้มันมีขีดจำกัดโว้ย! เพราะฉะนั้น เราจะถือว่านี่คือข้อพิสูจน์ที่ผ่านการตรวจสอบโดยเครื่องจักร 100% เรียบร้อยแล้ว และใครก็ตามที่กล้าเถียงเป็นอย่างอื่น ก็ไป suck my Coq (เครื่องพิสูจน์ทฤษฎีบท Coq) ซะเถอะ!

∴ QED □


[Box::into_raw]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw
