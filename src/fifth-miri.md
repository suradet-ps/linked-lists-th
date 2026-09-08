# Miri

*หัวเราะแห้งๆ อย่างประหม่า* แหม เรื่องโค้ด unsafe นี่มันช่างง่ายดายอะไรเบอร์นี้ ไม่เห็นเข้าใจเลยว่าทำไมใครๆ ถึงบ่นว่ายาก โปรแกรมของเราทำงานได้สมบูรณ์แบบไร้ที่ติสุดๆ

> **ผู้บรรยาย:** 🙂

...ใช่ไหมล่ะ?

> **ผู้บรรยาย:** 🙂

ก็นะ ตอนนี้เรากำลังเขียนโค้ด `unsafe` กันอยู่นี่นา คอมไพเลอร์เลยช่วยจับข้อผิดพลาดให้เราได้ไม่เต็มที่เหมือนแต่ก่อน เป็นไปได้ว่าเทสต์พวกนั้นมันอาจจะแค่*บังเอิญ*ทำงานได้เฉยๆ แต่เบื้องหลังอาจกำลังทำอะไรที่คาดเดาไม่ได้... อะไรที่เข้าข่าย Undefined Behaviour อยู่ก็ได้

แต่เราจะทำอะไรได้ล่ะ? ในเมื่อเรางัดหน้าต่างแล้วแอบโดดหนีออกจากห้องเรียนของ rustc มาแล้ว ตอนนี้ไม่มีใครช่วยเราได้อีกแล้ว

...เดี๋ยวนะ คนท่าทางลับๆ ล่อๆ ตรงตรอกซอยนั่นคือใครกันน่ะ?

*"เฮ้ หนูน้อย... สนใจมาลอง interpret โค้ด Rust ดูสักหน่อยไหม?"*

หะ... ไม่อะ! ทำไมต้อง...

*"ของมันแรงนะพวก มันเช็กได้เลยว่าการทำงานจริงๆ แบบ dynamic ตอนรันโปรแกรมเนี่ย มันสอดคล้องกับกฎ memory model ของ Rust หรือเปล่า รับรองว่าตาค้าง..."*

อะไรของลุงเนี่ย?

*"มันตรวจได้นะว่าเอ็งแอบทำ Undefined Behaviour หรือเปล่า"*

เอ่อ... ลองตัว interpreter ดูสักครั้งก็น่าจะไม่เสียหายมั้ง

*"เอ็งติดตั้ง rustup ไว้แล้วใช่ไหมล่ะ?"*

แหงอยู่แล้ว! นั่นมันคือ*เครื่องมือหลัก*สำหรับจัดการ Rust toolchain ให้ทันสมัยเลยนะ!

```text
> rustup +nightly-2022-01-21 component add miri

info: syncing channel updates for 'nightly-2022-01-21-x86_64-pc-windows-msvc'
info: latest update on 2022-01-21, rust version 1.60.0-nightly (777bb86bc 2022-01-20)
info: downloading component 'cargo'
info: downloading component 'clippy'
info: downloading component 'rust-docs'
info: downloading component 'rust-std'
info: downloading component 'rustc'
info: downloading component 'rustfmt'
info: installing component 'cargo'
info: installing component 'clippy'
info: installing component 'rust-docs'
info: installing component 'rust-std'
info: installing component 'rustc'
info: installing component 'rustfmt'
info: downloading component 'miri'
info: installing component 'miri'
```

นี่คุณเพิ่งลงบ้าอะไรบนคอมผมเนี่ย!?

*"ของดีไงล่ะน้อง"*

> **ผู้บรรยาย:** มีเรื่องพิลึกๆ เกี่ยวกับเวอร์ชันของ toolchain นิดหน่อย:
>
> เครื่องมือที่เรากำลังติดตั้งอยู่นี้ มีชื่อว่า `miri` ซึ่งตัวมันทำงานแนบแน่นกับไส้ในของ rustc มากๆ ดังนั้น มันจึงเปิดให้ใช้งานได้เฉพาะบน nightly toolchain เท่านั้น
>
> การระบุ `+nightly-2022-01-21` เป็นการบอก `rustup` ว่าเราต้องการติดตั้ง miri ร่วมกับชุด rust nightly toolchain ของวันที่นั้นๆ สาเหตุที่ผมต้องระบุวันที่เจาะจงลงไป เป็นเพราะบางครั้ง miri ก็ตาม rustc ไม่ทันจนบิลด์ไม่ผ่านไปหลายวัน (several nightlies) ซึ่ง rustup จะดาวน์โหลด toolchain วันที่ที่เราระบุหลังเครื่องหมาย `+` มาให้โดยอัตโนมัติหากในเครื่องเรายังไม่มี
>
> วันที่ 2022-01-21 เป็น nightly รุ่นหนึ่งที่ผมรู้แน่นอนว่า miri ใช้งานได้ ซึ่งคุณสามารถเข้าไปเช็กสถานะได้ที่ [หน้าประวัติคอมโพเนนต์นี้](https://rust-lang.github.io/rustup-components-history/) แต่ถ้าคุณรู้สึกว่าดวงกำลังดี จะใช้ `+nightly` เพียวๆ เลยก็ได้นะ
>
> และเมื่อใดก็ตามที่เราเรียก miri ผ่าน `cargo miri` เราก็ต้องใช้ไวยากรณ์ `+` แบบนี้เพื่อเจาะจง toolchain ตัวที่เราติดตั้ง miri เอาไว้ด้วยเสมอ แต่ถ้าไม่อยากพิมพ์ซ้ำทุกครั้ง คุณสามารถใช้คำสั่ง [`rustup override set`](https://rust-lang.github.io/rustup/overrides.html) ช่วยได้

```text
> cargo +nightly-2022-01-21 miri test

I will run `"cargo.exe" "install" "xargo"` to install
a recent enough xargo. Proceed? [Y/n]
```

เฮ้ยยยย แล้ว XARGO มันคืออะไรอีกเนี่ย!?

*"เอาน่า ไม่ต้องคิดมาก ชิลๆ"*

```text
> y

    Updating crates.io index
  Installing xargo v0.3.24
...
    Finished release [optimized] target(s) in 10.65s
  Installing C:\Users\ninte\.cargo\bin\xargo-check.exe
  Installing C:\Users\ninte\.cargo\bin\xargo.exe
   Installed package `xargo v0.3.24` (executables `xargo-check.exe`, `xargo.exe`)

I will run `"rustup" "component" "add" "rust-src"` to install 
the `rust-src` component for the selected toolchain. Proceed? [Y/n]
```

หืมมมมมมม!?

*"ใครล่ะจะไม่ชอบมีซอร์สโค้ดของ Rust เก็บไว้บนเครื่องตัวเอง?"*

```text
> y

info: downloading component 'rust-src'
info: installing component 'rust-src'
```

*"อ่าาา เรียบร้อยแล้ว พร้อมลุย! นี่แหละของเด็ด"*

```text
   Compiling lists v0.1.0 (C:\Users\ninte\dev\tmp\lists)
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests (lists-5cc11d9ee5c3e924.exe)

error: Undefined Behavior: trying to reborrow for Unique at alloc84055, 
       but parent tag <209678> does not have an appropriate item in 
       the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |
    = help: this indicates a potential bug in the program: 
      it performed an invalid operation, but the rules it 
      violated are still experimental
    = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information

    = note: inside `std::option::Option::<std::boxed::Box<fifth::Node<i32>>>::map::<i32, [closure@src\fifth.rs:31:30: 40:10]>` at \lib\rustlib\src\rust\library\core\src\option.rs:846:18

note: inside `fifth::List::<i32>::pop` at src\fifth.rs:31:9
   --> src\fifth.rs:31:9
    |
31  | /         self.head.take().map(|head| {
32  | |             let head = *head;
33  | |             self.head = head.next;
34  | |
...   |
39  | |             head.elem
40  | |         })
    | |__________^
note: inside `fifth::test::basics` at src\fifth.rs:74:20
   --> src\fifth.rs:74:20
    |
74  |         assert_eq!(list.pop(), Some(1));
    |                    ^^^^^^^^^^
note: inside closure at src\fifth.rs:62:5
   --> src\fifth.rs:62:5
    |
61  |       #[test]
    |       ------- in this procedural macro expansion
62  | /     fn basics() {
63  | |         let mut list = List::new();
64  | |
65  | |         // Check empty list behaves right
...   |
96  | |         assert_eq!(list.pop(), None);
97  | |     }
    | |_____^
 ...
error: aborting due to previous error
```

โอ้โห... เป็น error ที่ยาวเหยียดอลังการงานสร้างอะไรเบอร์นั้น

*"เห็นไหมล่ะ ความบรรลัยชิ้นเอกแบบนี้ ใครเห็นก็ต้องร้องว้าว"*

เอ่อ... ขอบคุณ?

*"อะ เอาขวด estradiol นี่ไปด้วยนะ เดี๋ยวอีกเดี๋ยวเอ็งได้ใช้แน่ๆ"*

เดี๋ยวสิ ทำไมอะ?

*"อีกเดี๋ยวเอ็งก็ต้องมานั่งกุมขมับคิดเรื่อง memory models แล้ว เชื่อข้าเถอะ"*

> **ผู้บรรยาย:** ชายลึกลับคนนั้นก็แปลงร่างกลายเป็นสุนัขจิ้งจอก แล้วมุดรูบนกำแพงเผ่นแน่บหายไป ทิ้งให้ผู้เขียนยืนเหม่อจ้องไปในความเวิ้งว้างอยู่นานหลายนาที พลางพยายามประมวลผลว่ามันเพิ่งเกิดเรื่องบ้าอะไรขึ้นกับชีวิตกันแน่


-------

จิ้งจอกลึกลับในตรอกซอยคนนั้นพูดถูกมากกว่าแค่เรื่องเพศของผมเสียอีก: miri นี่มันคือของดีตัวตึงของแท้จริงๆ!

เอาล่ะ แล้วตกลง [miri](https://github.com/rust-lang/miri) มัน*คือ*อะไรกันแน่?

> เป็นตัว interpreter ขั้นทดลองสำหรับแปลและประมวลผลตัวแทนโค้ดระดับกลาง (mid-level intermediate representation หรือ MIR) ของ Rust ตัวมันสามารถรันไฟล์ไบนารีและชุดการทดสอบของโปรเจกต์ Cargo เพื่อตรวจจับพฤติกรรม Undefined Behavior รูปแบบต่างๆ ได้ เช่น:
>
> * การเข้าถึงหน่วยความจำเกินขอบเขต (Out-of-bounds) และ use-after-free
> * การใช้งานข้อมูลที่ยังไม่ได้ถูกกำหนดค่าเริ่มต้น (uninitialized data) อย่างไม่ถูกต้อง
> * การละเมิดเงื่อนไขเบื้องต้นของ intrinsic functions (เช่น การหลุดเข้าไปถึงโค้ด unreachable_unchecked หรือการเรียก copy_nonoverlapping บนช่วงหน่วยความจำที่ทับซ้อนกัน เป็นต้น)
> * การอ้างอิงหรือเข้าถึงหน่วยความจำที่ alignment ไม่ถูกต้อง
> * การละเมิด invariants พื้นฐานของ type (เช่น ค่า bool ที่ไม่ใช่ 0 หรือ 1 หรือค่า discriminant ของ enum ที่ไม่ถูกต้อง)
> * อยู่ในขั้นทดลอง: การละเมิดกฎ Stacked Borrows ที่ควบคุมเรื่อง aliasing ของ reference types
> * อยู่ในขั้นทดลอง: Data races (แต่ยังไม่รวมถึงผลกระทบจาก weak memory)
>
> ยิ่งไปกว่านั้น Miri ยังสามารถแจ้งเตือนเรื่อง memory leak ได้อีกด้วย: หากมีหน่วยความจำที่ถูกจองค้างไว้จนจบโปรแกรมโดยไม่ถูก free และหน่วยความจำก้อนนั้นไม่สามารถเข้าถึงได้จากตัวแปร global static ตัว Miri ก็จะฟ้อง error ทันที
>
> ...
>
> อย่างไรก็ตาม พึงระลึกไว้ว่า Miri ไม่สามารถตรวจจับ Undefined Behavior ได้ครอบคลุมทุกกรณี และไม่สามารถรันโปรแกรมได้ทุกประเภท

สรุปสั้นๆ (TL;DR): มันคือตัวที่จะคอย interpret รันโปรแกรมของคุณทีละสเต็ป แล้วคอยจับตาดูว่าคุณแอบแหกกฎ*ตอนรันไทม์ (at runtime)* จนเกิด Undefined Behaviour ขึ้นมาหรือเปล่า ซึ่งจำเป็นมาก เพราะ Undefined Behaviour *โดยส่วนใหญ่*มักจะระเบิดขึ้นตอนรันไทม์เสมอ ถ้ามันเป็นปัญหาที่ดักจับได้ตั้งแต่ตอนคอมไพล์ คอมไพเลอร์ก็คงสั่ง compile error ฟ้องเราไปแต่แรกแล้ว!

ถ้าคุณคุ้นเคยกับเครื่องมือในภาษา C อย่าง ubsan และ tsan มาก่อน: มันก็คล้ายๆ กันนั่นแหละ แต่จับมารวมร่างเข้าด้วยกันและเข้มงวดยิ่งกว่าเดิมหลายเท่าตัว

-------

ตอนนี้ Miri ไปยืนเกาะอยู่นอกหน้าต่างห้องเรียนพร้อมถือมีดเล่มหนึ่ง... มีดแห่งการเรียนรู้

ถ้าเราอยากให้ Miri ช่วยตรวจโค้ดของเราเมื่อไหร่ เราสามารถสั่งให้มันช่วย interpret เทสต์สูทของเราได้ด้วยคำสั่ง:

```text
> cargo +nightly-2022-01-21 miri test
```

คราวนี้ มาดูสิ่งที่มันสลักไว้บนโต๊ะเรียนของเรากันชัดๆ อีกรอบ:

```text
error: Undefined Behavior: trying to reborrow for Unique at alloc84055, but parent tag <209678> does not have an appropriate item in the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |

    = help: this indicates a potential bug in the program: it 
      performed an invalid operation, but the rules it 
      violated are still experimental
    
    = help: see 
      https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information
```

เอาล่ะ เราเห็นแล้วว่าเราทำโค้ดพังจริงๆ แต่ข้อความ error นั้นช่างชวนสับสนเหลือเกิน เจ้า "borrow stack" ที่ว่านี้มันคืออะไรกันแน่?

เดี๋ยวเราจะมาไขปริศนานี้ในหัวข้อถัดไปกันครับ
