# Week 2 Deep‑Dive — Core Concepts & Move Basics  
*Zero‑to‑on‑chain in one weekend*

---

## Prerequisites

| Tool | Why you need it | Quick check |
|------|-----------------|-------------|
| **Sui CLI** | Build, test, publish & send txs | `sui client --version` → `sui-client 1.46.x` |
| **Rust** | Runs `sui start` localnet | `rustc --version` |
| **Git** | Fetch on‑chain dependencies | `git --version` |
| **Editor** | Auto‑complete / syntax | VS Code ➜ *Move Analyzer* |

---

## 1 · Move Packages

### 1.1 What is a package?

| Think… | Like… |
|--------|-------|
| Move package | **Cargo crate** (Rust) |
| Move module  | **Rust module** / **JS file** |
| `Move.toml`  | `Cargo.toml` / `package.json` |

A package is the **publishable unit**: all contained modules share **one on‑chain address**  
and are upgraded **atomically**.

---

### 1.2 Why should you care?

* Upgrading only one module = *broken imports* 🤕.  
  Publishing the whole package = zero mismatch.
* Import **by address** → no global namespace → name collisions impossible.

---

### 1.3 Skeleton of a fresh package

```

my\_pkg/
├─ Move.toml
├─ sources/     # on‑chain code
│  └─ \*.move
├─ tests/       # dev‑only
└─ examples/    # docs / demos

````

> Anything outside `sources/` **never hits the chain** – treat it like local helpers.

---

### 1.4 Hands‑on

```bash
# ❶ Scaffold
$ sui move new zoo

# ❷ Peek
$ tree zoo -L 2
````

Delete the `tests/` folder and run:

```bash
$ sui move test
```

*Expected* → compiler screams it cannot find test files it referenced.
That’s how you learn the default template wires tests into `Move.toml`.

---

## 2 · `Move.toml` Manifest

### 2.1 Key sections

| Header               | Purpose                             | Sneaky gotcha                            |
| -------------------- | ----------------------------------- | ---------------------------------------- |
| `[package]`          | name, semver, `edition = "2024"`    | Edition **must** be 2024 today.          |
| `[addresses]`        | Human aliases for 32‑byte addresses | Alias is global inside the package.      |
| `[dependencies]`     | Git / path deps                     | Std/Sui auto‑added since CLI 1.45        |
| `[dev-dependencies]` | Overrides only in test/dev          | Cannot *add* new aliases; only override. |

---

### 2.2 Two TOML styles

```toml
# Inline table
DeepBook = { git = "https://github.com/...", subdir = "packages/deepbook", rev = "mainnet" }

# Long form – identical
[dependencies.DeepBook]
git    = "https://github.com/..."
subdir = "packages/deepbook"
rev    = "mainnet"
```

---

### 2.3 Mini exercise

1. Inside `zoo/Move.toml` add:

   ```toml
   [dependencies]
   MoveStdlib = { git = "https://github.com/MystenLabs/sui.git", subdir = "crates/sui-framework/packages/move-stdlib", rev = "framework/mainnet" }
   ```
2. Run `sui move build`.
   *Observe* → CLI clones repo only *once*, then caches it in `.move`.
3. Switch `rev` to `main` and rebuild – note the re‑download. 🙈
   **Lesson**: pin to a commit for faster builds.

---

## 3 · Building, Testing, Dev‑build

| Command                 | Compiles                        | Runs tests? | When to use          |
| ----------------------- | ------------------------------- | ----------- | -------------------- |
| `sui move build`        | `sources/`                      | ✖           | Size & gas estimates |
| `sui move build --test` | `sources/ + tests/ + examples/` | ✖           | IDE IntelliSense     |
| `sui move test`         | same as above                   | ✔           | CI / local TDD       |

> **Pro tip 🛠️**
> Seeing *“Updating git dependency…”* every build?
> Use `rev = "<commit>"` instead of a branch tag.

---

## 4 · Modules

### 4.1 Two declaration syntaxes

```move
// ✨ Label syntax (modern)
module zoo::keeper;

// 🕰 Block syntax (legacy – still valid)
module zoo::keeper {
    // ...
}
```

Prefer label syntax – smaller diff noise during upgrades.

---

### 4.2 Visibility primer

| Keyword           | Who can call?                    | Real‑world usage       |
| ----------------- | -------------------------------- | ---------------------- |
| *(none)*          | Only same module                 | invariants, helpers    |
| `public`          | Anywhere                         | Library API            |
| `public(package)` | Any module *inside same package* | Multi‑module packages  |
| `entry`           | Transaction blocks               | Mint / burn / transfer |

