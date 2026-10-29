# Deploy Level 01 E2E Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #83 — Deploy Level 01 from the UI — end-to-end
**Issue group:** #83, #84, #85, #86

**Goal:** Select a topology in the Deploy tab, click Deploy, and have Level 01 provision 4 containers via Podman locally.

**Architecture:** Five workstreams — two CI health fixes, a UI selection+button fix in pages-runtime/scaffold, cross-repo branch landing, and a deployment target override in ops. Each workstream is independently testable; they converge at end-to-end verification.

**Tech Stack:** Quarkus 3.32.2, TypeScript (pages-runtime/pages-ui), Maven, GitHub Actions, Podman

## Global Constraints

- Quarkus version: `3.32.2` (from root pom.xml `version.quarkus.platform`)
- All ecosystem repos must match Quarkus version
- GitHub Packages for dependency resolution (`server-id: github`)
- Commits reference issues: `Refs #N` or `Closes #N`
- Use IntelliJ MCP for all code navigation and structural edits

---

## Batch 1: CI Health [reactor green, scaffold CI fires on push]

### Task 1: Fix casehub-all reactor Jandex build order

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/platform/aggregator-convergence.xml`
- Verify: casehub-all CI workflow

**Interfaces:**
- Consumes: Jandex indexes uploaded by slice B
- Produces: Green reactor build

The spring generator in `registry-spring-jpa` reads `registry-jpa/target/classes/META-INF/jandex.idx` from the filesystem at `generate-sources` phase. The CI downloads slice artifacts + runs the full reactor with `-T 1C` (parallel). If the Jandex download for `registry-jpa` is corrupt or missing, the parallel build may reach `registry-spring-jpa` before `registry-jpa` regenerates it.

The convergence aggregator (`aggregator-convergence.xml`) is designed to list Quarkus modules as Jandex fallbacks, but `registry-jpa` and 7 other paired modules are missing.

- [ ] **Step 1: Verify the download is the problem**

```bash
# Check the CI workflow's Jandex upload step in slice B
# File: /Users/mdproctor/claude/casehub/casehub-all/.github/workflows/mvn-all.yml
# Look for the upload-artifact step after slice-b and verify it globs registry-jpa
```

Read `/Users/mdproctor/claude/casehub/casehub-all/.github/workflows/mvn-all.yml` and search for the Jandex upload/download artifact steps. Confirm whether `registry-jpa`'s index is included in the upload glob and correctly downloaded.

- [ ] **Step 2: Add missing Quarkus modules to convergence aggregator**

In `/Users/mdproctor/claude/casehub/platform/aggregator-convergence.xml`, add the missing Quarkus modules alongside the existing ones in the safety-net comment block:

```xml
<!-- Add these missing Jandex source modules -->
<module>registry-jpa-common</module>
<module>registry-jpa</module>
```

Check for other missing pairs (persistence-jpa, datasource-jpa, acl-jpa, session-jpa, authn-webauthn-jpa, authn-social-jpa, subscriptions-jpa) and add them if their Spring counterparts are already in the convergence aggregator.

- [ ] **Step 3: Verify locally**

```bash
mvn --batch-mode install -f /Users/mdproctor/claude/casehub/platform/aggregator-convergence.xml -DskipTests
```

Expected: BUILD SUCCESS — `registry-spring-jpa` generates Spring code from `registry-jpa`'s Jandex index.

- [ ] **Step 4: Push and verify CI**

Push the convergence aggregator change. Trigger a casehub-all build and verify the "Convergence + Ecosystem" job passes.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform commit -am "fix: add missing Quarkus modules to convergence aggregator

The spring generator reads Jandex indexes from paired Quarkus modules.
When CI artifact downloads fail, the convergence aggregator rebuilds them
as a fallback. registry-jpa and 7 other paired modules were missing.

Refs casehubio/casehub-all#NNN"
```

---

