# Bino CLI GitHub Action

A GitHub Action to install and run [bino-cli](https://github.com/bino-bi/bino-cli) for linting and building reports.

## Usage

### Lint on Pull Requests

```yaml
name: Lint Reports

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint bino reports
        uses: bino-bi/cli-action@v1
        with:
          command: lint
          fail-on-warnings: 'true'
```

### Build Reports

```yaml
name: Build Reports

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build all reports
        uses: bino-bi/cli-action@v1
        with:
          command: build

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: reports
          path: dist/*.pdf
```

### Build Specific Artefacts

```yaml
- name: Build selected reports
  uses: bino-bi/cli-action@v1
  with:
    command: build
    artefacts: 'monthly-report, weekly-summary'
    exclude-artefacts: 'draft-report'
```

### Lint and Build

```yaml
- name: Lint and build
  uses: bino-bi/cli-action@v1
  with:
    command: both
    fail-on-warnings: 'true'
    out-dir: output
```

### Pin CLI Version

```yaml
- name: Build with specific version
  uses: bino-bi/cli-action@v1
  with:
    command: build
    version: 'v1.2.3'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `version` | Version of bino-cli to install (e.g., `v1.0.0` or `latest`) | No | `latest` |
| `command` | Command to run: `lint`, `build`, or `both` | **Yes** | - |
| `work-dir` | Working directory containing report manifests | No | `.` |
| `cache` | Cache bino CLI and browser runtime between runs | No | `true` |
| `fail-on-warnings` | Exit with non-zero code if lint warnings are found | No | `true` |
| `execute-queries` | Execute dataset queries during lint to validate data | No | `false` |
| `out-dir` | Output directory for build artifacts (relative to work-dir) | No | `dist` |
| `artefacts` | Comma-separated list of artefact names to build | No | (all) |
| `exclude-artefacts` | Comma-separated list of artefact names to exclude | No | (none) |
| `browser` | Browser engine: `chromium`, `firefox`, or `webkit` | No | `chromium` |
| `data-validation` | Data validation mode: `fail`, `warn`, or `off` | No | `warn` |
| `log-format` | Log format: `text` or `json` | No | `text` |
| `no-lint` | Skip lint during build | No | `false` |

## Outputs

| Output | Description |
|--------|-------------|
| `version` | Installed bino-cli version |
| `lint-log` | Path to lint log file (if lint was run) |
| `build-log` | Path to build log file (if build was run) |
| `cache-hit` | Whether the cache was hit (`true` or `false`) |

## Caching

By default, the action caches the bino CLI binary and browser runtime (`~/.local/bin/bino` and `~/.bn/`) to speed up subsequent runs. The cache key is based on:

- Runner OS
- CLI version
- Browser type

On cache hit, the installation and browser setup steps are skipped, which can save significant time (browser downloads are ~100MB+).

To disable caching:

```yaml
- uses: bino-bi/cli-action@v1
  with:
    command: lint
    cache: 'false'
```

## Examples

### Full CI/CD Pipeline

```yaml
name: Reports CI/CD

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint reports
        uses: bino-bi/cli-action@v1
        with:
          command: lint
          fail-on-warnings: 'true'
          execute-queries: 'true'

  build:
    runs-on: ubuntu-latest
    needs: lint
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Build reports
        uses: bino-bi/cli-action@v1
        with:
          command: build
          no-lint: 'true'  # Already linted in previous job

      - name: Upload PDF reports
        uses: actions/upload-artifact@v4
        with:
          name: pdf-reports
          path: dist/*.pdf
          retention-days: 30
```

### Using JSON Logs for Parsing

```yaml
- name: Build with JSON logs
  id: build
  uses: bino-bi/cli-action@v1
  with:
    command: build
    log-format: json

- name: Parse build results
  run: |
    cat "${{ steps.build.outputs.build-log }}" | jq '.artefacts'
```

### Custom Working Directory

```yaml
- name: Build reports from subdirectory
  uses: bino-bi/cli-action@v1
  with:
    command: build
    work-dir: './reports/monthly'
    out-dir: 'output'
```

## License

MIT
