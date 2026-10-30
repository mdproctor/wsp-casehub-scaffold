# Topology Catalogue + Deploy — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #76 — Topology catalogue + deploy
**Issue group:** #76

**Goal:** Add a topology catalogue to scaffold's Deploy tab — browse available topologies, view a deploy plan with DAG visualisation, and deploy with case lifecycle creation.

**Architecture:** Three REST endpoints at `/api/scaffold/catalogue` backed by `TopologyCatalogueService` (dual-impl CDI pattern). Topology YAML read from filesystem via `casehub.ops.topology.path`. Deploy compiles YAML via `TopologyCompilerService`, starts the reconciliation loop, and creates an `application-lifecycle` case. Frontend restructures the Deploy tab with a catalogue table, step viewer, DAG diagram, and plan table.

**Tech Stack:** Quarkus 3.32.2, casehub-ops TopologyCompilerService, casehub-desiredstate TransitionPlanner, casehub-engine EngineCaseApi, pages-ui TypeScript DSL, mosaic graph-renderer

## Global Constraints

- REST endpoints at `/api/scaffold/catalogue` (not `/topologies` — avoids collision with existing `/api/scaffold/topology`)
- Topology source: filesystem via `casehub.ops.topology.path` config property
- Deploy creates both reconciliation loop AND application-lifecycle case
- Plan preserves `List<List<OrderedStep>>` layered structure from TransitionPlan — no flattening
- Deploy has failure compensation: if case creation fails after reconciliation starts, stop the loop

---

## Batch 1: Ops Cross-Repo Changes + Scaffold DTOs

### Task 1: Extract compileTopology() and Change deploy() Return Type (ops repo)

**Files:**
- Modify: `ops/service/src/main/java/io/casehub/ops/service/topology/TopologyCompilerService.java`
- Test: existing tests in `ops/topology-tests/`

**Interfaces:**
- Produces: `compileTopology(String name) → DesiredStateGraph`, `deploy(tenancyId, topologyName) → String` (loopKey)

- [ ] **Step 1: Write test for compileTopology()**

```java
@Test
void compileTopologyReturnsGraphWithAllNodes() {
    var service = createServiceWithPath("src/test/resources/topologies/deployment");
    DesiredStateGraph graph = service.compileTopology("01-hello-llm");

    assertThat(graph.nodes()).hasSize(4);
    assertThat(graph.nodes().keySet().stream().map(NodeId::value).toList())
            .containsExactlyInAnyOrder("manifest", "eidos", "engine", "my-llm");
    assertThat(graph.dependencies()).isNotEmpty();
}
```

- [ ] **Step 2: Run test — expect compilation failure**

Run: `mvn test -pl service -Dtest=TopologyCompilerServiceTest#compileTopologyReturnsGraphWithAllNodes --batch-mode`
Expected: FAIL — `compileTopology` method doesn't exist

- [ ] **Step 3: Extract compileTopology() from deploy()**

In `TopologyCompilerService`, extract the YAML iteration, `compileDomain()` calls, and `GRAPH_FACTORY.of(allNodes, allDeps)` composition from `deploy()` into a new public method:

```java
public DesiredStateGraph compileTopology(String topologyName) {
    String topologyDir = topologyPath.orElseThrow(() ->
            new IllegalStateException("casehub.ops.topology.path not configured"))
            + "/" + topologyName;
    Path dir = Path.of(topologyDir);
    if (!Files.isDirectory(dir)) {
        throw new IllegalArgumentException("Topology directory not found: " + topologyDir);
    }

    Map<String, String> classRegistry = NodeTypeRegistryProducer.scanSealedInterfaces(
            io.casehub.ops.api.deployment.DeploymentNodeSpec.class,
            io.casehub.ops.api.infra.InfraNodeSpec.class);

    try {
        List<Path> yamlFiles;
        try (Stream<Path> files = Files.list(dir)) {
            yamlFiles = files.filter(p -> {
                String name = p.getFileName().toString().toLowerCase();
                return name.endsWith(".yaml") || name.endsWith(".yml");
            })
            .filter(p -> !p.getFileName().toString().startsWith("topology-metadata"))
            .filter(p -> !p.getFileName().toString().equals("playbook.yaml"))
            .sorted().toList();
        }

        if (yamlFiles.isEmpty()) {
            throw new IllegalArgumentException("No topology YAML files in: " + topologyName);
        }

        List<DesiredNode> allNodes = new ArrayList<>();
        List<Dependency> allDeps = new ArrayList<>();

        for (Path yamlFile : yamlFiles) {
            CompilationResult result = compileDomain(yamlFile.toString(), classRegistry);
            if (result instanceof CompilationResult.SingleGraph sg) {
                allNodes.addAll(sg.graph().nodes().values().stream().toList());
                allDeps.addAll(sg.graph().dependencies());
            }
        }

        return GRAPH_FACTORY.of(allNodes, allDeps);
    } catch (IOException e) {
        throw new IllegalStateException("Failed to compile topology: " + topologyName, e);
    }
}
```

