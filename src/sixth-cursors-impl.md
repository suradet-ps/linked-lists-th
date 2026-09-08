# การเขียนเคอร์เซอร์

โอเค เราจะสนใจแค่ `CursorMut` ของ standard library เท่านั้นนะครับ เพราะเวอร์ชัน immutable (อ่านอย่างเดียว) มันไม่ได้มีอะไรน่าสนใจเลย เฉกเช่นเดียวกับดีไซน์ดั้งเดิมของผม มันจะมีสมาชิก "ผี" (ghost element) ตัวหนึ่งที่เก็บค่า `None` ไว้เพื่อบ่งบอกหัว/ท้ายของลิสต์ และคุณสามารถ "เดินก้าวข้ามมัน" เพื่อวนรอบกลับไปยังอีกฝั่งหนึ่งของลิสต์ได้ ในการจะอิมพลีเมนต์มัน เราจำเป็นต้องมีสิ่งเหล่านี้:

* พอยน์เตอร์ชี้ไปยังโหนดปัจจุบัน
* พอยน์เตอร์อ้างอิงไปยังตัวลิสต์
* ดัชนี (index) ปัจจุบัน

เดี๋ยวนะ แล้วค่า index จะเป็นอะไรเวลาที่เราชี้ไปที่ "โหนดผี" ล่ะ?

*ขมวดคิ้ว* ... *เปิดดูโค้ดของ std* ... *ไม่ชอบคำตอบของ std เอาซะเลย*

โอเค อย่างที่พอจะคาดเดาได้อย่างสมเหตุสมผล เมธอด `index` บน Cursor จะส่งค่ากลับมาเป็น `Option<usize>` ในโค้ดอิมพลีเมนต์ของ `std` มีการเล่นท่ายากสารพัดเพื่อหลีกเลี่ยงการเก็บมันเป็น `Option` แต่... ช่างเถอะ ลิสต์เราเป็น linked list จะเก็บเป็น Option ก็ไม่เสียหายอะไร นอกจากนี้ใน `std` ยังมีเมธอดอย่าง `cursor_front` / `cursor_back` ที่จะเริ่มวางเคอร์เซอร์ไว้ที่สมาชิกตัวหน้าสุด/หลังสุด ซึ่งก็ฟังดูเข้าใจง่ายดี แต่ดันต้องไปเขียนท่าพิสดารจัดการเวลารายการว่างเปล่าอีก

ถ้าคุณอยากจะเขียนเมธอดพวกนั้นเล่นเองก็ตามสบายนะครับ แต่ผมขอตัดความซ้ำซ้อนยิบย่อยและเคสขอบพวกนั้นทิ้งไปให้หมด แล้วทำแค่เมธอด `cursor_mut` เพียว ๆ ที่เริ่มวางเคอร์เซอร์ไว้ที่โหนดผี จากนั้นใครอยากได้ตำแหน่งไหนก็เรียก `move_next` / `move_prev` เอาเอง (ถ้าอยากได้จริง ๆ ค่อยเอาไปห่อเป็น `cursor_front` ทีหลังก็ยังได้)

มาลุยกันเลยครับ:

```rust ,ignore
pub struct CursorMut<'a, T> {
    cur: Link<T>,
    list: &'a mut LinkedList<T>,
    index: Option<usize>,
}
```

