# bylaw-lang plan

This file holds the rulings, the reading of the no-trap rule on the
tcc-wasm kit, the base signature, the governance design, the export
surface, the agent surface, the slices and the open questions.
`design/DESIGN.md` is the only source of the rulings. S0 is a hand build
(USER ruling 2026-11-23).

## 1. Rulings

- R1 (no trap). "nothing that can cause a runtime error (trap) should be
  allowed to compile, period." (DESIGN.md, the request.)
- R2 (type formers). Cell monads and their duals only, as represented in a
  cubical type theory. This is the same ruling as restrict-lang R1, so the
  restrict-lang reading carries over (restrict-lang `docs/PLAN.md` section
  2).
- R3 (base types). The base types and operations of restrict-lang
  (DESIGN.md, "The base types and operations"), with one change: an
  offline threshold-authorized self-amending system governs each program.
  The DAO of orbit-lang does not.
- R4 (host). TinyCC. PIN: tcc 0.9.28rc 2026-09-04 mob@0fb54300.
- T1. Targets: a Wasm MVP module and an ES module + JSON. No EVM.
- T2. The Wasm MVP module comes first. The ES module + JSON target is a
  later slice.
- T3. Exhaustion: the checker proves a worst-case cost for each entry. The
  entry compares the supplied budget with the bound before it does work.
  It returns a typed `Underfunded` result. The bound is in the ABI and in
  the OpenAPI description.
- T4. Governance: each artifact carries a verifier. langc refuses a source
  whose amendment chain does not have valid t-of-n signatures. The scheme
  is FROST Schnorr over secp256k1.
- T5. The Wasm host is neutral: the tcc-wasm ABI plus one import,
  `budget_left`, that the embedder supplies.
- T6. The module keeps the ERC-3643 shape as Wasm exports.
- T7. The later target is an ES module plus the JSON result document, in
  the style of the tcc-js kit.
- T8. The name is bylaw-lang.
- A1. The agent criteria of DESIGN.md (the request, "Finally ...") are
  deliverables. Section 6 places each one.

USER answers to the S1 questions of section 8 (AskUserQuestion,
2026-11-23, verbatim option labels):

- Q-S1-1. "One form + obligation (Recommended)". (a): one form for each
  operation; the checker discharges a no-overflow obligation, or the
  program proves it; the fix hint names the missing bound.
- Q-S1-2. "Wasm instructions (Recommended)". (a): one unit for each Wasm
  instruction executed (the wasmtime fuel model).
- Q-S1-3. "t > n / 2 (Recommended)". (a): the checker requires a strict
  majority.
- Q-S1-4. "Rule sets + params (Recommended)". (a): `amend` moves among
  the rule sets and parameters in the artifact; new code needs a new
  artifact whose chain extends the stored epoch.
- Q-S1-5. "langc subcommands in C (Recommended)". (a): offline key
  generation, signing and aggregation in C, tested with the RFC 9591
  vectors.
- Q-S1-6. "Native langc serve (Recommended)". (a): a native `langc serve`
  that TinyCC builds. Prices, payee address and facilitator: pending USER
  data.
- Q-S1-7. "Account-wide FUNDING.yml (Recommended)". (a): until the USER
  gives the entity, the contact and the channels, the account-wide
  `MavenRain/.github/FUNDING.yml` applies and the repo has no
  `funding.json`. Entity, contact and channels: pending USER data.
- Q-S1-8. "Four i64 limbs (Recommended)". (a): `U256` is four i64 limbs
  that the kit lowers.
- Q-S1-9. "BN254 bLSAG in C (Recommended)". (a): keep the BN254 G1 bLSAG
  of restrict-lang, in C.
- Q-S1-10 (RULED 2026-11-25). "(a) Port restrict elim (Recommended)": the
  generated `R.decide` of a `rules` form needs a dependent eliminator, and
  bylaw has only `either` and `flagIf`. Port the dependent `elim` of
  restrict-lang. U4 splits into U4a (`elim`) and U4b (the `rules` form,
  dotted names, `Empty`, `R.Dec`).
- S1 build tier. "Hand build (Recommended)": main-loop edits, as in S0.
  This ruling covers S1 only.

## 2. The reading of R1 on the tcc-wasm kit

The kit traps today. Each row is a current trap site and its planned
removal (reading, not ruled; Q-S1-1 and Q-S1-8 decide the forms).

