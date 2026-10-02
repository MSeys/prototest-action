# ProtoTest Evidence

A GitHub Action for [ProtoTest](https://prototest.dev) suites. After the tests run, it:

- compares the pull request's run with the base branch's last green run, and names each test the change
  broke or fixed with the operation where it changed;
- posts that and the failure digest as one pull request comment, with one check annotation per failure;
- uploads the `.prototrace` file, so a reviewer opens the full trace from the comment;
- checks the reports both runs embedded: a unit the base branch covered and this run does not, a changed
  specification or a failed run gate;
- fails the step when a test that passed on the base branch fails now, or when that verdict fails.

```yaml
name: Integration tests

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  actions: read        # download the base branch's trace
  pull-requests: write # comment on the pull request

jobs:
  test:
    runs-on: ubuntu-latest
    env:
      PROTOTEST_RESULTS: ${{ github.workspace }}/TestResults/ProtoTest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x
      - run: dotnet test --configuration Release

      - name: ProtoTest evidence
        if: always()
        uses: MSeys/prototest-action@v1
        with:
          trace: ${{ env.PROTOTEST_RESULTS }}/run.prototrace
```

Run the workflow on pushes to the base branch too: each green run there keeps the trace that the next
pull request compares with.

## What the reviewer sees

```markdown
## ProtoTest run `0979490656fa4a00a6adc2798a14362d`

**2 tests · 1 failed · 1 succeeded**

**Compared with the base branch** (run `ae31c391eb5948d6940afd9468d976c4`)
- broke `orders are listed` at `http.request` `GET /api/orders`

- **FAILED `orders are listed`** (2.01 s)
  - `http.request` `List orders` · failed
  - The API did not answer within 2 seconds.
  - at `tests/OrderTests.cs:42`

[Full trace](https://github.com/you/repo/actions/runs/1/artifacts/2)
```

The job summary carries the full comparison and the run summary. A run with no failures and no changed
outcome posts no comment.

## Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `trace` | (required) | the `.prototrace` file the run wrote |
| `baseline` | `auto` | `auto` downloads the trace artifact of the newest successful run of this workflow on the base branch; a path uses that trace; `none` skips the comparison |
| `artifact-name` | `prototest-trace` | the uploaded artifact, and the one `auto` looks for on the base branch |
| `fail-on-broken` | `true` | fail the step when a test that passed on the base branch fails now |
| `fail-on-regression` | `true` | fail the step when the verdict over the embedded reports has a failing finding |
| `version` | latest | the `ProtoTest.Cli` version to install |
| `source` | | an extra NuGet source for the CLI, such as a folder of pre-release packages |
| `webhook-url`, `webhook-secret`, `webhook-secret-header` | | post the digest JSON to your endpoint as well |
| `dotnet-roll-forward` | `LatestMajor` | lets the .NET 8 tool run on a newer runtime |

Outputs: `artifact-url`, `baseline-run-id`, `broken` and `regressed`.

## Limits

- A comparison matches tests by name. A renamed test reads as one removed and one new test.
- The verdict needs both runs to embed a JSON report (`JsonReportSink`); without one it is skipped.
- `auto` finds a baseline only after the base branch has a successful run of the same workflow that
  uploaded the same artifact name.
- A trace can contain sanitized requests, responses and attachments. Treat the artifact and the comment
  like any other test output.

The [CI page](https://prototest.dev/docs/continuous-integration/) and
[the evidence loop](https://prototest.dev/docs/agent-workflows/loop) cover the rest.
