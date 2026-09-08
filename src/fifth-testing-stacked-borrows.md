# การทดสอบสแต็กด์บอร์โรวส์

> TL;DR ของโมเดลหน่วยความจำ (ที่ง่าย) ของ Rust ในบทก่อนหน้า:
>
> * Rust จัดการ reborrow ด้วยการรักษา "สแต็กยืม (borrow stack)"
> * มีเพียงตัวที่อยู่ด้านบนสุดของสแต็กเท่านั้นที่ "มีชีวิต" (มีสิทธิ์เข้าถึงแบบเอกสิทธิ์)
> * เมื่อคุณเข้าถึงตัวที่อยู่ต่ำกว่า มันจะกลายเป็น "มีชีวิต" และตัวที่อยู่เหนือมันจะถูกนำออกจากสแต็ก
> * คุณไม่ได้รับอนุญาตให้ใช้ตัวชี้ดิบ (raw pointer) ที่ถูกนำออกจากสแต็กยืมแล้ว
> * บอร์โรว์เช็กเกอร์ทำให้แน่ใจว่าโค้ดที่ปลอดภัยจะปฏิบัติตามกฎนี้
> * Miri ตรวจสอบว่าตัวชี้ดิบปฏิบัติตามนี้ที่รันไทม์

นี่คือทฤษฎีและแนวคิดมากมาย — เรามาต่อกับหัวใจและจิตวิญญาณที่แท้จริงของหนังสือเล่มนี้: เขียนโค้ดที่ไม่ดีและให้เครื่องมือของเราตะโกนใส่เรา เราจะผ่าน *จำนวนมาก* ของตัวอย่างเพื่อดูว่าโมเดลจิตใจของเราสมเหตุสมผลหรือไม่ และเพื่อให้ได้สัมผัสเชิงสัญชาตญาณเกี่ยวกับสแต็กด์บอร์โรวส์

> **ผู้บรรยาย:** การจับพฤติกรรมที่ไม่กำหนด (Undefined Behaviour) ในทางปฏิบัติเป็นเรื่องยุ่งยาก ในที่สุดคุณกำลังจัดการกับสถานการณ์ที่คอมไพเลอร์ *สมมติ* ว่าไม่เกิดขึ้น
>
> ถ้าคุณโชคดี วันนี้ทุกอย่างจะ "ดูเหมือนทำงานได้" แต่มันจะเป็นระเบิดเวลาสำหรับคอมไพเลอร์ที่ฉลาดขึ้นหรือการเปลี่ยนแปลงเล็กน้อยในโค้ด ถ้าคุณ *โชคดีมาก* ทุกอย่างจะขัดข้องอย่างน่าเชื่อถือเพื่อให้คุณจับข้อผิดพลาดและแก้ไขได้ แต่ถ้าคุณไม่โชคดี ทุกอย่างจะเสียหายในลักษณะที่แปลกและทำให้สับสน
>
> Miri พยายามหลีกเลี่ยงปัญหานี้โดยรับมุมมองที่ไม่ได้ปรับปรุงและไม่ได้เพิ่มประสิทธิภาพที่สุดของ rustc และติดตามสถานะเพิ่มเติมในขณะที่ตีความ ในฐานะ "ตัวตรวจสอบ" นี่เป็นวิธีที่ค่อนข้างสมเหตุสมผลและทนทาน แต่มันจะไม่ *สมบูรณ์แบบ* โปรแกรมทดสอบของคุณจะต้องมีการดำเนินการกับ UB นั้นจริง และสำหรับโปรแกรมที่ใหญ่พอจะง่ายมากที่จะนำความไม่แน่นอนทุกชนิดมาสู่ (HashMap ใช้ RNG โดยค่าเริ่มต้น!)
>
> เราไม่สามารถยอมรับ miri ว่าเห็นด้วยกับการดำเนินการของโปรแกรมของเราเป็นข้อความที่แน่นอนว่าไม่มี UB ได้ Miri ยังสามารถ *คิด* ว่าบางสิ่งเป็น UB ทั้งที่จริงๆ ไม่ใช่ แต่ถ้าเรามีโมเดลจิตใจเกี่ยวกับวิธีการทำงาน และ miri ดูเหมือนจะเห็นด้วยกับเรา นั่นเป็นสัญญาณที่ดีว่าเราอยู่ในเส้นทางที่ถูกต้อง




# การยืมพื้นฐาน

ในบทก่อนหน้าเราเห็นว่าบอร์โรว์เช็กเกอร์ไม่ชอบโค้ดนี้:

```rust ,ignore
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

// ORDER SWAPPED!
*ref1 += 1;
*ref2 += 2;

println!("{}", data);
```

เรามาดูกันว่าเกิดอะไรขึ้นเมื่อเราแทนที่ `ref2` ด้วย `*mut`:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;

    // ORDER SWAPPED!
    *ref1 += 1;
    *ptr2 += 2;

    println!("{}", data);
}
```

```text
cargo run
   Compiling miri-sandbox v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 0.71s
     Running `target\debug\miri-sandbox.exe`
13
```

Rustc ดูเหมือนจะพอใจกับสิ่งนี้: ไม่มีคำเตือนและโปรแกรมได้ผลลัพธ์ที่เราคาดหวัง! ตอนนี้เรามาดูกันว่า miri (ในโหมดเข้มงวด) คิดอย่างไร:

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running cargo-miri.exe target\miri

error: Undefined Behavior: no item granting read access 
to tag <untagged> at alloc748 found in borrow stack.

 --> src\main.rs:9:9
  |
9 |         *ptr2 += 2;
  |         ^^^^^^^^^^ no item granting read access to tag <untagged> 
  |                    at alloc748 found in borrow stack.
  |
  = help: this indicates a potential bug in the program: 
    it performed an invalid operation, but the rules it 
    violated are still experimental
 
```

ยอดเยี่ยม! โมเดลสัญชาตญาณของเราเกี่ยวกับวิธีการทำงานยังคงใช้ได้: แม้ว่าคอมไพเลอร์จะจับปัญหาไม่ได้ แต่ miri จับได้

เรามาลองอะไรที่ซับซ้อนขึ้น กรณี `&mut -> *mut -> *mut -> *mut` ที่เราพูดถึงก่อนหน้า:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;
    let ref3 = &mut *ptr2;
    let ptr4 = ref3 as *mut _;

    // Access the first raw pointer first
    *ptr2 += 2;

    // Then access things in "borrow stack" order
    *ptr4 += 4;
    *ref3 += 3;
    *ptr2 += 2;
    *ref1 += 1;

    println!("{}", data);
}
```

```text
cargo run
22

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
to tag <1621> at alloc748 found in borrow stack.

  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting read access to tag <1621> 
   |                at alloc748 found in borrow stack.
   |
```

ว้าว ใช่เลย! ในโหมดเข้มงวด miri สามารถ "แยก" ตัวชี้ดิบทั้งสองออกได้ และการใช้ตัวที่สองทำให้ตัวแรกไม่ถูกต้อง เรามาดูกันว่าทุกอย่างทำงานได้ไหมเมื่อเราลบการใช้งานแรกที่ทำให้ทุกอย่างยุ่งเหยิง:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;
    let ref3 = &mut *ptr2;
    let ptr4 = ref3 as *mut _;

    // Access things in "borrow stack" order
    *ptr4 += 4;
    *ref3 += 3;
    *ptr2 += 2;
    *ref1 += 1;

    println!("{}", data);
}
```

```text
cargo run
20

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
20
```

ดีมาก

ใช่ ฉันค่อนข้างแน่ใจว่าตอนนี้เราทุกคนจะได้ปริญญาเอกด้านการออกแบบและการดำเนินการของโมเดลหน่วยความจำในภาษาโปรแกรมมิ่ง ใคร *ต้องการ* คอมไพเลอร์ ของพวกนี้ *ง่าย*

> **ผู้บรรยาย:** มันไม่ง่าย แต่ฉันภูมิใจในตัวคุณอยู่ดี




# การทดสอบอาร์เรย์

เรามาเล่นกับอาร์เรย์และออฟเซ็ตตัวชี้ (`add` และ `sub`) นี่ควรทำงานได้ ใช่ไหม?

```rust ,ignore
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr3_at_1 = ptr2_at_0.add(1);       // Ptr to 1st element

    *ptr3_at_1 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[3, 3, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
to tag <1619> at alloc748+0x4 found in borrow stack.
 --> src\main.rs:8:5
  |
8 |     *ptr3_at_1 += 3;
  |     ^^^^^^^^^^^^^^^ no item granting read access to tag <1619>
  |                     at alloc748+0x4 found in borrow stack.
```

*ฉีกใบสมัครเข้ามหาวิทยาลัย*

