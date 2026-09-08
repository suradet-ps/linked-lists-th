# Drop และความปลอดภัยต่อการแพนิก

สวัสดี คุณสังเกตเห็น comment นี้ไหม:

```rust
// Note that we don't need to mess around with `take` anymore
// because everything is Copy and there are no dtors that will
// run if we mess up... right? :) Riiiight? :)))
```

มันถูกต้องหรือเปล่า?

ขอโทษ คุณลืมหนังสือที่กำลังอ่านไปแล้วหรือ? แน่นอนว่ามันผิด! (ประมาณนั้น)

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

คุณเห็นบั๊กไหม? น่าตกใจมาก มันคือบรรทัดนี้จริงๆ:

```rust ,ignore
debug_assert!(self.len == 1);
```

*จริงหรือ?* การตรวจสอบความสมบูรณ์สำหรับการทดสอบของเราเป็นบั๊กเหรอ? ใช่!!! ถ้าเรา implement collection ได้ถูกต้อง มัน*ไม่ควร*เป็น แต่มันสามารถเปลี่ยนสิ่งที่ดูไม่เป็นอันตรายอย่าง "อ้าว เราดูแลให้ len อัปเดตได้ไม่ดี" ให้กลายเป็น*บั๊กด้านความปลอดภัยของหน่วยความจำที่ถูก exploit ได้*! ทำไม? เพราะมันสามารถแพนิกได้! โดยทั่วไปคุณไม่จำเป็นต้องคิดหรือกังวลเกี่ยวกับการแพนิก แต่เมื่อคุณเริ่มเขียนโค้ด *unsafe* จริงๆ และเล่นกับ "invariants" อย่างหลวมๆ คุณต้องกลายเป็นคนที่ตื่นตัวเรื่องการแพนิกเป็นพิเศษ!

เราต้องพูดถึง [*exception safety*](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html) (หรือที่เรียกว่า ความปลอดภัยต่อการแพนิก, unwind safety, ...).

ดังนั้นเรื่องมีอยู่ว่า: โดยค่าเริ่มต้น การแพนิกจะ *unwinding* Unwinding เป็นคำหรูๆ ที่แปลว่า "ให้ทุกฟังก์ชันคืนค่าทันที" คุณอาจคิดว่า "ถ้า *ทุก* ฟังก์ชันคืนค่า โปรแกรมก็ใกล้ตายแล้ว ทำไมต้องกังวล?" แต่คุณคิดผิด!

เราต้องกังวลด้วยสองเหตุผล: เมื่อฟังก์ชันคืนค่า จะมี destructor ทำงาน และ unwind สามารถถูก *จับ* ได้ ในทั้งสองกรณี โค้ดยังสามารถทำงานต่อได้หลังจากเกิดการแพนิก ดังนั้นเราต้องระมัดระวังเป็นอย่างยิ่งและให้แน่ใจว่า collection unsafe ของเราอยู่ใน*สถานะที่สมเหตุสมผล*บางรูปแบบเสมอ เมื่อใดก็ตามที่อาจเกิดการแพนิก เพราะการแพนิกแต่ละครั้งคือ return ก่อนกำหนดโดยนัย!

ลองนึกดูว่า collection ของเราอยู่ในสถานะใดเมื่อเราไปถึงบรรทัดนั้น:

เรามี `boxed_node` อยู่บน stack และเราได้ดึง element ออกมาจากมันแล้ว ถ้าเรา return ณ จุดนี้ Box จะถูก drop และ node จะถูกปลดปล่อย คุณเห็นหรือยัง? `self.back` ยังคงชี้ไปที่ node ที่ถูกปลดปล่อยนั้น! เมื่อเรา implement ส่วนที่เหลือของ collection และเริ่มใช้ `self.back` เพื่อทำสิ่งต่างๆ นี่อาจส่งผลให้เกิด use-after-free! น่ากลัว!

น่าสนใจ บรรทัดนี้มีปัญหาที่คล้ายกัน แต่ปลอดภัยกว่ามาก:

```rust ,ignore
self.len -= 1;
```

