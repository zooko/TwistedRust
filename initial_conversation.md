# Nano-GPT.com

## User
##### 5/2/2026, 8:31:13 AM

Rust concurrency — how does async/await work?

---

## Assistant
##### 5/2/2026, 8:31:13 AM

## The core idea

`async`/`await` in Rust is **compile-time transformation of a function into a state machine** that implements the`Future` trait. There is no built-in scheduler, no green threads, no garbage collection — just a type the compiler generates for you.

```rust
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```
`Poll` is either`Ready(T)` or`Pending`. That's the entire contract.

## What`async fn` compiles to

An`async fn` does **not** execute when called. It returns an anonymous struct (an`impl Future`) that captures its arguments and locals. Every`.await` point is a state in the generated state machine; locals live across awaits become fields of that struct.

```rust
async fn foo(x: u32) -> u32 {
    let a = bar().await;
    a + x
}
```

roughly becomes (conceptually):

```rust
enum FooState {
    Start { x: u32 },
    AwaitingBar { x: u32, fut: BarFuture },
    Done,
}
```
`poll` is a generated function that matches on the state, drives sub-futures, and transitions states. This is why **futures are inert** — nothing happens until something calls`poll`.

## Who calls`poll`

A **runtime** (executor + reactor):`tokio`,`smol`,`async-std`,`embassy`, or your own. The standard library deliberately ships no runtime. The executor:

1. Owns a queue of ready tasks.
2. Calls`poll` on each.
3. If`poll` returns`Pending`, the future has arranged for a`Waker` (carried in`Context`) to be invoked when it can make progress.
4. The reactor (epoll/kqueue/io_uring/IOCP) calls`waker.wake()` when the OS reports the I/O is ready, which re-enqueues the task.

So`Pending` +`Waker` is the cooperative yield mechanism. No thread is parked per task; one OS thread can drive thousands of tasks.

## What`.await` actually does

`x.await` desugars to (roughly):

```rust
loop {
    match Pin::new(&mut x).poll(cx) {
        Poll::Ready(v) => break v,
        Poll::Pending => yield, // return Pending up the stack
    }
}
```

The`yield` returns`Pending` from the enclosing`poll`, after saving the state machine's current state. When the waker fires, the executor calls`poll` again and the state machine resumes at that point.

## Concurrency vs parallelism

Sequential awaits are sequential:

```rust
let a = f().await;
let b = g().await; // doesn't start until f completes
```

For concurrency within one task, combine futures:

```rust
let (a, b) = tokio::join!(f(), g());        // both progress concurrently
let winner = tokio::select! { ... };         // race
```

For parallelism across threads, **spawn** onto the runtime (`tokio::spawn`), which moves the future to the executor's task pool where worker threads can poll different tasks simultaneously. That's why spawned futures usually require`Send + 'static`.

##`Pin` — why it exists

The generated state machine is **self-referential**: a local`let s = String::from("hi"); let r = &s; other().await; use(r);` produces a struct where one field borrows another. Moving that struct after polling would invalidate the internal reference.`Pin<&mut F>` is the type-system promise "this won't move again," which is exactly what`poll` requires.

## Mental model

-`async fn` = sugar for "build a state machine struct."
-`.await` = sugar for "loop polling, yielding`Pending` upward."
-`Future` = a single`poll` method.
-`Waker` = the callback the future hands to whatever it's blocked on.
- Runtime = the loop that calls`poll` and the reactor that fires wakers.
- Concurrency is in the future combinators; parallelism is in`spawn`.

## Costs worth knowing

- The state machine is sized to the largest live set across all await points; deeply nested`async fn` can produce surprisingly large futures. Boxing (`Box::pin`) trades size for an indirection.
- Holding a`std::sync::Mutex` guard across`.await` can deadlock the executor — use`tokio::sync::Mutex` if a lock must span an await, or restructure to drop the guard first.
- Calling blocking syscalls (including`std::thread::sleep`, blocking file I/O, CPU-heavy loops) on an executor thread stalls every other task on that thread. Use`spawn_blocking` for those.
-`async` functions are lazy — calling one without awaiting or spawning does literally nothing.

---

## User
##### 5/2/2026, 8:33:18 AM

Rust concurrency paradigms. I want to write a new network server in Rust. I hate multithreading — based on my long experience I think it is a perennial source of bugs and performance problems (*even* though Rust's memory safety and concurrency features help with that. It's still not enough to be better than the alternatives). I'm kind of meh about async/await, because when reading a function, I'd like to know that *all* of the lines in that function get executed in order. If there is an `await` keyword in that function, then I have to split it into before and after in my mind.

So, what's the state of the art in writing a server in Rust using *only* one thread, and *only* synchronous functions. :-)


---

## Assistant
##### 5/2/2026, 8:33:18 AM

## The honest tension in your requirements

"Single-threaded, fully synchronous, scales to many connections" is a pick-two situation:

- One thread + truly blocking syscalls = one connection at a time. Fine for an admin socket, useless for a server.
- Many connections on one thread = something, somewhere, has to multiplex. The only question is whether that multiplexing is visible in your source code or hidden under it.

`async`/`await` makes the multiplexing visible (the`.await` you dislike). Threads make it invisible but bring the bugs you're rightly wary of. The third option — and the one that actually fits what you're asking for — hides multiplexing the way threads do, but inside a single OS thread.

## The state of the art that matches your taste: stackful coroutines

This is the "Go model": code looks and reads completely synchronously, with no`await` keyword. When you call`stream.read(&mut buf)?`, if data isn't ready, the runtime parks *that coroutine's stack* and runs another one. When the OS says the fd is readable, your coroutine resumes exactly where it left off. There is no syntactic mark on I/O points.

In Rust, the mature crate is **`may`** (and its ecosystem:`may_minihttp`,`may_postgres`, etc.).

```rust
use may::net::TcpListener;
use std::io::{Read, Write};

fn main() {
    may::config().set_workers(1).set_io_workers(1); // single-threaded
    let listener = TcpListener::bind("0.0.0.0:8080").unwrap();
    while let Ok((mut stream, _)) = listener.accept() {
        may::go!(move || {
            let mut buf = [0u8; 4096];
            loop {
                let n = stream.read(&mut buf).unwrap();
                if n == 0 { return; }
                stream.write_all(&buf[..n]).unwrap();
            }
        });
    }
}
```

Every line in that handler executes in order, top to bottom. There is no`async`, no`.await`, no`Future`, no`Pin`. The`read` call really does block — but it blocks *the coroutine*, not the OS thread. With`set_workers(1)` you get a single OS thread cooperatively running thousands of coroutines.

What you give up:

- No`Send`/`Sync` compile-time enforcement of the kind async Rust gives you. The borrow checker still works, but you do need to be careful with shared state across coroutines (though one-thread mode eliminates data races by construction).
- A bit of`unsafe` under the hood for stack switching. The crate has been around for years and is solid, but it's not "blessed" the way`tokio` is.
- Stack memory per coroutine (configurable, default ~4KiB-ish — much smaller than an OS thread's 8MiB but larger than an async task's state machine).
- Some libraries assume Tokio. You can't pull in a random`reqwest` and have it cooperate; you want crates from the`may_*` family or plain blocking crates that operate on`std::io::Read`/`Write` (which`may`'s socket types implement).

## The runner-up: explicit event loop with`mio`

If you want zero hidden control flow, you write the event loop yourself:`mio::Poll` wraps epoll/kqueue/IOCP, you register sockets, you get readiness events, you advance a state machine per connection. This is maximally explicit and maximally tedious. Every connection becomes an`enum ConnState { ReadingHeader, ReadingBody { … }, WritingResponse { … } }`. You will hand-write the same state machine the async compiler would have generated for you. People do this for specialized servers (databases, proxies) where they want exact control of buffers and scheduling.

This satisfies "synchronous functions" only in the trivial sense that there's no`async` keyword. The control flow per connection is *more* fragmented than async/await, not less.

## What I would not recommend for your case

- **Plain blocking`std::net` on one thread.** Handles one client. Done.
- **Thread-per-connection.** You ruled it out, and you're right to: it doesn't scale and it's the bug factory you've been burned by.
- **`async-std` /`smol` /`tokio` with a single-threaded runtime.** Yes it's one thread, but it's the`async` model you said you don't want.
- **`glommio` /`monoio`.** Thread-per-core io_uring runtimes. Excellent perf, but async-based.

## Honest recommendation

Use **`may`** with`set_workers(1)`. It is the closest thing in current Rust to "write straight-line synchronous code, run on one thread, scale to many connections." You'll read code top to bottom the way you want, and the multiplexing happens silently inside`read`/`write`/`accept`.

Two caveats to internalize before committing:

1. The "every line executes in order" property holds *within a coroutine*. Across coroutines, control still interleaves at I/O points — there is no escaping that if you want concurrency on one thread. The difference from async is that the interleaving points aren't marked in the source. Some people find that more readable; others find it scarier than`await` precisely because it's invisible. Try a small prototype before scaling up.
2. Ecosystem gravity is firmly with Tokio. If your server needs TLS, HTTP/2, gRPC, a Postgres driver, etc., check that`may`-compatible (or pure-blocking) crates exist for what you need. For a custom protocol over plain TCP, you're fine.

If after a prototype the invisible yield points bother you as much as`await` does, the next-best fit is`mio` plus hand-written state machines — at which point you're really just writing async-by-hand and Tokio starts looking attractive again.

---

## User
##### 5/2/2026, 8:44:52 AM

Show me a rust function named `do_something` that takes an argument named `foo`, and then it interacts with foo (either by invoking a method on it if it is an object or by using its value if it is a reference to a value), and stores the result in local variable named `x`, and then interacts with `foo` again and stores the result in local variable `y`. The point is, what can we conclude about the *consistency* of `x` with `y` after? If any concurrency mechanism can change the state of the thing that `foo` references between the first and second interaction can cause the resulting `x` and `y` to be inconsistent.

---

## Assistant
##### 5/2/2026, 8:44:52 AM

## The function

```rust
fn do_something(foo: &Foo) {
    let x = foo.read();
    let y = foo.read();
    // What is the relationship between x and y?
}
```

## The answer depends entirely on the *type* of`foo`, and Rust makes this answerable just by looking at the signature

This is one of the genuinely beautiful things about Rust, and it's a sharp contrast with C, C++, Java, Go, etc. In those languages you generally cannot answer the consistency question without reading the implementation of`read` *and* surveying every other thread in the program. In Rust the signature alone usually settles it.

### Case 1:`foo: &mut Foo`
```rust
fn do_something(foo: &mut Foo) {
    let x = foo.read();
    let y = foo.read();
}
```
`x` and`y` are **guaranteed consistent**, period. The`&mut` is exclusive: while`do_something` holds it, the borrow checker has statically proven that *no other reference of any kind* to that`Foo` exists anywhere in the program. No thread, no coroutine, no async task, no signal handler, no recursive call through a re-entrant API can touch it. Threads don't even enter the discussion.

### Case 2:`foo: Foo` (owned)

Same conclusion as Case 1 and for the same reason: nobody else has a reference.

### Case 3:`foo: &Foo` where`Foo` has no interior mutability

```rust
struct Foo { n: u32 }                       // plain data
fn do_something(foo: &Foo) {
    let x = foo.n;
    let y = foo.n;
}
```
`x` and`y` are **guaranteed consistent**. A`&T` in Rust statically guarantees that no`&mut T` exists concurrently *anywhere*, on any thread. Without a`&mut`, the value cannot legally be mutated. Other threads holding their own`&Foo` are fine — they can only read too.

This is the property C++`const&` *pretends* to have but doesn't.

### Case 4:`foo: &Foo` where`Foo` *does* have interior mutability

```rust
struct Foo { n: AtomicU32 }                 // or Mutex<…>, RefCell<…>, Cell<…>
fn do_something(foo: &Foo) {
    let x = foo.n.load(Ordering::Relaxed);
    let y = foo.n.load(Ordering::Relaxed);
}
```

Here`x` and`y` may **differ**. Interior mutability is exactly the opt-in that lets`&T` permit mutation:

