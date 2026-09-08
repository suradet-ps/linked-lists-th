# วาเรียนซ์และ PhantomData

ตอนนี้มันจะน่ารำคาญถ้าเราเลื่อนเรื่องนี้ไปแก้ทีหลัง ดังนั้นเราจะจัดการกับเนื้อหา Hardcore Layout ตอนนี้เลย

มีห้าสิ่งที่น่ากลัวในการสร้าง unsafe Rust collections:

1. [Variance](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)
2. [Drop Check](https://doc.rust-lang.org/nightly/nomicon/dropck.html)
3. [NonNull Optimizations](https://doc.rust-lang.org/nightly/std/ptr/struct.NonNull.html)
4. [The isize::MAX Allocation Rule](https://doc.rust-lang.org/nightly/nomicon/vec/vec-alloc.html)
5. [Zero-Sized Types](https://doc.rust-lang.org/nightly/nomicon/vec/vec-zsts.html)

โชคดีที่ 2 ตัวสุดท้ายจะไม่เป็นปัญหาสำหรับเรา

ตัวที่สามเรา *สามารถ*ทำให้เป็นปัญหาได้แต่มันยุ่งยากเกินไป -- ถ้าคุณเลือกใช้ LinkedList คุณก็ยอมแพ้เรื่องประสิทธิภาพความจำไปแล้ว 100 เท่า

ตัวที่สองเป็นสิ่งที่ผมเคยยืนยันว่าสำคัญมากและ std เคยใช้งานมัน แต่ค่าเริ่มต้นนั้นปลอดภัย วิธีใช้งานมันไม่เสถียร และคุณต้องพยายาม *มาก* กว่าจะสังเกตข้อจำกัดของค่าเริ่มต้นได้ ดังนั้นไม่ต้องกังวล

นั่นเหลือแค่ Vaเรียนซ์ จริงๆ แล้วคุณอาจจะเลื่อนเรื่องนี้ไปก่อนก็ได้ แต่ผมยังมีความภูมิใจในฐานะ Collections Person ดังนั้นเราจะทำเรื่อง Vaเรียนซ์ให้เสร็จ

เอาล่ะ เซอร์ไพรส์: Rust มีการเป็นซับไทป์ ในกรณีนี้ `&'big T` เป็น *ซับไทป์* ของ `&'small T` ทำไม? ก็เพราะว่าถ้าโค้ดบางตัวต้องการ reference ที่มีอายุเท่ากับ region ของโปรแกรมบางส่วน มันมักจะปลอดภัยที่จะให้ reference ที่อยู่ *ยาวกว่า* เช่น โดยสัญชาตญาณมันถูกต้อง จริงไหม?

ทำไมถึงสำคัญ? ลองนึกภาพโค้ดที่รับค่าสองค่าที่มี type เดียวกัน:

```rust ,ignore
fn take_two<T>(_val1: T, _val2: T) { }
```

นี่คือโค้ดที่น่าเบื่อมาก ดังนั้นเราควรคาดหวังว่ามันจะทำงานกับ T=&u32 ได้ใช่ไหม?

```rust
fn two_refs<'big: 'small, 'small>(
    big: &'big u32, 
    small: &'small u32,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

ใช่ คอมไพล์ผ่าน!

ตอนนี้มาสนุกกันโดยห่อหุ้มมันด้วย... ไม่รู้สิ `std::cell::Cell`:

```rust ,compilefail
use std::cell::Cell;

fn two_refs<'big: 'small, 'small>(
    // NOTE: these two lines changed
    big: Cell<&'big u32>, 
    small: Cell<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
error[E0623]: lifetime mismatch
 --> src/main.rs:7:19
  |
4 |     big: Cell<&'big u32>, 
  |               ---------
5 |     small: Cell<&'small u32>,
  |                 ----------- these two types are declared with different lifetimes...
6 | ) {
7 |     take_two(big, small);
  |                   ^^^^^ ...but data from `small` flows into `big` here
```

ห๊ะ??? เราไม่ได้แตะ lifetime เลย ทำไมคอมไพเลอร์ถึงโมโหตอนนี้!?

อ้าว สิ่งที่เกี่ยวกับ lifetime "subtyping" มันง่ายมาก ดังนั้นมันจึงพังเมื่อเราห่อหุ้ม reference ไว้ในบางสิ่ง ดูสิ มันพังกับ Vec ด้วย:

```rust
fn two_refs<'big: 'small, 'small>(
    big: Vec<&'big u32>, 
    small: Vec<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
    Finished dev [unoptimized + debuginfo] target(s) in 1.07s
     Running `target/debug/playground`
```

ดูสิ มันคอมไพล์ไม่ผ้-- เดี๋ยว Vec มีเวทย์มนต์?????

ก็... ใช่ แต่ก็ไม่ใช่ เวทย์มนต์อยู่ในตัวเราตลอดมา และเวทย์มนต์นั้นคือ ✨*วาเรียนซ์*✨

อ่าน [บทเรียนเรื่อง subtyping ใน nomicon](https://doc.rust-lang.org/nightly/nomicon/subtyping.html) ถ้าคุณต้องการรายละเอียดทั้งหมด แต่โดยทั่วไป subtyping *ไม่* ปลอดภัยเสมอไป ในกรณีเฉพาะมันไม่ปลอดภัยเมื่อมี mutable reference เข้ามาเกี่ยวข้องเพราะคุณสามารถใช้สิ่งต่างๆ เช่น `mem::swap` และจู่ๆ ก็มี dangling pointers!

สิ่งที่ "เหมือน mutable reference" จะเป็น *invariant* ซึ่งหมายความว่ามันขัดขวางไม่ให้ subtyping เกิดขึ้นกับ generic parameters ของมัน ดังนั้นเพื่อความปลอดภัย `&mut T` จะเป็น invariant กับ T และ `Cell<T>` จะเป็น invariant กับ T เพราะ `&Cell<T>` โดยพื้นฐานแล้วก็แค่ `&mut T` (เพราะ interior mutability)

ส่วนใหญ่สิ่งที่ไม่ใช่ invariant จะเป็น *covariant* ซึ่งหมายความว่า subtyping จะ "ผ่าน" มันไปและทำงานได้ตามปกติ (ยังมี contravariant types ที่ทำให้ subtyping ย้อนกลับ แต่แทบไม่มีใครใช้และไม่มีใครชอบมัน ดังนั้นผมจะไม่กล่าวถึงมันอีก)

Collections มักจะมี mutable pointer ไปยังข้อมูล ดังนั้นคุณอาจคาดหวังว่ามันจะเป็น invariant ด้วย แต่ในความเป็นจริงมันไม่จำเป็นต้องเป็น! เพราะระบบ ownership ของ Rust `Vec<T>` มีความหมายเทียบเท่ากับ T และนั่นหมายความว่ามันปลอดภัยที่จะเป็น covariant!

น่าเสียดายที่ definition นี้เป็น invariant:

```rust
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = *mut Node<T>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

แต่ Rust ตัดสินใจเรื่อง vaเรียนซ์ของสิ่งต่างๆ อย่างไร? ก็ในยุคก่อน 1.0 เราเคยใช้วิธีให้ผู้คนระบุ vaเรียนซ์ที่ต้องการ... และมันเป็นหายนะอย่างสิ้นเชิง! Subtyping และ vaเรียนซ์เป็นเรื่องยากที่จะเข้าใจ และนักพัฒนา core ไม่เห็นด้วยกับคำศัพท์พื้นฐาน! ดังนั้นเราจึงย้ายไปใช้วิธี "variance by example": คอมไพเลอร์จะดูฟิลด์ของคุณและคัดลอก vaเรียนซ์ของมัน ถ้ามีความขัดแย้ง invariant จะเป็นฝ่ายชนะเสมอ เพราะนั่นปลอดภัย

ดังนั้นสิ่งที่อยู่ใน type definitions ที่ Rust กำลังโกรธคืออะไร? `*mut`!

Raw pointers ใน Rust โดยทั่วไปพยายามให้คุณทำอะไรก็ได้ แต่มันมีคุณสมบัติความปลอดภัยอย่างเดียว: เพราะคนส่วนใหญ่ไม่รู้ว่า vaเรียนซ์และ subtyping มีอยู่ใน Rust และการเป็น covariant *อย่างไม่ถูกต้อง* จะอันตรายมาก `*mut T` จึงเป็น invariant เพราะมีโอกาสสูงที่มันจะถูกใช้งาน "ในฐานะ" `&mut T`

นี่น่ารำคาญมากสำหรับผมในฐานะผู้ที่ใช้เวลาเขียน collections ใน Rust มากมาย นี่คือเหตุผลที่ตอนที่ผมสร้าง [std::ptr::NonNull](https://doc.rust-lang.org/std/ptr/struct.NonNull.html) ผมได้เพิ่มเวทย์มนต์เล็กๆ น้อยๆ นี้:

> Unlike `*mut T`, `NonNull<T>` was chosen to be covariant over `T`. This makes it possible to use `NonNull<T>` when building covariant types, but introduces the risk of unsoundness if used in a type that shouldn’t actually be covariant.

แต่เดี๋ยว interface ของมันถูกสร้างจาก `*mut T` มันเป็นยังไง! มันเป็นเวทย์มนต์หรือเปล่า!? มาดูกัน:

```rust
pub struct NonNull<T> {
    pointer: *const T,
}


impl<T> NonNull<T> {
    pub unsafe fn new_unchecked(ptr: *mut T) -> Self {
        // SAFETY: the caller must guarantee that `ptr` is non-null.
        unsafe { NonNull { pointer: ptr as *const T } }
    }
}
```

ไม่ใช่ ไม่มีเวทย์มนต์ที่นี่! NonNull แค่ใช้ประโยชน์จากความจริงที่ว่า `*const T` เป็น covariant และเก็บมันไว้แทน โดยสลับไปมาระหว่าง `*mut T` ที่ API boundary เพื่อให้ดูเหมือนว่าเก็บ `*mut T` ไว้ นั่นคือทั้งหมด! นั่นคือวิธีที่ collections ใน Rust เป็น covariant! และมันน่าหงุดหงิด! ดังนั้นผมจึงให้ Good Pointer Type ทำมันให้คุณ ไม่ต้องห่วง สนุกกับ subtyping footgun ของคุณ!

ทางออกของปัญหาทั้งหมดคือใช้ NonNull แล้วถ้าคุณต้องการ pointer ที่เป็น nullable อีกครั้ง ใช้ `Option<NonNull<T>>` เราจะทำแบบนั้นจริงๆ เหรอ..?

ใช่ มันแย่ แต่เรากำลังสร้าง linked lists ระดับ *production grade* ดังนั้นเราจะรับประทานผักทั้งหมดและทำสิ่งต่างๆ แบบยาก (เราสามารถใช้ `*const T` เปล่าๆ และ cast ไปทุกที่ แต่ผมอยากเห็นว่ามันเจ็บปวดแค่ไหน... เพื่อวิทยาศาสตร์ Ergonomics)

ดังนั้นนี่คือ type definitions สุดท้ายของเรา:

```rust
use std::ptr::NonNull;

// !!!This changed!!!
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

...เดี๋ยวก่อนยังไม่เสร็จ สิ่งสุดท้ายทุกครั้งที่คุณทำ raw pointer คุณควรเพิ่ม Ghost เพื่อป้องกัน pointers ของคุณ:

```rust ,ignore
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    /// We semantically store values of T by-value.
    _boo: PhantomData<T>,
}
```

ในกรณีนี้ผมไม่คิดว่าเรา *จำเป็นต้อง* ใช้ [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html) แต่ทุกครั้งที่คุณ *ใช้* NonNull (หรือ raw pointers ทั่วไป) คุณควรเพิ่มมันเสมอเพื่อความปลอดภัยและให้แน่ใจว่าคอมไพเลอร์และคนอื่นเข้าใจว่าคุณ *คิดว่า* กำลังทำอะไร

PhantomData เป็นวิธีที่เราให้คอมไพเลอร์มีฟิลด์ "ตัวอย่าง" เพิ่มเติมที่ *มีอยู่ในทางทฤษฎี* ใน type ของคุณ แต่ด้วยเหตุผลต่างๆ (indirection, type erasure, ...) ไม่มีอยู่ ในกรณีนี้เราใช้ NonNull เพราะเรากล่าวอ้างว่า type ของเราทำงาน "ราวกับ" เก็บค่า T ไว้ ดังนั้นเราจึงเพิ่ม PhantomData เพื่อให้สิ่งนี้ชัดเจน

stdlib จริงๆ แล้วมีเหตุผลอื่นที่จะทำสิ่งนี้เพราะมันมีสิทธิ์เข้าถึง [Drop Check overrides](https://doc.rust-lang.org/nightly/nomicon/dropck.html) ที่น่ากลัว แต่ฟีเจอร์นั้นถูกแก้ไขหลายครั้งจนผมไม่แน่ใจว่า PhantomData ยัง *เป็นสิ่งที่จำเป็น* สำหรับมันหรือไม่ ผมจะ cargo-cult มันตลอดกาล เพราะ Drop Check Magic ถูกเผาเข้าไปในสมองของผมแล้ว!

(Node เก็บ T จริงๆ ดังนั้นมันไม่ต้องทำแบบนี้ ยินดี!)

...เอาล่ะพูดจริงๆ เราเสร็จกับ layout แล้ว! ไปต่อที่ฟังก์ชันพื้นฐานจริงๆ!
