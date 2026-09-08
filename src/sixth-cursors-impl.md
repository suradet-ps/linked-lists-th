# การ implements เคอร์เซอร์

โอเค ดังนั้นเราจะสนใจแค่ `CursorMut` ของ std เพราะเวอร์ชัน immutable ไม่ได้น่าสนใจจริงๆ เหมือนกับการออกแบบดั้งเดิมของผม มันมี "ghost" element ที่เก็บ `None` ไว้เพื่อบอกจุดเริ่มต้น/จุดสิ้นสุดของลิสต์ และคุณสามารถ "เดินข้ามมัน" เพื่อกลับไปยังอีกด้านหนึ่งของลิสต์ได้ ในการ implement มัน เราจะต้องการ:

* pointer ไปยัง node ปัจจุบัน
* pointer ไปยังลิสต์
* index ปัจจุบัน

เดี๋ยวก่อน index คืออะไรเมื่อเราชี้ไปที่ "ghost"?

*ขมวดคิ้ว* ... *ตรวจ std* ... *ไม่ชอบคำตอบของ std*

โอเค ดังนั้น `index` บน Cursor จะ return เป็น `Option<usize>` ซึ่งสมเหตุสมผลมาก implementation ของ std ทำอะไรมากมายเพื่อหลีกเลี่ยงการเก็บมันเป็น Option แต่... เราเป็น linked list มันก็โอเค นอกจากนี้ std ยังมี cursor_front/cursor_back ซึ่งเริ่มต้นเคอร์เซอร์ที่ element หน้า/หลัง ซึ่งรู้สึก intuitive แต่แล้วก็ต้องทำอะไรแปลกๆ เมื่อลิสต์ว่างเปล่า

คุณสามารถ implement อะไรพวกนั้นได้ถ้าอยาก แต่ผมจะตัดทอนความซ้ำซ้อนและกรณีขอบทั้งหมดออก และแค่สร้าง method `cursor_mut` เปล่าๆ ที่เริ่มต้นที่ ghost แล้วคนก็ใช้ `move_next`/`move_prev` เพื่อเลือกตัวที่ต้องการได้ (แล้วคุณก็ wrap มันเป็น `cursor_front` ได้ถ้าอยากจริงๆ)

มาเริ่มกันเลย:

```rust ,ignore
pub struct CursorMut<'a, T> {
    cur: Link<T>,
    list: &'a mut LinkedList<T>,
    index: Option<usize>,
}
```

ค่อนข้าง straightforward มี field หนึ่งตัวสำหรับแต่ละ item ในรายการที่เรากำหนดไว้! ตอนนี้มาดู method `cursor_mut`:

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

เนื่องจากเราเริ่มต้นที่ ghost เราสามารถเริ่มต้นด้วยทุกอย่างเป็น `None` ได้เลย ง่ายและสวย! ตอนนี้มาดูการเคลื่อนที่:


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

ดังนั้นมี 4 กรณีที่น่าสนใจ:

* กรณีปกติ
* กรณีปกติ แต่เราไปถึง ghost
* กรณี ghost ที่เราไปยังด้านหน้าของลิสต์
* กรณี ghost แต่ลิสต์ว่างเปล่า ดังนั้นไม่ต้องทำอะไร

`move_prev` มี logic เดียวกันเลย แค่กลับ front/back และกลับการเปลี่ยน index:

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

ตอนนี้มาเพิ่ม method สำหรับดู element รอบๆ เคอร์เซอร์: `current`, `peek_next`, และ `peek_prev` **หมายเหตุสำคัญมาก:** method เหล่านี้ต้องยืมเคอร์เซอร์ด้วย `&mut self` และผลลัพธ์ต้องเชื่อมกับการยืมนั้น เราไม่สามารถให้ผู้ใช้ได้หลายสำเนาของ mutable reference และเราไม่สามารถให้พวกเขาใช้ insert/remove/split/splice API ใดๆ ของเราในขณะที่ถือ reference นั้นอยู่!

โชคดีที่นี่คือสมมติฐานเริ่มต้นที่ Rust ทำเมื่อคุณใช้ lifetime elision ดังนั้นเราจะทำสิ่งที่ถูกต้องโดยค่าเริ่มต้น!

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

หัวว่างเปล่า, Option methods และ compiler errors (ที่ถูกตัดออก) ทำทุกอย่างให้ตอนนี้ ผมสงสัยเกี่ยวกับ `Option<NonNull>` เรื่องต่างๆ แต่ พระเจ้า มันช่วยให้ผม autopilot โค้ดนี้ได้จริงๆ ผมใช้เวลาไปมากเกินไปกับการเขียน array-based collections ที่คุณไม่เคยได้ใช้ Option ว้าว มันยอดมาก! (`(*node.as_ptr())` ยังคงน่าเบื่อ แต่นั่นก็เป็น raw pointers ของ Rust สำหรับคุณ...)