| # | Trap site | Removal |
|---|---|---|
| T-1 | `natAdd` and `natMul` trap on u64 overflow (`docs/host/README.md:105`) | Each operation has one canonical form. The checker discharges a no-overflow obligation for each use, or it refuses with `REFUSE_TRAP` and a fix hint (Q-S1-1). |
| T-2 | Lazy trap flags: each value carries a word and a trap flag (`docs/host/README.md:120`, `src/front/lower.c`, `src/ir.h`) | When R1 holds at check time, no value can carry a set flag. The flag words go away from the IR and the heap layout. |
| T-3 | `IR_STMT_TRAP` lowers to `unreachable` (`src/wasm.c:269`) | The IR loses `IR_STMT_TRAP`. |
| T-4 | A failed `memory.grow` lowers to `unreachable` (`src/wasm.c:296`, `docs/host/README.md:146`) | The cost bound gives the largest heap. The module declares a fixed memory (minimum = maximum) and has no `memory.grow`. |
| T-5 | An entry traps when a `Flag` argument is larger than 1 (`src/wasm.c:303`, `docs/host/README.md:149`) | Argument decode is total. A bad argument gives a typed `Refused` result. |
| T-6 | A tag out of range in `br_table` traps (`docs/host/README.md:152`) | The same total decode. A tag that comes from inside the module is in range by construction. |
| T-7 | A filler `unreachable` ends each function body (`src/wasm.c:237`, `src/wasm.c:315`) | Typed blocks give each body a value on every path, so the filler is not necessary. |
| T-8 | `i64.div_u` in the `natMul` overflow test (`src/wasm.c:184`) | The divisor is never 0: a `select` puts 1 in its place. The opcode scan admits `i64.div_u` only in this guarded shape, or the overflow test changes (Q-S1-1). |
| T-9 | Stack depth is engine-defined | Folds lower to loops. The lowered call graph must be acyclic, so the depth has a bound. The bound goes into the cost record. |
| T-10 | The evaluator prints `trap` (`docs/host/README.md:87`, `src/front/eval.c`) | `eval` has no trap result. A test that expects `trap` becomes a check refusal or a typed result. |

The gate proves R1 on the target with an opcode scan. The scan walks the
code section of each module (not the raw bytes). It refuses `unreachable`
(0x00), `memory.grow` (0x40) and each other opcode that can trap, except
the guarded shape of T-8.

The entry result is a sum: `Ok v`, `Refused code` or `Underfunded need`
(reading, not ruled). The entry prologue calls `budget_left`, compares the
result with the bound constant of the entry and returns `Underfunded`
when the budget is too small.

## 3. The base signature from restrict-lang

The source is restrict-lang at a0f4784: `SPEC.md` sections 5 and 6, and
`examples/mirror.lang`, `examples/restrict.lang`,
`examples/restrictkit.lang` and `examples/charter.lang`.

- Kept: `Move`, the mirror state, `R.Dec`, the `rules` form, `Env`,
  `step`, the hooks `transferred`, `created` and `destroyed`, the rule
  tests `holderCap`, `classCap`, `countryIn` and `lockUp`, `canTransfer`,
  `isVerified`, the class map `cls` and `inSet`, the credential
  operations `zkHpAt`, `zkSignAt`, `zkVerifyAt`, `issue` and `register`,
  the setters and `active`.
- Changed: `toNat` traps above 2^64 - 1 in restrict-lang (SPEC.md section
  6). Under R1 it needs a bound obligation or a sum result (Q-S1-1).
  `U256` has no native Wasm type (Q-S1-8).
- Dropped: the DAO state, `propose`, `enact`, the ballot context, `yn`,
  `passCon` and `regimeOf` (section 4 replaces them). The EVM pieces drop
  too: the MODEXP precompile path of `zkHpAt`, the kit CALL form, `revert`
  arms and the ABI selectors.

## 4. Governance

- The constitution is a record: the epoch, t, n, the FROST group key, the
  member keys and the hash of the active rule set.
- An amendment is a record: the next epoch, the hash of the new
  constitution, and a Schnorr signature under the group key of the current
  constitution. Signers make it offline. No online vote occurs.
- Self-amendment: an amendment can change t, n, the members and the group
  key (after a new offline key generation).
- Compile time: langc reads the chain and checks each link: a valid
  signature under the previous group key, an epoch that grows by 1, and
  1 <= t <= n (Q-S1-3 asks about a floor). A failed link is a check
  refusal with a fix hint.
- On the target: the export `amend` takes a signed amendment, checks it
  under the stored group key, and moves to the new constitution (Q-S1-4
  sets what it can change).
- The verifier is plain C in the kit: secp256k1 point arithmetic and the
  Schnorr check of RFC 9591 (FROST(secp256k1, SHA-256)). Verification does
  not need the FROST signing rounds. Test vectors come from RFC 9591.

## 5. The ERC-3643 export surface on Wasm

The module exports `canTransfer`, `transferred`, `created`, `destroyed`,
`isVerified`, `issue`, `register` and `amend`. `bindToken` and
`unbindToken` become amendments. Each export returns the entry sum of
section 2. A hook does not stop a forced transfer: when a rule fails, it
records `Broken`, as restrict-lang does (SPEC.md section 6).

