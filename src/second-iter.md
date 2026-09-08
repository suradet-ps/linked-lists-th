# Iter

เอาล่ะ ทีนี้มาลอง implement `Iter` กันดูบ้าง คราวนี้เราจะพึ่งพาฟังก์ชันของ `List` มารองรับให้ไม่ได้อีกแล้ว เราจำเป็นต้องลงมือเขียนลอจิกทั้งหมดด้วยตัวเอง โดยลอจิกพื้นฐานที่เราต้องการก็คือ การถือตัวชี้ไปยังโหนดปัจจุบันเพื่อรอส่งค่าคืนในรอบถัดไป และเนื่องจากโหนดนั้นอาจจะไม่มีอยู่จริงก็ได้ (เช่น ลิสต์ว่างเปล่า หรือเราวนลูปอ่านข้อมูลจนจบลิสต์แล้ว) เรเฟอเรนซ์ตัวนั้นจึงจำเป็นต้องถูกห่อหุ้มไว้ใน `Option` เสมอ และเมื่อเราส่งข้อมูลของโหนดปัจจุบันออกไปแล้ว เราก็จะต้องขยับตัวชี้ไปยังโหนด `next` ถัดไป

งั้นมาลองเขียนกันเลย:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

โอ้พระเจ้า... ไลฟ์ไทม์ (Lifetimes)! ผมเคยได้ยินกิตติศัพท์ของมันมานักต่อนัก และได้ยินมาหนาหูว่ามันคือฝันร้ายของคนเขียน Rust ชัด ๆ!

งั้นมาลองอะไรใหม่ ๆ ดูหน่อย: เห็นรหัส `error[E0106]` นั่นไหมครับ? นั่นคือรหัส error ของคอมไพเลอร์ ซึ่งเราสามารถสั่งให้ `rustc` ช่วยอธิบายที่มาที่ไปของมันได้ด้วยแฟล็ก `--explain`:

```text
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...

```

เอ่อ... อ่านแล้วก็ไม่ได้ช่วยให้เข้าใจกระจ่างขึ้นเท่าไหร่เลยแฮะ (เอกสารคู่มือพวกนี้มักจะทึกทักเอาเองว่าเราเข้าใจ Rust ดีกว่าความเป็นจริงเสมอ) แต่เท่าที่ดูคร่าว ๆ เหมือนว่าเราจะต้องเติม `'a` พวกนั้นเข้าไปใน struct ของเราใช่ไหม? งั้นมาลองใส่ดู:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

