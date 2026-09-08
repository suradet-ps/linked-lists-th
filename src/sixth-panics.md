# Drop และความปลอดภัยต่อการแพนิก

ว่าแต่ คุณสังเกตเห็นคอมเมนต์นี้หรือเปล่า:

```rust
// Note that we don't need to mess around with `take` anymore
// because everything is Copy and there are no dtors that will
// run if we mess up... right? :) Riiiight? :)))
```

คิดว่ามันถูกไหม?

โทษที คุณลืมไปแล้วหรือว่ากำลังอ่านหนังสือเล่มไหนอยู่? แน่นอนว่ามันผิดสิครับ! (ก็นิดหน่อยนะ)

มาดูเนื้อหาภายในของ `pop_front` อีกครั้ง:

```rust ,ignore
// Bring the Box back to life so we can move out its value and
// Drop it (Box continues to magically understand this for us).
let boxed_node = Box::from_raw(node.as_ptr());
let result = boxed_node.elem;

// Make the next node into the new front.
self.front = boxed_node.back;
if let Some(new) = self.front {
    // Cleanup its reference to the removed node
    (*new.as_ptr()).front = None;
} else {
    // If the front is now null, then this list is now empty!
    debug_assert!(self.len == 1);
    self.back = None;
}

self.len -= 1;
result
// Box gets implicitly freed here, knows there is no T.
```

คุณเห็นบั๊กไหม? น่าสยดสยองมาก เพราะจริง ๆ แล้วมันคือบรรทัดนี้ต่างหาก:

```rust ,ignore
debug_assert!(self.len == 1);
```

*จริงดิ?* โค้ด integrity check สำหรับเทสต์ที่เราอุตส่าห์ใส่ไว้เนี่ยนะเป็นบั๊ก?? ใช่แล้วครับ!!! เอาเข้าจริง ถ้าเราอิมพลีเมนต์คอลเลกชันได้ถูกต้อง มันก็*ไม่ควร*เป็นบั๊กหรอก แต่มันสามารถเปลี่ยนเรื่องไม่เป็นเรื่องอย่าง "อ้าว ลืมอัปเดต len ให้ตรงซะงั้น" ให้กลายเป็น *บั๊กด้าน memory safety ที่ถูก exploit ได้*! ทำไมล่ะ? ก็เพราะมันแพนิกได้ไงครับ! ปกติแล้วคุณแทบไม่ต้องกังวลเรื่องแพนิกเลย แต่พอคุณเริ่มเขียนโค้ด unsafe แบบ *จัดหนัก* และเล่นแร่แปรธาตุกับ "invariant" แบบตามใจชอบ คุณจะต้องระแวดระวังเรื่องการแพนิกขั้นสุดยอด!

เราต้องพูดถึง [*exception safety*](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html) (หรือที่รู้จักกันในชื่อ panic safety หรือ unwind safety...)

เรื่องมันมีอยู่ว่า: โดยดีฟอลต์แล้ว เมื่อเกิดแพนิก มันจะทำการ *unwind* ซึ่งการ unwinding ก็เป็นแค่ศัพท์หรู ๆ สำหรับบอกว่า "สั่งให้ทุกฟังก์ชัน return ออกมาทันทีเดี๋ยวนี้เลย" คุณอาจจะคิดว่า "เอ้า ถ้า *ทุกฟังก์ชัน* พากัน return หมด อีกเดี๋ยวโปรแกรมก็พังดับไปเอง แล้วจะไปแคร์ทำไมล่ะ?" คิดแบบนั้นคุณคิดผิดแล้วครับ!

เราต้องแคร์ด้วยสองเหตุผลหลัก: หนึ่งคือ destructor จะทำงานเมื่อฟังก์ชัน return และสองคือการ unwind นั้นสามารถถูก *ดักจับ (catch)* ได้! ทั้งสองกรณีนี้ทำให้โค้ดยังสามารถทำงานต่อไปได้หลังจากเกิดแพนิก เราจึงต้องระวังเป็นพิเศษและทำให้แน่ใจว่า unsafe collection ของเราจะอยู่ใน *สถานะที่สมบูรณ์และสมเหตุสมผล* เสมอเมื่อใดก็ตามที่อาจมีแพนิกเกิดขึ้น เพราะการแพนิกแต่ละครั้งก็คือการ return ออกมาก่อนกำหนดโดยนัยนั่นเอง!

