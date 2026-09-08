# กรณีสมมาตร

เอาล่ะ มาจัดการsymmetry แบบคอมบิเนเตอร์ทั้งหมดให้เสร็จ

สิ่งที่เราต้องทำคือการแทนที่ข้อความพื้นฐาน:

```text
tail <-> head
next <-> prev
front -> back
```

โอ้ นอกจากนี้เราต้องเพิ่มตัวแปร `_mut` สำหรับ peek

```rust ,ignore
use std::cell::{Ref, RefCell, RefMut};

//..

pub fn push_back(&mut self, elem: T) {
    let new_tail = Node::new(elem);
    match self.tail.take() {
        Some(old_tail) => {
            old_tail.borrow_mut().next = Some(new_tail.clone());
            new_tail.borrow_mut().prev = Some(old_tail);
            self.tail = Some(new_tail);
        }
        None => {
            self.head = Some(new_tail.clone());
            self.tail = Some(new_tail);
        }
    }
}

pub fn pop_back(&mut self) -> Option<T> {
    self.tail.take().map(|old_tail| {
        match old_tail.borrow_mut().prev.take() {
            Some(new_tail) => {
                new_tail.borrow_mut().next.take();
                self.tail = Some(new_tail);
            }
            None => {
                self.head.take();
            }
        }
        Rc::try_unwrap(old_tail).ok().unwrap().into_inner().elem
    })
}

pub fn peek_back(&self) -> Option<Ref<T>> {
    self.tail.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}

pub fn peek_back_mut(&mut self) -> Option<RefMut<T>> {
    self.tail.as_ref().map(|node| {
        RefMut::map(node.borrow_mut(), |node| &mut node.elem)
    })
}

pub fn peek_front_mut(&mut self) -> Option<RefMut<T>> {
    self.head.as_ref().map(|node| {
        RefMut::map(node.borrow_mut(), |node| &mut node.elem)
    })
}
```

และขยายเทสต์ของเราอย่างมาก:


```rust ,ignore
#[test]
fn basics() {
    let mut list = List::new();

    // Check empty list behaves right
    assert_eq!(list.pop_front(), None);

    // Populate list
    list.push_front(1);
    list.push_front(2);
    list.push_front(3);

    // Check normal removal
    assert_eq!(list.pop_front(), Some(3));
    assert_eq!(list.pop_front(), Some(2));

    // Push some more just to make sure nothing's corrupted
    list.push_front(4);
    list.push_front(5);

    // Check normal removal
    assert_eq!(list.pop_front(), Some(5));
    assert_eq!(list.pop_front(), Some(4));

    // Check exhaustion
    assert_eq!(list.pop_front(), Some(1));
    assert_eq!(list.pop_front(), None);

    // ---- back -----

    // Check empty list behaves right
    assert_eq!(list.pop_back(), None);

    // Populate list
    list.push_back(1);
    list.push_back(2);
    list.push_back(3);

    // Check normal removal
    assert_eq!(list.pop_back(), Some(3));
    assert_eq!(list.pop_back(), Some(2));

    // Push some more just to make sure nothing's corrupted
    list.push_back(4);
    list.push_back(5);

    // Check normal removal
    assert_eq!(list.pop_back(), Some(5));
    assert_eq!(list.pop_back(), Some(4));

    // Check exhaustion
    assert_eq!(list.pop_back(), Some(1));
    assert_eq!(list.pop_back(), None);
}

#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    assert!(list.peek_back().is_none());
    assert!(list.peek_front_mut().is_none());
    assert!(list.peek_back_mut().is_none());

    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
    assert_eq!(&mut *list.peek_front_mut().unwrap(), &mut 3);
    assert_eq!(&*list.peek_back().unwrap(), &1);
    assert_eq!(&mut *list.peek_back_mut().unwrap(), &mut 1);
}
```

มีกรณีที่เราไม่ได้ทดสอบไหม? บางที พื้นที่คอมบิเนเตอร์
ได้ระเบิดไปแล้ว โค้ดของเราอย่างน้อยไม่ได้*ผิดอย่างเห็นได้ชัด*

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

ดี การ copy-paste เป็นการเขียนโปรแกรมที่ดีที่สุด
