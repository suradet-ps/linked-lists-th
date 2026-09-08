# การทดสอบเคอร์เซอร์

ถึงเวลาค้นหาว่าฉันทำผิดพลาดอันน่าอายไปกี่ครั้งในส่วนที่แล้ว!

โอ้ พระเจ้า เราสร้าง API ที่ไม่เหมือนทั้ง std และ impl เก่า โอเค งั้นฉันจะรีบสร้างอะไรบางอย่างจากทั้งสองฝั่ง ใช่ เราจะ "ยืม" เทสต์เหล่านี้จาก std:

```rust ,ignore
    #[test]
    fn test_cursor_move_peek() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 1));
        assert_eq!(cursor.peek_next(), Some(&mut 2));
        assert_eq!(cursor.peek_prev(), None);
        assert_eq!(cursor.index(), Some(0));
        cursor.move_prev();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 2));
        assert_eq!(cursor.peek_next(), Some(&mut 3));
        assert_eq!(cursor.peek_prev(), Some(&mut 1));
        assert_eq!(cursor.index(), Some(1));

        let mut cursor = m.cursor_mut();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 6));
        assert_eq!(cursor.peek_next(), None);
        assert_eq!(cursor.peek_prev(), Some(&mut 5));
        assert_eq!(cursor.index(), Some(5));
        cursor.move_next();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 5));
        assert_eq!(cursor.peek_next(), Some(&mut 6));
        assert_eq!(cursor.peek_prev(), Some(&mut 4));
        assert_eq!(cursor.index(), Some(4));
    }

    #[test]
    fn test_cursor_mut_insert() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.splice_before(Some(7).into_iter().collect());
        cursor.splice_after(Some(8).into_iter().collect());
        // check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[7, 1, 8, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        cursor.splice_before(Some(9).into_iter().collect());
        cursor.splice_after(Some(10).into_iter().collect());
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[10, 7, 1, 8, 2, 3, 4, 5, 6, 9]);
        
        /* remove_current not impl'd
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(7));
        cursor.move_prev();
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), Some(9));
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(10));
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[1, 8, 2, 3, 4, 5, 6]);
        */

        let mut cursor = m.cursor_mut();
        cursor.move_next();
        let mut p: LinkedList<u32> = LinkedList::new();
        p.extend([100, 101, 102, 103]);
        let mut q: LinkedList<u32> = LinkedList::new();
        q.extend([200, 201, 202, 203]);
        cursor.splice_after(p);
        cursor.splice_before(q);
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]
        );
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        let tmp = cursor.split_before();
        assert_eq!(m.into_iter().collect::<Vec<_>>(), &[]);
        m = tmp;
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        let tmp = cursor.split_after();
        assert_eq!(tmp.into_iter().collect::<Vec<_>>(), &[102, 103, 8, 2, 3, 4, 5, 6]);
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[200, 201, 202, 203, 1, 100, 101]);
    }

    fn check_links<T>(_list: &LinkedList<T>) {
        // would be good to do this!
    }
```

เวลามาถึงแล้ว!

```text
cargo test

   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 1.03s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_debug ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_ord ... ok
test test::test_cursor_move_peek ... FAILED
test test::test_cursor_mut_insert ... FAILED
test test::test_iterator ... ok
test test::test_mut_iter ... ok
test test::test_eq ... ok
test test::test_rev_iter ... ok
test test::test_iterator_double_end ... ok
test test::test_hashmap ... ok
test test::test_ord_nan ... ok

failures:

---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `None`,
 right: `Some(1)`', src\lib.rs:1079:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1153:9


failures:
    test::test_cursor_move_peek
    test::test_cursor_mut_insert

test result: FAILED. 12 passed; 2 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ฉันยอมรับว่าฉันมีความมั่นใจเกินไปหน่อยและหวังว่าจะทำสำเร็จ นี่คือเหตุผลที่เราเขียนเทสต์ (แต่บางทีฉันอาจแค่ย้ายเทสต์มาได้ไม่ดี?)

ความล้มเหลวแรกคืออะไร?

```rust ,ignore
let mut m: LinkedList<u32> = LinkedList::new();
m.extend([1, 2, 3, 4, 5, 6]);
let mut cursor = m.cursor_mut();

