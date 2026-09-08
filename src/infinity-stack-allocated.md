# ลิงก์ลิสต์บนสแต็ก (The Stack-Allocated Linked List)

หนังสือเล่มนี้เน้นไปที่ linked list แบบ *จัดสรรบน heap* (heap-allocated) เป็นหลัก เพราะเป็นแบบที่พบเห็นได้ทั่วไปและใช้งานได้จริงมากที่สุด ทว่าเราก็ไม่ได้ *จำเป็น* จะต้องใช้ heap allocation เสมอไป การจองหน่วยความจำบน heap นั้นดีตรงที่ทำให้จัดสรรหน่วยความจำแบบไดนามิกได้ง่าย ส่วน stack allocation นั้นไม่ค่อยเป็นมิตรในด้านนี้นัก &mdash; ของพรรค์อย่าง `alloca` ในภาษา C นั้น คนทั่วไปต่างมองว่าเป็นอะไรที่ทั้งต้องสาปและสร้างปัญหาไม่รู้จบ

ดังนั้น เรามาจัดสรรหน่วยความจำบน stack ด้วยวิธีที่ง่ายที่สุดกันเถอะครับ: แค่เรียกฟังก์ชัน แล้วเราก็จะได้ stack frame ใหม่ที่มีพื้นที่เพิ่มเติมมาใช้งานแล้ว! นี่อาจจะฟังดูเป็นวิธีแก้ปัญหาที่ตลกโปกฮามาก แต่มันกลับใช้งานได้จริงและมีประโยชน์สุด ๆ ในโลกความเป็นจริง แถมยังถูกหยิบมาใช้กันอยู่ตลอดเวลา โดยที่หลายคนอาจไม่เคยนึกด้วยซ้ำว่านี่มันคือ linked list!

เวลาไหนก็ตามที่คุณทำอะไรแบบ recursive คุณสามารถส่งพอยน์เตอร์ที่ชี้ไปยัง state ของขั้นตอนปัจจุบันส่งต่อไปให้ขั้นตอนถัดไปได้เลย และถ้าตัวพอยน์เตอร์นั้นดันเป็น *ส่วนหนึ่ง* ของตัว state นั้นเองด้วยล่ะก็ ยินดีด้วยครับ คุณเพิ่งสร้าง linked list ที่จัดเก็บบน stack สำเร็จแล้ว!

แน่นอนว่าตอนนี้เรากำลังอยู่ในพาร์ต "ลิสต์โง่ ๆ" ของหนังสือเล่มนี้ เราก็เลยจะทำมันด้วยวิธีเพี้ยน ๆ: นั่นคือการยกให้ linked list เป็นพระเอก แล้วบีบบังคับให้โค้ดทั้งหมดของผู้ใช้ต้องไปจมปลักอยู่ในดง callback ซ้อนกันเป็นพรวน ใคร ๆ ก็ชอบ nested callback กันทั้งนั้นแหละจริงไหมครับ!

Type `List` ของเราจะเป็นแค่ Node ที่ถือ reference ชี้ไปยัง Node อีกตัวหนึ่ง:

```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a List<'a, T>>,
}
```

และมันจะมี operation เพียงแค่อย่างเดียวคือ `push` ซึ่งจะรับลิสต์ตัวเก่า, state ของโหนดปัจจุบัน และ callback โดยลิสต์ตัวใหม่จะถูกสร้างขึ้นมาภายใน callback นั้น นอกจากนี้เราจะเปิดให้ callback คืนค่าเป็น type อะไรก็ได้ ซึ่ง `push` จะนำค่านั้นมาส่งคืนต่อเมื่อทำงานเสร็จสิ้น:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&List<'a, T>) -> U,
    ) -> U {
        let list = List { data, prev };
        callback(&list)
    }
}
```

แค่นี้แหละครับ! เราสามารถเอามาใช้งานได้แบบนี้:

```rust ,ignore
List::push(None, 3, |list| {
    println!("{}", list.data);
    List::push(Some(list), 5, |list| {
        println!("{}", list.data);
        List::push(Some(list), 13, |list| {
            println!("{}", list.data);
        })
    })
})
```

มันช่างงดงามเหลือเกิน 😿

จริง ๆ ผู้ใช้สามารถท่องไปตามลิสต์นี้ได้อยู่แล้วผ่านการใช้ `while-let` เพื่อเดินไต่ตามค่า `prev` ไปเรื่อย ๆ แต่เพื่อความสนุก มาลอง implement iterator ในแบบที่เราคุ้นเคยกันดูครับ:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev;
            &node.data
        })
    }
}
```

