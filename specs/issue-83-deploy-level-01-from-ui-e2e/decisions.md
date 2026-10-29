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
