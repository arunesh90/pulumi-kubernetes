# Release Process for pulumi-kubernetes (extra-headers builds)

This document describes how to build and publish a custom `extra-headers` release of the Pulumi Kubernetes provider from this repository.

## Prerequisites

- Go installed (the repo uses `go 1.24.x` via toolchain in `provider/go.mod`).
- Node.js (LTS is fine) and npm.
- GitHub CLI (`gh`) authenticated with release permissions.
- Pulumi CLI installed (for manual verification).

## Version Naming Convention

Use semantic versioning with a suffix for custom builds:

- Format: `X.Y.Z-extra-headers` (example: `4.25.1-extra-headers`)
- The base version should match the upstream Pulumi Kubernetes provider version.

## Release Checklist Overview

1. Update version strings in all SDK metadata.
2. Build provider binaries and package them for all supported platforms.
3. Build the Node.js SDK package.
4. Verify `dist/` contents.
5. Create a GitHub release with artifacts and release notes.
6. Verify installation instructions.

## 1) Update version strings

Replace the current version with your new version in these files:

- `sdk/nodejs/package.json`
  - Update both the top-level `version` and `pulumi.version`.
- `sdk/python/pulumi_kubernetes/pulumi-plugin.json`
- `sdk/go/kubernetes/pulumi-plugin.json`
- `sdk/go/kubernetes/utilities/pulumiUtilities.go`
  - Update the `semver.MustParse("<version>")` values for `PkgResourceDefaultOpts` and `PkgInvokeDefaultOpts`.
- `sdk/dotnet/version.txt`
- `sdk/dotnet/pulumi-plugin.json`
- `sdk/dotnet/Pulumi.Kubernetes.csproj` (`<Version>`)
- `Makefile` (`PROVIDER_VERSION ?= <version>`)

Use a single, consistent version string everywhere:

```
4.25.1-extra-headers
```

## 2) Build provider binaries

From the repository root:

```bash
# Clean and create dist directory
rm -rf dist && mkdir -p dist

# Generate schema
cd provider
VERSION=4.25.1-extra-headers go generate cmd/pulumi-resource-kubernetes/main.go

# Set version in linker flags (match the provider module path)
LDFLAGS="-X github.com/pulumi/pulumi-kubernetes/provider/v4/pkg/version.Version=4.25.1-extra-headers"

# Build for macOS ARM64
GOOS=darwin GOARCH=arm64 CGO_ENABLED=0 go build -o ../dist/pulumi-resource-kubernetes -ldflags "$LDFLAGS" ./cmd/pulumi-resource-kubernetes
cd ../dist && tar -czvf "pulumi-resource-kubernetes-v4.25.1-extra-headers-darwin-arm64.tar.gz" pulumi-resource-kubernetes && rm pulumi-resource-kubernetes

# Build for Linux AMD64
cd ../provider
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o ../dist/pulumi-resource-kubernetes -ldflags "$LDFLAGS" ./cmd/pulumi-resource-kubernetes
cd ../dist && tar -czvf "pulumi-resource-kubernetes-v4.25.1-extra-headers-linux-amd64.tar.gz" pulumi-resource-kubernetes && rm pulumi-resource-kubernetes

# Build for Linux ARM64
cd ../provider
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -o ../dist/pulumi-resource-kubernetes -ldflags "$LDFLAGS" ./cmd/pulumi-resource-kubernetes
cd ../dist && tar -czvf "pulumi-resource-kubernetes-v4.25.1-extra-headers-linux-arm64.tar.gz" pulumi-resource-kubernetes && rm pulumi-resource-kubernetes
```

Notes:

- The repo uses `go toolchain` in `provider/go.mod`. If Go tries to download the toolchain, allow it or ensure the required version is already installed.
- If you see permissions errors while downloading the toolchain, re-run the command outside any sandbox or with proper permissions.

## 3) Build the Node.js SDK package

This repo does not ship a `yarn.lock`. If Yarn refuses to generate one, use npm for the build.

```bash
cd ./sdk/nodejs

# Clean and build TypeScript
rm -rf bin node_modules
npm install --no-package-lock
npm run build

# Copy required files into the package
cp ../../README.md ../../LICENSE package.json ./bin/

# Create npm package
cd bin
npm pack

# Copy to dist
cp *.tgz ../../dist/
```

Expected package name:

- `pulumi-kubernetes-4.25.1-extra-headers.tgz`

## 4) Verify dist contents

```bash
ls -la ./dist/
```

Expected files:

- `pulumi-resource-kubernetes-v4.25.1-extra-headers-darwin-arm64.tar.gz`
- `pulumi-resource-kubernetes-v4.25.1-extra-headers-linux-amd64.tar.gz`
- `pulumi-resource-kubernetes-v4.25.1-extra-headers-linux-arm64.tar.gz`
- `pulumi-kubernetes-4.25.1-extra-headers.tgz`

## 5) Publish to GitHub

Create the release with detailed notes and attach all artifacts:

```bash
cd /path/to/pulumi-kubernetes

gh release create v4.25.1-extra-headers \
  --title "v4.25.1-extra-headers" \
  --notes "### Added
- \`extraHeaders\` provider configuration option

### Usage
\`\`\`typescript
import * as k8s from \"@pulumi/kubernetes\";

const provider = new k8s.Provider(\"my-provider\", {
  kubeconfig: \"your-kubeconfig\",
  extraHeaders: {
    \"CF-Access-Client-Id\": \"your-client-id\",
    \"CF-Access-Client-Secret\": \"your-client-secret\",
  },
});
\`\`\`

### Installation

**Remove existing plugin:**
\`\`\`bash
rm -rf ~/.pulumi/plugins/resource-kubernetes-*
\`\`\`

**Plugin:**
\`\`\`bash
pulumi plugin install resource kubernetes v4.25.1-extra-headers \\
  --server https://github.com/arunesh90/pulumi-kubernetes/releases/download/v4.25.1-extra-headers
\`\`\`

**Node.js SDK:**
\`\`\`bash
npm install https://github.com/arunesh90/pulumi-kubernetes/releases/download/v4.25.1-extra-headers/pulumi-kubernetes-4.25.1-extra-headers.tgz
\`\`\`

### Platforms
- macOS ARM64 (Apple Silicon)
- Linux AMD64
- Linux ARM64" \
  --repo arunesh90/pulumi-kubernetes \
  dist/pulumi-resource-kubernetes-v4.25.1-extra-headers-darwin-arm64.tar.gz \
  dist/pulumi-resource-kubernetes-v4.25.1-extra-headers-linux-amd64.tar.gz \
  dist/pulumi-resource-kubernetes-v4.25.1-extra-headers-linux-arm64.tar.gz \
  dist/pulumi-kubernetes-4.25.1-extra-headers.tgz
```

## 6) User installation instructions (copy/paste)

### Install Pulumi plugin

```bash
# Remove any existing versions first
rm -rf ~/.pulumi/plugins/resource-kubernetes-*

# Install from GitHub
pulumi plugin install resource kubernetes v4.25.1-extra-headers \
  --server https://github.com/arunesh90/pulumi-kubernetes/releases/download/v4.25.1-extra-headers
```

### Install Node.js SDK

```bash
npm install https://github.com/arunesh90/pulumi-kubernetes/releases/download/v4.25.1-extra-headers/pulumi-kubernetes-4.25.1-extra-headers.tgz
```

## Optional: Local verification

1. Create a small Pulumi program that configures `extraHeaders` on a provider.
2. Run `pulumi up` and verify requests include the custom headers.
3. If headers are not visible, confirm that the correct plugin version is installed and that older plugins were removed.