มาลองรันเทสต์กันดู:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn elegance() {
        List::push(None, 3, |list| {
            assert_eq!(list.iter().copied().sum::<i32>(), 3);
            List::push(Some(list), 5, |list| {
                assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
                List::push(Some(list), 13, |list| {
                    assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
                })
            })
        })
    }
}
```

```text
> cargo test

running 18 tests
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::basics ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test second::test::iter_mut ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test second::test::peek ... ok
test third::test::iter ... ok

test result: ok. 18 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

มาถึงจุดนี้ คุณอาจจะเริ่มสงสัยแล้วว่า "เฮ้ แล้วเราจะ mutate ข้อมูลที่เก็บอยู่ในโหนดได้หรือเปล่านะ?" ก็อาจจะได้นะ! มาลองทำให้ตัวลิสต์หันมาใช้ mutable reference แทนที่จะเป็น shared reference ดูกันดีกว่า:


```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a mut List<'a, T>>,
}

pub struct Iter<'a, T> {
    next: Option<&'a List<'a, T>>,
}

impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a mut List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&mut List<'a, T>) -> U,
    ) -> U {
        let mut list = List { data, prev };
        callback(&mut list)
    }

    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev.as_ref().map(|prev| &**prev);
            &node.data
        })
    }
}

```


```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:47:32
   |
46 |  List::push(Some(list), 13, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
47 |      assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:45:28
   |
44 |  List::push(Some(list), 5, |list| {
   |                             ----
   |                             |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |      assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here


<ad infinitum>
```

อ้าว ดูเหมือนมันจะไม่ชอบขี้หน้า iterator ของเราแฮะ หรือว่าเราจะเขียนอะไรพลาดไปตรงไหน? งั้นเราลองลดความซับซ้อนของเทสต์ลงหน่อยเพื่อเช็คดูดีกว่า:


```rust ,ignore
#[test]
fn elegance() {
    List::push(None, 3, |list| {
        assert_eq!(list.data, 3);
        List::push(Some(list), 5, |list| {
            assert_eq!(list.data, 5);
            List::push(Some(list), 13, |list| {
                assert_eq!(list.data, 13);
            })
        })
    })
}
```

```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:46:17
   |
44 |   List::push(Some(list), 5, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |       assert_eq!(list.data, 5);
46 | /     List::push(Some(list), 13, |list| {
47 | |         assert_eq!(list.data, 13);
48 | |     })
   | |______^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:44:13
   |
42 |   List::push(None, 3, |list| {
   |                        ----
   |                        |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
43 |       assert_eq!(list.data, 3);
44 | /     List::push(Some(list), 5, |list| {
45 | |         assert_eq!(list.data, 5);
46 | |         List::push(Some(list), 13, |list| {
47 | |             assert_eq!(list.data, 13);
48 | |         })
49 | |     })
   | |______________^ `list` escapes the closure body here
```

อืม ไม่ใช่แฮะ โค้ดยังคงพังยับเยินเหมือนเดิม

