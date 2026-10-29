## D1: Fix selectionKey crash in pages-data activation

**Choice:** Fix activation.ts to resolve selectionKey through the lookup's ResultColumn mapping (source ID → output column ID), and make getRowKey NULL-safe
**Alternatives:**
- Use alias as selectionKey (`"Topology"` instead of `"name"`) — works but counterintuitive and fragile if aliases change
- Drop aliases from col() — raw field names as headers, loses human-readable labels
**Rationale:** This is a genuine framework bug where the abstraction leaks. Users define `col("name", "Topology")` and naturally expect `selectionKey: "name"` to work. The activation layer already has access to the lookup definition to perform the mapping.
**Trade-offs:** Requires a change in the pages-runtime package (activation.ts), which is a shared framework component — but the fix is small and prevents every future dataTable user from hitting the same crash.
**Sources:** pages-runtime/src/activation.ts:328-330, pages-data/src/dataset/group-eval.ts:800-809, pages-data/src/dataset/conversion.ts:57-88, pages-ui/src/dsl/lookup-helpers.ts:179-184
**Exploration:** quick
**Status:** captured

## D2: Deploy-time target override in resolveTarget()

**Choice:** Add a config property (`casehub.ops.deployment-target-override`) to CaseHubAppProvisionHandler.resolveTarget() that overrides the per-node deployment target ID at provision time
**Alternatives:**
- Compile-time override in DeploymentNodeSpecFactoryProvider — graph no longer represents the YAML, and injecting ConfigProperty into a Java SPI class is awkward
- Scaffold-level override passing target through the API boundary — requires API changes across scaffold→ops and doesn't help other ops consumers
**Rationale:** The graph should represent what the YAML says (useful for plan views and diagnostics). The runtime environment determines where things actually deploy. Both CaseHubAppProvisionHandler and CaseHubAppDriftChecker have resolveTarget() — a shared config property keeps them in sync. PodmanDeploymentTarget is already @ApplicationScoped and discoverable.
**Trade-offs:** The override is global — all nodes in all topologies use the same target. Per-node overrides would need a more complex mechanism, but for Level 01 local dev this is exactly what we want.
**Sources:** CaseHubAppProvisionHandler.java:83-88 (resolveTarget), CaseHubAppDriftChecker.java:59-65 (duplicate resolveTarget), DeploymentNodeSpecFactoryProvider.java:22-31 (compile-time bake), PodmanDeploymentTarget.java (targetId="podman")
**Exploration:** quick
**Status:** captured

## D3: Container image build strategy

**Choice:** Quarkus container-image-podman extension in each source repo, activated via Maven profile (`mvn package -Pcontainer`)
**Alternatives:**
- Dockerfile-based `podman build` — more manual, no pom.xml changes but not Quarkus-native
- Build script in scaffold — convenient but creates scaffold→source-repo coupling that violates boundary rules
**Rationale:** Quarkus-native approach integrates with the existing Maven build pipeline. Each repo owns its own image build, consistent with the platform's prebuilt-image-pipeline model where source repos publish and ops consumes.
**Trade-offs:** Requires changes in each source repo (manifest, eidos, engine, my-llm-app) — this is prerequisite work outside scaffold/ops. For #86, we document the prerequisite and focus on the deployment target override.
**Sources:** PrebuiltImagePipeline.java (default @DefaultBean), ApplicationBuildPipeline.java (SPI), topology-test-on-dispatch.yml (image-pushed dispatch pattern), platform/boundary-rules.md (SPI implementations belong in deploying applications)
**Exploration:** quick
**Status:** captured

## D4: Fix casehub-all reactor build before landing branches

**Choice:** Fix the Jandex index build-order issue in platform's Spring convergence layer before merging #85 cross-repo branches, to ensure the reactor can validate the full integration
**Alternatives:**
- Land branches anyway and fix reactor separately — risks compounding failures, can't verify integration
- Skip reactor validation and test locally only — local builds may miss cross-module issues
**Rationale:** The reactor has been red for multiple runs (all recent completions show failure). The error is in `registry-spring-jpa` which can't find `registry-jpa`'s Jandex index — a build-order problem, not a code error. Landing #85 branches on a red reactor means we can't verify they don't introduce new failures.
**Trade-offs:** Fixing a platform build-order issue blocks #83 progress, but without it there's no integration safety net.
**Sources:** casehub-all CI run 40964406385 (failure log), platform/registry-spring-jpa pom.xml, casehub-platform-spring-generator:generate goal
**Exploration:** quick
**Depends on:** None (prerequisite for all downstream work)
**Status:** captured

## D5: Add push trigger to scaffold CI workflow

**Choice:** Add `push: branches: [main]` to scaffold's `.github/workflows/maven.yml` triggers
**Alternatives:**
- Enforce PR-only merges — would work but requires branch protection changes and slows down the direct-push workflow currently used
- Rely on casehub-all reactor only — scaffold-specific tests would only run as part of the full reactor, slower feedback
**Rationale:** Scaffold's Build and Publish workflow triggers on `repository_dispatch`, `pull_request`, and `workflow_dispatch` — but not `push`. The #76 commits were pushed directly to main and the workflow never fired. Scaffold has been untested on CI since those commits landed.
**Trade-offs:** Adding a push trigger means builds fire on every push to main (additional CI cost), but scaffold pushes are infrequent enough that this is negligible.
**Sources:** .github/workflows/maven.yml trigger config, casehubio/scaffold CI run history (last Build and Publish: never visible in recent runs)
**Exploration:** quick
**Status:** captured