ลองมาคิดดูว่าคอลเลกชันของเราอยู่ในสถานะไหนเมื่อโค้ดวิ่งมาถึงบรรทัดนั้น:

เรามี `boxed_node` อยู่บนสแตก และเราได้ย้ายเอา element ออกมาจากมันแล้ว ถ้าหากฟังก์ชันเกิด return ณ จุดนี้ `Box` จะถูกดรอป และโหนดนั้นก็จะถูกคืนหน่วยความจำทันที... คุณเห็นภาพหรือยังล่ะครับ? `self.back` ยังคงชี้ไปยังโหนดที่ถูกปลดปล่อยไปแล้วนั้นอยู่เลย! และทันทีที่เราอิมพลีเมนต์ส่วนที่เหลือของคอลเลกชันแล้วเริ่มนำ `self.back` ไปใช้งาน สิ่งนี้จะทำให้เกิด use-after-free ในทันที! หลอนเลยไหมล่ะ!

ที่น่าสนใจคือ บรรทัดนี้ก็มีปัญหาคล้าย ๆ กัน แต่ปลอดภัยกว่ามาก:

```rust ,ignore
self.len -= 1;
```

โดยดีฟอลต์ใน debug build นั้น Rust จะคอยเช็กเรื่อง underflow และ overflow เสมอ และจะเกิดแพนิกขึ้นเมื่อมันเกิดขึ้น ใช่ครับ การคำนวณเลขคณิตทุกจุดมีความเสี่ยงต่อ panic safety ทั้งนั้น! แต่บรรทัดนี้ *ยังดีกว่า* ตรงที่มันเกิดขึ้นหลังจากที่เราซ่อมแซม invariant ทั้งหมดของเราเรียบร้อยแล้ว ดังนั้นมันจะไม่ก่อให้เกิดปัญหาด้าน memory safety... ตราบใดที่เราไม่ได้หลับหูหลับตาเชื่อว่าค่า `len` นั้นถูกต้อง แต่ก็นั่นแหละ ถ้ามันเกิด underflow ค่ามันก็ผิดตั้งแต่แรกอยู่แล้ว ยังไงเราก็ไม่รอดทั้งสองทาง! แต่ในอีกแง่หนึ่ง `debug_assert` ตัวนั้นกลับ *แย่กว่า* เพราะมันสามารถยกระดับจากปัญหาเล็ก ๆ ให้กลายเป็นช่องโหว่วิกฤตได้เลย!

ผมพูดถึงคำว่า "invariant" (สัจพจน์คงตัว) มาหลายครั้งแล้ว และนั่นก็เป็นเพราะมันเป็นคอนเซปต์ที่มีประโยชน์อย่างยิ่งสำหรับเรื่อง panic safety! โดยพื้นฐานแล้ว สำหรับมุมมองของบุคคลภายนอกที่มองเข้ามายังคอลเลกชันของเรา จะมีคุณสมบัติบางอย่างที่เราต้องรักษาไว้ให้เป็นจริงเสมอ สำหรับ LinkedList คุณสมบัติหนึ่งในนั้นก็คือ โหนดใดก็ตามที่ยังสามารถเข้าถึงได้ในลิสต์ จะต้องยังคงได้รับการจัดสรรหน่วยความจำและผ่านการกำหนดค่าเริ่มต้นแล้วเสมอ

