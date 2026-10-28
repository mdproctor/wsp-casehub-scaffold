# Topology Catalogue + Deploy — Design Spec

**Issue:** casehubio/scaffold#76
**Date:** 2026-10-28

## Problem

Scaffold has operational views for monitoring deployed infrastructure
(Fleet, Drift, Cases) but no way to browse available topologies,
understand what they do, or initiate a deployment. The Deploy tab shows
deployment records and approvals — it skips the step where you select
something to deploy.

## Goal

Add a topology catalogue to the Deploy tab that lets users browse
available topologies, see a playbook walkthrough via the mosaic step
viewer, see a terraform-style deploy plan (preserving parallel execution
layers), and deploy with a single action that creates a case owning the
full lifecycle.

## Scope — Level 01 MVP

One topology: ops Level 01 (hello-llm). Four nodes: manifest, eidos,
engine, my-llm. The feature works end-to-end for this single topology.

Deferred items captured as GitHub issues:
- casehubio/scaffold#TBD — Multi-topology catalogue and tutorial progression
- casehubio/scaffold#TBD — Client impl for TopologyCatalogueService

## Architecture

```
Deploy Tab (restructured)
├── Catalogue section (top)
│   └── dataTable: available topologies from filesystem
├── Detail view (on selection)
│   ├── Left panel: pages-step-viewer (playbook walkthrough from playbook.yaml)
│   ├── Right panel
│   │   ├── DAG diagram (graph-renderer) — nodes + dependencies
│   │   └── Plan table — layered steps showing parallel tiers
│   └── [Deploy] button
└── Records section (bottom, existing)
    ├── Deployments table
    └── Approvals table
```

### Backend

Three new REST endpoints on `TopologyCatalogueResource` at
`/api/scaffold/catalogue`:

**1. `GET /api/scaffold/catalogue`** — List available topologies

Delegates to `TopologyCompilerService.listTopologies()` which scans
the directory at `casehub.ops.topology.path`. Enriches each entry with
metadata parsed from `topology-metadata.yaml` in each topology
directory.

The metadata file is a structured YAML file:
```yaml
description: "Deploy a complete LLM app with four services"
methodologyStep: "DEPLOY — First App"
prerequisites: "None"
level: 1
```

Returns:
```json
[
  {
    "name": "01-hello-llm",
    "description": "Deploy a complete LLM app with four services",
    "methodologyStep": "DEPLOY — First App",
    "prerequisites": "None"
  }
]
```

**2. `GET /api/scaffold/catalogue/{name}/plan`** — Compile and plan

Three operations:

1. `TopologyCompilerService.compileTopology(name)` — compiles the
   topology into a `DesiredStateGraph`. This is a new method extracted
   from the existing `deploy()` logic — it iterates all YAML files in
   the topology directory, compiles each via `compileDomain()`, and
   composes the results into a single `DesiredStateGraph` via
   `GRAPH_FACTORY.of(allNodes, allDeps)`.

2. `TransitionPlanner.plan(graph, new ActualState(Map.of()))` —
   produces the deploy plan against an empty actual state (fresh
   deployment).

3. Read supplementary files from the topology directory (resolved via
   `casehub.ops.topology.path` + `/{name}/`):
   - `README.md` — supplementary documentation (raw markdown string)
   - `playbook.yaml` — structured deployment walkthrough steps for
     `<pages-step-viewer>` (raw YAML string)

   The service layer reads these files directly from the topology
   directory. Neither `compileTopology()` nor `TransitionPlanner`
   handles these — file reading is the catalogue service's
   responsibility.

Returns:
```json
{
  "graph": {
    "nodes": [
      {"id": "manifest", "type": "casehub_application", "dependsOn": []},
      {"id": "eidos", "type": "casehub_application", "dependsOn": ["manifest"]},
      {"id": "engine", "type": "casehub_application", "dependsOn": ["eidos"]},
      {"id": "my-llm", "type": "casehub_application", "dependsOn": ["engine"]}
    ],
    "dependencies": [
      {"from": "eidos", "to": "manifest"},
      {"from": "engine", "to": "eidos"},
      {"from": "my-llm", "to": "engine"}
    ]
  },
  "plan": {
    "additions": [
      [{"nodeId": "manifest", "action": "PROVISION", "type": "casehub_application"}],
      [{"nodeId": "eidos", "action": "PROVISION", "type": "casehub_application"}],
      [{"nodeId": "engine", "action": "PROVISION", "type": "casehub_application"}],
      [{"nodeId": "my-llm", "action": "PROVISION", "type": "casehub_application"}]
    ],
    "removals": [],
    "suspensions": [],
    "resumptions": [],
    "empty": false
  },
  "playbookYaml": "- block:\n    label: \"Deploy manifest\"\n    steps:\n      ...",
  "readme": "# Level 01: Hello LLM\n\nDeploy a complete LLM app..."
}
```