cursor.move_next();
assert_eq!(cursor.current(), Some(&mut 1));
assert_eq!(cursor.peek_next(), Some(&mut 2));
assert_eq!(cursor.peek_prev(), None);
assert_eq!(cursor.index(), Some(0));

cursor.move_prev();
assert_eq!(cursor.current(), None);
assert_eq!(cursor.peek_next(), Some(&mut 1)); // DIES HERE
```

ว้าว ฉันทำฟังก์ชันพื้นฐานบางอย่างพังจริงๆ เดี๋ยวก่อน

> หัวว่างเปล่า Option methods และ compiler errors (ที่ละไว้) ทำหน้าที่คิดทุกอย่างแทนตอนนี้

ฉันไม่เคยซื่อสัตย์เท่านี้มาก่อน

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

...ใช่ อันนี้ผิดจริงๆ ถ้า `self.cur` เป็น None เราไม่ได้แค่ยอมแพ้ เราต้องตรวจสอบ `self.list.front` ด้วย เพราะเราอยู่บน ghost! ดังนั้นเราแค่ต้องเพิ่ม or_else ไปยัง chain:

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .or_else(|| self.list.front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .or_else(|| self.list.back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

ซ่อมแล้วหรือยัง?

```text
---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `Some(6)`,
 right: `None`', src\lib.rs:1078:9
```

เดี๋ยวนะ ตอนนี้มันผิด *ย้อนกลับไปมากกว่าเดิม* โอเค ฉันต้องหยุดคิดแบบหัวว่างเปล่าเกี่ยวกับ peek เพราะมันยากกว่าที่ฉันคิดไว้มาก การพยายามเชื่อมต่อกรณีเหล่านี้แบบสุ่มสี่สุ่มห้าเป็นหายนะ เรามาเขียน if ที่เหมาะสมสำหรับกรณี ghost กับไม่ ghost กันดีกว่า:

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        let next = if let Some(cur) = self.cur {
            // Normal case, try to follow the cur node's back pointer
            (*cur.as_ptr()).back
        } else {
            // Ghost case, try to use the list's front pointer
            self.list.front
        };

        // Yield the element if the next node exists
        next.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        let prev = if let Some(cur) = self.cur {
            // Normal case, try to follow the cur node's front pointer
            (*cur.as_ptr()).front
        } else {
            // Ghost case, try to use the list's back pointer
            self.list.back
        };

        // Yield the element if the prev node exists
        prev.map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

มั่นใจกับอันนี้มาก!

```text
failures:

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1168:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    test::test_cursor_mut_insert

test result: FAILED. 13 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

เยสสส โอเค เหลืออีกความล้มเหลวเดียว... โอ้

คุณสังเกตไหมว่าฉัน comment โค้ดบางส่วนสำหรับทดสอบ remove_current ออก? ใช่ ฉันไม่ได้สังเกตว่าเทสต์นี้มีสถานะ เรามาสร้าง linked list ใหม่ด้วยสถานะที่ส่วน remove_current จะทิ้งเราไว้ดีกว่า:

```rust ,ignore
let mut m: LinkedList<u32> = LinkedList::new();
m.extend([1, 8, 2, 3, 4, 5, 6]);
```

