<details>
<summary><strong>Table of contents</strong></summary>

1. [Creating & Exploring a Package](#1)
2. [Package Manifest Deep‑Dive](#2)
3. [Addresses, Accounts & the CLI](#3)
4. [Writing & Building Your First Module](#4)
5. [Move Language Basics](#5)
     5.1 Variables & Constants 5.2 Types 5.3 Expressions & Scope
     5.4 Control Flow 5.5 Functions 5.6 Testing
6. [Objects, UID & Abilities](#6)
7. [Transactions in Practice](#7)
8. [Putting It All Together – a Mini app](#8)
9. [Appendix A – Common CLI snippets](#A)

</details>

---

<a name="1"></a>

### 1 · Creating & Exploring a Package

```bash
# ❶ scaffold
sui move new hello_world
tree hello_world -L 2
```

```text
hello_world
├─ Move.toml
├─ sources/
│  └─ hello_world.move
└─ tests/
   └─ hello_world_tests.move
```

```bash
# ❷ compile
cd hello_world
sui move build
```

*The compiler fetches Sui Framework/Stdlib, writes a **build/** folder and a checksum **Move.lock**.*

You are now ready to hack.

---

<a name="2"></a>

### 2 · Package Manifest Deep‑Dive

```toml
[package]
name = "hello_world"
version = "0.0.1"

[dependencies]                # ← git OR local
Sui = { git = "https://github.com/MystenLabs/sui.git", rev = "framework/mainnet", subdir = "crates/sui-framework/packages/sui-framework" }

[addresses]                   # ← aliases
hello_world = "0x0"           # local dev address
sui         = "0x2"
```

> **Tip –** after publishing, append
> `published-at = "0xabc…123"` so foreign packages may depend on you by alias.

`Move.lock` is emitted automatically and *locked into git* to guarantee deterministic builds.

---

<a name="3"></a>

### 3 · Addresses, Accounts & the CLI

| Command                           |  Purpose                          |
| --------------------------------- | --------------------------------- |
| `sui client new-address ed25519`  | create key‑pair & write `~/.sui/` |
| `sui client faucet`               | request devnet SUI                |
| `sui client switch --address 0x…` | choose active signer              |
| `sui client objects`              | list owned objects                |
| `sui client active-address`       | who am I?                         |

An **address literal** is written `@0x…`; an **alias** is `@sui`.

---

<a name="4"></a>

### 4 · Writing & Building Your First Module

```move
/// sources/hello_world.move
module hello_world::hello_world {

    use std::string;
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};

    /// A simple on‑chain greeting
    struct Greeting has key, store {
        id: UID,
        text: string::String,
    }

    /// Mint one Greeting and send it to the sender.
    public entry fun mint(ctx: &mut TxContext) {
        let g = Greeting {
            id: object::new(ctx),
            text: string::utf8(b"👋 Hello, Move!"),
        };
        transfer::public_transfer(g, tx_context::sender(ctx));
    }
}
```

*Build again →* `sui move build`

*Publish* (devnet):

```bash
sui client publish --gas-budget 100000000 ./hello_world
```

Copy the **PackageID** from the output – you’ll need it to call `mint`.

---

<a name="5"></a>

### 5 · Move Language Basics

#### 5.1 Variables & Constants

```move
let mut counter: u64 = 0;
const EOverflow: u64 = 1;
```

#### 5.2 Types

* Unsigned integers `u8…256`
* `bool`
* `address`
* `vector<T>`
* `String`, `Option<T>`, `TypeName`
* Your own `struct` / `enum`

#### 5.3 Expressions & Scope

```move
let area = {                 // block returns last expr
    let w = 3;
    let h = 4;
    w * h
}; // area == 12
```

#### 5.4 Control Flow

```move
if (n == 0) 1 else n * factorial(n - 1);

loop {
    if (done) break;
}

assert!(balance > 0, EInsufficient);
```

#### 5.5 Functions

```move
public fun add<T: copy>(a: T, b: T): T { a + b }
public(package) fun helper() { /* intra‑pkg only */ }
entry fun do_something(ctx: &mut TxContext) { /* tx entry‑point */ }
```

Multiple returns:

```move
fun split(x: u64): (u32, u32) { (x as u32, (x >> 32) as u32) }
```

#### 5.6 Testing

```move
#[test]
fun it_adds() { assert!(hello_world::add(1, 2) == 3) }

#[test, expected_failure(abort_code = EOverflow)]
fun it_fails() { abort EOverflow }
```

Run with `sui move test` – the CLI spins an isolated VM, no real chain required.

---

<a name="6"></a>

### 6 · Objects, UID & Abilities

* `has key` → object can live globally (needs a `UID` field)
* `has store` → object may be nested inside another `key` object
* `has copy` / `drop` → move semantics vs duplication/destruction

```move
struct Vault has key, store {
    id: UID,
    coins: vector<Coin<SUI>>,
}
```

Borrow instead of move:

```move
public fun deposit(v: &mut Vault, c: Coin<SUI>) { v.coins.push_back(c) }
```

---

<a name="7"></a>

### 7 · Transactions in Practice

1. *Create variables inside the PTB* (`--assign`)
2. *Call* functions (`--move-call`)
3. *Transfer* results (`--transfer-objects`)

```bash
sui client ptb \
  --gas-budget 100000000 \
  --assign sender @$MY_ADDR \
  --move-call $PKG::hello_world::mint \
  --assign greet \
  --transfer-objects "[greet]" sender
```

The entire command is one atomic tx block – either all succeed or all revert.

---

<a name="8"></a>

### 8 · Mini Project – **“Sui Recipe Book”**

A tiny dApp that lets any user publish a `Recipe` object, edit it later, list ingredients, and finally burn it when obsolete.
It demonstrates: packages, manifest, address aliases, objects with key/store, transactions, references, abilities, tests.

#### 8.1 Folder layout

```
recipe_book/
 ├─ Move.toml
 ├─ sources/
 │   └─ recipe_book.move
 ├─ tests/
 │   └─ recipe_tests.move
```

#### 8.2 Move.toml (abridged)

```toml
[package]
name = "recipe_book"
version = "0.1.0"

[addresses]
recipe_book = "0x0"
sui          = "0x2"
```

#### 8.3 Module code (sources/recipe\_book.move)

```move
module recipe_book::recipe_book {

    use std::{string, vector};
    use sui::object::{Self, UID};
    use sui::tx_context::{Self, TxContext};
    use sui::transfer;

    /// A public, user‑ownable recipe
    struct Recipe has key, store {
        id: UID,
        title: string::String,
        ingredients: vector<string::String>,
        steps: vector<string::String>,
    }

    /// Mint an empty recipe owned by the caller
    public entry fun mint(title_bytes: vector<u8>, ctx: &mut TxContext) {
        let r = Recipe {
            id: object::new(ctx),
            title: string::utf8(title_bytes),
            ingredients: vector::empty<string::String>(),
            steps: vector::empty<string::String>(),
        };
        transfer::public_transfer(r, tx_context::sender(ctx));
    }

    /// Add an ingredient (mutable ref demo)
    public fun add_ingredient(r: &mut Recipe, bytes: vector<u8>) {
        r.ingredients.push_back(string::utf8(bytes))
    }

    /// Add a cooking step
    public fun add_step(r: &mut Recipe, bytes: vector<u8>) {
        r.steps.push_back(string::utf8(bytes))
    }

    /// Delete recipe forever (drop demo)
    public entry fun burn(r: Recipe) {
        let Recipe { id, .. } = r;
        id.delete()
    }
}
```

#### 8.4 Unit tests (tests/recipe\_tests.move)

```move
#[test]
fun happy_path() {
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

#### 8.5 Build & Publish

```bash
sui move build
sui client publish --gas-budget 100000000 ./recipe_book
```

#### 8.6 Interact

```bash
export PKG=0x...          # your PackageID
export MYADDR=$(sui client active-address)

# mint new recipe
sui client ptb \
  --gas-budget 1e8 \
  --assign me @$MYADDR \
  --move-call $PKG::recipe_book::mint b"Pancakes" \
  --assign rec \
  --transfer-objects "[rec]" me
```

You can now mutate `rec` with `add_ingredient` and `add_step` in further PTBs, and finally `burn` it.

---

<a name="A"></a>

### Appendix A – CLI crib‑sheet

```bash
# build only (no deps fetch)
sui move build --skip-fetch

# run ONE test
sui move test test_single_thing

# decode an object to JSON
sui client object 0xOBJ --json

# view module ABI
sui move view-abi --module recipe_book
```

---

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

---

## 🎉 You’re ready for Week 3!

You now know:

* how to create & lay out Move packages;
* how manifests, addresses and objects fit together;
* the Move grammar (vars, types, flow, functions, testing);
* abilities & global‑storage rules;
* CLI workflow from build ➜ publish ➜ transact.

Clone the **recipe\_book** repo, tweak it, and share your transaction digests on Discord. Happy hacking 🚀
