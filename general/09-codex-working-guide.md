# Codex Working Guide for A2MaTeX

## 1. Purpose

This file tells a coding agent how to use the system documentation. It does not
replace repository-specific `AGENTS.md` instructions.

## 2. Read only what the task needs

| Task type | Required documents |
|---|---|
| Product boundary or new feature | `01-system-context.md`, `02-four-platforms.md`, `07-features-and-use-cases.md` |
| Canvas, graph, node, loop, scheduler | `03-workflow-and-runtime.md` |
| Submission, build, review, signing, package | `04-build-review-and-apkg.md` |
| Android permission, credential, entitlement, lease | `05-permissions-credentials-and-license.md` |
| Entity/state/limit/retention/error naming | `06-domain-model-and-policies.md` |
| UAT implementation plan or acceptance | `08-uat-plan.md` |
| Database work | Read `06` and `08`, then use the separate Database Design document when created |
| SDS/API work | Read all affected domain documents, then use the separate SDS/API document when created |

## 3. Non-negotiable invariants

Before proposing or implementing a change, preserve these rules unless the user
explicitly changes the product decision:

1. There are four products: Studio, Web Marketplace, Android Launcher, Runtime
   Agent.
2. Admin is a protected Web area, not a fifth product.
3. `.apkg` is a signed interpreted package, not an APK.
4. Draft is mutable; Revision and published Product Version are immutable.
5. Review Decision binds to one exact candidate hash.
6. Signing must fail if the candidate hash differs.
7. Custom code has no direct native/network/filesystem/process access.
8. Sensitive operations go through Capability Broker.
9. Android OS permission and Product-level grant are both required.
10. Raw credentials never enter graph, source snapshot, `.apkg`, or normal logs.
11. Entitlement and License Lease are separate concepts.
12. Offline use is bounded by a signed Lease.
13. Arbitrary cycles are forbidden; loops use Loop Controller.
14. Node count is not a paywall.
15. UAT supports restricted TypeScript only.

## 4. How to handle ambiguity

When documentation does not decide an implementation detail:

1. Check whether it belongs to SDS, Database Design, API, or Threat Model.
2. Do not silently turn an example into a permanent contract.
3. Present the choice and its impact.
4. Record the selected decision in the narrowest relevant document.
5. Update cross-references only if the decision changes another module's behavior.

## 5. How to add a feature

1. Identify the owning product and shared service boundary.
2. Add a stable feature ID to `07-features-and-use-cases.md`.
3. Add or update the use case.
4. Update the mechanism document if behavior changes.
5. Update UAT scope only if delivery scope changes.
6. Add an open SDS/DB decision rather than designing it prematurely.

Suggested ID prefixes:

| Prefix | Area |
|---|---|
| `F-STU` | Developer Studio |
| `F-PLT` | Shared platform |
| `F-BLD` | Build and validation |
| `F-REV` | Review |
| `F-SIG` | Signing |
| `F-MKT` | Marketplace |
| `F-LAU` | Android Launcher |
| `F-RUN` | Runtime Agent |
| `F-OPS` | Operations and audit |
| `UC` | Use case |
| `BR` | Business rule |
| `UAT` | Acceptance scenario |

Never renumber existing IDs merely to make a list visually contiguous. Mark a
removed item deprecated and add a replacement ID.

## 6. Terminology rules

Use these terms consistently:

- **Creator** writes and tests the workflow.
- **Publisher** registers, submits, and publishes.
- **Customer** acquires and runs a Product.
- **Draft** is mutable.
- **Revision** is immutable source state.
- **Submission** requests review.
- **Candidate Artifact** is built but not production-signed.
- **Product Version** is a published immutable release.
- **Entitlement** is the business right.
- **License Lease** is bounded signed execution authorization.
- **Product Grant** is Customer permission for a capability.
- **Android OS permission** applies to Launcher APK.
- **Credential Binding** is a reference, not a token.

## 7. Example versus contract

The following values are current UAT baselines and must remain configurable:

- 10 MB graph.
- 10 MB total source.
- 50 MB compressed submission.
- 100 MB expanded submission.
- 1,000 files.
- 20:1 compression ratio.
- 10,000 loop iterations.
- 24-hour Lease refresh threshold.
- 72-hour offline window.
- 6-10 initial System Nodes.

Do not hard-code these values across application code.

## 8. Documentation quality checks

Before completing a documentation change:

- Check relative links from `README.md`.
- Search for contradictory product counts or `.apkg` described as executable/APK.
- Search for raw token storage examples.
- Search for direct Custom Node access to Android/network/filesystem.
- Confirm new capability includes review, disclosure, Runtime enforcement, and
  revoke behavior.
- Confirm a new mutable object does not weaken Revision/Artifact immutability.
- Confirm a new offline behavior states its Lease/revocation limitation.
- Confirm feature/use-case IDs are stable and unique.

