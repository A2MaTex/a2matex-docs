# A2MaTeX DBML module views

The canonical complete schema is `10-database-design.dbml`. These smaller files are visualization views designed for dbdiagram.io.

- Tables in the module's primary `TableGroup` retain their complete definitions.
- `reference_stubs` contains reduced versions of cross-module tables.
- A stub is not a second physical table and must not be converted into a separate Prisma model or migration.
- Make schema changes in the canonical file first, then regenerate these views.

| Module | File | Full tables | Reference stubs | Size |
|---|---|---:|---:|---:|
| Identity & Access | `10a-identity-access.dbml` | 10 | 0 | 7.4 KiB |
| Publisher & Billing | `10b-publisher-billing.dbml` | 6 | 1 | 5.2 KiB |
| Authoring & Revision | `10c-authoring-revision.dbml` | 8 | 3 | 8.3 KiB |
| Registry & Manifest | `10d-registry-manifest.dbml` | 7 | 2 | 6.5 KiB |
| Build, Review & Signing | `10e-build-review-signing.dbml` | 11 | 5 | 11.9 KiB |
| Marketplace & Release | `10f-marketplace-release.dbml` | 6 | 8 | 8.1 KiB |
| Customer Access | `10g-customer-access.dbml` | 10 | 7 | 12.6 KiB |
| Runtime & Telemetry | `10h-runtime-telemetry.dbml` | 5 | 10 | 9.6 KiB |
| Platform Operations | `10i-platform-operations.dbml` | 5 | 6 | 8.1 KiB |

## Usage

Paste one `.dbml` file at a time into dbdiagram.io. Paste the raw file content without Markdown code fences.

Cross-module relationships terminate at a compact stub. Open the owning module file when you need the full definition of that table.
