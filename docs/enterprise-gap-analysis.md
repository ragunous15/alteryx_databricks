# Enterprise conversion gap analysis

This is the release claim boundary for the current a2d workspace. It should be
read with the [enterprise conversion contract](enterprise-conversion-contract.md).
The converter accepts and analyzes structurally valid workflows beyond its
implementation inventory, but it does **not** claim universal or lossless
Alteryx-to-Databricks conversion.

## IMPLEMENTED

- Hardened XML parsing discovers workflow nodes, configurations, annotations,
  positions, anchors, and connections without executing embedded code.
- A typed IR/DAG preserves graph structure and dispatches tool behavior through
  versionable registry adapters.
- Capability uses the exact states `SUPPORTED`, `PARTIALLY_SUPPORTED`,
  `UNSUPPORTED`, and `NOT_APPLICABLE`. Partial/stub/template/passthrough behavior
  is not counted as fully supported.
- PySpark and other Databricks candidate artifacts are generated separately from
  Databricks submission. Partial and unsupported nodes remain explicit records
  with redacted configuration, reason, downstream impact, and manual action.
- Static validation, content-bound Databricks run authorization, terminal run
  state, and representative output comparison have distinct lifecycle evidence.
- Generated/reporting surfaces redact credentials and do not copy storage keys,
  secrets, or connection strings from YXMD into artifacts.

### Multi-input behavior

One `.yxmd` is one workflow graph, not one CSV. It may contain zero, one, or many
source nodes. The generator emits one source binding/read for every source node
discovered in the uploaded XML. Therefore, one workflow joining three CSVs
correctly generates three `spark.read` operations. Each binding must be mapped
to a Databricks-accessible table, Unity Catalog Volume, or governed cloud path.

## TESTED

The release gate includes parser/graph/registry, converter/generator/expression,
server/API, security/redaction, and frontend tests. The final local release run
produced:

- Python unit suite: **1,414 passed, 9 skipped**.
- Python integration suite: **273 passed, 1 skipped**.
- Combined Python result: **1,687 passed, 10 skipped, 0 failed**.
- Ruff lint: **passed**. Mypy: **passed for 208 source files**. Git diff
  whitespace check: **passed**.
- Frontend: **32 tests passed**, typecheck **passed**, production build
  **passed**.
- Representative real-YXMD conversion: the supplied one-node
  `AzureDataLakeInput` workflow produced **one source binding and one
  `spark.read`**, passed Python syntax validation, and was correctly classified
  `PARTIALLY_SUPPORTED` because the referenced Azure path still requires
  Databricks identity/permission setup. Databricks execution and output
  equivalence were **not run**.

The runtime matrix currently contains **153 discovered tool/configuration
entries**: 40 `SUPPORTED`, 74 `PARTIALLY_SUPPORTED`, 37 `UNSUPPORTED`, and 2
`NOT_APPLICABLE`. These counts describe the implemented inventory, not all
possible Alteryx versions, tools, plugins, or configurations. Only eight matrix
entries currently carry direct named test-evidence metadata; passing suites do
not turn the remaining entries into proven support.

Mocked Databricks SDK tests and local Spark tests are test evidence only; they
must never be reported as a live Databricks run.

## SUPPORTED

Support is configuration- and target-specific. The runtime tool matrix is the
authoritative inventory and records parser implementation, PySpark
implementation, Databricks implementation evidence, test evidence, status, and
limitations. `SUPPORTED` means the exercised node configuration has a
deterministic implementation with no placeholder or manual step. It does not by
itself prove output equivalence.

## PARTIALLY SUPPORTED

Tools with implemented subsets, approximate semantics, unresolved environment
bindings, target-generator gaps, templates, or manual-review requirements are
reported as `PARTIALLY_SUPPORTED`. Common sources include expression variants,
multi-anchor behavior, database/cloud bindings, datatype nuances, spatial or
predictive library dependencies, and generator-specific limitations.

## UNSUPPORTED

Unknown/custom third-party tools, configurations without a safe deterministic
translation, executable R/Python/command payloads, and unimplemented semantic
branches are fail-closed. They remain in the graph and unsupported report; the
converter does not silently skip them or invent working behavior.

## DATABRICKS STATUS

Code generation does not contact Databricks. Live execution requires an
explicitly selected, authenticated workspace and a content-addressed artifact
that passes the execution policy. The current release report must use one of:

- `DATABRICKS_EXECUTION_NOT_RUN`
- `DATABRICKS_EXECUTION_PENDING`
- `DATABRICKS_EXECUTION_SUCCEEDED`
- `DATABRICKS_EXECUTION_FAILED`

Only a successful terminal workspace run for the exact artifact justifies
`DATABRICKS_EXECUTION_SUCCEEDED`. Output validation is still a separate state.

## KNOWN LIMITATIONS

- There is no exhaustive official XSD/version/plugin/configuration inventory;
  future Alteryx and third-party variants require new fixtures and adapters.
- Custom and iterative macros can be discovered without being semantically
  expanded for every case.
- Expression translation is bounded and fail-closed; supported names do not
  guarantee every signature, coercion, locale, or null edge case.
- Some Alteryx datatypes and file formats lack a lossless Spark equivalent or
  require external libraries and environment configuration.
- Local, UNC, Azure, S3, GCS, JDBC, and ODBC references require independent
  identity, permission, catalog, secret, and network setup in Databricks.
- Representative row/aggregate comparison proves only the supplied datasets
  and rules. Missing Alteryx/golden output means output validation is `NOT_RUN`
  or `INCONCLUSIVE`, never passed.
- Automatic Databricks execution remains intentionally narrower than code
  generation; reviewable output can exist while execution is blocked.
- Batch IDs, conversion history, and assistant sessions are not yet bound to an
  authenticated application principal. Treat the current server as
  local/single-user unless tenancy and authorization are added at deployment.
- The run-status surface does not yet provide complete captured Spark logs and
  output provenance for automated reconciliation.
- The frontend production build currently reports a non-fatal warning for a
  JavaScript chunk larger than 500 kB; this affects load optimization, not
  conversion semantics.

## NEXT STEP

Expand support through versioned adapters plus representative YXMD fixtures,
golden Alteryx outputs, generator tests, and execution-backed Databricks tests.
Prioritize the most common partial/unsupported tool configurations, then add
captured run output/log evidence and automated reconciliation for each proven
configuration. Update this document with final release test results without
converting static confidence or coverage into a correctness claim.