แต่ *ภายใน* ตัวอิมพลีเมนต์นั้น เราจะมีความยืดหยุ่นให้ละเมิด invariant ได้ *ชั่วคราว* ตราบใดที่เราทำให้แน่ใจว่าจะซ่อมแซมพวกมันให้กลับมาสมบูรณ์ *ก่อนที่จะมีใครทันสังเกตเห็น* นี่คือหนึ่งใน "killer feature" ของระบบ ownership และ borrowing ใน Rust สำหรับการเขียนคอลเลกชัน: หากฟังก์ชันต้องการ `&mut Self` เราก็ได้รับ *การรับประกัน* อย่างแน่นอนว่าเรามีสิทธิ์เข้าถึงคอลเลกชันนี้แต่เพียงผู้เดียว และมันปลอดภัยที่เราจะละเมิด invariant ชั่วคราวได้อย่างสบายใจ โดยมั่นใจได้ว่าไม่มีใครสามารถแอบเข้ามาแก้ไขหรือส่องดูมันได้

ตัวอย่างที่แสดงพลังของเรื่องนี้ได้ชัดเจนที่สุดคงหนีไม่พ้น [Vec::drain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.drain) ซึ่งยอมให้คุณทุบทำลาย invariant หลักของ `Vec` จนเละ แล้วเริ่มย้ายค่าออกมาจาก *ข้างหน้า* หรือแม้แต่ *ตรงกลาง* ของ `Vec` ได้เลย เหตุผลที่สิ่งนี้ *sound* (ถูกต้องปลอดภัย) ก็เพราะตัว iterator `Drain` ที่เราส่งกลับไปนั้นถือสิทธิ์ `&mut` อ้างอิงไปยัง `Vec` ไว้อยู่ ดังนั้นการเข้าถึงทั้งหมดจึงถูกปิดกั้นไว้เบื้องหลังมัน! ไม่มีใครสามารถส่องดู `Vec` ได้จนกว่า `Drain` จะหลุดพ้น scope ไป และจากนั้น destructor ของมันก็จะ "ซ่อมแซม" `Vec` ให้กลับมาเรียบร้อยก่อนที่จะมีใครทันสังเกตเห็น มันช่างเพอร์เฟกต์--

