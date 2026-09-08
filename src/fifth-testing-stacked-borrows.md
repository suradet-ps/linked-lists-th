# การทดสอบ Stacked Borrows

> สรุปย่อ (TL;DR) ของโมเดลหน่วยความจำ (ฉบับย่อ) ของ Rust จากหัวข้อก่อนหน้า:
>
> * ในทางแนวคิด Rust จัดการเรื่อง reborrow ด้วยการรักษาสิ่งที่เรียกว่า "borrow stack"
> * จะมีเพียงพอยน์เตอร์ที่อยู่บนยอดสุดของสแต็กเท่านั้นที่ถือว่า "มีชีวิต (live)" (มีสิทธิ์เข้าถึงแบบเอกสิทธิ์)
> * เมื่อคุณเข้าถึงพอยน์เตอร์ตัวที่อยู่ลึกลงไปในสแต็ก พอยน์เตอร์ตัวนั้นจะกลับมา "มีชีวิต" และพอยน์เตอร์ทั้งหมดที่อยู่เหนือมันจะถูก pop ทิ้งไป
> * คุณไม่ได้รับอนุญาตให้นำพอยน์เตอร์ที่เคยถูก pop ทิ้งออกจาก borrow stack ไปแล้วกลับมาใช้งานอีก
> * ตัว borrow checker จะช่วยรับประกันว่าโค้ดที่ปลอดภัย (safe code) ปฏิบัติตามกฎนี้เสมอ
> * ในทางทฤษฎี Miri จะคอยตรวจสอบว่า raw pointer ปฏิบัติตามกฎนี้หรือไม่ตอนรันไทม์

นั่นเป็นทฤษฎีและไอเดียที่อัดแน่นมาก — ตอนนี้เรามาเข้าสู่หัวใจและจิตวิญญาณที่แท้จริงของหนังสือเล่มนี้กันดีกว่า: นั่นคือการเขียนโค้ดแย่ๆ แล้วรอให้เครื่องมือต่างๆ กรีดร้องด่าทอใส่เรา เราจะมาลุยกับตัวอย่างโค้ด*สารพัดแบบ* เพื่อดูว่า mental model ในหัวของเรานั้นถูกต้องหรือไม่ และเพื่อสร้างสัญชาตญาณความเข้าใจในเรื่อง Stacked Borrows กันครับ

> **ผู้บรรยาย:** การตามล่าดักจับ Undefined Behaviour ในโลกความเป็นจริงนั้นเป็นเรื่องที่ยุ่งยากน่าปวดหัวอย่างยิ่ง เพราะท้ายที่สุดแล้ว คุณกำลังรับมือกับสถานการณ์ที่คอมไพเลอร์*ทึกทักเอาเอง*ว่ามันไม่มีทางเกิดขึ้น
>
> ถ้าคุณโชคดี โค้ดของคุณอาจจะ "ดูเหมือนทำงานได้ดี" ในวันนี้ แต่มันจะกลายเป็นระเบิดเวลาที่รอวันปะทุเมื่อมีคอมไพเลอร์รุ่นที่ฉลาดขึ้น หรือเมื่อมีการแก้โค้ดเพียงเล็กน้อยในอนาคต แต่ถ้าคุณ*โชคดีสุดๆ* โปรแกรมจะแครชพังแบบสม่ำเสมอจนคุณสามารถจับจุดผิดพลาดและตามแก้ได้ทันที ทว่า... ถ้าคุณโชคร้าย ทุกสิ่งทุกอย่างจะพังทลายลงมาในรูปแบบที่พิลึกกึกกือและชวนให้สับสนจนหาคำตอบไม่ได้
>
> Miri พยายามแก้ปัญหานี้โดยการดึงเอามุมมองของ rustc ในแบบที่ไร้เดียงสาและไม่ได้ optimize ที่สุดของโปรแกรมมา แล้วคอยติดตามสถานะเพิ่มเติมไปทีละสเต็ปขณะที่มัน interpret โค้ด ในบรรดาเครื่องมือประเภท "sanitizer" ทั้งหลาย นี่ถือเป็นวิธีที่มีความแน่นอน (deterministic) และทนทานทีเดียว แต่มันไม่มีทางที่จะ*สมบูรณ์แบบไร้ที่ติ*ได้ เพราะโปรแกรมทดสอบของคุณจำเป็นต้องมี execution path ที่วิ่งไปเจอกับจุดที่เกิด UB นั้นจริงๆ และสำหรับโปรแกรมขนาดใหญ่ มันง่ายมากที่จะเกิดความไม่แน่นอนสารพัดรูปแบบขึ้นมา (เช่น `HashMap` ที่ใช้ RNG เป็นค่าเริ่มต้น!)
>
> เราจึงไม่สามารถปักใจเชื่อแบบ 100% ได้ว่า การที่ Miri บอกว่าโปรแกรมผ่านนั้นแปลว่าจะไม่มี UB หลุดรอดอยู่เลย ยิ่งไปกว่านั้น เป็นไปได้เหมือนกันที่ Miri จะ*เข้าใจผิดคิดว่า*มีบางอย่างเป็น UB ทั้งที่จริงๆ แล้วมันไม่ใช่ แต่ถ้าเรามี mental model ที่ชัดเจนว่าสิ่งต่างๆ ทำงานอย่างไร และ Miri ดูเหมือนจะเห็นพ้องต้องกันกับเรา นั่นก็ถือเป็นสัญญาณที่ดีมากแล้วว่าเรากำลังเดินมาถูกทาง




