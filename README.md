# Week 2 – Core Concepts & Move Basics

---

## <a id="1"></a>1 · Creating & Exploring a Package

### 1.1 Why start with a package?

A **Move package** is the atomic *unit of code* on Sui.
Everything else you do in the ecosystem—publishing, upgrading, dependency management, IDE support—assumes your code lives in that structure. Creating one from the CLI therefore accomplishes three things at once:

1. Produces the directory scaffold learners will navigate in every lesson.
2. Teaches that **immutability** of code and **mutability** of data are separated (immutable byte‑code resides in the package, mutable objects are created at run time).
3. Gives an instant “Hello, World!” win when the first build succeeds—important psychological milestone.

### 1.2 Scaffolding

```bash
sui move new hello_world
tree hello_world -L 2
```

```
hello_world
├─ Move.toml        # human‑editable manifest
├─ sources/         # every .move file compiled by default
│  └─ hello_world.move
└─ tests/           # .move files compiled *only* under `sui move test`
   └─ hello_world_tests.move
```

*Pedagogy tip* – Ask students **why** tests live in a separate tree (answer: code in `tests/` is never published on‑chain and can freely use #\[test\_only] helpers).

### 1.3 The first build

```bash
cd hello_world
sui move build
```

*What the compiler does*

* Resolves the dependency graph declared in `Move.toml`.
* Downloads Move standard library and Sui framework (git SHAs are pinned in the output).
* Emits byte‑code plus ABI JSON into `build/`.
* Computes two hash digests and writes them to **Move.lock**—a deterministic “fingerprint” of the build.

If the build fails, 99 % of the time the message tells you *exactly* which line is wrong—highlight that to alleviate beginner anxiety.

---

## <a id="2"></a>2 · Package Manifest Deep‑Dive (`Move.toml`)

### 2.1 The `[package]` table

| Field          | Purpose                                                                                                                                         | Gotcha |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| `name`         | Human‑readable and importable id. Must be lower‑snake‑case, no spaces.                                                                          |        |
| `version`      | SemVer. Doesn’t affect the chain, but tooling (release pipelines, crates.io‑style registries) rely on it.                                       |        |
| `published-at` | *Optional.* Filled in **after** the first publish. Lets downstream packages refer to yours via the alias *instead* of the long 32‑byte address. |        |

### 2.2 `[dependencies]`

*Accepts either*

* **git** imports `(git = "...", rev = "...", subdir = "...")` – perfectly reproducible.
* **local** paths for iterative, intra‑repo work.

`dev-dependencies` override versions **only** for `sui move test` and `sui move build --dev`; perfect to pin a bleeding‑edge branch without breaking main‑net builds.

### 2.3 `[addresses]`

Think of it as the “hosts file” for Move: an alias on the left (`sui`) maps to a 32‑byte literal on the right (`"0x2"`). Inside code you can now write

```move
use sui::coin;
```

instead of

```move
use 0x2::coin;
```

Alias indirection means that
*you can upgrade dependencies* **without editing source code**—simply re‑point the alias to an upgraded package address.

---

## <a id="3"></a>3 · Addresses, Accounts & the CLI

### 3.1 Creating a keypair

```bash
sui client new-address ed25519
```

* Stores the private key encrypted by a pass‑phrase in `$HOME/.sui/keystore`.
* Adds an entry to `client.yaml` so the CLI knows which RPC url & which key belong together.

### 3.2 Funding with faucet

```bash
sui client faucet
```

*DevNet* is a public QA network reset every few weeks.
Emphasise that running a faucet locally (`sui start --with-faucet`) is also possible for offline workshops.

### 3.3 Everyday commands

```bash
sui client active-address        # current signer
sui client switch --address 0x…  # change signer
sui client objects               # list owned IDs
sui client gas                   # printable balance table
```

*Pedagogy tip* – Encourage learners to run `sui client objects --json` and notice how Sui already “knows” about `Coin<SUI>` types before they write any custom code.

---

## <a id="4"></a>4 · Writing & Building Your First Module

Below is a fully working “Hello World” contract with **four** learning hooks:

1. Shows the required `has key, store` abilities.
2. Demonstrates dependency imports (`std::string`, `sui::object`, …).
3. Uses an `entry` function (tx entry‑point).
4. Performs a *transfer* so the sender ends up owning the object.

```move
module hello_world::hello_world {

    use std::string;
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};

    /// The on‑chain greeting object
    struct Greeting has key, store {
        id: UID,
        text: string::String,
    }

    /// Mint and send to caller
    public entry fun mint(ctx: &mut TxContext) {
        // 1. instantiate object
        let g = Greeting {
            id: object::new(ctx),                  // fresh UID
            text: string::utf8(b"👋 Hello, Move!"),
        };
        // 2. transfer to the tx sender
        transfer::public_transfer(g, tx_context::sender(ctx));
    }
}
```

> **Why do we call `public_transfer`, not `public_share`?**
> Because `Greeting` has the `key` ability and is therefore *owned*, not *shared*. The two functions encode that distinction at the type level.

---

## <a id="5"></a>5 · Move Language Basics

### 5.1 Variables & Constants

* `let` bindings are immutable *unless* you prefix with `mut`.
* Numeric literals are inferred as `u64` unless suffixed (`10u8`).
* Unused *non‑droppable* values cause a compile‑time **EUnusedValue** error.
  Use `let _ = value;` or explicitly destructure if you only want the side‑effect.

### 5.2 Types: the essentials

| Category          | Examples              | Notes                                                              |
| ----------------- | --------------------- | ------------------------------------------------------------------ |
| Unsigned integers | `u8 … u256`           | All overflow/underflow aborts.                                     |
| Scalars           | `bool`, `address`     | `address` equality is constant‑time.                               |
| Collections       | `vector<T>`           | Only mutable via `&mut vector<T>` borrow.                          |
| Wrappers          | `String`, `Option<T>` | `String` is guaranteed UTF‑8; `Option` is a *struct*, not an enum. |
| User types        | `struct`, `enum`      | Enums added in Move 2024; no recursion allowed.                    |

### 5.3 Expressions & Scope

A block is an expression:

```move
let hyp = {
    let a = 3u64;
    let b = 4u64;
    (a*a + b*b).sqrt()
};   // hyp == 5
```

Ownership ends at the closing brace **unless** a value is moved out (returned) or borrowed.

### 5.4 Control Flow

There is no `for` statement in Move.
Iteration is either

* **`loop {}` + `break`/`continue`** – explicit gas management, or
* **recursion** – the preferred, verifiable style.

```move
fun gcd(mut a: u64, mut b: u64): u64 {
    loop {
        if (b == 0) break a;
        let tmp = b;
        b = a % b;
        a = tmp;
    }
}
```

### 5.5 Functions & Visibility

| Modifier          | Accessible from        | Typical use           |
| ----------------- | ---------------------- | --------------------- |
| *(none)*          | Same module            | private helpers       |
| `public(package)` | Any module in same pkg | internal API surface  |
| `public`          | Any package            | external API surface  |
| `entry`           | Transaction VM         | on‑chain entry points |

> **Rule** – An `entry` function *must* return `()`; if you need a value, write a `public` wrapper and call it from a tiny `entry` that discards the result.

### 5.6 Testing

* The Move VM starts with a fresh memory state every test.
* You can call `object::new_test_only()` to fabricate `UID`s without TxContext.
* `#[expected_failure(abort_code = …)]` validates that *exact* abort is thrown—perfect for security invariants.

---

## <a id="6"></a>6 · Objects, UID & Abilities

### 6.1 Why `key` and `store`?

* `key` marks a struct as **address‑owned** – it can sit directly under an address in the global object tree.
* `store` is the complementary permission that allows the struct to appear **inside another key struct’s field**.

Without `store`, you cannot nest; without `key`, you cannot publish.
For almost every “real” on‑chain asset you write, **`has key, store`** is the correct pair.

### 6.2 Lifecycle of an object

1. Creation: `object::new(ctx)` inserts a fresh UID into the TxContext.
2. Use/mutation: pass `&mut` borrows through entry or internal functions.
3. Transfer: `transfer::public_transfer` (owned) or `transfer::public_share` (shared).
4. Destruction: pattern‑match to extract the UID, then `uid.delete()`.

> **Security note** – Deleting an object refunds storage rebate. Teach learners that reckless delete/mint cycles are an anti‑pattern, because it can be abused for gas farming.

---

## <a id="7"></a>7 · Transactions in Practice

### 7.1 What is a Programmable Transaction Block (PTB)?

A text DSL cooked into the CLI that serialises into the on‑chain “programmable tx” format.
It is *transaction‑builder‑as‑a‑service*: you declare the DAG of calls and the CLI does all argument wiring.

### 7.2 Minimal PTB anatomy

```bash
sui client ptb \
  --gas-budget 1e8 \                # always first
  --assign me @$MY_ADDR \           # pure arg
  --move-call $PKG::mod::mint  \    # returns result‑0
  --assign obj \                    # bind to var
  --transfer-objects "[obj]" me     # final command
```

Rules to emphasise:

* Commands execute **sequentially**; each may refer to previous results.
* `"[obj]"` brackets create a *vector* of objects—even a single one.
* If any command aborts, the whole block reverts (atomicity).

---

## <a id="8"></a>8 · Mini app – Recipe Book

### 8.1 Learning objectives

* Demonstrates every concept from Week 2 in <60 lines of code.
* Gives students a meaningful artefact (their first NFT‑ish object).
* Serves as soil for Week 3 extensions (events, upgrade, cross‑module calls).

### 8.2 Walk‑through structure

1. **Mint** – shows object instantiation + transfer.
2. **Mutator functions** – show `&mut` borrowing and vector operations.
3. **Burn** – shows pattern matching and UID deletion.

At each stage, ask students to:

* Inspect the tx in Explorer.
* Query the object JSON and verify fields.
* Run `sui client objects | grep Recipe` to see ownership update.

---

## <a id="A"></a>9 · Appendix A – CLI snippets

| Use‑case                                 | Command                              |
| ---------------------------------------- | ------------------------------------ |
| Build skipping re‑fetch                  | `sui move build --skip-fetch`        |
| Run only tests whose name contains *foo* | `sui move test foo`                  |
| Pretty‑print ABI JSON                    | `sui move view-abi --module modname` |
| Fetch 1 block of chain history           | `sui client query --tx 0xDIGEST`     |

Encourage saving frequent CLI invocations into *Makefile recipes* or shell scripts—this leads naturally into CI automation later.

---

## 🎈 Creative “Hello, Colorful Sui!” NFT

### Why include this example?

* Colour cycling illustrates **state mutation** on subsequent calls.
* Randomness is mocked with cheap arithmetic on TxContext; no oracle needed.
* Students can see immediate feedback by querying the `rgb` field after each `cycle`.

## 🤹🏻‍♂️ Part C – A Creative “Hello World” Variation

> *“Hello, Colorful Sui!”* – the contract mints a **Color** object that stores an RGB tuple and randomly cycles through hues on every call.

```move
module colorful::colorful_hello {

    use std::{vector, string};
    use sui::{object::{Self, UID}, tx_context::{Self, TxContext}, transfer};

    struct Color has key, store {
        id: UID,
        rgb: vector<u8>,          // [R,G,B]
        greeting: string::String, // text
    }

    /// Mint a random cheerful color
    public entry fun mint(ctx: &mut TxContext) {
        let r = tx_context::fresh_id(ctx) as u8;
        let g = (r * 73) % 256;
        let b = (g * 59) % 256;

        let c = Color {
            id: object::new(ctx),
            rgb: vector[r, g, b],
            greeting: string::utf8(b"Hello, Colorful Sui!"),
        };
        transfer::public_transfer(c, tx_context::sender(ctx));
    }

    /// Spin the wheel – produces a new shade
    public entry fun cycle(c: &mut Color) {
        let r = c.rgb[0];
        c.rgb = vector[(r + 97) % 256, (r + 173) % 256, (r + 251) % 256];
    }
}
```

Every user ends up owning a uniquely‑colored greeting NFT that morphs each time they call `cycle`. Great first NFT experiment!


Challenge learners to:

1. Emit an `event` every time the colour changes (Week 3 topic).
2. Add a `copy, drop` “Badge” struct that lets only badge holders call `cycle`.

---

Here it the Recipe Book code again in one place—**copy‑paste ready**.

---

## 📂 Folder layout

```
recipe_book/
 ├─ Move.toml
 ├─ sources/
 │   └─ recipe_book.move          # ← full module below
 └─ tests/
     └─ recipe_tests.move         # ← unit test below
```

---

### 🔸 Move.toml (minimal)

```toml
[package]
name    = "recipe_book"
version = "0.1.0"

[addresses]
recipe_book = "0x0"      # local alias while developing
sui          = "0x2"      # Sui Framework
```

---

### 🔸 sources/recipe\_book.move

```move
module recipe_book::recipe_book {

    /* ---------- imports ---------- */
    use std::{string, vector};
    use sui::object::{Self, UID};
    use sui::tx_context::{Self, TxContext};
    use sui::{transfer, tx_context};

    /* ---------- object ---------- */
    /// A user‑owned recipe NFT
    struct Recipe has key, store {
        id: UID,
        title: string::String,
        ingredients: vector<string::String>,
        steps: vector<string::String>,
    }

    /* ---------- entry: mint ---------- */
    /// Mint a blank recipe owned by caller
    public entry fun mint(title_bytes: vector<u8>, ctx: &mut TxContext) {
        let r = Recipe {
            id: object::new(ctx),
            title: string::utf8(title_bytes),
            ingredients: vector::empty<string::String>(),
            steps: vector::empty<string::String>(),
        };
        transfer::public_transfer(r, tx_context::sender(ctx));
    }

    /* ---------- mutators ---------- */
    public fun add_ingredient(r: &mut Recipe, bytes: vector<u8>) {
        r.ingredients.push_back(string::utf8(bytes))
    }

    public fun add_step(r: &mut Recipe, bytes: vector<u8>) {
        r.steps.push_back(string::utf8(bytes))
    }

    /* ---------- entry: burn ---------- */
    /// Permanently delete the recipe (refunds storage rebate)
    public entry fun burn(r: Recipe) {
        let Recipe { id, .. } = r;
        id.delete()
    }
}
```

---

### 🔸 tests/recipe\_tests.move

```move
#[test]
fun happy_path() {
    use recipe_book::recipe_book;
    use sui::object;

    /* fabricate UID for local test */
    let mut r = recipe_book::Recipe {
        id: object::new_test_only(),
        title: b"Pancakes".to_string(),
        ingredients: vector[],
        steps: vector[],
    };

    recipe_book::add_ingredient(&mut r, b"Flour");
    recipe_book::add_ingredient(&mut r, b"Milk");
    assert!(r.ingredients.length() == 2);

    recipe_book::add_step(&mut r, b"Mix");
    recipe_book::add_step(&mut r, b"Fry");
    assert!(r.steps.length() == 2);
}
```

---

## 🏃‍♂️ Quick build & publish

```bash
# 1 — compile
sui move build

# 2 — publish on Devnet or Localnet
sui client publish --gas-budget 100000000 ./recipe_book
# ▸ Copy the PackageID from the output
```

## 🚀 Interact via PTB

```bash
PKG=0x...               # paste PackageID
ME=$(sui client active-address)

# mint a new recipe titled “Bread”
sui client ptb \
  --gas-budget 1e8 \
  --assign me @$ME \
  --move-call $PKG::recipe_book::mint b"Bread" \
  --assign rec \
  --transfer-objects "[rec]" me
```

You now own an on‑chain `Recipe` object that you can mutate with further PTB calls:

```bash
sui client ptb \
  --gas-budget 1e8 \
  --move-call $PKG::recipe_book::add_ingredient \
      @$REC_ID b"Yeast"
```

…and finally delete with:

```bash
sui client ptb \
  --gas-budget 1e8 \
  --move-call $PKG::recipe_book::burn @$REC_ID
```

---

That’s the complete **Recipe Book** example—module, tests, and CLI workflow.


