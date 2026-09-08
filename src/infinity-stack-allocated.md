# Linked List แบบจัดเก็บบน Stack

หนังสือเล่มนี้เน้นไปที่ linked list แบบ *จัดเก็บบน heap* เป็นหลัก เพราะเป็นแบบที่นิยมใช้และใช้งานได้จริงมากที่สุด แต่เราไม่ *จำเป็น* ต้องใช้ heap allocation เสมอไป Heap allocation นั้นดีเพราะทำให้จัดสรรหน่วยความจำแบบ dynamic ได้ง่าย Stack allocation นั้นไม่เป็นมิตรในแง่นี้นัก &mdash; ฟังก์ชันอย่าง `alloca` ของภาษา C ถูกมองว่าเป็นสิ่งที่ "ต้องสาปและมีปัญหามาก" อย่างกว้างขวาง

ดังนั้นเรามาจัดสรรหน่วยความจำบน stack แบบง่ายๆ กันเถอะ: ด้วยการเรียกฟังก์ชันและรับ stack frame ใหม่ที่มีพื้นที่เพิ่มเติม! นี่เป็นวิธีแก้ปัญหาที่ดูเหมือนจะตลกมาก แต่ก็ใช้งานได้จริงและมีประโยชน์อย่างแท้จริง มันถูกใช้ตลอดเวลา บางทีอาจไม่ได้คิดด้วยซ้ำว่ามันคือ linked list!

เมื่อใดก็ตามที่คุณทำงานแบบ recursive คุณสามารถส่ง pointer ไปยัง state ของขั้นตอนปัจจุบันไปยังขั้นตอนถัดไปได้เลย ถ้า pointer นั้นเป็น *ส่วนหนึ่ง* ของ state เอง คุณก็จะสร้าง linked list ที่จัดเก็บบน stack แล้ว!

ตอนนี้แน่นอนว่าเราอยู่ในส่วนที่ "ตลก" ของหนังสือ ดังนั้นเราจะทำแบบนี้ด้วยวิธีที่ตลก: ด้วยการทำให้ linked list เป็นตัวเอกและบังคับให้โค้ดทั้งหมดของผู้ใช้ต้องอยู่ใน callbacks ที่ซ้อนกัน ทุกคนชอบ nested callbacks!

List type ของเราจะเป็นเพียง Node ที่มี reference ไปยัง Node อื่น:

```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a List<'a, T>>,
}
```

และจะมีเพียง operation เดียวคือ `push` ซึ่งจะรับ list เก่า, state ของ node ปัจจุบัน, และ callback List ใหม่จะถูกสร้างขึ้นใน callback เราจะปล่อยให้ callback คืนค่าใดก็ได้ ซึ่ง `push` จะคืนค่าเมื่อทำงานเสร็จ:

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

แค่นั้นเลย! เราสามารถใช้มันแบบนี้:

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

มันสวยงาม 😿

ผู้ใช้สามารถ traverse list นี้ได้แล้วโดยใช้ while-let เพื่อเดินผ่านค่า `prev` แต่เพื่อความสนุก เรามา implement iterator แบบปกติกัน:

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

มาทดสอบกัน:

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

ตอนนี้คุณอาจจะสงสัยว่า "เฮ้ ฉันสามารถเปลี่ยนแปลงข้อมูลที่เก็บอยู่ใน node ได้ไหม?" บางทีได้! เรามาลองใช้ list แบบ mutable references แทน shared ones:


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

เฮ้อ ดูเหมือนมันไม่ชอบ iterator ของเรา บางทีเราทำผิดพลาดตรงไหน? เรามาลดความซับซ้อนของ test ลงเล็กน้อยเพื่อตรวจสอบ:


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

ฮึม ไม่ใช่ มันยังเป็นขยะร้อนอยู่เหมือนเดิม

