# บทนำเกี่ยวกับ Cursor

OK!!! ตอนนี้เรามี LinkedList ที่เทียบเท่ากับ std version 1.0 แล้ว! ซึ่งแน่นอนว่าหมายความว่า LinkedList ของเรายัง *ไม่มีประโยชน์อยู่ดี* เราได้รับผลกระทบด้านประสิทธิภาพมหาศาลจากการ implement Deque ในรูปแบบ linked list **แต่เรายังไม่มี API ที่ทำให้มันใช้งานได้จริง**

นี่คือสถานะของเราเมื่อเทียบกับ "killer apps" ของ linked lists:

* 🚫 สามารถทำ [stuff แปลกๆ](https://docs.rs/linked-hash-map/latest/linked_hash_map/)
* 🚫 สามารถทำ [stuff แปลกๆ แบบ lockfree](https://doc.rust-lang.org/std/sync/mpsc/)
* 🚫 สามารถเก็บ [Dynamically Sized Types](https://doc.rust-lang.org/nomicon/exotic-sizes.html#dynamically-sized-types-dsts)
* 🌟 O(1) push/pop โดยไม่ต้อง [amortization](https://en.wikipedia.org/wiki/Amortized_analysis) (ถ้าคุณยอมเชื่อว่า malloc เป็น O(1))
* 🚫 O(1) list splitting
* 🚫 O(1) list splicing

ก็... 1 จาก 6 ก็... ยังดีกว่าไม่มีเลย! คุณเห็นไหมว่าทำไมผมถึงอยากเอาตัวนี้ออกจาก std?

เราจะไม่ทำให้ list รองรับ "stuff แปลกๆ" หรอก เพราะมันล้วนเป็นแบบ adhoc และ domain-specific แต่เรื่อง splitting และ splicing นี่ล่ะ ที่เราทำได้!

แต่ปัญหาคือ จริงๆ แล้วการ *เข้าถึง* สมาชิกค<sup>th</sup> ใน LinkedList ใช้เวลา O(k) ดังนั้นเราจะ *เป็นไปได้อย่างไร* ที่จะทำ split และ merge ตามอำเภอใจใน O(1)? เคล็ดลับคือคุณไม่มี API แบบ `split_at(index)` -- คุณสร้างระบบที่ให้ผู้ใช้สามารถ iterate แบบ statefully ไปยังตำแหน่งหนึ่งใน list และทำการแก้ไข O(1) ณ จุดนั้นได้!

เฮ้ เรามี iterator อยู่แล้วนี่! เราใช้มันได้ไหม? ก็... ได้บางส่วน แต่ super-power ตัวหนึ่งของมันขวางทางเราอยู่ คุณอาจจำได้ว่าการเขียน lifetime สำหรับ iterator แบบ by-ref หมายความว่า reference ที่มันคืนมา *ไม่ได้* ผูกกับ iterator ตัวนั้น ทำให้เราเรียก `next` ซ้ำๆ และเก็บ element ไว้ได้:

```rust ,ignore
let mut list = ...;
let iter = list.iter_mut();
let elem1 = list.next();
let elem2 = list.next();

if elem1 == elem2 { ... }
```

ถ้า reference ที่คืนมา borrowing iterator แล้ว โค้ดนี้จะไม่ทำงานเลย Compiler จะบ่นเกี่ยวกับการเรียก `next` ครั้งที่สองแน่ๆ! ความยืดหยุ่นนี้เยี่ยมมาก แต่มันก็สร้างข้อจำกัดบางอย่างให้เราโดยนัย:

* Iterator แบบ By-Mutable-Ref ไม่สามารถย้อนกลับและ yield element เดิมซ้ำได้ เพราะผู้ใช้จะได้รับ `&mut` สองตัวไปยัง element เดียวกัน ซึ่งละเมิดกฎพื้นฐานของภาษา

* Iterator แบบ By-Ref ไม่สามารถมี method เพิ่มเติมที่อาจแก้ไข collection ที่อยู่เบื้องล่างในลักษณะที่จะทำให้ reference ที่ yield ออกไปแล้วใช้ไม่ได้

โชคร้ายที่ทั้งสองสิ่งนี้ *ตรงกันข้าม* กับสิ่งที่เราต้องการให้ LinkedList API ทำ! ดังนั้นเราจึงไม่สามารถใช้ iterator ได้ เราต้องการสิ่งใหม่: *Cursor*

Cursor เหมือนกับ `|` ที่กะพริบเวลาคุณแก้ไขข้อความบนคอมพิวเตอร์ มันคือตำแหน่งในลำดับ (ข้อความ) ที่คุณสามารถเคลื่อนย้ายไปมาได้ (ด้วยปุ่มลูกศร) และเมื่อคุณพิมพ์ แก้ไขก็จะเกิดขึ้น ณ จุดนั้น

ลองดู ถ้าผมแค่

กด

enter

ทั้งหมด

จะ

ถูกตัด

เป็นครึ่ง

ขอโทษนะ คุณยืนอยู่ข้างหลังผมและดูผมพิมพ์อยู่ใช่ไหม? ดังนั้นมันก็สมเหตุสมผลเลย ใช่ไหม? ใช่

ตอนนี้ ถ้าคุณเคยมีโชคร้ายที่มีคีย์บอร์ดที่มีปุ่ม "insert" และได้กดมันจริงๆ คุณจะรู้ว่า cursor มีการตีความสองแบบทางเทคนิค: มันสามารถอยู่ *ระหว่าง* สมาชิก (ตัวอักษร) หรืออยู่ *บน* สมาชิก ผมมั่นใจว่าไม่มีใครเคยกด "insert" ตั้งใจในชีวิต และมันมีอยู่เพื่อเป็นปุ่มทรมานล้วนๆ ดังนั้นมันจึงเห็นได้ชัดว่าแบบไหนที่ดีกว่าและถูกต้อง: cursor อยู่ *ระหว่าง* สมาชิก!

ตรรกะที่หนักแน่นมาก ผมคิดว่าไม่มีใครจะไม่เห็นด้วยกับผม

ขอโทษนะอะไรนะ? มี [RFC ในปี 2018 ที่จะเพิ่ม Cursor ให้กับ LinkedList ของ Rust](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md)?

> With a Cursor one can seek back and forth through a list and get the current element. With a CursorMut One can seek back and forth and get mutable references to elements, and it can insert and delete elements before and behind the current element (along with performing several list operations such as splitting and splicing).

*Current element*? Cursor ตัวนี้อยู่ *บน* สมาชิก ไม่ใช่ระหว่างพวกเขา! ผมไม่อยากเชื่อว่าพวกเขาไม่ยอมรับข้อถกเถียงที่หนักแน่นของผม! ดังนั้นคุณก็ใช้ Cursor ใน std ได้เลย... เดี๋ยวนะ นี่มัน [ปี 2022 และ Rust 1.60 ยังคงมี Cursor ที่ไม่ stable](https://doc.rust-lang.org/1.60.0/std/collections/linked_list/struct.CursorMut.html)?

เฮ้ เดี๋ยวก่อน:

> Cursors always rest between two elements in the list, and index in a logically circular way. To accommodate this, there is a "ghost" non-element that yields None between the head and tail of the list.

เฮ้ เดี๋ยวก่อน นี่ตรงข้ามกับที่ RFC บอกเลยนะ??? แต่เดี๋ยวก่อน documentation ทั้งหมดเกี่ยวกับ method ยังคงอ้างถึง "current" elements... เดี๋ยวก่อน ผมเคยเห็น ghost stuff แบบนี้ที่ไหนมาก่อนนะ? อ้อ เดี๋ยว ผมไม่ได้ทำแบบนั้นใน [linked-list fork เก่าของผม](https://docs.rs/linked-list/0.0.3/linked_list/struct.Cursor.html) ที่ผม prototyped ไว้หรอกหรือ?

> Cursors always rest between two elements in the list, and index in a logically circular way. To accomadate this, there is a "ghost" non-element that yields None between the head and tail of the List.

เดี๋ยว อะไรนะ มันไม่ใช่เรื่องตลก ผมกำลังอ่าน docs จริงๆ ตอนนี้ std RFC แตกต่างจากที่ผม propose ไว้ในปี 2015 จริงหรือ? แต่แล้วก็ copy-paste docs จาก prototype ของผม???? std กำลัง meta-shitpost ผมเรื่องที่ผมเขียนหนังสือเกี่ยวกับการเกลียด LinkedList????? เหมือนที่ผมสร้าง prototype เพื่อแสดงแนวคิดให้คนยอมรับให้ผมเพิ่มมันเข้า std และทำให้ LinkedList ไม่ไร้ประโยชน์ แต่ qu'est-ce que le fuck??????????????

_ok สิ่งที่ชัดเจนคือ std กำลังให้พร้อมกับ design ของผมว่าเป็นแบบที่ดีที่สุด ดังนั้นเราจะใช้ design ของผม นั่นดีมากเพราะทั้งบทนี้คือผมกำลังเขียนทับ library เดิมใหม่ทั้งหมด ดังนั้นไม่เปลี่ยน API ดูดีสำหรับผม!_

นี่คือ top-level docs ที่ผมเขียนทั้งหมด:

> A Cursor is like an iterator, except that it can freely seek back-and-forth, and can safely mutate the list during iteration. This is because the lifetime of its yielded references are tied to its own lifetime, instead of just the underlying list. This means cursors cannot yield multiple elements at once.
>
> Cursors always rest between two elements in the list, and index in a logically circular way. To accomadate this, there is a "ghost" non-element that yields None between the head and tail of the List.
>
> When created, cursors start between the ghost and the front of the list. That is, next will yield the front of the list, and prev will yield None. Calling prev again will yield the tail.

น่ารักนะ แม้ว่าเราจะสรุปว่า "sentinel-node" ทั้งหมดนั้นสร้างปัญหามากกว่าคุณค่า แต่เราจะยังคงได้รับ semantics ที่ "แกล้ง" ว่ามี sentinel node อยู่ เพื่อให้ cursor สามารถ wrap around ไปยังอีกด้านของ list ได้

*Skims over my old APIs some more*

```rust ,ignore
fn splice(&mut self, other: &mut LinkedList<T>)
```

> Inserts the entire list's contents right after the cursor.

อ้อ ใช่ ผมจำได้แล้ว ผมเขียนตอนที่โกรธ combinatoric explosion และพยายามหาทางให้มีสำเนาของแต่ละ operation เพียงชุดเดียว โชคร้ายที่มัน... มีปัญหาด้าน semantics ดูสิ เมื่อผู้ใช้ต้องการ splice list หนึ่งเข้าไปในอีก list หนึ่ง พวกเขาอาจต้องการให้ cursor อยู่ *ก่อน* splice หรือ *หลัง* splice list ที่แทรกเข้ามาอาจมีขนาดใหญ่เท่าใดก็ได้ ดังนั้นจึงเป็นปัญหาที่แท้จริงสำหรับเราที่จะอนุญาตเฉพาะทางเดียวและคาดว่าผู้ใช้จะเดินผ่าน list ทั้งหมดที่แทรกเข้ามา!

เราจะต้อง rework design นี้ใหม่ทั้งหมด Cursor type ของเราต้องมีอะไรบ้าง? มันต้อง:

* ชี้ "ระหว่าง" สองสมาชิก
* ในฐานะฟีเจอร์เล็กๆ ที่ดี ติดตามว่า "index" ใดถัดไป
* อัปเดต list เองเพื่อแก้ไข front/back/len

คุณจะชี้ระหว่างสองสมาชิกได้อย่างไร? ก็... คุณไม่ชี้ คุณแค่ชี้ไปที่ "next" member ดังนั้นแม้ว่าเราจะแสดง semantics "cursor อยู่ระหว่าง" แต่จริงๆ แล้วเรากำลัง implement มันแบบ "cursor อยู่บน" และแกล้งว่าทุกอย่างเกิดขึ้นก่อนหรือหลังจุดนั้น

แต่มีเหตุผล! กรณีการใช้งาน splice ต้องการให้ผู้ใช้เลือกว่าจะจบลงก่อนหรือหลัง list แต่มัน... *ซับซ้อนมาก* เมื่อแสดงด้วย std API! พวกเขามี splice_after และ splice_before แต่ไม่มีตัวไหนเปลี่ยนตำแหน่ง cursor ดังนั้นจริงๆ แล้วคุณต้องการ splice_after_before และ splice_after_after...

เดี๋ยวก่อน ผมไม่ได้กำลังไร้สาระ ใน std API คุณแค่เลือก node ที่คุณต้องการจบลง แล้วใช้ splice_after/before ตามความเหมาะสม

*squints*

เดี๋ยวก่อน std API ดีจริงๆ เหรอ?

*skims through the code*

_ok std API ดีจริงๆ_

เอาล่ะ ลืมมันไปเลย เราจะ [implement RFC](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md) หรืออย่างน้อยก็ส่วนที่น่าสนใจของมัน

ผมมีข้อสงสัยเกี่ยวกับคำศัพท์บางคำที่ std ใช้ แต่ cursor จะทำให้สมองละลายเสมอ: `iter().next_back()` ได้ `back()` ซึ่งดี แต่ `next_back()` ทุกครั้งถัดมาจะพาคุณ *เข้าใกล้ด้านหน้ามากขึ้น* และจริงๆ แล้ว pointer ทุกตัวที่เรา follow คือ pointer แบบ "front"! ถ้าผมคิดเกี่ยวกับความย้อนแย้งที่ดูเหมือนจะเป็นนี้มากเกินไป มันจะทำให้สมองเจ็บ ดังนั้นผมจึงเข้าใจการเลือกใช้คำศัพท์ที่ต่างกันเพื่อหลีกเลี่ยงสิ่งนี้

std API พูดถึงการดำเนินการ "before" (ไปทางด้านหน้า) และ "after" (ไปทางด้านหลัง) และแทนที่จะใช้ `next` และ `next_back` มัน... เรียกว่า `move_next` และ `move_prev` HRM. โอเค มันใช้คำศัพท์จาก iterator บ้าง แต่อย่างน้อย `next` ก็ไม่ได้ทำให้คิดถึง front/back และช่วยให้คุณเข้าใจว่าสิ่งต่างๆ ทำงานอย่างไรเมื่อเทียบกับ iterator

เราทำงานกับสิ่งนี้ได้
