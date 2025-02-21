In short, Rust compile will add a poll method for each task-like function,which is marked async. Also, the compile will add a state machine to the function object so the the executor can know where to start the task.

```rust=
bb0: {
        _20 = deref_copy (_1.0: &mut {async fn body of task1()});
        _19 = discriminant((*_20));
    // As you can see here, it cathc the state machine to _19 and jump to the corresponding block to run.
        switchInt(move _19) -> [0: bb1, 1: bb20, 2: bb19, 3: bb18, otherwise: bb10];
    }
```

Rust provides a playground that we can convert the rust code to different levles of compiled code. To observe how async works, we focue on MIR. Here is the playgound [link](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=30074856e62e74e91f06abd19bd72ece).


And this is the example I put in(It cannot run but can be compiled, I think it's enough to understand the async)
```rust=
use std::task::{Context, Poll};
use std::future::Future;
use std::pin::Pin;
use std::time::Duration;
use std::thread;
//use async_task::spawn;
use futures::task::waker_ref;
use futures::future::BoxFuture;
use std::sync::{Arc, Mutex};

struct MyExecutor {
    task_queue: Vec<BoxFuture<'static, ()>>,
}

impl MyExecutor {
    fn new() -> Self {
        MyExecutor { task_queue: Vec::new() }
    }

    fn spawn<F>(&mut self, task: F)
    where
        F: Future<Output = ()> + 'static + std::marker::Send,
    {
        let fut = Box::pin(task);
        self.task_queue.push(fut);
    }

    fn run(&mut self) {
        while !self.task_queue.is_empty() {
            let mut i = 0;
            while i < self.task_queue.len() {
                let task = &mut self.task_queue[i];

                let waker = futures::task::noop_waker_ref();
                let mut cx = Context::from_waker(waker);

                if task.as_mut().poll(&mut cx) == Poll::Ready(()) {
                    self.task_queue.remove(i);
                } else {
                    i += 1;
                }
            }

            thread::sleep(Duration::from_millis(10));
        }
    }
}

async fn task1() {
    println!("Task 1: Starting...");
    //thread::sleep(Duration::from_secs(2));
    tokio::time::sleep(Duration::from_secs(2)).await;
    println!("Task 1: Done!");
}

async fn task2() {
    println!("Task 2: Starting...");
    //thread::sleep(Duration::from_secs(1));
    tokio::time::sleep(Duration::from_secs(1)).await;
    println!("Task 2: Done!");
}

fn main() {
    let mut executor = MyExecutor::new();

    executor.spawn(task1());
    executor.spawn(task2());

    executor.run();
}

```

Here is the MIR result:
```rust=
fn <impl at src/main.rs:15:1: 15:16>::new() -> MyExecutor {
    let mut _0: MyExecutor;
    let mut _1: std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>;

    bb0: {
        _1 = Vec::<Pin<Box<dyn futures::Future<Output = ()> + std::marker::Send>>>::new() -> [return: bb1, unwind continue];
    }

    bb1: {
        _0 = MyExecutor { task_queue: move _1 };
        return;
    }
}

fn <impl at src/main.rs:15:1: 15:16>::spawn(_1: &mut MyExecutor, _2: F) -> () {
    debug self => _1;
    debug task => _2;
    let mut _0: ();
    let _3: std::pin::Pin<std::boxed::Box<F>>;
    let _4: ();
    let mut _5: &mut std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>;
    let mut _6: std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>;
    scope 1 {
        debug fut => _3;
    }

    bb0: {
        _3 = Box::<F>::pin(move _2) -> [return: bb1, unwind continue];
    }

    bb1: {
        _5 = &mut ((*_1).0: std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>);
        _6 = move _3 as std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>> (PointerCoercion(Unsize, Implicit));
        _4 = Vec::<Pin<Box<dyn futures::Future<Output = ()> + std::marker::Send>>>::push(move _5, move _6) -> [return: bb2, unwind continue];
    }

    bb2: {
        return;
    }
}

fn <impl at src/main.rs:15:1: 15:16>::run(_1: &mut MyExecutor) -> () {
    debug self => _1;
    let mut _0: ();
    let mut _2: bool;
    let mut _3: &std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>;
    let mut _4: usize;
    let mut _5: bool;
    let mut _6: usize;
    let mut _7: usize;
    let mut _8: &std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>;
    let mut _9: &mut std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>;
    let mut _10: &mut std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>;
    let mut _11: usize;
    let mut _14: bool;
    let mut _15: &std::task::Poll<()>;
    let _16: std::task::Poll<()>;
    let mut _17: std::pin::Pin<&mut dyn futures::Future<Output = ()> + std::marker::Send>;
    let mut _18: &mut std::task::Context<'_>;
    let mut _19: &std::task::Poll<()>;
    let _20: std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>;
    let mut _21: &mut std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>;
    let mut _22: usize;
    let mut _23: (usize, bool);
    let _24: ();
    let mut _25: std::time::Duration;
    scope 1 {
        debug i => _4;
        scope 2 {
            debug task => _9;
            let _12: &std::task::Waker;
            scope 3 {
                debug waker => _12;
                let mut _13: std::task::Context<'_>;
                scope 4 {
                    debug cx => _13;
                }
            }
        }
    }

    bb0: {
        goto -> bb1;
    }

    bb1: {
        _3 = &((*_1).0: std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>);
        _2 = Vec::<Pin<Box<dyn futures::Future<Output = ()> + std::marker::Send>>>::is_empty(move _3) -> [return: bb2, unwind continue];
    }

    bb2: {
        switchInt(move _2) -> [0: bb4, otherwise: bb3];
    }

    bb3: {
        return;
    }

    bb4: {
        _4 = const 0_usize;
        goto -> bb5;
    }

    bb5: {
        _6 = copy _4;
        _8 = &((*_1).0: std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>);
        _7 = Vec::<Pin<Box<dyn futures::Future<Output = ()> + std::marker::Send>>>::len(move _8) -> [return: bb6, unwind continue];
    }

    bb6: {
        _5 = Lt(move _6, move _7);
        switchInt(move _5) -> [0: bb18, otherwise: bb7];
    }

    bb7: {
        _10 = &mut ((*_1).0: std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>);
        _11 = copy _4;
        _9 = <Vec<Pin<Box<dyn futures::Future<Output = ()> + std::marker::Send>>> as IndexMut<usize>>::index_mut(move _10, move _11) -> [return: bb8, unwind continue];
    }

    bb8: {
        _12 = noop_waker_ref() -> [return: bb9, unwind continue];
    }

    bb9: {
        _13 = Context::<'_>::from_waker(copy _12) -> [return: bb10, unwind continue];
    }

    bb10: {
        _17 = Pin::<Box<dyn futures::Future<Output = ()> + std::marker::Send>>::as_mut(copy _9) -> [return: bb11, unwind continue];
    }

    bb11: {
        _18 = &mut _13;
        _16 = <dyn futures::Future<Output = ()> + std::marker::Send as futures::Future>::poll(move _17, copy _18) -> [return: bb12, unwind continue];
    }

    bb12: {
        _15 = &_16;
        _19 = const MyExecutor::run::promoted[0];
        _14 = <Poll<()> as PartialEq>::eq(move _15, move _19) -> [return: bb13, unwind continue];
    }

    bb13: {
        switchInt(move _14) -> [0: bb16, otherwise: bb14];
    }

    bb14: {
        _21 = &mut ((*_1).0: std::vec::Vec<std::pin::Pin<std::boxed::Box<dyn futures::Future<Output = ()> + std::marker::Send>>>);
        _22 = copy _4;
        _20 = Vec::<Pin<Box<dyn futures::Future<Output = ()> + std::marker::Send>>>::remove(move _21, move _22) -> [return: bb15, unwind continue];
    }

    bb15: {
        drop(_20) -> [return: bb5, unwind continue];
    }

    bb16: {
        _23 = AddWithOverflow(copy _4, const 1_usize);
        assert(!move (_23.1: bool), "attempt to compute `{} + {}`, which would overflow", copy _4, const 1_usize) -> [success: bb17, unwind continue];
    }

    bb17: {
        _4 = move (_23.0: usize);
        goto -> bb5;
    }

    bb18: {
        _25 = Duration::from_millis(const 10_u64) -> [return: bb19, unwind continue];
    }

    bb19: {
        _24 = std::thread::sleep(move _25) -> [return: bb1, unwind continue];
    }
}

const <impl at src/main.rs:15:1: 15:16>::run::promoted[0]: &Poll<()> = {
    let mut _0: &std::task::Poll<()>;
    let mut _1: std::task::Poll<()>;
    let mut _2: ();

    bb0: {
        _2 = ();
        _1 = Poll::<()>::Ready(move _2);
        _0 = &_1;
        return;
    }
}

fn task1() -> {async fn body of task1()} {
    let mut _0: {async fn body of task1()};

    bb0: {
        _0 = {coroutine@src/main.rs:49:18: 54:2 (#0)};
        return;
    }
}

fn task1::{closure#0}(_1: Pin<&mut {async fn body of task1()}>, _2: &mut Context<'_>) -> Poll<()> {
    debug _task_context => _18;
    let mut _0: std::task::Poll<()>;
    let _3: ();
    let mut _4: std::fmt::Arguments<'_>;
    let _5: &[&str; 1];
    let mut _6: tokio::time::Sleep;
    let mut _7: tokio::time::Sleep;
    let mut _8: std::time::Duration;
    let mut _9: std::task::Poll<()>;
    let mut _10: std::pin::Pin<&mut tokio::time::Sleep>;
    let mut _11: &mut tokio::time::Sleep;
    let mut _12: &mut std::task::Context<'_>;
    let mut _13: isize;
    let _15: ();
    let mut _16: std::fmt::Arguments<'_>;
    let _17: &[&str; 1];
    let mut _18: &mut std::task::Context<'_>;
    let mut _19: u32;
    let mut _20: &mut {async fn body of task1()};
    let mut _21: &mut {async fn body of task1()};
    let mut _22: &mut {async fn body of task1()};
    let mut _23: &mut {async fn body of task1()};
    let mut _24: &mut {async fn body of task1()};
    let mut _25: &mut {async fn body of task1()};
    let mut _26: &mut {async fn body of task1()};
    let mut _27: &mut {async fn body of task1()};
    scope 1 {
        debug __awaitee => (((*(_1.0: &mut {async fn body of task1()})) as variant#3).0: tokio::time::Sleep);
        let _14: ();
        scope 2 {
            debug result => _14;
        }
    }

    bb0: {
        _20 = deref_copy (_1.0: &mut {async fn body of task1()});
        _19 = discriminant((*_20));
        switchInt(move _19) -> [0: bb1, 1: bb20, 2: bb19, 3: bb18, otherwise: bb10];
    }

    bb1: {
        _18 = move _2;
        _5 = const task1::{closure#0}::promoted[1];
        _4 = Arguments::<'_>::new_const::<1>(copy _5) -> [return: bb2, unwind: bb17];
    }

    bb2: {
        _3 = _print(move _4) -> [return: bb3, unwind: bb17];
    }

    bb3: {
        _8 = Duration::from_secs(const 2_u64) -> [return: bb4, unwind: bb17];
    }

    bb4: {
        _7 = tokio::time::sleep(move _8) -> [return: bb5, unwind: bb17];
    }

    bb5: {
        _6 = <Sleep as std::future::IntoFuture>::into_future(move _7) -> [return: bb6, unwind: bb17];
    }

    bb6: {
        _21 = deref_copy (_1.0: &mut {async fn body of task1()});
        (((*_21) as variant#3).0: tokio::time::Sleep) = move _6;
        goto -> bb7;
    }

    bb7: {
        _22 = deref_copy (_1.0: &mut {async fn body of task1()});
        _11 = &mut (((*_22) as variant#3).0: tokio::time::Sleep);
        _10 = Pin::<&mut Sleep>::new_unchecked(copy _11) -> [return: bb8, unwind: bb16];
    }

    bb8: {
        _12 = copy _18;
        _9 = <Sleep as futures::Future>::poll(move _10, copy _12) -> [return: bb9, unwind: bb16];
    }

    bb9: {
        _13 = discriminant(_9);
        switchInt(move _13) -> [0: bb12, 1: bb11, otherwise: bb10];
    }

    bb10: {
        unreachable;
    }

    bb11: {
        _0 = Poll::<()>::Pending;
        _23 = deref_copy (_1.0: &mut {async fn body of task1()});
        discriminant((*_23)) = 3;
        return;
    }

    bb12: {
        _14 = copy ((_9 as Ready).0: ());
        _24 = deref_copy (_1.0: &mut {async fn body of task1()});
        drop((((*_24) as variant#3).0: tokio::time::Sleep)) -> [return: bb13, unwind: bb17];
    }

    bb13: {
        _17 = const task1::{closure#0}::promoted[0];
        _16 = Arguments::<'_>::new_const::<1>(copy _17) -> [return: bb14, unwind: bb17];
    }

    bb14: {
        _15 = _print(move _16) -> [return: bb15, unwind: bb17];
    }

    bb15: {
        _0 = Poll::<()>::Ready(const ());
        _25 = deref_copy (_1.0: &mut {async fn body of task1()});
        discriminant((*_25)) = 1;
        return;
    }

    bb16 (cleanup): {
        _26 = deref_copy (_1.0: &mut {async fn body of task1()});
        drop((((*_26) as variant#3).0: tokio::time::Sleep)) -> [return: bb17, unwind terminate(cleanup)];
    }

    bb17 (cleanup): {
        _27 = deref_copy (_1.0: &mut {async fn body of task1()});
        discriminant((*_27)) = 2;
        resume;
    }

    bb18: {
        _18 = move _2;
        goto -> bb7;
    }

    bb19: {
        assert(const false, "`async fn` resumed after panicking") -> [success: bb19, unwind continue];
    }

    bb20: {
        assert(const false, "`async fn` resumed after completion") -> [success: bb20, unwind continue];
    }
}

const task1::{closure#0}::promoted[0]: &[&str; 1] = {
    let mut _0: &[&str; 1];
    let mut _1: [&str; 1];

    bb0: {
        _1 = [const "Task 1: Done!\n"];
        _0 = &_1;
        return;
    }
}

const task1::{closure#0}::promoted[1]: &[&str; 1] = {
    let mut _0: &[&str; 1];
    let mut _1: [&str; 1];

    bb0: {
        _1 = [const "Task 1: Starting...\n"];
        _0 = &_1;
        return;
    }
}

fn task2() -> {async fn body of task2()} {
    let mut _0: {async fn body of task2()};

    bb0: {
        _0 = {coroutine@src/main.rs:56:18: 61:2 (#0)};
        return;
    }
}

fn task2::{closure#0}(_1: Pin<&mut {async fn body of task2()}>, _2: &mut Context<'_>) -> Poll<()> {
    debug _task_context => _18;
    let mut _0: std::task::Poll<()>;
    let _3: ();
    let mut _4: std::fmt::Arguments<'_>;
    let _5: &[&str; 1];
    let mut _6: tokio::time::Sleep;
    let mut _7: tokio::time::Sleep;
    let mut _8: std::time::Duration;
    let mut _9: std::task::Poll<()>;
    let mut _10: std::pin::Pin<&mut tokio::time::Sleep>;
    let mut _11: &mut tokio::time::Sleep;
    let mut _12: &mut std::task::Context<'_>;
    let mut _13: isize;
    let _15: ();
    let mut _16: std::fmt::Arguments<'_>;
    let _17: &[&str; 1];
    let mut _18: &mut std::task::Context<'_>;
    let mut _19: u32;
    let mut _20: &mut {async fn body of task2()};
    let mut _21: &mut {async fn body of task2()};
    let mut _22: &mut {async fn body of task2()};
    let mut _23: &mut {async fn body of task2()};
    let mut _24: &mut {async fn body of task2()};
    let mut _25: &mut {async fn body of task2()};
    let mut _26: &mut {async fn body of task2()};
    let mut _27: &mut {async fn body of task2()};
    scope 1 {
        debug __awaitee => (((*(_1.0: &mut {async fn body of task2()})) as variant#3).0: tokio::time::Sleep);
        let _14: ();
        scope 2 {
            debug result => _14;
        }
    }

    bb0: {
        _20 = deref_copy (_1.0: &mut {async fn body of task2()});
        _19 = discriminant((*_20));
        switchInt(move _19) -> [0: bb1, 1: bb20, 2: bb19, 3: bb18, otherwise: bb10];
    }

    bb1: {
        _18 = move _2;
        _5 = const task2::{closure#0}::promoted[1];
        _4 = Arguments::<'_>::new_const::<1>(copy _5) -> [return: bb2, unwind: bb17];
    }

    bb2: {
        _3 = _print(move _4) -> [return: bb3, unwind: bb17];
    }

    bb3: {
        _8 = Duration::from_secs(const 1_u64) -> [return: bb4, unwind: bb17];
    }

    bb4: {
        _7 = tokio::time::sleep(move _8) -> [return: bb5, unwind: bb17];
    }

    bb5: {
        _6 = <Sleep as std::future::IntoFuture>::into_future(move _7) -> [return: bb6, unwind: bb17];
    }

    bb6: {
        _21 = deref_copy (_1.0: &mut {async fn body of task2()});
        (((*_21) as variant#3).0: tokio::time::Sleep) = move _6;
        goto -> bb7;
    }

    bb7: {
        _22 = deref_copy (_1.0: &mut {async fn body of task2()});
        _11 = &mut (((*_22) as variant#3).0: tokio::time::Sleep);
        _10 = Pin::<&mut Sleep>::new_unchecked(copy _11) -> [return: bb8, unwind: bb16];
    }

    bb8: {
        _12 = copy _18;
        _9 = <Sleep as futures::Future>::poll(move _10, copy _12) -> [return: bb9, unwind: bb16];
    }

    bb9: {
        _13 = discriminant(_9);
        switchInt(move _13) -> [0: bb12, 1: bb11, otherwise: bb10];
    }

    bb10: {
        unreachable;
    }

    bb11: {
        _0 = Poll::<()>::Pending;
        _23 = deref_copy (_1.0: &mut {async fn body of task2()});
        discriminant((*_23)) = 3;
        return;
    }

    bb12: {
        _14 = copy ((_9 as Ready).0: ());
        _24 = deref_copy (_1.0: &mut {async fn body of task2()});
        drop((((*_24) as variant#3).0: tokio::time::Sleep)) -> [return: bb13, unwind: bb17];
    }

    bb13: {
        _17 = const task2::{closure#0}::promoted[0];
        _16 = Arguments::<'_>::new_const::<1>(copy _17) -> [return: bb14, unwind: bb17];
    }

    bb14: {
        _15 = _print(move _16) -> [return: bb15, unwind: bb17];
    }

    bb15: {
        _0 = Poll::<()>::Ready(const ());
        _25 = deref_copy (_1.0: &mut {async fn body of task2()});
        discriminant((*_25)) = 1;
        return;
    }

    bb16 (cleanup): {
        _26 = deref_copy (_1.0: &mut {async fn body of task2()});
        drop((((*_26) as variant#3).0: tokio::time::Sleep)) -> [return: bb17, unwind terminate(cleanup)];
    }

    bb17 (cleanup): {
        _27 = deref_copy (_1.0: &mut {async fn body of task2()});
        discriminant((*_27)) = 2;
        resume;
    }

    bb18: {
        _18 = move _2;
        goto -> bb7;
    }

    bb19: {
        assert(const false, "`async fn` resumed after panicking") -> [success: bb19, unwind continue];
    }

    bb20: {
        assert(const false, "`async fn` resumed after completion") -> [success: bb20, unwind continue];
    }
}

const task2::{closure#0}::promoted[0]: &[&str; 1] = {
    let mut _0: &[&str; 1];
    let mut _1: [&str; 1];

    bb0: {
        _1 = [const "Task 2: Done!\n"];
        _0 = &_1;
        return;
    }
}

const task2::{closure#0}::promoted[1]: &[&str; 1] = {
    let mut _0: &[&str; 1];
    let mut _1: [&str; 1];

    bb0: {
        _1 = [const "Task 2: Starting...\n"];
        _0 = &_1;
        return;
    }
}

fn main() -> () {
    let mut _0: ();
    let mut _1: MyExecutor;
    let _2: ();
    let mut _3: &mut MyExecutor;
    let mut _4: {async fn body of task1()};
    let _5: ();
    let mut _6: &mut MyExecutor;
    let mut _7: {async fn body of task2()};
    let _8: ();
    let mut _9: &mut MyExecutor;
    scope 1 {
        debug executor => _1;
    }

    bb0: {
        _1 = MyExecutor::new() -> [return: bb1, unwind continue];
    }

    bb1: {
        _3 = &mut _1;
        _4 = task1() -> [return: bb2, unwind: bb8];
    }

    bb2: {
        _2 = MyExecutor::spawn::<{async fn body of task1()}>(move _3, move _4) -> [return: bb3, unwind: bb8];
    }

    bb3: {
        _6 = &mut _1;
        _7 = task2() -> [return: bb4, unwind: bb8];
    }

    bb4: {
        _5 = MyExecutor::spawn::<{async fn body of task2()}>(move _6, move _7) -> [return: bb5, unwind: bb8];
    }

    bb5: {
        _9 = &mut _1;
        _8 = MyExecutor::run(move _9) -> [return: bb6, unwind: bb8];
    }

    bb6: {
        drop(_1) -> [return: bb7, unwind continue];
    }

    bb7: {
        return;
    }

    bb8 (cleanup): {
        drop(_1) -> [return: bb9, unwind terminate(cleanup)];
    }

    bb9 (cleanup): {
        resume;
    }
}
```

And here are some QAs to understand this MIR code:

***Q:Can you help me to understand this Rust MIR code:
_23 = deref_copy (_1.0: &mut {async fn body of task1()});***

The Rust **MIR (Mid-level Intermediate Representation)** is a representation of Rust code that is closer to machine-level operations than the high-level Rust source code. It helps to analyze how Rust compiles and optimizes code. The specific code you're referring to involves `deref_copy`, which is used in Rust to copy a value from a reference.

Let's break down the statement you provided:

```rust
_23 = deref_copy (_1.0: &mut {async fn body of task1()});
```

### Understanding the Components

1. **_1.0: &mut {async fn body of task1()}**:
   - This indicates that `_1` is a variable that holds a tuple, and the `.0` field of the tuple contains a mutable reference to an asynchronous function. In Rust's MIR, functions (or closures) are represented as references when they are passed around (they are essentially pointers to code). 
   - `&mut {async fn body of task1()}` means that `_1.0` is a mutable reference to the body of an async function (`task1`).
   - Async functions in Rust have a special structure internally, as they are compiled into state machines that can be paused and resumed during execution. The body of the async function will be a state machine of sorts.

2. **deref_copy**:
   - `deref_copy` is an operation in MIR that dereferences a reference and copies the value. In this case, it's dereferencing the mutable reference to the async function body (`&mut {async fn body of task1()}`), and then it copies the value.
   - Since async functions are usually compiled into a "state machine," the copying here might involve making a shallow copy of the function's internal state or its representation.
   
3. **_23 = deref_copy (...);**:
   - This assignment stores the result of the `deref_copy` operation into the variable `_23`. 
   - The idea is that `_23` now holds the copied value (or possibly the state of the function body). This could be part of an operation where the Rust compiler is handling the state of an async function by copying some internal state from the original function.

### What's Happening?

Here's a step-by-step explanation of what's likely happening here:

1. **Mutable Reference to the Async Function Body**:
   - `_1.0` contains a mutable reference to the body of the async function `task1()`. The async function is compiled into a state machine by the Rust compiler. This means `task1()` is not a normal function but rather a structure that implements `Future` or some similar trait.
   - The function's body (`async fn body of task1()`) here could be referring to the actual implementation of the state machine for that async function.

2. **Dereferencing and Copying**:
   - `deref_copy` is used to dereference the mutable reference (`&mut {async fn body of task1()}`) and copy the value.
   - In the case of async functions, copying the state machine might involve copying the state of the async operation or the future object that represents the function's execution.
   - The function body itself is not copied directly; rather, it's the internal structure (the state machine) that is copied.

3. **Assignment**:
   - The result of the `deref_copy` is assigned to `_23`, meaning `_23` now holds the copied internal state or a reference to the async function's state machine.

### Why Deref Copy?

The reason you would use `deref_copy` in this context is because of the need to work with a **reference** to the async function or its state machine, rather than the function or state machine itself. Since the function body (async function's state machine) is represented as a reference, you need to dereference it to access the actual internal value or state.

### More Detailed Example

Let's consider a simple example to help clarify the concept. 

```rust
async fn task1() {
    // Simulate some async work
    println!("Task 1 is running!");
}

fn main() {
    let async_fn = task1;
    let mut ref_fn = &async_fn; // Ref to async function

    // Deref-copy would be like creating a new reference or copying the state
    let copied_fn = ref_fn; // In MIR this could be represented by deref_copy
    // You could now work with `copied_fn` as needed
}
```

In this simplified example, we are taking a reference to `task1` (an async function) and then "copying" it. In MIR, `deref_copy` would be handling this reference and copying the internal state, allowing it to be used independently. In reality, async functions are much more complex because of their internal state machines, but the idea remains the same.

### Key Takeaways

1. **`deref_copy`** is a MIR operation used to dereference a reference and copy the value it points to.
2. In the case of async functions, this could involve copying the internal state machine that represents the asynchronous operation.
3. The operation is typically part of the compiler's process to manage asynchronous state and function bodies, especially when handling function pointers or closures in an async context.

If you need a deeper dive into the internals of async functions in MIR, I recommend exploring the `rustc` source code, as the handling of async functions involves specific transformations in the MIR that are not immediately obvious from just the high-level source code.

***Can you explain this as well?
fn task1::{closure#0}(_1: Pin<&mut {async fn body of task1()}>, _2: &mut Context<'_>) -> Poll<()>***

Certainly! The code you've provided is a part of the **MIR (Mid-level Intermediate Representation)** for an async function and represents how Rust compiles the asynchronous function and its underlying state machine. Let’s break it down step by step:

```rust
fn task1::{closure#0}(_1: Pin<&mut {async fn body of task1()}>, _2: &mut Context<'_>) -> Poll<()>
```

This line represents a **closure** that’s generated by Rust's async transformation. Specifically, it corresponds to the body of an **async function** (in this case, `task1`). Rust internally transforms `async` functions into state machines, and this line is a part of that transformation. Let's go through it to understand what's happening.

### Key Concepts

1. **Async Function Transformation**:  
   When you write an `async fn` in Rust, it is **not** directly translated into a normal function. Instead, it is transformed into a **state machine**. This means that the body of the async function is turned into a series of states that can be "paused" and "resumed" as needed, typically by calling `.poll()` on the returned `Future`.

2. **State Machine and Polling**:
   - The state machine uses the `poll` method, which is defined by the `Future` trait.
   - The state machine allows the function to yield control back to the executor (e.g., `tokio` or `async-std`) while waiting for asynchronous operations (like I/O, timers, etc.) to complete. The `poll` method is used to drive this state machine to completion.
   - The return value of `poll` is either `Poll::Pending` (meaning the task is not done yet) or `Poll::Ready(())` (meaning the task is complete).

### Breakdown of the Code

Let’s decompose each part of the line:

#### `fn task1::{closure#0}`
- This indicates that the function is part of the **closure** generated for the async function `task1`. Rust compiles the async function into a **closure** (which is an anonymous function) that implements the `Future` trait.
- The `{closure#0}` is an identifier used internally to represent this closure, which is part of the compiled state machine for the async function.

#### `(_1: Pin<&mut {async fn body of task1()}>, _2: &mut Context<'_>) -> Poll<()>`
This is the signature of the closure's function. Let's break it down:

1. **`_1: Pin<&mut {async fn body of task1()}>`**:
   - `_1` is the first argument to this closure. It represents a **mutable pinned reference** to the **state machine** of the async function `task1`.
   - **`Pin<&mut T>`** is a special kind of pointer in Rust that guarantees that the value it points to will not be moved in memory. This is important because async functions can be paused and resumed, and moving them around in memory could break the task’s ability to resume correctly.
   - **`{async fn body of task1()}`** refers to the actual body of the `task1` function, which has been transformed into a state machine (or the `Future` implementation) by Rust’s async transformation.

2. **`_2: &mut Context<'_>`**:
   - `_2` is the second argument to this closure, which is a mutable reference to the `Context` object.
   - The `Context` is used by the `poll` method to store and access information about the task’s execution state. This includes things like the **waker** (used to notify the executor when the task can make progress) and **task-local data**. The `Context` allows the function to schedule its own wake-up and resume when needed.

3. **`-> Poll<()>`**:
   - The closure returns a `Poll<()>` value.
   - The `Poll` enum has two variants:
     - **`Poll::Pending`**: Indicates that the task is not ready yet and needs to be polled again.
     - **`Poll::Ready(())`**: Indicates that the task is complete and can return a result (in this case, `()` is the result type).

### What is Happening Here?

The line you’ve shared is part of the internal representation of the async function `task1`. It is essentially the closure that Rust generates to represent the async function body once it has been compiled into a **state machine**. Here’s the breakdown of what happens:

1. **State Machine Creation**:  
   The async function's body is wrapped in a closure that implements the `Future` trait. The closure represents the state machine of the async function.

2. **Polling**:  
   The closure takes a **mutable pinned reference** to the state machine (`_1: Pin<&mut {async fn body of task1()}>`) and a **mutable reference** to a `Context` (`_2: &mut Context<'_>`). The `poll` method is called to advance the state machine.

3. **Progress and Resumption**:  
   - Inside the closure, the state machine may decide to **yield control** (by returning `Poll::Pending`) if it needs to wait on some external event (like an I/O operation or a timer).
   - The **waker** inside the `Context` will notify the executor when the state machine should be polled again (e.g., when the async operation completes).
   - If the state machine is done, it will return `Poll::Ready(())`.

### Simplified Example

Let's simplify this with a real Rust example. Here’s an async function:

```rust
use async_nursery::run_async;

async fn task1() {
    // Simulate some work
    println!("Task 1 is working...");
}

fn main() {
    run_async(task1());
}
```

When `task1` is compiled, the Rust compiler internally turns it into a **state machine** (as part of the `Future` trait). The `poll` method of the state machine is responsible for checking whether the async function is done or if it needs to yield control.

In MIR, this transformation would look similar to:

```rust
fn task1::{closure#0}(_1: Pin<&mut {async fn body of task1()}>, _2: &mut Context<'_>) -> Poll<()> {
    // The body of task1 turned into a state machine and poll checks its progress
    if <some async condition> {
        Poll::Pending
    } else {
        Poll::Ready(())
    }
}
```

The closure now represents the internal function of the state machine, and every time the task is polled, it advances through its states until it completes.

### Conclusion

In summary:

- `task1::{closure#0}` is the closure that represents the body of the async function `task1`.
- The closure takes a `Pin<&mut {async fn body of task1()}>` (the state machine of `task1`) and a `Context` to track its execution state.
- The closure returns `Poll<()>`, which can either be `Poll::Pending` (if the task is not yet complete) or `Poll::Ready(())` (if the task is complete).

This is a key part of Rust's async model, which transforms `async fn` into state machines that can be polled to allow non-blocking execution.

***Q: Can you also explain this :
bb2: {
        _3 = _print(move _4) -> [return: bb3, unwind: bb17];
    }***

Certainly! The code you've provided is an example of **Rust's MIR (Mid-level Intermediate Representation)**. This is a low-level representation of Rust code used during the compilation process. It helps the Rust compiler optimize and understand how to translate high-level Rust code into machine code.

Let’s break it down step by step:

### MIR Code Breakdown:

```rust
bb2: {
    _3 = _print(move _4) -> [return: bb3, unwind: bb17];
}
```

1. **Basic Block (bb2):**
   - **`bb2:`** denotes the start of a **basic block**. In MIR, a basic block represents a sequence of instructions that are executed sequentially.
   - A basic block is the unit of control flow in Rust's MIR and typically ends with a jump to another block (in this case, `bb3` or `bb17`).

2. **`_3 = _print(move _4)`**
   - **`_3`**: This is a **local variable** that is being assigned the result of the operation `_print(move _4)`. This means that the value of `_4` is being moved into the function `_print`.
   - **`_print`**: This is likely a function that is being called in the MIR. It is used to represent some side-effect or an action in the code (in this case, it could be printing something, but the exact details depend on the context). In this example, `_print` is a function that consumes the value from `_4` and returns some result that gets stored in `_3`.
   - **`move _4`**: This indicates a **move** operation. The `move` keyword is telling the Rust compiler to transfer ownership of `_4` to the `_print` function. After the move, `_4` is no longer valid in the current scope, and `_print` takes ownership of it.
   
3. **`-> [return: bb3, unwind: bb17]`**
   - This part specifies the control flow after executing this block (`bb2`).
   - **`return: bb3`**: If the operation in `bb2` is successful (i.e., `_print` completes successfully), control will jump to basic block `bb3`. In the context of Rust’s async functions or error handling, `bb3` might be where execution continues after `_print` is done.
   - **`unwind: bb17`**: If an error or panic occurs while executing the operation, control will jump to basic block `bb17`. This block could handle errors, panics, or cleanup operations.

### What's Happening Here?

This part of the MIR code represents the execution of some logic in a basic block (bb2), which includes calling a function (`_print`) and handling both normal control flow (moving to `bb3`) and exceptional control flow (moving to `bb17` in case of an unwind). Here's how you can interpret the code:

1. **Execution Flow**:
   - The basic block `bb2` calls the function `_print`, passing `_4` by move.
   - The result of `_print(move _4)` is stored in `_3`.
   - After the execution of `_print`, if everything goes as expected, control will move to `bb3` (the "return" block).
   - If there’s an error or panic (e.g., `_print` encounters an issue), execution will jump to `bb17` (the "unwind" block).

2. **Function Call (`_print`)**:
   - The function `_print` takes ownership of `_4` (because of the `move` keyword) and does something with it (likely prints it).
   - After calling `_print`, the result is stored in `_3`. This suggests that `_print` could have a return value (maybe the number of characters printed or some other result), which is captured in `_3`.

3. **Control Flow**:
   - The basic block system helps manage control flow. In this case:
     - **`bb3`** represents the continuation of normal execution.
     - **`bb17`** represents an error or exception case (such as a panic), where Rust will perform some kind of unwind operation, like cleaning up resources or handling the panic.

### Visualizing the Flow

Let's visualize the flow of this MIR snippet:

```text
   ┌──────────────┐
   │    bb2       │  (Call _print, move _4)
   └─────┬────────┘
         │
         ▼
    _3 = _print(move _4)
         │
         ├──► if successful → jump to bb3 (return)
         └──► if error/panic → jump to bb17 (unwind)
```

In the above diagram:
- The control starts at `bb2`, where `_print` is called with `_4` (moved).
- If the function call is successful, execution moves to `bb3`.
- If there is an error or panic, execution moves to `bb17`.

### When Might This Happen in Rust?

This MIR code might appear in the compiled output of an async function, or it could be part of a function with error handling or panics. For example, if you were calling a function that might panic (like `unwrap` on a `Result` or `Option`), the `unwind` block would be responsible for handling the panic.

```rust
let result = some_function();
let _ = result.unwrap(); // Could cause panic
```

In this case, if `unwrap()` fails, control will jump to a block that handles unwinding (e.g., cleaning up resources).

### Conclusion

- The `bb2` block represents a piece of Rust code that calls the `_print` function with a `move` argument.
- It then either continues to `bb3` (if everything works fine) or jumps to `bb17` (if there’s a panic or error).
- This is part of Rust's MIR, which helps the compiler represent control flow, handle function calls, and manage errors or panics.

***Q: then what about this:
bb11: {
        _0 = Poll::<()>::Pending;
        _23 = deref_copy (_1.0: &mut {async fn body of task1()});
        discriminant((*_23)) = 3;
        return;
    }***
    
Let’s break down the provided MIR code to understand it:

```rust
bb11: {
    _0 = Poll::<()>::Pending;
    _23 = deref_copy (_1.0: &mut {async fn body of task1()});
    discriminant((*_23)) = 3;
    return;
}
```

This is another **MIR (Mid-level Intermediate Representation)** snippet that provides insight into how the Rust compiler internally represents the state of an **async function**. Let's go step-by-step to analyze what's happening here.

### Detailed Explanation:

1. **Basic Block (bb11)**:
   - `bb11:` represents a basic block in the MIR flow. A basic block is a sequence of instructions with no jumps or branches, but it can jump to another basic block upon completion (in this case, the block ends with a `return`, so it doesn't jump elsewhere).

2. **First Line: `_0 = Poll::<()>::Pending`**
   - **`Poll::<()>::Pending`** is a value that represents the **in-progress state** of a future. In Rust, `Poll` is an enum used by the async runtime to track the state of a `Future`. The `Poll::Pending` variant indicates that the future is **not yet complete** and is still being awaited.
   - **`_0`** is a **local variable** being assigned the value `Poll::Pending`.
   - This line is likely part of the async function state machine. The task represented by this future (e.g., `task1`) is **not yet ready** to complete, so it returns `Poll::Pending`.

3. **Second Line: `_23 = deref_copy (_1.0: &mut {async fn body of task1()})`**
   - **`_1.0`**: This refers to the **first element** of a struct or tuple. In the context of async functions, the `async fn` body is often represented as a **state machine** with a struct that holds the state of the function.
   - **`deref_copy`** is a dereferencing operation that makes a **copy** of the value the reference points to. The dereferencing ensures that we’re accessing the actual data behind the reference.
   - **`(_1.0: &mut {async fn body of task1()})`**: This indicates that `_1.0` is a mutable reference to a structure that represents the async function (`task1`).
   - **`_23`** is a **local variable** that now holds the dereferenced value of `_1.0`. Essentially, `_23` now points to the state machine of the async function (`task1`).

4. **Third Line: `discriminant((*_23)) = 3`**
   - **`discriminant`**: In Rust, the `discriminant` function is used to access the **discriminant** of an enum variant. An enum’s discriminant is a way to identify which variant of the enum is currently being used.
   - **`(*_23)`**: This dereferences the pointer stored in `_23`, which holds the state machine for the async function `task1`. This will give access to the enum variant inside the state machine.
   - **`= 3`**: The value `3` is being assigned to the discriminant of the state machine (likely an enum). In the case of `async` functions, this might represent a particular state in the async function’s state machine.
     - For example, in the state machine of an async function, different states might represent different points of execution in the function. The number `3` could correspond to a specific state, like the function being in a waiting state (e.g., after an `await` point).
     - This might indicate that the state machine has reached a certain point, such as waiting on an I/O operation or waiting for the next async step.

5. **Last Line: `return;`**
   - The `return;` statement is the **end of this basic block**. It indicates that this part of the function has finished its execution. Since there is no jump to another block, it’s returning control to the async runtime or moving on to the next part of the function (which is probably another basic block or an await point).

### What’s Happening Here?

This part of the MIR corresponds to handling a **pending async task**. Here's a breakdown of the steps:

1. **Marking the Task as Pending**:
   - The task has not yet completed, so `Poll::Pending` is set in `_0`. This is the standard way async tasks inform the executor that they are not yet ready and need to be polled again later.

2. **Derefencing the Async Task’s State**:
   - `_1.0` contains the state machine for the async function (`task1`). Dereferencing it using `deref_copy` extracts the current state, which is stored in `_23`.

3. **Setting the Discriminant of the Async State Machine**:
   - The discriminant is set to `3`. This likely corresponds to a specific state in the async function’s state machine (such as waiting for an I/O operation, or awaiting another task).
   - Rust's async functions compile to a **state machine**. Every time an async function is polled, it progresses through different states. The discriminant indicates which state the function is currently in.

4. **Returning from the Basic Block**:
   - The basic block ends with a `return`, indicating that this part of the function has finished executing. The task will likely be scheduled for polling again when its state changes.

### Context

- This MIR likely represents part of the **polling loop** for an async function. The function is being polled, and at this particular point in time, it's still **pending** (i.e., it hasn't completed yet).
- Async tasks are executed in a **state machine** where each `await` point is a state. When the task is awaited again, it may transition to a new state. The discriminant tracks which state the task is in at any given time.

### Visualizing the Flow

Here's a basic idea of how the flow works:

1. **Poll the Task**: The task is polled to see if it is ready to continue.
2. **Task is Pending**: The task is not yet ready, so `Poll::Pending` is returned, and the task will be polled again later.
3. **State Machine Transition**: The task’s state is updated, and the discriminant is set to `3`, indicating a specific state (perhaps waiting for an external event).
4. **Return**: The function returns, and the task will be polled again later.

### Conclusion

- The code represents part of the **polling mechanism** for an async function.
- `Poll::Pending` indicates that the task is still in progress.
- The state machine for the async function is being updated, with the discriminant being set to `3`, which corresponds to a particular state in the task.
- This MIR code is part of how Rust compiles `async` functions to manage their execution in an asynchronous runtime.

This sequence is part of Rust’s **state machine** for async functions, which ensures that they can be polled, and their execution can resume later once the function is ready.