# การยืมพื้นฐาน

ในหัวข้อก่อนหน้านี้ เราได้เห็นแล้วว่า borrow checker ไม่ชอบโค้ดชุดนี้เอาเสียเลย:

```rust ,ignore
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

// ORDER SWAPPED!
*ref1 += 1;
*ref2 += 2;

println!("{}", data);
```

คราวนี้ มาดูกันว่าจะเกิดอะไรขึ้นเมื่อเราแทนที่ `ref2` ด้วย `*mut`:

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

ดูเหมือน rustc จะแฮปปี้กับโค้ดนี้มาก: ไม่มีคำเตือนใดๆ โผล่มาเลย และโปรแกรมก็ให้ผลลัพธ์ออกมาเป็น 13 ตามที่เราคาดหวังเป๊ะ! ทีนี้ลองมาดูซิว่า Miri (ในโหมดเข้มงวด) จะคิดเห็นอย่างไรกับโค้ดนี้:

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

เยี่ยมไปเลย! สัญชาตญาณของเราเกี่ยวกับกลไกการทำงานยังคงแม่นยำ: แม้ว่าคอมไพเลอร์จะตรวจจับปัญหานี้ไม่ได้ แต่ Miri จับได้คาหนังคาเขา

งั้นเรามาลองอะไรที่ซับซ้อนขึ้นอีกนิด กรณี `&mut -> *mut -> *mut -> *mut` ที่เราเคยพูดถึงก่อนหน้านี้:

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

โอ้โห ใช่เลย! ในโหมดเข้มงวด Miri สามารถ "แยกแยะ" raw pointer ทั้งสองตัวออกจากกันได้ และการเข้าถึงตัวแรกก่อนทำให้ตัวที่สร้างตามมาภายหลังกลายเป็นโมฆะทันที คราวนี้มาดูกันว่าโปรแกรมจะกลับมาทำงานได้ไหมถ้าเราเอาการเข้าถึงตัวแรกที่ทำให้สแต็กพังออกไป:

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

สวยงาม!

ใช่แล้วครับ ตอนนี้ผมค่อนข้างมั่นใจสุดๆ เลยว่าพวกเราทุกคนพร้อมจะรับปริญญาเอกสาขาการออกแบบและวางระบบ memory model ของภาษาโปรแกรมได้แล้ว ใครมันจะไป*ต้องการ*คอมไพเลอร์กัน เรื่องพวกนี้มัน*หมูๆ*

> **ผู้บรรยาย:** มันไม่ได้หมูเลยสักนิด... แต่ยังไงฉันก็ภูมิใจในตัวเธอนะ




# การทดสอบอาร์เรย์

คราวนี้มาลองเล่นกับอาร์เรย์และการคำนวณตำแหน่งพอยน์เตอร์ด้วย offset (`add` และ `sub`) กันบ้าง โค้ดแบบนี้ควรจะทำงานได้เนอะ ว่าไหม?

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

*ฉีกใบปริญญาทิ้งทันควัน*