ค่อนข้างตรงไปตรงมา มีฟิลด์หนึ่งตัวสำหรับแต่ละข้อในรายการด้านบน! คราวนี้มาดูเมธอด `cursor_mut`:

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}
```

ในเมื่อเราเริ่มจากโหนดผี เราก็เริ่มทุกอย่างด้วย `None` ได้เลย ง่ายและสะอาดตาดี! ต่อไปคือการขยับตำแหน่งเคอร์เซอร์:


```rust ,ignore
impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }
}
```

ดังนั้นเราจึงมี 4 เคสที่น่าสนใจ:

* เคสปกติ
* เคสปกติ แต่เดินไปจนถึงโหนดผี
* เคสโหนดผี ที่เราก้าวไปยังหน้าสุดของลิสต์
* เคสโหนดผี แต่ลิสต์ดันว่างเปล่า... ก็ไม่ต้องทำอะไร

ส่วน `move_prev` ก็ใช้ตรรกะเดียวกันเป๊ะ แค่สลับ `front` กับ `back` และกลับทิศทางการคำนวณ index:

```rust ,ignore
pub fn move_prev(&mut self) {
    if let Some(cur) = self.cur {
        unsafe {
            // We're on a real element, go to its previous (front)
            self.cur = (*cur.as_ptr()).front;
            if self.cur.is_some() {
                *self.index.as_mut().unwrap() -= 1;
            } else {
                // We just walked to the ghost, no more index
                self.index = None;
            }
        }
    } else if !self.list.is_empty() {
        // We're at the ghost, and there is a real back, so move to it!
        self.cur = self.list.back;
        self.index = Some(self.list.len - 1)
    } else {
        // We're at the ghost, but that's the only element... do nothing.
    }
}
```

ถัดไป เรามาเพิ่มเมธอดสำหรับแอบส่องสมาชิกที่อยู่รอบ ๆ ตัวเคอร์เซอร์กัน: `current`, `peek_next`, และ `peek_prev` **หมายเหตุสำคัญยิ่ง:** เมธอดเหล่านี้จะต้องยืมตัวเคอร์เซอร์ผ่าน `&mut self` และผลลัพธ์ที่ได้จะต้องถูกผูกเข้ากับการยืมนั้น เราจะยอมให้ผู้ใช้ได้ `&mut` อ้างอิงตัวแปรเดียวกันหลายตัวไม่ได้เด็ดขาด และเราจะปล่อยให้พวกเขาแอบเรียกใช้ API อย่าง `insert`/`remove`/`split`/`splice` ในขณะที่ยังถือ reference นั้นค้างอยู่ไม่ได้เป็นอันขาด!

โชคดีที่นี่คือข้อสมมติฐานตามธรรมชาติที่ Rust ยึดถืออยู่แล้วเมื่อคุณใช้กฎ lifetime elision ดังนั้นโค้ดของเราจึงประพฤติตัวได้อย่างถูกต้องปลอดภัยโดยอัตโนมัติ!

```rust ,ignore
pub fn current(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

สมองโล่งปลอดโปร่ง เมธอดของ Option กับสารพัด compiler error (ที่ขอละไว้ในฐานที่เข้าใจ) ทำหน้าที่คิดแทนผมหมดแล้ว ตอนแรกผมยังกังขาเรื่องการใช้ `Option<NonNull>` อยู่เลยนะ แต่ให้ตายสิ มันช่วยให้ผมเปิดโหมดออโต้ไพลอตเขียนโค้ดชุดนี้ได้อย่างไหลลื่นจริง ๆ ผมคงเสียเวลาไปกับการเขียนคอลเลกชันบนอาเรย์นานเกินไปจนลืมไปแล้วว่าการได้ใช้ `Option` มันฟินขนาดนี้ ว้าว มันดีต่อใจจริง ๆ! (ถึงแม้ `(*node.as_ptr())` จะยังชวนหดหู่เหมือนเดิมก็เถอะ แต่นั่นแหละคือธรรมชาติของ raw pointer ใน Rust...)

ต่อไปเรามีทางเลือกสองทาง: เราจะกระโดดข้ามไปทำ `split` กับ `splice` ซึ่งเป็นเป้าหมายหลักของการมี API พวกนี้เลย หรือจะค่อย ๆ ก้าวทีละขั้นด้วยการทำ `insert`/`remove` ทีละตัวก่อนดี ผมมีลางสังหรณ์ว่าสุดท้ายเราคงอยากอิมพลีเมนต์ `insert`/`remove` บนฐานของ `split` กับ `splice` อยู่ดี... งั้นเรามาลุยสองตัวนี้ก่อนเลยละกัน แล้วค่อยดูว่าไพ่จะเปิดออกมาหน้าไหน (ตอนที่กำลังพิมพ์อยู่นี่ผมก็ยังไม่รู้จริง ๆ เหมือนกันนะเนี่ย)




# Split

อันดับแรก `split_before` และ `split_after` ซึ่งจะส่งคืนสมาชิกทั้งหมดที่อยู่ก่อนหน้า/ข้างหลังสมาชิกปัจจุบันออกมาเป็น `LinkedList` (โดยหยุดที่โหนดผี เว้นแต่ว่าคุณจะยืนอยู่ที่โหนดผีอยู่แล้ว ซึ่งในกรณีนั้นเราจะส่งคืนลิสต์ทั้งหมดออกมา แล้วตัวเคอร์เซอร์จะชี้ไปยังลิสต์ว่างเปล่าแทน):

*หรี่ตามอง* โอเค ตรรกะของฟังก์ชันนี้ค่อนข้างซับซ้อนไม่ธรรมดาเลยแฮะ ดังนั้นเราคงต้องมาค่อย ๆ คุยกันทีละขั้นตอน

ผมเห็นเคสที่น่าสนใจอยู่ 4 เคสสำหรับ `split_before`:

* เคสปกติ
* เคสปกติ แต่ `prev` ดันเป็นโหนดผี
* เคสโหนดผี ที่เราส่งคืนลิสต์ทั้งหมดออกมา แล้วตัวเรากลายเป็นลิสต์ว่างเปล่า
* เคสโหนดผี แต่ตัวลิสต์ว่างเปล่าอยู่แล้ว... ก็ไม่ต้องทำอะไร แล้วส่งคืนลิสต์ว่างเปล่ากลับไป

มาเริ่มจากเคสขอบกันก่อน เคสที่สามผมเชื่อว่ามันก็แค่:

```rust
mem::replace(self.list, LinkedList::new())
```

ใช่ไหมล่ะครับ? เรากลายเป็นลิสต์ว่างเปล่า เราส่งคืนลิสต์ทั้งหมดออกมา และฟิลด์เดิมของเราก็เป็น `None` อยู่แล้วด้วย จึงไม่มีอะไรต้องอัปเดตเลย สวยงาม! อ้าวเฮ้ย บรรทัดนี้ดันทำงานได้ถูกต้องสมบูรณ์แบบกับเคสที่สี่ไปด้วยในตัวเลยนี่หว่า!

คราวนี้มาดูกลุ่มเคสปกติกันบ้าง... โอเค ผมคงต้องขอวาดแผนภาพ ASCII ประกอบสักหน่อย ในเคสทั่ว ๆ ไปที่สุด เราจะมีหน้าตาประมาณนี้:

```text
list.front -> A <-> B <-> C <-> D <- list.back
                          ^
                         cur
```

และเราต้องการแปลงให้กลายเป็นแบบนี้:

```text
list.front -> C <-> D <- list.back
              ^
             cur

return.front -> A <-> B <- return.back
```

ดังนั้นเราจึงต้องตัดสายสัมพันธ์เชื่อมโยงระหว่าง `cur` กับ `prev` และ... โอย มีของที่ต้องอัปเดตเยอะชะมัด โอเค ผมขอแบ่งมันออกเป็นขั้นตอนย่อย ๆ เพื่อให้ตัวผมเองมั่นใจว่ามันเมคเซนส์ โค้ดอาจจะดูเยิ่นเย้อไปสักนิด แต่อย่างน้อยมันก็ทำให้ผมเข้าใจได้ไม่หลงทางครับ:

```rust ,ignore
pub fn split_before(&mut self) -> LinkedList<T> {
    if let Some(cur) = self.cur {
        // We are pointing at a real element, so the list is non-empty.
        unsafe {
            // Current state
            let old_len = self.list.len;
            let old_idx = self.index.unwrap();
            let prev = (*cur.as_ptr()).front;
            
            // What self will become
            let new_len = old_len - old_idx;
            let new_front = self.cur;
            let new_back = self.list.back;
            let new_idx = Some(0);

            // What the output will become
            let output_len = old_len - new_len;
            let output_front = self.list.front;
            let output_back = prev;

            // Break the links between cur and prev
            if let Some(prev) = prev {
                (*cur.as_ptr()).front = None;
                (*prev.as_ptr()).back = None;
            }

            // Produce the result:
            self.list.len = new_len;
            self.list.front = new_front;
            self.list.back = new_back;
            self.index = new_idx;

            LinkedList {
                front: output_front,
                back: output_back,
                len: output_len,
                _boo: PhantomData,
            }
        }
    } else {
        // We're at the ghost, just replace our list with an empty one.
        // No other state needs to be changed.
        std::mem::replace(self.list, LinkedList::new())
    }
}
```

สังเกตว่าท่อน `if let` ตรงนี้ถูกเขียนขึ้นมาเพื่อรับมือกับสถานการณ์ "เคสปกติ แต่ `prev` เป็นโหนดผี" โดยเฉพาะ:

```rust ,ignore
if let Some(prev) = prev {
    (*cur.as_ptr()).front = None;
    (*prev.as_ptr()).back = None;
}
```

ถ้า *คุณ* อยากจะปรับโค้ดให้กระชับขึ้นและใส่ท่า optimize เพิ่มเติม คุณก็ทำได้นะครับ เช่น:

* ยุบการเข้าถึง `(*cur.as_ptr()).front` สองรอบ ให้เหลือแค่ `(*cur.as_ptr()).front.take()`
* สังเกตว่า `new_back` ไม่ได้ทำอะไรเปลี่ยนไปเลย ก็ตัดตัวแปรทั้งคู่ออกไปได้

เท่าที่ผมดู ส่วนอื่น ๆ ทั้งหมดมันก็บังเอิญทำงานได้อย่างถูกต้องตรงตามเป้าหมายของมันดีอยู่แล้ว เดี๋ยวเราค่อยไปดูกันตอนเขียนเทสต์อีกที! (แล้วก็ก็อปแปะโค้ดชุดนี้ไปสร้าง `split_after` ซะ)

ผมพอแล้วกับการทำผิดพลาด และผมจะพยายามเขียนโค้ดที่รัดกุมรอบคอบที่สุดเท่าที่จะทำได้ นี่คือวิธีที่ผมเขียนคอลเลกชัน *จริง ๆ*: แค่ย่อยปัญหาออกเป็นขั้นตอนและกรณีเล็ก ๆ ที่ไม่ซับซ้อน จนมันสามารถจุลงในหัวผมได้และมั่นใจว่าไม่มีช่องโหว่ จากนั้นก็กระหน่ำเขียนเทสต์ชุดใหญ่จนกว่าจะแน่ใจว่าผมไม่ได้แอบทำพังตรงไหน

เนื่องจากงานคอลเลกชันส่วนใหญ่ที่ผมเคยทำมัน *unsafe แบบสุดกู่* ผมจึงแทบไม่เคยได้พึ่งพาให้คอมไพเลอร์ช่วยดักจับความผิดพลาดเลย แถมในยุคนั้น miri ก็ยังไม่ถือกำเนิดขึ้นมาด้วยซ้ำ! ผมก็เลยต้องนั่งหรี่ตามองปัญหาจนปวดกบาล และพยายามทำทุกวิถีทางเพื่อที่จะ ไม่ มี วัน ทำ พลาด เด็ด ขาด

อย่าเขียนโค้ด Unsafe ใน Rust เลยครับ! โค้ด Safe ใน Rust มันดีกว่ากันเยอะมากจริง ๆ!!!!




# Splice

เหลือบอสตัวสุดท้ายที่ต้องปราบนั่นคือ `splice_before` และ `splice_after` ซึ่งผมคาดว่าจะเป็นจุดที่เคสขอบยุบยิบที่สุดในบรรดาฟังก์ชันทั้งหมด ทั้งสองฟังก์ชันนี้จะ *รับ* `LinkedList` เข้ามา แล้วนำเนื้อหาข้างในมาเสียบต่อเข้ากับลิสต์ของเรา ลิสต์เราอาจจะว่างเปล่า ลิสต์เขาก็อาจจะว่างเปล่า แถมยังมีโหนดผีที่ต้องคอยตามเช็ดตามล้างอีก... *ถอนหายใจยาววว* งั้นเรามาค่อย ๆ ทำไปทีละขั้นกับ `splice_before` กันครับ

* ถ้าลิสต์ของเขาว่างเปล่า เราก็ไม่ต้องทำอะไรเลย
* ถ้าลิสต์ของเราว่างเปล่า ลิสต์ของเราก็จะกลายสภาพเป็นลิสต์ของเขาไปเลย
* ถ้าเรากำลังชี้ไปที่โหนดผี นี่จะเป็นการต่อท้ายด้านหลัง (แก้ไข `list.back`)
* ถ้าเรากำลังชี้ไปที่สมาชิกตัวแรก (0) นี่จะเป็นการแทรกต่อข้างหน้า (แก้ไข `list.front`)
* ในเคสทั่ว ๆ ไป เราจะต้องสลับร่างพอยน์เตอร์กันอุตลุด

กรณีทั่วไปคือแบบนี้:

```text
input.front -> 1 <-> 2 <- input.back

 list.front -> A <-> B <-> C <- list.back
                     ^
                    cur
```

กลายเป็นแบบนี้:

```text
list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
```

เข้าใจตรงกันนะ? โอเค มาเขียนกันเลย... *สูดหายใจเข้าลึก ๆ แล้วดำดิ่งลงไป*:

```rust ,ignore
    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                if let Some(0) = self.index {
                    // We're appending to the front, see append to back
                    (*cur.as_ptr()).front = input.back.take();
                    (*input.back.unwrap().as_ptr()).back = Some(cur);
                    self.list.front = input.front.take();

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                } else {
                    // General Case, no boundaries, just internal fixups
                    let prev = (*cur.as_ptr()).front.unwrap();
                    let in_front = input.front.take().unwrap();
                    let in_back = input.back.take().unwrap();

                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                }
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                // We can either `take` the input's pointers or `mem::forget`
                // it. Using take is more responsible in case we do custom
                // allocators or something that also needs to be cleaned up!
                (*back.as_ptr()).back = input.front.take();
                (*input.front.unwrap().as_ptr()).front = Some(back);
                self.list.back = input.back.take();
                self.list.len += input.len;
                // Not necessary but Polite To Do
                input.len = 0;
            } else {
                // We're empty, become the input, remain on the ghost
                *self.list = input;
            }
        }
    }
```

โอเค โค้ดชุดนี้น่าสยดสยองอย่างแท้จริง และทำให้รู้สึกซาบซึ้งถึงความเจ็บปวดของการใช้ `Option<NonNull>` ขึ้นมาทันที แต่ยังมีจุดให้เราเก็บกวาดทำความสะอาดได้อีกเพียบ อย่างแรกคือเราสามารถดึงโค้ดชุดนี้ออกไปไว้ที่ท้ายสุดได้ เพราะยังไงเราก็ต้องทำมันเสมออยู่แล้ว แม้ผมจะไม่ได้ *ปลื้ม* มันเท่าไหร่ก็เถอะ (ถึงแม้บางครั้งมันจะเป็น no-op และการรีเซ็ต `input.len` ก็เป็นแค่ความระแวงเผื่ออนาคตหากมีการขยายโค้ดต่อ):

```rust ,ignore
self.list.len += input.len;
input.len = 0;
```

> Use of moved value: `input`

อ้อ ใช่แล้ว ในเคส "ลิสต์เราว่างเปล่า" เรากำลังย้าย (move) ค่าของลิสต์เข้ามา งั้นเปลี่ยนมาใช้การ `swap` แทนดีกว่า:

```rust ,ignore
// We're empty, become the input, remain on the ghost
std::mem::swap(self.list, &mut input);
```

ในกรณีนี้ การเขียนค่าทับจะไร้ประโยชน์ก็จริง แต่มันก็ยังทำงานได้ (หรือเราจะสั่ง early-return ในกิ่งนี้เพื่อเอาใจคอมไพเลอร์ก็ได้นะ)

ส่วนการ `unwrap` ตรงนี้เป็นผลพลอยได้จากการที่ผมคิดเคสกลับทิศกลับทางไปหน่อย ซึ่งแก้ได้ง่าย ๆ แค่ปรับ `if let` ให้ถามคำถามที่ตรงจุด:

```rust ,ignore
if let Some(0) = self.index {

} else {
    let prev = (*cur.as_ptr()).front.unwrap();
}
```

และการปรับค่า index มันมีเขียนซ้ำซ้อนกันในหลายกิ่งย่อย ดังนั้นเราจึงยกมันออกมารวมกันข้างนอกได้เลย:

```rust
*self.index.as_mut().unwrap() += input.len;
```

โอเค พอนำทุกอย่างมารวมเข้าด้วยกัน เราก็จะได้แบบนี้:

```rust
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        let in_front = input.front.take().unwrap();
        let in_back = input.back.take().unwrap();

        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // We're appending to the front, see append to back below
        (*cur.as_ptr()).front = input.back.take();
        (*input.back.unwrap().as_ptr()).back = Some(cur);
        self.list.front = input.front.take();
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    // We can either `take` the input's pointers or `mem::forget`
    // it. Using take is more responsible in case we do custom
    // allocators or something that also needs to be cleaned up!
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
    self.list.back = input.back.take();

} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

เอาล่ะ ถึงมันจะยังดูทุเรศอยู่ แต่ส่วนใหญ่เป็นเพราะ... เดี๋ยว ๆๆๆ ชิบหายละ เพิ่งเหลือบไปเห็นบั๊กเต็มตาเลย:

```rust
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
```

เราดันไป `take` ค่าของ `input.front` แล้วไปสั่ง `unwrap` มันในบรรทัดถัดไปเนี่ยนะ! *เฮ้ออออ* แถมเรายังทำแบบเดียวกันเป๊ะในเคสกระจกเงาอีกฝั่งหนึ่งด้วย! ถ้าเขียนเทสต์เราคงดักจับเรื่องนี้ได้ในพริบตาอยู่แล้ว แต่ในเมื่อตอนนี้เราตั้งเป้าว่าจะเป็นคนไร้ที่ติ และผมก็เขียนโค้ดสด ๆ แบบเรียลไทม์อยู่ตรงนี้ นี่คือวินาทีที่ผมสังเกตเห็นมันพอดี นี่แหละคือผลกรรมของการไม่ยอมเป็นคนน่าเบื่อตามปกติที่ค่อย ๆ ทำทีละขั้น คราวนี้ต้องเขียนให้ชัดเจนตรงไปตรงมาขึ้น!

```rust
// We can either `take` the input's pointers or `mem::forget`
// it. Using `take` is more responsible in case we ever do custom
// allocators or something that also needs to be cleaned up!
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // No prev, we're appending to the front
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
        self.list.front = Some(in_front);
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    (*back.as_ptr()).back = Some(in_front);
    (*in_front.as_ptr()).front = Some(back);
    self.list.back = Some(in_back);
} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

