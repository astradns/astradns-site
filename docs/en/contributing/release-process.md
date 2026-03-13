# Release Process

## Versioning

AstraDNS follows [Semantic Versioning](https://semver.org/): `vX.Y.Z`

- **Major (X):** Breaking API changes
- **Minor (Y):** New features, backward-compatible
- **Patch (Z):** Bug fixes

## Pre-Release Checklist

```bash
# 1. Run all tests
make test

# 2. Run linter
make lint

# 3. Run e2e tests
make test-e2e

# 4. Run integration tests (multi-K8s version)
make test-integration-matrix

# 5. Validate SLOs
make test-slo

# 6. Validate release artifacts
make release-check
```

## Release Steps

### 1. Update Version

Update `Chart.yaml` version and appVersion:

```yaml
version: 0.2.5
appVersion: "0.2.5"
```

### 2. Tag the Release

```bash
git tag -a v0.2.5 -m "Release v0.2.5"
git push origin v0.2.5
```

### 3. Automated Pipeline

The tag push triggers the `release-package.yml` workflow which:

1. Runs `make release-check`
2. Runs `make test` and `make lint`
3. Packages the Helm chart (`make package-chart`)
4. Uploads the chart artifact

### 4. Create GitHub Release

Create a GitHub release from the tag with:

- Changelog summarizing changes since the last release
- Link to the Helm chart artifact
- Upgrade instructions (if any)
- Known issues (if any)

## Helm Chart Packaging

```bash
# Package the chart
make package-chart

# Output: dist/astradns-0.2.5.tgz
```

## Rollback

If a release needs to be reverted:

```bash
# Helm rollback
helm rollback astradns <previous-revision> -n astradns-system

# Verify
kubectl get pods -n astradns-system
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```
