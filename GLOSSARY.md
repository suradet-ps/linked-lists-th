# พจนานุกรมศัพท์ (Glossary) — too-many-th ฉบับภาษาไทย

ตารางนี้เป็นคำศัพท์ที่ใช้อย่างสม่ำเสมอตลอดทั้งเล่ม เพื่อให้การแปลทุกบทใช้คำเดียวกัน
(มาจากหนังสือต้นฉบับ Learning Rust With Entirely Too Many Linked Lists)

| ศัพท์ต้นฉบับ | คำแปลไทย | หมายเหตุ |
|---|---|---|
| linked list | ลิงก์ลิสต์ | กล่าวถึงครั้งแรกด้วย "ลิงก์ลิสต์ (linked list)" |
| stack | สแต็ก | โครงสร้างข้อมูลแบบกองซ้อน |
| queue | คิว | |
| deque | เดก | double-ended queue; เรียกครั้งแรกด้วย "เดก (deque)" |
| node | โหนด | องค์ประกอบหนึ่งของลิงก์ลิสต์ |
| head / tail | หัว (head) / หาง (tail) | ปลายทั้งสองของลิสต์ |
| heap | ฮีป | หน่วยความจำส่วนที่จัดสรรแบบไดนามิก |
| stack (memory) | สแต็ก | หน่วยความจำส่วนที่ทำงานแบบ LIFO; แยกจาก "สแต็ก" ในฐานะโครงสร้างข้อมูลด้วยบริบท |
| Box / Rc / Arc | Box / Rc / Arc (คงชื่อเดิม) | สมาร์ตพอยน์เตอร์ของ Rust |
| ownership | ความเป็นเจ้าของ | |
| borrow / borrowing | ยืม / การยืม | |
| reference | เรเฟอเรนซ์ | กล่าวถึงครั้งแรกด้วย "การอ้างอิง (reference)"; `&mut` = เรเฟอเรนซ์ที่เปลี่ยนแปลงได้, `&` = เรเฟอเรนซ์ร่วม |
| pointer | ตัวชี้ (pointer) | |
| raw pointer | ตัวชี้ดิบ (raw pointer) | `*const` / `*mut` |
| smart pointer | สมาร์ตพอยน์เตอร์ | |
| allocation / allocate | การจัดสรร / จัดสรร | การจัดสรรหน่วยความจำ (memory allocation) |
| deallocation / deallocate | การปลดปล่อย / ปลดปล่อย | คืนหน่วยความจำ |
| enum | enum (คงชื่อเดิม) | |
| variant | วาเรียนต์ | รูปแบบหนึ่งของ enum |
| struct | struct (คงชื่อเดิม) | |
| field | ฟิลด์ | |
| trait | เทรต | กล่าวถึงครั้งแรกด้วย "เทรต (trait)" |
| generic | เจเนอริก | |
| method | เมธอด | |
| function | ฟังก์ชัน | |
| impl | impl (คงชื่อเดิม) | บล็อก `impl` |
| pattern matching | การจับคู่แพตเทิร์น | กล่าวถึงครั้งแรกด้วย "การจับคู่แพตเทิร์น (pattern matching)" |
| match | match (คงชื่อเดิม) | คีย์เวิร์ดของ Rust |
| Option / Some / None | Option / Some / None (คงชื่อเดิม) | |
| iterator | อิเทอเรเตอร์ | |
| Iter / IterMut / IntoIter | Iter / IterMut / IntoIter (คงชื่อเดิม) | ชื่อเมธอด/โครงสร้าง |
| lifetime | ไลฟ์ไทม์ | |
| variance | วาเรียนซ์ | |
| covariance | โควาเรียนซ์ | |
| contravariance | คอนทราเวเรียนซ์ | |
| invariance | อินวาเรียนซ์ | |
| subtyping | การเป็นซับไทป์ | กล่าวถึงครั้งแรกด้วย "การเป็นซับไทป์ (subtyping)" |
| Send / Sync | Send / Sync (คงชื่อเดิม) | เทรตของ Rust |
| aliasing | การแอลิเอส | การมีตัวชี้หลายตัวชี้ถึงข้อมูลเดียวกัน |
| stacked borrows | สแต็กด์บอร์โรวส์ | โมเดลการยืมแบบซ้อนของ Miri; คงชื่อเดิม |
| UnsafeCell | UnsafeCell (คงชื่อเดิม) | |
| Miri | Miri (คงชื่อเดิม) | ตัวตรวจสอบโค้ด unsafe |
| unsafe | unsafe (คงชื่อเดิม) | บล็อก/คีย์เวิร์ด |
| panic | แพนิก | กล่าวถึงครั้งแรกด้วย "การแพนิก (panic)" |
| panic safety | ความปลอดภัยต่อการแพนิก | |
| drop / destructor | drop / ดีสตรัคเตอร์ | คำกริยา "drop" คงศัพท์อังกฤษ; กล่าวถึงครั้งแรกด้วย "ดีสตรัคเตอร์ (destructor)" |
| recursion | การเรียกซ้ำ | |
| tail recursive | การเรียกซ้ำแบบท้าย | |
| expression | นิพจน์ | |
| scope | สโคป | ขอบเขตการใช้งานตัวแปร |
| borrow checker | บอร์โรว์เช็กเกอร์ | |
| compiler | คอมไพเลอร์ | |
| compile | คอมไพล์ | |
| runtime | รันไทม์ | |
| build | บิลด์ | |
| crate | ครีต | |
| module | โมดูล | |
| test | เทสต์ / การทดสอบ | `#[test]` = การทดสอบ |
| macro | มาโคร | |
| attribute | แอตทริบิวต์ | `#[...]` |
| slice | สไลซ์ | |
| Vec / VecDeque | Vec / VecDeque (คงชื่อเดิม) | คอลเลกชันมาตรฐานของ Rust |
| null pointer optimization | การเพิ่มประสิทธิภาพด้วยพอยน์เตอร์ null | NPO; กล่าวถึงครั้งแรกแบบเต็ม |
| interior mutability | มิวเทบิลิตี้ภายใน | |
| inherited mutability | มิวเทบิลิตี้ที่สืบทอด | |
| Copy | Copy (คงชื่อเดิม) | เทรต; แยกจากคำว่า "คัดลอก" ทั่วไป |
| Clone | Clone (คงชื่อเดิม) | เทรต |
| dereference / deref | ตามตัวชี้ | คำสั่ง/เมธอด `deref` คงชื่อเดิม |
| Smart pointer | สมาร์ตพอยน์เตอร์ | |
| cursor | เคอร์เซอร์ | ตำแหน่งอ้างอิงภายในคอลเลกชัน |
| combinator | คอมบิเนเตอร์ | |
| unit type | ชนิดยูนิต | `()` |
| ZST | ชนิดขนาดศูนย์ (ZST) | zero-sized type |
| mem::replace / mem::swap | mem::replace / mem::swap (คงชื่อเดิม) | |
| assert_eq! | assert_eq! (คงชื่อเดิม) | มาโคร |
| cargo / rustup | cargo / rustup (คงชื่อเดิม) | |
| trait bound | ทรตบาวด์ | |
| PR | PR (คงชื่อเดิม) | pull request; ใช้ "พีอาร์" ได้ในบริบทเล่าเรื่อง |
| commit | คอมมิต | คำกริยา "merge" คงศัพท์อังกฤษตามบริบท git |

