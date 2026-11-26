# bylaw-lang specification (draft)

Status: draft before milestone M0. `bylaw-lang` is a working name.

Fill each section. The quoted line under each heading says what to write
there. Delete the quoted line when the section is complete.

## 1. Purpose

bylaw-lang is a language for the transfer rules of one tokenized asset,
in the shape of an ERC-3643 compliance module. One program states the
rules of one asset: holder caps, class caps, country rules, lock-ups and
identity claims as ZK credentials. An offline threshold-authorized
self-amending system governs each program (`docs/PLAN.md` section 4).

The compiler `langc` checks a program. It refuses each program that can
cause a run-time error (trap) (R1, `docs/PLAN.md` section 2). It writes
a Wasm MVP module (T1, T2). A later slice adds an ES module and a JSON
result document (T7). The host is TinyCC (`PIN`).

Its type formers are F1 to F15 of `formers/FORMERS.md`. Its core data
types and core operations are only the types and operations of
`design/DESIGN.md` (the design): the base types and operations of
restrict-lang, with the governance of `docs/PLAN.md` section 4 in place
of the DAO.

## 2. Programs

A program is one file: a sequence of definitions with `--` comments
(`docs/host/README.md`, the program form):

```
def NAME : TYPE := TERM
```

The compiler refuses these host forms in a program (`docs/host/README.md`,
the rule table; `src/front/check.c`, the program walk):

| Rule | Form | Code |
|---|---|---|
| R1 | a `family` declaration | `REFUSE_DATA` |
| R2 | an `axiom` declaration | `REFUSE_AXIOM` |
| R3 | `def rec`, or a definition that uses itself or a later definition | `REFUSE_REC` |
| R4 | a core name, a domain name, a reserved name (`Bool`, `Text`) or a repeated name | `REFUSE_NAME` |

The lexer also refuses `"` (the language has no text type) and a Nat
literal above 2^64 - 1. Thus a program cannot add a data type, an unproved
fact or general recursion. Recursion comes only from `fold` and `unfold`
(F6, F7). The compiler writes the target; a program cannot.

The domain file `domain/domain.lang` gives the fixed first definitions.
Today it holds the kit sample. The S1 units replace it:

- `Empty` (`family Empty := emptyLoop (emptyOf : Empty)`, as in
  restrict-lang `domain/domain.lang`). `R.Dec` needs it (U4).
- `Env`, `Out` and the mirror `State` (U3).
- The trimmed `Charter` (U8): `Size`, `RuleSet` (`rs0`, `rs1`), `ruleOf`,
  `Charter` (`genesis`, `adopt`) and `active`. The restrict-lang `Charter`
  is a higher inductive type with a `Ratified` family; S1 drops both.

`U256`, `Addr` and `Map K V` are core types in C (U1, U2), not families.
The limits of restrict-lang (`capN`, `capCN`, `lockD`, `regD`) are words
of the state, not definitions (restrict-lang `SPEC.md` section 6). The
constitution record of `docs/PLAN.md` section 4 comes in S5.

## 3. Type formers

The type formers are F1 to F15 of `formers/FORMERS.md`. This language uses
the tcc-wasm column of the realization matrix. `formers/tcc-wasm.md` gives
the host form of each former.

| ID | Status on tcc-wasm | Effect on this language | Open item |
|---|---|---|---|
| F15 | PARTIAL: only the domain file declares an indexed family; a constructor takes the indices first; no field type refers to an index; the index erases on Wasm | The `rules` form (U4) and the trimmed `Charter` (U8) use families without indices, so S1 needs no more | none |
| F6 | DONE with a host limit: a fold over a run-time tree refuses with `REFUSE_RUNTIME_TREE` | `active` folds a charter that is known at compile time | none |
| F9 | DONE with a host limit: a function value at run time refuses with `REFUSE_RUNTIME_CLOSURE` | The entries are first order | none |

One deviation from R1: `formers/tcc-wasm.md` keeps the trap flags of
erased proofs. R1 removes them (`docs/PLAN.md` section 2, T-2; O1; slice
S3).

## 4. Structures

- **Monad**: `pure`, `map`, `bind` on `Option`, `Sum E` and `List`.
- **Algebra**: `fold` on `Nat`, `List` and the domain families. `unfold`
  into `List`, with a `Nat` step limit (three arguments).
- **Filterable**: `filter` on `Option` and `List`. `filter` on `Sum`
  refuses with `TYPE_MISMATCH`.

