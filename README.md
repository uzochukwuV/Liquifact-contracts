# LiquiFact Escrow Contract

Soroban smart contracts for **LiquiFact** on Stellar. This repository currently
contains the `escrow` contract crate and its supporting documentation.

## Storage Schema And Upgrade Compatibility

This section is the canonical reviewer-facing description of escrow storage for
Issue #21. It is anchored to the live source in `escrow/src/lib.rs`, but it does
not silently normalize source drift. The documentation below distinguishes:

- the raw storage inventory actually declared or referenced by source
- the narrative schema view reviewers should rely on when evaluating upgrades
- known divergences where code and documentation cannot yet be made perfectly clean

### Canonical source basis

For this task, storage documentation is derived only from the live source in
`escrow/src/lib.rs`:

- instance storage references currently use the literal keys `"escrow"` and `"version"`
- the persisted escrow record is the `InvoiceEscrow` struct as declared in source
- any API parameter or helper that is not written into storage is excluded from the
  persisted schema narrative and called out separately as divergence

### Raw storage inventory

#### Instance storage keys referenced by source

| Key | Source usage | Stored value |
|---|---|---|
| `"escrow"` | `get_escrow`, `init`, `fund`, `settle`, `update_maturity`, `withdraw`, migration example | `InvoiceEscrow` |
| `"version"` | `init`, `get_version`, `migrate` | `u32` schema version |

#### Persisted struct fields exactly as declared in `InvoiceEscrow`

| Order | Field | Rust type | Raw source note |
|---|---|---|---|
| 1 | `invoice_id` | `Symbol` | Declared once |
| 2 | `admin` | `Address` | First `admin` declaration |
| 3 | `sme_address` | `Address` | Declared once |
| 4 | `admin` | `Address` | Duplicate field name in source |
| 5 | `amount` | `i128` | Declared once |
| 6 | `funding_target` | `i128` | Declared once |
| 7 | `funded_amount` | `i128` | Declared once |
| 8 | `settled_amount` | `i128` | Declared once |
| 9 | `yield_bps` | `i64` | Declared as `i64` in struct |
| 10 | `maturity` | `u64` | Declared once |
| 11 | `status` | `u32` | Declared once |
| 12 | `version` | `u32` | Declared once |

### Narrative schema view

The contract is trying to persist a single escrow snapshot plus an explicit
schema version:

- `"escrow"` stores the full invoice escrow snapshot
- `"version"` stores the current schema version independently for upgrade checks

For reviewer guidance, the intended persisted escrow fields are:

| Field | Intended meaning |
|---|---|
| `invoice_id` | Short invoice identifier used as part of the escrow record |
| `admin` | Administrative address intended to control privileged maintenance actions |
| `sme_address` | SME beneficiary address |
| `amount` | Principal amount |
| `funding_target` | Funding threshold, currently initialized from `amount` |
| `funded_amount` | Running funded total |
| `settled_amount` | Running settled total; source support exists but behavior is still inconsistent |
| `yield_bps` | Yield in basis points |
| `maturity` | Maturity timestamp |
| `status` | Lifecycle state flag |
| `version` | Persisted schema version, expected to track `SCHEMA_VERSION` |

### Status values

The live source and comments currently imply these status codes:

| Value | Meaning | Notes |
|---|---|---|
| `0` | Open | Funding allowed |
| `1` | Funded | Funding target reached; additional flows diverge |
| `2` | Settled | Mentioned in source comments and settlement logic |
| `3` | Withdrawn | Used by `withdraw`, but not reflected consistently in top-level status docs |

### Versioning behavior

The source currently treats schema versioning as follows:

- `SCHEMA_VERSION` is set to `1`
- `init` writes `SCHEMA_VERSION` into both the escrow record's `version` field and the `"version"` instance key
- `get_version()` reads `"version"` and falls back to `0` if absent
- `migrate(from_version)` checks that the stored `"version"` matches the argument
- `migrate(from_version)` rejects `from_version >= SCHEMA_VERSION`
- no actual migration arm is implemented yet; current code ends in a panic path

### Upgrade compatibility guidance

Future upgrade work should preserve a strict distinction between additive
behavior changes and storage-breaking schema changes.

#### Additive changes

These are usually safe without rewriting existing bytes:

- adding new methods that read existing storage without changing the stored layout
- adding documentation-only clarifications
- adding new events that describe state already present in storage

#### Breaking schema changes

These require a version bump and a migration strategy:

