# พื้นฐาน

โอเค นี่คือส่วนที่แย่ที่สุดของหนังสือเล่มนี้ และเป็นเหตุผลว่าทำไมฉันถึงใช้เวลาเขียนบทนี้ถึง 7 ปี! ได้เวลาผ่านสิ่งที่น่าเบื่อมากมายที่เราเคยทำมาแล้ว 5 ครั้งแล้ว แต่จะอธิบายยาวและละเอียดกว่าเดิมเพราะเราต้องทำทุกอย่างสองครั้งและใช้ `Option<NonNull<Node<T>>>`!

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

PhantomData เป็นชนิดที่แปลกประหลาดที่ไม่มีฟิลด์ใดๆ ดังนั้นคุณแค่สร้างมันด้วยการบอกชื่อชนิดของมัน *ยักไหล่*

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


อ๋อ ใช่ ฉันเกลียดลูกๆ ที่เป็นตัวชี้ของฉันจริงๆ เราต้องดึง raw pointer ออกจาก NonNull ด้วย `as_ptr` อย่างชัดเจน เพราะ DerefMut ถูกนิยามในแง่ของ `&mut` และเราไม่ต้องการนำเรเฟอเรนซ์ที่ปลอดภัยมาใส่ใน unsafe code แบบสุ่ม!


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

ดีเลย ตอนนี้มาทำ pop (และ len):

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

ดูสมเหตุสมผลดี ได้เวลาเขียนเทสต์!

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

ฮูเรย์ เราสมบูรณ์แบบ!

...จริงไหม?
