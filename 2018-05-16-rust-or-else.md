# rust-or-else

Rust's `.unwrap_or_*()` methods are limited for complex cases.

## Solution
- https://play.rust-lang.org/?gist=061bd0c2a304a2329d1f87265b0e305e&version=stable&mode=debug
- https://play.rust-lang.org/?gist=57ab8b8d04e753933a0b5746b1b2c67b&version=stable&mode=debug

### Return Option
```rust
use std::error::Error;

fn main() {
    // Return type is `Ok(12)`.
    let mut res = cache_get1().or_else(|| db_get().ok());
    println!("1. {:?}", res);
    println!("1. {:?}", res.unwrap());

    // Return type is `Err(Ok(13))`.
    res = cache_get2().or_else(|| db_get().ok());
    println!("2. {:?}", res);
    println!("2. {:?}", res.unwrap());
}

fn cache_get1() -> Option<usize> {
    Some(12)
}

fn cache_get2() -> Option<usize> {
    None
}

fn db_get() -> Result<usize, Box<Error>> {
    Ok(13)
}
```

### Return Result
```rust
use std::error::Error;

fn main() {
    // Return type is `Ok(12)`.
    let mut res = cache_get1().ok_or_else(|| db_get());
    println!("1. {:?}", res);
    println!("1. {:?}", res.unwrap());

    // Return type is `Err(Ok(13))`.
    res = cache_get2().ok_or_else(|| db_get());
    println!("2. {:?}", res);
    println!("2. {:?}", res.unwrap());
}

fn cache_get1() -> Option<usize> {
    Some(12)
}

fn cache_get2() -> Option<usize> {
    None
}

fn db_get() -> Result<usize, Box<Error>> {
    Ok(13)
}
```

## Details
The use of `match` and `Option` is rather common in Rust. Often times you want
to check whether a value is `None`, and then perform an action to create a
value.

An example would be: "Look for an item in a local cache. If it's not in there,
get the value from the database."

```rust
let node = match self.cache.get(key) {
  Some(node) => node,
  None => &self.db.get(key)?,
};
```

For the sake of argument here, let's pretend that all lifetimes work well
(there's a separate piece on where this falls short).

In the example, the second line feels a bit boilerplaty to me. And it seems that
people agree with this, because `.unwrap_or()` exists.

```rust
let node = self.cache.get(key).unwrap_or(&self.db.get(key)?);
```

Now let's analyze this for a second. Our intended flow is: "check the cache
before doing a database lookup". But the code we wrote above _always_ performs
a database lookup. That's hardly what we intended.

But again, this case was accounted for. `.unwrap_or_else()` takes a closure,
conditionally executing the code within its block.

```rust
let node = self.cache.get(key).unwrap_or_else(|| {
  Ok(&self.db.get(key)?)
});
```

Except, this doesn't work. Lifetime issues here might be more relevant (read:
less solvable) than in the other two examples. But the biggest one is that
`.unwrap_or_else()` doesn't accept a `Result` type.

So we're forced to drop error handling, and write it using `.unwrap()`.

```rust
let node = self.cache.get(key).unwrap_or_else(|| {
  &self.db.get(key).unwrap();
});
```

Dropping error handling in favor of reducing boilerplate is not worth it.
Trading correctness for aesthetics doesn't seem like the right thing.

## Fixes
### Macro
The initial `match` statement we wrote felt right on the money. It works as
expected, albeit a bit verbose. With a macro we could do away with it.

We could do something like the std `assert!()` or failure `ensure!()` macros do
for error handling:

```rust
let node = unwrap!(self.cache.get(key), &self.db.get(key)?);
```

Which expands to the code we started off with:

```rust
let node = match self.cache.get(key) {
  Some(node) => node,
  None => &self.db.get(key)?,
};
```

The naming of the macro is entirely up for debate; I just chose something
because it needed a name. I hope the idea comes across regardless.

### Inlined Closures
Not all closures are the same. Some closures serve just to remove some
boilerplate inside a function. Whereas other closures allow spawning separate
threads on different CPUs. You often can't tell by looking at a closure.

A fun think I picked up from Kotlin the other day is that they allow annotating
their closures. You can tell whether a closure is inlined or not, and if it's
inlined you can call things like `continue` from them. It's all perfectly
safe because the annotations make its restrictions explicit.

I reckon Rust could so something similar. This could either be done in the
calling site, or inside the closure declaration. There's probably some more
thought that could go into that.

```rust
let node = self.cache.get(key).unwrap_or_else(inline || {
  Ok(&self.db.get(key)?)
});
```

It's probably worth pointing out that it is a bit verbose though. But I'm sure
that if that becomes a problem there might be ways to make it explicit for
certain types given enough time and use.

### Another Unwrap Variant
I don't like this one, but should probably be mentioned. There could be a
version where a fourth `.unwrap()` variant is added that accounts for `Result`
types.

```rust
let node = self.cache.get(key).unwrap_or_result(|| {
  Ok(&self.db.get(key)?)
});
```

But with 3 types already existing, there's a good chance this might confuse
people. I'm so-so about this.

### New Syntax
Well yeah, this is probably not a great place to start at. But it's probably
worth thinking about what a possible _future_ version of the macro version might
look like.

`if let` provides a specialization of the match statement (conditionally match
one arm). And this would be very similar.

For example, we could repurpose the `try` macro as a keyword. There's already a
proposal out to reserve it as a keyword for the 2018 edition.

```rust
let node = try self.cache.get(key) {
  &self.db.get(key)
};
```

_Note: It seems some folks want to nudge Rust's error handling towards allowing
`try..catch`, which I'm not a fan of. But that's a different discussion. My main
point is that reserving `try` seems like a reasonable thing, and it could be
nicely repurposed to streamline something like what we're experiencing here._

Some possible variations:

```rust
let node = if try self.cache.get(key) {
  &self.db.get(key)
};
```

```rust
let node = if unwrap self.cache.get(key) {
  &self.db.get(key)
};
```

```rust
let node = unwrap self.cache.get(key) else {
  &self.db.get(key)
};
```

```rust
let node = try self.cache.get(key) else {
  &self.db.get(key)
};
```

```rust
let node = self.cache.get(key) else {
  &self.db.get(key)
};
```

It goes without saying that these aren't the only ones possible. It'd be cool if
folks would think of what a this might look like in an ideal scenario - and then
trace back the least intrusive steps needed to get there.

## Wrapping up
I hope this all makes sense! I feel like the problem is an actual thing that
people run into. I'd be keen to hear what people think the right way would be to
solve this. Thanks!