โดยค่าเริ่มต้นใน debug builds Rust จะตรวจสอบ underflow และ overflow และจะแพนิกเมื่อเกิดขึ้น ใช่ ทุกการคำนวณทางคณิตศาสตร์เป็นความเสี่ยงต่อความปลอดภัยต่อการแพนิก! บรรทัดนี้ *ดีกว่า* เพราะมันเกิดขึ้นหลังจากเราซ่อม invariants ทั้งหมดของเราแล้ว ดังนั้นมันจะไม่ทำให้เกิดปัญหาด้านความปลอดภัยของหน่วยความจำ... ตราบเท่าที่เราไม่ไว้ใจว่า len ถูกต้อง แต่ถ้ามัน underflow มันก็ผิดแน่นอน ดังนั้นเราตายทั้งสองทาง! debug assert แย่กว่าในบางแง่เพราะมันสามารถยกระดับปัญหาเล็กน้อยให้กลายเป็นปัญหาวิกฤตได้!

ผมได้ใช้คำว่า "invariants" มาหลายครั้งแล้ว และนั่นเป็นเพราะมันเป็นแนวคิดที่มีประโยชน์มากสำหรับความปลอดภัยต่อการแพนิก! โดยพื้นฐานแล้ว สำหรับผู้สังเกตการณ์จากภายนอก collection ของเรา มีคุณสมบัติบางอย่างที่เรารักษาไว้เสมอ สำหรับ LinkedList คุณสมบัติหนึ่งคือ node ใดๆ ที่สามารถเข้าถึงได้ใน list ของเรายังคงถูกจัดสรรและเริ่มต้นค่าแล้ว

*ภายใน* การ implement เราสามารถ breaking invariants ได้*ชั่วคราว*ได้บ้าง ตราบเท่าที่เราแน่ใจว่าจะซ่อมมัน*ก่อนที่ใครจะสังเกตเห็น* นี่คือหนึ่งใน "killer apps" ของระบบ ownership และ borrowing ของ Rust สำหรับ collection: ถ้าการดำเนินการต้องใช้ `&mut Self` เรารับประกันได้ว่าเรามีสิทธิ์เข้าถึง collection แต่เพียงผู้เดียว และเราสามารถ breaking invariants ได้อย่างปลอดภัย โดยรู้ว่าไม่มีใครสามารถแอบมาทำอะไรกับมันได้

การแสดงออกที่ยิ่งใหญ่ที่สุดของเรื่องนี้อาจเป็น [Vec::drain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.drain) ซึ่งจริงๆ แล้วให้คุณทำลาย invariant หลักของ Vec ได้อย่างสมบูรณ์และเริ่มย้ายค่าออกจาก *ด้านหน้า* หรือแม้แต่ *ตรงกลาง* ของ Vec เหตุผลที่มัน *sound* เป็นเพราะ Drain iterator ที่เราคืนค่ามี `&mut` ไปที่ Vec ดังนั้นการเข้าถึงทั้งหมดจึงถูก gate ไว้เบื้องหลังมัน! ไม่มีใครสามารถสังเกต Vec ได้จนกว่า Drain iterator จะหายไป และจากนั้น destructor ของมันสามารถ "ซ่อม" Vec ได้ก่อนที่ใครจะสังเกตเห็น มันสมบูรณ์แบบ--

