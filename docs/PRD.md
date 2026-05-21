# Product Requirements Document: `rules_kind`

**Version:** 0.1.0  
**Status:** Draft  
**Last Updated:** 2026-05-21

---

## 1. Executive Summary

`rules_kind` is a Bazel ruleset for KinD (Kubernetes in Docker) based integration testing and local development. It provides hermetic toolchains for `kind` and `kubectl`, a `kind_object` rule for loading container images and applying manifests to a KinD cluster, and a `kind_test` rule for declaring integration tests with proper dependency tracking.

The project is the spiritual successor to the archived `bazelbuild/rules_k8s`, but deliberately scoped to KinD and designed for the modern Bazel ecosystem (bzlmod, Bazel 8+, `rules_oci`). By eliminating manifest substitution, registry interaction, and general-purpose K8s deployment concerns, `rules_kind` targets a tiny, maintainable surface area (~1,500–2,000 LOC) that avoids the architectural pitfalls that killed its predecessor.

**Initial home:** Personal/company GitHub org, with intent to migrate to `bazel-contrib` once stable.  
**Distribution:** Bazel Central Registry (BCR).

---

## 2. Problem Statement

### Historical Context: The Demise of `rules_k8s`

`bazelbuild/rules_k8s` was archived on February 6, 2024, after years of declining maintenance. The root causes:

1. **Hard dependency on `rules_docker`** — which itself entered maintenance mode, creating a cascading failure.
2. **Complex Go resolver** — a 540-line program tightly coupled to `rules_docker`'s image format performed manifest substitution (rewriting image tags to digests). No one volunteered to port it to `rules_oci`.
3. **Bazel compatibility rot** — broke at HEAD with no active maintainers.
4. **Overly broad scope** — attempted to be a general-purpose K8s deployment tool within the build system, a philosophical fit that the community increasingly questioned.

### The Gap Today

Teams using Bazel to build container images and wanting to run integration tests against a KinD cluster have no maintained, first-class Bazel integration. Current workarounds involve shell scripts, Makefile orchestration, or bespoke rule implementations — all of which break Bazel's dependency graph, caching, and change detection.

### What `rules_kind` Solves

- Declares the dependency from integration tests → deployed services → container images in Bazel's build graph.
- Enables `bazel-diff` and remote caching to correctly identify which integration tests are affected by source changes.
- Provides hermetic `kind` and `kubectl` binaries via toolchains.
- Offers a simple `bazel run :my_service.deploy` workflow for CI and local development.

---

## 3. Goals & Non-Goals

### Goals

| # | Goal |
|---|------|
| G1 | Provide a simple, 3-4 attribute rule (`kind_test`) that connects container images to integration tests through Bazel's dependency graph. |
| G2 | Ship hermetic `kind` and `kubectl` toolchains that work across linux/darwin × amd64/arm64. |
| G3 | Support `rules_oci` tarballs as the primary image input format, with a generic interface that accepts any Docker-format tarball. |
| G4 | Publish to the Bazel Central Registry (BCR) for easy consumption via bzlmod. |
| G5 | Maintain a surface area small enough (~1,500–2,000 LOC) for long-term maintainability by 1-2 people. |
| G6 | Enable both CI workflows (automated create→deploy→test→teardown) and local dev workflows (Tilt, manual deploy). |

### Non-Goals (v0.1)

| # | Non-Goal | Rationale |
|---|----------|-----------|
| NG1 | Cluster lifecycle management (create/delete) | BYOC model. Cluster lifecycle is infrastructure orchestration, not build system concern. |
| NG2 | Manifest substitution / templating | Eliminated the complexity that killed `rules_k8s`. KinD tags match tarballs directly. |
| NG3 | `rules_img` support | Future work. Interface accepts generic tarballs, so `rules_img` can be added later without API changes. |
| NG4 | Self-contained test macro (create→deploy→test→teardown) | Requires cluster lifecycle. Deferred. |
| NG5 | Tilt integration documentation | Tilt can consume the same artifacts. Docs are future work. |
| NG6 | Separate `.load` / `.apply` run targets | Premature granularity. `.deploy` does both. |
| NG7 | Namespace attribute on rules | Users declare namespaces in manifests. `kubectl apply` routes correctly via `metadata.namespace`. |
| NG8 | WORKSPACE support | bzlmod only. Bazel 8+ minimum. |

---

## 4. Target Users

### Primary: Standard Bazel Consumers

