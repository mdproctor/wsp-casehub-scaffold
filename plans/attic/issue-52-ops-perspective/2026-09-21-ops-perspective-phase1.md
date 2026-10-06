# Ops Perspective Phase 1 — MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #52 — Ops Perspective
**Issue group:** #52

**Goal:** Deliver the MVP ops perspective: scaffold reads a perspective YAML, deploys a single Java app to Podman with PostgreSQL, monitors health, detects drift with manual re-convergence, all orchestrated as engine cases.

**Architecture:** The perspective concept lives in scaffold (new `PerspectiveLoader` reads `perspective.yaml` at boot, replaces classpath probing for module/tab discovery). The Podman provisioning layer is a new ops module (`ops-container`) implementing the DesiredState SPI quad. Case-managed provisioning uses existing `ApplicationCaseDescriptor` from ops/app. The UI is pages-DSL-driven, replacing the hardcoded tab list with perspective-declared tabs.

**Tech Stack:** Java 21, Quarkus 3.32.2, DesiredState API, casehub-pages DSL, Lit web components, Podman REST API, PostgreSQL

## Global Constraints

- Quarkus version: `3.32.2` (from root `pom.xml` `version.quarkus.platform`)
- All new modules use the casehubio parent POM pattern
- DesiredState SPI implementations follow the existing quad pattern (GoalCompiler, ActualStateAdapter, NodeProvisioner, FaultPolicy)
- All REST endpoints use JAX-RS with `@Path`, `@Produces(APPLICATION_JSON)`
- Frontend uses casehub-pages DSL (`rows`, `columns`, `dataTable`, `hostPanel`, `tabs`)
- Podman API accessed via REST socket (`$XDG_RUNTIME_DIR/podman/podman.sock`)
- Commits reference `Refs #52`

---

## Batch 1: Perspective Loading [scaffold reads perspective.yaml and renders declared tabs]

### Task 1: Perspective YAML Parser and Schema

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/perspective/PerspectiveConfig.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/perspective/PerspectiveLoader.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/perspective/PerspectiveModule.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/perspective/PerspectiveTab.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/perspective/PerspectiveDataset.java`
- Modify: `scaffold-backend/src/main/java/io/casehub/flow/rest/ModuleRegistryResource.java:14-42`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/perspective/PerspectiveLoaderTest.java`

**Interfaces:**
- Produces: `PerspectiveConfig` record — `name()`, `description()`, `version()`, `modules()` (PerspectiveModule), `tabs()` (List<PerspectiveTab>), `datasets()` (List<PerspectiveDataset>)
- Produces: `PerspectiveLoader.load()` — returns `Optional<PerspectiveConfig>`, reads from classpath `META-INF/perspective.yaml`
- Produces: `ModuleRegistryResource.getModules()` — now returns perspective-declared modules when perspective is present, falls back to classpath probing when absent

- [ ] **Step 1: Write the failing test for PerspectiveLoader**

```java
package io.casehub.flow.perspective;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class PerspectiveLoaderTest {

    @Test
    void loadsValidPerspectiveYaml() {
        var yaml = """
                perspective:
                  name: ops
                  description: "CaseHub operations management"
                  version: "1.0"
                  modules:
                    required: [engine, desiredstate, ops-deployment, work]
                    optional: [ops-compliance]
                  views:
                    tabs:
                      - id: fleet
                        label: "Fleet"
                        view: fleet-dashboard
                      - id: deploy
                        label: "Deploy"
                        view: deployment-wizard
                  datasets:
                    deployments:
                      endpoint: /api/ops/deployments
                      source: ops-app
                """;
        var config = PerspectiveLoader.parse(yaml);
        assertThat(config).isPresent();
        assertThat(config.get().name()).isEqualTo("ops");
        assertThat(config.get().modules().required()).containsExactly("engine", "desiredstate", "ops-deployment", "work");
        assertThat(config.get().modules().optional()).containsExactly("ops-compliance");
        assertThat(config.get().tabs()).hasSize(2);
        assertThat(config.get().tabs().get(0).id()).isEqualTo("fleet");
        assertThat(config.get().tabs().get(0).label()).isEqualTo("Fleet");
        assertThat(config.get().datasets()).hasSize(1);
        assertThat(config.get().datasets().get(0).name()).isEqualTo("deployments");
    }

    @Test
    void returnsEmptyForMissingPerspective() {
        var config = PerspectiveLoader.parse("");
        assertThat(config).isEmpty();
    }

    @Test
    void returnsEmptyForInvalidYaml() {
        var config = PerspectiveLoader.parse("not: valid: yaml: {{");
        assertThat(config).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl scaffold-backend -Dtest=PerspectiveLoaderTest -f /Users/mdproctor/claude/casehub/scaffold/pom.xml`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Implement the perspective records and loader**