[มันไม่สมบูรณ์แบบ](https://doc.rust-lang.org/nightly/nomicon/leaking.html#drain) น่าเสียดาย คุณ[ไม่สามารถไว้ใจ destructor ในโค้ดที่คุณไม่ได้ควบคุมได้](https://doc.rust-lang.org/std/mem/fn.forget.html) และแม้แต่กับ Drain เราก็ต้องทำงานเพิ่มเติมอีกเล็กน้อยเพื่อให้ type ของเรา invariant ถูกสงวนไว้เสมอ แต่ในลักษณะที่แปลกประหลาด: [เราแค่ตั้ง len ของ Vec เป็น 0 ตอนเริ่มต้น](https://doc.rust-lang.org/std/mem/fn.forget.html) ดังนั้นถ้าใคร leak Drain พวกเขาจะได้ Vec ที่ *ปลอดภัย*... แต่พวกเขาจะสูญเสียข้อมูลจำนวนมาก คุณ leak ผม ผม leak คุณ! ตาต่อตา! ความยุติธรรมที่แท้จริง!

สำหรับสถานการณ์ที่คุณ *สามารถ* ใช้ destructor สำหรับความปลอดภัยต่อการแพนิกได้จริงๆ ลองดู [BinaryHeap::sift_up case study](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html#binaryheapsift_up)

อย่างไรก็ตาม เราจะไม่ต้องการสิ่งอำนวยความสะดวกเหล่านี้ทั้งหมดสำหรับ LinkedList ของเรา เราแค่ต้องตื่นตัวมากขึ้นเกี่ยวกับจุดที่เรา breaking invariants สิ่งที่เราไว้ใจ/ต้องการให้ถูกต้อง และหลีกเลี่ยงการ introductions unwind ที่ไม่จำเป็นในระหว่างงานที่ยุ่งยาก

ในกรณีนี้ เรามีสองทางเลือกเพื่อให้โค้ดของเราแข็งแกร่งขึ้นเล็กน้อย:

* ใช้การดำเนินการเช่น `Option::take` อย่างก้าวร้าวมากขึ้น เพราะมันมีลักษณะ "เป็นธุรกรรม" มากกว่าและมีแนวโน้มที่จะรักษา invariants ไว้

* ลบ `debug_asserts` ออกและไว้ใจตัวเองว่าจะเขียนการทดสอบที่ดีขึ้นด้วยฟังก์ชัน "integrity check" ที่จะไม่ทำงานในโค้ดของผู้ใช้เลย

โดยหลักการผมชอบทางเลือกแรก แต่มันไม่ได้ทำงานได้ดีนักสำหรับ doubly-linked list เพราะทุกอย่างถูก encode แบบ doubly-redundant `Option::take` จะไม่แก้ปัญหาที่นี่ แต่การย้าย `debug_assert` ลงมาบรรทัดเดียวจะแก้ได้ แต่จริงๆ แล้ว ทำไมต้องทำให้ตัวเองลำบาก? เรามาลบ `debug_asserts` เหล่านั้นออกและให้แน่ใจว่าสิ่งที่สามารถแพนิกได้อยู่ที่จุดเริ่มต้นหรือจุดสิ้นสุดของ method ของเรา ซึ่ง invariants ควรจะเป็นที่ทราบกันว่าเป็นจริง

(ในแง่นี้ อาจแม่นยำกว่าที่จะคิดว่ามันเป็น *preconditions* และ *postconditions* แต่จริงๆ แล้วคุณควรพยายามปฏิบัติต่อพวกเขาในฐานะ invariants ให้มากที่สุดเท่าที่จะเป็นไปได้!)

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

อะไรที่สามารถแพนิกที่นี่? การรู้เรื่องนี้ต้องใช้ความเป็นผู้เชี่ยวชาญ Rust พอสมควร แต่โชคดีที่ผมเป็น!

สิ่งเดียวที่ผมเห็นในโค้ดนี้ที่ *อาจ* สามารถแพนิกได้ (นอกจากเรื่องบ้าๆ บอๆ ที่มีคน recompile stdlib โดยเปิด debug_asserts แต่นี่ไม่ใช่สิ่งที่คุณควรทำเลย) คือ `Box::new` (สำหรับเงื่อนไข out-of-memory) และ len arithmetic ทั้งหมดนั้นอยู่ที่จุดสิ้นสุดหรือจุดเริ่มต้นของ method ของเรา ดังนั้น ใช่ เราปลอดภัยและดี!

...คุณประหลาดใจไหมที่ `Box::new` สามารถแพนิกได้? การแพนิกจะหลอกคุณแบบนี้! พยายามรักษา invariants เหล่านั้นไว้เพื่อที่คุณจะได้ไม่ต้องกังวลเรื่องนี้!
