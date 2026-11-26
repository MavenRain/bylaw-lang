# Host capability probe: tcc-wasm

Date: 2026-11-24. Host tree: `/Users/oobi/Documents/bylaw-lang` at 0020ab5
(tree b0bc2f63), read only; the S1 edits to `SPEC.md`, `docs/PLAN.md` and
`test/gate.sh` were not committed. Driver: `build/langc` (built
2026-11-23; `make -q build/langc` gave 0). Other work loaded the machine
(load average 12.4 at the start, 11.6 at the end), so each time is an
upper bound.

To run the probe again: `zsh -f bylaw-lang-work/s1/probe/probe.sh` (outside
the repo; the log is `probe.log` next to it, cited below as `Ln`). Each step
runs under `probe/guard.py --rss-mb 4096 --timeout 120`, which stops a
command above 4096 MB resident memory or above 120 s. The probe used at
most 793.5 MB (the fuel step); without that step, at most 41.7 MB.

`PIN` lines 1 and 3 to 5 give the tools: tcc 0.9.28rc 2026-09-04
mob@0fb54300, node v23.10.0, wasm-validate and wasm-objdump 1.0.39 (L8-11).
The `PIN` of relay-lang has sqlite3 lines from line 6. They do not apply:
bylaw-lang links no library, and `build/langc` is the only driver.

## Questions

Each host must answer these questions. Each answer is one numbered section
below. An answer sets the status of a former in `formers/tcc-wasm.md`.

| Question | Section | Formers |
|---|---|---|
| Which commands check a file, print a normal form, build and run? | 1 | all |
| Does the host take several files, or must the driver join them? | 2 | all |
| Which recursion does the host accept, and how does it refuse the rest? | 3 | F6, F7 |
| Which `Nat` primitives exist? Does `Nat` have an eliminator? | 4 | F6, F7 |
| How does a result leave the host? | 5 | target |
| What do check and build cost (time and resident memory) at N = 100 and N = 1000? | 6 | all |
| Can a family take a parameter? Can a constructor of such a family appear in a term? | 7 | F4, F12, F15 |
| Can a program project a Sigma? Does the kernel accept Sigma eta? | 8 | F11 |
| How does a program use equality: transport, cong, an index type? | 9 | F12, F13 |
| How does the host report an axiom? | 10 | R2 |

## 1. Commands

| Step | Command | Result |
|---|---|---|
| Check | `build/langc check FILE` | `ok` and exit 0 (L23; each of the 10 examples, L27-63); a refusal is exit 1 and `langc: CODE: ...` |
| Print normal form | `build/langc eval FILE NAME [ARGS]` | the value, exit 0 (`square 7` gives 49, L67); a missing argument is `EVAL_ARGS`, exit 2 (L76-78) |
| Build | `build/langc build FILE -o OUT [--runtime]` | a Wasm module, exit 0 (1366 bytes for `examples/entries.lang`, L92); `--runtime` gives the same bytes (L104-109) |
| Run | `node test/run-wasm.mjs WASM NAME ARGS` | the result of the export (`square 7` gives 49, L110) |

- `build/langc ir FILE` prints the lowered functions (L80-84). `build/langc
  abi FILE` prints one entry name on each line (L86-90).
- No arguments: `USAGE`, exit 2; the usage lists only `check`, `eval`,
  `ir`, `abi` and `build` (L14-22).
- Not in this host (each `USAGE`, exit 2): `check --json` (L137), `--kit`
  (L146), `explain` (L155), `sim` (L164). There is no `run` verb.

## 2. Several files

No. Two files give `USAGE` `too many arguments` (L173). `import domain`
gives `PARSE_EXPECT` (L182-184). The domain names need no import (`def c :
Color := red` checks, L190). A driver must join the parts into one file.

## 3. Recursion and totality

Only `fold` and `unfold` recurse. A plain check refuses the rest with
`REFUSE_REC` (rule R3): `def rec` (L194-196), a use of the definition
itself (L198-200) and a use of a later definition (L202-204). `fold` over
`Nat` works (L248). The evaluator stops at 20000000 steps (`EVAL_FUEL`,
L206-209) and at a depth of 2000 (`EVAL_DEPTH`, L399-401).

## 4. Nat

`Nat` is a core type: u64. The primitives are `natAdd`, `natSub`,
`natMul` (`Nat -> Nat -> Nat`), `natEq` and `natLe` (`-> Flag`).