-`AtomicU32`,`Mutex<T>`,`RwLock<T>` — another thread can mutate between the two reads (these types are`Sync`).
-`Cell<T>`,`RefCell<T>` — single-threaded interior mutability; another function on the same thread can mutate via re-entrancy (a callback you invoked, a`Drop`, a panic-unwind path, an async task on the same executor that ran in between if you yielded — though in this code there's no yield point).

So even on a single thread,`&Cell<T>` is enough to break consistency if`read()` itself can re-enter your code.

## The single rule that captures it

> If the signature is`&mut T`, or`T`, or`&T` where`T` contains no`UnsafeCell`, then`x` and`y` are consistent. Otherwise they are not, and the type telling you so is the point.

`UnsafeCell<T>` is the *only* primitive in the language that permits mutation through a shared reference;`Cell`,`RefCell`,`Mutex`,`RwLock`,`Atomic*` are all built on it. So "does this type transitively contain an`UnsafeCell`?" is *the* question, and the answer is visible from the type definition.

## Why this matters for your server design

This is the property that survives whether you choose threads, async, coroutines (`may`), or hand-rolled`mio`. In any of those models, when you read a function and see`fn handle(&self, conn: &Connection)`, and`Connection` is plain data, you know — *from the signature alone, with no global reasoning* — that two reads of the same field give the same answer. You only have to start worrying when you see`Mutex`,`Atomic*`,`Cell`, or`RefCell` in the type.

That's the actual answer to "can concurrency make`x` inconsistent with`y`": **only if the type system told you it could.** Threads and async don't change this; they just change *which* interior-mutability primitives are available (`Sync` ones for threads,`Cell`/`RefCell` are enough on a single thread).

---

## User
##### 5/2/2026, 8:52:36 AM

How is the presence of `UnsafeCell` transitively visible through the type declaration? It's not visible in the declaration of `do_something`. I would have to go read the type declaration of every object reachable from the arguments to `do_something`, right?

---

## Assistant
##### 5/2/2026, 8:52:36 AM

## You're right that it isn't in the signature — but you don't have to walk the type graph by hand

You've identified a real thing:`UnsafeCell` is not surfaced in the signature of`do_something`. There's no`UnsafeCell` marker in`fn do_something(foo: &Foo)`. To know whether`Foo` transitively contains one, *somebody* has to look through`Foo`'s fields, their fields, and so on.

The good news is that the compiler already does that walk for you, and exposes the answer as auto traits. The bad news is that the answer is exposed somewhat indirectly, and reading it off the signature takes a little practice.

## The auto trait that tracks it:`Freeze`

The compiler computes, for every type, whether it transitively contains an`UnsafeCell`. The internal name for "does **not** transitively contain an`UnsafeCell`" is the auto trait`Freeze`. It is automatically implemented for every type whose fields are all`Freeze`, and it is automatically *not* implemented the moment an`UnsafeCell` appears anywhere in the transitive field graph.

The catch:`Freeze` is currently unstable — you cannot write`where Foo: Freeze` in stable Rust. It exists in the compiler and governs important things (e.g., whether a`&T` can point into read-only memory), but it isn't a tool you can wield directly in your own bounds yet.

So in practice, the *direct* "is this guaranteed immutable through a shared reference?" predicate is computed but not nameable.

## What you *can* read off the signature on stable Rust

Two related auto traits are stable, and together they cover most real cases:

- **`Sync`**: "`&T` is safe to share across threads." Types with thread-safe interior mutability (`Mutex`,`RwLock`,`Atomic*`) are`Sync`. Types with thread-*unsafe* interior mutability (`Cell`,`RefCell`) are`!Sync`. Plain data is`Sync`.
- **`Send`**: "`T` can be moved to another thread." Most things are`Send`;`Rc`,`MutexGuard` (on some platforms), and raw pointers are not.

Neither is exactly`Freeze`. A`Mutex<T>` is`Sync` *and* contains`UnsafeCell`. So`Sync` does not mean "no interior mutability." But the combination with the rest of the signature is usually enough:

```rust
fn do_something<T: Sync + Copy>(foo: &T) { … }
```

Here`T: Copy` excludes`Mutex`,`RefCell`,`Cell<NonCopy>`, etc. (none of those are`Copy`), and what remains that is both`Sync` and`Copy` and offers interior mutability is essentially the`Atomic*` family — which is *not*`Copy` either. So in practice`T: Sync + Copy` gives you "plain immutable data" with high confidence, just from the bound.

For concrete (non-generic) parameters like`foo: &Foo`, the compiler still knows; you just can't express it as a bound. You read it off the type.

## So how do you actually answer the question for a concrete type?

Three practical strategies, in increasing order of rigor:

**1. The naming convention.** The interior-mutability primitives in`std` all advertise themselves in their names:`Cell`,`RefCell`,`OnceCell`,`UnsafeCell`,`Mutex`,`RwLock`,`Atomic*`,`OnceLock`. If none of those appear anywhere in the type's transitive definition, it's plain data. This is the convention 99% of code follows, including most of the ecosystem. It's not enforced by the compiler, but it's strong enough that experienced Rust programmers rely on it.

**2. Read the struct definition once.** In Rust, types are usually shallow and field-private. To know whether`Foo` has interior mutability, you open`Foo`'s definition and look at the field types. If they're all primitives or other plain structs, you're done. If one of them is a`Bar`, you glance at`Bar`. In practice this terminates in a few hops because Rust types tend to be tree-shaped, not graph-shaped, and leaves are primitives. This sounds tedious in the abstract; in real codebases it's a 10-second check, and only when you actually care.

**3. Ask the compiler with a static assertion.** When it really matters, force the compiler to do the transitive walk and fail compilation if interior mutability sneaks in:

```rust
const _: fn() = || {
    fn assert_sync<T: Sync>() {}
    assert_sync::<Foo>();
};
```

That catches`Cell`/`RefCell`. To catch`Mutex`/`Atomic*` too, the cleanest stable proxy is to require`Copy`:

```rust
const _: fn() = || {
    fn assert_plain<T: Copy + Sync>() {}
    assert_plain::<Foo>();
};
```

If`Foo` compiles under that, it has no interior mutability anywhere in its transitive structure (because no interior-mutability primitive in`std` is`Copy`). On nightly,`T: Freeze` says it directly.

## The honest summary

You're correct: the consistency property of`x` and`y` is determined by something — transitive presence of`UnsafeCell` — that is *not* printed in the signature of`do_something`. Rust's compiler tracks it precisely (as the`Freeze` auto trait) but does not yet expose it as a stable bound you can name. In stable Rust today, you rely on (a) the strict naming convention of interior-mutability types, (b) a quick look at the struct definition when it matters, or (c) a`Copy + Sync` static assertion as a stand-in.

That's a real ergonomic gap compared to the ideal "the signature alone tells you everything." The previous answer overstated the case; the corrected version is: **the *type* tells you everything, and in practice the type is one click away from the signature.** Threads and async still don't change the answer — only the type does — but you do have to glance at the type.

---

## User
##### 5/2/2026, 8:59:21 AM

Okay, so it is _not_ practical, in practice, for me to come along and read the declaration and the body of `do_something`, and know whether or not `x` and `y` are guaranteed consistent. I would have to use a not-yet-stable Rust feature, or read the definitions of the arguments, or something. Fine, so I'm convinced that the Rust type system (while great in some ways) does not solve this particular need for me. Now let me come at the same question a different way. Say `do_something` invokes `let x = f().await; let y = g();`. Or, on the other hand, say that the function `do_something2` invokes `let x = f(); let y = g();`. Since `do_something` allows other computations to happen in the system after `f()` returns and before `g()` runs, then *something* must possibly be changed in the system, and that change *might* affect the result of `g()`! Now we've already found one such case: it depends on there being interior mutability somewhere in the system. But, even if we could somehow track and constrain the use of interior mutability effecting the outcome of `g()`, I really don't think there is any way to guarantee that `g()` is going to return the same thing whether or not any other computation happened in the system! That's a stretch. In other words, the reader of the code can have a much simpler job in his head when reading `do_something2` and reasoning about the consistency of `x` and `y` than when reading `do_something`. Right? Or am I wrong?

---

## Assistant
##### 5/2/2026, 8:59:21 AM

## You're right, and the reason is broader than interior mutability

You've identified the real issue. Let me sharpen it.

```rust
async fn do_something()  { let x = f().await; let y = g(); }
fn       do_something2() { let x = f();       let y = g(); }
```

In`do_something2`, between`f()` returning and`g()` starting, **nothing else in this thread runs**. The set of things that could have changed the world is exactly: other threads (if any), signals, the OS, hardware, and re-entrancy through callbacks that`f` itself invoked. If you're on a single thread with no signals and`f` didn't call user code, the answer is "nothing changed,"and that conclusion is local — you got it from reading these two lines plus knowing the threading model of the program.

In`do_something`, between`f().await` resuming and`g()` starting, **arbitrary other tasks on this executor may have run to completion or partway**. Each of those tasks is some other`async fn` somewhere else in the program. They had full access to whatever shared state your runtime allows them to reach. To reason about what`g()` returns, you have to consider the union of effects of every task the executor might have scheduled in that window.

That is a categorically larger reasoning burden, and it is not just about interior mutability. It's about **what code ran**.

## Why "even without interior mutability" is correct

Consider`g()` reads a file, queries a database, reads the system clock, checks the size of a channel, looks at`/proc`, asks the allocator for stats, observes how many connections are open. None of that involves`UnsafeCell` in your process's type graph at all. Yet all of it can change because some other task ran in between. Interior mutability governs consistency of **in-memory values reachable through Rust references**. It says nothing about the rest of the universe — kernel state, peer processes, the network, the disk, the clock — which`g()` may very well consult.

So your generalization is right: even if Rust's type system *did* perfectly surface interior mutability at every call site, that would only close one of many channels through which "something else ran" can perturb`g()`'s result. The`.await` opens *all* of those channels at once.

## The precise statement

When reading`do_something2`, the set of agents that could have mutated state observable to`g()` between`f()` and`g()` is:

- other OS threads (only if the program has them and they share state with this one),
- the OS / hardware / external world (always, but usually only relevant if`g()` consults them).

When reading`do_something`, that set additionally includes:

- **every other task** the executor is currently driving, executing arbitrary user code in your own program, with access to whatever your runtime's shared state allows.

That last bullet is the one that matters, because it's *your own code* doing *anything*, not just incidental kernel state. It's qualitatively different.

## This is exactly the "function coloring" complaint, stated precisely

What you've articulated is the strongest version of the standard critique of`async`/`await`: every`.await` is a **scheduling point**, and a scheduling point is a place where the rest of your program can run. Reasoning about a function with`.await` in it requires reasoning about the rest of the program at every`.await`. Reasoning about a synchronous function on a single thread requires reasoning only about the function.

People sometimes defend`await` by saying "at least the yield points are visible, unlike threads." That's true and it's a real benefit *compared to threads*. But it's not a defense against your point, which is that visible yield points are still yield points, and a function with yield points is harder to reason about than one without.

## Where this lands for your server

This is, I think, the actual core of your aesthetic preference, and it's a defensible engineering position:

- **Threads**: yield points everywhere, invisible. Worst for local reasoning.
- **`async`/`await`**: yield points at`.await`, visible. Better than threads, worse than fully synchronous.
- **Stackful coroutines (`may`)**: yield points at I/O calls, invisible in the source. Same reasoning burden as`async` in principle, but without syntactic warning. *Worse* than`async` by the criterion you just articulated, not better.
- **Single-threaded synchronous + explicit event loop (`mio`)**: yield points only at the top of the event loop, between handling one event and the next. Inside an event handler,`do_something2`-style reasoning holds completely. This is the model where your preference is fully honored.

Given the principle you've now stated — "I want to read a function and not have to think about what else ran" — the`mio` event-loop model is actually the one that matches, and`may` does *not*, despite my earlier suggestion. I should correct that. The`may` model gives you syntactically synchronous-looking code, but it has exactly the property you object to: arbitrary other coroutines can run between any two lines that touch I/O, and the source doesn't tell you which lines those are. That's worse on your criterion, not better.

The honest hierarchy by your criterion is:

1. **`mio` + per-connection state machines**: matches your preference. Cost: you write the state machines by hand.
2. **`async`/`await` with single-threaded runtime**: matches it partially. Yield points are at least marked.
3. **`may` (stackful coroutines)**: does *not* match it. Yield points are unmarked.
4. **Threads**: does not match it, and adds data races.

So if the reasoning property you described is what you actually care about — and it's a perfectly reasonable thing to care about — the answer for your server is`mio`, accepting the cost of hand-written state machines, *or* a single-threaded`tokio` runtime where you treat every`.await` as the warning sign it is. Not`may`.

You're not wrong. You've put your finger on the thing.

---

## User
##### 5/2/2026, 9:11:42 AM

Show me a function built on Mio that reads a 4-byte unsigned int from a socket (i.e. probably a TCP stream), then reads that many bytes from the stream, and then calls `handle_message()` passing the buffer containing that many bytes.

---

## Assistant
##### 5/2/2026, 9:11:42 AM

## The function, plus the minimum context needed for it to make sense

In the`mio` model, the readable event fires; you call a function that reads as much as the socket has *right now*, advances a per-connection state, and either calls`handle_message` or returns "not done yet, wait for the next readable event." The state must live in the connection, not on the stack, because reads can split across many events.

```rust
use mio::net::TcpStream;
use std::io::{self, Read};

const MAX_MSG_LEN: usize = 16 << 20;

pub struct ReadBuf {
    bytes: Box<[u8]>,   // 4-byte length prefix + body, allocated once
    filled: usize,
}

impl ReadBuf {
    pub fn new() -> Self {
        Self {
            bytes: vec![0u8; 4 + MAX_MSG_LEN].into_boxed_slice(),
            filled: 0,
        }
    }
}

/// Drive one connection on a readable event.
/// Ok(true)  = a complete message was delivered to handle_message().
/// Ok(false) = socket would block; resume on the next readable event.
/// Err(_)    = peer closed mid-message, oversized length, or I/O error.
pub fn pump(stream: &mut TcpStream, rb: &mut ReadBuf) -> io::Result<bool> {
    if !read_until(stream, &mut rb.bytes, &mut rb.filled, 4)? {
        return Ok(false);
    }
    let len = u32::from_be_bytes(rb.bytes[..4].try_into().unwrap()) as usize;
    if len > MAX_MSG_LEN {
        return Err(io::Error::new(io::ErrorKind::InvalidData, "oversized"));
    }
    let total = 4 + len;
    if !read_until(stream, &mut rb.bytes, &mut rb.filled, total)? {
        return Ok(false);
    }
    handle_message(&rb.bytes[4..total]);
    rb.filled = 0;
    Ok(true)
}

fn read_until(
    stream: &mut TcpStream,
    buf: &mut [u8],
    filled: &mut usize,
    target: usize,
) -> io::Result<bool> {
    while *filled < target {
        match stream.read(&mut buf[*filled..target]) {
            Ok(0) => return Err(io::ErrorKind::UnexpectedEof.into()),
            Ok(n) => *filled += n,
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => return Ok(false),
            Err(e) if e.kind() == io::ErrorKind::Interrupted => continue,
            Err(e) => return Err(e),
        }
    }
    Ok(true)
}
```

## Why it has the shape it does

- **The state lives in`ReadBuf`, not on the stack.** A non-blocking`read` can return after delivering 1 byte of the 4-byte length, or 100 bytes of a 1000-byte body. The next readable event may arrive milliseconds later, after`pump` has long since returned. The`filled` cursor is what makes that resumable. This is the hand-written equivalent of the state machine`async`/`await` would generate for you.

- **Single pre-allocated buffer.** No allocation per message. Length prefix and body occupy one contiguous`Box<[u8]>`, sized for the worst-case message. Reuse is just`rb.filled = 0`.

- **`read_until` loops until either the target is reached or the kernel says`WouldBlock`.** That's the right behaviour with`mio`: drain everything available on this wakeup; only return to the event loop when the kernel actually has nothing more for you. (If you registered with edge-triggered mode, this loop is *required* — you won't get another readiness event until the socket transitions empty→nonempty again. With level-triggered you'd get re-notified anyway, but draining is still the lower-syscall-count strategy.)

-**`Interrupted` retries,`WouldBlock` yields, EOF errors, oversized length errors.**These four cases are the entire taxonomy of`read` outcomes you have to handle for a length-prefixed protocol; nothing else is real, nothing else is missing.

- **Length validation, not assertion.** A malicious or buggy peer absolutely *will* send a 4 GiB length someday. That's an expected adversarial case, not an "impossible" one, so it's a returned error, not a panic.

## How it plugs into the event loop

```rust
loop {
    poll.poll(&mut events, None)?;
    for event in &events {
        let conn = connections.get_mut(&event.token()).unwrap();
        if event.is_readable() {
            match pump(&mut conn.stream, &mut conn.rb) {
                Ok(true)  => { /* message delivered; loop to read another */ }
                Ok(false) => { /* wait for next readable event */ }
                Err(_)    => { connections.remove(&event.token()); }
            }
        }
    }
}
```

Inside`handle_message`, every line runs in order, on this thread, with no scheduling point anywhere — exactly the property you said you wanted. The only place the executor can interleave another connection's work is between iterations of the outer`poll` loop, which is one obvious, syntactically-marked place in the entire program.

---

## User
##### 5/2/2026, 10:42:09 AM

Okay, now show the same functionality (and the same amount of minimum context needed for it to make sense) using tokio and `.await`.

---

## Assistant
##### 5/2/2026, 10:42:09 AM

## The function, plus the minimum context needed for it to make sense

In the Tokio model, the runtime resumes your function when bytes are available; you ask for exactly the bytes you need and the runtime suspends you until they arrive. The per-connection state lives on the stack — across`.await` points it is automatically captured into the compiler-generated state machine, so you don't write a`ReadBuf` struct.

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::net::TcpStream;

const MAX_MSG_LEN: usize = 16 << 20;

pub async fn pump(stream: &mut TcpStream) -> io::Result<()> {
    let mut buf = vec![0u8; MAX_MSG_LEN];
    loop {
        let len = stream.read_u32().await? as usize;
        if len > MAX_MSG_LEN {
            return Err(io::Error::new(io::ErrorKind::InvalidData, "oversized"));
        }
        stream.read_exact(&mut buf[..len]).await?;
        handle_message(&buf[..len]);
    }
}
```

## Why it has the shape it does

-**`read_u32().await`** reads exactly 4 bytes, big-endian, and yields back to the runtime as many times as needed if those 4 bytes arrive in pieces.`read_exact(&mut buf[..len]).await` does the same for the body. Partial reads, EOF-mid-message, and`WouldBlock` are all collapsed into the await: either you get all the bytes you asked for, or you get an error.

- **No explicit state struct.** The`buf` and`len` locals straddle`.await` points, so the compiler lifts them into the generated future. That future *is* the state machine`mio` made you write by hand.

- **One allocation per connection, reused for every message.** Same property as the`mio` version, achieved by hoisting`buf` outside the loop.

- **No event-loop plumbing visible.** The Tokio runtime owns the`epoll`/`kqueue`/`io_uring` reactor and the task scheduler; you don't register sockets, you don't match on tokens, you don't dispatch events. The cost is exactly the cost you objected to: every`.await` is a scheduling point at which arbitrary other tasks may run.

## How it plugs into the runtime

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() -> io::Result<()> {
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    loop {
        let (mut stream, _) = listener.accept().await?;
        tokio::spawn(async move {
            let _ = pump(&mut stream).await;
        });
    }
}
```
`flavor = "current_thread"` keeps everything on one OS thread, which is what you asked for.`tokio::spawn` puts each connection's future on the single-threaded scheduler; the runtime drives them all by polling whichever one's I/O just became ready.

## The side-by-side that's worth seeing

| concern |`mio` version |`tokio` version |
|---|---|---|
| where per-connection state lives | hand-written`ReadBuf` struct | compiler-generated future, locals across`.await` |
| how partial reads are handled | explicit`filled` cursor,`WouldBlock` returns`Ok(false)` | implicit;`read_exact` loops internally and yields |
| where scheduling points are | top of`poll` loop only | every`.await` |
| lines of user code shown above | ~40 | ~10 |
| what you reason about per line | only this connection | this connection + every other task |

The trade is exactly the one you've been articulating. The Tokio version is shorter because the compiler is writing the state machine; the`mio` version is longer because you are. The Tokio version asks you to accept that arbitrary other code runs at every`.await`; the`mio` version asks you to accept that you maintain the resumption state by hand. Same functionality, opposite ends of the explicit/implicit axis.

---

## User
##### 5/2/2026, 10:49:10 AM

Okay, now show what the code would look like, in the fully synchronous paradigm (no calls to `.await`, only one event loop, only one thread in the entire process), *assuming* an abstraction layer that provided "just give me the bytes". For example, define a trait named `IStream` that has a method called `handle_bytes`, and create a struct implementing that trait that reads the first 4 bytes and uses them as the number of bytes to come, and reads that many bytes. And show, again *assuming* some kind of nice abstraction layer that provides this functionality, a call like `register_istream()` which connects that struct (instance) to a given socket or TCP connection.

---

## Assistant
##### 5/2/2026, 10:49:10 AM

## The function, plus the minimum context needed for it to make sense

```rust
pub trait IStream {
    fn handle_bytes(&mut self, r: &mut dyn ByteReader);
}

pub trait ByteReader {
    /// Returns exactly `n` bytes. The returned slice is invalidated by
    /// the next call on this reader.
    fn read_exact(&mut self, n: usize) -> &[u8];
}

pub struct LengthPrefixed;

impl IStream for LengthPrefixed {
    fn handle_bytes(&mut self, r: &mut dyn ByteReader) {
        loop {
            let len = u32::from_be_bytes(
                r.read_exact(4).try_into().unwrap()
            ) as usize;
            handle_message(r.read_exact(len));
        }
    }
}

fn main() {
    let listener = std::net::TcpListener::bind("0.0.0.0:8080").unwrap();
    loop {
        let (stream, _) = listener.accept().unwrap();
        register_istream(stream, LengthPrefixed);
    }
}
```

## What this looks like compared to the previous two

| concern |`mio` |`tokio`| this |
|---|---|---|---|
| where per-connection state lives |`ReadBuf` struct | compiler-generated future | abstraction-layer-generated, hidden |
| how partial reads are handled | explicit`filled` cursor |`.await` inside`read_exact` | hidden inside`read_exact` |
| where scheduling points appear in user code | top of`poll` loop | every`.await`| nowhere visible |
| body of message loop | event-driven, returns to event loop mid-message | straight-line with`.await` markers | straight-line, unmarked |

The`handle_bytes` body is what fully synchronous code looks like when "give me N bytes" is a primitive. No cursor, no resumption state, no`.await`, no`WouldBlock`. The state lives in`handle_bytes`'s own stack frame and locals:`len` is a local, the second`read_exact` slice is a local, the loop counter is the program counter.

## What the abstraction layer would have to be doing

This is the part worth being explicit about, given the thread of the conversation. For`read_exact(n)` to "just give you the bytes" while only one OS thread exists and only one event loop runs, the abstraction must do exactly one of:

1. **Stackful coroutines.**`register_istream` allocates a separate stack for the connection;`read_exact` does a non-blocking read, and if the bytes aren't all there, it swaps stacks back to the event loop. When the socket becomes readable again, the event loop swaps back to the coroutine's stack and`read_exact` returns. This is`may`. The code above is essentially what a`may` server's per-connection handler looks like.

2. **Source-level code transformation that generates the state machine for you, without the syntactic marker.** A macro or compiler pass that takes`handle_bytes`'s body and produces the same future Tokio's compiler produces — minus the`.await` keywords. No stable Rust mechanism does this; you'd be writing a custom proc macro or a fork of the compiler.

3. **One OS thread per connection blocking on real`read`.** This makes`register_istream` spawn a thread, which violates "only one thread in the entire process."

Of these, only (1) is real on stable Rust today, and it's the model whose properties you previously identified as problematic: the lines

```rust
let len = u32::from_be_bytes(r.read_exact(4).try_into().unwrap()) as usize;
handle_message(r.read_exact(len));
```

each contain a hidden yield point. Between computing`len` and calling`read_exact(len)`, every other connection's`handle_bytes` can run arbitrary code. The source doesn't tell you that. By the criterion you articulated — "I want to read a function and not have to think about what else ran" — this code is *worse* than the Tokio version, not better, because the Tokio version at least marks its yield points with`.await`.

So the surface syntax you're asking to see is achievable, and it's the cleanest of the three. The cost is the one you already named: the yield points become invisible. That's the irreducible trade between the three models — explicit state machine (`mio`), explicit yield syntax (`tokio`), invisible yields (this /`may`). Pick which cost you'd rather pay; there isn't a fourth option that escapes all three.

---

## User
##### 5/2/2026, 10:53:55 AM

Okay, that's dumb. The problem here is that `read_exact` is defined to block until 4 bytes are available. This violates the rules. Go back and try again. It is possible to define a nice abstraction layer with 1. a single thread per process, 2. a single event loop in that thread, 3. no calls to `.await`, and 4. no functions that block. The result is that every function is synchronous, which means a reader reading the source code can know that there are *no other effects* that can change the state of the process other than the code in this function he is looking at (and transitive effects *caused* by this code). (And signals. Oops. We'll get back to signals.) It is also possible to make an ergonomic framework around this rule which facilitates abstraction and composition. Do it. :-)

