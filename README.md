

*What it is → Why it matters → Syntax & rules → Common pitfalls → Mini exercise.*

---

# Week 2 Deep‑Dive — Core Concepts & Move Basics

---

## 1. Move Packages

### 1.1 What is a package?

* A **bundle** of one or more Move modules that will be published together under **one on‑chain address**.
* Comparable to an NPM package (JavaScript) or a Cargo crate (Rust).

### 1.2 Why do we need it?

* Atomic upgrades – you always upgrade **all** modules in lock‑step.
* Deterministic dependency graph – any package can import another **by address**; no global name collisions.

### 1.3 Baseline structure

```
my_pkg/
├─ Move.toml         # manifest
├─ sources/          # modules that end up on‑chain
│  └─ *.move
├─ tests/            # Move tests (compiled **only** in test mode)
└─ examples/         # usage demos; also test‑only
```

*Anything outside `sources/` never appears on chain.*

### 1.4 Mini exercise

1. `sui move new zoo`
2. `tree zoo -L 2` – identify the three default folders.
3. Delete *tests/* and run `sui move test` → see the failure message.

---

## 2. Move.toml (The Manifest)

### 2.1 Key sections

| Section              | What you put here                            | Sneaky gotcha                                 |
| -------------------- | -------------------------------------------- | --------------------------------------------- |
| `[package]`          | name, version, `edition = "2024"`            | Edition **must** be 2024 today.               |
| `[addresses]`        | human‑friendly aliases for 32‑byte addresses | Each alias is global to the package.          |
| `[dependencies]`     | git or local path to other packages          | Std/Sui deps auto‑injected since CLI 1.45.    |
| `[dev-dependencies]` | overrides only in test / dev mode            | Cannot add *new* aliases here, only override. |

### 2.2 Two TOML syntaxes

```toml
[dependencies]
DeepBook = { git = "...", subdir = "packages/deepbook", rev = "mainnet" }

[dependencies.DeepBook]  # ← identical, multiline style
git     = "..."
subdir  = "packages/deepbook"
rev     = "mainnet"
```

Pick whichever you find readable; the compiler doesn’t care.

---

## 3. Building & Testing

| Command                 | Mode     | What it does                                                                       |
| ----------------------- | -------- | ---------------------------------------------------------------------------------- |
| `sui move build`        | **prod** | Compiles `sources/` only.                                                          |
| `sui move test`         | **test** | Compiles `sources/ + tests/ + examples/` and runs every function marked `#[test]`. |
| `sui move build --test` | **dev**  | Same compile scope as test mode, but no test execution. Handy for IDEs.            |

> **Tip:** If the CLI starts re‑downloading git dependencies every time you build, pin `rev = "<commit‑hash>"` instead of a branch name.

---

## 4. Modules

### 4.1 Syntax variants

```move
// label syntax (recommended)
module zoo::keeper;

// block syntax (legacy, still valid)
module zoo::keeper {
    /* body */
}
```

### 4.2 Visibility matrix

| Keyword           | Callable from…               | Usual use‑case                   |
| ----------------- | ---------------------------- | -------------------------------- |
| *none*            | same module                  | helpers / invariants             |
| `public`          | anywhere                     | library API                      |
| `public(package)` | any module in *same* package | cross‑module helpers             |
| `entry`           | **transaction blocks only**  | mint / burn / transfer functions |

> All struct **fields are always private**; you expose them via public functions or methods.

---

## 5. Primitive Types

| Category      | Types                            | Notes                                |
| ------------- | -------------------------------- | ------------------------------------ |
| Booleans      | `bool`                           | literals `true / false`              |
| Unsigned ints | `u8 u16 … u256`                  | arithmetic aborts on overflow        |
| Address       | `address`                        | literal `@0x...` or alias `@sui`     |
| Vectors       | `vector<T>`                      | generic, variable length             |
| Strings       | `String` (UTF‑8) / `AsciiString` | library wrappers around `vector<u8>` |

Literals cheat‑sheet:

```move
0xFFu8      // int with suffix
0xFF        // hex int, defaults to u64
b"bytes"    // byte‑vector literal
x"0A"       // single byte
```

---

## 6. Structs & Abilities

```move
public struct Animal has key, store {
    id: UID,
    species: vector<u8>,
}
```

| Ability | Without it…                         | With it…                    |
| ------- | ----------------------------------- | --------------------------- |
| `key`   | cannot exist on‑chain               | becomes an object           |
| `store` | cannot be stored inside `key`       | can live inside objects     |
| `copy`  | value **moves** on assignment       | can duplicate (`let b = a`) |
| `drop`  | must unpack / use before scope ends | can ignore (`let _ = v`)    |

*Hot‑potato pattern* – a struct **without** `copy` or `drop` must be handed off or destructured: the compiler enforces correct asset flow.

---

## 7. Constants & Errors

```move
const INITIAL_FEED:u64 = 100;
#[error] const ENotKeeper: vector<u8> = b"caller is not a keeper";
```

Use in assertions:

```move
assert!(caller == keeper, ENotKeeper);
```

> Error constants are just normal `const`s; the `#[error]` attribute lets you store them as readable *strings* instead of numbers.

---

## 8. Comments

| Style | Example   | Use it for                                   |
| ----- | --------- | -------------------------------------------- |
| Line  | `// …`    | quick notes                                  |
| Block | `/* … */` | large sections / comment‑out                 |
| Doc   | `/// …`   | markdown docs → `move doctor` generates HTML |

