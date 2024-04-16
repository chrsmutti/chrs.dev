---
title: '"Clean" Code, Horrible Performance in Rust'
date: 2024-04-07T22:45:00-03:00
rss: true
slug: clean-code-rust
aliases:
  - 03-clean-code-rust
---

I was just recently going through some old episodes from
[Software Engineering Radio](https://se-radio.net/) when I came across
[this one episode](https://se-radio.net/2023/08/se-radio-577-casey-muratori-on-clean-code-horrible-performance/)
featuring [Casey Muratori](https://twitter.com/cmuratori), where he goes through
some of his thoughts around his video from February 2023, titled
["'Clean' Code, Horrible Performance"](https://www.youtube.com/watch?v=tD5NrevFtbU).
I was actually already aware of the video by this time, but listening through
the episode gave me an itch to see these concepts in my reality, experiment them
by myself.

I chose to rewrite the ideas Casey presented in the video in Rust, trying to be
idiomatic is not really the goal of the solutions you'll see below, but rather
experiment with how structuring the code may lead to a big performance penalty.

All the code I reference in this post is available in
[this Gist](https://gist.github.com/chrsmutti/698f979ad07be89ec3319dceb13565a9).

## Base Case: Traits

Starting with all the "Clean" Code principles, we would likely implement
something that looks like the code seen below:

```rust
trait Shape {
    fn area(&self) -> f32;
}

struct Square {
    side: f32,
}

impl Shape for Square {
    fn area(&self) -> f32 {
        self.side * self.side
    }
}

// ... the same for Rectangle, Triangle and Circle
```

Running the accumulator test Casey shows in the video, yields us a runtime of
`54,956 ns/iter` for `10240` items (more detailed information on the
[annex down below](#annex-1-benchmark-results)). We'll use this as our baseline
going forward.

```rust
    #[bench]
    fn total_area(b: &mut Bencher) {
        let shapes: Vec<Box<dyn Shape>> = init(COUNT);

        b.iter(|| {
            let mut accum = 0f32;
            for shape in &shapes {
                accum += shape.area();
            }

            accum
        });
    }
```

If you're more familiar with Rust you'll immediately catch the use of `Box`
there as a possible location for optimization. For our constraints, we do need
to box the values when constructing them as we need to have a "polymorphic
list", and because we're using a trait as our abstraction for `Shape`, the
values of our shapes vec are of unknown size to the compiler.

Our first performance gain comes without touching much of the implementation
code, but rather, the code in the "caller" side (the benchmark function). Using
4 accumulators instead of one:

```rust
    #[bench]
    fn total_area_sum4(b: &mut Bencher) {
        let shapes: Vec<Box<dyn Shape>> = init(COUNT);

        b.iter(|| {
            let mut accum1 = 0f32;
            let mut accum2 = 0f32;
            let mut accum3 = 0f32;
            let mut accum4 = 0f32;

            let count = COUNT / 4;
            let mut iter = 1;
            while count - iter > 0 {
                accum1 += shapes[iter * 4].area();
                accum2 += shapes[1 + (iter * 4)].area();
                accum3 += shapes[2 + (iter * 4)].area();
                accum4 += shapes[3 + (iter * 4)].area();

                iter += 1;
            }

            accum1 + accum2 + accum3 + accum4
        });
    }
```

This improves on the baseline by **3.1x**, executing at a runtime of
`17,725 ns/iter`. **Why?** A better utilization of CPU cycles. Modern CPUs are
able to handle parallel computations such as the one done above, but the
compiler may not take advantage of this if they're not easily recognizable.

Of course, the code above is not really something we can easily copy paste and
use in other solutions because there are a lot of assumptions we need to hold,
the biggest one being: Is count divisible by 4? But that is besides the point,
as we're mostly [bit twiddling](https://en.wikipedia.org/wiki/Bit_twiddler)
here.

## Still Idiomatic: Match

Going forward, Casey then breaks the "Polymorphism Rule" of "Clean" Code, and
writes the same implementation using a switch statement. In idiomatic Rust, we
can equate that to a match on a enum.

```rust
enum Shape {
    Square { side: f32 },
    Rectangle { width: f32, height: f32 },
    Triangle { base: f32, height: f32 },
    Circle { radius: f32 },
}

impl Shape {
    fn area(&self) -> f32 {
        match self {
            Shape::Square { side } => side * side,
            Shape::Rectangle { width, height } => width * height,
            Shape::Triangle { base, height } => 0.5f32 * base * height,
            Shape::Circle { radius } => PI * radius * radius,
        }
    }
}
```

The code here is still very readable, follows some very good conventions and is
something that you'd see every day in Rust. There is a drawback of this
implementation if you're writing a library: the dependents of said library would
not be able to extend the `Shape` system and add a `Hexagon` for example.

With the code structured this way, we see a whopping **3.2x** improvement from
the baseline when using the regular `for` loop, and a massive **8.5x**
improvement when using the `_sum4` variant (`17,141 ns/iter` and
`6,434 ns/iter`).

This is the knowledge I'll take from this experiment the most, because this is
still very idiomatic code, and knowing this performance difference is beneficial
when tackling performance critical code. This is where I can see most value from
the
["Performance-aware Programming"](https://www.computerenhance.com/p/welcome-to-the-performance-aware)
that Casey advocates for.

My assumptions is that the performance gains here come mostly from the removal
of the `Box` construct. Having the shapes represented as enum variants instead
of trait implementors makes their size known at compile time and reduces the
need of the `Box` to represent a "polymorphic list" (using quotes here is very
applicable, because this is not really a "polymorphic list" on the original
sense of the word).

## Breaking with Idiomatic Rust: Lookup Table

Moving forward with the optimizations we then come across using a lookup table
for the multiplier constants. If you look closely at the `match` code above,
you'll notice a pattern. Casey mentions this pattern in his video and also
claims that moving the code out of the Polymorphic variant and into the
`if-else/switch` style (or in our case the `match` style) highlights this
pattern.

Trying to still maintain some semblance of idiomatic Rust, I went with a
different approach for the lookup table, using a constructor function for
`Shape`s and storing the multiplier within the `struct` itself. This leaves us
with:

```rust
enum ShapeType {
    Square { side: f32 },
    Rectangle { width: f32, height: f32 },
    Triangle { base: f32, height: f32 },
    Circle { radius: f32 },
}

struct Shape {
    multiplier: f32,
    width: f32,
    height: f32,
}

impl Shape {
    fn new(shape_type: ShapeType) -> Self {
        match shape_type {
            ShapeType::Square { side } => Shape {
                multiplier: 1f32,
                width: side,
                height: side,
            },
            // ... the same for Rectangle, Triangle and Circle
        }
    }

    fn area(&self) -> f32 {
        self.multiplier * self.width * self.height
    }
}
```

Styling the code this way allows us to create shapes like so:

```rust
let shape = Shape::new(Square { side: 2f32 }) // Shape::Square { side: 2f32 }
```

This is not so different from what we saw in the more idiomatic examples using
`trait`s and `match`.

This code is **6.4x** faster than the baseline using the simple benchmark and
**11x** faster using the `_sum4` variant. In this example, we can see that the
sizes are known at compile time and there's probably some other optimization of
the instruction sent to the CPU that I'm not aware of.

## Breaking the "small functions" principle: `corner_area`

To break the "small functions that only do a single thing" mantra from "Clean"
Code, Casey introduces a `CornerCont()` information to the Shape `struct`. The
resulting value should be `1 / (1 + shape.CornerCount() + shape.Area())`.

Implementation wise they are very similar from what we've seen so far, and the
performance gains we seen before were also observed when calculating with the
`corner`.

The base case yielded a runtime of `54,302 ns/iter`. Changing the `trait`
implementation into a `match` statement, we see the performance jumping **4.4x**
this time (interestingly we don't see that much difference in the `_sum4`
variants, see the [annex for more information](#annex-1-benchmark-results)). The
lookup table implementation took the performance to new levels with a
performance boost of **6.4x** with the naive `for` and once again by **11x**
with the `_sum4` variant.

## Conclusion

Even though most of what we seen here is hardcore bit twiddling, being a
"Performance-aware Programmer" is not a bad thing. Most of the people reading
this (and me writing!) will most likely be developing Software that does not
necessarily needs to make these trade-offs for performance, but that's not the
point, at least not for me.

The point is: **we should be aware of them!**

We should always be aware when we're doing trade-offs and we should strive to
learn new things that challenge our status quo.

I'll probably not directly use many of the constructs I went through in this
post in my day-to-day job, but it sure as hell was fun researching and quickly
doing them this Sunday evening!

## Annex 1: Benchmark Results

For all the results we've seen thus far in this post, I used a
`Vec::with_capacity(10240)` and the `cargo bench` command, below you can see the
execution times:

```
$ cargo bench -- total_area
   Compiling clean_code v0.1.0 (/home/chrs/Workspace/clean_code)
    Finished `bench` profile [optimized] target(s) in 0.58s
     Running unittests src/lib.rs (target/release/deps/clean_code-3affde07f20d1563)

running 8 tests
test m01_trait::tests::total_area                  ... bench:      54,956 ns/iter (+/- 280)
test m01_trait::tests::total_area_sum4             ... bench:      17,725 ns/iter (+/- 1,373)
test m02_match::tests::total_area                  ... bench:      17,141 ns/iter (+/- 304)
test m02_match::tests::total_area_sum4             ... bench:       6,434 ns/iter (+/- 5,001)
test m03_table::tests::total_area                  ... bench:       8,560 ns/iter (+/- 169)
test m03_table::tests::total_area_sum4             ... bench:       5,734 ns/iter (+/- 1,005)
test m04_table_multiplier::tests::total_area       ... bench:       8,555 ns/iter (+/- 29)
test m04_table_multiplier::tests::total_area_sum4  ... bench:       5,002 ns/iter (+/- 82)

$ cargo bench -- corner_area
    Finished `bench` profile [optimized] target(s) in 0.01s
     Running unittests src/lib.rs (target/release/deps/clean_code-3affde07f20d1563)

running 8 tests
test m01_trait::tests::corner_area                 ... bench:      54,302 ns/iter (+/- 233)
test m01_trait::tests::corner_area_sum4            ... bench:      35,922 ns/iter (+/- 563)
test m02_match::tests::corner_area                 ... bench:      12,160 ns/iter (+/- 2,019)
test m02_match::tests::corner_area_sum4            ... bench:      12,128 ns/iter (+/- 41)
test m03_table::tests::corner_area                 ... bench:       8,554 ns/iter (+/- 23)
test m03_table::tests::corner_area_sum4            ... bench:       5,736 ns/iter (+/- 15)
test m04_table_multiplier::tests::corner_area      ... bench:       8,547 ns/iter (+/- 10)
test m04_table_multiplier::tests::corner_area_sum4 ... bench:       4,995 ns/iter (+/- 6)

test result: ok. 0 passed; 0 failed; 0 ignored; 8 measured; 8 filtered out; finished in 5.24s
```

You'll see a `m03_table` that has been omitted on the blog post just due to it
being very similar to the `m04_table_multiplier`, the `m03` module was my first
naive translation of the lookup table optimization (I've left in the gist if
you're curious).