```java
package io.casehub.flow.perspective;

public record PerspectiveConfig(
    String name,
    String description,
    String version,
    PerspectiveModule modules,
    java.util.List<PerspectiveTab> tabs,
    java.util.List<PerspectiveDataset> datasets
) {}
```

```java
package io.casehub.flow.perspective;

import java.util.List;

public record PerspectiveModule(
    List<String> required,
    List<String> optional
) {
    public PerspectiveModule {
        required = required != null ? List.copyOf(required) : List.of();
        optional = optional != null ? List.copyOf(optional) : List.of();
    }
}
```

```java
package io.casehub.flow.perspective;

public record PerspectiveTab(String id, String label, String view) {}
```

```java
package io.casehub.flow.perspective;

public record PerspectiveDataset(String name, String endpoint, String source) {}
```

```java
package io.casehub.flow.perspective;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;

public class PerspectiveLoader {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory());

    public static Optional<PerspectiveConfig> parse(String yaml) {
        if (yaml == null || yaml.isBlank()) return Optional.empty();
        try {
            var tree = YAML.readTree(yaml);
            var perspective = tree.get("perspective");
            if (perspective == null) return Optional.empty();

            var name = perspective.path("name").asText();
            var description = perspective.path("description").asText("");
            var version = perspective.path("version").asText("1.0");

            var modulesNode = perspective.path("modules");
            var required = readStringList(modulesNode, "required");
            var optional = readStringList(modulesNode, "optional");
            var modules = new PerspectiveModule(required, optional);

            var tabsList = new ArrayList<PerspectiveTab>();
            var viewsNode = perspective.path("views").path("tabs");
            if (viewsNode.isArray()) {
                for (var tabNode : viewsNode) {
                    tabsList.add(new PerspectiveTab(
                        tabNode.path("id").asText(),
                        tabNode.path("label").asText(),
                        tabNode.path("view").asText()
                    ));
                }
            }

            var datasetsList = new ArrayList<PerspectiveDataset>();
            var datasetsNode = perspective.path("datasets");
            if (datasetsNode.isObject()) {
                var it = datasetsNode.fields();
                while (it.hasNext()) {
                    var entry = it.next();
                    datasetsList.add(new PerspectiveDataset(
                        entry.getKey(),
                        entry.getValue().path("endpoint").asText(),
                        entry.getValue().path("source").asText()
                    ));
                }
            }

            return Optional.of(new PerspectiveConfig(name, description, version, modules, tabsList, datasetsList));
        } catch (Exception e) {
            return Optional.empty();
        }
    }

    public static Optional<PerspectiveConfig> loadFromClasspath() {
        try (InputStream is = Thread.currentThread().getContextClassLoader()
                .getResourceAsStream("META-INF/perspective.yaml")) {
            if (is == null) return Optional.empty();
            return parse(new String(is.readAllBytes()));
        } catch (Exception e) {
            return Optional.empty();
        }
    }

    private static List<String> readStringList(com.fasterxml.jackson.databind.JsonNode parent, String field) {
        var node = parent.path(field);
        if (!node.isArray()) return List.of();
        var list = new ArrayList<String>();
        for (var item : node) list.add(item.asText());
        return list;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl scaffold-backend -Dtest=PerspectiveLoaderTest -f /Users/mdproctor/claude/casehub/scaffold/pom.xml`
Expected: PASS

- [ ] **Step 5: Write test for ModuleRegistryResource with perspective**

```java
// Add to ModuleRegistryResourceTest.java
@Test
void usesPerpsectiveDeclaredModulesWhenPresent() {
    var config = new PerspectiveConfig("ops", "test", "1.0",
        new PerspectiveModule(List.of("engine", "ops-deployment"), List.of("ops-compliance")),
        List.of(), List.of());
    var resource = new ModuleRegistryResource(Optional.of(config));
    var result = resource.getModules();
    assertThat(result.get("modules")).contains("engine", "ops-deployment");
}
```

- [ ] **Step 6: Modify ModuleRegistryResource to accept perspective**

Update `ModuleRegistryResource` to accept an optional `PerspectiveConfig` via CDI. When present, use perspective-declared modules (validating required modules are on classpath). When absent, fall back to classpath probing (current behaviour).

- [ ] **Step 7: Run all tests**

Run: `mvn test -pl scaffold-backend -f /Users/mdproctor/claude/casehub/scaffold/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/perspective/ scaffold-backend/src/test/java/io/casehub/flow/perspective/ scaffold-backend/src/main/java/io/casehub/flow/rest/ModuleRegistryResource.java scaffold-backend/src/test/java/io/casehub/flow/rest/ModuleRegistryResourceTest.java
git commit -m "feat(#52): perspective YAML parser and module activation Refs #52"
```

