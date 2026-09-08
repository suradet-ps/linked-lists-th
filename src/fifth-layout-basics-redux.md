# เลย์เอาต์และพื้นฐาน 2: การใช้ตัวชี้ดิบ

> สรุปสั้นจากสามส่วนที่ผ่านมา: การผสมตัวชี้ปลอดภัย เช่น `&`, `&mut`, และ `Box` กับตัวชี้ดิบ (raw pointer) เช่น `*mut` และ `*const` อย่างสุ่มสี่สุ่มห้านั้นเป็นสูตรสำหรับพฤติกรรมที่ไม่กำหนด (Undefined Behaviour) เพราะตัวชี้ปลอดภัยจะเพิ่มข้อจำกัดเพิ่มเติมที่เราไม่ได้ปฏิบัติตามเมื่อใช้ตัวชี้ดิบ

โอ้พระเจ้า ฉันต้องเขียนลิงก์ลิสต์อีกแล้ว ได้ ได้เลย ไม่เป็นไร เราโอเค เราโอเค

เราจะทำส่วนนี้ให้เสร็จอย่างรวดเร็วเพราะเราได้พูดถึงดีไซน์ไปแล้วในการลองรอบแรก และทุกอย่างที่เราทำ *เกือบ* จะถูกต้องแล้วนอกจากวิธีที่เราผสมตัวชี้ปลอดภัยและตัวชี้ไม่ปลอดภัยเข้าด้วยกัน


# เลย์เอาต์

ในเลย์เอาต์ใหม่นี้ เราจะใช้ตัวชี้ดิบ (raw pointer) เท่านั้น และทุกอย่างจะสมบูรณ์แบบ และเราจะไม่ทำผิดพลาดอีกต่อไป

นี่คือเลย์เอาต์เก่าที่เสียหายของเรา:

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

และนี่คือเลย์เอาต์ใหม่ของเรา:

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

จำไว้ว่า Option ไม่ได้ดีหรือมีประโยชน์เท่าไหร่เมื่อเราใช้ตัวชี้ดิบ ดังนั้นเราจึงไม่ใช้มันอีกต่อไป ในส่วนถัดไปเราจะพิจารณาชนิด `NonNull` แต่ไม่ต้องกังวลเรื่องนั้นตอนนี้



# พื้นฐาน

List::new เกือบจะเหมือนเดิม

```rust ,ignore
use ptr;

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: ptr::null_mut(), tail: ptr::null_mut() }
    }
}
```