## 6. The agent surface

| Item | Place |
|---|---|
| One canonical form per operation, `langc fmt` | `src/fmt.c`; the gate checks that `fmt` is a fixed point on each example |
| `langc check --json` with fix hints and counts (parsed, typed, laws discharged) | `src/main.c` and `docs/CHECK-JSON.md` |
| `SKILL.md` with 3 to 5 graded examples | `SKILL.md` |
| `langc mcp`, an MCP server in C | `src/mcp.c` (stdio JSON-RPC) |
| `AGENTS.md` and the README table of when to use the language | `AGENTS.md`, `README.md` |
| Token-count comparison against Python and Rust | `docs/TOKENS.md` |
| MCP registry entry, OpenAPI description, the price in the 402 response | `docs/openapi.yaml`, `server.json` |
| `FUNDING.yml`, `funding.json`, `llms.txt`, `.well-known/ai-donate.json` | the repo root and `.github/` (Q-S1-7) |

## 7. Slices

| Slice | Deliverable | Gate row |
|---|---|---|
| S1 | Q-S1 rulings; `SPEC.md` and `probe/CAPABILITY.md` filled; the base signature ported to the tcc-wasm front end (check and eval) | the restrict-lang examples give the same verdicts in eval |
| S2 | R1 in the checker: `REFUSE_TRAP`, the arithmetic obligations, the entry sum, no `trap` in eval | each old trap test now refuses or gives a typed result |
| S3 | Wasm lowering with no trap: no `IR_STMT_TRAP`, fixed memory, typed blocks, wide integers | the opcode scan passes; the differential grid has 0 mismatches |
| S4 | The cost bound, `budget_left`, `Underfunded`, the bound in the ABI | the measured cost is at most the bound on each grid case |
| S5 | Governance: the chain check, the C verifier, `amend` | the RFC 9591 vectors pass; each bad chain refuses |
| S6 | The credentials on Wasm and the full ERC-3643 surface | the restrict-lang credential tests pass on Wasm |
| S7 | The ES module + JSON target | ES module and Wasm give the same results on the grid |
| S8 | `fmt`, `check --json`, `mcp`, `SKILL.md`, `AGENTS.md`, the README table, `docs/TOKENS.md` | `fmt` fixed point; MCP calls answer; the token table is reproducible |
| S9 | Discovery and funding files, the OpenAPI description, the x402 endpoint, the registry entry | the 402 response carries the price; the files validate |

### S1 units (hand build, USER ruling 2026-11-23)

The source is a read-only gap survey of 2026-11-23 (outside the repo:
`bylaw-lang-work/s1/gap-survey.md`). These are readings, not rulings:

- The restrict-lang verdicts are the check-ok runs of
  `examples/mirror.lang` and `examples/restrict.lang` (their `Path ...
  refl` rows) and the run scripts `test/run/mirror.script` and
  `test/run/restrict.script`. `examples/restrictkit.lang`,
  `test/restrict.sh` and `test/dao.sh` need the EVM or the DAO, so they
  drop. `examples/charter.lang` needs `yn`, `passCon`, `regimeOf` and a
  higher inductive type; a trimmed copy keeps only `active`.
- Trimmed copies go below `test/restrict/`, with
  `test/restrict/expect.txt`. They write `Eq` in place of `Path`.
  `test/gate.sh` gets a check-ok loop and an eval loop for that directory
  (copies of the loops for `examples/`). The copies stay out of
  `examples/`, because the IR and Wasm loops refuse them before S3.
- `bindToken` and `unbindToken` become amendments (section 5), so they
  are not in S1. The trimmed mirror sets the bound token in the initial
  state.
- A run row becomes a nullary definition and an eval row. S1 has no `run`
  verb.
- Each unit adds its rows when they pass, so each gate stays EXIT 0. The
  restrict copy splits in two: the rules part (U4) and the credential
  part (U7).

