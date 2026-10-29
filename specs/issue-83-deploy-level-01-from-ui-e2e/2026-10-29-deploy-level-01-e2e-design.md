# Deploy Level 01 from the UI — End-to-End

**Epic:** casehubio/scaffold#83
**Date:** 2026-10-29
**Branch:** issue-83-deploy-level-01-from-ui-e2e

## Overview

Make it possible to select a topology in the Deploy tab, click Deploy, and have Level 01 actually provision via Podman. Five workstreams — two CI health prerequisites discovered during design, plus three feature workstreams — converge on end-to-end verification.

### Dependency graph

```
W0a (fix reactor build) ──→ W2 (#85 land branches) ──→ W3 (#86 target routing + images)
W0b (scaffold CI trigger) ──→ W2 (#85 verify on CI)
W1 (#84 UI selection) ─────────────────────────────────→ end-to-end verification
```

W0a and W0b are prerequisites. W1 and W2 can run in parallel after W0a. W3 depends on W2. E2E verification needs W1, W2, and W3.

---

## Workstream 0a: Fix casehub-all reactor build (prerequisite)

The casehub-all reactor build is red. All recent completed runs fail with:

```
Failed to execute goal io.casehub:casehub-platform-spring-generator:0.4-SNAPSHOT:generate
  on project casehub-platform-registry-spring-jpa:
  Jandex index not found at .../registry-jpa/target/classes/META-INF/jandex.idx.
  Build the Quarkus module first.
```

**Root cause:** `registry-spring-jpa` depends on `registry-jpa`'s Jandex index at generate time, but the Quarkus module isn't built before the Spring generator runs. This is a build-order issue in the platform's `mvn-all.yml` or the module dependency declarations.

**Fix:** Either add an explicit dependency from `registry-spring-jpa` to `registry-jpa` in the reactor POM so Maven orders them correctly, or ensure the convergence build runs the Quarkus modules first. Investigate the exact build ordering in the reactor workflow.

**Repo:** `casehubio/platform` (the module) within `casehubio/casehub-all` (the reactor)

---

## Workstream 0b: Add push trigger to scaffold CI (prerequisite)

Scaffold's `.github/workflows/maven.yml` triggers on `repository_dispatch`, `pull_request`, and `workflow_dispatch` — but **not on `push`**. The #76 commits were pushed directly to main and the Build and Publish workflow never fired. Scaffold's CI hasn't validated the current main.

**Fix:** Add `push: branches: [main]` to the `on:` trigger block.

**Repo:** `casehubio/scaffold`

---

## Workstream 1: Fix catalogue table selection + wire deploy button (#84)

### 1.1 selectionKey crash fix (pages-runtime)

**Root cause:** `col("name", "Topology")` creates an output column with ID `"Topology"` (the alias) via `buildOutputColumns` in `group-eval.ts`. But `selectionKey: "name"` is passed verbatim to `row.text("name")` in `activation.ts:330`, which finds no column with that ID in the post-groupBy output dataset and throws `TYPE_MISMATCH: Column "name" is NULL, not TEXT or LABEL`.

**Fix location:** `pages-runtime/src/activation.ts`, in the selection wiring block (lines 328-330).

**Changes:**

1. When `selectionKey` is present on a data-table component, resolve it through the component's lookup definition. The lookup's `ResultColumn[]` maps `sourceId` → `columnId`. If `selectionKey` matches a `sourceId`, use the corresponding `columnId` instead.

2. Replace `row.text(selectionKey)` with a NULL-safe accessor: use `row.cell(resolvedKey)` and coerce to string via `String(cv.value)` instead of `row.text()` which throws on non-TEXT/LABEL types.

**Test:** Add a test case in `activation.test.ts` (or the appropriate test file) that verifies `selectionKey` resolves correctly when column aliases differ from source IDs.

### 1.2 Wire selection and deploy button (scaffold UI)

**File:** `scaffold-backend/src/main/webui/src/views/deploy.ts`

Add `selection: "single"` and `selectionKey: "name"` to the catalogue dataTable, then add an actionButton below it:

```typescript
dataTable({
  lookup: lookup("catalogue", groupBy(null,
    col("name", "Topology"),
    col("description", "Description"),
    col("methodologyStep", "Step"),
    col("prerequisites", "Prerequisites"),
  )),
  sortable: true,
  selection: "single",
  selectionKey: "name",
}),
actionButton({
  label: "Deploy Selected Topology",
  url: "/api/scaffold/catalogue/#{selection.catalogue.name}/deploy",
  method: "POST",
  confirm: "Deploy topology #{selection.catalogue.name}?",
  style: "primary",
  onSuccess: { refresh: ["catalogue"], message: "Deployment started" },
}),
```

**Template interpolation:** The `#{selection.<datasetId>.<field>}` syntax is a supported pages-runtime feature (confirmed in context-wiring tests — e.g. `#{selection.adverseEvents.id}`). For the catalogue table, the dataset ID is `"catalogue"` and the selection field is `"name"`, giving `#{selection.catalogue.name}`.

**Fallback:** If the actionButton doesn't support template URLs (distinct from the context-wiring layer), build a `<catalogue-deploy-view>` custom Lit element with per-row Deploy buttons.

---

## Workstream 2: Land cross-repo branches (#85)

Mechanical merge-and-install for two branches created during #76.

### 2.1 Engine — `fix/flow-execution-registry-scope`

- 1 commit: `fix(#1265): add @ApplicationScoped to FlowExecutionRegistry`
- Push branch, merge to main
- `mvn install -f ../engine/pom.xml -DskipTests`

### 2.2 Ops — `issue-76-topology-catalogue-deploy`

- 2 commits: extract `compileTopology()` + add metadata/playbook files for Level 01
- Push branch, merge to main
- `mvn install -f ../ops/pom.xml -pl service -am -DskipTests`

Both branches are ready — mechanical merge + install. **Prerequisite:** reactor must be green (W0a) before merging, so integration can be validated.

---

## Workstream 3: Level 01 deployable locally via Podman (#86)

### 3.1 Deployment target override

**Location:** `casehub-ops` deployment module.

Add a config property `casehub.ops.deployment-target-override` (optional, no default). When set, `CaseHubAppProvisionHandler.resolveTarget()` ignores the per-node `deploymentTargetId` from the graph and uses the override value instead.

**Files to change:**

1. `CaseHubAppProvisionHandler.java` (line 83-88) — inject the config property, apply override in `resolveTarget()`:
   ```java
   @ConfigProperty(name = "casehub.ops.deployment-target-override")
   Optional<String> targetOverride;

   private ApplicationDeploymentTarget resolveTarget(String targetId) {
       String effectiveTarget = targetOverride.orElse(targetId);
       if (effectiveTarget == null || effectiveTarget.isEmpty()) { return deployTarget; }
       return allTargets.stream()
                        .filter(t -> effectiveTarget.equals(t.targetId()))
                        .findFirst()
                        .orElse(deployTarget);
   }
   ```

2. `CaseHubAppDriftChecker.java` (line 59-65) — same override pattern in its `resolveTarget()` to keep provision and drift-check in sync.

3. Scaffold `application.properties` — set the override for dev mode:
   ```properties
   %dev.casehub.ops.deployment-target-override=podman
   ```

**The graph stays truthful** — plan views and diagnostics still show `deploymentTarget: claudony` from the YAML. The override is purely a runtime routing concern.

### 3.2 Container images

The four images needed for Level 01 (`casehubio/manifest:latest`, `casehubio/eidos:latest`, `casehubio/engine:latest`, `casehubio/my-llm-app:latest`) must be built locally from their source repos.

**Strategy:** Add the `quarkus-container-image-podman` extension to each source repo in a Maven profile:

```xml
<profile>
  <id>container</id>
  <dependencies>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-container-image-podman</artifactId>
    </dependency>
  </dependencies>
  <properties>
    <quarkus.container-image.build>true</quarkus.container-image.build>
    <quarkus.container-image.group>casehubio</quarkus.container-image.group>
  </properties>
</profile>
```

Then `mvn package -Pcontainer -DskipTests` builds the image into the local Podman store.

**Scope note:** Image build infrastructure is prerequisite work in each source repo, not part of scaffold or ops. For #86, document the prerequisite and verify images exist before attempting deploy. If images aren't available, the deploy will fail with a clear `PrebuiltImagePipeline` error ("no image set").

### 3.3 Podman connectivity

`PodmanDeploymentTarget` uses `PodmanClient` which connects via Unix domain socket. On macOS, the socket path is typically `$TMPDIR/podman/podman.sock` after `podman machine start`.

**Prerequisite:** `podman machine start` must have run. The `PodmanDeploymentTargetLiveTest` already validates this connectivity — replicate its setup expectations.

**Scaffold must have the ops `container` module on its classpath** to make `PodmanDeploymentTarget` discoverable as a CDI bean. Currently scaffold depends only on `casehub-ops-service` — `casehub-ops-container` is **not** on the classpath. Add the dependency to `scaffold-backend/pom.xml`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ops-container</artifactId>
</dependency>
```

---

## End-to-End Verification

**Prerequisites:**
1. Reactor green (W0a)
2. Scaffold CI trigger fixed (W0b)
3. Cross-repo branches landed and CI-validated (#85)
4. Podman running (`podman machine start`)
5. Container images built and available locally
6. Scaffold rebuilt with updated deps

**Steps:**
1. Start scaffold in dev mode (`mvn quarkus:dev`)
2. Open browser → Deploy tab
3. Catalogue table shows Level 01 (01-hello-llm) with description, step, prerequisites
4. Select the Level 01 row — row highlights, selection state propagates
5. Click "Deploy Selected Topology" — confirmation dialog appears
6. Confirm — POST to `/api/scaffold/catalogue/01-hello-llm/deploy`
7. Observe: 4 containers provisioned in dependency order (manifest → eidos → engine → my-llm)
8. Cases tab shows a new `application-lifecycle` case created
9. Deployment Records table refreshes showing the deployment

**Failure modes to verify:**
- Deploy when no row selected — button disabled or error message
- Deploy when topology already running — 409 Conflict response, user sees error
- Podman not running — deploy fails with connection error, not a crash

---

## Testing Strategy

| Layer | Test | Type |
|-------|------|------|
| Reactor build | casehub-all CI — verify green after platform fix | CI |
| Scaffold CI | Build and Publish — verify fires on push to main | CI |
| selectionKey resolution | activation.test.ts — alias vs source ID | Unit |
| actionButton wiring | Manual browser verification | Manual |
| Target override config | CaseHubAppProvisionHandlerTest — verify override takes precedence | Unit |
| Drift checker sync | CaseHubAppDriftCheckerTest — same override applies | Unit |
| Podman deploy | PodmanDeploymentTargetLiveTest — existing, reuse | Integration |
| End-to-end | Manual: dev mode → select → deploy → verify containers | Manual |

---

## Execution Sequence

1. **W0a** — Fix reactor build (platform Jandex index ordering)
2. **W0b** — Add push trigger to scaffold CI (can parallel with W0a)
3. **W2** — Land cross-repo branches (#85), verify reactor green
4. **W1** — Fix selectionKey + wire deploy button (#84) (can start in parallel with W2 after W0a)
5. **W3** — Deployment target override + image builds (#86) (after W2)
6. **E2E** — End-to-end verification (after W1 + W3)

---

## References

- `pages-runtime/src/activation.ts:328-330` — selectionKey wiring (crash site)
- `pages-data/src/dataset/group-eval.ts:800-809` — buildOutputColumns (alias as column ID)
- `pages-data/src/dataset/conversion.ts:57-88` — row.cell() and row.text() (NULL handling)
- `pages-ui/src/dsl/lookup-helpers.ts:179-184` — col() helper (sourceId vs columnId)
- `pages-ui/src/dsl/builders.ts:394,474` — dataTable and actionButton DSL
- `CaseHubAppProvisionHandler.java:83-88` — resolveTarget() (override injection point)
- `CaseHubAppDriftChecker.java:59-65` — duplicate resolveTarget()
- `DeploymentNodeSpecFactoryProvider.java:22-31` — compile-time target bake
- `PodmanDeploymentTarget.java` — targetId="podman", @ApplicationScoped
- `PrebuiltImagePipeline.java` — default @DefaultBean, pass-through image ref
- `01-hello-llm.yaml` — Level 01 topology, 4 nodes, deploymentTarget: claudony
- `platform/boundary-rules.md` — SPI implementations belong in deploying applications
- `platform/capability-ownership.md` — topology compilation owned by casehub-ops
- casehub-all CI run 40964406385 — reactor failure (Jandex index build order)
- `.github/workflows/maven.yml` — scaffold CI trigger config (missing push trigger)
- `casehub-platform-spring-generator:generate` — the Maven goal that requires Jandex index
