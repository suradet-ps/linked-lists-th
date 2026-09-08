# Iter

เอาล่ะ มาลอง implement Iter กัน ครั้งนี้เราจะไม่พึ่งพา List ที่ให้ฟีเจอร์ทั้งหมด
ที่เราต้องการ เราจะต้องเขียนเอง ตรรกะพื้นฐานที่เราต้องการคือถือตัวชี้ไปยัง
โหนดปัจจุบันที่เราต้องการคืนในครั้งถัดไป เพราะโหนดนั้นอาจไม่มีอยู่ (ลิสต์ว่างเปล่า
หรือเราวนซ้ำเสร็จแล้ว) เราต้องการให้เรเฟอเรนซ์นั้นเป็น Option
เมื่อเราคืนองค์ประกอบ เราต้องการไปยังโหนด `next` ของโหนดปัจจุบัน

เอาล่ะ ลองกัน:

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

โอ้พระเจ้า ไลฟ์ไทม์ ผมเคยได้ยินเรื่องพวกนี้ ผมได้ยินมาว่ามันเป็นฝันร้าย

มาลองอะไรใหม่: เห็น `error[E0106]` นั่นมั้ย? นั่นคือรหัสข้อผิดพลาดของคอมไพเลอร์
เราสามารถขอให้ rustc อธิบายมันได้ด้วย `--explain`:

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

เอ่อ... นั่นไม่ได้ทำให้ชัดเจนขึ้นเท่าไหร่ (เอกสารพวกนี้สมมติว่าเราเข้าใจ Rust
ดีกว่าที่เราเข้าใจตอนนี้) แต่ดูเหมือนว่าเราควรเพิ่ม `'a` พวกนั้นเข้าไปใน struct ของเรา?
มาลองกัน:

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

เอาล่ะ ผมเริ่มเห็นรูปแบบแล้ว... มาเพิ่มเจ้าพวกนี้เข้าไปในทุกอย่างที่ทำได้เลย:

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

โอ้พระเจ้า เราพัง Rust แล้ว

บางทีเราควรทำความเข้าใจว่าไลฟ์ไทม์พวกนี้หมายความว่าอะไรกันแน่

ไลฟ์ไทม์สามารถทำให้คนจำนวนมากกลัวได้เพราะมันเป็นการเปลี่ยนแปลง
สิ่งที่เรารู้จักและรักมาตั้งแต่ยุคแรกเริ่มของการเขียนโปรแกรม
เราจัดการหลีกเลี่ยงไลฟ์ไทม์มาได้จนถึงตอนนี้ แม้ว่ามันจะพันกันอยู่
ทั่วโปรแกรมของเราตลอดเวลา

ไลฟ์ไทม์ไม่จำเป็นในภาษาที่มี garbage collector เพราะ garbage collector
รับประกันว่าทุกอย่างจะมีชีวิตอยู่นานเท่าที่ต้องการ ข้อมูลส่วนใหญ่ใน Rust
ถูกจัดการ*ด้วยตนเอง* ดังนั้นข้อมูลเหล่านั้นต้องการวิธีแก้ปัญหาอื่น C และ
C++ ให้ตัวอย่างที่ชัดเจนว่าเกิดอะไรขึ้นถ้าคุณปล่อยให้คนทั่วไปใช้ตัวชี้ไปยัง
ข้อมูลแบบสุ่มบนสแต็ก: ความไม่ปลอดภัยที่ไม่สามารถจัดการได้ทุกที่
สิ่งนี้สามารถแยกออกเป็นสองประเภทของข้อผิดพลาด:

* การถือตัวชี้ไปยังสิ่งที่หลุดจากสโคปไปแล้ว
* การถือตัวชี้ไปยังสิ่งที่ถูกกลายพันธุ์ไปแล้ว

ไลฟ์ไทม์แก้ปัญหาทั้งสองนี้ และ 99% ของเวลา มันทำแบบนั้นด้วยวิธีที่
โปร่งใสอย่างสมบูรณ์

แล้วไลฟ์ไทม์คืออะไร?