เกิดอะไรขึ้น? เราใช้สแต็กยืมได้ดีมาก! มีบางสิ่งที่แปลกเกิดขึ้นเมื่อเราไป `ptr -> ptr` หรือ? ถ้าเราแค่คัดลอกตัวชี้เพื่อให้ทั้งหมดไปยังตำแหน่งเดียวกันล่ะ:

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr3_at_0 = ptr2_at_0;              // Ptr to 0th element

    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [6, 0, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

ไม่ นั่นทำงานได้ดี บางทีเราอาจโชคดี เรามาทำให้ตัวชี้ยุ่งเหยิงจริงๆ:

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];            // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;   // Ptr to 0th element
    let ptr3_at_0 = ptr2_at_0;               // Ptr to 0th element
    let ptr4_at_0 = ptr2_at_0.add(0);        // Ptr to 0th element
    let ptr5_at_0 = ptr3_at_0.add(1).sub(1); // Ptr to 0th element

    // An absolute jumbled hash of ptr usages
    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ptr4_at_0 += 4;
    *ptr5_at_0 += 5;
    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [20, 0, 0, ...]
    println!("{:?}", &data[..]);
}
```


```text
cargo run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

ไม่ Miri ยืดหยุ่นมากขึ้นเมื่อพูดถึงตัวชี้ดิบที่ได้มาจากตัวชี้ดิบอื่น ทั้งหมดใช้ "การยืม" เดียวกัน (หรือที่ miri เรียกว่า *แท็ก*)

เมื่อคุณเริ่มใช้ตัวชี้ดิบ พวกเขาสามารถแยกออกเป็นผู้ชายตัวเล็กที่โกรธของตัวเองและเล่นกับตัวเองได้อย่างอิสระ นี่ไม่เป็นไรเพราะคอมไพเลอร์เข้าใจสิ่งนี้และจะไม่เพิ่มประสิทธิภาพการอ่านและเขียนเหมือนที่ทำกับเรเฟอเรนซ์

> **ผู้บรรยาย:** ถ้าโค้ดง่ายพอ คอมไพเลอร์สามารถติดตามตัวชี้ที่ได้ทั้งหมดและยังเพิ่มประสิทธิภาพได้ แต่มันจะเปราะบางกว่าเหตุผลที่ใช้กับเรเฟอเรนซ์มาก

แล้ว *ปัญหาจริง* คืออะไร?

แม้ว่า `data` จะเป็น "การจัดสรร" เดียว (ตัวแปรภายใน) `ref1_at_0` กำลังยืมเฉพาะองค์ประกอบแรกเท่านั้น Rust อนุญาตให้การยืมถูกแบ่งเพื่อให้ใช้เฉพาะบางส่วนของการจัดสรรเท่านั้น! เรามาลองดู:

```rust ,ignore
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ref2_at_1 = &mut data[1];           // Reference to 1th element
    let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr4_at_1 = ref2_at_1 as *mut i32;   // Ptr to 1th element

    *ptr4_at_1 += 4;
    *ptr3_at_0 += 3;
    *ref2_at_1 += 2;
    *ref1_at_0 += 1;

    // Should be [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
error[E0499]: cannot borrow `data[_]` as mutable more than once at a time
 --> src\main.rs:5:21
  |
4 |     let ref1_at_0 = &mut data[0];           // Reference to 0th element
  |                     ------------ first mutable borrow occurs here
5 |     let ref2_at_1 = &mut data[1];           // Reference to 1th element
  |                     ^^^^^^^^^^^^ second mutable borrow occurs here
6 |     let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
  |                     --------- first borrow later used here
  |
  = help: consider using `.split_at_mut(position)` or similar method 
    to obtain two mutable non-overlapping sub-slices
```

แย่! Rust ไม่ได้ติดตามดัชนีอาร์เรย์เพื่อพิสูจน์ว่าการยืมเหล่านี้แยกจากกัน แต่มันให้ `split_at_mut` แก่เราเพื่อแบ่งสไลซ์เป็นหลายส่วนในลักษณะที่ปลอดภัย:

```rust
unsafe {
    let mut data = [0; 10];

    let slice1 = &mut data[..];
    let (slice2_at_0, slice3_at_1) = slice1.split_at_mut(1); 
    
    let ref4_at_0 = &mut slice2_at_0[0];    // Reference to 0th element
    let ref5_at_1 = &mut slice3_at_1[0];    // Reference to 1th element
    let ptr6_at_0 = ref4_at_0 as *mut i32;  // Ptr to 0th element
    let ptr7_at_1 = ref5_at_1 as *mut i32;  // Ptr to 1th element

    *ptr7_at_1 += 7;
    *ptr6_at_0 += 6;
    *ref5_at_1 += 5;
    *ref4_at_0 += 4;

    // Should be [10, 12, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]
```