Each element of the outer list in `additions` is a **layer** — steps
within the same layer can execute in parallel, and layers execute in
sequence (layer 0 completes before layer 1 starts). For Level 01's
linear dependency chain, each layer has exactly one step.

The `graph` field feeds the DAG diagram. The `plan` field feeds the
plan table. The `playbookYaml` field is parsed by the frontend via
`Walker.resolve()` from `@casehubio/yaml-core/step` to produce
`ResolvedStep[]` for `<pages-step-viewer>`. The `readme` field is
rendered separately via a standard markdown renderer for supplementary
documentation.

**3. `POST /api/scaffold/catalogue/{name}/deploy`** — Deploy and create case

Pre-check: calls `TopologyCompilerService.isRunning(tenancyId, name)`
to verify this topology is not already deployed. Returns 409 Conflict
if already running.

Two actions in sequence with failure compensation:

1. `TopologyCompilerService.deploy(tenancyId, name)` —
   compiles the YAML and starts the reconciliation loop, returns
   the loopKey
2. `EngineCaseApi.startCase(request, tenancyId)` — creates an
   `application-lifecycle` case with context:
   ```json
   {
     "namespace": "scaffold",
     "name": "application-lifecycle",
     "version": "1.0.0",
     "context": {
       "topologyName": "01-hello-llm",
       "applicationId": "01-hello-llm"
     }
   }
   ```

   The `applicationId` is set to the topology name (not a random UUID)
   to provide a stable, deterministic identifier for CBR feature
   matching across deployments of the same topology. The
   `application-lifecycle.yaml` case definition references
   `.applicationId` as a CBR feature.

Failure compensation: if `startCase()` fails after `deploy()` succeeds,
the endpoint calls `TopologyCompilerService.stop(tenancyId, name)` to
halt the orphaned reconciliation loop before propagating the error.

Returns `{ "caseId": "<uuid>", "loopKey": "topology:<tenancyId>:01-hello-llm" }`.

The case owns the lifecycle from this point — drift detection,
scaling events, incidents, and approvals all flow through sub-case
bindings defined in `application-lifecycle.yaml`.

### Cross-repo changes (casehub-ops)

The plan endpoint requires compilation logic currently embedded in
`TopologyCompilerService.deploy()`. Changes to the ops repo:

**API changes to `TopologyCompilerService`:**

1. **Extract `compileTopology(String name)`** — moves the multi-file
   YAML iteration, per-file `compileDomain()` calls, and graph
   composition into a reusable method. Uses the injected
   `casehub.ops.topology.path` config for directory resolution.
   Returns `DesiredStateGraph`:
   ```java
   public DesiredStateGraph compileTopology(String name)
   ```

2. **Change `deploy()` return type from `void` to `String`** — returns
   the loopKey (`"topology:" + tenancyId + ":" + topologyName`) so
   callers don't need to reconstruct the naming convention:
   ```java
   public String deploy(String tenancyId, String topologyName)
   ```

**New files in Level 01 topology directory**
(`ops/topology-tests/src/test/resources/topologies/deployment/01-hello-llm/`):

3. **`topology-metadata.yaml`** — structured metadata for the
   catalogue listing:
   ```yaml
   description: "Deploy a complete LLM app with four services"
   methodologyStep: "DEPLOY — First App"
   prerequisites: "None"
   level: 1
   ```

4. **`playbook.yaml`** — structured deployment walkthrough steps in
   pages step-viewer format. Uses `block`, `parallel`, and `plugin`
   step kinds to describe the deployment procedure:
   ```yaml
   - block:
       label: "Provision manifest service"
       steps:
         - provision-node:
             value: manifest
             type: casehub_application
   - block:
       label: "Provision eidos (depends on manifest)"
       steps:
         - provision-node:
             value: eidos
             type: casehub_application
   ```
   Parsed by the frontend via `Walker.resolve()` from
   `@casehubio/yaml-core/step` to produce `ResolvedStep[]` for the
   `<pages-step-viewer>` component.

