# Onyx

Onyx is an experimental opinionated high-level programming language natively interopeable with Rust and C.
See [@onyxlang](https://github.com/onyxlang) on GitHub.

## Brief History

After two years being an active member of Crystal community ([1](https://forum.crystal-lang.org/t/onyx-framework-is-released/433), [2](https://forum.crystal-lang.org/t/announcing-worcr-your-help-needed/585)) I, [Vlad Faust](https://vladfaust.com), realized that I wanted my own, perfect, programming language.
_Onyx_, the name, came from [Onyx Framework](https://onyxframework.com/), a web framework [developed](https://github.com/onyxframework) in Crystal by myself.

Back then in 2020, I tried to justify the language in these articles: [1](https://vladfaust.com/posts/2020-08-16-system-programming-in-2k20/), [2](https://vladfaust.com/posts/2020-08-20-the-onyx-programming-language/), primarily by roasting Rust.
Throughout the following years, I attempted to create the compiler in Crystal, Lua, C, C++ ([1](https://github.com/fancysofthq/phoenix), [2](https://github.com/fancysofthq/fnxc)), [Typescript](https://github.com/onyxlang/ts), and finally, in [Rust](https://github.com/onyxlang/rs).
I even began working on an [ISO-grade specification](https://nxsf.org/onyx/).

As it turns out, I never had a concrete idea of what Onyx should look like.
Instead, what I had was a feeling.
And that feeling used to change each time I switched my stack, i.e. the language I were implementing the compiler in.
At some moment I had favor for Ruby-like `def`s, then for Javascript-like `function`, and now it's Rust-like `fn`.
Looks like I'm not ready to commit to a language yet, but to a set of features a language should have.

My latest endeavor is to implement the compiler in Rust. 
And it turns out that Rust is mostly awesome, still having some quirks standing in the way of everyday programming.
Looks like the best application for Onyx would be an inter-set of Rust tuned for high-level programming; this would allow to leverage the existing ecosystem and tooling.

---

## Basic principles

An Onyx program may directly call Rust code, but not vice versa.
Onyx programs assume Rust and C stdlibs presence.
Onyx may use (properly licensed) third-party crates on the language level.

## Safety

In Onyx, there are three safety levels: unsafe, fragile and safe.
Compared to Rust, fragile is the level in-between unsafe and safe.

Onyx code is fragile by default, unless marked explicitly.
A Rust function call is safe by default.

It is legal to call a higher-safety code from a lower-safety context.
However, calling a lower-safety code from a higher-safety context requires explicit safety lowering, for example:

```onyx
extern void puts(char *);

fn main() {
  // puts($"Hello, world!") // Error! Unsafe call from safe context

  // Note that macros in Onyx are called in other way than in Rust; 
  // `unsafe!` is a keyword, not a macro.
  unsafe! puts($"Hello, world!") // OK
}
```

A reference with undefined lifetime is unsafe to access.
A function-local reference is safe to access, but it can not be returned.
Access to other references is generally fragile.

Introduction of the fragile safety level elides the exclusive mutable reference requirement, transferring the thread-safety guarantee to the programmer.

```onyx
fn mutate(x: &mut i32, y: &mut i32) {
  *x = 43
  *y = 44
}

fn main() {
  let mut i = 42
  mutate(&mut i, &mut i) // Multiple mutable references!
  #println("{}", i)
}
```

## Macros

Onyx features a flexible macro system for code augmentation.
Macros are written in a line-oriented language (like Lua) with rich access to the compilation context (AST nodes, environment etc).

`#{% %}` is a non-emitting macro invocation operator.
`#{{ }}` emits the result of the macro invocation into code.

```onyx
fn main() {
  #{% for i in 1..3 do %}
    puts(#{{ i }})
  #{% end %}

  // The macro expands literally to:
  // puts(1)
  // puts(2)
  // puts(3)
}
```

_Delayed macros_ are prefixed with `\` and evaluated not at the time of syntax parsing, but at the time of need, e.g. specialization or macro function call.

```onyx
pub fn foo<T>(x: T) {
  puts(#\{{ T.name.stringify() }})
}

foo(42)    // puts("Int32")
foo("bar") // puts("String")
```

### Macro functions

In Onyx, instead of `macro!` syntax, a macro function is called with `#macro`, which is also applicable to Rust macro calls.
This is done to reduce "screaming" code.

```onyx
pub macro foo(times) #\{%
  for i in 1..times do
    emit("puts(" .. i .. ");")
  end
%}

fn main() {
  #foo(3) // puts(1);puts(2);puts(3)
  #assert(true)
}
```

## FFI

An FFI entity is referenced from Onyx via the `$` prefix, followed by a C (and only C) entity.

### FFI literals

No need to for importing a `libc` crate.

```onyx
fn main() {
  let c_str: $char*'static = $"Hello, world!" # A NTBS (C) string literal
  let str: string = "Hello, world" # An `std::string` struct instance

  let c_int: $int = $42 # A C `int` literal (variable bytes)
  let int32: i32 = 42 # A `i32` struct, always 4 bytes

  let c_unsigned_long_int: $`unsigned long int` = $42LU
  let unsigned64: u64 = 42u64
}
```

### Extern

An `extern` statement generally uses C syntax for declarations.

```onyx
#[link(name = "snappy")]
extern {
  // Note that, unlike in Rust, this is native C syntax;
  // type mapping is performed on the caller site (or implicitly).
  size_t snappy_max_compressed_length(size_t source_length);
}
```

```onyx
// If an extern function declaration is followed by a body, 
// it is implemented in Onyx. `#[no_mangle]` is implied.
extern void onyx_db_init(void* db) {
  (unsafe! db as Box<DB>).init()
}

struct DB { 
  get initialized?: bool
}

impl DB {
  pub fn new() ->> Self { initialized?: false }
  pub fn init(&mut self) ->> self.initialized? = true
}

fn main() {
  let db = new Box(new DB())
  unsafe! $onyx_db_init(&db as $void*)
  #assert(db.initialized?())
}
```

## Structs

`new T` is an alias to `T::new`.

A struct field name may end with a single `?` (e.g. `pub ready?: bool`).

A `get x` modifier would implement a public getter `pub fn x()` for the field.

## Literals

`void` is an alias to `()`.

### Heredocs

```onyx
fn main() {
  #assert_eq(<<-EOF
  Hello,
    world!
  EOF, "\n  Hello,\n    world!\n")

  // Unindented heredoc (calculated from the least indented line):
  #assert_eq(<<-|EOF
  Hello,
    world!
  EOF, "Hello,\n  world!\n")
} 
```

## Functions

`fn foo() ->> x` infers type from its return value `x`. 

### Closures

A closure requires explicit capturing.

```onyx
fn main() {
  let color: string = "green"
  let print = [&color]() ->> #println("`color`: {}", color)
  print()
}
```

### Generators

A generator expands to literal code iff not moved.
This allows a generator to implicitly capture all the variables in its scope.

```onyx
forall T
impl List<T> {
  // Couldn't pass a generator here if `yield` were moved.
  pub fn each(&self, yield: fn (T) -> _) ->> {
    for i in 0..self.len() {
      yield(self[i])
    }
  }
}

fn main() {
  let list = new List<Int32>([1, 2, 3])
  list.each((x) =>> puts(x))
  // for i in 0..list.len() {
  //   puts(list[i])
  // }
}
```

## Variables

`x <-> y` to swap values; `x <<-> y` to swap values, returning the old value of `x` (_eject swap_).

Allow question mark in identifiers:

```onyx
pub fn ready?() -> bool !Err;
```

## Errors

Use `try expr` (Zig-like) instead of `expr?` (Rust-like).

`-> T !U` is a shortcut to `-> Result<T, U>`, `-> !T` is a shortcut to `-> Result<void, T>`.

`throw expr` is a shortcut to `return Err(expr)`.

```onyx
import { rand } from "std"

// Return type `-> i32 !&$char` is inferred.
fn answer() ->> {
  if rand(1) == 0 {
    throw $"Error!"
  }

  42 // Expands to `Ok(42)`
}

fn main() -> i32 !&$char {
  // let result: Result<i32, &$char> = answer()
  // result // Would work (same return type)

  let result: i32 = try answer()
  result
}
```

## Templates

`t<T>` instead of `t::<T>`.

May use literals as template arguments:

```onyx
pub struct Foo<T: #'bool> { }

forall T
impl Foo<T> {
  pub fn foo() {
    #{% if T.value then %}
      puts("true")
    #{% else %}
      puts("false")
    #{% end %}
  }
}

fn main() {
  Foo<true>.foo() // puts("true")
  Foo<false>.foo() // puts("false")
}
```

## Async

Onyx uses a well-known third-party module to implement async/await, such as tokio.
`#[tokio::main]` is implied for the entry-file `fn main()`.

```onyx
async fn foo() -> T !E {
  // let x: Future<Result<T, E>> = bar()
  // let x: Result<T, E> = await bar()
  let x: T = try await bar()
  x
}
```

IDEA: `~> T` is a shortcut to `-> Future<T>`.

## Option

`T??` is a shortcut to `Option<T>`.

## Semicolons

If a statement or expression evaluates to `void`, a semicolon is optional.

## More ideas

* Implicitly wrap top-level code into `fn main()`.

### Imports

`import { foo } from "bar.nx"` to import from Onyx files.
Familiar `use bar::foo` syntax for Rust modules.
This would allow to avoid ambiguity between Onyx and Rust modules.