---

## Assistant
##### 5/2/2026, 10:53:55 AM

## The shape of the abstraction

The rule "no function blocks, no`.await`, single thread, single event loop" forces this: a handler can't ask for bytes it doesn't have yet. It can only be *handed* the bytes the kernel just delivered, do bounded work on them, and return. State that needs to survive across kernel reads lives in the handler struct, not on the stack across a yield.

So`IStream` takes "here's a chunk that just arrived" and returns. The framework owns a per-connection buffer, accumulates bytes into it, and calls the handler whenever new bytes land. The handler decides whether enough has accumulated to do something.

The ergonomic move is to give the handler a *cursor* over the accumulated buffer with a`try_read_exact(n) -> Option<&[u8]>` method — non-blocking, returns`None` if fewer than`n` bytes are buffered. The handler is a small state machine that advances when it can and returns when it can't. The framework retains the unconsumed bytes for next time.

## The function

```rust
pub trait IStream {
    /// Called when new bytes have arrived. Consume as much as you can,
    /// leave the rest in the cursor; the framework retains it for the
    /// next call.
    fn on_bytes(&mut self, c: &mut Cursor);
}

pub struct Cursor<'a> {
    buf: &'a [u8],
    pos: usize,
}

impl<'a> Cursor<'a> {
    pub fn try_read_exact(&mut self, n: usize) -> Option<&'a [u8]> {
        if self.buf.len() - self.pos < n { return None; }
        let s = &self.buf[self.pos..self.pos + n];
        self.pos += n;
        Some(s)
    }
    pub fn consumed(&self) -> usize { self.pos }
}

enum LpState { Len, Body(usize) }

pub struct LengthPrefixed { state: LpState }

impl LengthPrefixed {
    pub fn new() -> Self { Self { state: LpState::Len } }
}

impl IStream for LengthPrefixed {
    fn on_bytes(&mut self, c: &mut Cursor) {
        loop {
            match self.state {
                LpState::Len => {
                    let Some(b) = c.try_read_exact(4) else { return };
                    let n = u32::from_be_bytes(b.try_into().unwrap()) as usize;
                    self.state = LpState::Body(n);
                }
                LpState::Body(n) => {
                    let Some(b) = c.try_read_exact(n) else { return };
                    handle_message(b);
                    self.state = LpState::Len;
                }
            }
        }
    }
}

fn main() {
    let mut rt = Runtime::new();
    let listener = std::net::TcpListener::bind("0.0.0.0:8080").unwrap();
    rt.register_listener(listener, |stream| {
        Box::new(LengthPrefixed::new()) as Box<dyn IStream>
    });
    rt.run();
}
```
`register_listener` takes a factory that, given an accepted connection, returns the`IStream` to drive it. The framework allocates one buffer per connection, registers the fd with`epoll`, and on every readable event: reads available bytes into the buffer, builds a`Cursor` over it, calls`on_bytes`, then compacts the buffer by dropping the`consumed()` prefix.

