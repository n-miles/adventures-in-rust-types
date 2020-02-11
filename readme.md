I'm trying to build a [k-d tree](https://en.wikipedia.org/wiki/K-d_tree) in Rust. It's essentially
a binary search tree for higher-dimensional spaces. They're good for doing things like finding places
close to other places on a 2D map or RGB colors that are close to other RGB colors (3 dimensions).
A k-d tree works equally well no matter how many dimensions are involved in the problem, and
implementations work the same way no matter how many dimensions are involved.

To this end, I want to define a `KDTree` type that is generic over the input type, and for the
input type to have a way to get an array of its dimensions. So for a 2D `Point` type and a 3D `Color`
type, I want to be able to interact with them like the following. I'm leaving the return type of the
method on `Dimensional` blank since that's the crux of the issue. I'll try to define it as we go on.
```rust
fn main() {
    let point = Point{x: 1, y: 2};
    let point_dimensions = point.dimensions();
    println!("{:?} has {} dimensions: {:?}", point, point_dimensions.len(), point_dimensions);

    let color = Color(1, 2, 3);
    let color_dimensions = color.dimensions();
    println!("{:?} has {} dimensions: {:?}", color, color_dimensions.len(), color_dimensions);
}

#[derive(Debug)]
struct Point {
    x: i32,
    y: i32
}

#[derive(Debug)]
struct Color(u8, u8, u8);

trait Dimensional {
    fn dimensions(&self) -> ????
}

impl Dimensional for Point {
    fn dimensions(&self) -> [i32; 2] {
        [self.x, self.y]
    }
}

impl Dimensional for Color {
    fn dimensions(&self) -> [u8; 3] {
        [self.0, self.1, self.2]
    }
}
```
This program should print
```
Point { x: 1, y: 2 } has 2 dimensions: [1, 2]
Color(1, 2, 3) has 3 dimensions: [1, 2, 3]
```
So what type can we put for `????`? We need a generic array type of some kind. Being generic over the element
type is relatively straight-forward.
```rust
trait Dimensional {
    type Units: PartialOrd + PartialEq;
    fn dimensions(&self) -> [Self::Units; ????];
}
```
and now `impl` blocks look like this:
```rust
impl Dimensional for Point {
    type Units = i32;

    fn dimensions(&self) -> [i32; 2] {
        [self.x, self.y]
    }
}

impl Dimensional for Color {
    type Units = u8;

    fn dimensions(&self) -> [u8; 3] {
        [self.0, self.1, self.2]
    }
}
```
But how can we specify the length of the array? I would _like_ to write the following trait definition:
```rust
trait Dimensional {
    type Units: PartialOrd + PartialEq;
    const DIMENSION_COUNT: usize;

    fn dimensions(&self) -> [Self::Units; Self::DIMENSION_COUNT];
}
```
however, compiling this fails with the following error:
```
error: constant expression depends on a generic parameter
  --> src\main.rs:55:29
   |
55 |     fn dimensions(&self) -> [Self::Units; Self::DIMENSION_COUNT];
   |                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: this may fail depending on what value the parameter takes
```
Turns out that Rust isn't generic over array size yet. But nightly has an unstable marker trait that all array
types with <= 32 elements implement named `FixedSizeArray<T>`. This trait definition gets us most of the way
there. It forces implementations to specify the type of the array rather than deriving the array type from the
element type and size.
```rust
trait Dimensional {
    type Units: PartialOrd + PartialEq;
    type ArrayType: core::array::FixedSizeArray<Self::Units>;

    fn dimensions(&self) -> Self::DimensionsArrayType;
}
```
The `impl` blocks now look like this:
```rust
impl Dimensional for Point {
    type Units = i32;
    type ArrayType = [i32; 2];

    fn dimensions(&self) -> [i32; 2] {
        [self.x, self.y]
    }
}

impl Dimensional for Color {
    type Units = u8;
    type ArrayType = [u8; 3];

    fn dimensions(&self) -> [u8; 3] {
        [self.0, self.1, self.2]
    }
}
```
The original main method now works! This:
```rust
fn main() {
    let point = Point{x: 1, y: 2};
    let point_dimensions = point.dimensions();
    println!("{:?} has {} dimensions: {:?}", point, point_dimensions.len(), point_dimensions);

    let color = Color(1, 2, 3);
    let color_dimensions = color.dimensions();
    println!("{:?} has {} dimensions: {:?}", color, color_dimensions.len(), color_dimensions);
}
```
now prints the desired output of
```
Point { x: 1, y: 2 } has 2 dimensions: [1, 2]
Color(1, 2, 3) has 3 dimensions: [1, 2, 3]
```
So now that we have the type signatures worked out, let's try to write a generic function to do the printing.
Non-generic code that still prints the desired output looks like this:
```rust
fn main() {
    print_point_dimensions(Point{x: 1, y: 2});
    print_color_dimensions(Color(1, 2, 3));
}

fn print_point_dimensions(point: Point) {
    let point_dimensions = point.dimensions();
    println!("{:?} has {} dimensions: {:?}", point, point_dimensions.len(), point_dimensions);
}

fn print_color_dimensions(color: Color) {
    let color_dimensions = color.dimensions();
    println!("{:?} has {} dimensions: {:?}", color, color_dimensions.len(), color_dimensions);
}
```
so let's try to generify that function.
```rust
fn main() {
    print_dimensions(Point{x: 1, y: 2});
    print_dimensions(Color(1, 2, 3));
}

fn print_dimensions<T: Dimensional>(thing: T) {
    let thing_dimensions = thing.dimensions();
    println!("{:?} has {} dimensions: {:?}", thing, thing_dimensions.len(), thing_dimensions);
}
```
But this doesn't work. I get this error:
```
error[E0599]: no method named `len` found for associated type `<T as Dimensional>::DimensionsArrayType` in the current scope
  --> src\main.rs:11:70
   |
11 |     println!("{:?} has {} dimensions: {:?}", thing, thing_dimensions.len(), thing_dimensions);
   |                                                                      ^^^ method not found in `<T as Dimensional>::DimensionsArrayType`
```
It turns out that the `FixedSizeArray` trait doesn't actually accomplish the goal of generifying the `Dimensional`
trait. When we aren't being generic, the compiler knows the concrete type that `Point` and `Color` specify for
`ArrayType`. But `FixedSizeArray` isn't magic. It's just a normal trait. It looks like this (abbreviated for
clarity):
```rust
pub trait FixedSizeArray<T> {
    fn as_slice(&self) -> &[T];
    fn as_mut_slice(&mut self) -> &mut [T];
}
```
but we can't call those methods because they aren't `pub`. All this means that using `FixedSizeArray`, there's no
way to get either the actual array or a slice of the array in generic code. So basically, I can't find a way to
interact with an array of a generic size. There is [a crate](https://docs.rs/generic-array/0.13.2/generic_array/)
to do this, but if I had used the crate, I wouldn't have learned a whole lot about Rust's type system. Also, almost
every trait and function in that crate is `unsafe` and `unsafe` skeeves me out.