`Map K V` has no `fold`, no size and no key list (as in restrict-lang). The
design does not use the other carriers of restrict-lang (`FreeMon`, the
sealed `Enc`, the cofamily `Stream`, `Size`, `Bag`, `ChoiceBag`, `Pos`,
`Word`), so the host does not give them.

## 5. Core types

The types of restrict-lang `SPEC.md` section 5 and its host table that
`docs/PLAN.md` section 3 keeps.

| Type | Meaning (design section) | Definition |
|---|---|---|
| `U256` | A 256-bit unsigned word: balances, counts, class numbers, scalars (base types) | Core type in C. Literal `Nu` (for example `1000u`); a literal above 2^256 - 1 refuses. Four i64 limbs in eval and on Wasm (O8). Probe-forced: Wasm MVP has no 256-bit type. |
| `Addr` | A 160-bit account address (base types) | Core type in C. Literal `0x` and 40 hex digits. Only `addrEq` compares two addresses. |
| `Map K V` | A finite map: balances, lock ends, classes, credential slots (base types) | Core type in C. A missing key gives the zero of `V`. A set to the zero of `V` removes the key. No delete, fold, size or key list. Print form `mapOf [(k, v), ...]`. |
| `Move` | One move of the token: transfer, mint or burn (base types) | `Sum (Prod Addr (Prod Addr U256)) (Sum (Prod Addr U256) (Prod Addr U256))`, with the helpers `transfer f t x`, `mint t x` and `burn f x`. |
| `State` | The mirror state of the asset (base types) | The `state` form (U3). The fields of restrict-lang: `bal`, `holders`, `holdersIn`, `lockEnd`, `bound`; `used`, `clsOf`, `expires`, `ringX`, `ringY`; `pCapN`, `pCapCN`, `pLockD`, `pRegD`, `issuers`, `passCls`. `decided` drops (DAO). `bound` is a constant of the initial state, because `bindToken` is an amendment (`docs/PLAN.md` section 5). |
| `Env` | The call context (base types) | `family Env := makeEnv (now : Nat) (block : U256) (caller : Addr) (self : Addr)` |
| `Out` | A record that an entry writes (base types) | The `event` form gives the constructor `emit (emitTag : Nat) (emitFields : List U256)`. The `pay` and `pull` records of restrict-lang move EVM tokens, and `docs/PLAN.md` section 3 does not keep them (reading). |
| `Empty` | The type with no closed term (rules) | `family Empty := emptyLoop (emptyOf : Empty)` in the domain file. |
| `R.Dec P` | A decided rule (rules) | `Sum P (P -> Empty)`. Probe-forced (restrict-lang). |
| `R` | The conjunction of the rules of one `rules` declaration (rules) | `rules R S M T := r1 \| ... \| rk` gives `R : S -> Env -> M -> Type 0`, the right-nested `Prod` of `R.rule.ri.holds`. |
| `G1` | A point of the BN254 group G1 (credentials) | `Prod U256 U256`. A ring is `List (Prod U256 U256)`; a signature is `List U256`. |
| Entry result | The result of an entry (base types) | `Option (Prod State (List Out))` in S1 (U3). S2 replaces it with the entry sum `Ok`, `Refused`, `Underfunded` (`docs/PLAN.md` section 2). |

## 6. Core operations

The operations of restrict-lang `SPEC.md` section 6 and its host table
that `docs/PLAN.md` section 3 keeps. "Trap in restrict-lang" marks an
operation that S2 changes: the checker discharges a no-overflow or bound
obligation, or it refuses (O1).

