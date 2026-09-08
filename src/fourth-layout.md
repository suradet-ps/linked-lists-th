# โครงสร้างในหน่วยความจำ

หัวใจของการออกแบบของเราคือชนิด `RefCell` หัวใจของ
RefCell คือคู่ของเมธอด:

```rust ,ignore
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```

กฎของ `borrow` และ `borrow_mut` เหมือนกับ `&` และ `&mut` ทุกประการ:
คุณสามารถเรียก `borrow` กี่ครั้งก็ได้ตามต้องการ แต่ `borrow_mut`
ต้องการความเป็นเอกเทศ

แทนที่จะบังคับใช้แบบ.statically, RefCell บังคับใช้ที่รันไทม์
ถ้าคุณฝ่าฝืนกฎ RefCell จะแพนิกและทำให้โปรแกรมพัง
ทำไมมันถึงคืนค่า Ref และ RefMut เหล่านี้? พวกมันทำงานเหมือน
`Rc` แต่สำหรับการยืม พวกมันยังเก็บ RefCell ไว้ในสถานะถูกยืมจนกว่า
จะออกจากสโคป เราจะพูดถึงเรื่องนี้ทีหลัง

ตอนนี้ด้วย Rc และ RefCell เราจะกลายเป็น... ภาษาที่มีมิวเทบิลิตี้ทุกหนทุกแห่ง
แต่ไม่สามารถเก็บวัฏจักรได้! ย-ย-ย-ย-ย-ย...

เอาล่ะ เราต้องการเป็นแบบ*ลิงก์คู่* นั่นหมายความว่าแต่ละโหนดมีตัวชี้
ไปยังโหนดถัดไปและโหนดก่อนหน้า นอกจากนี้ตัวลิสต์เองมีตัวชี้ไปยัง
โหนดแรกและโหนดสุดท้าย ทำให้เราเพิ่มและลบได้เร็วที่*ทั้งสอง*ปลาย

ดังนั้นเราคงต้องการบางอย่างแบบนี้:

```rust ,ignore
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

```text
> cargo build

warning: field is never used: `head`
 --> src/fourth.rs:5:5
  |
5 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fourth.rs:6:5
  |
6 |     tail: Link<T>,
  |     ^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/fourth.rs:13:5
   |
13 |     next: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `prev`
  --> src/fourth.rs:14:5
   |
14 |     prev: Link<T>,
   |     ^^^^^^^^^^^^^
```

ฮิ บิลด์ได้แล้ว! มีคำเตือนโค้ดที่ไม่ใช้เยอะมาก แต่บิลด์ได้!
มาลองใช้กัน