เฮ้ มันทำงานได้! สไลซ์บอกคอมไพเลอร์และ miri อย่างถูกต้องว่า "เฮ้ ฉันกำลังยืมหน่วยความจำทั้งหมดในช่วงของฉัน" ดังนั้นพวกเขาจึงรู้ว่าองค์ประกอบทั้งหมดสามารถเปลี่ยนแปลงได้

นอกจากนี้ โปรดสังเกตว่าการดำเนินการเช่น `split_at_mut` ที่ได้รับอนุญาตบอกเราว่าการยืมอาจไม่ใช่ *สแต็ก* มากเท่ากับ *ต้นไม้* เพราะเราสามารถแบ่งการยืมใหญ่หนึ่งรายการเป็นหลายส่วนที่แยกจากกัน และทุกอย่างยังทำงานได้

(ฉันคิดว่าในโมเดลสแต็กด์บอร์โรวส์จริง ทุกอย่างยังคงเป็นสแต็กเพราะสแต็กกำลังติดตามสิทธิ์สำหรับแต่ละไบต์ของโปรแกรม)

ถ้าเรา *เปลี่ยน* สไลซ์เป็นตัวชี้โดยตรงล่ะ? ตัวชี้นั้นจะมีสิทธิ์เข้าถึงสไลซ์ทั้งหมดหรือไม่?

```rust
unsafe {
    let mut data = [0; 10];

    let slice1_all = &mut data[..];         // Slice for the entire array
    let ptr2_all = slice1_all.as_mut_ptr(); // Pointer for the entire array
    
    let ptr3_at_0 = ptr2_all;               // Pointer to 0th elem (the same)
    let ptr4_at_1 = ptr2_all.add(1);        // Pointer to 1th elem
    let ref5_at_0 = &mut *ptr3_at_0;        // Reference to 0th elem
    let ref6_at_1 = &mut *ptr4_at_1;        // Reference to 1th elem

    *ref6_at_1 += 6;
    *ref5_at_0 += 5;
    *ptr4_at_1 += 4;
    *ptr3_at_0 += 3;

    // Just for fun, modify all the elements in a loop
    // (Could use any of the raw pointers for this, they share a borrow!)
    for idx in 0..10 {
        *ptr2_all.add(idx) += idx;
    }

    // Safe version of this same code for fun
    for (idx, elem_ref) in slice1_all.iter_mut().enumerate() {
        *elem_ref += idx; 
    }

    // Should be [8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
    println!("{:?}", &data[..]);
}
```


```text
cargo run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
```


ดี! ตัวชี้ไม่ใช่แค่จำนวนเต็ม: มันมีช่วงหน่วยความจำที่เกี่ยวข้อง และด้วย Rust เราได้รับอนุญาตให้แคบลง!




# การทดสอบเรเฟอเรนซ์ร่วม

ในตัวอย่างทั้งหมดนี้ ฉันระมัดระวังมากที่จะใช้เฉพาะเรเฟอเรนซ์ที่เปลี่ยนแปลงได้และทำ Operations อ่าน-แก้ไข-เขียน (`+=`) เพื่อให้ทุกอย่างง่ายที่สุด

แต่ Rust มีเรเฟอเรนซ์ร่วมที่อ่านอย่างเดียวและสามารถคัดลอกได้อย่างอิสระ สิ่งเหล่านี้ควรทำงานอย่างไร? เราได้เห็นว่าตัวชี้ดิบสามารถคัดลอกได้อย่างอิสระและเราจัดการได้โดยบอกว่าพวกเขา "ใช้" การยืมเดียวกัน บางทีเราอาจคิดเกี่ยวกับเรเฟอเรนซ์ร่วมในลักษณะเดียวกัน?

เรามาทดสอบกับฟังก์ชันที่อ่านค่า (`println!` อาจมีเวทย์มนตร์เล็กน้อยกับ auto-ref/deref ดังนั้นฉันจึงห่อไว้ในฟังก์ชันเพื่อให้แน่ใจว่าเรากำลังทดสอบสิ่งที่เราต้องการ):

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let sref2 = &mref1;
    let sref3 = sref2;
    let sref4 = &*sref2;

    // Random hash of shared reference reads
    opaque_read(sref3);
    opaque_read(sref2);
    opaque_read(sref4);
    opaque_read(sref2);
    opaque_read(sref3);

    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run

warning: unnecessary `unsafe` block
 --> src\main.rs:6:1
  |