## หลักการทั่วไป

- ชื่อเครื่องมือ คำสั่ง CLI ตัวเลือก (flag) ชื่อแพ็กเกจ URL คีย์เวิร์ด และชื่อชนิด/เมธอดของ Rust **ไม่แปล** เช่น `Box`, `Option`, `push`, `mem::replace`, `cargo`
- โค้ดทุกบล็อก (``` ... ```) เก็บไว้ตามต้นฉบับทุกตัวอักษร รวมถึงคอมเมนต์ภายในโค้ด
- ลิงก์ (ทั้ง inline และ reference-style) คง path เดิม เพื่อให้ mdbook ยัง build ได้
- ชื่อไฟล์ต้นทางคงตามต้นฉบับ (`first.md`, `sixth-layout.md`, ...) เพื่อให้โครงสร้างตรงกับ repo ต้นทางแบบไฟล์ต่อไฟล์
- ศัพท์ที่ยังไม่มีคำแปลตายตัว ใช้ศัพท์อังกฤษเป็นหลักแล้วตามด้วยคำแปลในวงเล็บในครั้งแรก แล้วค่อยใช้คำแปลต่อจากนั้น
- เอกสารอ้างอิงภายนอก (เช่น คู่มือ Nomicon, คู่มือ Rust) คงชื่อเดิมและคงลิงก์เดิม