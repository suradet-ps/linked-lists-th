# IntoIter

ในภาษา Rust คอลเลกชันต่าง ๆ จะถูกท่องวนลูปอ่านข้อมูลผ่านทางเทรต *Iterator* ซึ่งมันจะมีความซับซ้อนมากกว่าเทรต `Drop` ขึ้นมาอีกนิดหน่อย:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

หน้าใหม่ที่เพิ่งโผล่เข้ามาในบล็อกนี้ก็คือ `type Item` ซึ่งเป็นการประกาศว่า การนำเทรต Iterator ไปปฏิบัติ (implement) ทุกครั้ง จะต้องระบุ *associated type (ชนิดข้อมูลร่วม)* ที่ชื่อว่า `Item` เสมอ ซึ่งในกรณีนี้ มันก็คือชนิดข้อมูลของค่าที่จะถูกพ่นส่งกลับออกมาเวลาที่เราเรียกเมธอด `next` นั่นเอง

สาเหตุที่ `Iterator` ส่งค่ากลับออกมาเป็น `Option<Self::Item>` ก็เพราะอินเทอร์เฟซนี้ได้ยุบรวมแนวคิดของ `has_next` และ `get_next` เข้ามาไว้ด้วยกันอย่างแนบเนียน กล่าวคือ เมื่อยังมีค่าถัดไปอยู่ มันก็จะส่ง `Some(value)` ออกมา และเมื่อข้อมูลหมดแล้ว มันก็จะส่ง `None` ออกมาแทน วิธีนี้ช่วยให้ API ใช้งานง่ายขึ้น ปลอดภัยขึ้น เขียนง่ายขึ้น แถมยังหลีกเลี่ยงการเช็กเงื่อนไขซ้ำซ้อนไปมาระหว่าง `has_next` กับ `get_next` ได้อย่างหมดจด ยอดเยี่ยมไปเลยใช่ไหมล่ะ!

แต่น่าเสียดายที่ Rust ยังไม่มีคำสั่งจำพวก `yield` (อย่างน้อยก็ในตอนนี้) เราจึงต้องเขียนลอจิกในการจำสถานะและดึงข้อมูลขึ้นมาเอง และจริง ๆ แล้ว มีรูปแบบ iterator อยู่ 3 ชนิดด้วยกันที่คอลเลกชันที่ดีควรพยายามเขียนเตรียมไว้ให้ครบ:

* `IntoIter` - คืนค่าเป็น `T` (ย้ายความเป็นเจ้าของ)
* `IterMut` - คืนค่าเป็น `&mut T` (ยืมแบบแก้ไขได้)
* `Iter` - คืนค่าเป็น `&T` (ยืมแบบอ่านอย่างเดียว)

และจริง ๆ แล้ว ตอนนี้เรามีเครื่องมือทุกอย่างครบถ้วนสำหรับเขียน `IntoIter` บนอินเทอร์เฟซเดิมของ `List` อยู่แล้ว: ก็แค่เรียกเมธอด `pop` ซ้ำไปเรื่อย ๆ จนกว่าข้อมูลจะหมดนั่นเอง! ดังนั้น เราจึงเขียน `IntoIter` ขึ้นมาเป็น newtype wrapper ห่อหุ้ม `List` เอาไว้ได้เลย:

```rust ,ignore
// Tuple structs are an alternative form of struct,
// useful for trivial wrappers around other types.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // access fields of a tuple struct numerically
        self.0.pop()
    }
}
```

แล้วมาลองเขียนเทสต์กัน:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

ยอดเยี่ยมมาก!