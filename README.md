# bylaw-lang

bylaw-lang is a Cubical TT language for the transfer rules of tokenized
securities. It takes the base types and operations of restrict-lang:
holder caps, country rules and lock-ups as decidable propositions, and
identity claims as ZK credentials. A program that can cause a runtime
error (a trap) does not compile. An offline threshold-authorized
self-amending constitution governs each program: t of n signers sign each
amendment offline with FROST Schnorr over secp256k1. The only type
constructors are cell monads and their duals. The host is TinyCC. The
compiler writes a Wasm MVP module with the ERC-3643 exports. An ES module
and JSON target follows in a later slice.

bylaw-lang comes from lang-template with the tcc-wasm host.

- `design/DESIGN.md`: the request, the target rulings and the inherited
  restrict-lang brief, verbatim. It is the only source of the core types
  and the core operations.
- `docs/PLAN.md`: the rulings, the reading of the no-trap rule on the kit,
  the base signature, the governance design, the agent surface, the slices
  and the open questions.
- `SPEC.md`: the specification.
- `formers/FORMERS.md`: the type formers. `formers/tcc-wasm.md` gives their
  status on the tcc-wasm host.
- `probe/CAPABILITY.md`: the host probe.
- `docs/STATUS.md` and `docs/VALIDATION.md`: the status and the gate results.
- `docs/host/README.md`: how to build and test the host kit, and how to
  replace the sample domain in `domain/`.
