# Workflow compilation fixtures

The five cases come from Backend `WorkflowCompilerService.compileWorkflow`
and are covered by Runtime's compilation parity test.

`node_definitions.json` contains the frozen `BASIC_NODE_DEFINITIONS` from
Backend commit `8d21b3218f158ad5266a2870fbc53e119e2f57d9` plus the staged
`feat/node-binding-kind` definition changes. The staged diff for
`src/routes/node-catalog/definitions` has SHA-256
`4465ebcd943f2a9b75af4d1595669dc5eacaddaaae81c29d0d7b6b4832978d2d`.
The manifest pins the full snapshot by digest.
