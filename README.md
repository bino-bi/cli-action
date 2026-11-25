# Bino CLI GitHub Action

A GitHub Action to install and run [bino-cli](https://github.com/bino-bi/bino-cli) for linting and building reports.

It runs on Linux and macOS runners.

## Pin the bino version

Always set `version`. With `latest`, every new bino release can change your build without a change in your repository, so a report from last month cannot be built again in the same way. The action prints a warning when `version` is `latest`, and the warning names the release it used.

The downloaded binary is checked against the `checksums.txt` of that release.

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
        uses: bino-bi/cli-action@v2
        with:
          version: v0.120.0
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
        uses: bino-bi/cli-action@v2
        with:
          version: v0.120.0
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
  uses: bino-bi/cli-action@v2
  with:
    version: v0.120.0
    command: build
    artefacts: 'monthly-report, weekly-summary'
    exclude-artefacts: 'draft-report'
```

### Lint and Build

```yaml
- name: Lint and build
  uses: bino-bi/cli-action@v2
  with:
    version: v0.120.0
    command: both
    fail-on-warnings: 'true'
    out-dir: output
```

### Pin the Template Engine

Pin the engine with `engine-version` in your `bino.toml`. bino then uses that engine on every machine, and `bino build` downloads it when it is missing. The action warns when `bino.toml` has no pin.

For a project without a pin, set the `engine-version` input. The action then installs that engine before the build:

```yaml
- name: Build with a fixed engine
  uses: bino-bi/cli-action@v2
  with:
    version: v0.120.0
    command: build
    engine-version: v1.0.0-next.87
```

### Projects with Registry Packages

When the work dir has a `bino.lock`, the action runs `bino registry install` before lint and build, so the packages are restored exactly as locked. Set `registry-install: 'false'` to skip this step.

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `version` | bino-cli release to install, e.g. `v0.120.0`. `latest` works but prints a warning | No | `latest` |
| `command` | Command to run: `lint`, `build`, or `both` | **Yes** | - |
| `work-dir` | Working directory containing report manifests | No | `.` |
| `cache` | Cache the bino binary, Chrome headless shell and template engine between runs | No | `true` |
| `engine-version` | Template engine version to install for build. Leave it empty when `bino.toml` pins `engine-version` | No | (none) |
| `registry-install` | Run `bino registry install` when the work dir has a `bino.lock` | No | `true` |
| `fail-on-warnings` | Exit with non-zero code if lint warnings are found | No | `true` |
| `execute-queries` | Execute dataset queries during lint to validate data | No | `false` |
| `out-dir` | Output directory for build artifacts and log files (relative to work-dir) | No | `dist` |
| `artefacts` | Comma-separated list of artefact names to build | No | (all) |
| `exclude-artefacts` | Comma-separated list of artefact names to exclude | No | (none) |
| `data-validation` | Data validation mode: `fail`, `warn`, or `off` | No | `warn` |
| `log-format` | Format of the lint and build log files: `text` or `json` | No | `text` |
| `no-lint` | Skip lint during build | No | `false` |

`command: lint` does not download Chrome or the template engine. Lint does not need them.

## Outputs

| Output | Description |
|--------|-------------|
| `version` | Installed bino-cli version, as printed by `bino version` |
| `lint-log` | Path to lint log file (if lint was run) |
| `build-log` | Path to build log file (if build was run) |
| `cache-hit` | Whether the bino binary came from the cache (`true` or `false`; empty with `cache: 'false'`) |

## Caching

By default, the action uses two caches:

- The bino binary (`~/.local/bin/bino`), keyed by OS, architecture and bino version. With `version: latest` the key uses the release that `latest` resolved to, so a new release never reuses an old binary.
- For `build` and `both` only: Chrome headless shell (`~/.bino/chrome-headless-shell`) and the template engines (`~/.bino/cdn`). The key contains the Chrome version, the `engine-version` input and a hash of `bino.toml`, so a new engine pin is saved into a new cache entry.

`bino setup` always installs the newest stable Chrome; bino has no option to fix the Chrome version. So two builds with the same bino version and engine pin can still render with different Chrome versions. If you need the renderer fixed too, build with the container image `ghcr.io/bino-bi/bino-cli:<version>`.

To disable caching:

```yaml
- uses: bino-bi/cli-action@v2
  with:
    version: v0.120.0
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
        uses: bino-bi/cli-action@v2
        with:
          version: v0.120.0
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
        uses: bino-bi/cli-action@v2
        with:
          version: v0.120.0
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
  uses: bino-bi/cli-action@v2
  with:
    version: v0.120.0
    command: build
    log-format: json

- name: Parse build results
  run: jq '.artefacts' "${{ steps.build.outputs.build-log }}"
```

### Custom Working Directory

```yaml
- name: Build reports from subdirectory
  uses: bino-bi/cli-action@v2
  with:
    version: v0.120.0
    command: build
    work-dir: './reports/monthly'
    out-dir: 'output'
```

## Upgrading from v1

- The `browser` input is gone. bino renders with Chrome headless shell only.
- `log-format` now sets the format of the lint and build log files.
- `engine-version` and `registry-install` are new.
- Windows runners are not supported.

## Development

`.github/workflows/test.yml` runs the action against the small reports in `test/`: on every pull request, and every week with the newest bino release. When that weekly run fails, a new bino release has broken the action.

## License

MIT