โอเค พอเป็นแบบนี้... แบบนี้แหละที่ผมพอจะรับได้ ข้อติติงเดียวที่ผมยังมีอยู่ก็คือเราไม่ได้ลบความซ้ำซ้อนของ `in_front`/`in_back` (จริง ๆ คงพอจัดระเบียบเงื่อนไขใหม่ได้ แต่ช่างหัวมันเถอะ) เอาจริง ๆ โค้ดชุดนี้ก็คือสิ่งที่คุณจะเขียนในภาษา C นั่นแหละ เพียงแต่มีขยะของ `Option<NonNull>` มาทำให้ชีวิตมันน่าเบื่อขึ้นเฉย ๆ ซึ่งผมยอมทนอยู่กับมันได้ (แต่ก็นั่นแหละ จริง ๆ Rust ควรพัฒนา raw pointer ให้จัดการเรื่องพวกนี้ได้ดีกว่านี้ แต่ก็ถือว่าอยู่นอกเหนือขอบเขตของหนังสือเล่มนี้ละกัน)

เอาเป็นว่า หลังจากผ่านศึกนี้มาได้ ผมหมดเรี่ยวหมดแรงเกลี้ยงแล้วครับ ดังนั้นเมธอดอย่าง `insert`, `remove`, และ API อื่น ๆ ที่เหลือทั้งหมด ผมขอยกให้เป็นแบบฝึกหัดสำหรับผู้อ่านไปทำต่อเองละกันนะครับ