6 | unsafe {
  | ^^^^^^ unnecessary `unsafe` block
  |
  = note: `#[warn(unused_unsafe)]` on by default

warning: `miri-sandbox` (bin "miri-sandbox") generated 1 warning

10
10
10
10
10
11
```

ใช่ เราลืมทำอะไรกับตัวชี้ดิบ แต่อย่างน้อยเราเห็นว่าเรเฟอเรนซ์ร่วมทั้งหมดสามารถใช้แทนกันได้ ตอนนี้เรามาผสมตัวชี้ดิบ:

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &mref1;
    let ptr4 = sref3 as *mut i32;

    *ptr4 += 4;
    opaque_read(sref3);
    *ptr2 += 2;
    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run

error[E0606]: casting `&&mut i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

โอ้ โทษที เราจริงๆ กำลังเล่นกับ `& &mut` แทน `&`! Rust เก่งมากในการซ่อนสิ่งนี้เมื่อไม่สำคัญ เรามา reborrow อย่างถูกต้องด้วย `let sref3 = &*mref1`:


```text
cargo run

error[E0606]: casting `&i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

ไม่ Rust ยังไม่ชอบสิ่งนั้น! คุณสามารถ cast เรเฟอเรนซ์ร่วมเป็น `*const` ได้เท่านั้น ซึ่งอ่านได้อย่างเดียว แต่ถ้าเราแค่... ทำ... แบบนี้...?

```rust ,ignore
    let ptr4 = sref3 as *const i32 as *mut i32;
```

```text
cargo run

14
17
```

อะไร?! OK แน่ใจ Fine? ระบบ cast ที่ยอดเยี่ยมที่นี่ Rust มันเกือบจะเหมือน `*const` เป็นชนิดที่ไม่มีประโยชน์จริงๆ ที่มีอยู่จริงเพื่ออธิบาย C APIs และแนะนำการใช้งานอย่างคร่าวๆ (มันเป็นอย่างนั้น) Miri คิดอย่างไร?

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting write access to 
tag <1621> at alloc742 found in borrow stack.
  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting write access to tag <1621>
   |                at alloc742 found in borrow stack.
```

โชคร้าย แม้ว่าเราจะหลีกเลี่ยงคอมไพเลอร์บ่นด้วย double cast ได้ แต่มันไม่ได้ทำให้การดำเนินการนี้ *ได้รับอนุญาต* เมื่อเราใช้เรเฟอเรนซ์ร่วม เราสัญญาว่าจะไม่แก้ไขค่า

สิ่งนี้สำคัญเพราะนั่นหมายความว่าเมื่อการยืมร่วมถูกนำออกจากสแต็กยืม ตัวชี้ที่เปลี่ยนแปลงได้ด้านล่าง *สามารถ* สมมติว่าหน่วยความจำไม่เปลี่ยนแปลง อาจมีผู้ชายตัวเล็กบางตัว *อ่าน* หน่วยความจำ (ดังนั้นการเขียนต้องถูกดำเนินการ) แต่พวกเขาไม่สามารถแก้ไขได้ และตัวชี้ที่เปลี่ยนแปลงได้สามารถสมมติว่าค่าสุดท้ายที่เขียนยังคงอยู่!

**เมื่อเรเฟอเรนซ์ร่วมอยู่ในสแต็กยืม ทุกสิ่งที่ถูกดันขึ้นไปด้านบนจะมีสิทธิ์อ่านเท่านั้น**

อย่างไรก็ตาม เราสามารถทำแบบนี้:

```rust
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*mref1;
    let ptr4 = sref3 as *const i32 as *mut i32;

    opaque_read(&*ptr4);
    opaque_read(sref3);
    *ptr2 += 2;
    *mref1 += 1;

    opaque_read(&data);
}
```

สังเกตว่ายัง "ไม่เป็นไร" ที่จะสร้างตัวชี้ดิบที่เปลี่ยนแปลงได้ ตราบใดที่เราอ่านจากมันจริงๆ เท่านั้น:

```text
cargo run
10
10
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
10
10
13
```

และเพื่อความแน่ใจ เรามาตรวจสอบว่าเรเฟอเรนซ์ร่วมถูกนำออกตามปกติ:

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*mref1;

    *ptr2 += 2;
    opaque_read(sref3); // Read in the wrong order?
    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly 
at alloc742, but parent tag <1620> does not have an appropriate 
item in the borrow stack

  --> src\main.rs:13:17
   |
13 |     opaque_read(sref3); // Read in the wrong order?
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc742, but parent tag <1620> 
   |                       does not have an appropriate item 
   |                       in the borrow stack
   |
```