[มันไม่ได้เพอร์เฟกต์ขนาดนั้น](https://doc.rust-lang.org/nightly/nomicon/leaking.html#drain) น่าเสียดายที่คุณ [ไม่สามารถพึ่งพาให้ destructor ในโค้ดที่คุณไม่ได้เป็นผู้ควบคุมทำงานได้เสมอไป](https://doc.rust-lang.org/std/mem/fn.forget.html) และด้วยเหตุนี้ แม้แต่กับ `Drain` เราก็ยังต้องลงแรงเพิ่มอีกนิดเพื่อให้ type ของเรารักษา invariant ไว้ได้เสมอ แต่เป็นวิธีที่ออกจะเกรียน ๆ หน่อย: [เราแค่รีเซ็ต len ของ Vec ให้กลายเป็น 0 ตั้งแต่เริ่มเลย](https://doc.rust-lang.org/std/mem/fn.forget.html) ดังนั้นถ้าใครทำ `Drain` รั่วไหล (leak) พวกเขาก็จะได้ `Vec` ที่ *ปลอดภัย* ไปใช้... ทว่าข้อมูลทั้งหมดข้างในก็จะหายเกลี้ยง นายทำฉันรั่วเหรอ? งั้นฉันทำข้อมูลนายรั่วบ้าง! ตาต่อตาฟันต่อฟัน! ยุติธรรมอย่างแท้จริง!

สำหรับสถานการณ์ที่คุณ *สามารถ* นำ destructor มาช่วยรักษา panic safety ได้จริง ๆ ลองแวะไปดู [กรณีศึกษา BinaryHeap::sift_up](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html#binaryheapsift_up) ได้ครับ

อย่างไรก็ตาม เราไม่จำเป็นต้องใช้ลูกเล่นหวือหวาพวกนี้ใน LinkedList ของเราหรอกครับ เราแค่ต้องระมัดระวังให้มากขึ้นว่าเรากำลังแหก invariant ตรงจุดไหน สิ่งใดที่เราเชื่อถือได้หรือต้องควบคุมให้ถูกต้อง และหลีกเลี่ยงการก่อให้เกิด unwind โดยไม่จำเป็นในระหว่างขั้นตอนที่ซับซ้อนยุ่งเหยิง

ในกรณีนี้ เรามีทางเลือกสองทางที่จะทำให้โค้ดของเราแข็งแกร่งขึ้นอีกหน่อย:

* ใช้ฟังก์ชันอย่าง `Option::take` ให้บ่อยและหนักหน่วงขึ้น เพราะมันมีความเป็น "transactional" (ทำรายการแบบเบ็ดเสร็จ) มากกว่า และช่วยรักษา invariant ได้ดีกว่า
* กำจัด `debug_assert` ทิ้งไปให้หมด แล้วเชื่อมั่นในตัวเองด้วยการเขียนเทสต์ที่ดีขึ้น พร้อมฟังก์ชัน "integrity check" แยกต่างหากที่จะไม่มีวันไปรันอยู่ในโค้ดของผู้ใช้อย่างแน่นอน

โดยหลักการแล้วผมชอบตัวเลือกแรกนะ แต่มันไม่ได้ผลดีเท่าไหร่กับ doubly-linked list เพราะทุกอย่างถูกเข้ารหัสเชื่อมโยงซ้ำซ้อนแบบไปกลับสองทาง การใช้ `Option::take` ไม่ได้ช่วยแก้ปัญหาตรงนี้ แต่การเลื่อน `debug_assert` ลงไปอีกหนึ่งบรรทัดต่างหากที่ช่วยได้ ทว่าถามจริง เราจะทำให้ชีวิตตัวเองลำบากไปทำไมกัน? ลบ `debug_assert` พวกนั้นทิ้งไปให้หมดเลยดีกว่า แล้วทำให้แน่ใจว่าจุดใดก็ตามที่อาจเกิดการแพนิก จะต้องอยู่ที่ตอนต้นหรือตอนท้ายสุดของเมธอดเสมอ ซึ่งเป็นจุดที่ invariant ของเราควรจะได้รับการรักษาให้อยู่ในสถานะที่ถูกต้องแล้ว

(ในแง่นี้ การมองว่าพวกมันเป็น *preconditions* (เงื่อนไขก่อนรัน) และ *postconditions* (เงื่อนไขหลังรัน) อาจจะตรงตัวกว่า แต่คุณก็ควรพยายามมองและดูแลพวกมันให้เป็น invariant ให้ได้มากที่สุดเท่าที่จะทำได้!)

นี่คือ implementation เต็มรูปแบบของเราตอนนี้:

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Put the new front before the old one
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // If there's no front, then we're the empty list and need 
                // to set the back too.
                self.back = Some(new);
            }
            // These things always happen!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a front node to pop.
            self.front.map(|node| {
                // Bring the Box back to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new front.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).front = None;
                } else {
                    // If the front is now null, then this list is now empty!
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }
}
```

แล้วมีอะไรที่แพนิกได้ที่นี่บ้างล่ะ? เอาจริง ๆ การจะมองเรื่องนี้ออกคุณต้องมีความเป็นเซียน Rust อยู่พอตัวเลยล่ะ แต่โชคดีหน่อยที่ผมเป็น!

จุดเดียวในโค้ดชุดนี้ที่ผมเห็นว่า *อาจจะ* แพนิกได้ (ตัดเรื่องพิเรนทร์สุดกู่ประเภทมีคนทะลึ่ง recompile stdlib โดยเปิด debug_assert ทิ้งไว้ออกไปนะ ซึ่งนั่นไม่ใช่สิ่งที่คุณควรทำเด็ดขาด) ก็คือ `Box::new` (เมื่อเกิดกรณี out-of-memory หน่วยความจำหมด) กับการคำนวณเลขคณิตของ `len` ซึ่งของพวกนั้นทั้งหมดอยู่ตรงหัวหรือท้ายสุดของเมธอดเราทั้งสิ้น ดังนั้น ใช่ครับ โค้ดเราตอนนี้ปลอดภัยไร้กังวลอย่างสวยงาม!

...คุณประหลาดใจไหมล่ะที่ `Box::new` ก็แพนิกได้ด้วย? การแพนิกมันชอบแอบมาตุ๋ยคุณแบบนี้แหละ! พยายามรักษา invariant เอาไว้ให้มั่น จะได้ไม่ต้องมานั่งปวดหัวกับเรื่องพวกนี้ครับ!