Teams that use Bazel for builds but are relatively standard consumers — they run `bazel build` and `bazel test`, consume published rulesets, but don't author custom Starlark rules.

**Persona:** Backend team building microservices. They have `rules_oci` producing images, KinD for local/CI testing, and want the simplest possible integration.

**What they need:**
- Copy-paste example that works in 10 minutes
- 3-4 attributes to configure
- `bazel test :my_integration_test` just works (given cluster exists)

### Secondary: Platform/Infra Teams

Teams deep in Bazel monorepos who might wrap `rules_kind` in their own macros, integrate with custom CI systems, or extend the toolchains.

**What they need:**
- Extensible toolchain model
- Clean provider interfaces for rule composition
- Predictable, debuggable behavior

---

## 5. Design Principles

1. **Simple happy path, extensible for power users** — `kind_test` with 3-4 attributes covers 80% of use cases. Toolchain escape hatches and provider-based composition handle the rest.

2. **No magic** — No resolvers, no manifest rewriting, no registry interaction. What you write in your manifest is what gets applied.

3. **Generic image interface** — Accept any Docker-format tarball. Don't couple to `rules_oci` internals. This is the lesson from `rules_k8s` → `rules_docker` coupling.

4. **BYOC (Bring Your Own Cluster)** — Rules don't manage cluster lifecycle. This separates concerns cleanly and avoids the trap of trying to orchestrate infrastructure from the build system.

5. **Dependency graph correctness** — The primary value proposition is that Bazel's graph correctly tracks source → image → deployment → test dependencies, enabling accurate caching and change detection.

6. **Tiny surface area** — Every line of code is a maintenance liability. Target 1,500–2,000 LOC total. Resist feature creep.

---

## 6. Technical Architecture

### Layer Diagram

```
┌─────────────────────────────────────────────────────────┐
│                     User BUILD files                     │
│         kind_object(...) / kind_test(...)                │
├─────────────────────────────────────────────────────────┤
│                      Rule Layer                          │
│   kind_object: generates .deploy / .delete scripts      │
│   kind_test: wires objects → test binary + env vars     │
├─────────────────────────────────────────────────────────┤
│                   Toolchain Layer                        │
│   KindToolchainInfo: path to `kind` binary              │
│   KubectlToolchainInfo: path to `kubectl` binary        │
├─────────────────────────────────────────────────────────┤
│                 Module Extension Layer                   │
│   kind.toolchain() / kind.kubectl_toolchain()           │
│   Downloads platform-specific binaries                  │
├─────────────────────────────────────────────────────────┤
│                   External Deps                          │
│   rules_oci (user's image build) │ KinD cluster (BYOC)  │
└─────────────────────────────────────────────────────────┘
```

### Key Architectural Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Image input format | Docker-format tarball (list of labels) | `oci_load` produces this by default; `kind load image-archive` consumes it directly. Zero conversion. |
| Tag management | Baked into tarball | No `{"tag": ":target"}` dict. Tags in manifests match tags in tarballs. No resolver needed. |
| Cluster reference | String name | BYOC model. Name is sufficient for `kind load --name` and kubeconfig context. |
| Manifest ordering | List order = apply order | User controls topology (namespace before deployment before service). |
| Kubeconfig | Inherited from environment | KinD writes to `~/.kube/config`. Optional `kubeconfig` label as escape hatch. |
| Test networking | User-managed | Users configure `extraPortMappings` in KinD config or NodePort services. Test binary hits `localhost:<port>`. |

### Provider Interface

```python
KindObjectInfo = provider(
    fields = {
        "cluster": "String: KinD cluster name",
        "images": "List[File]: Docker-format tarball files",
        "manifests": "List[File]: Kubernetes manifest files",
    },
)
```

---

## 7. Rule API Reference

### `kind_object`

Represents a set of container images and Kubernetes manifests to be deployed to a KinD cluster.

```python
kind_object(
    name = "my_service",
    cluster = "my-dev-cluster",
    images = [":image_tarball"],
    manifests = [
        ":namespace.yaml",
        ":deployment.yaml",
        ":service.yaml",
    ],
)
```

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | `string` | ✅ | Rule name. Generates `{name}.deploy` and `{name}.delete` run targets. |
| `cluster` | `string` | ✅ | KinD cluster name (passed to `kind load --name` and `kubectl --context`). |
| `images` | `list[label]` | ✅ | Docker-format tarball files to load into the cluster. Tags baked into tarballs. |
| `manifests` | `list[label]` | ✅ | Kubernetes manifest files, applied in order via `kubectl apply -f`. |