push ก็เกือบจะเ-


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(
```

เดี๋ยวก่อน เราไม่ใช้ Box อีกแล้ว เราจะจัดสรรหน่วยความจำได้อย่างไรโดยไม่ใช้ Box?

ก็ เรา *สามารถ* ใช้ `std::alloc::alloc` ได้ แต่นั่นเหมือนการนำคาตานะเข้ามาในห้องครัว มันจะทำงานได้สำเร็จแต่เกินความจำเป็นและจัดการยาก

เราต้องการ *มี* Box แต่ *ไม่ต้องการ* ทางเลือกที่บ้าคลั่งแต่ *อาจ* เป็นไปได้ทางหนึ่งคือทำแบบนี้:

```
struct Node<T> {
    elem: T,
    real_next: Option<Box<Node<T>>>,
    next: *mut Node<T>,
}
```

โดยแนวคิดคือเราสร้าง Box เก็บไว้ในโหนด แต่แล้วเราเอาตัวชี้ดิบ (raw pointer) ไปชี้ และใช้ตัวชี้นั้นจนกว่าเราจะทำเสร็จกับโหนดและต้องการทำลายมัน จากนั้นเราสามารถ `take` Box ออกจาก `real_next` แล้ว drop มัน *ฉันคิด* ว่ามันจะสอดคล้องกับโมเดลสแต็กด์บอร์โรวส์ที่ง่ายมากของเรา?

ถ้าคุณอยากลองทำแบบนั้น สนุกกับมัน แต่มันดูแย่มากเลยใช่ไหม? นี่ไม่ใช่บทเกี่ยวกับ Rc และ RefCell เราจะไม่เล่น *เกม* นี้อีกต่อไป เราจะสร้างของที่เรียบง่ายและสะอาด

ดังนั้นเราจะใช้ฟังก์ชัน [Box::into_raw][] ที่ดีมาก:

> ```rust ,ignore
>   pub fn into_raw(b: Box<T>) -> *mut T
> ```
>
> บริโภค Box และคืนตัวชี้ดิบที่ห่อหุ้ม
>
> ตัวชี้จะจัดตำแหน่งอย่างถูกต้องและไม่เป็น null
>
> หลังจากเรียกฟังก์ชันนี้ ผู้เรียกจะรับผิดชอบต่อหน่วยความจำที่จัดการโดย Box ก่อนหน้านี้ โดยเฉพาะ ผู้เรียกควรทำลาย T อย่างถูกต้องและปลดปล่อยหน่วยความจำ (deallocate) โดยคำนึงถึงเลย์เอาต์หน่วยความจำที่ Box ใช้ วิธีที่ง่ายที่สุดคือแปลงตัวชี้ดิบ (raw pointer) กลับเป็น Box ด้วยฟังก์ชัน `Box::from_raw` ทำให้ดีสตรัคเตอร์ของ Box ทำงานทำความสะอาด
>
> หมายเหตุ: นี่คือฟังก์ชันในคลาส ซึ่งหมายความว่าคุณต้องเรียกมันเป็น `Box::into_raw(b)` แทนที่จะเป็น `b.into_raw()` นี่เป็นเพื่อไม่ให้มีความขัดแย้งกับเมธอดบนชนิดภายใน
>
> **ตัวอย่าง**
>
> แปลงตัวชี้ดิบ (raw pointer) กลับเป็น Box ด้วย Box::from_raw เพื่อทำความสะอาดอัตโนมัติ:
>
> ```
>  let x = Box::new(String::from("Hello"));
>  let ptr = Box::into_raw(x);
>  let x = unsafe { Box::from_raw(ptr) };
> ```

เยี่ยม ดูเหมือน *แทบจะ* ออกแบบมาเพื่อกรณีการใช้งานของเรา มันยังตรงกับกฎที่เราพยายามปฏิบัติตาม: เริ่มจากของปลอดภัย แปลงเป็นตัวชี้ดิบ (raw pointer) และจากนั้นค่อยแปลงกลับเป็นของปลอดภัยตอนท้าย (เมื่อเราต้องการ Drop มัน)

นี่เหมือนกับการทำ `real_next` ประหลาดๆ เกือบจะเป๊ะเลย แต่ไม่ต้องเสียเวลาเก็บ Box เมื่อมันเป็นตัวชี้เดียวกันกับตัวชี้ดิบอยู่แล้ว

นอกจากนี้ตอนนี้เราใช้ตัวชี้ดิบ (raw pointer) ทุกที่แล้ว เรามาไม่ต้องกังวลว่าจะเก็บบล็อก `unsafe` ให้แคบ: ตอนนี้มัน unsafe หมดแล้ว (มันเป็นมาตลอด แต่บางครั้งก็ดีที่หลอกตัวเอง)


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


เฮ้ โค้ดนั้นดูสะอาดขึ้นมากตอนที่เราใช้ตัวชี้ดิบ (raw pointer) ตลอด!

ต่อไปคือ pop ซึ่งก็คล้ายกับที่เราทิ้งไว้มาก แต่เราต้องจำว่าต้องใช้ `Box::from_raw` เพื่อทำความสะอาดการจัดสรรหน่วยความจำ:

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

`take` และ `map` ที่ดีของเราตายแล้ว ต้องตรวจสอบและตั้ง `null` ด้วยตนเองตอนนี้

และในขณะที่เราอยู่ที่นี่ เรามาใส่ดีสตรัคเตอร์ (destructor) เลย คราวนี้เราจะ implement มันเป็นแค่การ pop ซ้ำๆ เพราะมันน่ารักและเรียบง่าย:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() { }
    }
}
```


ตอนนี้ มาถึงเวลาแห่งความจริง:

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

ดี แต่ miri เห็นด้วยไหม?

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

เฮ้ยยยย!!!!

มันทำงานได้แล้ว!!!!

น่าจะ!!!!

การไม่พบพฤติกรรมที่ไม่กำหนด (Undefined Behaviour) ไม่ได้พิสูจน์ว่ามันไม่ได้อยู่รอทำให้เกิดปัญหา แต่มีขีดจำกัดว่าฉันจะจริงจังแค่ไหนสำหรับหนังสือเรื่องตลกเกี่ยวกับลิงก์ลิสต์ ดังนั้นเราจะนับว่ามันเป็นหลักฐานที่ยืนยันด้วยเครื่องจักร 100% และใครก็ตามที่บอกว่าไม่ใช่ก็สามารถ kiss my coq ได้เลย!

∴ QED □


[Box::into_raw]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw
