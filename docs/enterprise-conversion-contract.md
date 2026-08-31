# Enterprise conversion contract

This document defines what a2d is allowed to claim when translating an Alteryx
workflow. It deliberately separates accepting a workflow, generating candidate
code, executing that code, and proving output equivalence.

## Core pipeline

```text
YXMD -> hardened XML parser -> parsed workflow -> typed IR/DAG
     -> tool adapters -> semantic translators -> Databricks artifacts
     -> Databricks execution -> output reconciliation -> migration report
```

The parser and graph are independent from the adapter inventory. A structurally
valid workflow containing an unknown tool must still be analyzed: a2d preserves
the tool identity, bounded configuration, position, anchors, and connections,
then emits an explicit unsupported-tool record. It must never execute embedded
Python, R, commands, macros, or SDK payloads while parsing an upload.

## Tool capability states

Tool status is evidence-based and is not inferred merely from registry presence.

| Status | Meaning |
|---|---|
| `SUPPORTED` | The configuration used by this node has a deterministic implementation and tested generator semantics. No placeholder or manual step was emitted for the node. |
| `PARTIALLY_SUPPORTED` | A safe subset was translated, but one or more configuration branches, output anchors, or semantic details require review. |
| `UNSUPPORTED` | No safe equivalent was generated. Downstream output is incomplete until the node is replaced. |
| `NOT_APPLICABLE` | The node is intentionally non-transforming for the selected target, such as a visual container. The reason must be recorded. |

A converter that emits passthrough data, `TODO`, `PLACEHOLDER`, or stub logic is
not `SUPPORTED`. Coverage is node-weighted so repeated unsupported nodes cannot
be hidden by counting only unique tool names.

Every partial or unsupported node report must include:

- tool name, plugin name, tool ID, and category;
- bounded, credential-redacted XML configuration;
- capability status and reason;
- generated behavior, if any;
- explicit manual action;
- downstream impact.

## Migration lifecycle states

These states describe different evidence and must never be collapsed into one
generic "success" label.

| State | Required evidence |
|---|---|
| `PARSED` | XML accepted, all nodes and connections discovered, graph diagnostics recorded. |
| `CODE_GENERATED` | One or more target artifacts were produced. This does not imply executable or equivalent code. |
| `REQUIRES_REVIEW` | Generated artifacts contain partial/unsupported nodes, unresolved configuration, semantic warnings, or placeholders. |
| `CONVERSION_VALIDATED` | Static and configured semantic checks passed for every node; no unresolved review items remain. |
| `DATABRICKS_EXECUTION_PENDING` | A specific content-addressed artifact was submitted to a selected workspace. |
| `DATABRICKS_EXECUTION_SUCCEEDED` | Databricks reported a successful terminal run for that exact artifact. |
| `DATABRICKS_EXECUTION_FAILED` | Databricks reported failure or submission/monitoring failed. |
| `OUTPUT_VALIDATION_PASSED` | Expected Alteryx output and Databricks output passed the configured reconciliation rules. |
| `OUTPUT_VALIDATION_FAILED` | Reconciliation found a material difference. |

`CODE_GENERATED`, static syntax validity, conversion coverage, and heuristic
confidence are not proof of Databricks execution or output equivalence.

## Input and environment contract

Source paths are business configuration, not converter-owned data. A workflow
may contain zero, one, or many input tools. Each input node produces one binding
record and must be mapped to a Databricks-accessible table, Unity Catalog volume,
or governed cloud location before execution. Credentials are never copied from
YXMD into generated code or reports.

Local, UNC, and unresolved paths become explicit configuration requirements.
Cloud URIs may be preserved only as references; readiness requires independent
proof that the selected Databricks identity can access them.

## Databricks execution contract

Automatic execution is allowed only for the exact generated artifact authorized
by the conversion response and only when every execution policy check passes.
Submissions are idempotent, user/workspace-bound, and imported into a private
workspace path. API tokens, storage keys, client secrets, and connection strings
must not appear in code, reports, browser storage, or logs.

A local or mocked SDK test is reported as a test, never as a live Databricks run.
Live execution requires an explicitly selected authenticated profile/workspace.

## Output reconciliation contract

Validation records the expected and actual source of each dataset and compares,
as configured:

- row/column counts, names, order, and data types;
- null, distinct, and duplicate counts;
- numeric aggregates and minimum/maximum values;
- deterministic row or aggregate hashes;
- sample records and business-specific assertions.

Missing expected Alteryx output makes reconciliation `NOT_RUN` or
`INCONCLUSIVE`, never passed.

## Development gates

Each phase must report implemented behavior, executed tests, supported and
partial capabilities, unsupported behavior, Databricks execution evidence, known
limitations, and the next bounded phase. Universal semantic coverage is a target,
not a release claim; it can only grow through versioned adapters, representative
YXMD fixtures, golden outputs, and execution-backed tests.
