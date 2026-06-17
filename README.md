# Zarf Package Helm Chart Action

A GitHub Action to package Helm charts into Zarf packages for air-gapped deployment.

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Zarf%20Package%20Helm%20Chart-blue?logo=github)](https://github.com/marketplace/actions/zarf-package-helm-chart)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Table of Contents

- [Quick Start](#quick-start) - Get running in 5 minutes
- [How It Works](#how-it-works) - Understand the flow
- [Step-by-Step Guide](#step-by-step-guide) - Detailed setup instructions
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
      - 'zarf.yaml'
      - 'chart/**'

jobs:
  package:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Package Helm Chart
        uses: your-username/zarf-package-helm-chart-action@v1
        with:
          chart_path: .
          architecture: amd64
```

**Required file in your repo**: `zarf.yaml` (see [Step 2](#step-2-create-zarfyaml) for example)

---

## How It Works

```
Your Repository                    GitHub Action
┌─────────────────┐               ┌─────────────────┐
│  zarf.yaml      │──────────────▶│  Install Zarf   │
│  (config)       │               │     CLI         │
├─────────────────┤               └────────┬────────┘
│  chart/         │                        │
│  ├── Chart.yaml │                        ▼
│  ├── values.yaml│               ┌─────────────────┐
│  └── templates/ │               │ Package Chart   │
└─────────────────┘               │ for Architecture│
                                └────────┬────────┘
                                         │
                                         ▼
                                ┌─────────────────┐
                                │ Upload Artifact │
                                │ (.tar.zst)      │
                                └─────────────────┘
```

**What this action does:**
1. Installs the Zarf CLI tool
2. Reads your `zarf.yaml` configuration
3. Packages your Helm chart for the specified architecture
4. Uploads the resulting `.tar.zst` file as a GitHub artifact

---

## Step-by-Step Guide

### Step 1: Prepare Your Repository

Your project should have this structure:

```
my-project/
├── .github/
│   └── workflows/          # (we'll create this)
│       └── package.yml
├── chart/                  # Your Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       └── ...
├── zarf.yaml               # (we'll create this)
└── README.md
```

**Prerequisites:**
- A Helm chart (local or remote)
- GitHub repository

### Step 2: Create `zarf.yaml`

Create a `zarf.yaml` file in your repository root. This tells Zarf how to package your chart:

**Option A: Local Helm Chart** (most common)
```yaml
kind: ZarfPackageConfig
metadata:
  name: my-application
  description: My application for air-gapped deployment
  version: 1.0.0

components:
  - name: helm-chart
    required: true
    charts:
      - name: my-chart
        version: 1.0.0
        namespace: default
        localPath: ./chart          # Path to your local chart
        valuesFiles:
          - values.yaml             # Override values file
```

**Option B: Remote Chart from OCI Registry**
```yaml
kind: ZarfPackageConfig
metadata:
  name: my-application
  version: 1.0.0

components:
  - name: helm-chart
    required: true
    charts:
      - name: remote-chart
        version: 2.5.0
        namespace: production
        url: oci://ghcr.io/company/charts
        valuesFiles:
          - values.yaml
```

> **Tip**: See [`example-zarf.yaml`](example-zarf.yaml) in this repo for more examples (Git repos, Helm repositories, etc.)

### Step 3: Create GitHub Workflow

Create `.github/workflows/package.yml` in your repository:

```yaml
name: Package Helm Chart

on:
  # Trigger on changes to relevant files
  push:
    paths:
      - 'zarf.yaml'
      - 'Chart.yaml'
      - 'values.yaml'
      - 'chart/**'
    branches:
      - main
      - master

  # Trigger on pull requests
  pull_request:
    paths:
      - 'zarf.yaml'
      - 'chart/**'

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
        uses: your-username/zarf-package-helm-chart-action@v1
        with:
          # Required settings
          chart_path: .                    # Directory containing zarf.yaml
          architecture: amd64              # or: arm64, both

          # Optional settings
          output_dir: build                # Where to put .tar.zst files
          zarf_version: v0.52.1            # Pin Zarf version (default: latest)
          upload_artifact: true            # Upload as artifact? (default: true)
          artifact_name: zarf-package      # Artifact name
          artifact_retention_days: 30      # How long to keep artifacts

      - name: Show Package Info
        run: |
          echo "Generated package: ${{ steps.package.outputs.package_filename }}"
          ls -lh build/
```

### Step 4: Run the Workflow

1. Push the workflow file to your repository
2. Go to **Actions** tab in your GitHub repo
3. Click on "Package Helm Chart" workflow
4. Click **"Run workflow"** (or wait for automatic trigger on push)

### Step 5: Download the Package

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
| `chart_path` | Path to directory containing `zarf.yaml` | No | `.` |
| `package_name` | Override package name from zarf.yaml | No | *(from zarf.yaml)* |
| `package_version` | Override version from zarf.yaml | No | *(from zarf.yaml)* |
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
      - uses: your-username/zarf-package-helm-chart-action@v1
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
      - uses: your-username/zarf-package-helm-chart-action@v1
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
        uses: your-username/zarf-package-helm-chart-action@v1
        with:
          architecture: both
          output_dir: dist

      - name: Upload to Release
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*.tar.zst
```

### Example 4: Package with Custom Values

```yaml
name: Package for Production
on: [workflow_dispatch]

jobs:
  package:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Create production values file
      - name: Prepare Production Values
        run: |
          cat > prod-values.yaml << 'EOF'
          replicaCount: 3
          resources:
            limits:
              cpu: 1000m
              memory: 1Gi
          EOF

      - name: Package
        uses: your-username/zarf-package-helm-chart-action@v1
        with:
          chart_path: .
          output_dir: dist
          artifact_name: zarf-package-prod
```

---

## Troubleshooting

### Error: "zarf.yaml not found"

**Cause**: The action can't find your zarf.yaml file.

**Solution**: 
- Ensure `zarf.yaml` is in the directory specified by `chart_path`
- Check file is committed to git: `git add zarf.yaml && git commit -m "Add zarf config"`

### Error: "Chart not found"

**Cause**: Zarf can't find your Helm chart.

**Solution**:
- Verify `localPath` in `zarf.yaml` points to correct directory
- Ensure the chart directory contains a valid `Chart.yaml`
- For remote charts, check the URL is accessible

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
- [Open an Issue](https://github.com/your-username/zarf-package-helm-chart-action/issues)

---

## Resources

- [Zarf Documentation](https://zarf.dev/docs/)
- [Zarf GitHub Repository](https://github.com/defenseunicorns/zarf)
- [Helm Charts in Zarf](https://zarf.dev/docs/ref/components/#helm-charts)
- [Zarf Setup Action](https://github.com/zarf-dev/setup-zarf)
- [example-zarf.yaml](example-zarf.yaml) - Full configuration examples
- [.github/workflows/example-usage.yml](.github/workflows/example-usage.yml) - Complete workflow example

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

**Created by David Abrams <9200200@gmail.com>**

Made with ❤️ for the Zarf community