| Operation | Type | Meaning (design section) |
|---|---|---|
| `u256Add` | `U256 -> U256 -> U256` | The sum. Trap in restrict-lang past 2^256 - 1. |
| `u256Sub` | `U256 -> U256 -> U256` | The difference; it stops at 0. |
| `u256Eq`, `u256Le` | `U256 -> U256 -> Flag` | Equal; less than or equal. |
| `toU256` | `Nat -> U256` | The only conversion into `U256`. |
| `toNat` | `U256 -> Nat` | Trap in restrict-lang above 2^64 - 1. |
| `addrEq` | `Addr -> Addr -> Flag` | Equal addresses. |
| `addrToU256` | `Addr -> U256` | The address as the low 160 bits. |
| `mapEmpty`, `mapGet`, `mapSet` | `Map K V`; `Map K V -> K -> V`; `Map K V -> K -> V -> Map K V` | The keyed map. `mapOf` is the print form. |
| `hash` | `U256 -> U256 -> U256` | Keccak-256 of the 64 bytes of two words, in C (U5). |
| `ecAdd`, `ecMul` | `G1 -> G1 -> G1`; `G1 -> U256 -> G1` | The sum of two points of BN254 G1; k times a point. In C (U6). `(0, 0)` is the point at infinity. Trap in restrict-lang: a coordinate at or above p, or a point that is not on the curve. |
| `zkKey`, `zkHpAt`, `zkSignAt`, `zkVerifyAt` | `Nat -> U256`; `G1 -> U256 -> G1`; `List G1 -> U256 -> U256 -> U256 -> List U256`; `List G1 -> U256 -> U256 -> List U256 -> Flag` | bLSAG ring signatures over BN254 G1, in C (U7, front/blsag.c): the test key i, the hash to point at a context word w, the signature `c_1, s_1 .. s_n, I.x, I.y` of ring member x on (w, m), and its check. Trap: an empty ring or a ring above 28 members, a bad point, a signer key that is zero, at or above r or not in the ring. An open argument keeps the call stuck. |
| `step` | `State -> Env -> Move -> State` | The effect of a hook. `holders` changes only when a balance crosses zero. A mint moves the lock end of the receiver. |
| `transferred`, `created`, `destroyed` | entries | Run `step`. `transferred` records `Broken` when `canTransfer` fails and still steps (a forced transfer). The revert arm of restrict-lang becomes `Refused` in S2. |
| `holderCap`, `classCap`, `countryIn`, `lockUp` | `State -> Env -> Move -> Flag` | The rule tests. A cap decides on the state after the move. |
| `canTransfer` | `(env : Env) -> (s : State) -> (f t : Addr) -> (x : U256) -> Flag` | The first bit of `R.dec`. Mint and burn are not checked. |
| `elim` | `elim P c u` on `Unit`; `elim P l r s` on `Sum A B` | The dependent eliminator (U4a, Q-S1-10 (a)): the motive `P` may depend on the scrutinee. `either` is the non-dependent form. Lowering refuses an `elim` on a run-time value with `REFUSE_ELIM` (HOST-LIMIT). |
| `rules R S M T := r1 \| ... \| rk` | declaration | The rules form (U4b). Each `ri` is a test `S -> Env -> M -> Flag`. It gives `R.rule.ri.holds`, `R.rule.ri.dec`, `R`, `R.dec`, `R.bit` (the `flagIf` of the tests), the helpers `R.B`, `R.yes`, `R.lift`, `R.Holds`, `R.Dec`, `R.Code`, `R.no`, `R.decide`, `R.both`, `R.bitOf`, and `canTransfer`. A rule that is not a Flag test fails in `R.rule.ri.holds` with `TYPE_MISMATCH`. A name may contain dots. |
| `cls`, `inSet` | `U256 -> U256`; `State -> U256 -> Flag` | The class of a key; membership in `passCls`. |
| `zkKey` | `Nat -> U256` | The test key `hash(tag, i) mod r`. |
| `zkHpAt` | `G1 -> U256 -> G1` | Hash to the curve with the context word. |
| `zkSignAt`, `zkVerifyAt` | `zkSignAt ring x w m : List U256`; `zkVerifyAt ring w m sig : Flag` | The bLSAG ring signature with context `w`. Trap in restrict-lang: `zkSignAt` needs 0 < x < r and xG in the ring. |
| `issue`, `register` | entries | `issue` writes a ring slot and records `Issued`. `register` checks a credential with `zkVerifyAt` and sets `used`, `clsOf` and `expires` (`now + regD`: trap in restrict-lang). |
| `isVerified` | `(env : Env) -> (s : State) -> (w : Addr) -> Flag` | `now < expires s w`. |
| `setCapN`, `setCapCN`, `setLockD`, `setRegD`, `setIssuer`, `setPass` | `State -> ... -> State` | Each changes one word of the state. In S5 an amendment applies them. `withGov` drops (DAO). |
| `active` | `Charter -> RuleSet` | The fold of the charter history (U8, trimmed). |
| `natAdd`, `natMul` | `Nat -> Nat -> Nat` | Host operations. They trap on u64 overflow today (`docs/PLAN.md` section 2, T-1). |

Dropped (`docs/PLAN.md` section 3): `bindToken`, `unbindToken`,
`isTokenBound`, `getTokenBound`, `propose`, `enact`, the ballot context,
`yn`, `passCon`, `regimeOf`, the kit CALL form, the ABI selectors and the
precompile calls. Eval computes the curve operations in C (U6).

