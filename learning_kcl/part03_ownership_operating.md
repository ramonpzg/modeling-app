# Part 03 — Ownership, borrowing, and how KCL stands aside

Ownership is the Rust gatekeeper. Python hides it; KCL abstracts it away through configuration immutability. You will practice moving, borrowing, and referencing while keeping mental anchors in Python and KCL. Four-plus reps per language keep the muscle memory forming.

## Moves versus copies
Python passes references; copying is explicit:
```python
def mutate(lst):
    lst.append(99)

nums = [1, 2, 3]
mutate(nums)
print(nums)
```
Rust moves by default; `Copy` types duplicate cheaply:
```rust
fn consume(v: Vec<i32>) {
    println!("{:?}", v);
}

fn main() {
    let data = vec![1, 2, 3];
    consume(data);
    // println!("{:?}", data); // move error
    let a: i32 = 5;
    let b = a; // Copy
    println!("{} {}", a, b);
}
```
KCL assigns values immutably; no move semantics, but you should think in terms of overlays rather than mutation:
```kcl
schema Data:
    items: [int]

original: Data = {items:[1,2,3]}
updated: Data = {items: original.items + [99]}

__main__:
    print(original.items)
    print(updated.items)
```
Python explicit copy to mimic move safety:
```python
import copy
values = [1,2,3]
cloned = copy.copy(values)
cloned.append(42)
print(values, cloned)
```
Rust cloning intentionally:
```rust
fn main() {
    let values = vec![1, 2, 3];
    let cloned = values.clone();
    println!("orig {:?} cloned {:?}", values, cloned);
}
```
KCL overlay alternative:
```kcl
schema Bag:
    items: [int]

bag: Bag = {items:[1,2,3]}
bag2: Bag = {items: bag.items + [42]}

__main__:
    print([bag.items, bag2.items])
```

## Borrowing and references
Python reference passing:
```python
def length(vec):
    return len(vec)

nums = [1,2,3]
print(length(nums))
print(nums)  # still accessible
```
Rust borrowing with `&T` and `&mut T`:
```rust
fn length(vec: &Vec<i32>) -> usize {
    vec.len()
}

fn push_one(vec: &mut Vec<i32>) {
    vec.push(1);
}

fn main() {
    let mut nums = vec![1, 2, 3];
    println!("{}", length(&nums));
    push_one(&mut nums);
    println!("{:?}", nums);
}
```
KCL access is immutable; mutation is via new values:
```kcl
schema Counter:
    nums: [int]
    len: int = len(nums)

c1: Counter = {nums:[1,2,3]}
c2: Counter = {nums:c1.nums + [1]}

__main__:
    print([c1.len, c2.len])
```
Python simulation of borrow checker with context management:
```python
class Borrowed:
    def __init__(self, data):
        self.data = data
    def __enter__(self):
        return self.data
    def __exit__(self, *args):
        self.data = None

with Borrowed([1,2,3]) as ref:
    print(len(ref))
```
Rust lifetime hint (preview):
```rust
fn first<'a>(items: &'a [i32]) -> Option<&'a i32> {
    items.first()
}

fn main() {
    let arr = [10, 20, 30];
    println!("{:?}", first(&arr));
}
```
KCL computed references via expressions:
```kcl
schema First:
    items: [int]
    first: int = items[0] if len(items) > 0 else -1

sample: First = {items:[10,20,30]}

__main__:
    print(sample.first)
```

## Slices and views
Python slicing shares backing storage for lists of lists but copies for slicing? Actually list slicing copies:
```python
nums = [0,1,2,3]
sub = nums[1:3]
sub[0] = 99
print(nums, sub)
```
Rust slices borrow view without owning:
```rust
fn sum(slice: &[i32]) -> i32 {
    slice.iter().sum()
}

fn main() {
    let data = [0,1,2,3];
    let view = &data[1..3];
    println!("{}", sum(view));
    println!("{:?}", data);
}
```
KCL slicing returns new list:
```kcl
nums: [int] = [0,1,2,3]
sub = nums[1:3]

__main__:
    print([nums, sub])
```
Python memoryview alternative:
```python
import array
buf = array.array('i', [1,2,3,4])
v = memoryview(buf)[1:3]
print(v.tolist())
```
Rust mutable slices:
```rust
fn bump(slice: &mut [i32]) {
    for n in slice.iter_mut() {
        *n += 10;
    }
}

fn main() {
    let mut data = [1,2,3,4];
    bump(&mut data[1..3]);
    println!("{:?}", data);
}
```
KCL derived arrays:
```kcl
arr: [int] = [1,2,3,4]
arr2 = [n + 10 if i in [1,2] else n for i, n in enumerate(arr)]

__main__:
    print([arr, arr2])
```

