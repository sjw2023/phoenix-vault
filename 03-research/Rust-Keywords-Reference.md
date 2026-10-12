---
title: Rust keywords — full reference, with where each appears in Phoenix
date: 2026-10-12
status: reference
revision: r1
tags: [phoenix, rust, reference, language]
source: "The Rust Programming Language, Appendix A: Keywords"
---

# Rust keywords — full reference

Definitions are **verbatim from** *The Rust Programming Language*, Appendix A — *Keywords*
(<https://doc.rust-lang.org/book/appendix-01-keywords.html>, checked 2026-10-12). The third column
is this project: counts come from `grep -rohwE` over `~/Workspace/phoenix/src/` on the same date,
so "not yet" means genuinely unused here, not unimportant.

## Keywords currently in use

| keyword | what it does (the Book's words) | in Phoenix |
|---|---|---|
| `as` | Perform primitive casting, disambiguate the specific trait containing an item, or rename items in `use` statements. | 1 — `hit.amount as i32` |
| `async` | Return a `Future` instead of blocking the current thread. | not yet |
| `await` | Suspend execution until the result of a `Future` is ready. | not yet |
| `break` | Exit a loop immediately. | not yet |
| `const` | Define constant items or constant raw pointers. | 13 — `ATTACK_RANGE`, `KNOCKBACK_SPEED`, `PLAYER_SPAWN` |
| `continue` | Continue to the next loop iteration. | 14 — the guard-and-skip shape in every reader |
| `crate` | In a module path, refers to the crate root. | 16 — `use crate::combat::Health` |
| `dyn` | Dynamic dispatch to a trait object. | not yet |
| `else` | Fallback for `if` and `if let` control flow constructs. | 13 — mostly `let … else { return; }` |
| `enum` | Define an enumeration. | 2 — `AppState`, `DeathMode` |
| `extern` | Link an external function or variable. | not yet |
| `false` | Boolean `false` literal. | not yet |
| `fn` | Define a function or the function pointer type. | 40 |
| `for` | Loop over items from an iterator, implement a trait, or specify a higher ranked lifetime. | 25 — both senses: `for x in &query` **and** `impl Plugin for EnemyPlugin` |
| `if` | Branch based on the result of a conditional expression. | 28 |
| `impl` | Implement inherent or trait functionality. | 9 — all trait impls; no inherent impl yet |
| `in` | Part of `for` loop syntax. | 16 |
| `let` | Bind a variable. | 35 |
| `loop` | Loop unconditionally. | not yet |
| `match` | Match a value to patterns. | 3 — `match *mode` over `DeathMode` |
| `mod` | Define a module. | 8 — one per file, all in `main.rs` |
| `move` | Make a closure take ownership of all its captures. | not yet |
| `mut` | Denote mutability in references, raw pointers, or pattern bindings. | **114** — the most-used keyword in the project |
| `pub` | Denote public visibility in struct fields, impl blocks, or modules. | 30 — note `pub struct MoveTarget(pub Vec3)` needs two |
| `ref` | Bind by reference. | not yet — modern match ergonomics usually remove the need |
| `return` | Return from function. | 14 — early returns in guards |
| `Self` | A type alias for the type we are defining or implementing. | 2 — `fn from_world(...) -> Self` |
| `self` | Method subject or current module. | 8 |
| `static` | Global variable or lifetime lasting the entire program execution. | not yet as a **keyword**; the `'static` **lifetime** appears in Bevy's bounds |
| `struct` | Define a structure. | 24 |
| `super` | Parent module of the current module. | not yet |
| `trait` | Define a trait. | **not yet** — Phoenix *implements* traits constantly but has never *defined* one |
| `true` | Boolean `true` literal. | 2 |
| `type` | Define a type alias or associated type. | not yet |
| `union` | Define a union; is a keyword only when used in a union declaration. | not yet |
| `unsafe` | Denote unsafe code, functions, traits, or implementations. | not yet |
| `use` | Bring symbols into scope. | 25 |
| `where` | Denote clauses that constrain a type. | not yet — bounds so far are inline, `impl<T: Fn(&mut App)>` style |
| `while` | Loop conditionally based on the result of an expression. | not yet |

## Keywords reserved for future use

No functionality today; reserved by Rust so they cannot be used as identifiers:

`abstract` · `become` · `box` · `do` · `final` · `gen` · `macro` · `override` · `priv` · `try` ·
`typeof` · `unsized` · `virtual` · `yield`

## Raw identifiers

`r#` lets a keyword be used as an identifier:

```rust
fn r#match(needle: &str, haystack: &str) -> bool {
    haystack.contains(needle)
}
```

Needed when calling into a crate from a different edition (`try` is a keyword from 2018 onward but
not in 2015), or binding to a language where the word is ordinary.

## Notable absences, and what they would unlock

- **`trait`** — defining one, rather than implementing Bevy's. The moment Phoenix has two things
  that must be interchangeable (two attack kinds, two AI behaviours), this is the tool.
- **`loop` / `while` / `break`** — a game loop never blocks, so every "repeat" here is instead
  *one frame's work* done by a system. Their absence is a property of the architecture, not an
  oversight.
- **`dyn`** — runtime polymorphism. Bevy leans on generics and monomorphisation, so it rarely
  surfaces in user code.
- **`move`** — needed when a closure must own its captures, which arrives with custom
  `Commands::queue` closures.
- **`where`** — same job as inline bounds, better when the list is long.
- **`unsafe`** — Bevy contains it; a gameplay project should not need it.

## Related

- [[Bevy-ECS-Architecture]] — how these keywords assemble into the engine's model
- [[Build-Log]] — where each one was first met in practice