| Unit | Content | C lines (est.) | Rows that pass |
|---|---|---|---|
| U0 | `probe/CAPABILITY.md`; `SPEC.md` sections 2 to 8; the two gate loops for `test/restrict/` | 0 | none (loops only) |
| U1 | Words: `U256` as four i64 limbs, `Addr`, the literals `Nu` and `0x` with 40 hex digits, add, sub, eq, le, `addrEq`, `addrToU256`, `toU256`, `toNat`, print | 500 | the word rows of the unit |
| U2 | The keyed map `Map K V`: `mapEmpty`, `mapGet`, `mapSet`, `mapOf` | 350 | the map rows of the unit |
| U3 | The `state` form; `Env` (`makeEnv`, `now`, `caller`, `self`); the `event` form and `Out`; the entry result `Option (Prod State (List Out))` | 450 | the mirror rows |
| U4a | The dependent eliminator `elim` (Q-S1-10) | 200 | the `elim` rows of the unit |
| U4b | The `rules` form, dotted names, `Empty`, `R.Dec` | 250 | the restrict rule and `canTransfer` rows |
| U5 | `hash` (keccak-256 of words) | 200 | the hash rows of the unit |
| U6 | BN254 Fp and G1 | 450 | the curve rows of the unit |
| U7a | bLSAG; `zkKey`, `zkHpAt`, `zkSignAt`, `zkVerifyAt` | 450 | the `zk` rows of the unit |
| U7b | `issue`, `register` and `isVerified` in `test/restrict/rules.lang` | 0 | the restrict credential rows |
| U8 | A trimmed `Charter` in `domain/domain.lang`; `active` by `fold` | 0 (about 50 lines of .lang) | the `active` rows |

S1 close (2026-11-25). U0 to U8 are DONE, each with a green gate. U7 is
split in two: U7a (the C half: bLSAG, `zkKey`, `zkHpAt`, `zkSignAt` and
`zkVerifyAt`) and U7b (the credential rows `issue`, `register` and
`isVerified` in `test/restrict/rules.lang`). U8 puts `Size`, `RuleSet`,
`ruleOf`, `Charter` and `active` in `domain/domain.lang`; the rows are in
`test/restrict/charter.lang`. The USER ruled the open findings on
2026-11-25:

- Finding 9 (keep public): `ecAdd`, `ecMul`, `zkKey`, `zkHpAt`,
  `zkSignAt` and `zkVerifyAt` stay public builtins (restrict-lang: kit
  only).
- Finding 12 (keep R4): an event field is a domain name, so a definition
  cannot have the name of an event field. The restrict row `regClass` is
  `regDaveClass` in `test/restrict/rules.lang`.
- Carry to S2: (1) the `EVAL_OVERFLOW` text names Nat for the U256 and
  curve traps; (2) the `REFUSE_NAME` text of an event says "domain
  constructor"; (3) a `Flag` value prints `1`, restrict-lang prints
  `flagYes`; (4) the text of `elim-motive-dom` follows the motive
  domain; (6) `parse_rules` does not set `is_rec`; (7) `canTransfer` is
  a fixed name; (8) the test `State` keeps `decided`, which SPEC.md
  section 5 drops; (10) dead branches in the zk glue; (11) no
  independent oracle for bLSAG.

## 8. Open questions (S1, RULED 2026-11-23)

Each question has a recommended answer (a). The USER ruled all nine on
2026-11-23 and chose (a) for each. Section 1 holds the answers.

- Q-S1-1. The arithmetic form. (a) One form for each operation, with a
  no-overflow obligation that the checker discharges or that the program
  proves; the fix hint names the missing bound. (b) Checked operations
  that return a sum. (c) Saturating operations.
- Q-S1-2. The cost unit of the bound. (a) Wasm instructions executed, one
  unit each (the wasmtime fuel model). (b) A weighted table for each
  opcode.
- Q-S1-3. A floor on t. (a) The checker requires t > n / 2. (b) Any t from
  1 to n. (c) t >= 2n / 3.
- Q-S1-4. What `amend` can change on the target. (a) It moves among the
  rule sets and parameters that are in the artifact; new code needs a new
  artifact whose chain extends the stored epoch. (b) Parameters only.
- Q-S1-5. Key generation and signing. (a) `langc` subcommands in C for
  offline key generation, signing and aggregation, tested with the RFC
  9591 vectors. (b) An external FROST tool; `langc` only verifies.
- Q-S1-6. The x402 endpoint. (a) A native `langc serve` that TinyCC
  builds. (b) A Cloudflare Worker that runs the ES module target. The USER
  supplies the prices, the payee address and the facilitator.
- Q-S1-7. `funding.json` and `FUNDING.yml`. The USER supplies the entity,
  the contact and the channels. (a) Until then, the account-wide
  `MavenRain/.github/FUNDING.yml` applies and the repo has no
  `funding.json`.
- Q-S1-8. `U256` on Wasm. (a) Four i64 limbs that the kit lowers. (b) Only
  u64, with bound obligations.
- Q-S1-9. The credential curve. (a) Keep the BN254 G1 bLSAG of
  restrict-lang, in C. (b) Move to secp256k1 and share the FROST code.
- Q-S1-10 (asked 2026-11-25, in U4). How `R.decide` decides a rule. (a)
  Port the dependent `elim` of restrict-lang. (b) A new primitive
  `decide : (f : Flag) -> Dec (Holds (lift f))`. (c) Defer `R.Dec`.
