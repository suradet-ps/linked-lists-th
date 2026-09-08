# พื้นฐาน

เอาล่ะ นี่คือพาร์ตที่น่าเบื่อหน่ายที่สุดของหนังสือเล่มนี้ และเป็นเหตุผลว่าทำไมผมถึงใช้เวลาดองบทนี้นานถึง 7 ปีเต็มกว่าจะเขียนเสร็จ! ได้เวลามานั่งก้มหน้าก้มตาเขียนเรื่องน่าเบื่อพรรค์นี้ที่เราเคยทำซ้ำมาแล้วถึง 5 รอบ แต่รอบนี้จะเวิ่นเว้อยืดยาวกว่าเดิมเป็นพิเศษ เพราะเราต้องเขียนทุกอย่างซ้ำสองรอบ แถมยังต้องแบกรับความยุ่งยากของ `Option<NonNull<Node<T>>>` อีกต่างหาก!

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }
}
```

`PhantomData` เป็น type ที่พิลึกกึกกือตรงที่มันไม่มีฟิลด์อะไรข้างในเลย ดังนั้นคุณก็แค่เสกมันขึ้นมาดื้อๆ ด้วยการพิมพ์ชื่อ type ของมันออกมาเฉยๆ นั่นแหละ *ยักไหล่*

```rust ,ignore
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
            (*old).front = Some(new);
            (*new).back = Some(old);
        } else {
            // If there's no front, then we're the empty list and need 
            // to set the back too. Also here's some integrity checks
            // for testing, in case we mess up.
            debug_assert!(self.back.is_none());
            debug_assert!(self.front.is_none());
            debug_assert!(self.len == 0);
            self.back = Some(new);
        }
        self.front = Some(new);
        self.len += 1;
    }
}
```

```text
error[E0614]: type `NonNull<Node<T>>` cannot be dereferenced
  --> src\lib.rs:39:17
   |
39 |                 (*old).front = Some(new);
   |                 ^^^^^^
```


อ้อ ใช่แล้ว... ผมเกลียดบรรดาลูกๆ ที่เป็นพอยน์เตอร์ของผมเข้าไส้จริงๆ เราจำเป็นต้องดึงเอา raw pointer ออกมาจาก `NonNull` แบบโจ่งแจ้งด้วยคำสั่ง `as_ptr` เสียก่อน เพราะว่า `DerefMut` มันถูกนิยามไว้บนฐานของ `&mut` และเราไม่อยากจะแอบพา safe reference เข้ามาเพ่นพ่านในโค้ด unsafe ของเราตามอำเภอใจ!


```rust ,ignore
            (*old.as_ptr()).front = Some(new);
            (*new.as_ptr()).back = Some(old);
```

```text
   Compiling linked-list v0.0.3
warning: field is never read: `elem`
  --> src\lib.rs:16:5
   |
16 |     elem: T,
   |     ^^^^^^^
   |
   = note: `#[warn(dead_code)]` on by default

warning: `linked-list` (lib) generated 1 warning (1 duplicate)
warning: `linked-list` (lib test) generated 1 warning
    Finished test [unoptimized + debuginfo] target(s) in 0.33s
```

เยี่ยมไปเลย คราวนี้มาต่อกันที่ `pop` (แล้วก็แถม `len` ด้วย):

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    unsafe {
        // Only have to do stuff if there is a front node to pop.
        // Note that we don't need to mess around with `take` anymore
        // because everything is Copy and there are no dtors that will
        // run if we mess up... right? :) Riiiight? :)))
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
                debug_assert!(self.len == 1);
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
```

```text
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.37s
```

ดูเข้าท่าใช้ได้ ได้เวลาเขียนเทสต์กันแล้ว!

```rust ,ignore
#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Try to break an empty list
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }
}
```


```text
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.40s
     Running unittests src\lib.rs

running 1 test
test test::test_basic_front ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

เย่ๆๆๆ โค้ดของเราเพอร์เฟกต์ไร้ที่ติ!

...ใช่ไหมนะ?