**Generated targets:**

- **`{name}.deploy`** (`bazel run`): Loads all images into the KinD cluster via `kind load image-archive`, then applies all manifests in order via `kubectl apply -f`.
- **`{name}.delete`** (`bazel run`): Deletes all manifests via `kubectl delete -f` (reverse order).

### `kind_test`

Declares an integration test that depends on deployed `kind_object` targets. Creates a proper DAG edge so that changes to source code → images → kind_objects → tests are tracked by Bazel.

```python
kind_test(
    name = "my_service_test",
    cluster = "my-dev-cluster",
    objects = [":my_service"],
    test = ":integration_test_binary",
    tags = ["exclusive"],
    size = "large",
)
```

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | `string` | ✅ | Test target name. |
| `cluster` | `string` | ✅ | KinD cluster name. |
| `objects` | `list[label]` | ✅ | `kind_object` targets this test depends on. Creates DAG edges for caching/change detection. |
| `test` | `label` | ✅ | Test binary to execute. Responsible for all assertions. |
| `kubeconfig` | `label` | ❌ | Optional path to kubeconfig file. Defaults to environment kubeconfig (`~/.kube/config`). |
| `tags` | `list[string]` | ❌ | Standard Bazel test tags. Recommend `["exclusive"]` for integration tests. |
| `size` | `string` | ❌ | Standard Bazel test size. Recommend `"large"`. |

**Runtime environment provided to test binary:**

| Variable | Source | Description |
|----------|--------|-------------|
| `KIND_CLUSTER_NAME` | `cluster` attribute | The KinD cluster name for kubectl context selection. |
| `KUBECTL` | kubectl toolchain | Path to the hermetic kubectl binary. |

**Important:** `kind_test` does NOT execute the deploy. It only declares the dependency relationship. Deploy is a separate `bazel run :obj.deploy` step (typically in CI script or done manually for local dev).

### Toolchains

#### `kind` Toolchain

```python
# MODULE.bazel
kind = use_extension("@rules_kind//kind:extensions.bzl", "kind")
kind.toolchain(kind_version = "0.31.0")
```

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `kind_version` | `string` | ✅ | KinD release version to download. |

**Escape hatch:** If `kind_version` is omitted or set to `"system"`, uses `kind` from `$PATH`.

**Platforms:** linux_amd64, linux_arm64, darwin_amd64, darwin_arm64.

#### `kubectl` Toolchain

```python
# MODULE.bazel
kind.kubectl_toolchain(kubectl_version = "1.31.0")
```

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `kubectl_version` | `string` | ✅ | kubectl release version to download. |

**Escape hatch:** If `kubectl_version` is omitted or set to `"system"`, uses `kubectl` from `$PATH`.

**Platforms:** linux_amd64, linux_arm64, darwin_amd64, darwin_arm64.

---

## 8. User Workflows

### CI Workflow

The standard CI flow for integration testing:

```bash
# 1. Create cluster (outside Bazel)
kind create cluster --name test-cluster

# 2. Deploy services (Bazel run target)
bazel run //services/my-app:my_service.deploy

# 3. Run integration tests (Bazel test)
bazel test //services/my-app:my_service_test

# 4. Teardown (outside Bazel)
kind delete cluster --name test-cluster
```

**GitHub Actions example:**

```yaml
jobs:
  integration-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: engineerd/setup-kind@v0.6.0
        with:
          name: test-cluster
      - run: bazel run //services/my-app:my_service.deploy
      - run: bazel test //services/my-app:my_service_test
      - if: always()
        run: kind delete cluster --name test-cluster
```

### Local Development with Tilt

Tilt owns the deploy loop but can consume `rules_kind` artifacts:

| Concern | Owner |
|---------|-------|
| Image build | `rules_oci` (invoked by Tilt via `custom_build`) |
| Image load | Tilt (`kind load image-archive`) |
| Manifest apply | Tilt (`k8s_yaml`) |
| File watch + rebuild | Tilt |
| Integration test | `kind_test` (via `bazel test` separately) |

In this mode, `kind_test` still provides value: it declares the correct dependency graph for `bazel test` and `bazel-diff`, even though Tilt handles the deploy loop during development.

### Consumption Mode Matrix