### Task 2: Add push trigger to scaffold CI

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/scaffold/.github/workflows/maven.yml`

**Interfaces:**
- Consumes: Nothing
- Produces: Scaffold CI fires on push to main

- [ ] **Step 1: Add push trigger**

In `/Users/mdproctor/claude/casehub/scaffold/.github/workflows/maven.yml`, add `push` to the `on:` block:

```yaml
on:
  push:
    branches: [main]
  repository_dispatch:
    types: [upstream-published]
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches: [main]
  workflow_dispatch:
```

- [ ] **Step 2: Verify the build job condition handles push events**

Read the `if:` conditions on the build steps. The existing condition is:
```yaml
if: github.event_name != 'pull_request' || github.event.action != 'closed' || github.event.pull_request.merged == true
```

For `push` events, `github.event_name` is `"push"` (not `"pull_request"`), so the condition evaluates to `true`. All steps will run. No changes needed.

- [ ] **Step 3: Push and verify**

Push to main. Verify the "Build and Publish" workflow fires within a few minutes on the GitHub Actions tab.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/scaffold add .github/workflows/maven.yml
git -C /Users/mdproctor/claude/casehub/scaffold commit -m "fix: add push trigger to Build and Publish workflow

The workflow only triggered on repository_dispatch, pull_request, and
workflow_dispatch — not on direct pushes to main. Recent commits pushed
directly were never CI-validated.

Closes #NNN Refs #83"
```

---

## Batch 2: Land Cross-Repo Branches [engine + ops merged, local deps rebuilt]

### Task 3: Merge engine and ops branches to main

**Files:**
- Engine branch: `fix/flow-execution-registry-scope` in `/Users/mdproctor/claude/casehub/engine`
- Ops branch: `issue-76-topology-catalogue-deploy` in `/Users/mdproctor/claude/casehub/ops`

**Interfaces:**
- Consumes: Green reactor (Task 1)
- Produces: Updated local Maven artifacts for engine and ops

- [ ] **Step 1: Push and merge engine branch**

```bash
git -C /Users/mdproctor/claude/casehub/engine push origin fix/flow-execution-registry-scope
```

Create PR via `gh pr create`, merge via `gh pr merge`, then update local main:

```bash
git -C /Users/mdproctor/claude/casehub/engine checkout main
git -C /Users/mdproctor/claude/casehub/engine pull origin main
```

- [ ] **Step 2: Install engine to local Maven repo**

```bash
/opt/homebrew/bin/mvn --batch-mode install -f /Users/mdproctor/claude/casehub/engine/pom.xml -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Push and merge ops branch**

```bash
git -C /Users/mdproctor/claude/casehub/ops push origin issue-76-topology-catalogue-deploy
```

Create PR, merge, update local main:

```bash
git -C /Users/mdproctor/claude/casehub/ops checkout main
git -C /Users/mdproctor/claude/casehub/ops pull origin main
```

- [ ] **Step 4: Install ops to local Maven repo**

```bash
/opt/homebrew/bin/mvn --batch-mode install -f /Users/mdproctor/claude/casehub/ops/pom.xml -pl service -am -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Verify scaffold builds with updated deps**

```bash
/opt/homebrew/bin/mvn --batch-mode install -f /Users/mdproctor/claude/casehub/scaffold/pom.xml -DskipTests
```

Expected: BUILD SUCCESS — scaffold compiles against the new engine and ops artifacts.

- [ ] **Step 6: Verify casehub-all reactor is green**

Check the CI dashboard. The engine and ops pushes should trigger reactor runs. Wait for green.

---

## Batch 3: UI Selection + Deploy Button [catalogue table selectable, deploy button wired]

### Task 4: Fix selectionKey resolution in activation.ts

**Files:**
- Modify: `scaffold-backend/src/main/webui/.casehub-packages/packages/pages-runtime/src/activation.ts` (lines 323-331)
- Test: `scaffold-backend/src/main/webui/.casehub-packages/packages/pages-runtime/src/activation.test.ts`

**Interfaces:**
- Consumes: `DataSetLookup` from component props, `TypedRow` from pages-data
- Produces: Working `selectionKey` that resolves source IDs through column aliases

- [ ] **Step 1: Write the failing test**

Add to `activation.test.ts` a test that verifies `selectionKey` with aliased columns:

```typescript
it("resolves selectionKey through column alias when source ID differs", () => {
  // Setup: dataTable with lookup that has col("name", "Topology")
  // selectionKey: "name" should resolve to output column "Topology"
  const component: TypedComponent<"data-table"> = {
    type: "data-table",
    props: {
      lookup: lookup("test-ds", groupBy(null, col("name", "Topology"))),
      selection: "single",
      selectionKey: "name",
    },
  };

  const container = document.createElement("div");
  activateComponents(container, [component], mockOptions());

  const dataTable = container.querySelector("pages-data-table") as any;
  expect(dataTable.getRowKey).toBeDefined();

  // Create a mock TypedRow with column ID "Topology" (the alias)
  const mockRow = createMockTypedRow({ "Topology": "01-hello-llm" });
  expect(dataTable.getRowKey(mockRow)).toBe("01-hello-llm");
});
```

Adapt the test to match the existing test patterns in `activation.test.ts` — check how other tests create components and mock data.

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/mdproctor/claude/casehub/scaffold/scaffold-backend/src/main/webui
npx vitest run --reporter=verbose -- activation.test.ts
```

Expected: FAIL with TYPE_MISMATCH or "Column 'name' is NULL"

- [ ] **Step 3: Implement the fix**

In `activation.ts` around lines 323-331, replace:

```typescript
const selectionKey = (component.props as Record<string, unknown> | undefined)?.selectionKey as string | undefined;
if (selectionKey) {
  (vizEl as unknown as Record<string, unknown>).getRowKey = (row: TypedRow) => row.text(selectionKey as ColumnId);
}
```

With:

```typescript
const selectionKey = (component.props as Record<string, unknown> | undefined)?.selectionKey as string | undefined;
if (selectionKey) {
  let resolvedKey: ColumnId = selectionKey as ColumnId;
  if (lookup) {
    const matchingCol = lookup.operations
      .flatMap((op: any) => op.resultColumns ?? [])
      .find((rc: any) => rc.sourceId === selectionKey && rc.columnId !== selectionKey);
    if (matchingCol) {
      resolvedKey = matchingCol.columnId as ColumnId;
    }
  }
  (vizEl as unknown as Record<string, unknown>).getRowKey = (row: TypedRow) => {
    const cv = row.cell(resolvedKey);
    if (cv.type === "NULL") return "";
    return String(cv.value);
  };
}
```

This resolves `selectionKey` through the lookup's `ResultColumn` mapping (sourceId → columnId) and uses `row.cell()` + String coercion instead of `row.text()` which throws on non-TEXT types.

- [ ] **Step 4: Run test to verify it passes**

```bash
cd /Users/mdproctor/claude/casehub/scaffold/scaffold-backend/src/main/webui
npx vitest run --reporter=verbose -- activation.test.ts
```

Expected: PASS

- [ ] **Step 5: Run all pages-runtime tests**

```bash
cd /Users/mdproctor/claude/casehub/scaffold/scaffold-backend/src/main/webui
npx vitest run --reporter=verbose -- .casehub-packages/packages/pages-runtime/
```

Expected: All existing tests still pass — no regressions.

- [ ] **Step 6: Commit**

```bash
git add scaffold-backend/src/main/webui/.casehub-packages/packages/pages-runtime/src/activation.ts
git add scaffold-backend/src/main/webui/.casehub-packages/packages/pages-runtime/src/activation.test.ts
git commit -m "fix(#84): resolve selectionKey through column alias mapping

col('name', 'Topology') creates output column ID 'Topology', but
selectionKey: 'name' looked up the source ID verbatim. Now resolves
through the lookup's ResultColumn sourceId→columnId mapping. Also
switches from row.text() (throws on NULL) to row.cell() + String
coercion for robustness.

Refs #84"
```

---

### Task 5: Wire selection and deploy button in deploy view

**Files:**
- Modify: `scaffold-backend/src/main/webui/src/views/deploy.ts`

**Interfaces:**
- Consumes: Fixed `selectionKey` resolution (Task 4), `actionButton` from pages-ui
- Produces: Catalogue table with row selection + deploy button

- [ ] **Step 1: Add actionButton import**

In `deploy.ts` line 1, add `actionButton` to the imports:

```typescript
import { rows, columns, dataTable, title, actionButton } from "@casehubio/pages-ui";
```

- [ ] **Step 2: Add selection and deploy button to catalogue table**

Replace the current catalogue `dataTable` block (lines 6-14) with:

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

- [ ] **Step 3: Build the web UI**

```bash
cd /Users/mdproctor/claude/casehub/scaffold/scaffold-backend/src/main/webui
npm run build
```

Expected: No TypeScript compilation errors.

- [ ] **Step 4: Manual browser verification**

Start dev mode and verify in the browser:

```bash
/opt/homebrew/bin/mvn --batch-mode quarkus:dev -f /Users/mdproctor/claude/casehub/scaffold/scaffold-backend/pom.xml
```

1. Open browser → Deploy tab
2. Catalogue table renders with rows
3. Click a row → row highlights (selection works)
4. "Deploy Selected Topology" button appears below the table
5. Click button → confirmation dialog shows topology name
6. Cancel — no deploy (just verify the dialog works)

If `#{selection.catalogue.name}` doesn't interpolate in the actionButton URL, fall back to a custom Lit element (see spec's fallback section).

- [ ] **Step 5: Commit**

```bash
git add scaffold-backend/src/main/webui/src/views/deploy.ts
git commit -m "feat(#84): wire catalogue table selection + deploy button

Add single-row selection with selectionKey 'name' to the catalogue
dataTable. Add an actionButton that POSTs to the deploy endpoint with
the selected topology name, with confirmation dialog and auto-refresh.

Closes #84 Refs #83"
```

---

## Batch 4: Podman Deployment Target [target override wired, scaffold has Podman dep]

### Task 6: Add deployment-target-override to ops handlers

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/ops/deployment/src/main/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandler.java`
- Modify: `/Users/mdproctor/claude/casehub/ops/deployment/src/main/java/io/casehub/ops/deployment/drift/CaseHubAppDriftChecker.java`
- Modify: `/Users/mdproctor/claude/casehub/ops/deployment/src/test/java/io/casehub/ops/deployment/handler/CaseHubAppProvisionHandlerTest.java`
- Modify: `/Users/mdproctor/claude/casehub/ops/deployment/src/test/java/io/casehub/ops/deployment/drift/CaseHubAppDriftCheckerTest.java`

**Interfaces:**
- Consumes: `@ConfigProperty(name = "casehub.ops.deployment-target-override")`
- Produces: `resolveTarget()` respects override when set

- [ ] **Step 1: Write the failing test for provision handler**

Add to `CaseHubAppProvisionHandlerTest.java`:

```java
@Test
void resolveTargetUsesOverrideWhenSet() throws Exception {
    var podmanTarget = new StubDeploymentTarget() {
        @Override public String targetId() { return "podman"; }
    };
    var claudonyTarget = new StubDeploymentTarget() {
        @Override public String targetId() { return "claudony"; }
    };

    var overrideHandler = new CaseHubAppProvisionHandler(
            buildPipeline, deployTarget,
            new io.casehub.ops.deployment.InstanceIdStore(),
            stubInstance(deployTarget, podmanTarget, claudonyTarget));

    // Set the override via reflection
    var field = CaseHubAppProvisionHandler.class.getDeclaredField("targetOverride");
    field.setAccessible(true);
    field.set(overrideHandler, "podman");

    var spec = new CaseHubApplicationSpec(
        "app-1", null, "img:1.0",
        new RuntimeConfig("0.2", List.of()),
        Map.of(), List.of(), 1);

    // Call provision with deploymentTargetId "claudony" — should be overridden to "podman"
    var graph = new DefaultDesiredStateGraphFactory().empty();
    var result = overrideHandler.provision(spec, new ProvisionContext("t1", graph), "claudony", null);

    assertThat(result).isInstanceOf(ProvisionResult.Success.class);
    // podmanTarget should have received the deploy, not claudonyTarget
    assertThat(podmanTarget.deployedInstances()).containsKey("app-1-instance");
}

@SuppressWarnings("unchecked")
private static Instance<ApplicationDeploymentTarget> stubInstance(ApplicationDeploymentTarget... targets) {
    return new Instance<>() {
        @Override public Instance<ApplicationDeploymentTarget> select(java.lang.annotation.Annotation... q) { return this; }
        @Override public <U extends ApplicationDeploymentTarget> Instance<U> select(Class<U> s, java.lang.annotation.Annotation... q) { return (Instance<U>) this; }
        @Override public <U extends ApplicationDeploymentTarget> Instance<U> select(jakarta.enterprise.util.TypeLiteral<U> s, java.lang.annotation.Annotation... q) { return (Instance<U>) this; }
        @Override public boolean isUnsatisfied() { return false; }
        @Override public boolean isAmbiguous() { return targets.length > 1; }
        @Override public boolean isResolvable() { return targets.length == 1; }
        @Override public ApplicationDeploymentTarget get() { return targets[0]; }
        @Override public java.util.Iterator<ApplicationDeploymentTarget> iterator() { return List.of(targets).iterator(); }
        @Override public java.util.stream.Stream<ApplicationDeploymentTarget> stream() { return java.util.stream.Stream.of(targets); }
        @Override public void destroy(ApplicationDeploymentTarget i) {}
        @Override public Handle<ApplicationDeploymentTarget> getHandle() { return null; }
        @Override public Iterable<Handle<ApplicationDeploymentTarget>> handles() { return List.of(); }
    };
}
```

Note: the drift checker test already has `stubInstance()` — the provision handler test needs one too. Check if `StubDeploymentTarget` supports `targetId()` override; if not, use inline anonymous classes as shown above.

- [ ] **Step 2: Run test to verify it fails**

```bash
/opt/homebrew/bin/mvn --batch-mode test -f /Users/mdproctor/claude/casehub/ops/deployment/pom.xml -pl . -Dtest=CaseHubAppProvisionHandlerTest#resolveTargetUsesOverrideWhenSet
```

Expected: FAIL — `targetOverride` field doesn't exist yet.

- [ ] **Step 3: Add targetOverride field to CaseHubAppProvisionHandler**

Add field injection after the existing fields (line 29):

```java
import org.eclipse.microprofile.config.inject.ConfigProperty;
import java.util.Optional;

// ... inside the class, after the allTargets field:
@ConfigProperty(name = "casehub.ops.deployment-target-override", defaultValue = "")
String targetOverride;
```

Modify `resolveTarget()` (lines 83-89):

```java
private ApplicationDeploymentTarget resolveTarget(String targetId) {
    String effectiveTarget = (targetOverride != null && !targetOverride.isEmpty())
            ? targetOverride : targetId;
    if (effectiveTarget == null || effectiveTarget.isEmpty()) { return deployTarget; }
    return allTargets.stream()
                     .filter(t -> effectiveTarget.equals(t.targetId()))
                     .findFirst()
                     .orElse(deployTarget);
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
/opt/homebrew/bin/mvn --batch-mode test -f /Users/mdproctor/claude/casehub/ops/deployment/pom.xml -pl . -Dtest=CaseHubAppProvisionHandlerTest
```

Expected: All tests PASS.

- [ ] **Step 5: Write failing test for drift checker**

Add to `CaseHubAppDriftCheckerTest.java`:

```java
@Test
void resolveTargetUsesOverrideWhenSet() throws Exception {
    var podmanTarget = new StatusReturningTarget("podman", NodeStatus.PRESENT);
    var claudonyTarget = new StatusReturningTarget("claudony", NodeStatus.ABSENT);
    var defaultTarget = new StatusReturningTarget("", NodeStatus.UNKNOWN);

    var overrideChecker = new CaseHubAppDriftChecker(defaultTarget,
            new io.casehub.ops.deployment.InstanceIdStore(),
            stubInstance(defaultTarget, podmanTarget, claudonyTarget));

    var field = CaseHubAppDriftChecker.class.getDeclaredField("targetOverride");
    field.setAccessible(true);
    field.set(overrideChecker, "podman");

    var appSpec = new CaseHubApplicationSpec(
            "app-1", null, "img:1.0",
            new RuntimeConfig("0.2", List.of()),
            Map.of(), List.of(), 1);
    var wrappedSpec = new DeploymentApplicationNodeSpec(appSpec, "claudony", null);

    // Should use podman (override), not claudony (from spec)
    assertThat(overrideChecker.check(wrappedSpec, "tenant-1")).isEqualTo(NodeStatus.PRESENT);
}
```

- [ ] **Step 6: Add targetOverride field to CaseHubAppDriftChecker**

Same pattern as the provision handler — add after the `allTargets` field:

```java
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ConfigProperty(name = "casehub.ops.deployment-target-override", defaultValue = "")
String targetOverride;
```

Modify `resolveTarget()` (lines 59-65):

```java
private ApplicationDeploymentTarget resolveTarget(String targetId) {
    String effectiveTarget = (targetOverride != null && !targetOverride.isEmpty())
            ? targetOverride : targetId;
    if (effectiveTarget == null || effectiveTarget.isEmpty() || allTargets == null) { return deployTarget; }
    return allTargets.stream()
            .filter(t -> effectiveTarget.equals(t.targetId()))
            .findFirst()
            .orElse(deployTarget);
}
```

- [ ] **Step 7: Run all deployment module tests**

```bash
/opt/homebrew/bin/mvn --batch-mode test -f /Users/mdproctor/claude/casehub/ops/deployment/pom.xml
```

Expected: All tests PASS.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ops add deployment/
git -C /Users/mdproctor/claude/casehub/ops commit -m "feat(#86): deployment-target-override config property

Add casehub.ops.deployment-target-override to CaseHubAppProvisionHandler
and CaseHubAppDriftChecker. When set, overrides the per-node
deploymentTarget from the topology YAML. The graph stays truthful —
plan views still show the YAML value. The override is purely a runtime
routing concern for local dev (e.g. podman instead of claudony).

Refs casehubio/scaffold#86"
```

- [ ] **Step 9: Install ops to local Maven repo**

```bash
/opt/homebrew/bin/mvn --batch-mode install -f /Users/mdproctor/claude/casehub/ops/pom.xml -DskipTests
```

---

### Task 7: Add ops-container dependency to scaffold + configure dev mode

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/scaffold/scaffold-backend/pom.xml`
- Modify: `/Users/mdproctor/claude/casehub/scaffold/scaffold-backend/src/main/resources/application.properties`

**Interfaces:**
- Consumes: `casehub-ops-container` module (PodmanDeploymentTarget), deployment-target-override config (Task 6)
- Produces: Scaffold discovers PodmanDeploymentTarget via CDI, dev mode routes to Podman

- [ ] **Step 1: Add casehub-ops-container dependency**

In `/Users/mdproctor/claude/casehub/scaffold/scaffold-backend/pom.xml`, add after the existing `casehub-ops-service` dependency:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ops-container</artifactId>
</dependency>
```

- [ ] **Step 2: Add dev-mode deployment target override**

In `/Users/mdproctor/claude/casehub/scaffold/scaffold-backend/src/main/resources/application.properties`, add:

```properties
%dev.casehub.ops.deployment-target-override=podman
```

- [ ] **Step 3: Verify scaffold builds**

```bash
/opt/homebrew/bin/mvn --batch-mode install -f /Users/mdproctor/claude/casehub/scaffold/pom.xml -DskipTests
```

Expected: BUILD SUCCESS — `PodmanDeploymentTarget` is now on the classpath.

- [ ] **Step 4: Commit**

```bash
git add scaffold-backend/pom.xml scaffold-backend/src/main/resources/application.properties
git commit -m "feat(#86): add ops-container dep + dev-mode Podman target override

Add casehub-ops-container to scaffold-backend classpath so
PodmanDeploymentTarget is CDI-discoverable. Set
casehub.ops.deployment-target-override=podman for dev profile so
local topology deployments route to Podman instead of Claudony.

Refs #86 Refs #83"
```

---

## Batch 5: End-to-End Verification [Level 01 deploys via UI]

### Task 8: Build container images and verify end-to-end deploy

**Files:**
- No code changes — verification only

**Interfaces:**
- Consumes: Everything from Batches 1-4
- Produces: 4 running Podman containers + application-lifecycle case

- [ ] **Step 1: Ensure Podman is running**

```bash
podman machine start
```

- [ ] **Step 2: Build container images for Level 01**

Build the four images needed for 01-hello-llm. Each source repo needs:

```bash
# Check if images already exist
podman images | grep casehubio
```

If not available, build from each source repo using Quarkus container-image extension or Dockerfile:

```bash
# Example for each repo (adapt paths):
/opt/homebrew/bin/mvn --batch-mode package -Dquarkus.container-image.build=true -Dquarkus.container-image.group=casehubio -DskipTests -f /path/to/repo/pom.xml
```

If the source repos don't have container-image extension configured, use the Dockerfiles:

```bash
# Build from Quarkus fast-jar output
podman build -f src/main/docker/Dockerfile.jvm -t casehubio/<name>:latest target/
```

- [ ] **Step 3: Start scaffold in dev mode**

```bash
/opt/homebrew/bin/mvn --batch-mode quarkus:dev -f /Users/mdproctor/claude/casehub/scaffold/scaffold-backend/pom.xml
```

- [ ] **Step 4: End-to-end verification in browser**

1. Open browser → Deploy tab
2. Catalogue table shows `01-hello-llm` with description, step, prerequisites
3. Select the `01-hello-llm` row — row highlights
4. Click "Deploy Selected Topology" — confirmation dialog: "Deploy topology 01-hello-llm?"
5. Confirm — POST fires to `/api/scaffold/catalogue/01-hello-llm/deploy`
6. Verify response: 201 Created (or observe deployment progress)
7. Check Podman: `podman ps` — should show 4 containers in dependency order
8. Cases tab shows a new `application-lifecycle` case

- [ ] **Step 5: Verify failure modes**

1. Click Deploy with no row selected — button should be disabled or show error
2. Click Deploy again on already-running topology — should get 409 Conflict
3. Stop Podman machine, click Deploy — should fail with connection error, not crash

- [ ] **Step 6: Final commit if any adjustments were needed**

```bash
git add -A
git commit -m "feat(#83): end-to-end deploy Level 01 from UI verified

Verified: select topology in Deploy tab, click Deploy, 4 Podman
containers provision in dependency order (manifest → eidos → engine →
my-llm), application-lifecycle case created.

Closes #86 Closes #83"
```

---

## References

- [2026-10-29-deploy-level-01-e2e-design.md] — design spec this plan implements
- [activation.ts:328-330] — selectionKey wiring (crash site)
- [CaseHubAppProvisionHandler.java:83-89] — resolveTarget() override point
- [CaseHubAppDriftChecker.java:59-65] — resolveTarget() duplicate
- [CaseHubAppProvisionHandlerTest.java] — existing handler tests
- [CaseHubAppDriftCheckerTest.java] — existing drift checker tests with stubInstance()
- [deploy.ts] — scaffold deploy view (catalogue table)
- [aggregator-convergence.xml] — reactor convergence module list
- [mvn-all.yml] — casehub-all CI workflow
- [maven.yml] — scaffold CI workflow (missing push trigger)
- [GitHub #83] — epic: Deploy Level 01 from the UI — end-to-end
- [GitHub #84] — fix catalogue table selection + wire deploy button
- [GitHub #85] — land cross-repo branches
- [GitHub #86] — Level 01 deployable locally via Podman