เกิดอะไรขึ้นกันล่ะเนี่ย? เราก็จัดลำดับการใช้งานตาม borrow stack เป็นอย่างดีแล้วนี่นา! มันมีอะไรพิลึกๆ เกิดขึ้นตอนที่เราแปลงจาก `ptr -> ptr` หรือเปล่า? แล้วถ้าเราแค่ก็อปปี้พอยน์เตอร์ให้ชี้ไปที่ตำแหน่งเดียวกันล่ะ:

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

อ้าว แบบนั้นกลับทำงานได้ฉลุย หรือว่าเราแค่ฟลุก? งั้นมาลองสับพอยน์เตอร์ให้ยุ่งเหยิงแบบสุดขั้วไปเลย:

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

ไม่แฮะ Miri ค่อนข้างผ่อนปรนมากเมื่อเป็นเรื่องของ raw pointer ที่ถูกแตกแขนงมาจาก raw pointer ตัวอื่น เพราะพวกมันทั้งหมดจะใช้ "สิทธิ์การยืม" อันเดียวกัน (หรือที่ Miri เรียกว่า *แท็ก (tag)*)

เมื่อไหร่ก็ตามที่คุณก้าวเข้าสู่โลกของ raw pointer พวกมันสามารถแตกตัวออกเป็นเจ้าคนแคระขี้โมโหของตัวเอง แล้ววิ่งเล่นกันได้อย่างอิสระเสรี ซึ่งตรงนี้ไม่เป็นไร เพราะคอมไพเลอร์เข้าใจพฤติกรรมนี้ดี และจะไม่พยายาม optimize การอ่านเขียนหน่วยความจำแบบดุดันเหมือนที่มันทำกับ reference ปกติ

> **ผู้บรรยาย:** หากโค้ดมีความเรียบง่ายเพียงพอ คอมไพเลอร์ก็อาจจะสามารถแกะรอยตามพอยน์เตอร์ที่แตกแขนงออกมาทั้งหมดได้ และยังคง optimize ให้ได้อยู่ แต่การวิเคราะห์แบบนั้นจะเปราะบางกว่าการวิเคราะห์ reference ธรรมดาๆ มาก

ถ้าอย่างนั้น *ปัญหาที่แท้จริง* มันคืออะไรกันแน่ล่ะ?

คำตอบก็คือ ถึงแม้ว่า `data` จะเป็น "การจัดสรร (allocation)" ก้อนเดียวกัน (ตัวแปร local ตัวหนึ่ง) แต่ตัว `ref1_at_0` นั้นกำลังยืมข้อมูลเฉพาะเอลิเมนต์แรกตัวเดียวเท่านั้น! ในภาษา Rust การยืมสามารถถูกจำกัดขอบเขตให้ครอบคลุมเฉพาะพื้นที่บางส่วนของ allocation ได้! มาลองพิสูจน์ดู:

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

พังยับ! Rust ไม่ได้ฉลาดพอที่จะมานั่งแกะรอยดัชนีของอาร์เรย์เพื่อพิสูจน์ว่าการยืมสองจุดนี้ไม่ได้ทับซ้อนกัน แต่มันได้มอบฟังก์ชัน `split_at_mut` มาให้เราใช้แบ่ง slice ออกเป็นสองส่วนได้อย่างปลอดภัย:

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

เฮ้ ทำงานได้แล้ว! slice ได้บอกคอมไพเลอร์และ Miri ไว้อย่างชัดเจนว่า "เฮ้ย ฉันกำลังขอยืมหน่วยความจำทั้งหมดในช่วงของฉันอยู่นะ" ทั้งคอมไพเลอร์และ Miri จึงรับรู้ได้ว่าเอลิเมนต์ทั้งหมดในช่วงนั้นสามารถแก้ไขค่าได้

นอกจากนี้ การที่ Rust ยอมให้มี operation อย่าง `split_at_mut` ยังบอกเราอีกด้วยว่า การยืมอาจจะไม่ได้มีโครงสร้างเป็น *สแต็ก* เป๊ะๆ เสมอไป แต่อาจจะคล้ายกับ *ต้นไม้ (Tree)* มากกว่า เพราะเราสามารถหั่นแบ่งการยืมก้อนใหญ่ก้อนหนึ่งออกเป็นชิ้นส่วนย่อยๆ ที่ไม่ทับซ้อนกัน แล้วทุกอย่างก็ยังทำงานได้อย่างราบรื่น

