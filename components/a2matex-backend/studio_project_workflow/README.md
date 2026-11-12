# Studio Project Workflow — Backend Specs

This folder is the compact implementation reference for the Backend portion of Studio workflow.
The database overview remains in
[`../db_design/02_studio_project_workflow.md`](../db_design/02_studio_project_workflow.md).

| Feature                   | Status                                                         | Spec                                               | Backend phase                     |
| ------------------------- | -------------------------------------------------------------- | -------------------------------------------------- | --------------------------------- |
| Node Catalog              | Implemented and smoke-tested                                   | [`node_catalog.md`](node_catalog.md)               | `phase1.md`, checklist phases 1–4 |
| Project Module            | Implemented, migrated, permissioned; multi-user checks pending | [`project_module.md`](project_module.md)           | `phase2.md`, checklist phase 5    |
| Project Draft             | Backend complete; migrated, permissioned, and live-verified    | [`project_draft.md`](project_draft.md)             | `phase3.md`, checklist phase 6    |
| Basic Workflow Validation | Backend complete and approved                                  | [`workflow_validation.md`](workflow_validation.md) | `phase4.md`, checklist phase 7    |
| Workflow Revision         | Source snapshot core implemented; asynchronous compilation finalization pending | [`workflow_revision.md`](workflow_revision.md) | `phase5.md` |

Decision precedence for these features:

1. An approved feature spec in this folder.
2. The matching current backend `phase*.md` checklist.
3. The database overview.
4. Older drafts or reference specs.

The current MVP deliberately has no node-level version. Any future introduction of immutable node
versions requires a new reviewed spec and migration; it must not be inferred from older documents.