เฮ้ เราได้รับข้อความแสดงข้อผิดพลาดที่แตกต่างกันเล็กน้อยเกี่ยวกับ SharedReadOnly แทนแท็กเฉพาะ นั่นสมเหตุสมผล: เมื่อมี *เรเฟอเรนซ์ร่วม* ใดๆ ทุกอย่างอื่นก็เป็นแค่ซุป SharedReadOnly ขนาดใหญ่ จึงไม่จำเป็นต้องแยกความแตกต่าง!




# การทดสอบมิวเทบิลิตี้ภายใน

จำบทที่แย่มากของหนังสือที่เราพยายามสร้างลิงก์ลิสต์ด้วย RefCell และ Rc และทุกอย่างแย่กว่าปกติเมื่อพยายามเขียนลิงก์ลิสต์ที่น่ารังเกียจนี้?

เราได้ยืนยันว่าเรเฟอเรนซ์ร่วมไม่สามารถใช้สำหรับการเปลี่ยนแปลงได้ แต่บทนั้นเกี่ยวกับวิธีที่คุณสามารถเปลี่ยนแปลงผ่านเรเฟอเรนซ์ร่วมด้วย *มิวเทบิลิตี้ภายใน* เรามาลองชนิด [std::cell::Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html) ที่สวยงามและเรียบง่าย:

```rust
use std::cell::Cell;

unsafe {
    let mut data = Cell::new(10);
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut Cell<i32>;
    let sref3 = &*mref1;

    sref3.set(sref3.get() + 3);
    (*ptr2).set((*ptr2).get() + 2);
    mref1.set(mref1.get() + 1);

    println!("{}", data.get());
}
```

อา ความยุ่งเหยิงที่สวยงาม มันจะน่ารักมากที่เห็น miri ถ่มน้ำลายใส่


```text
cargo run
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
16
```

รอ จริงเหรอ? *นั้น* ไม่เป็นไร? ทำไม? อย่างไร? *Cell* คืออะไร?

*ทำลายกุญแจบน stdlib*

```rust ,ignore
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}
```

`UnsafeCell` คืออะไร?

*ทำลายกุญแจอีกตัวเพื่อแสดงให้ stdlib รู้ว่าเราจริงจัง*

```rust ,ignore
#[lang = "unsafe_cell"]
#[repr(transparent)]
#[repr(no_niche)]
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}
```

โอ้ มันเป็นเวทย์มนตร์ OK ฉันเดา `#[lang = "unsafe_cell"]` กำลังบอกว่า UnsafeCell คือ UnsafeCell จริงๆ เรามาหยุดทำลายกุญแจและตรวจสอบเอกสารจริงของ [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html)

> หลักการพื้นฐานสำหรับมิวเทบิลิตี้ภายในใน Rust
>
> ถ้าคุณมีเรเฟอเรนซ์ `&T` โดยปกติใน Rust คอมไพเลอร์จะเพิ่มประสิทธิภาพตามความรู้ที่ว่า `&T` ชี้ไปยังข้อมูลที่ไม่เปลี่ยนแปลง การเปลี่ยนแปลงข้อมูลนั้น ตัวอย่างเช่นผ่านการแอลิเอสหรือโดยการเปลี่ยน `&T` เป็น `&mut T` ถือเป็นพฤติกรรมที่ไม่กำหนด (Undefined Behaviour) `UnsafeCell<T>` ออกจากการรับประกันความไม่เปลี่ยนแปลงสำหรับ `&T`: เรเฟอเรนซ์ร่วม `&UnsafeCell<T>` อาจชี้ไปยังข้อมูลที่กำลังเปลี่ยนแปลง สิ่งนี้เรียกว่า "มิวเทบิลิตี้ภายใน"

โอ้ มัน *จริงๆ* เป็นแค่เวทย์มนตร์

UnsafeCell บอกคอมไพเลอร์โดยพื้นฐานว่า "เฮ้ ฟัง เราจะเล่นกับหน่วยความจำนี้ อย่าทำข้อสันนิษฐานเกี่ยวกับการแอลิเอสตามปกติ" เหมือนการตั้งป้าย "ระวัง: ผู้ชายตัวเล็กที่โกรธกำลังข้าม"

เรามาดูว่าการเพิ่ม UnsafeCell ทำให้ miri มีความสุขอย่างไร:

```rust ,ignore
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = data.get_mut();      // Get a mutable ref to the contents
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*ptr2;

    *ptr2 += 2;
    opaque_read(sref3);
    *mref1 += 1;

    println!("{}", *data.get());
}
```

```text
cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly
at alloc748, but parent tag <1629> does not have an appropriate
item in the borrow stack

  --> src\main.rs:15:17
   |
15 |     opaque_read(sref3);
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc748, but parent tag <1629> does
   |                       not have an appropriate item in the
   |                       borrow stack
   |

```

รอ อะไรนะ? เราพูดคำเวทย์แล้ว! ฉันจะทำยังไงกับเลือดแพะที่ได้รับอนุมัติจากรัฐบาลสำหรับพิธีกรรมเหล่านี้?

เราพูดแล้ว แต่เราทิ้งคาถาด้วยการใช้ `get_mut` ซึ่งแอบดูด้านใน UnsafeCell และสร้าง `&mut i32` ที่ถูกต้องให้มัน!

คิดเกี่ยวกับมัน: ถ้าคอมไพเลอร์ต้องสมมติว่า `&mut i32` *อาจ* กำลังมองเข้าไปใน `UnsafeCell` มันจะไม่สามารถทำข้อสันนิษฐานใดๆ เกี่ยวกับการแอลิเอสได้เลย! ทุกอย่างอาจเต็มไปด้วยผู้ชายตัวเล็กที่โกรธ

ดังนั้นสิ่งที่เราต้องทำคือรักษา `UnsafeCell` ไว้ในชนิดตัวชี้ของเราเพื่อให้คอมไพเลอร์เข้าใจสิ่งที่เราทำ

```rust
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = &mut data;              // Mutable ref to the *outside*
    let ptr2 = mref1.get();             // Get a raw pointer to the insides
    let sref3 = &*mref1;                // Get a shared ref to the *outside*

    *ptr2 += 2;                         // Mutate with the raw pointer
    opaque_read(&*sref3.get());         // Read from the shared ref
    *sref3.get() += 3;                  // Write through the shared ref
    *mref1.get() += 1;                  // Mutate with the mutable ref

    println!("{}", *data.get());
}
```


```text
cargo run
12
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
12
16
```

มันทำงานได้! ฉันไม่ต้องทิ้งเลือดทั้งหมดนี้

จริงๆ เฮ้ รอ เราตอนนี้ยังเล่นกับลำดับอยู่เล็กน้อย เราสร้าง ptr2 ก่อน แล้วสร้าง sref3 จากตัวชี้ที่เปลี่ยนแปลงได้ แล้วเราใช้ตัวชี้ดิบก่อนตัวชี้ร่วม ทั้งหมดดู... ผิด

จริงๆ รอ เราทำแบบนี้กับตัวอย่าง Cell ด้วย HMMMM

เราถูกบังคับให้สรุปหนึ่งในสองสิ่ง:

* Miri ไม่สมบูรณ์แบบและนี่จริงๆ ยังเป็น UB
* โมเดลที่ง่ายของเราจริงๆ เป็นการง่ายเกินไป

ฉันจะเดิมพันกับอันที่สอง แต่เพื่อความปลอดภัย เรามาสร้างเวอร์ชันที่แน่นอนว่ารั่วไม่ได้ในโมเดลที่ง่ายของเราของสแต็กด์บอร์โรวส์:

```rust
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = &mut data;
    // These two are swapped so the borrows are *definitely* totally stacked
    let sref2 = &*mref1;
    // Derive the ptr from the shared ref to be super safe!
    let ptr3 = sref2.get();             

    *ptr3 += 3;
    opaque_read(&*sref2.get());
    *sref2.get() += 2;
    *mref1.get() += 1;

    println!("{}", *data.get());
}
```

```text
cargo run
13
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
13
16
```

ตอนนี้ เหตุผลหนึ่งที่การใช้งานแรกที่เราทำ *อาจ* จริงๆ ถูกต้องเป็นเพราะถ้าคุณ *จริงๆ* คิดเกี่ยวกับมัน `&UnsafeCell<T>` จริงๆ ไม่ต่างจาก `*mut T` เมื่อพูดถึงการแอลิเอส คุณสามารถคัดลอกมันได้ไม่จำกัดและเปลี่ยนแปลงผ่านมัน!