(ในความเข้าใจของผม ในโมเดล Stacked Borrows จริงๆ ทุกสิ่งทุกอย่างก็ยังคงเป็นสแต็กอยู่ดี เพราะสแต็กจะคอยติดตามสิทธิ์สำหรับแต่ละไบต์ของหน่วยความจำในโปรแกรมแยกกัน)

แล้วถ้าเรา *แปลง* slice ออกมาเป็นพอยน์เตอร์โดยตรงเลยล่ะ? พอยน์เตอร์ตัวนั้นจะมีสิทธิ์เข้าถึงครอบคลุมทั้ง slice เลยไหมนะ?

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


ยอดเยี่ยม! พอยน์เตอร์ไม่ใช่แค่ตัวเลขจำนวนเต็มธรรมดา: แต่มันมีช่วงของหน่วยความจำผูกติดมาด้วย และใน Rust เราได้รับอนุญาตให้จำกัดช่วงนั้นให้แคบลงได้!




# การทดสอบ Shared Reference

ในตัวอย่างก่อนหน้านี้ทั้งหมด ผมตั้งใจระมัดระวังเป็นพิเศษที่จะใช้เฉพาะ mutable reference และทำเฉพาะคำสั่งประเภท read-modify-write (`+=`) เพื่อให้ภาพการทดลองเข้าใจง่ายที่สุด

ทว่า Rust ยังมี shared reference ที่เป็นแบบ read-only และสามารถก็อปปี้ไปมาได้อย่างอิสระอีกด้วย แล้วของพวกนี้มันมีพฤติกรรมอย่างไรกันนะ? เราได้เห็นกันไปแล้วว่า raw pointer สามารถก็อปปี้ได้อย่างอิสระ โดยโมเดลจะมองว่าพวกมัน "แชร์" สิทธิ์การยืมอันเดียวกัน แล้วเราจะมอง shared reference ในลักษณะเดียวกันได้หรือไม่?

มาลองทดสอบกับฟังก์ชันที่ทำหน้าที่อ่านค่าเฉยๆ ดูกันครับ (เนื่องจาก `println!` อาจมีลูกเล่นเวทมนตร์พวก auto-ref/deref ซ่อนอยู่ ผมจึงขอห่อมันไว้ในฟังก์ชันธรรมดา เพื่อให้มั่นใจได้ว่าเรากำลังทดสอบสิ่งที่เราตั้งใจจะทดสอบจริงๆ):

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

ใช่ครับ เราลืมใส่ลูกเล่นเกี่ยวกับ raw pointer ลงไป แต่ก็ยังดีที่ทำให้เราเห็นว่า shared reference ทั้งหมดสามารถสลับใช้งานแทนกันได้อย่างอิสระ คราวนี้ลองมาผสม raw pointer เข้าไปด้วย:

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

อุ๊ย ขออภัยครับ เราดันเผลอไปเล่นกับ `& &mut` แทนที่จะเป็น `&` เสียได้! Rust มักจะเก่งเรื่องการซ่อนสิ่งพวกนี้ไว้เมื่อมันไม่สำคัญ งั้นเรามาทำการ reborrow ให้ถูกต้องด้วย `let sref3 = &*mref1` กันใหม่:


```text
cargo run

error[E0606]: casting `&i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

ไม่ได้แฮะ Rust ยังคงไม่ชอบสิ่งนี้อยู่ดี! คุณสามารถ cast shared reference ไปเป็น `*const` ได้เท่านั้น ซึ่งเป็นแบบอ่านได้อย่างเดียว แต่... ถ้าเราแอบ... ทำ... แบบนี้ล่ะ...?

```rust ,ignore
    let ptr4 = sref3 as *const i32 as *mut i32;
```

```text
cargo run

14
17
```

อะไรกันเนี่ย?! โอเค้ ได้ เอาเลยตามสบาย! ระบบ type casting ของ Rust ช่างยอดเยี่ยมเสียจริง ราวกับว่าชนิด `*const` นั้นเป็นเพียง type ที่ไร้ประโยชน์โดยสิ้นเชิง ซึ่งมีตัวตนอยู่เพียงเพื่อใช้อธิบาย C API และให้คำแนะนำการใช้งานอย่างหลวมๆ เท่านั้น (ซึ่งมันก็เป็นอย่างนั้นจริงๆ นั่นแหละ) แล้ว Miri คิดอย่างไรกับท่าพิสดารนี้กันล่ะ?

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

อนิจจา ถึงแม้เราจะรอดพ้นจากการบ่นของคอมไพเลอร์ด้วยการ cast สองเด้งมาได้ แต่นั่นไม่ได้ทำให้ operation นี้*ได้รับอนุญาต*แต่อย่างใด ทันทีที่เราหยิบ shared reference มาใช้งาน เราได้ให้สัญญากับ Rust ไปแล้วว่าจะไม่มีการแก้ไขค่าใดๆ ทั้งสิ้น

เรื่องนี้สำคัญมาก เพราะมันหมายความว่า เมื่อการยืมแบบ shared ถูก pop ออกจาก borrow stack ไปแล้ว ตัว mutable pointer ที่อยู่ข้างล่าง*จะสามารถ*มั่นใจได้ว่าหน่วยความจำนั้นไม่มีการเปลี่ยนแปลง อาจจะมีเจ้าคนแคระบางตัวแอบมา*อ่าน*หน่วยความจำไปบ้าง (ดังนั้นการเขียนลงหน่วยความจำจึงต้องเกิดขึ้นจริง) แต่มันไม่มีสิทธิ์แก้ไขค่า และตัว mutable pointer ก็สามารถเชื่อใจได้ว่าค่าที่มันเขียนลงไปล่าสุดจะยังคงอยู่ครบถ้วน!

**เมื่อมี shared reference วางอยู่บน borrow stack ทุกสิ่งทุกอย่างที่ถูกวางทับซ้อนอยู่เหนือมันจะมีสิทธิ์เพียงแค่อ่านเท่านั้น**

อย่างไรก็ตาม เรายังสามารถเขียนแบบนี้ได้:

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

สังเกตไหมครับว่า เรายังคง "ทำได้" ที่จะสร้าง raw pointer แบบ mutable ขึ้นมา ตราบใดที่เราใช้มันแค่อ่านค่าเท่านั้น:

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

และเพื่อให้มั่นใจ เรามาลองตรวจสอบดูว่า shared reference จะถูก pop ออกจากสแต็กตามปกติหรือไม่:

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

เฮ้ รอบนี้เราได้ error message ที่ต่างออกไปนิดหน่อยแฮะ มันฟ้องเรื่อง `SharedReadOnly` แทนที่จะเป็นเลขแท็กเฉพาะ ซึ่งก็สมเหตุสมผลดี: เมื่อไหร่ก็ตามที่มี *shared reference* โผล่เข้ามา ทุกสิ่งทุกอย่างที่ตามมาก็เป็นแค่ก้อนซุป SharedReadOnly รวมมิตรก้อนใหญ่ จึงไม่จำเป็นต้องมานั่งแยกแยะแท็กให้วุ่นวายอีกต่อไป!




# การทดสอบ Interior Mutability

ยังจำบทอันแสนทรมานของหนังสือเล่มนี้ได้ไหมครับ ตอนที่เราพยายามจะสร้าง linked list ด้วย `RefCell` กับ `Rc` แล้วทุกสิ่งทุกอย่างก็เลวร้ายยิ่งกว่าเดิมเมื่อต้องมานั่งเขียนลิสต์พิลึกกึกกือนั่น?

เราเพิ่งจะยืนยันไปหยกๆ ว่า shared reference ไม่สามารถนำมาใช้แก้ไขข้อมูลได้ แต่บทนั้นมันเป็นเรื่องของการที่เราสามารถแก้ไขข้อมูลผ่าน shared reference ได้โดยอาศัยสิ่งที่เรียกว่า *Interior Mutability* งั้นเรามาลองทดสอบ type อย่าง [std::cell::Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html) ที่แสนเรียบง่ายและงดงามกันดูดีกว่า:

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

อ่า... ช่างเป็นความยุ่งเหยิงที่งดงาม คงจะฟินพิลึกถ้าได้เห็น Miri พ่นความผิดพลาดใส่หน้าเรา

```text
cargo run
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
16
```