| Mode | `.deploy`/`.delete` | `kind_test` | Image tarballs | Resolved manifests |
|------|---------------------|-------------|----------------|--------------------|
| CI | ✅ | ✅ | ✅ (via .deploy) | ✅ (via .deploy) |
| Tilt dev | ❌ (Tilt owns deploy) | ✅ | ✅ (Tilt builds via Bazel) | ✅ (Tilt can consume) |

---

## 9. Integration with `rules_oci`

### Image Pipeline

```
Source code
    ↓  (oci_image)
OCI image
    ↓  (oci_load, format = "docker")
Docker-format tarball (with baked-in tag)
    ↓  (kind_object.images)
kind load image-archive --name <cluster>
    ↓
Image available in KinD cluster
    ↓  (kubectl apply -f manifest)
Pod pulls image by tag (local, no registry)
```

### Why No Format Conversion Is Needed

`rules_oci`'s `oci_load` rule with `format = "docker"` (the default) produces a Docker V2 tarball. This is exactly the format that `kind load image-archive` expects. The tag specified in `repo_tags` is baked into the tarball and matches what's referenced in Kubernetes manifests. No resolver, no digest rewriting, no registry interaction.

### Example Integration

```python
load("@rules_oci//oci:defs.bzl", "oci_image", "oci_load")
load("@rules_kind//kind:defs.bzl", "kind_object", "kind_test")

# Build the image
oci_image(
    name = "app_image",
    base = "@distroless_base",
    entrypoint = ["/app"],
    tars = [":app_layer"],
)

# Produce a Docker-format tarball with a known tag
oci_load(
    name = "app_tarball",
    image = ":app_image",
    repo_tags = ["my-app:dev"],
    format = "docker",
)

# Declare the deployment
kind_object(
    name = "app",
    cluster = "test-cluster",
    images = [":app_tarball"],
    manifests = ["deployment.yaml"],  # references image: my-app:dev
)

# Declare the integration test
kind_test(
    name = "app_integration_test",
    cluster = "test-cluster",
    objects = [":app"],
    test = ":integration_test_binary",
    size = "large",
    tags = ["exclusive"],
)
```

---

## 10. Implementation Milestones

### Phase 1: Toolchains

**What's built:** Hermetic `kind` and `kubectl` toolchain infrastructure — module extensions that download platform-specific binaries and register toolchains.

**Deliverables:**
- `//kind:extensions.bzl` — module extension with `kind.toolchain()` and `kind.kubectl_toolchain()` tags
- `KindToolchainInfo` and `KubectlToolchainInfo` providers
- Platform-specific repository rules for binary download (linux/darwin × amd64/arm64)
- System path escape hatch for both tools
- `MODULE.bazel` with proper `bazel_dep` declarations

**Acceptance Criteria:**
- [ ] `bazel build @kind_toolchains//:all` succeeds on all 4 platform combinations
- [ ] A test target can resolve `KindToolchainInfo` and access the `kind` binary path
- [ ] A test target can resolve `KubectlToolchainInfo` and access the `kubectl` binary path
- [ ] System escape hatch works when version is set to `"system"`
- [ ] Starlark analysis tests validate toolchain resolution
- [ ] Downloaded binaries are cached (not re-downloaded on every build)

**Estimated effort:** 3–5 days  
**Dependencies:** None (foundational)

---

### Phase 2: `kind_object` Rule

**What's built:** The `kind_object` rule that generates `.deploy` and `.delete` run targets using the toolchains from Phase 1.

**Deliverables:**
- `//kind:defs.bzl` — exports `kind_object` rule
- `KindObjectInfo` provider for downstream consumption
- Shell script generation for `.deploy` (load images + apply manifests)
- Shell script generation for `.delete` (delete manifests in reverse order)
- Proper `runfiles` handling for images and manifests

**Acceptance Criteria:**
- [ ] `bazel run :my_object.deploy` loads images via `kind load image-archive --name <cluster>` and applies manifests via `kubectl apply -f`
- [ ] `bazel run :my_object.delete` deletes manifests via `kubectl delete -f`
- [ ] Multiple images and manifests are handled correctly
- [ ] Manifests are applied in declared order
- [ ] `KindObjectInfo` provider is populated and accessible to downstream rules
- [ ] Starlark analysis tests validate rule attributes, provider output, and generated actions
- [ ] Error messages are clear when toolchain is not configured

**Estimated effort:** 5–7 days  
**Dependencies:** Phase 1 (toolchains must be resolvable)