## 7. Target and instance encoding

S1 does not change the target. The tcc-wasm module today
(`docs/host/README.md`, entries and layout):

- An entry is a top-level definition of type `Nat` or `Flag`, or a first
  order function from `Nat` and `Flag` arguments to `Nat` or `Flag`. It is
  exported by its name. Each parameter and the result is an i64.
- Compilation is partial evaluation of each entry body. Types and
  universes erase. Proofs erase, but their trap flags stay until S3.
- One memory (at most 16384 pages) and one heap pointer global. A heap
  block is a tag, then a word and a flag for each field.

An instance is a top-level definition whose type is a data type (F14).

| Type | Encoding |
|---|---|
| `Nat` | u64 in an i64 |
| `Flag` | 0 or 1 in an i64 |
| A data type | a heap block: the tag, then the fields |
| `U256` | four i64 limbs (O8, slice S3) |
| `Addr`, `Map K V` | slice S3 (open) |
| Entry result | the entry sum of `docs/PLAN.md` section 2 (slices S2, S4) |

Until S3, lowering refuses a run-time word with `REFUSE_WORD`, a run-time
map with `REFUSE_MAP` and an `elim` on a run-time value with
`REFUSE_ELIM` (HOST-LIMIT).

The exports of `docs/PLAN.md` section 5 come in S5 and S6.

## 8. Host and target

The host is TinyCC. `PIN`: tcc 0.9.28rc 2026-09-04 mob@0fb54300 (AArch64
Darwin), node v23.10.0, wasm-validate and wasm-objdump 1.0.39. The `PIN`
of relay-lang has two more lines (sqlite3 and the `-lsqlite3` link) from
line 6; they do not apply here, because bylaw-lang links no library.

The facts of `probe/CAPABILITY.md` that set the design: a program is one
file; `natAdd` and `natMul` trap on u64 overflow; the evaluator prints
`trap`; the module grows its memory and can trap at `unreachable`. These
facts are the trap sites of `docs/PLAN.md` section 2.

The checks on the output (`test/gate.sh`): each module passes
`wasm-validate` and `wasm-objdump -x`; the differential grid compares the
Wasm result with `langc eval` on each case and must give 0 mismatches.
There is no axiom report, because a program cannot declare an axiom (R2).
S3 adds the opcode scan of `docs/PLAN.md` section 2.

## 9. Open items

Each item is a question of `docs/PLAN.md` section 8. Section 1 of the
plan has the USER answers verbatim.

- O1. The arithmetic form. RULED 2026-11-23 (USER): one form for each
  operation; the checker discharges a no-overflow obligation, or the
  program proves it; the fix hint names the missing bound (Q-S1-1 (a)).
  Slice S2.
- O2. The cost unit of the bound. RULED 2026-11-23 (USER): one unit for
  each Wasm instruction executed (Q-S1-2 (a)). Slice S4.
- O3. The floor on t. RULED 2026-11-23 (USER): t > n / 2 (Q-S1-3 (a)).
  Slice S5.
- O4. What `amend` can change. RULED 2026-11-23 (USER): the rule sets
  and parameters in the artifact; new code needs a new artifact whose
  chain extends the stored epoch (Q-S1-4 (a)). Slice S5.
- O5. Key generation and signing. RULED 2026-11-23 (USER): `langc`
  subcommands in C, tested with the RFC 9591 vectors (Q-S1-5 (a)). Slice
  S5.
- O6. The x402 endpoint. RULED 2026-11-23 (USER): a native `langc serve`
  that TinyCC builds (Q-S1-6 (a)). The prices, the payee address and the
  facilitator: pending USER data. Slice S9.
- O7. The funding files. RULED 2026-11-23 (USER): the account-wide
  `MavenRain/.github/FUNDING.yml` applies; the repo has no `funding.json`
  (Q-S1-7 (a)). The entity, the contact and the channels: pending USER
  data. Slice S9.
- O8. `U256` on Wasm. RULED 2026-11-23 (USER): four i64 limbs that the
  kit lowers (Q-S1-8 (a)). Slices S1 (eval) and S3 (lowering).
- O9. The credential curve. RULED 2026-11-23 (USER): the BN254 G1 bLSAG
  of restrict-lang, in C (Q-S1-9 (a)). Slice S6.

## 10. Milestones

The slices S1 to S9 of `docs/PLAN.md` section 7 are the milestones.

Status 2026-11-23: S1 is open. The USER ruled O1 to O9. The port of the
base signature to the front end has not started.