เอาล่ะ ผมเริ่มจับทางรูปแบบของมันได้ละ... งั้นมาประเคนใส่เจ้าเครื่องหมาย `'a` นี้เข้าไปในทุก ๆ จุดที่เราทำได้เลยดีกว่า!:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```text
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

คุณพระช่วย... เราทำระบบไวยากรณ์ของ Rust พังยับเยินไปเรียบร้อยแล้ว

บางที เราคงต้องหยุดพักแล้วมาทำความเข้าใจกันจริง ๆ จัง ๆ สักทีว่าไอ้เจ้าไลฟ์ไทม์พวกนี้มันมีความหมายว่าอะไรกันแน่

ไลฟ์ไทม์ (Lifetimes) เป็นเรื่องที่ทำให้คนจำนวนมากรู้สึกขยาดและหวาดกลัว เพราะมันเป็นแนวคิดใหม่ที่เข้ามาสั่นคลอนสิ่งที่เราคุ้นเคยและรักมาตั้งแต่ยุคบุกเบิกของการเขียนโปรแกรม ที่ผ่านมาเราพยายามหลบเลี่ยงไม่ยอมพูดถึงมัน ทั้ง ๆ ที่จริง ๆ แล้วมันแอบแฝงและพัวพันอยู่ในทุกอณูของโค้ดโปรแกรมเรามาโดยตลอด

ในภาษาที่มีระบบ Garbage Collector (GC) เราไม่จำเป็นต้องมีไลฟ์ไทม์เลยแม้แต่น้อย เพราะตัว GC จะคอยการันตีให้ว่าข้อมูลทุกชิ้นจะมีชีวิตอยู่ตราบเท่าที่ยังมีคนต้องการใช้งานมันอยู่ แต่ข้อมูลส่วนใหญ่ในภาษา Rust ถูกจัดการและควบคุมอายุขัย *ด้วยระบบเมโมรีแบบเจาะจง* ดังนั้น ข้อมูลพวกนี้จึงต้องการทางออกอื่น ภาษาอย่าง C และ C++ เป็นตัวอย่างชั้นดีที่แสดงให้เห็นว่า จะเกิดหายนะอะไรขึ้นบ้างถ้าคุณปล่อยให้ใครก็ได้ใช้ตัวชี้ชี้ไปยังข้อมูลบนสแต็กแบบตามใจชอบ: ความพังพินาศและช่องโหว่ด้านความปลอดภัยจะลุกลามไปทั่วทุกหนทุกแห่ง ซึ่งมักจะสรุปออกมาเป็นบั๊กยอดฮิตได้ 2 รูปแบบ:

* การถือตัวชี้ที่ชี้ไปยังข้อมูลที่หลุดออกจากสโคปไปแล้ว (dangling pointer)
* การถือตัวชี้ที่ชี้ไปยังข้อมูลที่ถูกคนอื่นแอบแก้ไขค่าไปแล้ว (aliasing mutation)

ระบบไลฟ์ไทม์เกิดมาเพื่อแก้ปัญหาใหญ่ทั้งสองข้อนี้ และใน 99% ของเวลาทั้งหมด มันทำงานอยู่เบื้องหลังอย่างเงียบเชียบและโปร่งใสจนคุณแทบไม่รู้สึกตัว

แล้วตกลงว่า ไลฟ์ไทม์คืออะไรกันแน่?

พูดให้เข้าใจง่ายที่สุด ไลฟ์ไทม์ก็คือ "ชื่อเรียกของช่วงสโคปในโค้ด" สักแห่งหนึ่งในโปรแกรมนั่นแหละครับ แค่นั้นจริง ๆ! เมื่อเรเฟอเรนซ์ตัวหนึ่งถูกผูกติดอยู่กับไลฟ์ไทม์ มันจะมีความหมายว่าเรเฟอเรนซ์ตัวนั้นจะต้องยังคงชี้ข้อมูลที่ถูกต้องปลอดภัยอยู่ตลอด *ทั้งช่วงสโคป* นั้น โครงสร้างหรือฟังก์ชันต่าง ๆ จะมีข้อกำหนดเรื่องอายุขัยว่าเรเฟอเรนซ์ต้องถูกต้องอยู่นานแค่ไหน ระบบไลฟ์ไทม์ทั้งหมดจริง ๆ แล้วเป็นเพียงแค่ระบบแก้สมการเงื่อนไข (constraint solver) ที่พยายามจำกัดขอบเขตของเรเฟอเรนซ์ทุกตัวให้เล็กที่สุดเท่าที่จะทำได้ และถ้ามันสามารถหาช่วงไลฟ์ไทม์ที่สอดคล้องกับเงื่อนไขทั้งหมดได้ลงตัว โปรแกรมของคุณก็จะคอมไพล์ผ่านฉลุย! แต่ถ้าหาไม่ได้ คอมไพเลอร์ก็จะฟ้อง error ออกมาเตือนคุณว่า มีข้อมูลบางตัวอายุสั้นเกินไป ไม่สามารถอยู่ยาวได้ตามที่ต้องการ

ภายในบล็อกของฟังก์ชัน โดยทั่วไปคุณไม่ต้องไปยุ่งเกี่ยวกับไลฟ์ไทม์เลย (และจริง ๆ คุณก็ไม่อยากไปยุ่งกับมันอยู่แล้ว) เพราะคอมไพเลอร์มีข้อมูลโค้ดทั้งหมดอยู่ในมือ จึงสามารถอนุมาน (infer) เงื่อนไขต่าง ๆ เพื่อหาช่วงไลฟ์ไทม์ที่สั้นที่สุดให้เองได้อัตโนมัติ ทว่า ในระดับการประกาศชนิดข้อมูลและ API ภายนอก คอมไพเลอร์ *ไม่ได้* มีข้อมูลทั้งหมด มันจึงต้องการให้คุณช่วยบอกความสัมพันธ์ระหว่างไลฟ์ไทม์ของข้อมูลแต่ละตัวให้มันรู้ เพื่อที่มันจะได้เข้าใจลอจิกของคุณได้อย่างถูกต้อง

ในทางทฤษฎีแล้ว คอมไพเลอร์จะแอบคำนวณไลฟ์ไทม์ข้ามฟังก์ชันไปเลยก็ได้ แต่ถ้าทำแบบนั้น ระบบ borrow checker จะต้องวิเคราะห์โค้ดโปรแกรมทั้งก้อนข้ามไฟล์ข้ามโมดูล ซึ่งจะทำให้เกิด error กระจายมั่วไปหมดจนหาสาเหตุไม่เจอ การที่ Rust บังคับให้เราระบุไลฟ์ไทม์ที่หัวฟังก์ชัน ทำให้การตรวจสอบการยืมสามารถทำจบได้ภายในแต่ละฟังก์ชันอย่างเป็นเอกเทศ และเวลาเกิด error ขึ้นมา มันก็จะชี้เป้าได้อย่างแม่นยำตรงจุดในฟังก์ชันนั้น ๆ

แต่เดี๋ยวก่อนนะ ที่ผ่านมาเราก็เคยเขียนเรเฟอเรนซ์ในพารามิเตอร์ของฟังก์ชันมาตั้งหลายรอบแล้วนี่ ทำไมตอนนั้นมันถึงไม่พังล่ะ? นั่นเป็นเพราะว่ามีบางกรณีที่เจอบ่อยเสียจน Rust แอบเดาและเลือกไลฟ์ไทม์ที่เหมาะสมให้เราโดยอัตโนมัติ ซึ่งกลไกนี้เราเรียกว่า *การละเว้นไลฟ์ไทม์ (lifetime elision)*

โดยมีกฎเกณฑ์หลัก ๆ ดังนี้:

```rust ,ignore
// Only one reference in input, so the output must be derived from that input
fn foo(&A) -> &B; // sugar for:
fn foo<'a>(&'a A) -> &'a B;