### Frontend

**Deploy tab restructured (`deploy.ts`):**

Top section — topology catalogue:
```typescript
dataTable({
  lookup: lookup("catalogue", groupBy(null,
    col("name", "Topology"),
    col("description", "Description"),
    col("methodologyStep", "Step"),
  )),
  sortable: true,
  selectionTopic: "selected-topology",
})
```

On selection — detail panel (new component or inline expansion):
- Left: `<pages-step-viewer>` rendering the `playbookYaml` field.
  The frontend parses the raw YAML string via
  `parse(playbookYaml)` → `Walker.resolve(steps, catalog)` →
  `ResolvedStep[]`, then passes the resolved steps to the
  component's `.steps` property. Below the step viewer, the
  `readme` field is rendered via a standard markdown renderer for
  supplementary documentation.
- Right top: `graph-renderer` DAG showing nodes and dependency edges
- Right bottom: plan table showing layered deployment steps with
  tier indicators for parallel execution
- Deploy button: POST to deploy endpoint, then navigate to Cases tab

Bottom section — existing deployment records and approvals tables
(unchanged).

**New datasets in app.yaml:**
```yaml
catalogue:
  endpoint: /api/scaffold/catalogue
  source: topology-catalogue-service
```

The plan, playbook, and readme are fetched on-demand when a topology
is selected (not preloaded in the dataset).

### Service Layer

New `TopologyCatalogueService` interface following the established
dual-impl pattern:

```java
public interface TopologyCatalogueService {
    List<TopologyEntry> listTopologies();
    TopologyPlan getTopologyPlan(String name);
    DeployResult deploy(String name, String tenancyId);
}
```

Records:
- `TopologyEntry(String name, String description, String methodologyStep, String prerequisites)`
- `TopologyPlan(GraphView graph, PlanView plan, String playbookYaml, String readme)`
- `GraphView(List<GraphNode> nodes, List<GraphEdge> dependencies)`
- `GraphNode(String id, String type, List<String> dependsOn)`
- `GraphEdge(String from, String to)`
- `PlanView(List<List<PlanStep>> additions, List<List<PlanStep>> removals, List<List<PlanStep>> suspensions, List<List<PlanStep>> resumptions, boolean empty)`
- `PlanStep(String nodeId, String action, String type)`
- `DeployResult(UUID caseId, String loopKey)`

`EmbeddedTopologyCatalogueService` injects `TopologyCompilerService`,
`TransitionPlanner`, and `EngineCaseApi`. The embedded impl injects
`EngineCaseApi` directly for case creation — this follows the same
pattern as `EmbeddedCaseService` which also injects `EngineCaseApi`
directly. The deploy operation is cohesive: creating the reconciliation
loop and its lifecycle case is a single atomic action from the
catalogue's perspective.

Client impl deferred — tracked as casehubio/scaffold#TBD.

### Testing

- `TopologyCatalogueResourceTest` — annotation-based JAX-RS verification
  (follows `CaseResourceTest` pattern)
- `EmbeddedTopologyCatalogueServiceTest` — unit test with mocked
  `TopologyCompilerService` and `EngineCaseApi`, verifies plan
  generation with layered structure and deploy-with-case-creation flow
  including failure compensation
- Integration test: compile Level 01 YAML → verify graph has 4 nodes →
  verify plan has 4 addition layers with 1 step each in dependency order
  → verify all four dimensions present (additions, removals, suspensions,
  resumptions)

## References

- `ops/service/src/main/java/io/casehub/ops/service/topology/TopologyCompilerService.java` — existing compiler (cross-repo changes needed)
- `desiredstate/runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java` — plan generation
- `desiredstate/api/src/main/java/io/casehub/desiredstate/api/TransitionPlan.java` — plan data model (List<List<OrderedStep>> per dimension)
- `desiredstate/api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java` — graph interface
- `scaffold-backend/src/main/resources/casehub/ops/application-lifecycle.yaml` — case definition
- `scaffold-backend/src/main/webui/.casehub-packages/packages/pages-step-viewer/` — playbook walkthrough component (accepts `ResolvedStep[]`)
- `mosaic/packages/graph-renderer/` — DAG visualisation component
- `ops/topology-tests/src/test/resources/topologies/deployment/01-hello-llm/` — Level 01 YAML
- casehubio/scaffold#76
