# The Rust Guessing Game, with Functional Programming

I wrote my first Rust code, taken from [Chapter
2](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html)
of The Rust Book. It was a great example; quite simple (a number
guessing game) but showing off enough of Rust's ways of doing things
to be instructive. Here's the vanilla code for completeness:


```rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1, 101);

    loop {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("You guessed: {}", guess);

        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}
```

I thought I'd try to take a functional-programming approach to the exercise.
It turned out to be a bit more work than I thought, but not by much.
The list is, more or less:

1. The syntax for returning the last expression in a function
   (importantly, no `;` is used after the expression in this case).
2. The syntax for returning values from a function.
3. Functions do not gain access to values in their environment.

These are all things I'm sure that I would have learned in short order
by continuing reading the book, though I'm less sure of when I would
have learned about (3). In effect, this just meant that we had to explicitly
pass in `secret_number` to functions that use it.

Now we arrive at a working version with recursive fuctions:

```rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1, 101);

    main_loop(&secret_number);

    fn main_loop(sec_num: &u32) -> () {
        let guess: u32 = get_valid_guess();
        println!("You guessed: {}", guess);

        match guess.cmp(&sec_num) {
            Ordering::Less => {
                println!("Too small!");
                main_loop(&sec_num);
            }
            Ordering::Greater => {
                println!("Too big!");
                main_loop(&sec_num);
            }
            Ordering::Equal => {
                println!("You win!");
            }
        }
    }

    fn get_valid_guess() -> u32 {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => get_valid_guess(),
        }
    }
}
```

This is a bit longer than the original example, but in my opinion,
the logic is slighly more clear, as we can see explicitly where
the `main_loop` recurses, and perhaps more importantly, we were
able to isolate `get_valid_guess` as a function (also recursive),
so that no additional control-flow logic or error-handling related
to obtaining invalid numbers appears outside of the function.
This, in turn, makes `main_loop` easier to reason about.

But Rust currently does not have tail-call elimination [due
largely](https://github.com/rust-lang/rfcs/pull/1888) to an
underlying, long-standing issue in LLVM, which means we should always
be careful with recursive functions in Rust. If you're interested, you
can see [this blog post](https://dev.to/seanchen1991/the-story-of-tail-call-optimizations-in-rust-35hf)
showing some of the related history.

## The Final Result with Tramp

To demonstrate the best way to do this currently, we can use the
[Tramp](https://docs.rs/tramp) macro. I don't know anything about Rust
macros as of this writing, but following the directions from the Tramp
page were clear enough:


```rust
#[macro_use]
extern crate tramp;

use rand::Rng;
use std::cmp::Ordering;
use std::io;
use tramp::{tramp, Rec};

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1, 101);

    tramp(main_loop(secret_number));

    fn main_loop(sec_num: u32) -> Rec<()> {
        let guess: u32 = tramp(get_valid_guess());
        println!("You guessed: {}", guess);

        match guess.cmp(&sec_num) {
            Ordering::Less => {
                println!("Too small!");
                rec_call!(main_loop(sec_num));
            }
            Ordering::Greater => {
                println!("Too big!");
                rec_call!(main_loop(sec_num));
            }
            Ordering::Equal => {
                rec_ret!(println!("You win!"));
            }
        }
    }

    fn get_valid_guess() -> Rec<u32> {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        match guess.trim().parse() {
            Ok(num) => rec_ret!(num),
            Err(_) => rec_call!(get_valid_guess()),
        }
    }
}
```

You'll notice that we are no longer passing a reference `u32` to `main_loop`,
but are copying a value to passed in. This is becase, if we just add the Tramp-related
calls and types, such as `rec_call!`, we run into this error.

```
  --> src/main.rs:23:17
   |
16 |     fn main_loop(sec_num: &u32) -> Rec<()> {
   |                           ---- help: add explicit lifetime `'static` to the type of `sec_num`: `&'static u32`
...
23 |                 rec_call!(main_loop(&sec_num));
   |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ lifetime `'static` required
   |
   = note: this error originates in a macro (in Nightly builds, run with -Z macro-backtrace for more info)

```

This is a point I don't fully understand yet, but Tramp's
documentation seems to [point
out](https://docs.rs/tramp/0.3.0/tramp/#types) that static lifetimes
are expected. This might be an efficiency consideration in some code.

There are no [pure
functions](https://github.com/JasonShin/fp-core.rs#purity) in this
simple game, thus there was no need to try using Rusts's [const
functions](https://doc.rust-lang.org/reference/items/functions.html#const-functions)
to try to have the Rust's type checker guarantee our assumptions of
purity.

Overall, transitioning the example to FP was pleasant enough, and obviously,
this wasn't performance-critical code. For code where execution times becomes
immportant, even an FP supporter like myself should make a comment indicating
why using loop control-flow statements like `loop`, `continue`, and `break`
are necessary, as well as other notions like local mutability that Rust is
so good at making safe.

