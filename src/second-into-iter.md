# IntoIter

คอลเลกชันถูกวนซ้ำใน Rust ด้วยเทรต *Iterator* มันซับซ้อนกว่า `Drop` เล็กน้อย:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

น้องใหม่ในบล็อกตรงนี้คือ `type Item` นี่คือการประกาศว่าทุกการ implement ของ Iterator
มี*ชนิดที่เกี่ยวข้อง (associated type)* ที่เรียกว่า Item ในกรณีนี้ นี่คือชนิดที่มันสามารถ
พ่นออกมาได้เมื่อคุณเรียก `next`

เหตุผลที่ Iterator คืน `Option<Self::Item>` เป็นเพราะอินเทอร์เฟซได้รวม
แนวคิด `has_next` และ `get_next` เข้าด้วยกัน เมื่อคุณมีค่าถัดไป คุณจะคืน
`Some(value)` และเมื่อไม่มีคุณจะคืน `None` สิ่งนี้ทำให้ API โดยรวมใช้งาน
และ implement ได้ง่ายกว่าและปลอดภัยกว่า ในขณะที่หลีกเลี่ยงการตรวจสอบ
และตรรกะซ้ำซ้อนระหว่าง `has_next` และ `get_next` ยอดเยี่ยม!

โชคร้ายที่ Rust ไม่มีคำสั่งแบบ `yield` (ยัง) ดังนั้นเราจะต้อง implement
ตรรกะเอง นอกจากนี้ จริง ๆ แล้วมี iterators 3 แบบที่คอลเลกชันแต่ละตัว
ควรพยายาม implement:

* IntoIter - `T`
* IterMut - `&mut T`
* Iter - `&T`

จริง ๆ แล้วเรามีเครื่องมือทั้งหมดที่จะ implement IntoIter ด้วยอินเทอร์เฟซ
ของ List อยู่แล้ว: แค่เรียก `pop` ซ้ำไปเรื่อย ดังนั้นเราจะ implement
IntoIter เป็น newtype wrapper รอบ List:

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

มาเขียนเทสต์กัน:

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

ยอดเยี่ยม!