ตอนนี้เรามีทางเลือก: เราสามารถกระโดดไปที่ split และ splice ซึ่งเป็นจุดประสงค์ทั้งหมดของ API เหล่านี้ หรือเราจะก้าวเล็กๆ ด้วย insert/remove element เดี่ยว ผมรู้สึกว่าเราจะ implement insert/remove ในแง่ของ split และ splice ดังนั้น... มาทำ split และ splice ก่อนแล้วดูว่ามันจะเป็นยังไง (ผมไม่รู้จริงๆ ตอนที่พิมพ์นี้)




# Split

อันดับแรก, `split_before` และ `split_after` ซึ่ง return ทุกอย่างก่อน/หลัง element ปัจจุบันเป็น LinkedList (หยุดที่ ghost element เว้นแต่คุณจะอยู่ที่ ghost ซึ่งในกรณีนี้เราจะ return ทั้งลิสต์และเคอร์เซอร์ตอนนี้ชี้ไปที่ลิสต์ว่างเปล่า):

*หรี่ตา* โอเคอันนี้มี logic ที่ไม่ธรรมดาจริงๆ ดังนั้นเราจะต้องคุยกันทีละขั้นตอน

ผมเห็น 4 กรณีที่น่าสนใจสำหรับ `split_before`:

* กรณีปกติ
* กรณีปกติ แต่ prev เป็น ghost
* กรณี ghost ที่เรา return ทั้งลิสต์และกลายเป็นว่างเปล่า
* กรณี ghost แต่ลิสต์ว่างเปล่า ดังนั้นไม่ต้องทำอะไรและ return ลิสต์ว่างเปล่า

มาเริ่มที่กรณีขอบก่อน กรณีที่สามผมเชื่อว่าแค่

```rust
mem::replace(self.list, LinkedList::new())
```

ใช่ไหม? เราเป็นว่างเปล่า เรา return ทั้งลิสต์ และ field ของเราเป็น None อยู่แล้ว ดังนั้นไม่ต้องอัปเดตอะไร สวย! โอ้ย นี่ยังทำงานถูกต้องกับกรณีที่สี่ด้วย!

ดังนั้นตอนนี้มาดูกรณีปกติ... โอเค ผมต้องการแผนภาพ ASCII สำหรับเรื่องนี้ ในกรณีทั่วไปที่สุด เรามีอะไรแบบนี้:

```text
list.front -> A <-> B <-> C <-> D <- list.back
                          ^
                         cur
```

และเราต้องการผลลัพธ์แบบนี้:

```text
list.front -> C <-> D <- list.back
              ^
             cur

return.front -> A <-> B <- return.back
```

ดังนั้นเราต้องตัดลิงก์ระหว่าง cur และ prev และ... พระเจ้า มีอะไรมากมายที่ต้องเปลี่ยน โอเค ผมแค่ต้องแบ่งมันออกเป็นขั้นตอนเพื่อให้ผมมั่นใจว่ามันสมเหตุสมผล นี่จะเยอะเกินไปหน่อย แต่ผมอย่างน้อยก็ทำให้มันเข้าใจได้:

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

สังเกตว่า if-let นี้กำลังจัดการสถานการณ์ "กรณีปกติ แต่ prev เป็น ghost":

```rust ,ignore
if let Some(prev) = prev {
    (*cur.as_ptr()).front = None;
    (*prev.as_ptr()).back = None;
}
```

ถ้า *คุณ*อยาก คุณสามารถรวมมันเข้าด้วยกันและใช้ optimization ได้เช่น:

* พับการเข้าถึง `(*cur.as_ptr()).front` สองครั้งเป็นแค่ `(*cur.as_ptr()).front.take()`
* สังเกตว่า `new_back` เป็น noop และลบออกทั้งคู่

เท่าที่ผมรู้ ทุกอย่างที่เหลือบังเอิญทำงานถูกต้อง เราจะเห็นเมื่อเราเขียนทดสอบ! (copy-paste เพื่อสร้าง `split_after`)

ผมเสร็จสิ้นการทำความผิดพลาดแล้ว และผมจะแค่พยายามเขียนโค้ดที่ปลอดภัยที่สุดเท่าที่ผมจะทำได้ นี่คือวิธีที่ผม *actually* เขียน collections: แค่แบ่งสิ่งต่างๆ ออกเป็นขั้นตอนและกรณีเล็กๆ จนมันเข้าหัวผมได้และดูปลอดภัย แล้วเขียนทดสอบจำนวนมากจนผมมั่นใจว่าผมไม่ได้ทำมันพัง

เพราะว่างาน collections ส่วนใหญ่ที่ผมทำนั้น *extremely unsafe* ผมโดยทั่วไปไม่สามารถพึ่งพา compiler ในการจับข้อผิดพลาดได้ และ miri ไม่มีอยู่ในสมัยก่อน! ดังนั้นผมแค่ต้องหรี่ตาดูปัญหาจนหัวปวดและพยายามอย่างเต็มที่เพื่อ *ไม่เคยทำข้อผิดพลาด*

อย่าเขียน Unsafe Rust Code! Safe Rust ดีกว่ามาก!!!!




# Splice