เดี๋ยวสิ จริงดิ?! โค้ดแบบ*นั้น*ผ่านฉลุยเนี่ยนะ? ทำไมล่ะ? ได้ยังไง? แล้ว *Cell* มันคือตัวบ้าอะไรกันแน่?

*งัดแม่กุญแจของ stdlib พังกระจุย*

```rust ,ignore
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}
```

แล้ว `UnsafeCell` คืออะไรอีกล่ะ?

*งัดแม่กุญแจอีกดอกทิ้ง เพื่อให้ stdlib รู้ว่าเราเอาจริง!*

```rust ,ignore
#[lang = "unsafe_cell"]
#[repr(transparent)]
#[repr(no_niche)]
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}
```

อ๋อ... เป็นเวทมนตร์ของพวกพ่อมดนี่เอง โอเค เข้าใจละ ไอ้ attribute `#[lang = "unsafe_cell"]` เนี่ยมันคงกำลังบอกคอมไพเลอร์ว่า UnsafeCell ตัวนี้คือ UnsafeCell ตัวจริงเสียงจริงสินะ เอาล่ะ เลิกงัดแม่กุญแจแล้วไปเปิดดูเอกสารทางการของ [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html) กันดีกว่า

> สิ่งพื้นฐานหลักสำหรับ interior mutability ในภาษา Rust
>
> หากคุณมี reference ชนิด `&T` โดยปกติคอมไพเลอร์ของ Rust จะทำการ optimize โค้ดบนสมมติฐานที่ว่า `&T` ชี้ไปยังข้อมูลที่ไม่สามารถเปลี่ยนแปลงค่าได้ การแอบแก้ไขข้อมูลดังกล่าว เช่น ผ่านการ alias หรือการแปลงชนิดข้อมูลจาก `&T` ไปเป็น `&mut T` จะถือเป็นพฤติกรรม Undefined Behavior ทันที แต่ `UnsafeCell<T>` จะขอถอนตัวออกจากการการันตีความไม่เปลี่ยนแปลงของ `&T`: ส่งผลให้ shared reference ชนิด `&UnsafeCell<T>` อาจจะชี้ไปยังข้อมูลที่กำลังถูกแก้ไขอยู่ได้ ซึ่งสิ่งนี้เราเรียกว่า "interior mutability"

โอ้โห มันเป็นเวทมนตร์มนต์ดำ*ของแท้แน่นอน*เลยแฮะ

โดยพื้นฐานแล้ว `UnsafeCell` จะบอกคอมไพเลอร์ว่า "เฮ้ ฟังนะพวก เรากำลังจะเล่นแผลงๆ กับหน่วยความจำก้อนนี้ อย่าได้ริอ่านเอาสมมติฐานเรื่อง aliasing ตามปกติมาใช้เชียวนะ" เหมือนกับการปักป้ายเตือนตัวเบ้อเริ่มว่า "ระวัง: คนแคระหัวร้อนกำลังข้ามถนน"

มาดูซิว่าการเติม `UnsafeCell` เข้าไปจะทำให้ Miri มีความสุขขึ้นอย่างไร:

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

เดี๋ยวนะ อะไรเนี่ย!? เราก็ร่ายคาถาเวทมนตร์ถูกต้องครบถ้วนแล้วนี่นา! แล้วผมจะเอาเลือดแพะเสริมพลังพิธีกรรมที่ผ่านการรับรองจากรัฐบาลพวกนี้ไปเททิ้งที่ไหนล่ะเนี่ย?

อืม... จริงอยู่ที่เราร่ายคาถาไปแล้ว แต่เราดันทำลายเวทมนตร์นั้นทิ้งไปเอง ด้วยการเรียกใช้ `get_mut` ซึ่งแอบมุดเข้าไปในไส้ในของ `UnsafeCell` แล้วแปลงร่างออกมาเป็น `&mut i32` ธรรมดาๆ ซะงั้น!

ลองคิดดูสิครับ: ถ้าคอมไพเลอร์ต้องมาคอยระแวงว่า `&mut i32` ทั่วๆ ไป*อาจจะ*แอบมุดเข้าไปใน `UnsafeCell` หรือเปล่า มันก็จะไม่สามารถตั้งสมมติฐานเรื่อง aliasing อะไรได้อีกต่อไปเลยในชีวิตนี้! โลกนี้คงเต็มไปด้วยคนแคระหัวร้อนวิ่งเพ่นพ่านไปหมด

