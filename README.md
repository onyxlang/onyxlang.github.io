# Onyx

Onyx is an experimental systems programming language heavily inspired by Rust.
See [@onyxlang](https://github.com/onyxlang) on GitHub.

## Brief History

After two years being an active member of Crystal community ([1](https://forum.crystal-lang.org/t/onyx-framework-is-released/433), [2](https://forum.crystal-lang.org/t/announcing-worcr-your-help-needed/585)) I, [Vlad Faust](https://vladfaust.com), realized that I wanted my own, perfect, programming language; I wanted to move on.
_Onyx_, the name, came from [Onyx Framework](https://onyxframework.com/), a web framework [developed](https://github.com/onyxframework) in Crystal by myself.
That's how Onyx began.

Back then in 2020, I tried to justify the language in these articles: [1](https://vladfaust.com/posts/2020-08-16-system-programming-in-2k20/), [2](https://vladfaust.com/posts/2020-08-20-the-onyx-programming-language/), primarily by roasting Rust.
Throughout the following years, I attempted to create the compiler in Crystal, Lua, C, C++ ([1](https://github.com/fancysofthq/phoenix), [2](https://github.com/fancysofthq/fnxc)), [Typescript](https://github.com/onyxlang/ts), and finally, in [Rust](https://github.com/onyxlang/rs).
I even began working on an [ISO-grade specification](https://nxsf.org/onyx/).

As it turns out, I never had a concrete idea of what Onyx should look like.
Instead, what I had was a feeling.
And that feeling used to change each time I switched my current stack, i.e. the language I were implementing the compiler in.
At some moment I had favor for Ruby-like `def`s, then for Javascript-like `function`, and now it's Rust-like `pub fn`.
Looks like I'm not ready to commit to a language yet, but to a set of features a language should have.

My latest endeavor is to implement the compiler in Rust. 
As it turns out, Rust is awesome.
And looks like there is practically little need for another language.
Yet, this journey's been fastinating.
I've learned a lot about compiler development, and was able to put my hands on plethora of technologies.
I'm grateful to my life for the chance to do this.

For the time being, I'm not sure if I'll continue working on Onyx.
With my passion for generalization, it may become a frontend for the HLIR project I'm also working on.

---

## Safety

In Onyx, there are three safety levels: `unsafe`, `fragile` and `safe`.
The code is usually implicitly `fragile`, unless it is marked as `unsafe` or `safe`.

It is legal to call a higher-safety code from a lower-safety context.
However, calling a lower-safety code from a higher-safety context requires explicit safety lowering, for example:

```onyx
extern void puts(char *);
// puts($"Hello, world!") // Error! Unsafe call from fragile context
unsafe! puts($"Hello, world!") // OK
```

A pointer with undefined lifetime is unsafe to access.
A function-local pointer is safe to access, but it can not be returned.
Access to other pointers is generally fragile.
Introduction of the `fragile` safety level elides the exclusive mutable reference requirement.

```onyx
threadsafe fn main() -> *Int32 {
  let x = 42;
  &x = 43; // OK, safe
  // &x // Error! Cannot return a local pointer
}
```

Pointer casting safety table:

| Lifetime  | To undefined | To `'static` | To caller | To local |
| --------- | ------------ | ------------ | --------- | -------- |
| Undefined |              | Unsafe       | Unsafe    | Unsafe   |
| `'static` | Safe         |              | Fragile   | Fragile  |
| Caller    | Safe         | Unsafe       |           | Fragile  |
| Local     | Safe         | Unsafe       | Unsafe    |

## Macros

Onyx has a flexible macro system for code augmentation.
Macros are written in a turing-complete dynamic language (presumably Lua) with full writeable access to the compilation context (nodes, environment etc).

```onyx
fn main() {
  #% for i in 1..3 do
    puts(#{ i })
  #% end

  // The macro is expanded literally to:
  // puts(1)
  // puts(2)
  // puts(3)
}
```

_Delayed macros_ are evaluated not at the time of syntax parsing, but at the time of need, be it specialization or macro function call.

```onyx
pub fn foo<T>(x: T) {
  puts(\#{ T.name.stringify() });
}

foo(42)    // puts("Int32")
foo("bar") // puts("String")
```

### Macro functions (exp)

Macro functions may be defined right within Onyx code, and called from it using the `#` prefix.

```onyx
pub macro print(times) \#{{ "%" }}{
  for i in 1..times do
    emit("puts(" .. i .. ");")
  end
}

#print(3)

// Evaluates to:
// puts(1);puts(2);puts(3)
```

## FFI

FFI is denoted by the `$` prefix, mostly designed after C.

### Extern

An `extern` statement is followed by a single FFI statement.
Onyx macros may be used in extern blocks.

```onyx
extern void onyx_db_init(void* db) {
  (unsafe! db as Box<DB>).init()
}

struct DB { 
  initialized: Bool
}

impl DB {
  pub fn new() ->> Self { initialized: false }
  pub fn init(&mut self) ->> self.initialized = true;
  pub fn is_initialized(&self) ->> self.initialized
}

fn main() {
  let db = new Box(new DB());
  unsafe! $onyx_db_init(&db as $void*);
  @assert(db.is_initialized());
}
```

## Functions

`fn foo() ->> x` infers type from its return value `x`. 

## Generators

A generator expands to literal code.

```onyx
forall T
impl List<T> {
  pub fn each(&self, yield: fn(T) -> _) ->> {
    for i in 0..self.len() {
      yield(self[i])
    }
  }
}

fn main() {
  let list = new List<Int32>([1, 2, 3]);
  list.each((x) =>> puts(x));
}
```

## Variables

`x <-> y` to swap values; `x <<-> y` to swap values, returning the old value of `x` (_eject swap_).

Wrapping identifiers to allow Unicode:

```onyx
pub fn `фу`();
```

Allow question mark in identifiers:

```onyx
pub fn ready?() -> Result<Bool, Err>;

pub fn foo() -> Result<Bool, Err> {
  ready?()?; // To demonstrate the error propagation shortcut
}
```

## Templates

Literals as template arguments:

```onyx
pub struct Foo<T: #'bool> { }

forall T
impl Foo<T> {
  pub fn foo() {
    #% if T.value then
      puts("true");
    #% else
      puts("false");
    #% end
  }
}

fn main() {
  Foo<true>.foo(); // puts("true")
  Foo<false>.foo(); // puts("false")
}
```