## Why this satisfies the rule

Read`on_bytes` top to bottom. There is no`.await`, no hidden coroutine swap, no blocking call.`try_read_exact` either returns a slice from a buffer the framework already filled, or returns`None`. The function runs to completion and returns. Between two calls to`on_bytes`, of course other things happen — other connections' handlers run, the kernel does I/O, timers fire — but *within* a single call, nothing else in the process executes. That's exactly the local-reasoning property you wanted: while this function is on the stack, this function is the only code running. Same property as a synchronous function in a single-threaded program with no signal handlers.

The state-machine boilerplate (`LpState::Len` /`LpState::Body`) is the price. It is the *same* state machine the Tokio compiler generates for you and the *same* one you'd hand-write in`mio`; the difference is that here it lives in named enum variants you can read, and the framework's job is reduced to "fill a buffer, hand it to you, compact it."

## Composition and ergonomics

The reason this is a real framework and not a one-off is that`IStream` composes. A few patterns drop out:

**Framing layered on framing.**`LengthPrefixed` could, instead of calling a free`handle_message`, own an inner`Box<dyn IStream>` and synthesize`Cursor`s over each completed frame to feed it. That gives you "length-prefixed envelopes carrying some inner protocol" as a one-liner of composition.

**Codec adapters.** A`JsonLines``IStream` scans for`\n`, hands complete UTF-8 lines to a`serde::Deserialize` handler. A`Tls``IStream` wraps another`IStream`, decrypting bytes from the cursor and feeding plaintext into the inner one via a small internal`Cursor`. None of these adapters block or await.

**Outbound side.** Symmetric trait`OStream` with`on_writable(&mut self, w: &mut WriteCursor)` where`WriteCursor::try_write(&[u8]) -> usize` returns how many bytes the kernel accepted. Framework calls it whenever the fd is writable. Handlers that have nothing to send return immediately; handlers with a queued message write what fits and keep the rest in their own state for next time.

**Timers.**`rt.register_timer(deadline, Box<dyn FnOnce(&mut Runtime)>)`. Fires from the same event loop, same single thread, same non-blocking discipline. The callback runs to completion, no yields inside.

**Cross-handler messaging.**`rt.post(target_id, msg)` enqueues a message; the framework delivers it by calling a`on_message` method on the target handler in a future loop iteration. Because delivery happens between event-loop iterations, never mid-handler, the local-reasoning property is preserved across handlers too.

The whole framework is one`epoll` loop, one buffer per connection, a slab of handlers, and a queue of pending posted messages. Every handler method is a synchronous function with the property you asked for: while it runs, nothing else in the process runs.

## The signals footnote you flagged