Remember: **struct fields are *always* private**.
Expose them via `public fun` or `public use fun … as Struct.method`.

---

### 4.3 Exercise – hello module

Create `sources/hello.move`:

```move
/// Module that greets.
module zoo::hello;

use std::string::String;

public fun greet(name:String): String {
    b"Hello, ".to_string().append(name)
}
```

Run:

```bash
$ sui move build
```

Add a test in `tests/hello_test.move`:

```move
#[test_only] module zoo::hello_test;

use zoo::hello;

#[test]
fun greet_world() {
    assert!( hello::greet(b"World".to_string())
             == b"Hello, World".to_string(), 0);
}
```

```bash
$ sui move test greet
```

---

## 5 · Primitive Types in practice

```move
let flag = true;
let score:u16 = 900;
let addr  = @0x42;
let data  = vector[1u8, 2, 3];
let text  = b"Sui ♥ Move".to_string();
```

**Overflow safety**

```move
let x:u8 = 255;
let y = x + 1;   // aborts at runtime
```

“Opt‑out” overflow? Call `u8::add_mod`.

---

## 6 · Structs + Ability Grid (deep dive)

### 6.1 Cheat card

| Want this behaviour                | Set abilities                      |
| ---------------------------------- | ---------------------------------- |
| Plain data ‑ copyable, discardable | `copy, drop`                       |
| *Ticket* – must be used once       | *(none)*                           |
| On‑chain object                    | `key, store`                       |
| Object that nests assets           | `key, store` outer + `store` inner |

### 6.2 Demo – capability ticket

```move
module zoo::tickets;

use sui::{tx_context, object};

/// One‑time permission to mint any animal.
struct MintCap has key { id: UID }

public entry fun create_cap(ctx:&mut tx_context::TxContext): MintCap {
    MintCap { id: object::new(ctx) }
}

public entry fun mint_animal(
    cap: MintCap, ctx:&mut tx_context::TxContext, species:vector<u8>
) {
    let MintCap { id:_ } = cap;              // consume ticket
    let animal = zoo::Animal {               // mint
        id: object::new(ctx), species
    };
    transfer::transfer(animal, tx_context::sender(ctx));
}
```

Try calling `mint_animal` twice with the same `cap` – compiler forbids; the value is gone.

---

## 7 · Constants & human‑readable errors

```move
const MAX_USES:u8 = 3;

#[error] const ECardEmpty: vector<u8> = b"metro card empty";
```

Usage:

```move
assert!(card.uses > 0, ECardEmpty);
```

Shows a *string* in explorer / RPC instead of opaque numbers.

---

## 8 · Comments & docs

* `//` – single‑line
* `/* … */` – block or comment‑out
* `/// markdown` – doc comment → `move doctor` generates static HTML.

Doc comments support **GitHub‑flavoured MD** including code fences.

---

## 9 · Control Flow recipes

### 9.1 Guard clause

```move
public fun withdraw(balance:&mut u64, amt:u64) {
    if (*balance < amt) abort 0;
    *balance = *balance - amt;
}
```

### 9.2 Loop with filter

```move
let mut sum = 0;
for (x in 0..vec.length()) {      // sugar coming soon – for now use while
    let v = vec.borrow(x);
    if (*v % 2 == 1) continue;
    sum = sum + *v;
}
```

*(Real loop sugar arrives in 2025‑edition)*

---

## 10 · Ownership & borrowing quick‑start

```move
fun demo() {
    let mut c = Card { uses:3 };

    inspect(&c);       // read‑only borrow
    ride(&mut c);      // mut borrow
    recycle(c);        // move → c invalid afterwards
}
```

---

## 11 · Generics & phantom tags

### 11.1 Generic struct

```move
struct Box<T> has drop { val:T }

public fun wrap<T>(v:T): Box<T> { Box { val:v } }
```

### 11.2 Phantom type param

```move
struct Coin<phantom C> has drop { value:u64 }

struct USD {}
struct EUR {}

let usd: Coin<USD> = Coin { value:100 };
let eur: Coin<EUR> = Coin { value:100 };
```

Compiler forbids mixing `Coin<USD>` with `Coin<EUR>`.

---

## 12 · Enums + pattern match

```move
public enum Result has copy, drop {
    Ok,
    Err { code:u64 },
}

match (r) {
    Result::Ok => (),
    Result::Err { code } if (code == 404) => debug::print(&b"not found".to_string()),
    _ => abort 0,
}
```