บอสตัวสุดท้ายที่ต้องสู้, `splice_before` และ `splice_after` ซึ่งผมคาดว่าจะเป็นตัวที่มีกรณีขอบมากที่สุด สองฟังก์ชันนี้ *รับ* LinkedList เข้ามาและต่อเนื้อหาของมันเข้ากับของเรา ลิสต์ของเราอาจว่างเปล่า ลิสต์ของพวกเขาอาจว่างเปล่า เรามี ghost ที่ต้องจัดการ... *ถอนหายใจ* มาทำทีละขั้นตอนกับ `splice_before`

* ถ้าลิสต์ของพวกเขาว่างเปล่า เราไม่ต้องทำอะไร
* ถ้าลิสต์ของเราว่างเปล่า ลิสต์ของเราก็จะกลายเป็นลิสต์ของพวกเขา
* ถ้าเราชี้ไปที่ ghost, นี่จะต่อท้ายด้านหลัง (เปลี่ยน list.back)
* ถ้าเราชี้ไปที่ element แรก (0), นี่จะต่อท้ายด้านหน้า (เปลี่ยน list.front)
* ในกรณีทั่วไป เราทำ pointer manipulation เยอะมาก

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

โอเคไหม? โอเค มาเขียนมัน... *หายใจลึกและดำดิ่งลงไป*:

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

โอเค อันนี้น่ากลัวจริงๆ และรู้สึกถึงความเจ็บปวดของ `Option<NonNull>` ตอนนี้ แต่มียังมีอีกมาก cleanup ที่เราทำได้ อย่างแรก เราสามารถย้ายโค้ดนี้ไปท้ายสุด เพราะเราต้องการทำมันเสมอ ผมไม่ *ชอบ* (แม้ว่าบางครั้งมันจะเป็น noop และการตั้ง `input.len` เป็นเรื่องของความระมัดระวังเกี่ยวกับการขยายในอนาคตของโค้ด):

```rust ,ignore
self.list.len += input.len;
input.len = 0;
```

> Use of moved value: `input`

อ้อ ใช่ ในกรณี "เราเป็นว่างเปล่า" เรากำลังย้ายลิสต์ มาเปลี่ยนมันเป็น swap:

```rust ,ignore
// We're empty, become the input, remain on the ghost
std::mem::swap(self.list, &mut input);
```

ในกรณีนี้การเขียนจะไม่มีประโยชน์ แต่มันยังทำงานได้ (เราสามารถ return ก่อนใน branch นี้เพื่อพอใจ compiler ก็ได้)

unwrap นี้เป็นผลมาจากการที่ผมคิด.case ย้อนกลับ และสามารถแก้ไขได้โดยให้ if-let ถามคำถามที่ถูกต้อง:

```rust ,ignore
if let Some(0) = self.index {

} else {
    let prev = (*cur.as_ptr()).front.unwrap();
}
```

การปรับ index ซ้ำกันภายใน branches ดังนั้นสามารถย้ายออกได้:

```rust
*self.index.as_mut().unwrap() += input.len;
```

โอเค รวมทุกอย่างเข้าด้วยกันเราจะได้แบบนี้:

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

โอเค นี่ยังแย่อยู่ แต่ส่วนใหญ่เป็นเพราะ... ไม่ โอเค เจอ bug แล้ว:

```rust
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
```

เรา `take` `input.front` แล้ว unwrap มันในบรรทัดถัดไป! *ถอนหายใจ* และเราทำแบบเดียวกันในกรณี mirror ที่เทียบเคียงกัน เราจะจับมันได้ทันทีในการทดสอบ แต่เราพยายามจะสมบูรณ์แบบตอนนี้ และผมกำลังทำแบบ live อยู่ และนี่คือช่วงเวลาที่ผมเห็นมัน นี่คือสิ่งที่ผมได้รับจากการที่ไม่ได้เป็นตัวผมที่น่าเบื่อตามปกติและทำสิ่งต่างๆ เป็นขั้นตอน ชัดเจนมากขึ้น!

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

โอเค ตอนนี้อันนี้ อันนี้ผมทนได้ ข้อร้องเรียนเดียวที่ผมมีคือเราไม่ dedupe `in_front`/`in_back` (อาจจะปรับเงื่อนไขของเราได้แต่ช่างเถอะ) จริงๆ แล้วนี่โดยพื้นฐานเป็นสิ่งที่คุณจะเขียนใน C แต่ด้วย `Option<NonNull>` ที่ทำให้มันน่าเบื่อ ผมอยู่กับมันได้ แต่ไม่ เราควรจะทำ raw pointers ให้ดีขึ้นสำหรับเรื่องพวกนี้ แต่นอกเหนือจากขอบเขตของหนังสือเล่มนี้

ยังไงก็ตาม ผมหมดแรงจริงๆ หลังจากนั้น ดังนั้น `insert` และ `remove` และ API อื่นๆ ทั้งหมดสามารถทิ้งไว้เป็นแบบฝึกหัดสำหรับผู้อ่านได้

นี่คือโค้ดสุดท้ายสำหรับ Cursor ของเรากับความพยายามของผมในการ copy-paste combinatorics ผมทำถูกไหม? ผมจะรู้ก็ต่อเมื่อผมเขียนบทถัดไปและทดสอบความน่ากลัวนี้!

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
