# A2MaTeX System Documentation

This directory is the living product and system knowledge base for A2MaTeX.
It is intended for developers, Codex, product stakeholders, and future BA/SDS work.

The documents describe what the system must do, the boundaries between the four
products, and the mechanisms already agreed for UAT. They deliberately stop
before low-level software design and physical database design.

## Product in one paragraph

A2MaTeX lets a Publisher build an automation application as a visual workflow,
combine platform-provided nodes with restricted TypeScript nodes, test it in the
Developer Studio, submit an immutable revision for validation and human review,
and publish it as a signed `.apkg`. A Customer acquires the product from the Web
Marketplace, installs it through the Android Launcher, grants product-level
permissions and credential bindings, and runs it through the shared Runtime
Agent. A product is not a separate Android APK.

## The four products

1. **Developer Studio** - build and test workflows.
2. **Web Marketplace** - discovery, Publisher operations, Customer library, and
   protected Admin/Reviewer operations.
3. **Android Launcher** - install, authorize, configure, run, and monitor products.
4. **Runtime Agent** - verify packages and execute workflows under policy.

Backend, build, review, signing, licensing, entitlement, audit, and storage are
shared platform capabilities serving these four products. Admin Portal is a
protected area of the Web Platform, not a fifth product.

## Document map

| File                                                                                   | Read when working on                                                                                       |
| -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| [01-system-context.md](01-system-context.md)                                           | Product goals, actors, scope, principles, and system-wide flow                                             |
| [02-four-platforms.md](02-four-platforms.md)                                           | Responsibility and boundaries of each product                                                              |
| [03-workflow-and-runtime.md](03-workflow-and-runtime.md)                               | Graph, nodes, branching, merge, loop, TypeScript, scheduler, execution                                     |
| [04-build-review-and-apkg.md](04-build-review-and-apkg.md)                             | Revision, validation, build, review, signing, package, install, update                                     |
| [05-permissions-credentials-and-license.md](05-permissions-credentials-and-license.md) | Capabilities, Android permissions, credentials, entitlement, offline lease                                 |
| [06-domain-model-and-policies.md](06-domain-model-and-policies.md)                     | Conceptual entities, states, limits, retention, analytics, error taxonomy                                  |
| [07-features-and-use-cases.md](07-features-and-use-cases.md)                           | Feature inventory, use-case inventory, and critical scenario details                                       |
| [08-uat-plan.md](08-uat-plan.md)                                                       | UAT boundary, five-week delivery plan, acceptance flows, risks, backlog                                    |
| [09-codex-working-guide.md](09-codex-working-guide.md)                                 | How Codex should navigate and update this documentation                                                    |
| [10-database-design.dbml](10-database-design.dbml)                                     | PostgreSQL logical design with table/field notes, relationships, indexes, states, and retention boundaries |

## Source-of-truth rules

- The newest explicit product decision overrides an older description.
- A mutable Draft must never be treated as a submitted or published version.
- A Review Decision applies to one exact candidate artifact hash.
- An `.apkg` is a signed package interpreted by the shared Runtime Agent; it is
  not an APK and is not directly executed by Android.
- Human review complements Runtime enforcement. It never replaces it.
- Raw credentials must never be stored in a workflow graph or `.apkg`.
- Arbitrary graph cycles are forbidden. Repetition is represented by a
  controlled Loop Controller.
- UAT uses restricted TypeScript only. Python, Go, arbitrary npm packages, and
  native custom nodes are later scope.

## How to update the docs

1. Update the narrowest relevant document.
2. Preserve existing IDs such as `F-STU-01` and `UC-021`.
3. If a decision affects multiple modules, update the primary document first,
   then update summaries or links elsewhere.
4. Record unresolved implementation choices under **Open decisions** instead of
   silently inventing an answer.
5. Do not add table/column/index designs here. Put them in the future Database
   Design document.
6. Do not add class/module/deployment detail here unless needed to explain a
   product boundary. Put those details in the future SDS.

## Next documentation layers

These living documents are the input to, not a replacement for:

- BA-owned Software Requirements Specification.
- Software Design Specification.
- Database Design Specification.
- API contracts.
- Threat model and security test plan.
- QA test cases and UAT scripts.