ปัญหาคือ list ของเราโดยบังเอิญ(😉) กำลังพึ่งพา *variance* [Variance เป็นหัวข้อที่ซับซ้อน](https://doc.rust-lang.org/nomicon/subtyping.html) แต่เรามาดูกันในรูปแบบที่ง่ายขึ้น:

แต่ละ list มี reference ไปยัง List ที่มี *type เหมือนกันเองเป๊ะๆ* ในมุมมองของ list ที่ซ้อนอยู่ลึกสุด นั่นหมายความว่าทุก list ใช้ lifetime เดียวกัน แต่นี่เป็น *สิ่งที่ไม่จริงโดยเป้าประสงค์*: node แต่ละตัวใน list มีอายุยืนยาวกว่า node ถัดไป เพราะพวกมันอยู่ใน nested scopes จริงๆ!

ดังนั้น... ทำไมโค้ดถึง compile ได้ตอนที่เราใช้ shared references? เพราะในหลายๆ กรณี compiler รู้ว่ามันปลอดภัยที่จะมีบางสิ่งที่มีอายุ "ยาวเกินไป"! เมื่อเราใส่ reference ของ list ไปยัง list ถัดไป compiler จะ "ย่อ" lifetime ลงอย่างเงียบๆ ให้เข้ากับสิ่งที่ list ใหม่คาดหวัง การย่อ lifetime นี้คือ *variance*

มันเป็นเทคนิคเดียวกันเลยกับภาษาที่มี inheritance ซึ่งให้คุณส่ง Cat ไปยังที่ที่คาดหวัง Animal (supertype ของ Cat) ได้ โดยสัญชาตญาณเรารู้ว่ามันโอเคที่จะส่ง Cat เมื่อคาดหวัง Animal เพราะ Cat เป็นได้แค่ Animal *และอะไรเพิ่มเติมอีก* มัน *โอเค* ที่จะลืมส่วน "และอะไรเพิ่มเติมอีก" ไปสักพัก จริงไหม?

ในทำนองเดียวกัน lifetime ที่ใหญ่กว่าเป็นเพียง lifetime ที่เล็กกว่า *และอะไรเพิ่มเติมอีก* ดังนั้นมันจึงโอเคที่จะลืมส่วน "และอะไรเพิ่มเติมอีก" ตรงนี้เช่นกัน!

แต่แน่นอนว่าตอนนี้คุณอาจจะสงสัย: แล้วทำไมเวอร์ชันที่ใช้ mutable references ถึงไม่ทำงาน!?

ก็ variance *ไม่ได้* ปลอดภัยเสมอไป ถ้าโค้ดของเรา *compile* ได้ เราจะเขียน use-after-free แบบนี้ได้:

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

ปัญหาของการลืมรายละเอียดคือ *ที่อื่นอาจจำรายละเอียดเหล่านั้นได้และคาดหวังว่ามันจะยังเป็นจริงอยู่* นั่นเป็นปัญหาใหญ่มากเมื่อคุณ introduc *mutation* ถ้าคุณไม่ระวัง โค้ดที่ไม่จำ "และอะไรเพิ่มเติมอีก" ที่เราทิ้งไปอาจคิดว่ามันโอเคที่จะเขียนบางสิ่งไปยังที่ที่ "จำ" และ *คาดหวัง* ว่า "และอะไรเพิ่มเติมอีก" จะยังคงอยู่

ในแง่ของ inheritance: โค้ดนี้ต้องผิดกฎหมาย:

```rust ,ignore
let mut my_kitty = Cat;                  // Make a Cat (long lifetime)
let animal: &mut Animal = &mut my_kitty; // Forget it's a Cat (shorten lifetime)
*animal = Dog;                           // Write a Dog (short lifetime)
my_kitty.meow();                         // Meowing Dog! (Use After Free)
```

ดังนั้นแม้ว่าคุณ *จะสามารถ* ย่อ lifetime ของ mutable reference ได้ แต่เมื่อคุณเริ่ม *nesting* พวกมัน สิ่งต่างๆ จะกลายเป็น "invariant" และคุณจะไม่สามารถย่อ lifetime ได้อีกต่อไป

โดยเฉพาะอย่างยิ่ง `&mut &'big mut T` ไม่สามารถแปลงเป็น `&mut &'small mut T` ได้ ซึ่ง `'big` ใหญ่กว่า `'small` หรืออย่างเป็นทางการมากขึ้น `&'a mut T` เป็น covariant เหนือ `'a` แต่ invariant เหนือ `T`

เกร็ดความรู้: Java จริงๆ แล้ว *อนุญาต* ให้คุณทำสิ่งนี้ได้โดยเฉพาะ แต่มัน [ตรวจสอบ runtime เพื่อป้องกัน dogs ที่ร้องเหมียว](https://docs.oracle.com/javase/7/docs/api/java/lang/ArrayStoreException.html)

----

แล้วเราสามารถทำอะไรเพื่อเปลี่ยนแปลงข้อมูลได้? ใช้ interior mutability! วิธีนี้ทำให้เราบอก compiler ว่าเราแค่ต้องการเปลี่ยนแปลง *ข้อมูล* แต่จะไม่แตะ references

เราสามารถกลับไปใช้เวอร์ชันก่อนหน้าของโค้ดที่ใช้ shared references และใช้ `Cell` ใน test ใหม่ได้:

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

ง่ายเหมือน recursive pie! ✨