## Ownership gotchas
Python surprises with shared mutable defaults:
```python
def append_default(item, bucket=[]):
    bucket.append(item)
    return bucket

print(append_default(1))
print(append_default(2))  # same list reused
```
Rust prevents shared mutability by design:
```rust
fn append(item: i32, bucket: &mut Vec<i32>) {
    bucket.push(item);
}

fn main() {
    let mut bucket = Vec::new();
    append(1, &mut bucket);
    append(2, &mut bucket);
    println!("{:?}", bucket);
}
```
KCL default values evaluated per instantiation:
```kcl
schema Bucket:
    bucket: [int] = []

b1: Bucket = {bucket: append(bucket, 1)}
```
(This line would fail because KCL has no direct mutation; instead build new lists.)
```kcl
schema FixedBucket:
    bucket: [int] = []

b1: FixedBucket = {bucket: [1]}
b2: FixedBucket = {bucket: [2]}

__main__:
    print([b1.bucket, b2.bucket])
```
Python context manager to simulate borrow rules:
```python
from contextlib import contextmanager

@contextmanager
def immutable_view(data):
    yield tuple(data)

with immutable_view([1,2,3]) as view:
    print(view)
```
Rust Rc/RefCell preview for interior mutability:
```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let shared = Rc::new(RefCell::new(vec![1,2,3]));
    {
        let mut borrow = shared.borrow_mut();
        borrow.push(4);
    }
    println!("{:?}", shared.borrow());
}
```
KCL avoids interior mutability; think overlay again:
```kcl
schema Shared:
    values: [int]

base: Shared = {values:[1,2,3]}
extended: Shared = {values: base.values + [4]}

__main__:
    print(extended.values)
```

## Exercises
1. For each Rust snippet, write the equivalent Python and KCL behavior, then annotate where Rust’s compiler prevented a bug.
2. Modify the borrow examples to hold strings and observe lifetime inference changes.
3. Implement a Rust function returning a borrowed value from a struct; then mirror the idea in KCL by exposing a computed field.
4. Create a small table comparing when data is copied, moved, or shared in each language based on these snippets.


## Extra code reps (ownership and moves)

Python (4 blocks):
```python
# 1) Passing lists to functions (mutates original)
def append_item(xs: list[int]) -> None:
    xs.append(99)

nums = [1, 2]
append_item(nums)
print(nums)
```
```python
# 2) Copy vs reference
import copy
src = {"a": [1, 2]}
shallow = copy.copy(src)
deep = copy.deepcopy(src)
shallow["a"].append(3)
print(src, shallow, deep)
```
```python
# 3) Context manager ensures release
with open("/tmp/own.txt", "w") as f:
    f.write("ownership ends at block exit")
```
```python
# 4) Generator consuming iterator
nums = (n for n in range(3))
print(list(nums))
# nums is now exhausted
```

Rust (4 blocks):
```rust
// 1) Move vs borrow
fn move_vec(mut v: Vec<i32>) {
    v.push(99);
    println!("{:?}", v);
}

fn main() {
    let nums = vec![1, 2];
    move_vec(nums);
    // println!("{:?}", nums); // would fail: moved
}
```
```rust
// 2) Clone to keep ownership
fn clone_vec() {
    let nums = vec![1, 2];
    let copy = nums.clone();
    println!("orig: {:?} copy: {:?}", nums, copy);
}
```
```rust
// 3) Borrowing with lifetimes
fn sum_slice(xs: &[i32]) -> i32 {
    xs.iter().sum()
}
```
```rust
// 4) Iterator consumption
fn exhaust_iter() {
    let mut nums = (0..3);
    let collected: Vec<i32> = nums.by_ref().collect();
    println!("{:?}", collected);
    println!("Remaining count: {}", nums.count());
}
```

KCL (4 blocks):
```kcl
# 1) Rebinding imitates move
nums = [1, 2]
other = nums
# nums is still accessible, but we treat rebinding as a move mental model
print(other)
```
```kcl
# 2) Copy via slicing
nums = [1, 2]
copy = nums[:]
nums = nums + [3]
print(nums, copy)
```
```kcl
# 3) Functions return new values
sum_list = lambda xs: sum(xs)
print(sum_list([1,2,3]))
```
```kcl
# 4) Iterator-like comprehension consumption
nums = [n for n in range(3)]
print(nums)
```
