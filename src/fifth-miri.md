# Miri

*หัวเราะอย่างประหม่า* เรื่อง unsafe นี่มันง่ายจังเลย ไม่รู้ว่าทำไมทุกคนถึงบอกว่ามันยาก โปรแกรมของเราทำงานสมบูรณ์แบบ

> **ผู้บรรยาย:** 🙂

...จริงไหม?

> **ผู้บรรยาย:** 🙂

ตอนนี้เราเขียน `unsafe` โค้ดแล้ว ดังนั้นคอมไพเลอร์จึงช่วยเราจับข้อผิดพลาดได้ไม่ดีเท่าที่ควร อาจเป็นไปได้ว่าเทสต์*บังเอิญ*ผ่าน แต่จริงๆ แล้วทำสิ่งที่ไม่แน่นอน พฤติกรรมที่ไม่กำหนด (Undefined Behaviour) บางอย่าง

แต่เราจะทำอะไรได้ล่ะ? เราแงะหน้าต่างออกและหลบหนีออกจากห้องเรียนของ rustc แล้ว ไม่มีใครช่วยเราได้แล้ว

...เดี๋ยวก่อน คนหน้าตาแปลกๆ ในซอยนั่นคือใคร?

*"เฮ้เด็ก อยากลองตีความโค้ด Rust สักหน่อยไหม?"*

ห- ไม่ ทำไม,

*"มันเจ๋งมากเลยนะ มันสามารถตรวจสอบว่าการทำงานจริงๆ ของโปรแกรมคุณสอดคล้องกับโมเดลความจำของ Rust ได้ ทำให้หัวคุณระเบิดเลยล่ะ..."*

อะไรนะ?

*"มันตรวจว่าคุณทำพฤติกรรมที่ไม่กำหนด (Undefined Behaviour) หรือเปล่า"*

ก็...ลองตีความดูสักครั้งก็ได้มั้ง

*"คุณติดตั้ง rustup แล้วใช่ไหม?"*

แน่นอนสิ นี่เป็น*เครื่องมือ*สำหรับ Rust toolchain ที่ทันสมัย!

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

คุณเพิ่งติดตั้งอะไรบนคอมของผม!?

*"ของดีน่ะ"*