```text
 cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.70s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_cursor_move_peek ... ok
test test::test_eq ... ok
test test::test_cursor_mut_insert ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_ord_nan ... ok
test test::test_mut_iter ... ok
test test::test_hashmap ... ok
test test::test_debug ... ok
test test::test_ord ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

เฮ้ย ดูนั่นสิ... โอเค ตอนนี้ฉันเริ่มระแวงแล้ว เรามาเติม check_links ให้สมบูรณ์และทดสอบภายใต้ miri กันดีกว่า:

```rust ,ignore
fn check_links<T: Eq + std::fmt::Debug>(list: &LinkedList<T>) {
    let from_front: Vec<_> = list.iter().collect();
    let from_back: Vec<_> = list.iter().rev().collect();
    let re_reved: Vec<_> = from_back.into_iter().rev().collect();

    assert_eq!(from_front, re_reved);
}
```

นี่เป็นวิธีที่ดีที่สุดหรือเปล่า? ไม่ ใช่ได้ไหม? ได้

```text
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo miri test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic ... ok
test test::test_basic_front ... ok
test test::test_cursor_move_peek ... ok
test test::test_cursor_mut_insert ... ok
test test::test_debug ... ok
test test::test_eq ... ok
test test::test_hashmap ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_mut_iter ... ok
test test::test_ord ... ok
test test::test_ord_nan ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.10s
```

เสร็จสิ้น.

เสร็จแล้ว.

เราทำสำเร็จแล้ว เราสร้าง LinkedList คุณภาพระดับ production ที่มีฟังก์ชันเกือบทั้งหมดเหมือนกับใน std แล้ว เราขาด method อำนวยความสะดวกเล็กๆ น้อยๆ ที่นี่ที่นั่นไหม? แน่นอน ฉันจะเพิ่มมันในเวอร์ชันที่เผยแพร่สุดท้ายของ crate ไหม? น่าจะ!

แต่... ฉันเหนื่อยมาก

ดังนั้น... เราชนะแล้ว

เดี๋ยวก่อน เชี่ต เราต้องการคุณภาพระดับ production โอเค บอสตัวสุดท้าย: clippy

```text
cargo clippy

cargo clippy
    Checking linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:189:19
    |
189 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: `#[warn(clippy::redundant_pattern_matching)]` on by default
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: method `into_iter` can be confused for the standard trait method `std::iter::IntoIterator::into_iter`
   --> src\lib.rs:210:5
    |
210 | /     pub fn into_iter(self) -> IntoIter<T> {
211 | |         IntoIter {
212 | |             list: self
213 | |         }
214 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::should_implement_trait)]` on by default
    = help: consider implementing the trait `std::iter::IntoIterator` or choosing a less ambiguous method name
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#should_implement_trait

warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:228:19
    |
228 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: re-implementing `PartialEq::ne` is unnecessary
   --> src\lib.rs:275:5
    |
275 | /     fn ne(&self, other: &Self) -> bool {
276 | |         self.len() != other.len() || self.iter().ne(other)
277 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::partialeq_ne_impl)]` on by default
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#partialeq_ne_impl

warning: `linked-list` (lib) generated 4 warnings
    Finished dev [unoptimized + debuginfo] target(s) in 0.29s
```

ได้เลย clippy มาเริ่มกัน

ข้อร้องเรียนที่ 1 (และ 3): เราใช้ `while let Some(_) = ` แทน `while .is_some()` ลูปว่างเปล่าดังนั้นมันจึงไม่สำคัญจริงๆ แต่โอเค ได้เลย clippy ฉันจะทำตามแบบของนาย

ข้อร้องเรียนที่ 2: เรามี inherent method into_iter จริงๆ เดี๋ยวก่อน อะไรนะ *เช็ค std* โอเค ยอมรับว่า clippy พูดถูก IntoIterator อยู่ใน prelude (และแทบจะเป็น lang item) ดังนั้นเราไม่จำเป็นต้องมีเวอร์ชัน inherent ด้วย

ข้อร้องเรียนที่ 4: เราคัดลอก cargocult แปลกๆ มาจาก std *ชrug* ได้เลย ฉันจะลบมัน

```text
cargo clippy
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
```

ดี สิ่งสุดท้ายที่ต้องทำก่อนเรียกว่าคุณภาพระดับ production: fmt

```text
cargo fmt
```

...ใช่ มันเพิ่ม newline บางบรรทัดและลบ trailing whitespace บางส่วน ไม่มีอะไรน่าสนใจ

**ตอนนี้เราเสร็จสิ้นอย่างแท้จริงแล้ว!!!!!!!!!!!!!!!!!!!!!**
