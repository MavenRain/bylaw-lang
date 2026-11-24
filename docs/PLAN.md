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

## 8. Open questions (S1)

Each question has a recommended answer (a).

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