ดังนั้นในบางแง่เราแค่สร้างตัวชี้ดิบสองตัวและใช้แทนกันตามปกติ มัน *เล็กน้อย* น่าสงสัยที่ทั้งสองได้มาจากเรเฟอเรนซ์ที่เปลี่ยนแปลงได้ ดังนั้นบางทีการสร้างตัวที่สองควรยังนำตัวแรกออกจากสแต็กยืม แต่ไม่จำเป็นจริงๆ เพราะเราไม่ *จริงๆ* เข้าถึงเนื้อหาของเรเฟอเรนซ์ที่เปลี่ยนแปลงได้ แค่คัดลอกที่อยู่ของมัน

บรรทัดเช่น `let sref2 = &*mref1` เป็นสิ่งที่หลอกลวง *ทางไวยากรณ์* ดูเหมือนเรากำลัง dereference มัน แต่ dereference ด้วยตัวเองไม่ใช่ *สิ่ง* จริงๆ? พิจารณา `&my_tuple.0`: คุณไม่ได้ทำอะไรกับ `my_tuple` หรือ `.0` คุณแค่ใช้พวกเขาเพื่ออ้างอิงตำแหน่งในหน่วยความจำและใส่ `&` ไว้ด้านหน้าที่บอกว่า "อย่าโหลดสิ่งนี้ แค่เขียนที่อยู่"

`&*` เป็นสิ่งเดียวกัน: `*` กำลังบอกว่า "เฮ้ มาพูดคุยเกี่ยวกับตำแหน่งที่ตัวชี้นี้ชี้ถึง" และ `&` กำลังบอกว่า "ตอนนี้เขียนที่อยู่นั้น" ซึ่งจริงๆ เป็นค่าเดียวกับที่ตัวชี้ดั้งเดิมมี แต่ชนิดของตัวชี้เปลี่ยนไป เพราะ เอ่อ ชนิด!

นั่นพูดแล้ว ถ้าคุณทำ `&**` จริงๆ คุณกำลังโหลดค่าด้วย `*` ตัวแรก! `*` แปลก!

> **ผู้บรรยาย:** ไม่มีใครสนว่าคุณรู้คำว่า "lvalue" *โจนาธาน* ใน Rust เราเรียกมันว่า *places* ซึ่งต่างกันโดยสิ้นเชิงและ *เท่กว่า* มาก?




# การทดสอบ Box

เฮ้ จำได้ว่าทำไมเราเริ่มเนื้อเรื่องยาวนี้? คุณไม่จำ? แปลก

ก็เพราะเราผสม Box และตัวชี้ดิบ Box *คล้าย* กับ `&mut` เพราะมันอ้างความเป็นเจ้าของที่ไม่ซ้ำกันของหน่วยความจำที่มันชี้ถึง เรามาทดสอบการอ้างนั้น:

```rust ,ignore
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *data += 10;
    *ptr1 += 1;

    // Should be 21
    println!("{}", data);
}
```

```text
cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
       to tag <1707> at alloc763 found in borrow stack.

 --> src\main.rs:7:5
  |
7 |     *ptr1 += 1;
  |     ^^^^^^^^^^ no item granting read access to tag <1707> 
  |                at alloc763 found in borrow stack.
  |
```

ใช่ miri เกลียดสิ่งนั้น เรามาตรวจสอบว่าการทำสิ่งต่างๆ ในลำดับที่ถูกต้องไม่เป็นไร:

```rust
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *ptr1 += 1;
    *data += 10;

    // Should be 21
    println!("{}", data);
}
```

```text
cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
21
```

ใช่!

นั่นคือทั้งหมด ทุกคน ในที่สุดเราก็เสร็จสิ้นการพูดคุยและคิดเกี่ยวกับสแต็กด์บอร์โรวส์!

...รอ เราจะแก้ปัญหานี้กับ Box ได้อย่างไร? เหมือน แน่ใจ เราสามารถเขียนโปรแกรมของเล่นเช่นนี้ได้ แต่เราต้องเก็บ Box ไว้ที่ไหนสักแห่งและถือตัวชี้ดิบของเราเป็นเวลานาน ของจะต้องยุ่งเหยิงและไม่ถูกต้อง?

คำถามที่ดี! เพื่อตอบคำถามนี้ ในที่สุดเราจะกลับไปยัง Calling ที่แท้จริงของเรา: เขียนลิงก์ลิสต์ที่น่ารังเกียจ

รอ ฉันต้องเขียนลิงก์ลิสต์อีกครั้ง? อย่ารีบ ทุกคน มีเหตุผล รอสักครู่ ฉันแน่ใจว่ามีปัญหาที่น่าสนใจอื่นๆ ให้ฉันพูดคุ&mdash;