// Many inputs, assume they're all independent
fn foo(&A, &B, &C); // sugar for:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// Methods, assume all output lifetimes are derived from `self`
fn foo(&self, &B, &C) -> &D; // sugar for:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

แล้วการเขียนว่า `fn foo<'a>(&'a A) -> &'a B` มัน *หมายความว่าอะไร* กันแน่? ในทางปฏิบัติ มันหมายความว่า อินพุตตัวนี้จะต้องมีอายุขัยอยู่นานเท่ากับเอาต์พุตที่ส่งออกไป ดังนั้น ถ้าคุณเก็บเอาต์พุตนั้นไว้นานแค่ไหน ช่วงสโคปที่อินพุตจะต้องถูกต้องก็จะถูกยืดขยายตามออกไปนานเท่านั้น และทันทีที่คุณเลิกใช้เอาต์พุต คอมไพเลอร์ก็จะรู้ได้ทันทีว่า ตอนนี้อินพุตตัวนั้นปลอดภัยพอที่จะหมดอายุขัยลงได้แล้ว

ด้วยระบบอันชาญฉลาดนี้ Rust จึงสามารถการันตีได้ 100% ว่าจะไม่มีการนำข้อมูลที่ถูกคืนหน่วยความจำไปแล้วกลับมาใช้อีก (no use-after-free) และจะไม่มีใครแอบแก้ไขข้อมูลในขณะที่ยังมีเรเฟอเรนซ์ค้างคาอยู่ ทั้งหมดนี้เกิดขึ้นได้เพียงแค่คอมไพเลอร์เช็กให้แน่ใจว่าเงื่อนไขอายุขัยทั้งหมดลงล็อกกันพอดี!

เอาล่ะ ทีนี้กลับมาดูที่ `Iter` ของเรากันต่อ

งั้นเราลองถอยหลังกลับไปตั้งหลักที่สถานะเดิมที่ยังไม่มีไลฟ์ไทม์กันก่อน:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

สิ่งที่เราต้องทำก็คือการเพิ่มไลฟ์ไทม์เข้าไปเฉพาะที่ลายเซ็นของชนิดข้อมูลและฟังก์ชันเท่านั้น (ห้ามใส่ในตัวแปรข้างใน!):

```rust ,ignore
// Iter is generic over *some* lifetime, it doesn't care
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// No lifetime here, List doesn't have any associated lifetimes
impl<T> List<T> {
    // We declare a fresh lifetime here for the *exact* borrow that
    // creates the iter. Now &self needs to be valid as long as the
    // Iter is around.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// We *do* have a lifetime here, because Iter has one that we need to define
impl<'a, T> Iterator for Iter<'a, T> {
    // Need it here too, this is a type declaration
    type Item = &'a T;

    // None of this needs to change, handled by the above.
    // Self continues to be incredibly hype and amazing
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

เอาล่ะ ผมคิดว่าคราวนี้เรามาถูกทางแน่ ๆ ลองคอมไพล์ดูซิ:

```text
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

(╯°□°)╯︵ ┻━┻

โอเค... เราแก้ error เรื่องไลฟ์ไทม์จบไปแล้ว แต่ตอนนี้เราเจอปัญหาเรื่องชนิดข้อมูลไม่ตรงกันแทน

สิ่งที่เราต้องการเก็บคือ `&Node` แต่สิ่งที่เราได้มาดันเป็น `&Box<Node>` ซะงั้น! โอเค เรื่องนี้แก้ง่ายนิดเดียว เราก็แค่ต้องดีเรเฟอเรนซ์ (dereference) ตัว Box ออกมาก่อนที่จะหยิบเรเฟอเรนซ์ไปใช้งาน:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