- adding, removing, renaming, reordering, or retyping any persisted `InvoiceEscrow` field
- changing the serialization shape of the `"escrow"` value
- changing the meaning or type of the `"version"` key

#### Required migration discipline

For any future schema bump:

1. Bump `SCHEMA_VERSION`.
2. Keep historical decoders available, typically as explicit legacy structs.
3. Add a migration arm that reads the old layout and writes the new layout.
4. Update both the `"escrow"` value and the `"version"` key atomically.
5. Preserve old migration arms so historical deployments remain upgradable.

### Known schema and documentation divergences

These source inconsistencies affect how the storage story must be documented
today. They are called out explicitly instead of being papered over in prose.

| Source divergence | Documentation treatment | Why it matters for upgrades |
|---|---|---|
| Duplicate `admin` field in `InvoiceEscrow` | Treated as code drift, not as two meaningful persisted admin roles | Reviewers should not design migrations around a fictitious dual-admin schema |
| `funding_deadline` parameter in `init` is not written into storage | Excluded from persisted schema narrative | API shape and storage shape are currently different |
| `settled_amount` exists in the struct but surrounding settlement flow is inconsistent | Documented as persisted with unstable behavioral support | A future migration must preserve the field even if semantics are tightened |
| `yield_bps` is `i64` in `InvoiceEscrow` but `u32` in `init` | Documented as a source/interface mismatch | Retyping one side later is a breaking compatibility concern |
| `DataKey::Escrow` is referenced in `init` but no `DataKey` enum is defined | Storage docs use the actual live keys `"escrow"` and `"version"` | Reviewers should anchor upgrades to real storage references, not undefined helper machinery |

### Rustdoc rendering limitation

The rustdoc clarifications added to `escrow/src/lib.rs` are source-accurate, but
they are not currently renderable in generated documentation because the crate
does not compile. In particular, the duplicate `admin` field declaration in
`InvoiceEscrow` blocks successful compilation. That duplicate should be removed
in a separate fix PR; this documentation PR does not change contract behavior or
repair compile-time source drift.

### Test-state notes for this documentation task

`escrow/src/test.rs` currently mixes live escrow expectations with obvious branch
drift from unrelated work. For documentation purposes:

- the file still confirms that storage assumptions in the repo are unstable
- it is not safe to treat the full test suite as an authoritative schema spec
- the schema narrative in this README is therefore anchored to `lib.rs`, with
  divergences surfaced explicitly
- the crate does not currently compile, and that pre-existing failure is
  independent of these documentation edits

### Security and upgrade notes

- Re-initialization protection is intended, but the current guard references undefined helper machinery.
- Upgrade paths should be admin-gated before production deployment.
- Migration code must never silently drop or reinterpret historical fields.
- Reviewers should treat source drift as a signal to preserve backward decoders and add focused migration tests before any schema evolution lands.

## Validation Commands

| Step | Command | Fails if... |
|---|---|---|
| Format | `cargo fmt --all -- --check` | any file is not formatted |
| Build | `cargo build` | compilation error |
| Tests | `cargo test` | any test fails |
| Coverage | `cargo llvm-cov --features testutils --fail-under-lines 95` | line coverage < 95% |

### Coverage gate

The pipeline uses [`cargo-llvm-cov`](https://github.com/taiki-e/cargo-llvm-cov)
to measure line coverage and fail when it drops below **95%**.

```bash
cargo install cargo-llvm-cov
rustup component add llvm-tools-preview
cargo llvm-cov --features testutils --fail-under-lines 95 --summary-only
```

---

## Contributing

1. **Fork** the repo and clone your fork.
2. **Create a branch** from `main`: `git checkout -b feature/your-feature` or `fix/your-fix`.
3. **Setup**: ensure Rust stable is installed; run `cargo build` and `cargo test`.
4. **Make changes**:
   - Follow existing patterns in `escrow/src/lib.rs`.
   - Add or update tests in `escrow/src/test.rs`.
   - Format with `cargo fmt`.
5. **Verify locally**:
   - `cargo fmt --all -- --check`
   - `cargo build`
   - `cargo test --features testutils`
6. **Commit** with clear messages (e.g. `feat(escrow): X`, `test(escrow): Y`).
7. **Push** to your fork and open a **Pull Request** to `main`.
8. Wait for CI and address review feedback.

We welcome new contracts (e.g. settlement, tokenization helpers), tests, and docs that align with LiquiFact's invoice financing flow.

---

## Future Improvements

- Multi-escrow support
- Role-based access control
- Token integration
- Event emission
- Formal verification
