---
title: Arc<str> vs String, is Arc<str> really faster?
description: Analysis of Arc<str> vs String cloning performance in contentended scenarios.
pubDate: 'Dec 31 2023'
---
I was watching [The Primagen's][6] [reaction][5] to the video ["Use Arc Instead of Vec"][4] by
[Logan Smith][7], when after [experiencing live-demo-syndrome][8], Prime [raised the question][12] whether
or not cloning `Arc<str>` is always faster than cloning `String`, which I thought was a proposterous
question, since `malloc` is obviously slow, since that's what we're always told. "Reduce allocations"
is almost always the first thing to do when performing non datastructure-related optimisation.

The short (and possibly incorrect) answer is probably yes, cloning `Arc<str>` is faster
than `String` in most cases, but that's not what you're here for.

## How does this even work?
You might be wandering, "Why does this even work? I thought `str` always needs to be behind
`&` or `&mut`, so why does `Arc<str>` still work? Look at this error message."

Attempting to compile:
```rust
fn main() {
    let data: str = *"Hello, World!";
}
```
Results in this error:
```
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> error_str_size_not_known_at_runtime.rs:2:9
  |
2 |     let data: str = *"Hello, World!";
  |         ^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `Sized` is not implemented for `str`
  = note: all local variables must have a statically known size
  = help: unsized locals are gated as an unstable feature
help: consider borrowing here
  |
2 |     let data: &str = *"Hello, World!";
  |               +

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
```

Well the answer to that is pretty simple: `str` is a [Dynamically Sized Type][9] (or DST for short),
which is a result of it not implenting `Sized`, which means it can only exist behind a reference
*or a pointer*, dropping is handled by simply deallocating the pointer. Any pointer to a DST is
slightly different from normal pointer by the virtue of it being wide pointer, which the [Rustonomicon][10]
describes using the following words:

> Any pointer to a DST consequently becomes a _wide_ pointer consisting of the pointer and the information that "completes" them.

But believe it or not, but `str` isn't actually special, there are other DSTs such as `[T]` and `dyn Trait`.
The information stored in the wide pointer is either the length in the case of `[T]` and `str` or
the trait [VTable][13] in the case `dyn Trait`. But you can create your own DSTs as well, by having the last field of your struct be a DST.
Which is exactly what `Arc<T>` does, it's actually represented like this (omitting some irrelevant information):
```rust
pub struct Arc<
    T: ?Sized,
> {
    // For DSTs this turns into a wide pointer containing the necessary metadata
    // for ArcInner::<T>.data.
    ptr: NonNull<ArcInner<T>>,
    // a few Zero Sized Types follow this for some borrowing stuff and custom allocators.
}

// The ?Sized trait bound means the type is allowed to not implement Sized.
// By default all struct generics need to implement Sized.
struct ArcInner<T: ?Sized> {
    strong: atomic::AtomicUsize,
    weak: atomic::AtomicUsize,

    data: T,
}
```
This code was grabbed from the [docs][11] for `Arc<T>`.
(You can see the code for any item in a crate by just clicking the **source** button on its docs page.)

## Terminlogy
- `malloc` refers to the C function which allocates new memory.
- `free` refers to the C function which deallocates memory.
- `memcopy` as the name implies, copies memory between memory locations.

## What does "faster" even mean?
In this case we're talking about cloning performance, aka how long
it takes to call the `.clone()` function on the element.

In the case of `String` this means `malloc`ing a new string and
then `memcopy`ing the data over.
`Arc<str>` by comparison simply increments the strong reference counter.

## Expectations
You would expect a single increment to always be faster than a call
to `malloc` and then `memcopy`, since, that's supposedly a complex
call which should take much longer than what amounts to a single
instruction (in the case of x86 its `LOCK INC`).

## Methodology
I created a small benchmark program to test out my hypothesis.
If you want to check my answers for yourself, you can find it [on my github][3].
Since we're benchmarking performance here I assumed that you're going
to use `Jemalloc`, if you're trying to have the best possible performance.

Its pretty simple, the basic benchmark function looks like this:
```rust
#[global_allocator]
static GLOBAL: Jemalloc = Jemalloc;

fn bench_clone<T: Clone + Send>(base: T) {
    // Makes sure all threads are dead when we leave this function.
    // scope(f) returns the value returned by f.
    let elapsed = scope(|s| {
        // This is simply used to stop the spinning threads when we're done.
        let running = Arc::new(AtomicBool::new(true));

        // This spawns multiple threads spinning on clone to generate lock contention.
        // We iterate from 1..n to spawn n-1 threads, since one of our threads will
        // be used for benchmarking.
        for _ in 1..total_thread_count() {
            let local_base = base.clone();
            let local_running = running.clone();

            s.spawn(move || while local_running.load(Ordering::Relaxed) {
                run_clone(&local_base)
            });
        }

        let start = Instant::now();

        // This is the actual loop
        for _ in 0..samples() {
            run_clone(&base);
        }

        let elapsed = start.elapsed();

        running.store(false, Ordering::Relaxed);

        elapsed
    });

    println!("Took {}ms to perform {} clone operations on {}.",
        elapsed.as_millis(), samples(), std::any::type_name::<T>());
}

fn run_clone<T: Clone>(base: &T) {
    let c = base.clone();

    // `should_drop` returns a bool, which equates
    // to a bench having the suffix "_TRUE" or "_FALSE"
    // in the graph.
    if !should_drop() {
        std::mem::forget(c);
    }
}
```

### System specs
The system I tested this on has the following specs:
- CPU: Ryzen 5600X 6-Core CPU
- RAM: 2x8GB Crucial Ballistix CL16 3600 MHz
- OS: Arch Linux with 6.6.8-zen1-1-zen kernel

## Results
The left-most column shows the amount of threads used for the test. The next column shows
the cloned type `Arc<str>` or `String` and whether we drop the value after cloning. `_TRUE` means
we drop and `_FALSE` means we don't. Time is displayed in milliseconds.
![Chart showing performance differences between Arc<str>- and String-cloning](/blog/assets/arc_str_vs_string_graph.png)
As you can see, our expectations are only representative of reality, when a single
thread accesses the `Arc<str>`. As soon as multiple threads are contending
for the `Arc<str>`, we start having issues and the performance of `String`
surpasses the performance of `Arc<str>` significantly. Now the question
remains, why does it have much greater performance?

## Explanation
This information is based on `Jemalloc`, but may also be applicable to other multithreaded allocators.

Firstly, all references of `Arc<str>` refer to the same instance, which
means all clone operations operate on the same atomic integer, which, as
I alluded to before, causes significant lock contention. And yes even atomics use CPU-internal 
locks! For more information read Chapter 9.1 "Locked Atomic Operations"
(specifically 9.1.2 "Bus Locking") of Volume 3A "System Programming guide" of the
["Intel® 64 and IA-32 Architectures Software Developer's Manual"][1]
(Which is a 24mb pure-text PDF file. You can fit [three copies of Shrek][2] in that.).
`String` by contrast, calls `Jemalloc`s `malloc` implementation, which uses **thread-local** memory 
arenas to get around this.

What's still unexplained is why `String` gets *slower* when we *don't* drop
the `String` after each use. We would expect the call to `free` to take longer
than just leaving it there. This is simply explained by how `Jemalloc` uses
memory. As you might already know allocating memory from the kernel on Linux
happens using the `mmap` syscall, which takes time. Therefore the allocator
attempts to call `mmap` as little as possible, by putting multiple allocations
into a single `mmap`ed "bucket". Therefore, if we constantly `malloc` then
immediately `free` the allocated memory, we are always just using _already allocated_
**thread-local** "buckets" for our allocation, no syscalls involved!
If we instead leak the memory, the allocator now has to constantly allocate
new memory using `mmap`, which, while probably amortized using bigger allocations,
still takes quite a bit of time.

This can be seen in the following comparison using the `tikv_jemalloc_ctl::stats::mapped` API to get the amount
of mapped memory:

Mapped Memory when dropping:
```
Running on 8 threads concurrently.
Using strings of size 64.
Running 16777216 samples.
Dropping cloned elements.
Current mapped amount: 8.5 MB
Current mapped amount: 8.5 MB
[...]
Current mapped amount: 8.5 MB
Current mapped amount: 8.5 MB
Took 3125ms to perform 16777216 clone operations on alloc::sync::Arc<str>.
Current mapped amount: 8.6 MB
Current mapped amount: 29.8 MB
Current mapped amount: 29.8 MB
[...]
Current mapped amount: 29.8 MB
Current mapped amount: 29.8 MB
Took 308ms to perform 16777216 clone operations on alloc::string::String.
```

Mapped memory when not dropping:
```
Running on 8 threads concurrently.
Using strings of size 64.
Running 16777216 samples.
Not dropping cloned elements.
Current mapped amount: 8.5 MB
Current mapped amount: 8.5 MB
[...]
Current mapped amount: 8.5 MB
Current mapped amount: 8.5 MB
Took 1952ms to perform 16777216 clone operations on alloc::sync::Arc<str>.
Current mapped amount: 53.7 MB
Current mapped amount: 156.6 MB
Current mapped amount: 268.9 MB
Current mapped amount: 385.7 MB
Current mapped amount: 501.3 MB
Current mapped amount: 630.1 MB
[...]
Current mapped amount: 9.3 GB
Current mapped amount: 9.4 GB
Current mapped amount: 9.5 GB
Current mapped amount: 9.6 GB
Current mapped amount: 9.7 GB
Current mapped amount: 9.8 GB
Current mapped amount: 9.9 GB
Took 968ms to perform 16777216 clone operations on alloc::string::String.
```

As you can see, the amount of mapped memory when using `Arc<str>` is obviously
static, since it doesn't allocate memory, but the `String` allocates new memory
on each clone and if we don't drop the cloned `String`, we create a memory leak,
which means the allocator has to constantly request more memory using the `mmap`
syscall.

Verifying this can be done using the `strace` command (mmap with the MAP_ANONYMOUS flag allocates memory
and &| pipes STDERR to the STDIN of the next process in the pipeline):
```
# TOTAL_THREAD_COUNT=8 SHOULD_DROP=yes strace ./target/release/arc_str_perf &| grep "MAP_ANONYMOUS" | wc -l
18
# TOTAL_THREAD_COUNT=8 SHOULD_DROP=no strace ./target/release/arc_str_perf &| grep "MAP_ANONYMOUS" | wc -l
46
```

## Conclusion
In most cases (when the value is not **extremely** contended) cloning `Arc<str>` is definitely faster than
`String`, as you can see from the single-threaded result in the benchmark. But if you're cloning data at a
rate where performance is limited by CPU internal locking, I simply recommend you don't and try something else
instead of optimising your cloning. And always remember to follow the prime directive of optimisation, measure
everything, don't just apply "Golden Rules" and assume that everything's now magically faster.

## Notes
- Someone asked why I'm not including `Rc<str>` in this comparison, the reason is single-threaded `Arc<str>` has a pretty similar performance profile.

[1]: https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
[2]: https://old.reddit.com/r/AV1/comments/msb74a/shrek_but_its_8mb_again/
[3]: https://github.com/BlockListed/arc_str_perf/
[4]: https://www.youtube.com/watch?v=A4cKi7PTJSs
[5]: https://www.youtube.com/watch?v=OMmMfGaeRyI
[6]: https://www.youtube.com/@ThePrimeTimeagen
[7]: https://www.youtube.com/@_noisecode
[8]: https://www.youtube.com/watch?v=OMmMfGaeRyI&t=1552s
[9]: https://doc.rust-lang.org/nomicon/exotic-sizes.html#dynamically-sized-types-dsts
[10]: https://doc.rust-lang.org/nomicon/
[11]: https://doc.rust-lang.org/stable/std/sync/struct.Arc.html
[12]: https://www.youtube.com/watch?v=OMmMfGaeRyI&t=1779s
[13]: https://doc.rust-lang.org/reference/types/trait-object.html