อย่างง่ายที่สุด ไลฟ์ไทม์คือชื่อของพื้นที่ (~บล็อก/สโคป) ของโค้ดที่ไหนสักแห่ง
ในโปรแกรม แค่นั้น เมื่อเรเฟอเรนซ์ถูกติดแท็กด้วยไลฟ์ไทม์ เราหมายความว่า
มันต้องถูกต้องสำหรับพื้นที่*ทั้งหมด*นั้น สิ่งต่าง ๆ ที่แตกต่างกันมีข้อกำหนด
เกี่ยวกับระยะเวลาที่เรเฟอเรนซ์ต้องและสามารถถูกต้องได้ ระบบทั้งหมดของ
ไลฟ์ไทม์เป็นเพียงระบบแก้ไขข้อจำกัดที่พยายามลดขนาดพื้นที่ของเรเฟอเรนซ์
ทุกตัว ถ้ามันค้นหาชุดของไลฟ์ไทม์ที่ทำให้ข้อจำกัดทั้งหมดพอใจได้สำเร็จ
โปรแกรมของคุณจะคอมไพล์ผ่าน! ไม่อย่างนั้นคุณจะได้รับข้อผิดพลาดกลับมา
ว่ามีบางอย่างไม่มีชีวิตนานพอ

ภายในบล็อกของฟังก์ชัน คุณโดยทั่วไปไม่สามารถพูดถึงไลฟ์ไทม์ได้ และไม่อยากจะทำ
*อยู่ดี* คอมไพเลอร์มีข้อมูลทั้งหมดและสามารถ Infer ข้อจำกัดทั้งหมดเพื่อหา
ไลฟ์ไทม์ที่น้อยที่สุด อย่างไรก็ตาม ที่ระดับชนิดและ API คอมไพเลอร์*ไม่ได้*
มีข้อมูลทั้งหมด มันต้องการให้คุณบอกมันเกี่ยวกับความสัมพันธ์ระหว่าง
ไลฟ์ไทม์ต่าง ๆ เพื่อที่มันจะได้เข้าใจว่าคุณกำลังทำอะไรอยู่

ในหลักการ ไลฟ์ไทม์เหล่านั้น*อาจ*ถูกละเว้นได้เช่นกัน แต่ถ้าทำแบบนั้น
การตรวจสอบการยืมทั้งหมดจะเป็นการวิเคราะห์ทั้งโปรแกรมที่ใหญ่โตมโหฬาร
ซึ่งจะผลิตข้อผิดพลาดที่ไม่เฉพาะท้องถิ่นอย่างน่าตกตะลึง ระบบของ Rust
หมายความว่าการตรวจสอบการยืมทั้งหมดสามารถทำได้ในบล็อกของฟังก์ชัน
แต่ละตัวอย่างอิสระ และข้อผิดพลาดทั้งหมดของคุณควรมีลักษณะเฉพาะท้องถิ่น
(หรือชนิดของคุณมีลายเซ็นไม่ถูกต้อง)

แต่เราเคยเขียนเรเฟอเรนซ์ในลายเซ็นของฟังก์ชันมาก่อนหน้านี้ และมันก็ไม่เป็นไร!
นั่นเป็นเพราะมีบางกรณีที่\commonมากจน Rust จะเลือกไลฟ์ไทม์ให้คุณโดยอัตโนมัติ
นี่คือ*การละเว้นไลฟ์ไทม์ (lifetime elision)*

โดยเฉพาะอย่างยิ่ง:

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

แล้ว `fn foo<'a>(&'a A) -> &'a B` *หมายความว่าอะไร*? ในทางปฏิบัติ มันหมายความว่า
อินพุตต้องมีชีวิตนานพอ ๆ กับเอาต์พุต ดังนั้นถ้าคุณเก็บเอาต์พุตไว้นาน มันจะขยาย
พื้นที่ที่อินพุตต้องถูกต้องสำหรับ เมื่อคุณหยุดใช้เอาต์พุต คอมไพเลอร์จะรู้ว่า
สำหรับอินพุตก็โอเคที่จะกลายเป็นไม่ถูกต้องเช่นกัน

ด้วยระบบนี้ Rust สามารถรับประกันว่าไม่มีอะไรถูกใช้หลัง free และไม่มีอะไร
ถูกกลายพันธุ์ในขณะที่เรเฟอเรนซ์ที่ outstanding ยังมีอยู่ มันแค่ทำให้แน่ใจว่า
ข้อจำกัดทั้งหมดทำงานได้!

เอาล่ะ งั้น Iter

มาถอยกลับไปสถานะที่ไม่มีไลฟ์ไทม์:

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

เราต้องเพิ่มไลฟ์ไทม์เฉพาะในลายเซ็นของฟังก์ชันและชนิดเท่านั้น:

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

เอาล่ะ ผมคิดว่าครั้งนี้เราได้แล้ว

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

โอเค งั้น เราแก้ข้อผิดพลาดเรื่องไลฟ์ไทม์แล้วแต่ตอนนี้เราเจอข้อผิดพลาดชนิดใหม่

เราต้องการเก็บ `&Node` แต่เราได้ `&Box<Node>` โอเค งั้นง่ายมาก
เราแค่ต้อง dereference Box ก่อนที่จะ.take เรเฟอเรนซ์:

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

เราลืม `as_ref` ดังนั้นเราจึงย้าย box เข้าไปใน `map` ซึ่งหมายความว่ามันจะ
ถูก drop ซึ่งหมายความว่าเรเฟอเรนซ์ของเราจะห้อยอยู่:

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

`as_ref` เพิ่มชั้นการอ้อมอีกชั้นหนึ่งที่เราต้องกำจัด:

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

ฟังก์ชัน as_deref และ as_deref_mut มีความเสถียรตั้งแต่ Rust 1.40 ก่อนหน้านั้น
คุณจะต้องใช้ `map(|node| &**node)` และ `map(|node| &mut**node)`
คุณอาจคิดว่า "ว้าว `&**` นั่นมันดูแย่มาก" และคุณไม่ผิด
แต่เหมือนไวน์ชั้นดี Rust ดีขึ้นตามกาลเวลา และเราไม่จำเป็นต้องทำแบบนั้นอีกแล้ว
โดยปกติ Rust เก่งมากในการแปลงแบบนี้โดยปริยาย ผ่านกระบวนการที่เรียกว่า
*deref coercion* ซึ่งโดยพื้นฐานแล้วมันสามารถแทรก \* ทั่วโค้ดของคุณเพื่อ
ให้ผ่านการตรวจสอบชนิด มันทำแบบนี้ได้เพราะเรามีบอร์โรว์เช็กเกอร์เพื่อ
รับประกันว่าเราจะไม่ทำพังตัวชี้!

แต่ในกรณีนี้ คลอ저ประกอบกับข้อเท็จจริงที่ว่าเรามี `Option<&T>` แทนที่จะเป็น
`&T` มันซับซ้อนเกินไปเล็กน้อยจนมันไม่สามารถคิดออก ดังนั้นเราจึงต้อง
ช่วยมันด้วยการระบุอย่างชัดเจน โชคดีที่จากประสบการณ์ของผม สิ่งนี้หายากมาก

เพื่อความสมบูรณ์ เรา*สามารถ*ให้มันเป็นคำใบ้*อื่น*ด้วย *turbofish*:

```rust ,ignore
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

เห็นมั้ย map เป็นฟังก์ชันเจเนอริก:

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
```

Turbofish `::<>` ให้เราบอกคอมไพเลอร์ว่าเราคิดว่าชนิดของเจเนอริกเหล่านั้น
ควรเป็นอะไร ในกรณีนี้ `::<&Node<T>, _>` บอกว่า "มันควรคืน `&Node<T>`
และผมไม่รู้/ไม่แคร์เกี่ยวกับชนิดอื่น"

สิ่งนี้ทำให้คอมไพเลอร์รู้ว่า `&node` ควรได้รับ deref coercion
ดังนั้นเราจึงไม่ต้องใช้ \* ทั้งหมดด้วยตนเอง!

แต่ในกรณีนี้ผมไม่คิดว่ามันเป็นการปรับปรุงจริง ๆ นี่เป็นแค่ข้ออ้าง
อำพรางเพื่ออวด deref coercion และ turbofish ที่บางครั้งก็มีประโยชน์ 😅

มาเขียนเทสต์เพื่อให้แน่ใจว่าเราไม่ได้ no-op หรืออะไรทำนองนั้น:

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

สุดยอด

สุดท้าย ควรสังเกตว่าเรา*สามารถ*ใช้การละเว้นไลฟ์ไทม์ที่นี่ได้จริง ๆ:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}
```

เทียบเท่ากับ:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```

เย่ ไลฟ์ไทม์น้อยลง!

หรือถ้าคุณไม่สบายใจที่จะ "ซ่อน" ว่า struct มีไลฟ์ไทม์ คุณสามารถใช้
ไวยากรณ์ "ไลฟ์ไทม์ที่ละเว้นอย่างชัดเจน" ของ Rust 2018 `'_`:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}
```