และนี่คือโค้ดสุดท้ายของ Cursor ของเรา พร้อมกับความพยายามในการก็อปแปะสารพัดคอมบิเนชันลงไป แล้วผมทำถูกหรือเปล่าเนี่ย? เดี๋ยวคงได้รู้คำตอบตอนที่เขียนบทถัดไปแล้วจับสัตว์ประหลาดตัวนี้มาเทสต์นั่นแหละครับ!

```rust ,ignore
pub struct CursorMut<'a, T> {
    list: &'a mut LinkedList<T>,
    cur: Link<T>,
    index: Option<usize>,
}

impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}

impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its previous (front)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real back, so move to it!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn current(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).front)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn split_before(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                               ^
        //                              cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> C <-> D <- list.back
        //                   ^
        //                  cur
        //
        // 
        //    return.front -> A <-> B <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let prev = (*cur.as_ptr()).front;
                
                // What self will become
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = self.list.front;
                let output_back = prev;

                // Break the links between cur and prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn split_after(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                         ^
        //                        cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> A <-> B <- list.back
        //                         ^
        //                        cur
        //
        // 
        //    return.front -> C <-> D <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let next = (*cur.as_ptr()).back;
                
                // What self will become
                let new_len = old_idx + 1;
                let new_back = self.cur;
                let new_front = self.list.front;
                let new_idx = Some(old_idx);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = next;
                let output_back = self.list.back;

                // Break the links between cur and next
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
        //                                 ^
        //                                cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // General Case, no boundaries, just internal fixups
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // No prev, we're appending to the front
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // Index moves forward by input length
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }

    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> B <-> 1 <-> 2 <-> C <- list.back
        //                     ^
        //                    cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // General Case, no boundaries, just internal fixups
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // No next, we're appending to the back
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
                // Index doesn't change
            } else if let Some(front) = self.list.front {
                // We're on the ghost but non-empty, append to the front
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }
}
```
