# การทดสอบเคอร์เซอร์

ถึงเวลามาดูกันแล้วว่าผมทำเรื่องผิดพลาดชวนหน้าแตกหมอไม่รับเย็บไว้กี่จุดในหัวข้อที่แล้ว!

พระเจ้าช่วย เราดันไปทำ API ให้ไม่เหมือนทั้งของ std และของเก่าทั้งคู่เลยแฮะ โอเค งั้นผมจะรีบจับแพะชนแกะเอาเทสต์จากทั้งสองฝั่งมายำรวมกันหน่อยละกัน ใช่ครับ เรามา "ยืม" เทสต์พวกนี้มาจาก std กันเถอะ:

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

ช่วงเวลาแห่งความจริงมาถึงแล้ว!

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

ผมยอมรับตรง ๆ เลยว่าเผลอมั่นใจเกินเหตุไปหน่อยและแอบหวังว่าจะทำรอบเดียวผ่านฉลุย นี่แหละครับคือเหตุผลที่เราเขียนเทสต์ (แต่ก็ไม่แน่นะ บางทีผมอาจจะพอร์ตเทสต์มาไม่ดีเองหรือเปล่า..?)

เทสต์แรกที่พังคืออะไรนะ?

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

โอ้โห นี่ผมทำฟังก์ชันพื้นฐานพังเละเทะเลยหรือเนี่ย เดี๋ยวนะ

> สมองว่างเปล่า ปล่อยให้เมธอดของ Option และ error จากคอมไพเลอร์ (ที่ละไว้) คิดแทนทุกอย่างในตอนนี้

แหม อย่างน้อยผมก็เป็นคนซื่อสัตย์ล่ะนะ

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

...ใช่เลย อันนี้ผิดเต็มประตู ถ้า `self.cur` เป็น `None` เราจะยอมแพ้ง่าย ๆ ไม่ได้ เราต้องเช็ค `self.list.front` ด้วย เพราะตอนนี้เราอยู่ที่ ghost! ดังนั้นเราแค่ต้องเพิ่ม `or_else` เข้าไปใน chain:

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

แก้หายไหมนะ?

```text
---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `Some(6)`,
 right: `None`', src\lib.rs:1078:9
```

เดี๋ยวก่อน ตอนนี้มันดันไปพัง *ลึกเข้าไปอีก* โอเค ผมต้องเลิกเขียน `peek` ในโหมดสมองโล่งแล้วล่ะ เพราะดูเหมือนมันจะยากกว่าที่ผมประเมินไว้เยอะ การพยายาม chain เคสพวกนี้มั่ว ๆ เข้าด้วยกันนี่มันหายนะชัด ๆ มาเขียน `if` แบ่งเคสระหว่างเป็น ghost กับไม่ ghost ให้เป็นเรื่องเป็นราวดีกว่า:

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

รอบนี้รู้สึกมั่นใจสุด ๆ!

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

เยสสสส โอเค เหลืออีกแค่จุดเดียวที่พัง... อ้าว

คุณสังเกตไหมครับว่าผมคอมเมนต์โค้ดบางส่วนสำหรับเทสต์ `remove_current` ทิ้งไป? ใช่แล้ว ผมไม่ได้สนใจเลยว่าเทสต์นี้มันมีการเปลี่ยนแปลงสถานะสะสม (stateful) งั้นเรามาสร้างลิสต์ตัวใหม่ที่มีสถานะแบบเดียวกับที่ฝั่ง `remove_current` ควรจะทิ้งไว้ให้เราดีกว่า:

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

เฮ้ ดูนั่นสิครับบ... โอเค ตอนนี้ผมชักเริ่มระแวงแล้วสิ เรามาเขียน `check_links` ให้สมบูรณ์แล้วรันเทสต์ใต้ Miri กันจริงจังเลยดีกว่า:

```rust ,ignore
fn check_links<T: Eq + std::fmt::Debug>(list: &LinkedList<T>) {
    let from_front: Vec<_> = list.iter().collect();
    let from_back: Vec<_> = list.iter().rev().collect();
    let re_reved: Vec<_> = from_back.into_iter().rev().collect();

    assert_eq!(from_front, re_reved);
}
```

นี่เป็นวิธีที่ดีที่สุดไหม? ไม่ แต่ใช้ได้ไหม? ได้เลย

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

เสร็จสิ้น!

เสร็จจริง ๆ แล้ว!

เราทำได้แล้วครับ! เราสร้าง `LinkedList` คุณภาพระดับ production ขึ้นมาสำเร็จ แถมยังมีฟังก์ชันการทำงานแทบจะถอดแบบมาจากตัวที่มีอยู่ใน std ทุกประการ ถามว่าเรายังขาด helper method เล็ก ๆ น้อย ๆ ตรงนั้นตรงนี้ไหม? ขาดแน่นอนครับ ถามว่าผมจะใส่มันเข้าไปในเวอร์ชันที่จะ publish ตัว crate ไหม? ก็น่าจะนะ!

แต่... ตอนนี้ผม... เหนื่อย... มาก... เหลือเกิน

เพราะงั้น... เราชนะแล้วครับ!

เดี๋ยวนะ เชี่ยเอ๊ย นี่เรากำลังทำโค้ดคุณภาพระดับ production อยู่นี่หว่า... โอเค บอสใหญ่ตัวสุดท้าย: Clippy

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

เอาล่ะ Clippy มาลุยกันเลย

คำบ่นที่ 1 (และ 3): เราใช้ `while let Some(_) = ` แทนที่จะเป็น `while .is_some()` ในเมื่อในลูปมันว่างเปล่า เรื่องนี้เลยไม่ได้สลักสำคัญอะไรเลยจริง ๆ แต่เอาเถอะ ได้เลย Clippy ผมยอมทำตามทางของนายก็ได้

คำบ่นที่ 2: เราดันไปเขียน inherent method ชื่อ `into_iter` ซะงั้น เดี๋ยวนะ อะไรนะ? *เปิดเช็คโค้ดของ std* โอเค Clippy ได้ไปหนึ่งแต้ม `IntoIterator` มันอยู่ใน prelude อยู่แล้ว (แถมแทบจะเป็น lang item ด้วยซ้ำ) ดังนั้นเราเลยไม่จำเป็นต้องมีเวอร์ชันที่เป็น inherent method ซ้ำซ้อนอีก

คำบ่นที่ 4: เราดันไปก๊อปปี้ความเชื่อผิด ๆ (cargo cult) แปลก ๆ มาจาก std ซะงั้น *\*ยักไหล่\** ได้เลย เดี๋ยวลบทิ้งให้

```text
cargo clippy
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
```

เยี่ยมเลย เหลือสิ่งสุดท้ายที่ต้องทำก่อนที่เราจะเรียกมันได้อย่างเต็มปากว่าคุณภาพระดับ production: `fmt`

```text
cargo fmt
```

...ก็นั่นแหละครับ มันแค่เพิ่มบรรทัดใหม่นิดหน่อยแล้วก็ตัดช่องว่างท้ายบรรทัดทิ้งไป ไม่มีอะไรน่าตื่นเต้น

**และตอนนี้ เราก็เสร็จสมบูรณ์จริง ๆ เสียที!!!!!!!!!!!!!!!!!!!!!**