ดังนั้น สิ่งที่เราต้องทำก็คือ คงสถานะของ `UnsafeCell` เอาไว้ใน pointer type ของเราต่อไป เพื่อให้คอมไพเลอร์เข้าใจตรงกันว่าเรากำลังทำอะไรอยู่:

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

รอดแล้ว! ในที่สุดผมก็ไม่ต้องเอากะละมังเลือดแพะพวกนี้ไปเททิ้งแล้ว

จริงสิ เฮ้ย เดี๋ยวก่อนนะ เรายังคงจัดลำดับพิสดารอยู่นิดหน่อยในโค้ดนี้ เพราะเราสร้าง `ptr2` ขึ้นมาก่อน แล้วค่อยสร้าง `sref3` จาก mutable pointer ตัวเดิม จากนั้นเราก็เรียกใช้ raw pointer ก่อน shared pointer ซึ่งทั้งหมดนี้มันดู... ผิดหลักการชอบกล

เดี๋ยวนะ ตอนตัวอย่าง `Cell` เราก็ทำแบบนี้นี่หว่า หืมมมมม...

เราคงต้องยอมรับข้อสรุปข้อใดข้อหนึ่งในสองข้อนี้แล้วล่ะ:

* Miri ยังคงไม่สมบูรณ์แบบ และโค้ดนี้จริงๆ แล้วยังคงมี UB แฝงอยู่
* หรือไม่ก็ โมเดลฉบับย่อของเรามันย่อจนตกหล่นรายละเอียดบางอย่างไปจริงๆ

ผมขอแทงข้างข้อสองครับ แต่เพื่อความปลอดภัย เรามาสร้างเวอร์ชันที่ถูกต้องรัดกุม 100% ตามโมเดล Stacked Borrows ฉบับย่อของเรากันดีกว่า:

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

เหตุผลหนึ่งที่โค้ดแบบแรกที่เราเขียน*อาจจะ*ถูกต้องจริงๆ ก็คือ: ถ้าเราลองคิดดู*ให้ลึกซึ้งจริงๆ* ในมุมมองของเรื่อง aliasing แล้ว `&UnsafeCell<T>` แทบจะไม่ได้ต่างอะไรกับ `*mut T` เลยแม้แต่น้อย เพราะคุณสามารถก็อปปี้มันไปมาได้ไม่รู้จบ และสามารถแก้ไขข้อมูลผ่านมันได้ตลอดเวลา!

ดังนั้น ในมุมหนึ่ง เราก็แค่สร้าง raw pointer ขึ้นมาสองตัว แล้วสลับใช้งานพวกมันตามปกติเท่านั้นเอง มันอาจจะดู*ตะขิดตะขวงใจนิดหน่อย*ตรงที่พอยน์เตอร์ทั้งคู่ถูกแตกแขนงมาจาก mutable reference ตัวเดียวกัน ซึ่งตามหลักแล้ว การสร้างตัวที่สองควรจะทำให้ตัวแรกหลุดออกจาก borrow stack ไป แต่ในทางปฏิบัติมันไม่จำเป็นต้องทำแบบนั้น เพราะเราไม่ได้*เข้าถึงไส้ใน*ของ mutable reference จริงๆ เราแค่ก็อปปี้ memory address ของมันออกมาเฉยๆ

โค้ดอย่าง `let sref2 = &*mref1` เป็นอะไรที่ชวนเข้าใจผิดมาก *ในเชิงไวยากรณ์* มันดูเหมือนว่าเรากำลังทำการ dereference ค่าข้างในออกมา แต่การ dereference โดดๆ เดี่ยวๆ แบบนั้นในความเป็นจริงมันไม่ได้ทำอะไรกับข้อมูลเลย! ลองนึกถึงคำสั่ง `&my_tuple.0` ดูสิครับ: คุณไม่ได้ไปแตะต้องอะไรกับ `my_tuple` หรือ `.0` เลย คุณแค่ใช้มันเป็นตัวระบุตำแหน่งในหน่วยความจำ แล้วแปะเครื่องหมาย `&` ไว้ข้างหน้าเพื่อบอกคอมไพเลอร์ว่า "ไม่ต้องโหลดค่านั้นขึ้นมานะ แค่จด memory address ของมันเอาไว้ก็พอ"