---

### Phase 3: `kind_test` Rule

**What's built:** The `kind_test` rule that wires `kind_object` dependencies to a test binary with proper environment variables.

**Deliverables:**
- `kind_test` rule in `//kind:defs.bzl`
- Environment variable injection (`KIND_CLUSTER_NAME`, `KUBECTL`)
- DAG edge creation from `objects` to test target
- Optional `kubeconfig` label support
- Proper data dependency handling (test binary has access to kubectl via runfiles)

**Acceptance Criteria:**
- [ ] `bazel test :my_test` executes the test binary with `KIND_CLUSTER_NAME` set correctly
- [ ] `KUBECTL` env var points to the hermetic kubectl binary from the toolchain
- [ ] Changes to source files that feed into `kind_object` images cause `kind_test` to be re-run (DAG correctness)
- [ ] `bazel-diff` correctly identifies affected tests when image sources change
- [ ] Optional `kubeconfig` attribute overrides default behavior
- [ ] Standard test attributes (`tags`, `size`, `timeout`) work as expected
- [ ] Starlark analysis tests validate provider wiring and environment setup

**Estimated effort:** 3–5 days  
**Dependencies:** Phase 2 (`kind_test` consumes `KindObjectInfo` from `kind_object`)

---

### Phase 4: End-to-End Example & Documentation

**What's built:** A complete, working example in the repository that demonstrates the full pipeline with `rules_oci`, plus user documentation.

**Deliverables:**
- `examples/basic/` — complete example with `oci_image` → `oci_load` → `kind_object` → `kind_test`
- Simple Go or shell test binary that validates a deployed service responds
- `README.md` — quickstart, API reference, CI integration guide
- GitHub Actions CI workflow that runs e2e tests (create cluster → deploy → test → teardown)
- e2e tests tagged `manual` (not run in analysis-only CI)

**Acceptance Criteria:**
- [ ] `examples/basic/` builds and tests successfully against a real KinD cluster
- [ ] GitHub Actions CI creates a KinD cluster and runs the full e2e flow
- [ ] CI passes on ubuntu-latest (linux_amd64)
- [ ] README provides copy-paste setup that works within 10 minutes for a new user
- [ ] All example code compiles and is tested in CI (no stale examples)
- [ ] e2e tests are tagged `manual` and only run in CI, not during `bazel test //...`

**Estimated effort:** 5–7 days  
**Dependencies:** Phases 1-3 (complete rule implementation)

---

### Phase 5: BCR Submission & Release

**What's built:** Packaging, versioning, and publication to the Bazel Central Registry.

**Deliverables:**
- `MODULE.bazel` with correct metadata (version, compatibility_level, etc.)
- BCR metadata files (`source.json`, `MODULE.bazel`, `patches/` if needed)
- GitHub Release with changelog
- BCR pull request
- Semantic versioning: `0.1.0`

**Acceptance Criteria:**
- [ ] `bazel_dep(name = "rules_kind", version = "0.1.0")` resolves from BCR
- [ ] A fresh project can consume `rules_kind` from BCR with only `MODULE.bazel` configuration
- [ ] BCR presubmit tests pass
- [ ] GitHub Release published with changelog and tagged source archive
- [ ] Documentation references BCR as the installation method

**Estimated effort:** 2–3 days  
**Dependencies:** Phase 4 (must have working example and CI to demonstrate for BCR review)

---

### Timeline Summary

| Phase | Effort | Cumulative |
|-------|--------|-----------|
| 1. Toolchains | 3–5 days | 3–5 days |
| 2. kind_object | 5–7 days | 8–12 days |
| 3. kind_test | 3–5 days | 11–17 days |
| 4. E2E + Docs | 5–7 days | 16–24 days |
| 5. BCR Release | 2–3 days | 18–27 days |
| **Total** | | **~4–5 weeks** |

---

## 11. Future Roadmap (v0.2+)

Items explicitly deferred from v0.1, ordered by likely priority:

| Version | Feature | Description |
|---------|---------|-------------|
| v0.2 | Cluster lifecycle | `kind_cluster` rule or macro for create/delete. Enables self-contained test macro. |
| v0.2 | Self-contained test macro | `kind_integration_test` that creates cluster → deploys → tests → tears down. Single `bazel test` invocation. |
| v0.3 | Tilt integration docs | Documentation and examples for using `rules_kind` artifacts with Tilt's `custom_build`. |
| v0.3 | `rules_img` support | Validate and document usage with `rules_img` tarballs (should work already via generic interface). |
| v0.4 | Separate `.load` / `.apply` | Split `.deploy` into separate run targets for users who want finer control. |
| v0.4 | Wait conditions | Optional readiness checks between deploy and test (wait for pods ready, endpoints available). |
| Future | Multi-cluster | Support for deploying across multiple KinD clusters in a single `kind_object`. |
| Future | Helm support | `kind_helm_release` rule for Helm chart installation. |

---

## 12. Risks & Mitigations

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|------------|
| R1 | KinD release breaks `kind load image-archive` interface | Low | High | Pin to known-good versions in toolchain. CI tests against latest KinD. Detect breakage early. |
| R2 | `rules_oci` changes `oci_load` output format | Low | High | Interface is generic (any tarball). Only documentation/examples would need updating, not rule code. |
| R3 | Bazel 8 breaking changes during development | Medium | Medium | Track Bazel 8 RCs. Use `bazel_compatibility` in MODULE.bazel. |
| R4 | BCR review process delays | Medium | Low | Not blocking development. Users can consume from git_override during review. |
| R5 | Scope creep (requests for cluster lifecycle, templating) | High | Medium | This PRD explicitly documents non-goals. Defer firmly to v0.2+. |
| R6 | Hermetic toolchain download URLs change or disappear | Low | Medium | Use official GitHub release URLs. Mirror as fallback. Integrity hashes in repository rules. |
| R7 | Test flakiness in CI (Docker/KinD reliability) | Medium | Medium | `exclusive` tag prevents parallel test interference. Retry logic in CI workflow. Clear error messages. |
| R8 | Low adoption due to "deployment doesn't belong in build system" sentiment | Medium | Medium | Position as "dependency tracking + CI convenience", not as a deployment tool. KinD scope makes this clearly testing infrastructure. |

---

## 13. Success Metrics

### Adoption Metrics (6 months post-release)

| Metric | Target | Measurement |
|--------|--------|-------------|
| GitHub stars | 50+ | GitHub API |
| BCR downloads | Tracked via BCR analytics (if available) | BCR |
| GitHub issues (non-bug) | 10+ feature requests (indicates active user base) | GitHub |
| External contributors | 2+ PRs merged from non-authors | GitHub |

### Quality Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| CI pass rate | >95% on main branch | GitHub Actions |
| Time to first successful `bazel test` for new user | <10 minutes | User testing / feedback |
| Open bug count | <5 at any time | GitHub Issues |
| LOC (Starlark + shell) | <2,000 | `wc -l` |

### Ecosystem Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Migration to `bazel-contrib` | Within 6 months of v0.1.0 | Organization transfer |
| BCR publication | Within 2 weeks of v0.1.0 tag | BCR registry |
| Compatible Bazel versions | 8.x current + one prior minor | CI matrix |

---

## Appendix A: Full MODULE.bazel Example

```python
module(
    name = "my_project",
    version = "0.0.0",
)

bazel_dep(name = "rules_kind", version = "0.1.0")
bazel_dep(name = "rules_oci", version = "2.0.0")

# Configure kind + kubectl toolchains
kind = use_extension("@rules_kind//kind:extensions.bzl", "kind")
kind.toolchain(kind_version = "0.31.0")
kind.kubectl_toolchain(kubectl_version = "1.31.0")
use_repo(kind, "kind_toolchains")
register_toolchains("@kind_toolchains//:all")
```

## Appendix B: How `rules_kind` Avoids the Fate of `rules_k8s`

| Failure Mode (`rules_k8s`) | Mitigation (`rules_kind`) |
|-----------------------------|---------------------------|
| Hard dependency on `rules_docker` | No dependency on any image build ruleset. Accepts generic tarballs. |
| Complex Go resolver (540 LOC) for manifest substitution | No resolver. No manifest substitution. Tags match between tarball and manifest by construction. |
| Broad scope (general K8s deployment) | KinD-specific. Not trying to deploy to production clusters. |
| WORKSPACE-only, broke at Bazel HEAD | bzlmod from day 1. Bazel 8+ minimum. No WORKSPACE maintenance. |
| No active maintainers | Tiny surface area (~1,500–2,000 LOC). Sustainable for 1-2 maintainers. |
| Community philosophical opposition | Positioned as testing infrastructure (KinD), not deployment tooling. |