(ﾉಥ益ಥ）ﾉ﻿ ┻━┻

เราดันลืมใส่ `as_ref`! ทำให้เราเผลอย้าย (move) ตัว Box ทั้งก้อนเข้าไปใน `map` ซึ่งนั่นหมายความว่าตัว Box จะถูกสั่ง drop ทิ้งทันทีที่จบฟังก์ชัน ส่งผลให้เรเฟอเรนซ์ของเรากลายเป็นตัวชี้เคว้งคว้าง (dangling reference) ทันที!:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

```

😭

กลายเป็นว่า `as_ref` ดันเพิ่มการชี้ทางอ้อม (indirection) ซ้อนเข้ามาอีกหนึ่งชั้นที่เราต้องแกะออก:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

```text
cargo build

```

🎉 🎉 🎉

เมธอด `as_deref` และ `as_deref_mut` ได้รับการบรรจุเข้าสู่สถานะเสถียรตั้งแต่ Rust 1.40 เป็นต้นมา ซึ่งในยุคก่อนหน้านั้น คุณจะต้องมานั่งพิมพ์ว่า `map(|node| &**node)` หรือ `map(|node| &mut**node)` คุณอาจจะกำลังคิดในใจว่า "โอ้โห ไอ้การเขียน `&**` ซ้อนกันเนี่ยมันดูพิกลพิการสิ้นดี" ซึ่งคุณคิดถูกแล้วครับ! แต่เฉกเช่นเดียวกับไวน์ชั้นเลิศ ภาษา Rust ยิ่งนานวันก็ยิ่งพัฒนาดีขึ้นเรื่อย ๆ จนเราไม่ต้องมานั่งเขียนอะไรทุลักทุเลแบบนั้นอีกต่อไปแล้ว โดยปกติแล้ว Rust เก่งมากในการช่วยแปลงชนิดข้อมูลพวกนี้ให้เราโดยอัตโนมัติ ผ่านกระบวนการที่เรียกว่า *Deref coercion (การแปลงชนิดข้อมูลผ่าน Deref โดยอัตโนมัติ)* ซึ่งโดยหลักการแล้ว คอมไพเลอร์สามารถแอบแทรกเครื่องหมาย `*` เข้าไปตามจุดต่าง ๆ ในโค้ดของคุณ เพื่อช่วยให้ชนิดข้อมูลตรงกันและคอมไพล์ผ่านได้ มันกล้าทำแบบนี้ได้ก็เพราะ Rust มี borrow checker คอยรับประกันความปลอดภัยของตัวชี้อยู่เบื้องหลังเสมอ!

แต่ในกรณีนี้ การใช้โคลเชอร์ร่วมกับข้อเท็จจริงที่ว่าเรามี `Option<&T>` แทนที่จะเป็น `&T` เพียว ๆ มันดันซับซ้อนเกินไปหน่อยจนคอมไพเลอร์คิดเชื่อมโยงเองไม่ไหว เราจึงต้องยื่นมือเข้าไปช่วยสะกิดบอกมันตรง ๆ แต่โชคดีที่จากประสบการณ์ของผม เหตุการณ์แบบนี้เกิดขึ้นไม่บ่อยเท่าไหร่ครับ

และเพื่อความสมบูรณ์แบบของเนื้อหา จริง ๆ แล้วเรา *สามารถ* ให้คำใบ้อีกแบบหนึ่งกับคอมไพเลอร์ได้ด้วยการใช้ไวยากรณ์ปลาติดเทอร์โบอย่าง *turbofish*:

```rust ,ignore
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

เห็นไหมครับว่า จริง ๆ แล้ว `map` เป็นฟังก์ชันแบบเจเนอริก:

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
```

และไวยากรณ์ turbofish อย่าง `::<...>` เปิดโอกาสให้เราบอกคอมไพเลอร์ได้อย่างชัดเจนว่าเราต้องการระบุให้เจเนอริกแต่ละตัวมีชนิดข้อมูลเป็นอะไร ซึ่งในที่นี้ `::<&Node<T>, _>` กำลังบอกว่า "ช่วยทำให้มันคืนค่าเป็น `&Node<T>` ออกมานะ ส่วนชนิดข้อมูลอีกตัวหนึ่ง ฉันไม่รู้และไม่สนใจ ปล่อยให้เธอเดาเอาเองได้เลย"

และสิ่งนี้ก็ช่วยให้คอมไพเลอร์รู้ทันทีว่า เรเฟอเรนซ์ `&node` ควรจะได้รับการทำ Deref coercion อัตโนมัติ เราจึงไม่ต้องมานั่งพิมพ์เครื่องหมาย `*` ซ้อนกันหลาย ๆ อันด้วยตัวเองอีกต่อไป!

แต่ในกรณีนี้ ผมไม่คิดว่ามันช่วยให้โค้ดดูดีขึ้นเท่าไหร่หรอกนะ นี่เป็นแค่ข้ออ้างแบบเนียน ๆ ที่ผมหาเรื่องเอามาโชว์ของทั้งเรื่อง Deref coercion และไวยากรณ์ Turbofish สุดเท่ต่างหาก 😅

งั้นมาเขียนเทสต์เพื่อให้มั่นใจว่าโค้ดของเราทำงานได้จริง ไม่ได้หลอกกันเล่น:

```rust ,ignore
#[test]
fn iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

แจ๋วไปเลย!

และสุดท้ายนี้ ควรระบุไว้ด้วยว่าจริง ๆ แล้วเรา *สามารถ* ละเว้นไลฟ์ไทม์ (lifetime elision) ตรงจุดนี้ได้ด้วยนะ:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}
```

สามารถเขียนย่อให้กระชับขึ้นเป็น:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```

เย่! ไม่ต้องเห็นเครื่องหมายไลฟ์ไทม์ให้รกสายตาแล้ว!

หรือถ้าคุณรู้สึกไม่ค่อยสบายใจที่จะ "แอบซ่อน" ความจริงที่ว่า struct ตัวนี้เก็บเรเฟอเรนซ์ที่มีไลฟ์ไทม์อยู่ข้างใน คุณก็สามารถใช้ไวยากรณ์ "ระบุว่ามีการละเว้นไลฟ์ไทม์อย่างชัดเจน" ของ Rust 2018 ได้ด้วยเครื่องหมาย `'_`:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}
```