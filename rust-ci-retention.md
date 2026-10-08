# Rust CI prerequisite for the URL-prefix contribution

Feature: https://github.com/BerriAI/litellm/pull/45316
Isolated CI correction: https://github.com/BerriAI/litellm/pull/45364

## Source and failure evidence

Before source: `d8c0e2c7153d82234ec46f0231bf9b9714e7d52a`.
After test/CI source: `8e4e349ca84394cc0fc79518a0086ff8d018df35`.
The entire production Python and Rust source is unchanged by the correction.

The unchanged main branch fails the same retention assertion:
https://github.com/BerriAI/litellm/actions/runs/37772159931

The original test and native schema are byte-identical on main and feature tip `bbc43e9195cd7cb84cb39906e117ca9081bde096`:

- `test_traces.py`: SHA-256 `dc44e4ac679e2af2beb5b181271b622c183c4ba89b1c06755ccba9b4b010fd5c`
- `schema.rs`: SHA-256 `d1175cb734e4b7c0c1e6e9f57d2bddb89a94cde4831131c2d9e6649f84d46c8e`

Production commit `03ba79d261af22800fc3c603d3e0b610f700656b` added Lens feedback retention. The assertion still expected three final statements. Historical migrations also contain retention statements, so the correction retains their configured-day assertion and checks the final four reconciliation requests.

## Real database proof

Use the exact ClickHouse image already pinned by the repository's native test harness:

```sh
docker run --rm -d --name retention-proof -e CLICKHOUSE_SKIP_USER_SETUP=1 -p 127.0.0.1:43048:8123 clickhouse/clickhouse-server:26.9.6.6@sha256:eb4870e7ca7ed70c259eebfcfbee6cf797017f6b5436c2926bbbfe3d4d28486e
```

Build/install the unmodified native wheel using `uv build --python 3.12 --wheel`, then run against the real database with `LITELLM_RUST=1` and `LITELLM_LOCAL_MODEL_COST_MAP=True`:

```python
import asyncio
from litellm.rust_bridge._native import NativeTraceConfig, NativeTraceStorage

async def provision():
    storage = NativeTraceStorage(NativeTraceConfig("trace_test", "http://localhost:43048", 7, 65536))
    await storage.ensure_schema()
    print("Native schema provisioned with seven-day retention")

asyncio.run(provision())
```

Observed output, before and after the test-only correction:

```text
Native schema provisioned with seven-day retention
```

Query the actual table definitions:

```sh
curl -fsS http://localhost:43048/ --data-binary "SELECT name, position(create_table_query, 'toIntervalDay(7)') > 0 AS seven_day_ttl FROM system.tables WHERE database = 'trace_test' AND name IN ('otel_traces','agent_traces_by_key','spend_logs','lens_feedback') ORDER BY name FORMAT TSV"
```

Actual output:

```text
agent_traces_by_key  1
lens_feedback       1
otel_traces         1
spend_logs          1
```

Local native-extension verification: the original retention assertion fails; the corrected schema-setup and writer-credential-rejection tests both pass. No models, provider credentials, inference calls, skips or retries are added.

## Runner artifact limit

Feature-matrix job https://github.com/BerriAI/litellm/actions/runs/37775491302/job/113305424494 fails with:

```text
rustc-LLVM ERROR: IO failure on output stream: No space left on device
error: could not compile `litellm-secrets` (test "resolution")
No space left on device (os error 28)
```

The correction scopes `CARGO_PROFILE_DEV_DEBUG=0` and `CARGO_PROFILE_TEST_DEBUG=0` to the Rust test job. These standard Cargo profile overrides omit debugger metadata while preserving debug assertions, overflow checks, every test and every feature combination. Production release builds are unaffected. The full CI matrix must finish successfully before this draft is ready for maintainer review.