### Task 2: Perspective-Driven REST Endpoint for Frontend Tab Discovery

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/rest/PerspectiveResource.java`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/rest/PerspectiveResourceTest.java`

**Interfaces:**
- Consumes: `PerspectiveLoader.loadFromClasspath()` → `Optional<PerspectiveConfig>`
- Produces: `GET /api/perspective` → JSON `{ "name": "ops", "tabs": [...], "datasets": [...] }` or 404 when no perspective

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.flow.rest;

import io.casehub.flow.perspective.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import static org.assertj.core.api.Assertions.assertThat;

class PerspectiveResourceTest {

    @Test
    void returnsPerspectiveWhenPresent() {
        var config = new PerspectiveConfig("ops", "test", "1.0",
            new PerspectiveModule(List.of("engine"), List.of()),
            List.of(new PerspectiveTab("fleet", "Fleet", "fleet-dashboard")),
            List.of(new PerspectiveDataset("deployments", "/api/ops/deployments", "ops-app")));
        var resource = new PerspectiveResource(Optional.of(config));

        var response = resource.getPerspective();
        assertThat(response.getStatus()).isEqualTo(200);
    }

    @Test
    void returns404WhenNoPerspective() {
        var resource = new PerspectiveResource(Optional.empty());
        var response = resource.getPerspective();
        assertThat(response.getStatus()).isEqualTo(404);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl scaffold-backend -Dtest=PerspectiveResourceTest -f /Users/mdproctor/claude/casehub/scaffold/pom.xml`
Expected: FAIL

- [ ] **Step 3: Implement PerspectiveResource**

```java
package io.casehub.flow.rest;

import io.casehub.flow.perspective.PerspectiveConfig;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.Optional;

@Path("/api/perspective")
@Produces(MediaType.APPLICATION_JSON)
public class PerspectiveResource {

    private final Optional<PerspectiveConfig> config;

    @Inject
    public PerspectiveResource(Optional<PerspectiveConfig> config) {
        this.config = config;
    }

    @GET
    public Response getPerspective() {
        return config
            .map(c -> Response.ok(c).build())
            .orElse(Response.status(404).build());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl scaffold-backend -Dtest=PerspectiveResourceTest -f /Users/mdproctor/claude/casehub/scaffold/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/rest/PerspectiveResource.java scaffold-backend/src/test/java/io/casehub/flow/rest/PerspectiveResourceTest.java
git commit -m "feat(#52): perspective REST endpoint for frontend tab discovery Refs #52"
```

### Task 3: Frontend Perspective-Driven Tab Rendering

**Files:**
- Modify: `scaffold-backend/src/main/webui/src/index.ts:1-63`
- Modify: `scaffold-backend/src/main/webui/src/datasets.ts:1-43`
- Create: `scaffold-backend/src/main/webui/src/views/fleet.ts`
- Create: `scaffold-backend/src/main/webui/src/views/deploy.ts`
- Create: `scaffold-backend/src/main/webui/src/views/health.ts`
- Create: `scaffold-backend/src/main/webui/src/views/drift.ts`
- Create: `scaffold-backend/src/main/webui/src/views/situations.ts`
- Create: `scaffold-backend/src/main/webui/src/views/updates.ts`
- Modify: `scaffold-backend/src/main/webui/src/navigation.ts`
- Create: `scaffold-backend/src/main/resources/META-INF/perspective.yaml`

**Interfaces:**
- Consumes: `GET /api/perspective` → tab list and datasets
- Consumes: `GET /api/modules` → module list (fallback)
- Produces: Perspective-driven tab rendering when `/api/perspective` returns 200, falls back to current hardcoded tabs when 404

- [ ] **Step 1: Create the ops perspective YAML**

```yaml
# scaffold-backend/src/main/resources/META-INF/perspective.yaml
perspective:
  name: ops
  description: "CaseHub operations management"
  version: "1.0"

  modules:
    required: [engine, desiredstate, work]
    optional: [ops-deployment, ops-compliance]

  views:
    tabs:
      - id: fleet
        label: "Fleet"
        view: fleet-dashboard
      - id: deploy
        label: "Deploy"
        view: deployment-wizard
      - id: health
        label: "Health"
        view: health-monitor
      - id: drift
        label: "Drift"
        view: drift-detection
      - id: situations
        label: "Situations"
        view: situation-manager
      - id: updates
        label: "Updates"
        view: update-manager

  datasets:
    deployments:
      endpoint: /api/ops/deployments
      source: ops-app
    health:
      endpoint: /api/ops/health
      source: ops-app
    drift:
      endpoint: /api/ops/drift
      source: desiredstate
```

- [ ] **Step 2: Create stub ops perspective view files**

Create `fleet.ts`, `deploy.ts`, `health.ts`, `drift.ts`, `situations.ts`, `updates.ts` in `scaffold-backend/src/main/webui/src/views/`. Each is a minimal pages-DSL view following the existing pattern:

```typescript
// fleet.ts
import { rows, columns, dataTable, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";

export const fleetView = rows(
  title("Fleet", "h2"),
  dataTable({
    lookup: lookup("deployments", groupBy(null,
      col("id"), col("name"), col("archetype"),
      col("status"), col("health"),
    )),
    sortable: true,
    selectionTopic: "selected-deployment",
  }),
);
```

Repeat pattern for `deploy.ts` (form layout), `health.ts` (health table), `drift.ts` (drift alerts), `situations.ts` (situation list), `updates.ts` (update table).

- [ ] **Step 3: Modify index.ts for perspective-driven boot**

Update `start()` in `index.ts`:
1. Try `GET /api/perspective` first
2. If 200: build tabs from perspective response, using a view registry mapping `view` names to imported DSL views
3. If 404: fall back to current hardcoded tab behaviour

```typescript
// View registry — maps perspective view names to DSL view objects
const VIEW_REGISTRY: Record<string, unknown> = {
  "fleet-dashboard": fleetView,
  "deployment-wizard": deployView,
  "health-monitor": healthView,
  "drift-detection": driftView,
  "situation-manager": situationsView,
  "update-manager": updatesView,
};

async function start() {
  const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;

  const perspectiveResp = await fetch("/api/perspective");
  let allTabs: TabEntry[];
  let datasets;

  if (perspectiveResp.ok) {
    const perspective = await perspectiveResp.json();
    allTabs = perspective.tabs
      .map((t: { label: string; view: string }) => [t.label, VIEW_REGISTRY[t.view]])
      .filter(([, v]: TabEntry) => v != null);
    datasets = createPerspectiveDatasets(perspective.datasets);
  } else {
    // Fallback: existing module-detection behaviour
    const { modules } = await fetch("/api/modules").then(r => r.json());
    allTabs = buildDefaultTabs(modules);
    datasets = createDatasets(modules);
  }

  // ... rest of start() unchanged
}
```

- [ ] **Step 4: Update navigation.ts for perspective tabs**

Add perspective tab slugs and topics to the navigation mappings, or make the navigation system dynamic based on the perspective response.

- [ ] **Step 5: Update datasets.ts for perspective datasets**

Add `createPerspectiveDatasets(datasets)` function that converts perspective dataset declarations to the same `bind(id, restSource(...))` format.

- [ ] **Step 6: Verify scaffold builds and boots with perspective**

Run: `mvn install -pl scaffold-backend -DskipTests -f /Users/mdproctor/claude/casehub/scaffold/pom.xml`
Expected: BUILD SUCCESS

Start scaffold and verify the ops perspective tabs render instead of the default console tabs.

- [ ] **Step 7: Commit**

```bash
git add scaffold-backend/src/main/webui/src/ scaffold-backend/src/main/resources/META-INF/perspective.yaml
git commit -m "feat(#52): perspective-driven tab rendering with ops perspective views Refs #52"
```

---

## Batch 2: Podman Provisioning Layer [ops-container module, DesiredState SPI for containers]

### Task 4: ContainerNodeSpec Hierarchy

**Files:**
- Create: `ops/container/pom.xml`
- Create: `ops/container/src/main/java/io/casehub/ops/container/ContainerNodeSpec.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/AppContainerSpec.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/DatabaseContainerSpec.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/NetworkSpec.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/VolumeSpec.java`
- Modify: `ops/pom.xml` (add `<module>container</module>`)
- Test: `ops/container/src/test/java/io/casehub/ops/container/ContainerNodeSpecTest.java`

**Interfaces:**
- Produces: `ContainerNodeSpec` sealed interface extending `NodeSpec` with `nodeType()` returning container-specific `NodeType`s
- Produces: `AppContainerSpec` record — `image`, `ports` (Map<Integer,Integer>), `env` (Map<String,String>), `volumeMounts` (Map<String,String>), `name`
- Produces: `DatabaseContainerSpec` record — `image` (default `postgres:16`), `databaseName`, `username`, `password`, `dataVolume`
- Produces: `NetworkSpec` record — `name`
- Produces: `VolumeSpec` record — `name`, `driver` (default `local`)

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.NodeType;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class ContainerNodeSpecTest {

    @Test
    void appContainerSpecHasCorrectNodeType() {
        var spec = new AppContainerSpec("my-app", "casehub/scaffold:latest",
            Map.of(8080, 8080), Map.of("QUARKUS_PROFILE", "prod"), Map.of());
        assertThat(spec.nodeType()).isEqualTo(ContainerNodeTypes.APP_CONTAINER);
        assertThat(spec).isInstanceOf(ContainerNodeSpec.class);
    }

    @Test
    void databaseContainerSpecDefaults() {
        var spec = new DatabaseContainerSpec("my-db", "testdb", "admin", "secret", "db-data");
        assertThat(spec.nodeType()).isEqualTo(ContainerNodeTypes.DATABASE);
        assertThat(spec.image()).isEqualTo("postgres:16");
    }

    @Test
    void networkSpecHasCorrectType() {
        var spec = new NetworkSpec("ops-net");
        assertThat(spec.nodeType()).isEqualTo(ContainerNodeTypes.NETWORK);
    }

    @Test
    void volumeSpecHasCorrectType() {
        var spec = new VolumeSpec("data-vol", "local");
        assertThat(spec.nodeType()).isEqualTo(ContainerNodeTypes.VOLUME);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl container -Dtest=ContainerNodeSpecTest -f /Users/mdproctor/claude/casehub/ops/pom.xml`
Expected: FAIL — module and classes do not exist

- [ ] **Step 3: Create ops-container Maven module**

Create `ops/container/pom.xml` with parent `casehub-ops-parent`, dependency on `casehub-desiredstate-api`. Add `<module>container</module>` to `ops/pom.xml`.

- [ ] **Step 4: Implement the node spec hierarchy**

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.NodeSpec;

public sealed interface ContainerNodeSpec extends NodeSpec
    permits AppContainerSpec, DatabaseContainerSpec, NetworkSpec, VolumeSpec {}
```

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.NodeType;
import java.util.Map;

public record AppContainerSpec(
    String name,
    String image,
    Map<Integer, Integer> ports,
    Map<String, String> env,
    Map<String, String> volumeMounts
) implements ContainerNodeSpec {
    @Override public NodeType nodeType() { return ContainerNodeTypes.APP_CONTAINER; }
}
```

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.NodeType;

public record DatabaseContainerSpec(
    String name,
    String databaseName,
    String username,
    String password,
    String dataVolume
) implements ContainerNodeSpec {
    @Override public NodeType nodeType() { return ContainerNodeTypes.DATABASE; }
    public String image() { return "postgres:16"; }
}
```

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.NodeType;

public record NetworkSpec(String name) implements ContainerNodeSpec {
    @Override public NodeType nodeType() { return ContainerNodeTypes.NETWORK; }
}
```

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.NodeType;

public record VolumeSpec(String name, String driver) implements ContainerNodeSpec {
    @Override public NodeType nodeType() { return ContainerNodeTypes.VOLUME; }
}
```

```java
package io.casehub.ops.container;

import io.casehub.desiredstate.api.NodeType;

public final class ContainerNodeTypes {
    public static final NodeType APP_CONTAINER = NodeType.of("container:app");
    public static final NodeType DATABASE = NodeType.of("container:database");
    public static final NodeType NETWORK = NodeType.of("container:network");
    public static final NodeType VOLUME = NodeType.of("container:volume");
    private ContainerNodeTypes() {}
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl container -Dtest=ContainerNodeSpecTest -f /Users/mdproctor/claude/casehub/ops/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ops add container/ pom.xml
git -C /Users/mdproctor/claude/casehub/ops commit -m "feat(#52): ContainerNodeSpec hierarchy for Podman provisioning Refs casehubio/scaffold#52"
```

### Task 5: ContainerGoalCompiler

**Files:**
- Create: `ops/container/src/main/java/io/casehub/ops/container/ContainerGoalCompiler.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/DeploymentDescriptor.java`
- Test: `ops/container/src/test/java/io/casehub/ops/container/ContainerGoalCompilerTest.java`

**Interfaces:**
- Consumes: `DeploymentDescriptor` — `appName`, `image`, `ports`, `env`, `databaseName`
- Produces: `ContainerGoalCompiler implements GoalCompiler<DeploymentDescriptor>` — compiles a deployment descriptor into a `DesiredStateGraph` with network → database → app container dependency chain

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ops.container;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class ContainerGoalCompilerTest {

    @Test
    void compilesDeploymentWithDependencyChain() {
        var descriptor = new DeploymentDescriptor(
            "fsitrading", "casehub/fsitrading:latest",
            Map.of(8080, 18080), Map.of("QUARKUS_PROFILE", "prod"),
            "fsitrading_db");

        var compiler = new ContainerGoalCompiler();
        var result = compiler.compile(descriptor, new TestGraphFactory());

        assertThat(result.graph().nodes()).hasSize(4); // network, volume, database, app
        var appNode = result.graph().nodeByType(ContainerNodeTypes.APP_CONTAINER);
        assertThat(appNode).isNotNull();
        assertThat(appNode.dependsOn()).contains(
            result.graph().nodeByType(ContainerNodeTypes.DATABASE).id(),
            result.graph().nodeByType(ContainerNodeTypes.NETWORK).id()
        );
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement DeploymentDescriptor and ContainerGoalCompiler**

`DeploymentDescriptor`: record with `appName`, `image`, `ports`, `env`, `databaseName`.

`ContainerGoalCompiler`: creates a graph with:
- `NetworkSpec` node (no dependencies)
- `VolumeSpec` node for DB data (no dependencies)
- `DatabaseContainerSpec` node (depends on network + volume)
- `AppContainerSpec` node (depends on database + network), env includes `QUARKUS_DATASOURCE_JDBC_URL` pointing to the database container

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ops add container/
git -C /Users/mdproctor/claude/casehub/ops commit -m "feat(#52): ContainerGoalCompiler — deployment descriptor to DesiredState graph Refs casehubio/scaffold#52"
```

### Task 6: PodmanClient — REST API Client for Podman Socket

**Files:**
- Create: `ops/container/src/main/java/io/casehub/ops/container/podman/PodmanClient.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/podman/PodmanContainer.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/podman/PodmanNetwork.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/podman/PodmanVolume.java`
- Test: `ops/container/src/test/java/io/casehub/ops/container/podman/PodmanClientTest.java`

**Interfaces:**
- Produces: `PodmanClient` — `listContainers()`, `createContainer(CreateContainerRequest)`, `startContainer(String id)`, `stopContainer(String id)`, `removeContainer(String id)`, `listNetworks()`, `createNetwork(String name)`, `listVolumes()`, `createVolume(String name, String driver)`, `pullImage(String image)`
- Produces: `PodmanContainer` record — `id`, `names`, `state`, `image`, `ports`
- Produces: `PodmanNetwork` record — `name`, `id`
- Produces: `PodmanVolume` record — `name`, `driver`

- [ ] **Step 1: Write the failing test with WireMock**

Use WireMock to stub Podman REST API responses. Test `listContainers()` returns parsed container list.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement PodmanClient**

HTTP client using `java.net.http.HttpClient` with Unix socket support (Quarkus provides this). Targets `$XDG_RUNTIME_DIR/podman/podman.sock`. Endpoints: `/v5.0.0/libpod/containers/json`, `/v5.0.0/libpod/containers/create`, `/v5.0.0/libpod/networks/json`, `/v5.0.0/libpod/volumes/json`, etc.

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ops add container/
git -C /Users/mdproctor/claude/casehub/ops commit -m "feat(#52): PodmanClient — REST API client for Podman socket Refs casehubio/scaffold#52"
```

### Task 7: PodmanActualStateAdapter and PodmanNodeProvisioner

**Files:**
- Create: `ops/container/src/main/java/io/casehub/ops/container/PodmanActualStateAdapter.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/PodmanNodeProvisioner.java`
- Create: `ops/container/src/main/java/io/casehub/ops/container/ContainerFaultPolicy.java`
- Test: `ops/container/src/test/java/io/casehub/ops/container/PodmanActualStateAdapterTest.java`
- Test: `ops/container/src/test/java/io/casehub/ops/container/PodmanNodeProvisionerTest.java`

**Interfaces:**
- Consumes: `PodmanClient` — all container/network/volume operations
- Consumes: `ContainerNodeSpec` hierarchy — node type dispatch
- Produces: `PodmanActualStateAdapter implements ActualStateAdapter` — `readActual()` queries Podman API, maps container state to `NodeStatus` (running→PRESENT, exited→DRIFTED, absent→ABSENT)
- Produces: `PodmanNodeProvisioner implements NodeProvisioner` — sealed type dispatch: creates/starts containers, networks, volumes via `PodmanClient`
- Produces: `ContainerFaultPolicy implements FaultPolicy` — after 3 consecutive PROVISION_FAILED, adds human review node

- [ ] **Step 1: Write the failing test for PodmanActualStateAdapter**

Test with mocked `PodmanClient`: one running container, one missing container → asserts PRESENT and ABSENT `NodeStatus`.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement PodmanActualStateAdapter**

`readActual()` lists containers/networks/volumes from Podman, matches against desired graph nodes by name/label, returns `ActualState` with `NodeStatus` per node.

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write the failing test for PodmanNodeProvisioner**

Test with mocked `PodmanClient`: provision a `DatabaseContainerSpec` → asserts `createContainer` and `startContainer` called with correct params.

- [ ] **Step 6: Run test to verify it fails**

- [ ] **Step 7: Implement PodmanNodeProvisioner**

Sealed type dispatch on `ContainerNodeSpec` subtypes. Each handler calls `PodmanClient` methods: pull image → create container → start container. For databases, sets `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` env vars.

- [ ] **Step 8: Run test to verify it passes**

- [ ] **Step 9: Implement ContainerFaultPolicy**

Threshold-based: after 3 consecutive `PROVISION_FAILED` for a node, returns `HUMAN_REVIEW` action.

- [ ] **Step 10: Run all container module tests**

Run: `mvn test -pl container -f /Users/mdproctor/claude/casehub/ops/pom.xml`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ops add container/
git -C /Users/mdproctor/claude/casehub/ops commit -m "feat(#52): PodmanActualStateAdapter + PodmanNodeProvisioner — DesiredState SPI quad Refs casehubio/scaffold#52"
```

---

## Batch 3: Fleet Management and Health [deployment registry, health polling, Fleet/Deploy UI]

### Task 8: Deployment Registry and REST API

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/DeploymentRegistry.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/Deployment.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/DeploymentArchetype.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/DeploymentStatus.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/OpsDeploymentResource.java`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/ops/DeploymentRegistryTest.java`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/ops/OpsDeploymentResourceTest.java`

**Interfaces:**
- Produces: `Deployment` record — `id`, `name`, `archetype` (YAML/JAVA/HYBRID), `gitUrl`, `status` (PROVISIONING/RUNNING/DEGRADED/STOPPED), `health`, `containerIds`
- Produces: `DeploymentRegistry` CDI bean — `register(Deployment)`, `findById(String)`, `listAll()`, `updateStatus(String, DeploymentStatus)`
- Produces: `GET /api/ops/deployments` → list of deployments with status and health
- Produces: `POST /api/ops/deployments` → trigger new deployment (creates provisioning case)

- [ ] **Step 1: Write the failing test for DeploymentRegistry**

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement Deployment records and DeploymentRegistry**

In-memory registry for MVP (no persistence). `DeploymentArchetype` enum: `YAML`, `JAVA`, `HYBRID`.

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write the failing test for OpsDeploymentResource**

- [ ] **Step 6: Implement OpsDeploymentResource**

`GET /api/ops/deployments` returns all deployments from registry. `POST /api/ops/deployments` accepts a JSON body with `gitUrl` and `appName`, detects archetype (presence of `pom.xml` → JAVA, else YAML), creates a provisioning case via engine API.

- [ ] **Step 7: Run all tests**

- [ ] **Step 8: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/ops/ scaffold-backend/src/test/java/io/casehub/flow/ops/
git commit -m "feat(#52): deployment registry and REST API for fleet management Refs #52"
```

### Task 9: Health Polling and Fleet Tab Wiring

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/HealthPoller.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/OpsHealthResource.java`
- Modify: `scaffold-backend/src/main/webui/src/views/fleet.ts`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/ops/HealthPollerTest.java`

**Interfaces:**
- Consumes: `DeploymentRegistry.listAll()` → active deployments
- Consumes: `PodmanClient.listContainers()` → container state
- Produces: `HealthPoller` CDI bean with `@Scheduled(every = "10s")` — polls container health for each deployment, updates `DeploymentRegistry` status
- Produces: `GET /api/ops/health` → aggregated health across all deployments

- [ ] **Step 1: Write the failing test for HealthPoller**

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement HealthPoller**

Quarkus `@Scheduled` bean. For each registered deployment, queries `PodmanClient` for container state. Maps to health status. Updates registry.

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Implement OpsHealthResource**

`GET /api/ops/health` returns per-deployment health with container-level detail.

- [ ] **Step 6: Wire Fleet tab to live data**

Update `fleet.ts` to use the `deployments` dataset (bound to `/api/ops/deployments`). Show deployment cards with name, archetype badge, health status indicator, convergence state.

- [ ] **Step 7: Wire Deploy tab**

Update `deploy.ts` with a form: git repo URL input, app name, topology selection. POST to `/api/ops/deployments` on submit.

- [ ] **Step 8: Verify end-to-end**

Start scaffold with ops perspective, navigate to Fleet tab, verify empty state. Navigate to Deploy tab, verify form renders.

- [ ] **Step 9: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/ops/ scaffold-backend/src/test/java/io/casehub/flow/ops/ scaffold-backend/src/main/webui/src/views/
git commit -m "feat(#52): health polling + Fleet/Deploy tab wiring Refs #52"
```

---

## Batch 4: Case-Managed Provisioning [engine integration, provisioning case lifecycle]

### Task 10: Provisioning Case Integration

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/ProvisioningCaseManager.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/ProvisioningWorker.java`
- Modify: `scaffold-backend/src/main/java/io/casehub/flow/ops/OpsDeploymentResource.java`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/ops/ProvisioningCaseManagerTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime` (engine API) — `createCase()`, `signal()`
- Consumes: `ContainerGoalCompiler` — compiles deployment descriptor to DesiredState graph
- Consumes: `PodmanNodeProvisioner` — provisions containers
- Consumes: `PodmanActualStateAdapter` — reads actual state
- Produces: `ProvisioningCaseManager` — `provision(DeploymentDescriptor)` creates an engine case, runs the provisioning workflow
- Produces: `ProvisioningWorker` — engine worker implementation that executes provisioning tasks (compile graph → provision infra → deploy app → health check)

- [ ] **Step 1: Write the failing test**

Test that `provision()` creates a case via the engine, and the case progresses through provisioning stages.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement ProvisioningCaseManager**

Creates a case using the engine API. The case definition has capabilities: `validate-spec`, `provision-infrastructure`, `deploy-app`, `health-check`. Each capability maps to a stage of the provisioning lifecycle.

- [ ] **Step 4: Implement ProvisioningWorker**

Engine worker that claims provisioning tasks. For `provision-infrastructure`: compiles DeploymentDescriptor via ContainerGoalCompiler, runs TransitionPlanner + TransitionExecutor for network, volume, database nodes. For `deploy-app`: provisions the app container node. For `health-check`: polls container health until healthy or timeout.

- [ ] **Step 5: Wire into OpsDeploymentResource**

`POST /api/ops/deployments` now calls `ProvisioningCaseManager.provision()`, which creates the case. The worker executes the provisioning asynchronously. Returns the case ID for tracking.

- [ ] **Step 6: Run tests**

- [ ] **Step 7: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/ops/ scaffold-backend/src/test/java/io/casehub/flow/ops/
git commit -m "feat(#52): case-managed provisioning — engine cases drive deployment lifecycle Refs #52"
```

---

## Batch 5: Drift Detection and Manual Reconciliation [convergence loop, drift UI]

### Task 11: Drift Detection via ReconciliationLoop

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/DriftDetectionService.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/ops/OpsDriftResource.java`
- Modify: `scaffold-backend/src/main/webui/src/views/drift.ts`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/ops/DriftDetectionServiceTest.java`

**Interfaces:**
- Consumes: `PodmanActualStateAdapter` — reads actual container state
- Consumes: `DeploymentRegistry` — active deployments with their DesiredState graphs
- Produces: `DriftDetectionService` — `@Scheduled(every = "30s")`, for each deployment compares desired vs actual state, emits drift events
- Produces: `GET /api/ops/drift` → list of active drift alerts with deployment ID, node, expected vs actual
- Produces: `POST /api/ops/drift/{id}/reconcile` → triggers manual re-convergence for a specific drift

- [ ] **Step 1: Write the failing test**

Test with a deployment whose database container has stopped → drift detected with status DRIFTED.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement DriftDetectionService**

Scheduled bean. For each deployment, calls `PodmanActualStateAdapter.readActual()` against the stored DesiredState graph. Nodes with status != PRESENT are drift alerts. Stores drift state in-memory.

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Implement OpsDriftResource**

`GET /api/ops/drift` returns all active drift alerts. `POST /api/ops/drift/{id}/reconcile` triggers `TransitionPlanner` + `TransitionExecutor` for the drifted nodes.

- [ ] **Step 6: Wire Drift tab**

Update `drift.ts` to show drift alerts from the `drift` dataset. Each alert shows deployment name, drifted node, expected state, actual state, and a "Reconcile" button that POSTs to the reconcile endpoint.

- [ ] **Step 7: Run all tests**

Run: `mvn test -pl scaffold-backend -f /Users/mdproctor/claude/casehub/scaffold/pom.xml`
Expected: PASS

- [ ] **Step 8: End-to-end verification**

Start scaffold with ops perspective. Deploy an app via Deploy tab. Stop the database container manually (`podman stop`). Verify drift appears in Drift tab. Click Reconcile. Verify container restarts.

- [ ] **Step 9: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/ops/ scaffold-backend/src/test/java/io/casehub/flow/ops/ scaffold-backend/src/main/webui/src/views/drift.ts
git commit -m "feat(#52): drift detection and manual reconciliation — completes Phase 1 MVP Refs #52"
```

---

## References

- [2026-09-21-ops-perspective-design.md] — design spec this plan implements (§9 Phase 1)
- [decisions.md] — 12 validated decisions (D1–D12)
- [platform-investigation.md] — platform investigation synthesis
- scaffold-backend/src/main/java/io/casehub/flow/rest/ModuleRegistryResource.java:14 — existing module detection
- scaffold-backend/src/main/webui/src/index.ts:1 — console entry point
- scaffold-backend/src/main/webui/src/views/operations.ts:1 — existing ops view pattern
- scaffold-backend/src/main/webui/src/datasets.ts:1 — existing dataset binding pattern
- io.casehub.desiredstate.api.GoalCompiler — DesiredState SPI interface
- io.casehub.desiredstate.api.ActualStateAdapter — DesiredState SPI interface
- io.casehub.desiredstate.api.NodeSpec — node spec interface
- io.casehub.ops.app.case_.ApplicationCaseDescriptor — existing ops case descriptor
- io.casehub.ops.app.rest.ApplicationResource — existing ops REST resource
- GitHub casehubio/scaffold#52