> **ผู้บรรยาย:** มีบางอย่างเกี่ยวกับเวอร์ชัน toolchain ที่ดูแปลกๆ:
>
> เครื่องมือที่เราติดตั้ง `miri` ทำงานใกล้ชิดกับ rustc ภายใน
> ดังนั้นจึงใช้ได้เฉพาะกับ nightly toolchain เท่านั้น
>
> `+nightly-2022-01-21` บอก `rustup` ว่าเราต้องการติดตั้ง miri ด้วย Rust
> nightly toolchain ของวันที่นั้น ผมให้วันที่เฉพาะเพราะบางครั้ง
> miri ล้าหลังและไม่สามารถคอมไพล์ได้บน nightly บางตัว rustup จะ
> ดาวน์โหลด toolchain ที่เรากำหนดด้วย `+` โดยอัตโนมัติหากเรายัง
> ไม่ได้ติดตั้ง
>
> 2022-01-21 เป็น nightly ที่ผมรู้ว่ามีการสนับสนุน miri ซึ่งคุณสามารถตรวจสอบ
> [ในหน้าสถานะนี้](https://rust-lang.github.io/rustup-components-history/)
> คุณสามารถใช้ `+nightly` ได้เลยถ้าอยากรู้สึกโชคดี
>
> เมื่อใดก็ตามที่เราเรียก miri ผ่าน `cargo miri` เราจะใช้ไวยากรณ์ `+` นี้เพื่อ
> กำหนด toolchain ที่เราติดตั้ง miri ไว้ ถ้าคุณไม่อยากต้อง
> กำหนดทุกครั้ง คุณสามารถใช้ [`rustup override set`](https://rust-lang.github.io/rustup/overrides.html)

```text
> cargo +nightly-2022-01-21 miri test

I will run `"cargo.exe" "install" "xargo"` to install
a recent enough xargo. Proceed? [Y/n]
```

อ้าว XARGO คืออะไร!?

*"ไม่เป็นไร ไม่ต้องห่วง"*

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

อ้าว???

*"ใครบ้างล่ะที่ไม่อยากได้สำเนาซอร์สโค้ดของ Rust?"*

```text
> y

info: downloading component 'rust-src'
info: installing component 'rust-src'
```

*"อ๋อย ตอนนี้พร้อมแล้วล่ะ มาถึงส่วนที่ดีกัน"*

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

ว้าว. นี่เป็นข้อผิดพลาดที่อลังการมาก

*"ใช่ มองดูมันสิ คุณก็ชอบที่จะเห็นมัน"*

ขอบคุณ?

*"นี่ เอาขวด estradiol ไปด้วยนะ คุณจะต้องใช้มันทีหลัง"*

เดี๋ยวทำไม?

*"คุณกำลังจะคิดเรื่องโมเดลความจำ เชื่อเถอะ"*

> **ผู้บรรยาย:** คนลึกลับคนนั้นก็แปลงร่างเป็นสุนัขจิ้งจอกและวิ่งผ่านรูในกำแพงไป ผู้เขียนจ้องมองไปที่ระยะกลางเป็นเวลาหลายนาทีขณะที่พยายามประมวลผลทุกสิ่งที่เพิ่งเกิดขึ้น


-------

สุนัขจิ้งจอกลึกลับในซอยนั้นพูดถูกมากกว่าแค่เพศของผม: miri เป็นของดีจริงๆ

แล้ว [miri](https://github.com/rust-lang/miri) *คือ*อะไร?

> An experimental interpreter for Rust's mid-level intermediate representation (MIR). It can run binaries and test suites of cargo projects and detect certain classes of undefined behavior, for example:
>
> * Out-of-bounds memory accesses and use-after-free
> * Invalid use of uninitialized data
> * Violation of intrinsic preconditions (an unreachable_unchecked being reached, calling copy_nonoverlapping with overlapping ranges, ...)
> * Not sufficiently aligned memory accesses and references
> * Violation of some basic type invariants (a bool that is not 0 or 1, for example, or an invalid enum discriminant)
> * Experimental: Violations of the Stacked Borrows rules governing aliasing for reference types
> * Experimental: Data races (but no weak memory effects)
>
> On top of that, Miri will also tell you about memory leaks: when there is memory still allocated at the end of the execution, and that memory is not reachable from a global static, Miri will raise an error.
>
> ...
>
> However, be aware that Miri will not catch all cases of undefined behavior in your program, and cannot run all programs

TL;DR: มันตีความโค้ดของคุณและสังเกตว่าคุณฝ่าฝืนกฎ*ในรันไทม์*หรือไม่ และทำพฤติกรรมที่ไม่กำหนด (Undefined Behaviour) สิ่งนี้จำเป็นเพราะว่าพฤติกรรมที่ไม่กำหนด (Undefined Behaviour) *โดยทั่วไป*เป็นสิ่งที่เกิดขึ้นในรันไทม์ ถ้าปัญหาสามารถหาได้ในเวลาคอมไพล์ คอมไพเลอร์ก็แค่ทำให้เป็นข้อผิดพลาดไปสิ!

ถ้าคุณคุ้นเคยกับเครื่องมืออย่าง ubsan และ tsan: มันก็คล้ายๆ กัน แต่รวมอยู่ด้วยกันและรุนแรงกว่า

-------

Miri ตอนนี้แขวนอยู่นอกหน้าต่างห้องเรียนพร้อมมีด มีดเพื่อการเรียนรู้

ถ้าเราต้องการให้ miri ตรวจสอบงานของเรา เราสามารถขอให้มันตีความชุดเทสต์ของเราด้วย

```text
> cargo +nightly-2022-01-21 miri test
```

ตอนนี้มาดูใกล้ๆ ว่ามันสลักอะไรไว้บนโต๊ะเรียนของเรา:

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

ผมเห็นว่าเราทำข้อผิดพลาด แต่ข้อความข้อผิดพลาดนี้สับสนมาก "สแต็กยืม (borrow stack)" คืออะไร?

เราจะพยายามหาคำตอบในส่วนถัดไป
