# Declaration — Fe Builders Challenge submission

Required by the Fe Builders Challenge rules: a declaration of pre-existing
work, reused code, and AI assistance. (This file is also the "Declaration"
section source for the submission form.)

## AI assistance — DECLARED

**This project was designed and written with substantial AI assistance.**
An AI coding agent (Muse, operated by the entrant) researched the Fe
documentation, drafted the contract and tests, and iterated against the
Fe 26.5.0 compiler and test runner until all tests passed. The entrant
directed the work, chose the project (scoped agent spending sessions),
specified the required behaviour (per-session cap, target allowlist,
expiry, event log, owner reclaim), and reviewed the result.

Fe's contest rules explicitly welcome AI assistance provided it is
disclosed; this file is that disclosure. No part of the submitted code is
presented as unaided human work.

## Pre-existing work

- **Concept:** "scoped session keys / spend caps for agents" is a pattern
  the entrant had previously explored in a different stack (an EVM
  exit-automation project). No code from that project is used here; the
  idea only. All Fe code in this repository was written during the
  contest build window (from November 13, 2026).
- **Entrant's own prior tooling patterns** (session budgets, allowlists,
  expiry-first execution guards) informed the design, again as ideas, not
  code.

## Third-party code

- **Fe Builders Challenge starter repository**
  (github.com/fe-lang/builders-challenge-starter, Apache-2.0, by the Fe
  team / Argot Collective): this project started from its layout. Kept:
  the CI workflow structure (`.github/workflows/ci.yml`), the `LICENSE`
  file, and the README/SUBMISSION structure. The starter's example vault
  contract and its tests were **replaced entirely** by SpendGuard and its
  own tests; the vault was used as a syntax reference for Fe idioms
  (message definitions, `StorageMap`, events, test harness calls).
- **Fe standard library** (`std::evm`), part of Fe 26.5.0 itself.
- No other third-party code. No copied Solidity; the Solidity-style
  selectors in `sol("...")` strings are computed by Fe's own helper.

## Tools

- Fe 26.5.0 (installed via the official `feup` script).
- Git, and the AI tooling declared above.

## Contact

One responsible human contact, as required by the contest: the entrant,
reachable through the email given in the project's Fe Builders Challenge
registration.
