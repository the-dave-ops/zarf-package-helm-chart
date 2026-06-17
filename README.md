# Zarf Package Helm Chart Action

A GitHub Action to package Helm charts into Zarf packages for air-gapped deployment.

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Zarf%20Package%20Helm%20Chart-blue?logo=github)](https://github.com/marketplace/actions/zarf-package-helm-chart)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Table of Contents

- [Quick Start](#quick-start) - Get running in 5 minutes
- [How It Works](#how-it-works) - Understand the flow
- [Step-by-Step Guide](#step-by-step-guide) - Detailed setup instructions (no zarf.yaml needed!)
- [Inputs & Outputs](#inputs--outputs) - Reference documentation
- [Examples](#examples) - Real-world usage patterns
- [Troubleshooting](#troubleshooting) - Common issues and solutions

---

## Quick Start

Add this to your repository's `.github/workflows/package.yml`:

```yaml
name: Package Helm Chart
on:
  push:
    paths:
      - 'Chart.yaml'
      - 'values.yaml'
      - 'templates/**'

jobs:
  package:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Package Helm Chart
        uses:  the-dave-ops /zarf-package-helm-chart@v1
        with:
          chart_path: .           # Path to your Helm chart
          architecture: amd64
          images: "nginx:1.25"    # Container images to include
```

**Only required**: Your Helm chart with `Chart.yaml` - the action generates `zarf.yaml` automatically!

---

## How It Works

```
Your Repository                    GitHub Action
┌─────────────────┐               ┌─────────────────┐
│  chart/         │──────────────▶│  Install Zarf   │
│  ├── Chart.yaml │               │     CLI         │
│  ├── values.yaml│               └────────┬────────┘
│  └── templates/ │                        │
└─────────────────┘                        ▼
                                ┌─────────────────┐
                                │  Generate       │
                                │  zarf.yaml      │
                                └────────┬────────┘
                                         │
                                         ▼
                                ┌─────────────────┐
                                │ Package Chart   │
                                │ + Images        │
                                │ for Architecture│
                                └────────┬────────┘
                                         │
                                         ▼
                                ┌─────────────────┐
                                │ Upload Artifact │
                                │ (.tar.zst)      │
                                └─────────────────┘
```

**What this action does:**
1. Installs the Zarf CLI tool and yq for YAML parsing
2. Reads your Helm chart's `Chart.yaml` to extract metadata
3. Generates a `zarf.yaml` file automatically with your chart configuration
4. Packages your Helm chart and container images for the specified architecture
5. Uploads the resulting `.tar.zst` file as a GitHub artifact

---

## Step-by-Step Guide

### Step 1: Prepare Your Repository

Your project only needs the Helm chart:

```
my-project/
├── .github/
│   └── workflows/          # (we'll create this)
│       └── package.yml
├── Chart.yaml              # Your Helm chart metadata
├── values.yaml             # Default values
├── templates/              # Helm templates
│   └── ...
└── README.md
```

**Prerequisites:**
- A Helm chart with `Chart.yaml`
- GitHub repository

### Step 2: Create GitHub Workflow

Create `.github/workflows/package.yml` in your repository:

```yaml
name: Package Helm Chart

on:
  # Trigger on changes to chart files
  push:
    paths:
      - 'Chart.yaml'
      - 'values.yaml'
      - 'templates/**'
    branches:
      - main
      - master

  # Trigger on pull requests
  pull_request:
    paths:
      - 'Chart.yaml'
      - 'values.yaml'
      - 'templates/**'

  # Allow manual trigger
  workflow_dispatch:
    inputs:
      architecture:
        description: 'Target architecture'
        required: true
        default: 'amd64'
        type: choice
        options:
          - amd64
          - arm64
          - both

jobs:
  package:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Package Helm Chart with Zarf
        uses:  the-dave-ops /zarf-package-helm-chart@v1
        with:
          # Required settings
          chart_path: .                    # Path to Helm chart directory
          architecture: amd64                # or: arm64, both

          # Optional settings
          namespace: default               # Kubernetes namespace
          images: "nginx:1.25"             # Comma-separated container images
          output_dir: build                # Where to put .tar.zst files
          zarf_version: v0.52.1            # Pin Zarf version (default: latest)
          upload_artifact: true              # Upload as artifact? (default: true)
          artifact_name: zarf-package        # Artifact name
          artifact_retention_days: 30      # How long to keep artifacts

      - name: Show Package Info
        run: |
          echo "Generated package: ${{ steps.package.outputs.package_filename }}"
          ls -lh build/
```

### Step 3: Run the Workflow

1. Push the workflow file to your repository
2. Go to **Actions** tab in your GitHub repo
3. Click on "Package Helm Chart" workflow
4. Click **"Run workflow"** (or wait for automatic trigger on push)

### Step 4: Download the Package

After the workflow completes:
1. Go to the workflow run page
2. Scroll down to **Artifacts** section
3. Download the `zarf-package` artifact
4. Extract and deploy using: `zarf package deploy zarf-package-*.tar.zst`

---

## Inputs & Outputs

### Inputs

| Input | Description | Required | Default |
|-------|-------------|:--------:|---------|
| `chart_path` | Path to Helm chart directory (contains `Chart.yaml`) | **Yes** | - |
| `namespace` | Kubernetes namespace for chart deployment | No | `default` |
| `images` | Comma-separated container images to include (e.g., "nginx:1.25,postgres:15") | No | *(empty)* |
| `values_file` | Path to Helm values file (relative to chart_path) | No | *(empty)* |
| `package_name` | Override Zarf package name (defaults to chart name) | No | *(from Chart.yaml)* |
| `package_version` | Override Zarf package version (defaults to chart version) | No | *(from Chart.yaml)* |
| `architecture` | Target architecture: `amd64`, `arm64`, or `both` | No | `amd64` |
| `output_dir` | Output directory for `.tar.zst` files | No | `build` |
| `zarf_version` | Zarf CLI version to use | No | `latest` |
| `upload_artifact` | Upload package as GitHub artifact | No | `true` |
| `artifact_name` | Name for uploaded artifact | No | `zarf-packages` |
| `artifact_retention_days` | Days to retain artifact | No | `30` |

### Outputs

| Output | Description |
|--------|-------------|
| `package_path` | Full path to generated package |
| `package_filename` | Filename of generated package |
| `architecture` | Architecture of the package |

---

## Examples

### Example 1: Simple Package on Every Push

```yaml
name: Package on Push
on:
  push:
    branches: [main]

jobs:
  package:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses:  the-dave-ops /zarf-package-helm-chart@v1
```

### Example 2: Multi-Architecture Build Matrix

```yaml
name: Build All Architectures
on: [push]

jobs:
  package:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        arch: [amd64, arm64]
    steps:
      - uses: actions/checkout@v4
      - uses:  the-dave-ops /zarf-package-helm-chart@v1
        with:
          architecture: ${{ matrix.arch }}
          artifact_name: package-${{ matrix.arch }}
```

### Example 3: Package and Create Release

```yaml
name: Package and Release
on:
  push:
    tags:
      - 'v*'

jobs:
  package:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Package
        uses:  the-dave-ops /zarf-package-helm-chart@v1
        with:
          architecture: both
          output_dir: dist

      - name: Upload to Release
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*.tar.zst
```

### Example 4: Package with Container Images

```yaml
name: Package with Images
on: [workflow_dispatch]

jobs:
  package:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Package
        uses:  the-dave-ops /zarf-package-helm-chart@v1
        with:
          chart_path: ./chart
          namespace: production
          images: "nginx:1.25,postgres:15,myapp:v1.2.3"
          values_file: values.yaml
          output_dir: dist
          artifact_name: zarf-package-prod
```

### Example 5: Create Release with Download Links

This example creates a GitHub release with the Zarf package attached and includes download instructions in the release notes:

```yaml
name: Release Zarf Package
on:
  push:
    tags:
      - 'v*'

jobs:
  package-and-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Package Helm Chart
        id: package
        uses: the-dave-ops/zarf-package-helm-chart@v1
        with:
          chart_path: .
          architecture: amd64
          images: "nginx:1.25"
          output_dir: dist

      - name: Create Release with Package
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*.tar.zst
          body: |
            ## Zarf Package

            This release includes a Zarf package for air-gapped deployment.

            ### Download

            Download the package from the assets below or directly:
            ```bash
            # Download using gh CLI
            gh release download ${{ github.ref_name }} --pattern "*.tar.zst"
            ```

            ### Deploy

            To deploy this package in an air-gapped environment:
            ```bash
            # Load the package into Zarf
            zarf tools archiver decompress zarf-package-*.tar.zst

            # Or deploy directly
            zarf package deploy zarf-package-*.tar.zst
            ```

            **Package:** ${{ steps.package.outputs.package_filename }}
            **Architecture:** ${{ steps.package.outputs.architecture }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Troubleshooting

### Error: "Chart.yaml not found"

**Cause**: The action can't find your Helm chart.

**Solution**: 
- Ensure `chart_path` points to a directory containing `Chart.yaml`
- Check the chart is committed to git
- Verify the path is correct relative to repository root

### Error: "Package creation failed"

**Cause**: Helm chart has errors or missing dependencies.

**Solution**:
1. Test locally first:
   ```bash
   helm lint ./chart
   helm template ./chart > /dev/null
   zarf tools archiver compress . test.tar.zst
   ```
2. Check Zarf logs in the GitHub Actions output

### Artifact not showing up

**Cause**: `upload_artifact` is false or no package was created.

**Solution**:
- Set `upload_artifact: true` in your workflow
- Check the workflow logs to confirm package was created
- Verify the package file path matches `output_dir` pattern

### Need more help?

- [Zarf Documentation](https://zarf.dev/docs/)
- [Zarf Troubleshooting](https://zarf.dev/docs/troubleshooting/)
- [Open an Issue](https://github.com/ the-dave-ops /zarf-package-helm-chart/issues)

---

## Resources

- [Zarf Documentation](https://zarf.dev/docs/)
- [Zarf GitHub Repository](https://github.com/defenseunicorns/zarf)
- [Helm Charts in Zarf](https://zarf.dev/docs/ref/components/#helm-charts)
- [Zarf Setup Action](https://github.com/zarf-dev/setup-zarf)
- [.github/workflows/example-usage.yml](.github/workflows/example-usage.yml) - Complete workflow example

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

**Created by David Abrams <9200200@gmail.com>**

Made with ❤️ for the Zarf community