Match **must** cover all possibilities; wildcard `_` is your friend.

---

## 13 · Transactions & PTB deep dive

### 13.1 Key commands

| Command           | Purpose                     |
| ----------------- | --------------------------- |
| `SplitCoins`      | break gas coin into change  |
| `MergeCoins`      | merge many into one         |
| `MoveCall`        | call module function        |
| `TransferObjects` | move object(s) to new owner |
| `Publish`         | upload package              |

### 13.2 Full PTB walk‑through

```bash
# variables
export ME=$(sui client active-address)
export PKG=0x...          # your package
export CARD=0x...         # object IDs printed after earlier tx

sui client ptb                               \
  --gas-budget 100000000                      \
  --assign me @$ME                            \
  --move-call $PKG::metro_pass::enter_metro @$CARD \
  --move-call $PKG::metro_pass::enter_metro @$CARD \
  --move-call $PKG::metro_pass::enter_metro @$CARD \
  --move-call $PKG::metro_pass::recycle      @$CARD
```

Observe gas & object changes:

```bash
sui client object $CARD        # should now be gone (recycled)
```

---

## 14 · Testing like a pro

```move
#[test]
fun happy() {
    let c = my::create();
    assert!(c.value == 1);
}

#[test, expected_failure(abort_code = my::EOverflow)]
fun overflow() { my::add(255u8, 1u8); }

#[test_only]
public fun helper(): u64 { 42 }     // visible only in tests
```

Run them:

```bash
$ sui move test                    # all
$ sui move test over               # by substring
```

---

## 15 · Publish to Localnet – step‑by‑step

```bash
# A · Start node (one terminal)
$ RUST_LOG="off,sui_node=info" sui start --with-faucet --force-regenesis

# B · New terminal – create account
$ sui client
$ sui client faucet

# C · Publish
$ cd zoo
$ sui client publish --gas-budget 200000000 --json > result.json
$ cat result.json | jq '.objectChanges[] | select(.objectType|test("::package::Package")) .packageId'
```

Copy that `packageId` for PTB scripts.

---

## 16 · PROJECT — “Todo + Bank”

> Touches every topic we’ve learned.

### 16.1 Repo layout

```
todo_bank/
├─ Move.toml
└─ sources/
   ├─ todo_list.move
   ├─ bank.move
   └─ entry.move
```

### 16.2 Important bits

* `TodoList` is `key, store` object (on‑chain state).
* `Coin<phantom C>` shows generics w/ phantom.
* `entry::demo` is a multi‑package transaction block.
* Unit tests cover happy path & failure path.

### 16.3 Try it yourself

```bash
git clone https://github.com/your-handle/todo_bank
cd todo_bank
sui client publish --gas-budget 2e8
# copy package id → $PKG
sui client ptb --gas-budget 1e8 \
  --move-call $PKG::entry::demo<usd> "'Read Week 2 Guide'" 100
```

Inspect objects:

```bash
sui client objects
```

You should see a coin & a list with one task.

---

## 17 · Troubleshooting quick‑ref

| Message fragment               | Meaning                                   | Fix                                |
| ------------------------------ | ----------------------------------------- | ---------------------------------- |
| *does not have ability ‘copy’* | tried `let b = a` but type isn’t copyable | pass by move or add `copy` ability |
| *Unused value of type…*        | type lacks `drop` & you ignored it        | destructure or mark `let _ = v`    |
| *Borrow check failed*          | simultaneous `&mut` and `&`               | shorten borrows; clone primitives  |
| *Port 9000 already in use*     | stale localnet                            | `pkill sui`                        |
| *re-downloads deps each build* | `rev` points to branch                    | pin to commit hash                 |

---

## 🚀 Practice checklist (do these & you *own* Week 2)

1. `sui move new bank`
2. Implement `Coin<phantom T>` (`drop`)
3. Entry fns: `mint`, `transfer`, `balance_of`
4. Write **3 tests** (happy, insufficient, multi‑transfer)
5. Publish + PTB two transfers in one tx

DM a maintainer your tx digest – we’ll send swag 🎉

---

### Next up…

*Week 3 →* Object upgrades · Events · Deep capability pattern ·
Cross‑package interfaces. Stay tuned!

```

---

**Teaching tip 📚**  
Spin up a live‑coding session where learners create `todo_bank` from scratch.  
Pause after each section, let them run the mini exercises, then resume.  
Nothing beats seeing the compiler *protect you* in real time!

Enjoy, and happy Moving 🦦🛠️
```