- `natSub 4 9` gives 0 (L210). The literal 2^64 - 1 evaluates (L214); 2^64
  is `LEX_NAT_RANGE` (L264-266).
- `natAdd` and `natMul` trap on u64 overflow: `eval` prints `trap` on
  stdout, `langc: EVAL_OVERFLOW: NAME: ...` on stderr, and exits 0
  (L222-225, L235-238). `def sum : Option Nat := natAdd 2 3` is
  `TYPE_MISMATCH` (L227). The edge 4294967295 * 4294967297 = 2^64 - 1 does
  not trap (L240).
- Traps are lazy: `flagIf flagYes 1 (natAdd max 1)` gives 1 (L244).
- The eliminator is `fold` (L248). `natDiv`, `natMod` and `natRec` are
  `NAME_UNKNOWN` (L252-260).
- On Wasm an overflow also traps (`addPrice` at the maximum, L114; `pick 2 4
  9`, L122).

## 5. How a result leaves the host

As printed text (`eval`, `ir`, `abi`) or as one Wasm module (`build -o`).
`wasm-validate` accepts the module (L95) and node runs it through
`test/run-wasm.mjs` (L110-122). There is no JSON, SQL or TypeScript output.

## 6. Cost

The input is a chain of `def dI : Nat := natAdd dJ 1`. Each cell is the wall
time and the peak resident memory.

| Step | N = 100 | N = 1000 |
|---|---|---|
| Check | 0.027 s, 2.1 MB (L371) | 0.023 s, 5.5 MB (L389) |
| Build | 0.027 s, 2.6 MB (L375) | 0.027 s, 6.8 MB (L393) |
| Run | eval 0.027 s, 2.2 MB, 100 (L381); node 0.052 s, 41.0 MB (L385) | eval `EVAL_DEPTH` (L399-402); node 0.08 s, 41.7 MB, 1000 (L403-406) |

The chain definitions are `Nat` entries, so each is exported: the module
grows from 1366 to 28994 bytes (L408-412). A command takes at least about
0.023 s.

## 7. Parametric families

A program cannot declare a family (`REFUSE_DATA`, L268, L272). The domain
file can. The shipped domain has one index, `family Lot (size : Nat)`:
`eval lot` gives `makeLot 3 250` (L276), `makeLot 250` is `TYPE_ARITY`
(L280), `Lot 4 := makeLot 3 250` is `TYPE_MISMATCH` (L284), and `(n : Nat)
-> (l : Lot n) -> Nat` checks (L288). No shipped family takes a type
parameter. A kit copy with `family Wrap (A : Type 0) := wrap (unwrap : A)`
builds, `wrap Nat 3` checks and evaluates, and `wrap 3` is `TYPE_ARITY`
(L296-314): the constructor takes the type first.

## 8. Sigma projection and eta

`witness` and `payload` of a `pack` check (L326); `eval` of the witness
gives 1200 (L330). `.1` and the pattern `(n, p)` are `LEX_CHAR` (L336,
L340). The kernel identifies `pack (witness s) (payload s)` with `s` by
`refl` (L342).

## 9. Equality

`Eq`, `refl`, `symm`, `trans`, `transport` and `cong` are core names.
`examples/equality.lang` checks (L35). `refl` at `Eq Flag`, `Eq Color` and
`Eq Item` checks (L350): one `Eq` serves each index type. `Path` is
`NAME_UNKNOWN` (L354), so the copies of restrict-lang write `Eq`.

## 10. Axiom detection

No command lists the axioms. A program cannot declare one:
`test/check/axiom.lang` is `REFUSE_AXIOM` `magic: an axiom has no proof
(rule R2)`, exit 1 (L358). A file with no axiom gives `ok` (L186).

## Consequences for M0

1. The restrict-lang copies are single files with the domain names in
   scope, and they write `Eq` in place of `Path` (sections 2, 9).
2. The run scripts of restrict-lang become nullary definitions and `eval`
   rows, because the host has no `run` verb (section 1).
3. `natAdd`, `natMul` and the lazy trap flags are trap sites that R1
   removes in S2 and S3 (section 4; `docs/PLAN.md` section 2).
4. `U256`, `Addr` and `Map K V` are new core types in C, because a program
   cannot declare a family and Wasm MVP has no 256-bit type (sections 4, 7;
   O8).
5. `Env`, `Out`, `Empty` and the trimmed `Charter` go in the domain file,
   which takes families with parameters (section 7).
