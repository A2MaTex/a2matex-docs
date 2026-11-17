# Workflow compilation fixtures

The five cases come from Backend `WorkflowCompilerService.compileWorkflow`
and are covered by Runtime's compilation parity test.

`node_definitions.json` contains the frozen Backend `BASIC_NODE_DEFINITIONS` snapshot, including the
self-describing Set Variable and UUID-based Switch authoring contracts. The manifest pins the full
snapshot by digest so Backend and Runtime compilation fixtures cannot silently use different
definitions.