Refactor `deploy()` to call `compileTopology()` and return the loopKey:

```java
public String deploy(String tenancyId, String topologiesBasePath, String topologyName) {
    DesiredStateGraph composedGraph = compileTopology(topologyName);
    String loopKey = "topology:" + tenancyId + ":" + topologyName;

    try {
        reconciliationLoop.start(loopKey, composedGraph);
    } catch (IllegalStateException e) {
        reconciliationLoop.updateDesired(loopKey, composedGraph);
    }

    return loopKey;
}
```

- [ ] **Step 4: Run test — expect pass**

Run: `mvn test -pl service -Dtest=TopologyCompilerServiceTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add ops/service/src/main/java/io/casehub/ops/service/topology/TopologyCompilerService.java
git commit -m "refactor(#76): extract compileTopology() + deploy() returns loopKey Refs casehubio/scaffold#76"
```

### Task 2: Add Metadata + Playbook Files to Level 01 (ops repo)

**Files:**
- Create: `ops/topology-tests/src/test/resources/topologies/deployment/01-hello-llm/topology-metadata.yaml`
- Create: `ops/topology-tests/src/test/resources/topologies/deployment/01-hello-llm/playbook.yaml`

**Interfaces:**
- Produces: metadata YAML read by scaffold's catalogue listing; playbook YAML passed to frontend `<pages-step-viewer>`

- [ ] **Step 1: Create topology-metadata.yaml**

```yaml
description: "Deploy a complete LLM app with four services"
methodologyStep: "DEPLOY — First App"
prerequisites: "None"
level: 1
```

- [ ] **Step 2: Create playbook.yaml**

```yaml
- step: "Provision manifest service"
  block:
    - provision-node:
        value: manifest
        type: casehub_application
- step: "Provision eidos (depends on manifest)"
  block:
    - provision-node:
        value: eidos
        type: casehub_application
- step: "Provision engine (depends on eidos)"
  block:
    - provision-node:
        value: engine
        type: casehub_application
- step: "Provision my-llm (depends on engine)"
  block:
    - provision-node:
        value: my-llm
        type: casehub_application
```

- [ ] **Step 3: Verify existing tests still pass**

Run: `mvn test -pl topology-tests --batch-mode`
Expected: PASS — new files don't affect compilation tests (filtered out by extension checks)

- [ ] **Step 4: Commit**

```bash
git add ops/topology-tests/src/test/resources/topologies/deployment/01-hello-llm/topology-metadata.yaml ops/topology-tests/src/test/resources/topologies/deployment/01-hello-llm/playbook.yaml
git commit -m "feat(#76): add topology-metadata.yaml + playbook.yaml for Level 01 Refs casehubio/scaffold#76"
```

### Task 3: Scaffold Service Interface + DTO Records

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/TopologyCatalogueService.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/TopologyEntry.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/TopologyPlan.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/GraphView.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/GraphNode.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/GraphEdge.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/PlanView.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/PlanStep.java`
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/DeployResult.java`

**Interfaces:**
- Produces: `TopologyCatalogueService` interface consumed by Task 4 (embedded impl) and Task 5 (resource)

- [ ] **Step 1: Create interface and records**

```java
// TopologyCatalogueService.java
package io.casehub.flow.service;

import java.util.List;

public interface TopologyCatalogueService {
    List<TopologyEntry> listTopologies();
    TopologyPlan getTopologyPlan(String name);
    DeployResult deploy(String name, String tenancyId);
}
```

```java
// TopologyEntry.java
package io.casehub.flow.service;

public record TopologyEntry(String name, String description,
                             String methodologyStep, String prerequisites) {}
```

```java
// TopologyPlan.java
package io.casehub.flow.service;

public record TopologyPlan(GraphView graph, PlanView plan,
                            String playbookYaml, String readme) {}
```

```java
// GraphView.java
package io.casehub.flow.service;

import java.util.List;

public record GraphView(List<GraphNode> nodes, List<GraphEdge> dependencies) {}
```

