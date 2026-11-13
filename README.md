# SpendGuard

**Pitch:** SpendGuard is a Fe contract that lets a human owner give an autonomous agent a *scoped* ETH budget — a per-session spending cap, a target allowlist, and an expiry — enforced on-chain on every payment, so an agent can pay for what it needs without ever holding an unbounded allowance.

## The problem

Agents increasingly need to spend money on their operator's behalf: paying for API calls, data, compute, or other agents' services. The common patterns are all bad:

- **Give the agent a funded wallet** — a compromised or buggy agent can drain all of it.
- **ERC-20 `approve` allowances** — usually set to huge/infinite amounts, per-token, and silent about *who* may be paid and *until when*.
- **Off-chain policy engines** — the enforcement lives in software the agent itself runs, so it is only as trustworthy as the agent.

What is missing is a small, auditable on-chain primitive: *this agent, at most this much, only to these addresses, only until then* — with every payment logged.

## Approach

One contract, `SpendGuard` (`src/lib.fe`), written in Fe 26.5:

1. The **owner** funds the contract with `deposit()` (owner-only, payable).
2. The owner opens a **session** for one agent with `createSession(agent, cap, valid_for)`. The cap is the *cumulative* total the agent may spend; the expiry is `block.timestamp + valid_for`.
3. The owner allowlists payment targets per session with `allowTarget` / `revokeTarget`.
4. The **agent** pays with `spend(session, to, amount)`. The contract checks, in order: session active → caller is the session's agent → target allowlisted → not expired → cumulative cap not exceeded. It then records the new spent total **before** sending value (checks-effects-interactions) and emits a `Spent` event with the running total.
5. The owner can `closeSession` at any time (ids are never reused) and `withdraw` unspent funds.

Every state-changing operation emits an event (`Deposited`, `SessionCreated`, `TargetAllowed`, `TargetRevoked`, `SessionClosed`, `Spent`, `Withdrawn`), so an operator can reconstruct the complete spending history from logs alone.

## Fe highlight

Fe's **message + effects model** makes the guardrails read exactly like the policy they enforce. The contract's interface is a single `msg` enum whose variants carry Solidity-compatible selectors, and each handler declares precisely what it may touch in its `uses` clause — e.g. `Spend` is the only handler that gets `mut call` (the ability to send value) *and* write access to the spent totals:

```fe
Spend { session, to, amount } -> Wei uses (ctx, mut call, agents, caps, mut spents, expiries, actives, allowed, mut log) {
    assert!(amount > Wei::zero(), "Nothing to spend")
    assert!(actives.get(key: session), "Session is not active")
    let agent = agents.get(key: session)
    assert!(ctx.caller() == agent, "Caller is not the session agent")
    assert!(allowed.get(key: (session, to)), "Target is not allowlisted")
    assert!(ctx.timestamp() < expiries.get(key: session), "Session expired")

    let new_total = spents.get(key: session) + amount
    assert!(new_total <= caps.get(key: session), "Session cap exceeded")
    spents.set(key: session, value: new_total)
    // ...
}
```

Two more Fe features do quiet work here: typed value types (`Wei`, `Timestamp`, `Duration`) make unit mix-ups compile errors (a duration can never be added to a balance), and the per-session allowlist is a single `StorageMap<(u256, Address), bool>` — a tuple-keyed storage map, so no manual key hashing.

## Architecture

```
fe-spendguard/
├── fe.toml                  # ingot manifest (name: spendguard)
├── src/
│   ├── lib.fe               # the SpendGuard contract (only source file)
│   └── tests.fe             # 8 tests, run with `fe test`
├── .github/workflows/ci.yml # fmt --check + test + build on Fe v26.5.0
├── DECLARATION.md           # pre-existing work / third-party code / AI use
└── LICENSE                  # Apache-2.0
```

Storage layout (all keyed by session id unless noted): parallel scalar `StorageMap`s for the session's agent, cap, cumulative spent, expiry, and active flag, plus the tuple-keyed allowlist map. Session data is kept in parallel scalar maps because Fe's `StorageMap::get` requires copyable values — a deliberate, documented trade-off (see Limitations).

## Build and test

```sh
fe --version   # fe 26.5.0 (3dab95f) — any Fe release from 26.5.0 on works
fe test        # 8 passed; 0 failed
fe build       # writes out/SpendGuard.bin, .runtime.bin, .abi.json
```

Install Fe with feup:

```sh
curl -fsSL https://raw.githubusercontent.com/argotorg/fe/master/feup/feup.sh | bash -s -- --version v26.5.0
```

No other tools are required. (Deploying to a local chain with Foundry's `anvil` + `cast` works the same way as in the Fe guide's first-contract tutorial, using `out/SpendGuard.bin`.)

## Demo

Reproducible CLI example — the test suite *is* the demo; it deploys the contract in Fe's local EVM sandbox and walks a full session lifecycle:

```sh
fe test
```

Expected output (abridged):

```
PASS  create_session_and_spend_within_cap
PASS  spend_above_cap_reverts
PASS  cap_accumulates_across_spends
PASS  spend_to_unallowed_target_reverts
PASS  spend_after_expiry_reverts
PASS  spend_by_non_agent_reverts
PASS  spend_after_close_reverts
PASS  owner_withdraws_unspent_funds
test result: ok. 8 passed; 0 failed
```

The happy-path test funds the guard with 2 ETH, opens a 1 ETH / 1-day session, allowlists a payee, spends exactly the cap, and verifies the payee's balance plus the `remaining == 0` accounting on-chain.

## Limitations

- **Native ETH only.** Sessions spend the contract's ETH balance. ERC-20 budgets would need a token-transfer variant of `spend` (the session/allowlist/expiry machinery carries over unchanged).
- **One agent per session, one currency of trust.** The owner is a single address set at deploy time; there is no owner rotation or multisig in this version.
- **Cap is cumulative per session, not rate-limited.** An agent may spend the whole cap in one payment to an allowlisted target. Per-payment limits or velocity windows are natural follow-ups.
- **Test coverage of caller roles.** The Fe test EVM sends every call from the test account, so owner-only restrictions are exercised indirectly (e.g. the non-agent spend revert) rather than by impersonating a third account. The owner checks themselves are single `assert!`s on `ctx.caller()`, identical in shape to the agent check that *is* directly tested.
- **Not audited.** This is a contest prototype built and tested locally; do not hold real funds in it without an independent security review.

## Declaration

See [DECLARATION.md](DECLARATION.md) for the full declaration of pre-existing work, third-party code, and AI assistance (also summarized in the submission form).

## Team

Solo build by Abdul99090 (GitHub), with AI assistance as declared in DECLARATION.md. Contest contact: see the project's Fe Builders Challenge registration.

## License

Apache-2.0 — see [LICENSE](LICENSE).