---

## 9. Control‑Flow Essentials

```move
// if‑else is an expression
let sign = if (n > 0) 1 else -1;

// while returns ()
while (x < 10) { x = x + 1; };

// loop + break/continue
loop {
    if (done) break;
    if (skip) { continue };
}

// return early
if (!ok) return error_code;
```

---

## 10. References & Ownership

| Pass style | Example              | Result                                |
| ---------- | -------------------- | ------------------------------------- |
| **Move**   | `fun take(v:T)`      | caller loses access                   |
| **& read** | `fun view(v:&T)`     | read‑only borrow                      |
| **\&mut**  | `fun edit(v:&mut T)` | mutable borrow; still owned by caller |

Rules:

1. Only one active mutable reference per value.
2. Cannot move a value while a reference (any kind) is alive.
3. The borrow ends when the reference goes out of scope.

---

## 11. Generics

```move
struct Box<T> has drop { val:T }

public fun wrap<T>(v:T): Box<T> { Box { val:v } }

struct Coin<phantom C> has drop { amount:u64 }   // phantom type tag
```

**Constraints**

```move
struct CopyVec<T: copy + drop> has drop {
    data: vector<T>
}
```

---

## 12. Enums & `match`

```move
public enum FeedResult has copy, drop {
    Ok,
    OutOfFood { missing:u64 },
}

fun feed(n:u64): FeedResult { /* … */ }

match (feed(5)) {
    FeedResult::Ok => { /* happy path */ },
    FeedResult::OutOfFood { missing } => abort missing,
}
```

*Every possible variant must be handled; add `_ => …` as catch‑all.*

---

## 13. Transactions in Depth

### 13.1 Anatomy

```
Sender  — address who signs
Gas     — Coin<SUI> object paying fees
Inputs  — pure values & object refs
Commands
  0: MoveCall(...)
  1: SplitCoins(...)
  2: TransferObjects(...)
```

### 13.2 Building with PTB

```bash
sui client ptb \
  --gas-budget 1e8 \
  --move-call $PKG::zoo::register @$ANIMAL "'Panda'" \
  --transfer-objects "[result_0]" $MY_ADDR
```

*Command outputs (`result_i`) can feed later commands in the same block – that’s the “programmable” part.*

---

## 14. Testing Toolkit

```move
#[test]
fun happy_path() { /* should pass */ }

#[test, expected_failure(abort_code = 0)]
fun must_fail() { abort 0 }

#[test_only]
public fun helper(): u64 { 42 }
```

Run subsets:

```bash
sui move test animal   # runs tests whose names contain “animal”
```

---

## 15. Publishing & Localnet Workflow

```bash
# 1. spin up a local chain
RUST_LOG="off,sui_node=info" sui start --with-faucet --force-regenesis

# 2. fund your account
sui client faucet

# 3. build & publish
cd zoo
sui client publish --gas-budget 2e8      # note the PackageID

# 4. interact
export PKG=0x...           # package address
sui client ptb ...         # send transactions
```

| Stage      | Common pitfall                                 |
| ---------- | ---------------------------------------------- |
| Start node | Port 9000 already in use.                      |
| Faucet     | Forgot to set env to localnet in `sui client`. |
| Publish    | Gas budget too low: raise to 0.2 SUI in dev.   |

---

## 16. End‑to‑End Demo Project

> *Full repo: `github.com/<you>/library_plus` (≈100 LOC).*

### 16.1 Features covered

* **Package** with manifest & dependency on Sui framework.
* **Modules**: `library`, `admin`, `tests`.
* **Structs with key/store**, **enums**, **generic Box\<T> helper**.
* **Abilities**: `Book` (drop), `Library` (key, store).
* **Entry functions** (`admin::bootstrap`) invoked via PTB.
* **Unit tests** using `#[test]` and helpers using `#[test_only]`.

### 16.2 Deploy script

```bash
sui client publish --gas-budget 2e8 --json \
 | jq -r '.objectChanges[] | select(.objectType|test("::package::UpgradeCap")) .objectId' \
 | xargs -I {} echo "UpgradeCap at {} (store it safely)"
```

That’s it—you now own a mutable on‑chain library where you can donate books and query inventory!

---

## 17. Quick Troubleshooting Table

| Symptom                                      | Root cause                              | Fix                                                 |
| -------------------------------------------- | --------------------------------------- | --------------------------------------------------- |
| **`Unused value of type …`**                 | Type lacks `drop`; you ignored it.      | Destructure or explicitly discard.                  |
| `Instance of … does not have ability 'copy'` | You tried to duplicate a non‑copy type. | Move it instead, or add `copy`.                     |
| `Borrow check failed`                        | Overlapping `&mut` and `&` borrows.     | Refactor to shorter borrow scopes.                  |
| `Gas budget too low`                         | publish / tx exceeded limit.            | Increase `--gas-budget` (in **MIST**, 1 SUI = 1e9). |

---

## ☑️ Practice Checklist

1. **Create** a package called **bank**.
2. Add a **Coin<phantom T>** struct with `drop` ability.
3. Implement `mint`, `transfer`, `balance_of` entry functions.
4. Write **three unit tests**: happy path, insufficient balance (expected failure), and multi‑transfer.
5. Publish to localnet; send a PTB making two transfers in one transaction.

Complete those five steps and you’ve cemented <u>every</u> Week‑2 concept!

---

> **Next week:** object upgrades, events, capability pattern, and cross‑package interfaces. Stay tuned 🚀