```java
// GraphNode.java
package io.casehub.flow.service;

import java.util.List;

public record GraphNode(String id, String type, List<String> dependsOn) {}
```

```java
// GraphEdge.java
package io.casehub.flow.service;

public record GraphEdge(String from, String to) {}
```

```java
// PlanView.java
package io.casehub.flow.service;

import java.util.List;

public record PlanView(List<List<PlanStep>> additions,
                        List<List<PlanStep>> removals,
                        List<List<PlanStep>> suspensions,
                        List<List<PlanStep>> resumptions,
                        boolean empty) {}
```

```java
// PlanStep.java
package io.casehub.flow.service;

public record PlanStep(String nodeId, String action, String type) {}
```

```java
// DeployResult.java — in service package, distinct from ops' DeployResult
package io.casehub.flow.service;

import java.util.UUID;

public record DeployResult(UUID caseId, String loopKey) {}
```

- [ ] **Step 2: Verify compilation**

Run: `mvn compile -pl scaffold-backend --batch-mode -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/service/TopologyCatalogueService.java scaffold-backend/src/main/java/io/casehub/flow/service/TopologyEntry.java scaffold-backend/src/main/java/io/casehub/flow/service/TopologyPlan.java scaffold-backend/src/main/java/io/casehub/flow/service/GraphView.java scaffold-backend/src/main/java/io/casehub/flow/service/GraphNode.java scaffold-backend/src/main/java/io/casehub/flow/service/GraphEdge.java scaffold-backend/src/main/java/io/casehub/flow/service/PlanView.java scaffold-backend/src/main/java/io/casehub/flow/service/PlanStep.java scaffold-backend/src/main/java/io/casehub/flow/service/DeployResult.java
git commit -m "feat(#76): TopologyCatalogueService interface + DTO records Refs #76"
```

## Batch 2: Scaffold Backend — Embedded Impl + REST Resource

### Task 4: EmbeddedTopologyCatalogueService

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/service/embedded/EmbeddedTopologyCatalogueService.java`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/service/embedded/EmbeddedTopologyCatalogueServiceTest.java`

**Interfaces:**
- Consumes: `TopologyCatalogueService` (Task 3), `TopologyCompilerService.compileTopology()` (Task 1), `TopologyCompilerService.deploy()` (Task 1), `TransitionPlanner.plan()`, `EngineCaseApi.startCase()`
- Produces: `EmbeddedTopologyCatalogueService` — `@Alternative @Priority(100)` impl

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.flow.service.embedded;