ปัญหาคือลิสต์ของเราดันไปพึ่งพา *variance* เข้าให้โดยบังเอิญ (😉) [Variance เป็นเรื่องที่ซับซ้อนและชวนปวดหัว](https://doc.rust-lang.org/nomicon/subtyping.html) แต่เดี๋ยวเรามาลองทำความเข้าใจกันในมุมมองที่ย่อยง่ายดูครับ:

แต่ละลิสต์จะถือ reference ชี้ไปยัง `List` ที่มี *type เดียวกับตัวมันเองเป๊ะ ๆ* ในมุมมองของลิสต์ตัวในสุด นั่นแปลว่าทุกลิสต์กำลังแชร์ lifetime เดียวกันกับตัวมันเอง แต่นี่มัน *ขัดกับความเป็นจริงอย่างสิ้นเชิง* เลยครับ: เพราะโหนดแต่ละตัวในลิสต์ย่อมมีอายุยืนยาวกว่าโหนดถัดไปอย่างแน่นอน เนื่องจากพวกมันถูกสร้างอยู่ใน nested scope ที่ซ้อนกันจริง ๆ!

แล้ว... ทำไมโค้ดเวอร์ชันที่ใช้ shared reference ถึงคอมไพล์ผ่านฉลุยล่ะ? ก็เพราะว่าในหลาย ๆ สถานการณ์ คอมไพเลอร์รู้ดีว่ามันปลอดภัยที่จะยอมรับสิ่งที่อายุยืน "ยาวเกินไป"! พอเรายัด reference ของลิสต์ตัวหนึ่งเข้าไปในลิสต์ถัดไป คอมไพเลอร์ก็จะแอบ "หด" (shrink) lifetime ให้สั้นลงเงียบ ๆ เพื่อให้เข้ากับสิ่งที่ลิสต์ตัวใหม่คาดหวัง การหด lifetime ตรงนี้นี่แหละครับที่เราเรียกว่า *variance*

มันเป็นกลเม็ดเดียวกับในภาษาเชิงวัตถุที่มี inheritance เลยครับ ที่ยอมให้คุณส่งออบเจกต์ `Cat` ไปให้ฟังก์ชันที่ต้องการ `Animal` (ซึ่งเป็น supertype ของ `Cat`) ตามสามัญสำนึก เรารู้อยู่แล้วว่าการส่ง `Cat` ไปให้ฟังก์ชันที่ต้องการ `Animal` ย่อมทำได้อย่างปลอดภัยไร้ปัญหา เพราะ `Cat` ก็คือ `Animal` *แถมยังมีคุณสมบัติอื่นพ่วงมาด้วย* การที่เราจะ "แกล้งลืม" ส่วนที่คุณสมบัติพ่วงเกินมาสักพักมันก็ *ไม่เห็นจะเป็นอะไร* จริงไหมครับ?

ในทำนองเดียวกัน lifetime ที่ยาวกว่า ก็คือ lifetime ที่สั้นกว่า *แถมมีส่วนที่ยาวพ่วงเกินมา* ดังนั้นการจะแกล้งลืมส่วนที่เกินมานี้ไป ก็ปลอดภัยหายห่วงเช่นกัน!

แต่แน่นอนว่าตอนนี้คุณคงกำลังสงสัย: แล้วทำไมเวอร์ชันที่ใช้ mutable reference ถึงไม่ทำงานล่ะ!?

นั่นก็เพราะว่า variance *ไม่ได้* ปลอดภัยเสมอไปน่ะสิครับ ถ้าโค้ดของเราดัน *คอมไพล์ผ่าน* ขึ้นมาจริง ๆ เราก็จะสามารถแอบเขียน use-after-free แบบนี้ได้เลย:

```rust ,ignore
List::push(None, 3, |list| {
    List::push(Some(list), 5, |list| {
        List::push(Some(list), 13, |list| {
            // HAHAHA all the lifetimes are the same, so the compiler will
            // let me rewrite my parent to hold a mutable reference to myself!
            // I will create all the use-after-frees!!
            *list.prev.as_mut().unwrap().prev = Some(list);
        })
    })
})
```

ปัญหาของการแกล้งลืมรายละเอียดไปก็คือ *จุดอื่นอาจจะยังจำรายละเอียดเหล่านั้นได้ และคาดหวังว่ามันจะยังคงเป็นจริงอยู่เสมอ* ซึ่งนี่คือปัญหาคอขาดบาดตายทันทีที่คุณนำเรื่อง *mutation* เข้ามาร่วมวงด้วย ถ้าคุณไม่ระวัง โค้ดที่ไม่จดจำส่วน "ที่คุณสมบัติพ่วงเกินมา" ที่เราตัดทิ้งไป อาจจะนึกว่ามันปลอดภัยที่จะเขียนข้อมูลลงไปยังจุดที่ "ยังจำได้" และ *คาดหวัง* ว่าส่วนที่พ่วงเกินมานั้นจะยังอยู่ครบถ้วน

ถ้าเปรียบเทียบในมุมของ inheritance: โค้ดแบบนี้ต้องถือว่าผิดกฎหมายอย่างเด็ดขาด:

```rust ,ignore
let mut my_kitty = Cat;                  // Make a Cat (long lifetime)
let animal: &mut Animal = &mut my_kitty; // Forget it's a Cat (shorten lifetime)
*animal = Dog;                           // Write a Dog (short lifetime)
my_kitty.meow();                         // Meowing Dog! (Use After Free)
```

ดังนั้น แม้ว่าคุณ *จะสามารถ* หด lifetime ของ mutable reference ตัวเดี่ยว ๆ ได้ แต่ทันทีที่คุณเริ่มนำมันมา *ซ้อนกัน (nesting)* ทุกอย่างจะกลายเป็น "invariant" ทันที และคุณจะไม่ได้รับอนุญาตให้หด lifetime ได้อีกต่อไป

โดยเฉพาะอย่างยิ่ง `&mut &'big mut T` จะไม่สามารถแปลงเป็น `&mut &'small mut T` ได้ (โดยที่ `'big` นั้นยาวกว่า `'small` ) หรือถ้าจะพูดให้เป็นทางการขึ้นก็คือ `&'a mut T` จะเป็น covariant เหนือ `'a` แต่เป็น invariant เหนือ `T`

เกร็ดน่ารู้: จริง ๆ แล้วภาษา Java *อนุญาต* ให้คุณทำเรื่องแบบนี้ได้นะ แต่ต้องแลกกับการที่มันต้อง [คอยเช็คตอนรันไทม์ (runtime check) เพื่อป้องกันไม่ให้หมาส่งเสียงร้องเหมียว ๆ](https://docs.oracle.com/javase/7/docs/api/java/lang/ArrayStoreException.html)

----

แล้วถ้าเราอยากจะ mutate ข้อมูล เราจะทำอย่างไรได้บ้าง? คำตอบคือใช้ interior mutability ครับ! วิธีนี้จะช่วยให้เราบอกกับคอมไพเลอร์ได้ว่า เราแค่ต้องการ mutate ตัว *ข้อมูล* แต่จะไม่ไปแตะต้องหรือแก้ไขตัว reference เลย

เราสามารถย้อนกลับไปใช้โค้ดเวอร์ชันก่อนหน้าที่ใช้ shared reference แล้วนำ `Cell` มาใช้ในเทสต์ตัวใหม่แบบนี้:

```rust ,ignore
#[test]
fn cell() {
    use std::cell::Cell;

    List::push(None, Cell::new(3), |list| {
        List::push(Some(list), Cell::new(5), |list| {
            List::push(Some(list), Cell::new(13), |list| {
                // Multiply every value in the list by 10
                for val in list.iter() {
                    val.set(val.get() * 10)
                }

                let mut vals = list.iter();
                assert_eq!(vals.next().unwrap().get(), 130);
                assert_eq!(vals.next().unwrap().get(), 50);
                assert_eq!(vals.next().unwrap().get(), 30);
                assert_eq!(vals.next(), None);
                assert_eq!(vals.next(), None);
            })
        })
    })
}
```

```text
> cargo test

running 19 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter_mut ... ok
test fifth::test::iter ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test first::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fifth::test::miri_food ... ok
test silly2::test::cell ... ok
test third::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test third::test::basics ... ok
test second::test::iter ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

ง่ายเหมือนปอกกล้วยแบบ recursive เข้าปาก! ✨