# bylaw-lang domain design

> Paste the domain design below the line, verbatim. Do not edit it.

This file is the only source of the core types and the core operations of
bylaw-lang. `SPEC.md` sections 5 and 6 cite its sections. Do not add a core
type or a core operation that this file does not give.

When the design changes, paste the new version. Then record the change as an
open item in `SPEC.md` section 9.

---

## The request (verbatim, 2026-11-23)

I want to write a new programming language with the following characteristics:

First, nothing that can cause a runtime error (trap) should be allowed to compile, period.

As for type constructors/higher-kinded types, admit only cell monads and their duals as represented computationally in a cubical type theory.

For base types and operations, admit what is covered in restrict-lang, except have governance be by an offline threshold-authorized self-amending system (offline version of a DAO).

The host language shall be TinyCC.  Ask multiple-choice questions to arrive at the compilation target(s).

Finally, in order to encourage usage by agents, adhere to the following additional criteria: 

The plan for making agents reach for the language includes:
  - one canonical form per operation and a langc fmt canonical printer;
  - langc check --json, giving fix hints and progress counts (parsed, typed, laws
    discharged);
  - a SKILL.md with 3 to 5 graded examples;
  - an MCP server written in TinyCC (langc mcp);
  - AGENTS.md and a README table saying when to use the language;
  - a token-count comparison against Python and Rust.
  - List where agents look. That means an MCP server entry in the public
     registries, an OpenAPI description, and the price in the 402 response.


1. Make the work agent-consumable and payable (highest leverage)
* Wrap existing formalizations or verification workflows as MCP tools or simple x402-gated endpoints. Examples: “verify this Halo2 gadget against the formalization,” “search formalized properties of this circuit,” “check this snippet for known invariants,” or “request a formalization sketch of gadget X.”
* Price per use in the $0.01–$0.20 range (or higher for heavier verification). Agents (or their principals) will use tools that improve success rate or reduce verification cost.
* Publish Agent Skills (SKILL.md with name + description frontmatter) that teach agents how and when to use your libraries or tools. Skills are an open standard (originally from Anthropic, now under the Agentic AI Foundation) and load progressively, so discovery is cheap.
2. Publish machine-readable discovery and funding signals
* Add FUNDING.yml plus a simple funding.json (or equivalent) to GitHub repos.
* Put an llms.txt (and, where relevant, .well-known/ai-donate.json or similar) on any site or content property that describes the projects, transparency, and explicit welcome for agent payments.
* Include AGENTS.md or Skills in the repos so coding agents that encounter your code can surface funding options.
Agents parse these files the way humans read a README or Sponsors page.
3. Use agent-native funding rails for larger efforts
* Run specific formalization milestones on AgentFund (or similar) with escrow and clear deliverables. Agents can discover, evaluate, contribute, and vote.
* Keep human channels (GitHub Sponsors, Open Collective, Patreon) as the primary near-term path; treat agent micropayments and campaigns as complementary. Better tools that agents actually use also generate recommendations and visibility that drive human sponsors.
4. Frame the value in agent terms
Position the work as shared infrastructure for agentic systems that need verifiable correctness: formalized payment or agent protocols, ZK compliance layers, verified gadgets that other agents can rely on. Agents (and the people who fund them) respond to clear ROI on reliability and reduced verification cost more than to general appeals.
Practical starting sequence
1. Add machine-readable funding metadata (FUNDING.yml, llms.txt, funding manifest) to the main repos.
2. Expose one useful Lean/ZK capability as an MCP tool or lightweight x402 endpoint (Cloudflare Workers + facilitator, or a simple Express + x402 middleware, are common starting points).
3. Publish a short Skill that documents how to use it.
4. List the endpoint/tool in relevant MCP or x402 discovery surfaces and mention it in content that reaches agent developers.
5. Measure actual agent traffic and payments before investing heavily in larger campaigns.

## The target rulings (USER answers to the multiple-choice questions, verbatim, 2026-11-23)

Round 1:

- Q: "Which compilation targets should langc ship for the new language?"
  A: "Wasm MVP module, JSON / ES module"
- Q: "Which target should the first slices build and the gate prove? The
  other targets you chose become later slices."
  A: "Wasm MVP module"
- Q: "Gas, fuel, stack and memory exhaustion depend on what the caller
  supplies, not on the program. How should the no-runtime-error rule handle
  them?"
  A: "Bound + entry pre-check (Recommended)". The option text: "The checker
  proves a worst-case cost for each entry (every program is total, so a
  bound exists). Before doing any work, the entry compares the supplied
  budget with that bound and returns a typed Underfunded result instead of
  trapping. The bound is published in the ABI and the OpenAPI description."
- Q: "Under offline threshold governance, amendments are signed t-of-n
  offline, and the constitution (t and n included) amends itself. What must
  each compiled target contain for this?"
  A: "Verifier on target, Schnorr (Recommended)". The option text: "langc
  refuses any source whose amendment chain lacks valid t-of-n signatures.
  Each artifact also carries a small verifier, so a deployed instance can
  accept a signed amendment with no online vote. FROST Schnorr over
  secp256k1 verifies on the EVM through ecrecover, with no precompile, and
  in plain C."

Round 2:

- Q: "Which host should the Wasm MVP module be built for?"
  A: "Host-neutral (Recommended)". The option text: "Keep the tcc-wasm kit
  ABI and add one import, budget_left, that the embedder supplies (wasmtime
  fuel, a counter in Node or the browser). Runs anywhere Wasm runs, with no
  chain lock-in. Chain adapters can come in later slices."
- Q: "Without the EVM, should the compiled module keep the ERC-3643
  interface shape? The restrict-lang base types stay either way."
  A: "Keep the ERC-3643 model (Recommended)". The option text: "Identity
  registry, compliance module, canTransfer, and the
  transferred/created/destroyed hooks become Wasm exports with the same
  names and meanings, so the rules stay comparable with T-REX deployments."
- Q: "For the later JSON / ES module target, which output should langc
  write?"
  A: "ES module + JSON (Recommended)". The option text: "tcc-js kit: an ES
  module that evaluates the program, plus the JSON result document."
- Q: "What should the language be called?"
  A: "bylaw-lang"

## The base types and operations (inherited from restrict-lang)

The request admits "what is covered in restrict-lang". The source of those
base types and operations is the pasted design of restrict-lang, copied
below verbatim from MavenRain/restrict-lang `design/DESIGN.md` at a0f4784.
The request replaces its governance clause: an offline threshold-authorized
self-amending system governs each program, not the DAO of orbit-lang.

### The pasted design of restrict-lang (verbatim)

  3. A transfer-restriction language for tokenized assets

  - Activity:
    - On Sept 17, the SEC exemption let permissioned on-chain venues trade
      tokenized US stocks.
    - On Sept 1, the SEC proposed letting transfer agents keep records on chain.
    - Today's Federal Register carries the OCC's proposed rules for stablecoin
      issuers under the GENIUS Act, the 2025 US stablecoin law.
  - The language: ERC-3643 is the token standard most used for permissioned
    securities. Today its compliance rules are hand-written Solidity: holder
    caps, country rules, lock-ups. The language would state those rules as types
    and identity claims as ZK credentials, then compile to an ERC-3643
    compliance contract.