คำสั่ง `&*` ก็ทำหน้าที่แบบเดียวกันเป๊ะ: ตัว `*` แค่บอกว่า "เฮ้ เรากำลังจะพูดถึงตำแหน่งหน่วยความจำที่พอยน์เตอร์ตัวนี้ชี้อยู่นะ" แล้วตัว `&` ก็บอกว่า "งั้นช่วยจด memory address ตรงนั้นเก็บไว้ให้ที" ซึ่งแน่นอนว่ามันก็คือค่าเดียวกันกับที่พอยน์เตอร์เดิมถืออยู่นั่นแหละ เพียงแต่ชนิดของพอยน์เตอร์เปลี่ยนไป เพราะว่า... ก็นะ เรื่องของ Types ไงล่ะ!

อย่างไรก็ตาม ถ้าคุณเขียน `&**` เมื่อไหร่ คราวนี้แหละที่คุณจะทำการโหลดค่าจริงๆ ผ่านตัว `*` ตัวแรก! เครื่องหมาย `*` นี่มันพิลึกดีแท้!

> **ผู้บรรยาย:** ไม่มีใครเขาแคร์หรอกนะว่าเธอรู้จักคำว่า "lvalue" น่ะ *โจนาธาน* ในโลกของ Rust เราเรียกสิ่งนี้ว่า *places* ย่ะ ซึ่งมันคนละเรื่องกันเลยและ*เท่กว่าตั้งเยอะ* จริงมะ?




# การทดสอบ Box

เฮ้ จำได้ไหมว่าทำไมเราถึงเริ่มออกทะเลมายาวเหยียดขนาดนี้? จำไม่ได้เหรอ? แปลกจัง

ก็เพราะเราดันเอา Box ไปผสมปนเปกับ raw pointer ไงล่ะ! Box มีพฤติกรรม*คล้ายๆ* กับ `&mut` ตรงที่มันอ้างสิทธิ์ความเป็นเจ้าของแต่เพียงผู้เดียว (unique ownership) บนหน่วยความจำที่มันชี้อยู่ งั้นเรามาลองทดสอบข้ออ้างนี้กันดูหน่อย:

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

ใช่แล้ว Miri เกลียดโค้ดแบบนั้นเข้าไส้เลย คราวนี้มาลองจัดลำดับการใช้งานให้ถูกต้องดูซิว่าจะรอดไหม:

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

รอดฉลุย!

เอาล่ะครับทุกท่าน นั่นคือทั้งหมดทั้งมวล! ในที่สุดเราก็พูดคุยและขบคิดเรื่อง Stacked Borrows กันจนจบสิ้นเสียที!

...เดี๋ยวนะ แล้วเราจะแก้ปัญหาคาราคาซังนี้กับ `Box` ยังไงกันล่ะ? คือจริงอยู่ว่าเราเขียนโปรแกรมของเล่นแบบนี้ให้ผ่านได้ แต่ในโค้ดจริงเราต้องเก็บกล่อง `Box` เอาไว้ที่ไหนสักแห่ง แล้วต้องถือ raw pointer ค้างไว้นานพอดู ข้าวของมันจะไม่ตีกันมั่วจนสิทธิ์การยืมพังพินาศไปหมดงั้นเหรอ?

เป็นคำถามที่ยอดเยี่ยมมากครับ! และเพื่อที่จะตอบคำถามนี้ ในที่สุดเราก็จะได้หวนคืนสู่ภารกิจอันแท้จริงในชีวิตของเราเสียที: นั่นคือการกลับไปก้มหน้าก้มตาเขียนไอ้ linked list เวรตะไลนี่ต่อ

เดี๋ยวสิ... ผมต้องกลับไปเขียน linked list อีกแล้วเหรอเนี่ย? ใจเย็นๆ ก่อนพวกเรา อย่าเพิ่งรีบร้อนสิ ฟังเหตุผลกันก่อน เดี๋ยวนะ ผมมั่นใจว่ามันต้องมีประเด็นน่าสนใจอื่นๆ ให้ผมหยิบยกมาพู&mdash;