A signal handler can interrupt any instruction and run arbitrary code on the same thread. That breaks the "nothing else runs while this function runs" property at the hardware level. The standard fix in this style of framework: install signal handlers that do nothing but`write()` one byte to a self-pipe (or use`signalfd` /`kqueue`'s`EVFILT_SIGNAL`). The event loop sees the pipe become readable and dispatches a normal, synchronous`on_signal` handler at a loop boundary. Application code never runs in async-signal context; the only code that does is the trivial pipe-write, which is async-signal-safe by construction. With that in place, the local-reasoning rule holds for every handler you write.

This is, incidentally, the model`libuv` (Node.js's core),`libevent`, classic`nginx` modules, and Twisted/asyncio-without-await all use. It's a well-trodden path. The thing that's *not* on this path is a way to write`read_exact(n)` as a call that returns`n` bytes — that's the part that genuinely requires either coroutines,`await`, or threads. Once you accept that handlers consume what's available and stash state explicitly, everything else falls into place and stays synchronous.

---

## User
##### 5/2/2026, 10:57:05 AM

Wait, this still sucks, and the reason it sucks is that you're trying again to have the code that wants access to the bytes be *above* (before) the code that currently has the bytes on the call stack. Why not just pass the bytes as one of the arguments to `on_bytes`? That would be simpler to reason about and less code.

---

## Assistant
##### 5/2/2026, 10:57:05 AM

You're right. The buffer-and-cursor scheme is the framework re-implementing what the kernel already did: it handed you bytes, you stashed them, you handed back a view into the stash. Cut out the middleman. The kernel's`read` already produces "here are some bytes that just arrived" — pass that slice straight to the handler.

```rust
pub trait IStream {
    fn on_bytes(&mut self, bytes: &[u8]);
}
```

The handler is now a pure byte-stream consumer. It gets called with whatever just arrived — 1 byte, 4000 bytes, whatever`read` returned — and it updates its own state. The framework's job collapses to: read into a scratch buffer, call`on_bytes(&scratch[..n])`, done. No per-connection buffer owned by the framework, no cursor type, no compaction.

The length-prefix parser becomes a state machine over its own fields:

```rust
pub struct LengthPrefixed {
    len_buf: [u8; 4],
    len_filled: u8,
    body: Vec<u8>,
    body_needed: usize, // 0 means "still reading length"
}

impl LengthPrefixed {
    pub fn new() -> Self {
        Self { len_buf: [0; 4], len_filled: 0, body: Vec::new(), body_needed: 0 }
    }
}

impl IStream for LengthPrefixed {
    fn on_bytes(&mut self, mut bytes: &[u8]) {
        while !bytes.is_empty() {
            if self.body_needed == 0 {
                let take = (4 - self.len_filled as usize).min(bytes.len());
                self.len_buf[self.len_filled as usize..][..take]
                    .copy_from_slice(&bytes[..take]);
                self.len_filled += take as u8;
                bytes = &bytes[take..];
                if self.len_filled == 4 {
                    self.body_needed =
                        u32::from_be_bytes(self.len_buf) as usize;
                    self.len_filled = 0;
                    self.body.clear();
                    self.body.reserve(self.body_needed);
                }
            } else {
                let take = (self.body_needed - self.body.len()).min(bytes.len());
                self.body.extend_from_slice(&bytes[..take]);
                bytes = &bytes[take..];
                if self.body.len() == self.body_needed {
                    handle_message(&self.body);
                    self.body_needed = 0;
                }
            }
        }
    }
}

fn main() {
    let mut rt = Runtime::new();
    let listener = std::net::TcpListener::bind("0.0.0.0:8080").unwrap();
    rt.register_listener(listener, |_stream| {
        Box::new(LengthPrefixed::new()) as Box<dyn IStream>
    });
    rt.run();
}
```

The framework side is now genuinely small:`epoll_wait`, for each readable fd`read` into a stack-local`[u8; N]`, call`on_bytes` with the resulting slice, repeat. No buffer ownership, no cursors, no compaction. The bytes flow through; nobody stores them except the handler that actually needs to remember them across calls (and it stores exactly what it needs — 4 length bytes plus the body it's accumulating, nothing more).

This is also more honest about cost. The previous version copied bytes from kernel → framework buffer → cursor view → handler's own storage. This version copies kernel → handler's own storage, once, and only the bytes the handler actually needs to retain. A handler that doesn't need to retain anything (a stream cipher, a hash, a passthrough) can process`bytes` in place and store nothing.

The property you care about is preserved: read`on_bytes` top to bottom, no yield points, nothing else in the process runs while it's executing. And the framework no longer pretends to offer "give me N bytes" — it offers "here are the bytes," which is what the kernel actually offers, which is the truth.

---

## User
##### 5/2/2026, 11:00:40 AM

Great! Now assuming we have some variant of the code above, show me the code in the same style, matching the same safety constraints (no yield points except for the one inside the singular event loop — all other functions are purely synchronous and cannot block nor yield), in which there is a trait to represent a given socket/TCP-stream, and it parses incoming bytes starting by the first 4 bytes being the length (by reusing the existing `LengthPrefixed`), and then it reads the *next* four bytes and treats them as a "type" field, and then branches and invokes one function if the "type" field is 0x00000000, another if it is 0x00000001, and a third if it is 0x00000002.

---

## Assistant
##### 5/2/2026, 11:00:40 AM

## The shape

`LengthPrefixed` already does the right thing: it accumulates bytes until a complete frame is in hand, then delivers it as one slice. Reuse it by giving it a *callback* for completed frames — making it a generic adapter rather than a thing hardwired to a free`handle_message`. The outer protocol is then an`IStream` that owns a`LengthPrefixed` configured to call back into the outer protocol's "I have a full frame" method, which switches on the type field.

The tidy way to do that without any indirection at runtime is to make`LengthPrefixed` generic over the frame handler:

```rust
pub trait FrameHandler {
    fn on_frame(&mut self, frame: &[u8]);
}

pub struct LengthPrefixed<H: FrameHandler> {
    len_buf: [u8; 4],
    len_filled: u8,
    body: Vec<u8>,
    body_needed: usize, // 0 means "still reading length"
    handler: H,
}

impl<H: FrameHandler> LengthPrefixed<H> {
    pub fn new(handler: H) -> Self {
        Self {
            len_buf: [0; 4],
            len_filled: 0,
            body: Vec::new(),
            body_needed: 0,
            handler,
        }
    }
}

impl<H: FrameHandler> IStream for LengthPrefixed<H> {
    fn on_bytes(&mut self, mut bytes: &[u8]) {
        while !bytes.is_empty() {
            if self.body_needed == 0 {
                let take = (4 - self.len_filled as usize).min(bytes.len());
                self.len_buf[self.len_filled as usize..][..take]
                    .copy_from_slice(&bytes[..take]);
                self.len_filled += take as u8;
                bytes = &bytes[take..];
                if self.len_filled == 4 {
                    self.body_needed =
                        u32::from_be_bytes(self.len_buf) as usize;
                    self.len_filled = 0;
                    self.body.clear();
                    self.body.reserve(self.body_needed);
                }
            } else {
                let take = (self.body_needed - self.body.len()).min(bytes.len());
                self.body.extend_from_slice(&bytes[..take]);
                bytes = &bytes[take..];
                if self.body.len() == self.body_needed {
                    self.handler.on_frame(&self.body);
                    self.body_needed = 0;
                }
            }
        }
    }
}
```

That's the same`LengthPrefixed` you had, with`handle_message(&self.body)` replaced by`self.handler.on_frame(&self.body)`. One line of real change.

## The typed protocol on top

```rust
pub struct TypedProtocol;

impl FrameHandler for TypedProtocol {
    fn on_frame(&mut self, frame: &[u8]) {
        assert!(frame.len() >= 4);
        let ty = u32::from_be_bytes(frame[..4].try_into().unwrap());
        let payload = &frame[4..];
        match ty {
            0x00000000 => handle_type_0(payload),
            0x00000001 => handle_type_1(payload),
            0x00000002 => handle_type_2(payload),
            _ => panic!("unknown type field {ty:#010x}"),
        }
    }
}

pub type Connection = LengthPrefixed<TypedProtocol>;

pub fn new_connection() -> Connection {
    LengthPrefixed::new(TypedProtocol)
}
```
`Connection: IStream` — it's what you register with the framework. Bytes arrive, flow through`LengthPrefixed`'s framing logic, and complete frames pop out into`TypedProtocol::on_frame`, which peels the 4-byte type and dispatches.

```rust
fn main() {
    let mut rt = Runtime::new();
    let listener = std::net::TcpListener::bind("0.0.0.0:8080").unwrap();
    rt.register_listener(listener, |_stream| {
        Box::new(new_connection()) as Box<dyn IStream>
    });
    rt.run();
}
```

## What this preserves and why

Every function above —`on_bytes`,`on_frame`,`handle_type_0/1/2` — is synchronous, runs to completion, contains no`.await`, makes no blocking call. The only yield point in the whole program is`epoll_wait` at the top of`Runtime::run`. Reading any one of these functions top to bottom, you can be certain that nothing else in the process executes while it's on the stack.

The composition stays honest about state.`LengthPrefixed<H>` retains exactly what it must: 4 length bytes plus the partially-accumulated body.`TypedProtocol` retains nothing — it's a unit struct, because peeling a type field and dispatching needs no state across calls. If a future`handle_type_1` needs to remember things between messages, that state goes in`TypedProtocol`, where it belongs, not in the framing layer.

The two rejections of bad alternatives worth being explicit about: there is no second buffer (the framing layer's`body` is the only accumulator;`TypedProtocol` borrows from it), and there is no dynamic dispatch on the hot path (`LengthPrefixed<TypedProtocol>` monomorphises; the`Box<dyn IStream>` at the framework boundary is the one virtual call per readable event, which is the right place for it). The composition is generic-over-handler all the way down to where the framework needs to hold a heterogeneous collection of connections, and only there does it become dynamic.

The validation choices match the protocol's threat model. The`assert!(frame.len() >= 4)` is a real assertion: if framing delivered a sub-4-byte frame, the framing layer is broken — that's a bug, not an adversarial input, so panic. The unknown-type`panic!` is the question to ask the protocol designer: is an unknown type a peer bug (panic), an expected forward-compat case (ignore), or a malicious input (close connection with an error)? Pick one deliberately; the placeholder panic at least makes the choice visible rather than silently dropping.

---

## User
##### 5/2/2026, 11:07:51 AM

Great! Now let's go back to just the length-prefixed-parsing part, and compare the *testability* of the fully-synchronous-version and the Tokio+await version. Show me unit tests for each version, which test what happens when: a. zero bytes are received and then the socket/connection is closed, b. >0 and < 4 bytes are received and then the socket/connection is closed, c. 4 bytes are received and then the socket connection is closed, 5. > 4 but < 4+length bytes are received and then the socket connection is closed, 6. 4+length bytes are received and then the socket connection is left open, 7. 4+length bytes are received and then the socket connection is closed, 8. 4+length+1 bytes are received and then the socket connection is left open, and 9. 4+length+1 bytes are received and then the socket connection is closed. So to be clear, I want to see two separate code snippets. The first is the unit tests to test all 9 of those cases against the mio+fully-synchronous+`LengthPrefixed` implementation, and the second is the unit tests to test all 9 of those cases against the Tokio+`.await` implementation.

---

## Assistant
##### 5/2/2026, 11:07:51 AM

## The synchronous version

The handler is a pure function from "bytes that arrived" to "messages delivered + state mutated." Testing it requires no I/O, no runtime, no mocks. Feed slices to`on_bytes`, observe what`handle_message` saw. "Connection closed" is just "we stop calling`on_bytes` and check what state we're in."

The one accommodation needed: the original`LengthPrefixed` called a free`handle_message`. For testability, capture deliveries explicitly. Either parameterise over a`FrameHandler` (as in the previous step), or have the test inject a collector. I'll use the`FrameHandler` form since it's already what we built.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[derive(Default)]
    struct Collect(Vec<Vec<u8>>);
    impl FrameHandler for Collect {
        fn on_frame(&mut self, f: &[u8]) { self.0.push(f.to_vec()); }
    }

    fn feed(chunks: &[&[u8]]) -> (LengthPrefixed<Collect>, Vec<Vec<u8>>) {
        let mut lp = LengthPrefixed::new(Collect::default());
        for c in chunks { lp.on_bytes(c); }
        let frames = std::mem::take(&mut lp.handler.0);
        (lp, frames)
    }

    fn frame(payload: &[u8]) -> Vec<u8> {
        let mut v = (payload.len() as u32).to_be_bytes().to_vec();
        v.extend_from_slice(payload);
        v
    }

    // a. zero bytes, then close.
    #[test]
    fn a_zero_then_close() {
        let (lp, frames) = feed(&[]);
        assert!(frames.is_empty());
        assert_eq!(lp.len_filled, 0);
        assert_eq!(lp.body_needed, 0);
    }

    // b. >0 and <4 bytes, then close. Partial length, no frame delivered.
    #[test]
    fn b_partial_length_then_close() {
        let (lp, frames) = feed(&[&[0, 0, 5]]);
        assert!(frames.is_empty());
        assert_eq!(lp.len_filled, 3);
        assert_eq!(lp.body_needed, 0);
    }

    // c. exactly 4 bytes (full length, zero of body), then close.
    #[test]
    fn c_length_only_then_close() {
        let (lp, frames) = feed(&[&(5u32).to_be_bytes()]);
        assert!(frames.is_empty());
        assert_eq!(lp.len_filled, 0);
        assert_eq!(lp.body_needed, 5);
        assert_eq!(lp.body.len(), 0);
    }

    // d. >4 but <4+len bytes, then close. Partial body.
    #[test]
    fn d_partial_body_then_close() {
        let msg = frame(b"hello");           // 4 + 5 = 9 bytes
        let (lp, frames) = feed(&[&msg[..7]]); // length + 3 of 5 body bytes
        assert!(frames.is_empty());
        assert_eq!(lp.body_needed, 5);
        assert_eq!(&lp.body[..], b"hel");
    }

    // e. exactly 4+len bytes, connection left open. Frame delivered, ready for next.
    #[test]
    fn e_full_frame_open() {
        let msg = frame(b"hello");
        let (lp, frames) = feed(&[&msg]);
        assert_eq!(frames, vec![b"hello".to_vec()]);
        assert_eq!(lp.len_filled, 0);
        assert_eq!(lp.body_needed, 0);
    }

    // f. exactly 4+len bytes, then close. Same observable behaviour as (e) for the parser.
    #[test]
    fn f_full_frame_then_close() {
        let msg = frame(b"hello");
        let (lp, frames) = feed(&[&msg]);
        assert_eq!(frames, vec![b"hello".to_vec()]);
        assert_eq!(lp.body_needed, 0);
    }

    // g. 4+len+1 bytes, connection left open. One frame delivered, 1 byte buffered as start of next length.
    #[test]
    fn g_full_frame_plus_one_open() {
        let mut msg = frame(b"hello");
        msg.push(0xAB);
        let (lp, frames) = feed(&[&msg]);
        assert_eq!(frames, vec![b"hello".to_vec()]);
        assert_eq!(lp.len_filled, 1);
        assert_eq!(lp.len_buf[0], 0xAB);
        assert_eq!(lp.body_needed, 0);
    }

    // h. 4+len+1 bytes, then close.
    #[test]
    fn h_full_frame_plus_one_then_close() {
        let mut msg = frame(b"hello");
        msg.push(0xAB);
        let (lp, frames) = feed(&[&msg]);
        assert_eq!(frames, vec![b"hello".to_vec()]);
        assert_eq!(lp.len_filled, 1);
    }
}
```

Nine tests, no async, no I/O, no scheduler, no timeouts, no mocks. Each one is "construct, feed bytes, assert on observable state." A test runs in microseconds and the assertions reach into the parser's fields because the parser *is* the unit under test and its fields *are* its state. "Connection closed" reduces to "we stopped calling`on_bytes`" — there's nothing to mock, because the parser never knew there was a connection in the first place.

## The Tokio version

The Tokio`pump` was:

```rust
pub async fn pump(stream: &mut TcpStream) -> io::Result<()> {
    let mut buf = vec![0u8; MAX_MSG_LEN];
    loop {
        let len = stream.read_u32().await? as usize;
        if len > MAX_MSG_LEN { return Err(...); }
        stream.read_exact(&mut buf[..len]).await?;
        handle_message(&buf[..len]);
    }
}
```

It's coupled to`TcpStream` and to a free`handle_message`. To test it you need (1) something that can stand in for`TcpStream`, (2) a way to capture deliveries, (3) an async runtime. The standard moves: generalise`pump` over`AsyncRead`, inject a delivery callback (or capture via a channel),
and drive it with`tokio::test`. Use`tokio::io::duplex` to get an in-memory pair where one end is fed by the test and the other is what`pump` reads.

For the test to be deterministic, the test must drop its write half to signal "connection closed" — that's what makes`read_exact` /`read_u32` return EOF. And because`pump` is a loop that only returns on error or EOF, the test has to`await` the join handle to see the final result; "connection left open" cases require a different shape — spawn`pump`, send the bytes, give it a moment to deliver, then assert without awaiting termination.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tokio::io::{AsyncWriteExt, AsyncRead};
    use tokio::sync::mpsc;

    // Test-friendly variant: generic over reader, deliveries via channel.
    async fn pump_into<R: AsyncRead + Unpin>(
        mut r: R,
        tx: mpsc::UnboundedSender<Vec<u8>>,
    ) -> std::io::Result<()> {
        use tokio::io::AsyncReadExt;
        let mut buf = vec![0u8; 1 << 20];
        loop {
            let len = r.read_u32().await? as usize;
            r.read_exact(&mut buf[..len]).await?;
            let _ = tx.send(buf[..len].to_vec());
        }
    }

    fn frame(payload: &[u8]) -> Vec<u8> {
        let mut v = (payload.len() as u32).to_be_bytes().to_vec();
        v.extend_from_slice(payload);
        v
    }

    /// Run pump against `chunks`, then close the writer. Return delivered
    /// frames and the terminal error kind pump returned.
    async fn run_to_close(chunks: &[&[u8]]) -> (Vec<Vec<u8>>, std::io::ErrorKind) {
        let (client, server) = tokio::io::duplex(64 * 1024);
        let (tx, mut rx) = mpsc::unbounded_channel();
        let task = tokio::spawn(pump_into(server, tx));
        let mut client = client;
        for c in chunks { client.write_all(c).await.unwrap(); }
        drop(client); // close
        let err = task.await.unwrap().unwrap_err().kind();
        let mut out = Vec::new();
        while let Ok(f) = rx.try_recv() { out.push(f); }
        (out, err)
    }

    /// Run pump, send chunks, do NOT close, give it time to deliver, then
    /// abort the task. Returns delivered frames.
    async fn run_open(chunks: &[&[u8]]) -> Vec<Vec<u8>> {
        let (client, server) = tokio::io::duplex(64 * 1024);
        let (tx, mut rx) = mpsc::unbounded_channel();
        let task = tokio::spawn(pump_into(server, tx));
        let mut client = client;
        for c in chunks { client.write_all(c).await.unwrap(); }
        // Yield enough times for pump to drain whatever it can.
        for _ in 0..16 { tokio::task::yield_now().await; }
        task.abort();
        let _ = task.await;
        let mut out = Vec::new();
        while let Ok(f) = rx.try_recv() { out.push(f); }
        // Keep client alive until here so the pipe doesn't EOF.
        drop(client);
        out
    }

    // a. zero bytes, then close.
    #[tokio::test]
    async fn a_zero_then_close() {
        let (frames, kind) = run_to_close(&[]).await;
        assert!(frames.is_empty());
        assert_eq!(kind, std::io::ErrorKind::UnexpectedEof);
    }

    // b. partial length, then close. read_u32 returns UnexpectedEof.
    #[tokio::test]
    async fn b_partial_length_then_close() {
        let (frames, kind) = run_to_close(&[&[0, 0, 5]]).await;
        assert!(frames.is_empty());
        assert_eq!(kind, std::io::ErrorKind::UnexpectedEof);
    }

    // c. length only, then close. read_u32 succeeds, read_exact for body hits EOF.
    #[tokio::test]
    async fn c_length_only_then_close() {
        let (frames, kind) = run_to_close(&[&(5u32).to_be_bytes()]).await;
        assert!(frames.is_empty());
        assert_eq!(kind, std::io::ErrorKind::UnexpectedEof);
    }

    // d. partial body, then close.
    #[tokio::test]
    async fn d_partial_body_then_close() {
        let msg = frame(b"hello");
        let (frames, kind) = run_to_close(&[&msg[..7]]).await;
        assert!(frames.is_empty());
        assert_eq!(kind, std::io::ErrorKind::UnexpectedEof);
    }

    // e. full frame, connection left open. One delivery, pump still running.
    #[tokio::test]
    async fn e_full_frame_open() {
        let msg = frame(b"hello");
        let frames = run_open(&[&msg]).await;
        assert_eq!(frames, vec![b"hello".to_vec()]);
    }

    // f. full frame, then close. One delivery, then EOF on next read_u32.
    #[tokio::test]
    async fn f_full_frame_then_close() {
        let msg = frame(b"hello");
        let (frames, kind) = run_to_close(&[&msg]).await;
        assert_eq!(frames, vec![b"hello".to_vec()]);
        assert_eq!(kind, std::io::ErrorKind::UnexpectedEof);
    }

    // g. full frame + 1 byte, connection left open. One delivery; the extra byte
    // is buffered inside read_u32's partial state.
    #[tokio::test]
    async fn g_full_frame_plus_one_open() {
        let mut msg = frame(b"hello");
        msg.push(0xAB);
        let frames = run_open(&[&msg]).await;
        assert_eq!(frames, vec![b"hello".to_vec()]);
    }

    // h. full frame + 1 byte, then close. Delivery, then partial-length EOF.
    #[tokio::test]
    async fn h_full_frame_plus_one_then_close() {
        let mut msg = frame(b"hello");
        msg.push(0xAB);
        let (frames, kind) = run_to_close(&[&msg]).await;
        assert_eq!(frames, vec![b"hello".to_vec()]);
        assert_eq!(kind, std::io::ErrorKind::UnexpectedEof);
    }
}
```

## What the comparison shows

The synchronous tests test the parser. The Tokio tests test the parser *plus*`tokio::io::duplex` *plus* the runtime *plus* the channel *plus* the scheduler's willingness to actually run the task before the assertions. Concretely:

- **Surface area.** Sync version: feed slices, read fields. Tokio version: spawn a task, write to a duplex pipe, drain a channel, decide whether to abort or await termination, distinguish "open" cases from "closed" cases via two different harness functions.

- **Determinism.** Sync version is fully deterministic — the order of operations is`for c in chunks { lp.on_bytes(c); }`. The Tokio "open" cases need`yield_now().await` in a loop, hoping the pump task has been scheduled enough times to consume the bytes. That hope is well-founded with`current_thread` runtimes and small inputs, but it's hope. Increase the message size or add a real socket and you'll be reaching for timeouts and retries, which is the well-known async-test pathology.

- **Observability of state.** Sync version asserts on`lp.len_filled`,`lp.body_needed`,`lp.body` — the parser's actual state, the thing you want to verify. Tokio version can't assert on the equivalent state at all, because that state lives inside an opaque future on a task you don't have a handle into. You can only observe what crossed the channel and what error the task ended with. "Did pump correctly buffer 3 of the 5 body bytes" is unaskable.

- **What "closed" means.** Sync: stop calling. Tokio: drop the writer half, then`await` the task, then inspect the error kind, and the error kind is`UnexpectedEof` for *every* mid-frame close — (a), (b), (c), (d), (f), (h) all produce the same terminal error, so the test only distinguishes them by what was delivered before, not by how the parser reacted to the close. The sync version distinguishes them by the parser's internal state at the moment of close.

- **What you can't easily test in the Tokio version at all.** "After receiving 3 of 4 length bytes, what's the parser's internal buffer?" — there's no way to ask. "If I send the bytes one at a time with arbitrary delays, does the parser still produce one frame?" — possible, but now you're orchestrating timing. The sync version answers both with`feed(&[&[0], &[0], &[0], &[5], b"hello"])` and an assertion.

- **Failure modes of the tests themselves.** Sync tests can fail for one reason: the parser is wrong. Tokio tests can fail because the parser is wrong, *or* because the yield-loop count was too low, *or* because`duplex`'s buffer interacted with chunking in an unexpected way, *or* because`task.abort()` raced with the final delivery. Every one of those has been a real flake in real codebases.

The asymmetry isn't an artifact of how the tests are written; it's structural. The synchronous parser exposes its state as data, so tests are data-in / data-out. The async parser hides its state inside a future running on a scheduler, so tests have to coax behaviour out through I/O channels and time. Even when both arrive at the same coverage, the sync tests are shorter, faster, deterministic, and more precise about what they're asserting. That's the testability dividend of the constraint you've been defending.

---

## User
##### 5/2/2026, 11:12:58 AM

Okay, now let's add another layer of abstraction and composition! Show me a struct to handle "all my connections" with a method that takes an incoming TCP connection, creates a struct to handle "this connection", and connect that struct to that incoming TCP connection, and adds the "this connection" to struct to an internal collection. Then add a method to the "all my connections" struct to handle the case that "one of my connections sent a message of type 0x00000000". That method increments a counter showing how many total connections have sent a message of that type, and also sets a flag in the "this connection" struct showing that this connection has sent a message of that type.

---

## Assistant
##### 5/2/2026, 11:12:58 AM

## The shape of the composition

Two structs:`Server` owns the collection and the cross-connection counter;`Conn` owns the per-connection state and the protocol stack. The protocol stack inside`Conn` is`LengthPrefixed<TypedProtocol>` from before, and`TypedProtocol` is what dispatches on the type field. The thing to figure out is: when`TypedProtocol` decides "this was type 0", how does it tell *both* the`Server` (increment counter) and the`Conn` (set flag)?

The clean answer in this single-threaded, no-yields style is that dispatch is an upcall: the framing layer hands a complete frame back up to the`Conn`, and the`Conn` — which has access to itself and to the`Server` — does the type switch and the bookkeeping in one place. No callbacks-into-callbacks, no`Rc<RefCell>`, no shared ownership puzzles. Just the call stack, going up.

The framework gives the runtime the ability to call`Server::on_accept(stream) -> ConnId` and`Server::on_bytes(ConnId, &[u8])`. Those two methods are the entire interface between the event loop and the application. Inside them, everything is an ordinary method call on`&mut self`.

```rust
use std::collections::HashMap;

pub type ConnId = u64;

pub struct Server {
    next_id: ConnId,
    conns: HashMap<ConnId, Conn>,
    type0_total: u64,
}

pub struct Conn {
    id: ConnId,
    framer: LengthPrefixed,   // non-generic now; see note below
    sent_type0: bool,
}

impl Server {
    pub fn new() -> Self {
        Self { next_id: 0, conns: HashMap::new(), type0_total: 0 }
    }

    pub fn on_accept(&mut self, _stream: std::net::TcpStream) -> ConnId {
        let id = self.next_id;
        self.next_id += 1;
        self.conns.insert(id, Conn {
            id,
            framer: LengthPrefixed::new(),
            sent_type0: false,
        });
        id
    }

    pub fn on_bytes(&mut self, id: ConnId, bytes: &[u8]) {
        // Take the framer out so we can pass `&mut self` to on_frame
        // without aliasing the framer through the conns map.
        let conn = self.conns.get_mut(&id).unwrap();
        let mut framer = std::mem::replace(
            &mut conn.framer,
            LengthPrefixed::new_empty(),
        );
        framer.on_bytes(bytes, |frame| Conn::on_frame(self, id, frame));
        // Restore.
        self.conns.get_mut(&id).unwrap().framer = framer;
    }

    fn on_type0(&mut self, id: ConnId) {
        self.type0_total += 1;
        self.conns.get_mut(&id).unwrap().sent_type0 = true;
    }

    fn on_type1(&mut self, _id: ConnId, _payload: &[u8]) { /* ... */ }
    fn on_type2(&mut self, _id: ConnId, _payload: &[u8]) { /* ... */ }
}

impl Conn {
    fn on_frame(server: &mut Server, id: ConnId, frame: &[u8]) {
        assert!(frame.len() >= 4);
        let ty = u32::from_be_bytes(frame[..4].try_into().unwrap());
        let payload = &frame[4..];
        match ty {
            0x00000000 => server.on_type0(id),
            0x00000001 => server.on_type1(id, payload),
            0x00000002 => server.on_type2(id, payload),
            _ => panic!("unknown type {ty:#010x} on conn {id}"),
        }
    }
}
```

And the framer becomes a callback-taking version, which is what makes the upcall work cleanly:

```rust
pub struct LengthPrefixed {
    len_buf: [u8; 4],
    len_filled: u8,
    body: Vec<u8>,
    body_needed: usize,
}

impl LengthPrefixed {
    pub fn new() -> Self {
        Self { len_buf: [0; 4], len_filled: 0, body: Vec::new(), body_needed: 0 }
    }
    pub fn new_empty() -> Self { Self::new() }

    pub fn on_bytes(&mut self, mut bytes: &[u8], mut on_frame: impl FnMut(&[u8])) {
        while !bytes.is_empty() {
            if self.body_needed == 0 {
                let take = (4 - self.len_filled as usize).min(bytes.len());
                self.len_buf[self.len_filled as usize..][..take]
                    .copy_from_slice(&bytes[..take]);
                self.len_filled += take as u8;
                bytes = &bytes[take..];
                if self.len_filled == 4 {
                    self.body_needed =
                        u32::from_be_bytes(self.len_buf) as usize;
                    self.len_filled = 0;
                    self.body.clear();
                    self.body.reserve(self.body_needed);
                }
            } else {
                let take = (self.body_needed - self.body.len()).min(bytes.len());
                self.body.extend_from_slice(&bytes[..take]);
                bytes = &bytes[take..];
                if self.body.len() == self.body_needed {
                    on_frame(&self.body);
                    self.body_needed = 0;
                }
            }
        }
    }
}
```

## The borrow-checker move that matters

The interesting line is`std::mem::replace(&mut conn.framer, LengthPrefixed::new_empty())`. The reason it's there:`on_frame` needs`&mut Server`, but the framer is *inside* the server (via the`HashMap`). If you call`framer.on_bytes(bytes, |f| Conn::on_frame(self, id, f))` directly,`framer` is borrowed from`self`, the closure borrows`self` again — aliasing. Rust says no.

Pulling the framer out, running it against`&mut self`, then putting it back, dissolves the conflict. It's three lines of plumbing and it costs nothing at runtime (`Vec`'s buffer is moved by pointer;`new_empty` allocates nothing). This is the standard Rust idiom for "I need to mutate a field while also passing the parent`&mut`," and it's the reason this composition is ergonomic in the synchronous style: the move is local, explicit, and obviously sound.

The alternative — putting`Conn` behind`Rc<RefCell<...>>` so the framer can call back into it — would also work and is what an OO codebase would reach for first. It's worse here. It moves a check from compile time to runtime, it costs a refcount and a borrow flag per call, and it muddies the local-reasoning property: with`RefCell`, you have to convince yourself that no method on the call stack also holds a borrow. With`mem::replace`, the borrow checker convinces itself for you.

## Why the upcall lives on`Conn`, not in a generic`FrameHandler`

Earlier we had`LengthPrefixed<TypedProtocol>` where`TypedProtocol` was its own type implementing`FrameHandler`. That was clean when the type dispatch was self-contained. It stops being clean once dispatch needs both`&mut Server` and`&mut Conn` —`TypedProtocol` would have to hold references to both, which is exactly the aliasing problem above wearing a different hat.

The right factoring once handlers need access to multiple owners is: framing is a *value* that belongs to`Conn`, and dispatch is a *method* on`Conn` that takes whatever it needs as parameters.`Conn::on_frame` is a free-standing fn (not even a method, to avoid borrowing`self` while`server` is also borrowed) that takes`&mut Server` and a`ConnId` and does the work. The`ConnId` indirection is what lets`on_type0` mutate both the counter and the connection's flag without any reference juggling — it looks the connection up by id when it needs it. That's also why the connection table is keyed by an integer id rather than by holding`Conn` values directly somewhere on the stack.

## What the runtime looks like

```rust
fn main() {
    let mut server = Server::new();
    let mut rt = Runtime::new();
    let listener = std::net::TcpListener::bind("0.0.0.0:8080").unwrap();
    rt.register_listener(listener, |stream, rt| {
        let id = server.on_accept(stream);
        rt.on_readable(id, |bytes| server.on_bytes(id, bytes));
    });
    rt.run();
}
```

Hand-wavy on the runtime API because that's not the part being designed here, but the shape is: the runtime calls into`Server` at exactly two points — accept and bytes-arrived — and`Server` does everything else as ordinary synchronous method calls. No yields below`rt.run()`. Reading`on_type0` top to bottom: increment counter, set flag, return. Nothing else in the process ran.

## What this composes to next

The pattern generalises. If a future message type needs to broadcast to all connections — say, type 3 means "send a notification to everyone" — the method is`Server::on_type3(&mut self, from: ConnId, payload: &[u8])` and it iterates`self.conns.values_mut()`. If a per-connection state machine needs to react to type 0 by also doing something connection-local beyond setting a flag, that logic goes in`on_type0` next to the flag-set, because`&mut self` already gives access to`self.conns.get_mut(&id).unwrap()`. The`Server` is the
single place where cross-cutting state lives; the`Conn` is the single place where per-connection state lives; the framing layer doesn't know either of them exist. Each layer's state is exactly what it needs, and the call stack carries the`&mut` that authorises mutation. That's the composition story this style gives you, and it stays this clean as long as you keep the rule that nothing yields below the event loop.

---

## User
##### 5/2/2026, 11:19:49 AM

Great! Next question: is there a framework/library in Rust land, possibly Mio itself or an add-on that depends upon Mio, or a separate Mio-like thing, that provides abstractions like `LengthPrefixed`, or `Conn`, or other tools for this sort of fully-synchronous server programming? I basically just wanted Twisted Python back, except written in Rust. :-)

---

## Assistant
##### 5/2/2026, 11:19:49 AM

Short answer: not really. There is no maintained Rust framework that gives you the Twisted-shaped thing — a synchronous, single-threaded reactor with a layered protocol/transport abstraction where you write`dataReceived(self, data)` style handlers and compose`Int32StringReceiver` on top of`LineReceiver` on top of`Protocol`. The Rust ecosystem went hard in the async/await direction around 2018–2019, and the libraries that *were* trying to be Twisted-like either got absorbed into Tokio's async model or were abandoned.

Here's what exists and why none of it is quite what you want:

## mio itself

Just the reactor —`Poll`,`Events`,`Token`, the`Evented`/`Source` trait. No protocol abstractions, no connection table, no framing helpers. It's explicitly designed as a low-level primitive that *other* libraries build on, and the mio book says as much: "We have designed mio to serve as a low-level primitive for higher-level abstractions that people can build outside of the core mio repository." [The mio authors' intent was that someone would build the Twisted layer on top — that someone largely didn't, because async/await ate the oxygen.]

## Things that built on mio and are now dead or async

**mioco** (stackful coroutines on mio) — exactly the "write synchronous code, get async I/O" model, like goroutines or gevent. Last meaningful activity ~2017. Abandoned.

**rotor** — was an explicit attempt at a Twisted-style state-machine framework on mio, with a`Machine` trait and composable protocols. Abandoned around 2017 when the author concluded async/await was going to win.

**amy**, **eventual_io**, **coio-rs**, the early **tokio-core** with`Handle` and`Core::run` — all variations on the theme, all either dead or subsumed into modern Tokio.

The pattern is consistent: anyone who tried to build the Twisted-shape abstraction in 2015–2018 either gave up or pivoted to async/await. The survivors are the async/await stacks (Tokio, async-std, smol), which is a different programming model from what you're asking for.

## What's left if you want synchronous reactor code today

**mio directly**, plus you write the`LengthPrefixed`/`Conn`/`Server` layer yourself. This is what you've been doing in this conversation, and honestly it's not that much code — the framing layer is ~40 lines, the connection table is ~30, and the runtime glue is ~50. You write it once per project (or factor it into a personal crate) and you're done. The mio API is stable and well-maintained because Tokio depends on it.

**polling** (the smol project's reactor crate) — slightly higher-level than mio's raw`Poll`, cross-platform abstraction over epoll/kqueue/IOCP/io_uring. Same situation though: it's a reactor primitive, not a protocol framework. You'd build the same layers on top.

**may** — stackful coroutines, the spiritual successor to mioco. This *is* close to what Twisted feels like from the application programmer's perspective: you write synchronous-looking code,`read()` "blocks" the coroutine but not the thread, the scheduler multiplexes. But it's coroutine-based, not reactor-with-callbacks-based, so the programming model is "goroutines" rather than "Twisted protocols." And it has the local-reasoning caveat we discussed earlier — any I/O call is a yield point.

## The honest assessment

The Twisted shape —`Protocol.dataReceived`,`Transport.write`,`Factory.buildProtocol`, layered receivers — maps almost perfectly onto what you've been building by hand:`IStream::on_bytes`,`LengthPrefixed<H: FrameHandler>`,`Server::on_accept`. The mapping is so direct that I suspect the reason no one shipped "Twisted for Rust" is that once you have mio, the remaining layer is small enough that every project rolls its own to fit its exact protocol, and no one's version became canonical.

If you wanted to publish it, there's a real gap in the ecosystem. Something called e.g.`reactor` or`synio` that exposed:

- a`Reactor` owning the`Poll` and a connection table keyed by`ConnId`
- a`Protocol` trait with`on_bytes(&mut self, &mut Transport, &[u8])` and`on_close`
- a`Transport` handle for writing (with the write-buffering and writable-event dance handled internally)
- composable framing:`LengthPrefixed<P: Protocol>`,`LineDelimited<P>`,`Netstring<P>`
- a`Listener` registration API

…would be genuinely useful, would be maybe 800 lines of code, and would give Rust the Twisted-shaped option that's currently missing. The async/await crowd wouldn't use it, but the "I want local reasoning and a single thread" crowd — which is real, especially in trading, embedded-adjacent, and game-server work — would.

So: no, the library you're looking for doesn't exist in maintained form. The closest things are dead (rotor, mioco) or are a different model (may's coroutines, Tokio's async). What you've been writing in this conversation is, more or less, the missing library.

---

## User
##### 5/2/2026, 11:36:16 AM

Okay, now please help me *summarize* this conversation to communicate the critical bits to someone without them reading through our entire back and forth. Here are the parts that I'd like to include: 1. The fully-synchronous model allows easier local reasoning than the `.await` model does, and Rust's type system does not fix this. 2. The fully-synchronous model makes it easier to write thorough and deterministic unit tests. 3. It is pretty straightforward to add abstraction and composition on top of the fully-synchronous model, once you stop trying to make functions that block until they can return what the caller is looking for, and instead pass handlers that *get* called once the thing that the handler is looking for is synchronously available. 4. There is no currently-maintained library or framework to support this model (a la Twisted Python) in Rust, but on the other hand it is little-enough source code that you can write it yourself for your purpose. If someone wrote one that provided some commonly-needed functionality (a la Twisted Python), it would fill a gap in the Rust ecosystem. Please summarize all that, including code examples as necessary to demonstrate the points. I'm especially fond of the gap between the 9-case unit tests for the fully-synchronous version versus the attempt to test the same functionality for the Tokio version, so maybe include that, although maybe move it into a footnote/appendix so as not to disrupt the reader reading your summary.

---

## Assistant
##### 5/2/2026, 11:36:16 AM

## Synchronous reactor programming in Rust: a case for the missing middle

Rust's concurrency story is dominated by two options:`std::thread` for parallelism and`async`/`await` for concurrent I/O. There is a third model that the ecosystem has largely forgotten — the fully synchronous, single-threaded reactor pattern, of the kind Twisted Python popularised — and it has properties that the`async` model does not, properties that matter for code where local reasoning and testability are paramount.

### 1. Local reasoning: the synchronous model wins, and the type system does not save`async`

In a fully synchronous reactor, the application code runs in a single thread, and the only place control returns to the runtime is at the top of the event loop. Inside any handler —`on_bytes`,`on_frame`,`on_type0` — you can read top to bottom and know that nothing else in the process ran between any two lines. The`&mut self` you hold is the only`&mut self` in the universe for the duration of the call, by construction.

In the`async`/`await` model, every`.await` is a yield point. Any task waiting on the same runtime can run there, mutate any state it has access to, drop guards, complete or cancel futures, and return control to you with the world rearranged. Rust's borrow checker does not protect you here, because the data those other tasks mutate is *their* data — they hold their own`&mut` references. The invariant the borrow checker enforces (no aliased mutation at any instant) is preserved; the invariant you actually want when reading code (nothing else changes between these two lines) is not the same invariant, and Rust does not enforce it.

This is not hypothetical. Consider:

```rust
async fn handle(&mut self, msg: Msg) {
    let count = self.shared.read_count().await;  // yield point
    // Another task may have run here, including code that
    // increments the count, removes entries from self.shared,
    // or sends messages that this task assumed couldn't arrive yet.
    self.shared.set_count(count + 1).await;       // yield point
}
```

The compiler accepts this. Two tasks running this concurrently produce a lost update. The fix requires either a mutex held across the yield (which serialises and risks deadlock) or a CAS-style retry loop or a redesign — none of which the type system prompts you to do. In the synchronous model the equivalent code is correct as written, because there is no yield point.

### 2. Testability: deterministic and thorough vs. runtime-dependent and approximate

The synchronous model makes the protocol layer a plain state machine: a struct with methods that take bytes and produce frames. Unit tests construct it, call methods with chosen byte slices, and assert on observable state. There is no runtime, no executor, no time, no scheduling.

Consider a length-prefixed framer that reads a 4-byte big-endian length followed by a body. Here is the synchronous version:

```rust
pub struct LengthPrefixed {
    len_buf: [u8; 4],
    len_filled: u8,
    body: Vec<u8>,
    body_needed: usize,
}

impl LengthPrefixed {
    pub fn on_bytes(&mut self, mut bytes: &[u8],
                    mut on_frame: impl FnMut(&[u8])) {
        while !bytes.is_empty() {
            if self.body_needed == 0 {
                let take = (4 - self.len_filled as usize).min(bytes.len());
                self.len_buf[self.len_filled as usize..][..take]
                    .copy_from_slice(&bytes[..take]);
                self.len_filled += take as u8;
                bytes = &bytes[take..];
                if self.len_filled == 4 {
                    self.body_needed = u32::from_be_bytes(self.len_buf) as usize;
                    self.len_filled = 0;
                    self.body.clear();
                    self.body.reserve(self.body_needed);
                }
            } else {
                let take = (self.body_needed - self.body.len()).min(bytes.len());
                self.body.extend_from_slice(&bytes[..take]);
                bytes = &bytes[take..];
                if self.body.len() == self.body_needed {
                    on_frame(&self.body);
                    self.body_needed = 0;
                }
            }
        }
    }
}
```

A test exercises it like this:

```rust
#[test]
fn length_split_across_three_reads() {
    let mut f = LengthPrefixed::new();
    let mut frames: Vec<Vec<u8>> = Vec::new();
    let mut push = |b: &[u8]| frames.push(b.to_vec());
    f.on_bytes(&[0x00, 0x00], &mut push);
    assert_eq!(f.len_filled, 2);
    f.on_bytes(&[0x00], &mut push);
    assert_eq!(f.len_filled, 3);
    f.on_bytes(&[0x03, b'a', b'b', b'c'], &mut push);
    assert_eq!(frames, vec![b"abc".to_vec()]);
}
```

The Tokio version of the same logic — a`pump` function reading from an`AsyncRead` and dispatching frames — requires for each test case constructing a mock`AsyncRead` that produces bytes in the chunks you want, spawning the pump on a runtime, awaiting completion, and inspecting outputs. The test is longer, depends on the runtime, and cannot directly observe`len_filled` mid-stream because there is no mid-stream — the pump runs to completion or is cancelled, and partial-read state lives in stack locals inside an opaque future. Asserting "after exactly two bytes arrived, the framer is in this specific intermediate state" is straightforward synchronously and awkward-to-impossible asynchronously.

The appendix lists nine specific unit-test scenarios for`LengthPrefixed` and contrasts them with the corresponding Tokio test attempts.

### 3. Composition: pass handlers, do not block

The intuition that synchronous code cannot compose well comes from a habit: writing functions that block until they can return what the caller asked for.`read_frame() -> Frame` cannot exist in a single-threaded non-blocking reactor, because there is no thread to block. The fix is to invert: instead of returning the thing, accept a handler that gets called when the thing is synchronously available.

`LengthPrefixed::on_bytes` takes`impl FnMut(&[u8])` and calls it once per complete frame. The caller — a`Conn` — has its own`on_frame` that does type dispatch. The`Server` owns the connection table and cross-cutting state. The composition looks like this:

```rust
pub struct Server {
    conns: HashMap<ConnId, Conn>,
    type0_total: u64,
}

pub struct Conn {
    framer: LengthPrefixed,
    sent_type0: bool,
}

impl Server {
    pub fn on_bytes(&mut self, id: ConnId, bytes: &[u8]) {
        let conn = self.conns.get_mut(&id).unwrap();
        let mut framer = std::mem::replace(&mut conn.framer,
                                            LengthPrefixed::new());
        framer.on_bytes(bytes, |frame| Conn::on_frame(self, id, frame));
        self.conns.get_mut(&id).unwrap().framer = framer;
    }

    fn on_type0(&mut self, id: ConnId) {
        self.type0_total += 1;
        self.conns.get_mut(&id).unwrap().sent_type0 = true;
    }
}

impl Conn {
    fn on_frame(server: &mut Server, id: ConnId, frame: &[u8]) {
        let ty = u32::from_be_bytes(frame[..4].try_into().unwrap());
        match ty {
            0x00000000 => server.on_type0(id),
            _ => panic!("unknown type"),
        }
    }
}
```

The`mem::replace` is the one piece of plumbing: the framer lives inside the server, so calling`framer.on_bytes(_, |f| use(self))` would alias`self`. Pulling the framer out, running it against`&mut self`, putting it back is local, explicit, costs nothing at runtime, and lets the borrow checker prove soundness rather than punting to`RefCell`. This pattern composes indefinitely: more layers, more message types, broadcasting to all connections — all are ordinary`&mut self` method calls.

The mental shift required is small but real: stop looking for`let frame = read_frame().await;` and start writing`framer.on_bytes(bytes, |frame| ...);`. Once made, the abstraction story is no worse than`async`'s, and the local-reasoning property is preserved because no method below the event loop yields.

### 4. The ecosystem gap

There is no currently-maintained Rust library that provides Twisted-shaped abstractions: a reactor, a`Protocol` trait with`on_bytes`/`on_close`, a`Transport` for writing, composable framers (`LengthPrefixed`,`LineDelimited`,`Netstring`), a connection table. The libraries that tried —`rotor`,`mioco`, early`tokio-core` — were abandoned around 2017–2018 when`async`/`await` consumed the ecosystem's attention.`mio` itself is maintained and stable (Tokio depends on it) but is deliberately just the reactor primitive —`Poll`,`Events`,`Token` — with no protocol layer.`polling` (from the smol project) is similar.`may` exists and provides stackful coroutines, but that is a different programming model (Go-style goroutines) with the same yield-point local-reasoning caveat as`async`.

The pragmatic situation: writing the missing layer for one project takes roughly 200 lines (framer ~40, connection table ~30, runtime glue ~50, plus protocol-specific code). It is feasible to roll your own and many projects effectively have. The cost is that no one's version became canonical, so each project reinvents it.

A published crate — something like`reactor` or`synio` — exposing the Twisted-shaped abstractions on top of`mio` would fill a real gap. It would not appeal to the`async` mainstream, but the constituency that wants single-threaded local reasoning (trading systems, game servers, embedded-adjacent work, anyone porting from Twisted) is real and currently underserved. The work is bounded — perhaps 800 lines of library code plus documentation — and the API surface is well-understood from twenty years of Twisted's existence.

---

## Appendix: nine test scenarios, two models

The following nine scenarios fully specify the behaviour of`LengthPrefixed` for one frame's worth of input. In the synchronous model each is a five-to-ten-line`#[test]` function that constructs the framer, calls`on_bytes` with chosen slices, and asserts on collected frames and on internal state (`len_filled`,`body.len()`,`body_needed`).

| # | Scenario | Synchronous test | Tokio test |
|---|---|---|---|
| 1 | All bytes in one read | trivial | requires mock`AsyncRead`, runtime |
| 2 | Length split 2+2, body whole | direct; assert`len_filled` after first call | cannot observe intermediate state |
| 3 | Length whole, body split 1+rest | direct; assert`body.len()` mid-stream | cannot observe intermediate state |
| 4 | Length split 1+1+1+1 | direct | mock must yield four times |
| 5 | Two frames in one read | direct; assert two frames collected | runtime-dependent |
| 6 | Frame boundary mid-buffer | direct | requires careful mock |
| 7 | Zero-length body | direct; assert frame fires immediately | requires runtime |
| 8 | Bytes after frame remain buffered (next frame's length) | direct; assert state across calls | requires multi-step mock |
| 9 | Empty`on_bytes` call is a no-op | direct | not naturally expressible |

The synchronous tests run in microseconds, are fully deterministic, observe internal state directly, and require no runtime or mock. The Tokio tests require a mock`AsyncRead` whose chunking schedule must be carefully designed for each scenario, depend on the runtime's scheduling for ordering, cannot observe partial-read state because that state lives in stack locals inside the pump's future, and add roughly 15–30 lines of harness per case. Scenarios 2, 3, and 8 — the ones that assert on intermediate state — cannot be written for the Tokio version at all without changing the implementation to expose state it would not naturally have.

---

## User
##### 5/2/2026, 11:48:43 AM

Correct me if I'm wrong, Javascript today uses this model ? Excluding the worker-threads option. And Javascript uses what in Twisted Python were "deferreds" and in Javascript are "promises", for the purpose of easing ergonomics. (Historical footnote: it's no accident, the Javascript people learned it from Twisted Python, and/or from the E programming language by Mark S. Miller and Marc Stiegler, and Twisted Python learned it from the E programming language.)

---

## Assistant
##### 5/2/2026, 11:48:43 AM

You're right on both the technical claim and the history.

## The technical claim

JavaScript (excluding Web Workers and Node's worker_threads) is a single-threaded event loop with non-blocking I/O, which is the same model as Twisted and the same model as the synchronous reactor pattern we've been discussing. Every JavaScript function runs to completion before any other JavaScript runs — the "run-to-completion" guarantee. I/O completions, timers, and`Promise` resolutions enqueue callbacks that the event loop dispatches one at a time, at the top of the loop, never in the middle of another function. This is precisely the local-reasoning property: between any two synchronous statements in a JavaScript function, nothing else in the program ran.

`Promise` (and`async`/`await`, which is sugar over`Promise`) is the ergonomic layer on top, exactly analogous to Twisted's`Deferred`. The underlying model is still callbacks-on-a-reactor; promises are a composable representation of "a value that will be delivered to a callback later," with chaining (`.then`) and error propagation built in so you do not have to hand-write the callback pyramid.

The one caveat worth stating precisely:`await` in JavaScript *is* a yield point in the same sense as Rust's`.await`, so the local-reasoning property weakens at`await` boundaries within an`async` function — other queued tasks can run there. But the underlying model is still single-threaded and run-to-completion between yield points, which is stronger than what you get with OS threads, and the yield points are syntactically explicit. It is the same trade-off Twisted made when it added`inlineCallbacks` and the same one Rust made with`async`/`await`.

## The history

The lineage is well-documented and goes roughly: the term and concept of "promise" for concurrent computation traces to Daniel P. Friedman and David Wise's 1976 work and Henry Baker and Carl Hewitt's 1977 "futures," but the *modern* form — promise pipelining, the resolver/promise split, the chaining semantics — was developed by Mark S. Miller, Dean Tribble, and Rob Jellinghaus in the Project Xanadu work around 1989, and then refined substantially in the E programming language by Miller and others through the late 1990s and 2000s. E established the vocabulary and semantics that everything downstream inherited.

Twisted's`Deferred` (Glyph Lefkowitz, around 2002) was directly inspired by E's promises. The Python community got the abstraction from E via Twisted.

JavaScript got it next, but through Twisted rather than directly from E. The Dojo Toolkit added`dojo.Deferred` in 2007, explicitly modelled on Twisted's`Deferred` (Alex Russell cited Twisted and MochiKit, and MochiKit's author Bob Ippolito has confirmed he ported the design from Twisted). Kris Zyp's 2009 CommonJS Promises/A proposal cited`dojo.Deferred` as its influence. Promises/A+ and eventually the ES2015`Promise` standard descend from that line.

So the chain is: E (Miller, Tribble, Stiegler, et al.) → Twisted (Lefkowitz) → Dojo (Russell, via Ippolito's MochiKit) → CommonJS Promises/A (Zyp) → Promises/A+ → ES2015`Promise` →`async`/`await`. Mark Miller himself was later directly involved in the JavaScript standardisation work (he was on TC39 and contributed to the promise specification), so the influence is not just genealogical — he showed up in person to make sure JavaScript got it right.

The Rust`async`/`await` design absorbed lessons from the JavaScript and C# experience but chose a different runtime model (zero-cost futures, polling, no built-in executor), so the family resemblance is at the syntax-and-ergonomics level rather than the underlying mechanism. But the conceptual lineage — "represent a not-yet-available value as a first-class object you can chain operations onto" — runs through all of them and originates with E.

Your footnote is correct and, if anything, understates how direct the influence was: it was not parallel evolution or convergent design, it was Mark Miller's ideas propagating through specific people who cited their sources.

---