import io.casehub.api.engine.rest.EngineCaseApi;
import io.casehub.api.view.CaseInstanceView;
import io.casehub.api.model.CaseStatus;
import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.TransitionPlan;
import io.casehub.desiredstate.api.OrderedStep;
import io.casehub.desiredstate.api.StepAction;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.runtime.TransitionPlanner;
import io.casehub.flow.service.TopologyEntry;
import io.casehub.ops.service.topology.TopologyCompilerService;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class EmbeddedTopologyCatalogueServiceTest {

    @Test
    void getTopologyPlanReturnsGraphAndLayeredPlan() {
        var compilerService = mock(TopologyCompilerService.class);
        var engineCaseApi = mock(EngineCaseApi.class);

        var factory = new DefaultDesiredStateGraphFactory();
        var node1 = new DesiredNode(NodeId.of("manifest"),
                mock(io.casehub.desiredstate.api.NodeSpec.class), HumanGating.NONE);
        var node2 = new DesiredNode(NodeId.of("eidos"),
                mock(io.casehub.desiredstate.api.NodeSpec.class), HumanGating.NONE);
        var graph = factory.of(List.of(node1, node2),
                List.of(new io.casehub.desiredstate.api.Dependency(NodeId.of("eidos"), NodeId.of("manifest"))));

        when(compilerService.compileTopology("01-hello-llm")).thenReturn(graph);

        var service = new EmbeddedTopologyCatalogueService(
                compilerService, engineCaseApi, "/test/topologies/deployment");
        var plan = service.getTopologyPlan("01-hello-llm");

        assertThat(plan.graph().nodes()).hasSize(2);
        assertThat(plan.plan().additions()).isNotEmpty();
        assertThat(plan.plan().empty()).isFalse();
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**

Run: `mvn test -pl scaffold-backend -Dtest=EmbeddedTopologyCatalogueServiceTest --batch-mode`
Expected: FAIL — `EmbeddedTopologyCatalogueService` doesn't exist

- [ ] **Step 3: Implement EmbeddedTopologyCatalogueService**

```java
package io.casehub.flow.service.embedded;

import io.casehub.api.engine.rest.EngineCaseApi;
import io.casehub.api.view.StartCaseRequest;
import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.OrderedStep;
import io.casehub.desiredstate.runtime.TransitionPlanner;
import io.casehub.flow.service.*;
import io.casehub.ops.service.topology.TopologyCompilerService;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.stream.Stream;

@ApplicationScoped
@Alternative
@Priority(100)
public class EmbeddedTopologyCatalogueService implements TopologyCatalogueService {

    private final TopologyCompilerService compilerService;
    private final EngineCaseApi engineCaseApi;
    private final String topologyBasePath;
    private final TransitionPlanner planner = new TransitionPlanner();

    @Inject
    public EmbeddedTopologyCatalogueService(
            TopologyCompilerService compilerService,
            EngineCaseApi engineCaseApi,
            @ConfigProperty(name = "casehub.ops.topology.path") String topologyBasePath) {
        this.compilerService = compilerService;
        this.engineCaseApi = engineCaseApi;
        this.topologyBasePath = topologyBasePath;
    }

    // Test constructor
    EmbeddedTopologyCatalogueService(TopologyCompilerService compilerService,
                                      EngineCaseApi engineCaseApi,
                                      String topologyBasePath) {
        this.compilerService = compilerService;
        this.engineCaseApi = engineCaseApi;
        this.topologyBasePath = topologyBasePath;
    }

    @Override
    public List<TopologyEntry> listTopologies() {
        var descriptors = compilerService.listTopologies();
        List<TopologyEntry> entries = new ArrayList<>();
        for (var desc : descriptors) {
            var metadata = readMetadata(desc.name());
            entries.add(new TopologyEntry(desc.name(),
                    metadata.getOrDefault("description", ""),
                    metadata.getOrDefault("methodologyStep", ""),
                    metadata.getOrDefault("prerequisites", "")));
        }
        return entries;
    }

    @Override
    public TopologyPlan getTopologyPlan(String name) {
        DesiredStateGraph graph = compilerService.compileTopology(name);
        var transitionPlan = planner.plan(graph,
                new io.casehub.desiredstate.api.ActualState(Map.of()));

        GraphView graphView = toGraphView(graph);
        PlanView planView = toPlanView(transitionPlan);
        String playbookYaml = readFileFromTopology(name, "playbook.yaml");
        String readme = readFileFromTopology(name, "README.md");

        return new TopologyPlan(graphView, planView, playbookYaml, readme);
    }

    @Override
    public DeployResult deploy(String name, String tenancyId) {
        if (compilerService.isRunning(tenancyId, name)) {
            throw new IllegalStateException("Topology already deployed: " + name);
        }

        String loopKey = compilerService.deploy(tenancyId, topologyBasePath, name);

        try {
            var caseView = engineCaseApi.startCase(
                    new StartCaseRequest("scaffold", "application-lifecycle", "1.0.0",
                            Map.of("topologyName", name, "applicationId", name)),
                    tenancyId);
            return new DeployResult(caseView.caseId(), loopKey);
        } catch (Exception e) {
            compilerService.stop(tenancyId, name);
            throw new IllegalStateException("Case creation failed after deploy — reconciliation stopped", e);
        }
    }

    private GraphView toGraphView(DesiredStateGraph graph) {
        List<GraphNode> nodes = graph.nodes().entrySet().stream()
                .map(e -> new GraphNode(e.getKey().value(),
                        e.getValue().type().value(),
                        graph.dependenciesOf(e.getKey()).stream()
                                .map(io.casehub.desiredstate.api.NodeId::value).toList()))
                .toList();
        List<GraphEdge> edges = graph.dependencies().stream()
                .map(d -> new GraphEdge(d.dependent().value(), d.dependency().value()))
                .toList();
        return new GraphView(nodes, edges);
    }

    private PlanView toPlanView(io.casehub.desiredstate.api.TransitionPlan plan) {
        return new PlanView(
                toLayers(plan.additions()),
                toLayers(plan.removals()),
                toLayers(plan.suspensions()),
                toLayers(plan.resumptions()),
                plan.isEmpty());
    }

    private List<List<PlanStep>> toLayers(List<List<OrderedStep>> layers) {
        return layers.stream()
                .map(layer -> layer.stream()
                        .map(step -> new PlanStep(step.node().id().value(),
                                step.action().name(), step.node().type().value()))
                        .toList())
                .toList();
    }

    private Map<String, String> readMetadata(String name) {
        String content = readFileFromTopology(name, "topology-metadata.yaml");
        if (content == null) return Map.of();
        Map<String, String> result = new java.util.LinkedHashMap<>();
        for (String line : content.split("\n")) {
            int colon = line.indexOf(':');
            if (colon > 0) {
                String key = line.substring(0, colon).trim();
                String value = line.substring(colon + 1).trim().replaceAll("^\"|\"$", "");
                result.put(key, value);
            }
        }
        return result;
    }

    private String readFileFromTopology(String topologyName, String fileName) {
        Path file = Path.of(topologyBasePath, topologyName, fileName);
        if (!Files.isRegularFile(file)) return null;
        try {
            return Files.readString(file);
        } catch (IOException e) {
            return null;
        }
    }
}
```

- [ ] **Step 4: Run test — expect pass**

Run: `mvn test -pl scaffold-backend -Dtest=EmbeddedTopologyCatalogueServiceTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/service/embedded/EmbeddedTopologyCatalogueService.java scaffold-backend/src/test/java/io/casehub/flow/service/embedded/EmbeddedTopologyCatalogueServiceTest.java
git commit -m "feat(#76): EmbeddedTopologyCatalogueService — compile, plan, deploy with case creation Refs #76"
```

### Task 5: TopologyCatalogueResource

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/rest/TopologyCatalogueResource.java`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/rest/TopologyCatalogueResourceTest.java`

**Interfaces:**
- Consumes: `TopologyCatalogueService` (Task 3), `CurrentPrincipal`
- Produces: REST endpoints at `/api/scaffold/catalogue`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.flow.rest;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TopologyCatalogueResourceTest {

    @Test
    void resourceAnnotatedWithCorrectPath() {
        var path = TopologyCatalogueResource.class.getAnnotation(Path.class);
        assertThat(path).isNotNull();
        assertThat(path.value()).isEqualTo("/api/scaffold/catalogue");
    }

    @Test
    void listEndpointExists() throws Exception {
        var method = TopologyCatalogueResource.class.getMethod("listTopologies");
        assertThat(method.getAnnotation(GET.class)).isNotNull();
    }

    @Test
    void planEndpointExists() throws Exception {
        var method = TopologyCatalogueResource.class.getMethod("getTopologyPlan", String.class);
        assertThat(method.getAnnotation(GET.class)).isNotNull();
    }

    @Test
    void deployEndpointExists() throws Exception {
        var method = TopologyCatalogueResource.class.getMethod("deploy", String.class);
        assertThat(method.getAnnotation(POST.class)).isNotNull();
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**

Run: `mvn test -pl scaffold-backend -Dtest=TopologyCatalogueResourceTest --batch-mode`
Expected: FAIL

- [ ] **Step 3: Implement TopologyCatalogueResource**

```java
package io.casehub.flow.rest;

import io.casehub.flow.service.DeployResult;
import io.casehub.flow.service.TopologyCatalogueService;
import io.casehub.flow.service.TopologyEntry;
import io.casehub.flow.service.TopologyPlan;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.smallrye.common.annotation.RunOnVirtualThread;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.List;

@Path("/api/scaffold/catalogue")
@Produces(MediaType.APPLICATION_JSON)
@ApplicationScoped
@RunOnVirtualThread
public class TopologyCatalogueResource {

    @Inject TopologyCatalogueService catalogueService;
    @Inject CurrentPrincipal currentPrincipal;

    @GET
    public List<TopologyEntry> listTopologies() {
        return catalogueService.listTopologies();
    }

    @GET
    @Path("/{name}/plan")
    public TopologyPlan getTopologyPlan(@PathParam("name") String name) {
        return catalogueService.getTopologyPlan(name);
    }

    @POST
    @Path("/{name}/deploy")
    public Response deploy(@PathParam("name") String name) {
        try {
            DeployResult result = catalogueService.deploy(name, currentPrincipal.tenancyId());
            return Response.status(201).entity(result).build();
        } catch (IllegalStateException e) {
            if (e.getMessage().contains("already deployed")) {
                return Response.status(409).entity(java.util.Map.of("error", e.getMessage())).build();
            }
            throw e;
        }
    }
}
```

- [ ] **Step 4: Run test — expect pass**

Run: `mvn test -pl scaffold-backend -Dtest=TopologyCatalogueResourceTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Update app.yaml with catalogue dataset**

Add to `datasets:` in `scaffold-backend/src/main/resources/META-INF/app.yaml`:

```yaml
    catalogue:
      endpoint: /api/scaffold/catalogue
      source: topology-catalogue-service
```

- [ ] **Step 6: Update AppResourceTest and AppLoaderTest**

Update expected dataset counts in both test files (add 1 for `catalogue`).

- [ ] **Step 7: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/rest/TopologyCatalogueResource.java scaffold-backend/src/test/java/io/casehub/flow/rest/TopologyCatalogueResourceTest.java scaffold-backend/src/main/resources/META-INF/app.yaml scaffold-backend/src/test/java/io/casehub/flow/rest/AppResourceTest.java scaffold-backend/src/test/java/io/casehub/flow/app/AppLoaderTest.java
git commit -m "feat(#76): TopologyCatalogueResource — list, plan, deploy endpoints Refs #76"
```

## Batch 3: Frontend + Full Build

### Task 6: Deploy Tab Restructure

**Files:**
- Modify: `scaffold-backend/src/main/webui/src/views/deploy.ts`
- Modify: `scaffold-backend/src/main/webui/src/index.ts` (if needed)

**Interfaces:**
- Consumes: `catalogue` dataset, `/api/scaffold/catalogue/{name}/plan` on-demand fetch

- [ ] **Step 1: Restructure deploy.ts**

Add catalogue table as the top section. Keep existing deployments + approvals as bottom section:

```typescript
import { rows, columns, dataTable, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";

export const deployView = rows(
  title("Topology Catalogue", "h2"),
  dataTable({
    lookup: lookup("catalogue", groupBy(null,
      col("name", "Topology"),
      col("description", "Description"),
      col("methodologyStep", "Step"),
      col("prerequisites", "Prerequisites"),
    )),
    sortable: true,
    selectionTopic: "selected-topology",
  }),
  title("Deployment Records", "h2"),
  columns([6, 6],
    [rows(
      title("Deployments", "h3"),
      dataTable({
        lookup: lookup("deployments", groupBy(null,
          col("deploymentId", "ID"),
          col("status", "Status"),
          col("message", "Message"),
          col("caseStatus", "Case"),
        )),
        sortable: true,
      }),
    )],
    [rows(
      title("Approvals", "h3"),
      dataTable({
        lookup: lookup("approvals", groupBy(null,
          col("id", "ID"),
          col("planId", "Plan"),
          col("status", "Status"),
          col("reviewer", "Reviewer"),
        )),
        sortable: true,
      }),
    )],
  ),
);
```

Note: The detail view (step viewer, DAG diagram, plan table) requires
interactive component wiring beyond the declarative DSL. For the MVP,
the catalogue table provides the browsing experience and clicking a
topology opens the plan view. The `selectionTopic: "selected-topology"`
enables downstream components to react to selection. The step viewer
and graph-renderer integration are wired as custom Lit elements in a
follow-up — this task establishes the catalogue data flow.

- [ ] **Step 2: Build frontend**

Run: `mvn install -pl scaffold-backend -DskipTests --batch-mode`
Expected: BUILD SUCCESS (Quinoa builds the webui)

- [ ] **Step 3: Commit**

```bash
git add scaffold-backend/src/main/webui/src/views/deploy.ts
git commit -m "feat(#76): deploy tab restructure — catalogue table + existing records Refs #76"
```

### Task 7: Full Build Verification

- [ ] **Step 1: Run full build with tests**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS with all tests passing

- [ ] **Step 2: Commit any fixes**

If the full build reveals issues, fix and commit.

## References

- [2026-10-28-topology-catalogue-deploy-design.md] — design spec this plan implements
- [ops/service/src/main/java/io/casehub/ops/service/topology/TopologyCompilerService.java] — compiler to modify
- [desiredstate/runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java] — plan generation
- [desiredstate/api/src/main/java/io/casehub/desiredstate/api/TransitionPlan.java] — plan data model
- [desiredstate/api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java] — graph interface
- [scaffold-backend/src/main/java/io/casehub/flow/rest/TopologyResource.java] — existing resource pattern
- [scaffold-backend/src/main/java/io/casehub/flow/service/embedded/EmbeddedCaseService.java] — existing embedded pattern
- [scaffold-backend/src/main/resources/casehub/ops/application-lifecycle.yaml] — case definition
- [GitHub #76] — Topology catalogue + deploy
