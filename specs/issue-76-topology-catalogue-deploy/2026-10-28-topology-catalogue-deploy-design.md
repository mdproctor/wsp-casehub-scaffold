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
available topologies, see a deploy plan (terraform-style), and deploy
with a single action that creates a case owning the full lifecycle.

## Scope — Level 01 MVP

One topology: ops Level 01 (hello-llm). Four nodes: manifest, eidos,
engine, my-llm. The feature works end-to-end for this single topology.
Multi-topology catalogue and tutorial progression are follow-up work.

## Architecture

```
Deploy Tab (restructured)
├── Catalogue section (top)
│   └── dataTable: available topologies from filesystem
├── Detail view (on selection)
│   ├── Left panel: README.md rendered as markdown
│   ├── Right panel
│   │   ├── DAG diagram (graph-renderer) — nodes + dependencies
│   │   └── Plan table — name, type, action, dependsOn, order
│   └── [Deploy] button
└── Records section (bottom, existing)
    ├── Deployments table
    └── Approvals table
```

### Backend

Three new REST endpoints on `TopologyCatalogueResource` at
`/api/scaffold/topologies`:

**1. `GET /api/scaffold/topologies`** — List available topologies

Delegates to `TopologyCompilerService.listTopologies()` which scans
the directory at `casehub.ops.topology.path`. Returns:

```json
[
  {
    "name": "01-hello-llm",
    "path": "/path/to/topologies/deployment/01-hello-llm",
    "source": "filesystem"
  }
]
```

Enriches each entry with metadata parsed from the README.md header
comments (Level number, methodology step, prerequisites, description).

**2. `GET /api/scaffold/topologies/{name}/plan`** — Compile and plan

Compiles the topology YAML via `TopologyCompilerService.compileDomain()`.
Runs `TransitionPlanner.plan(graph, new ActualState(Map.of()))` to
produce the deploy plan against an empty actual state (fresh deployment).

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
      {"order": 1, "nodeId": "manifest", "action": "ADD", "type": "casehub_application"},
      {"order": 2, "nodeId": "eidos", "action": "ADD", "type": "casehub_application"},
      {"order": 3, "nodeId": "engine", "action": "ADD", "type": "casehub_application"},
      {"order": 4, "nodeId": "my-llm", "action": "ADD", "type": "casehub_application"}
    ],
    "removals": [],
    "empty": false
  },
  "readme": "# Level 01: Hello LLM\n\nDeploy a complete LLM app..."
}
```

The `graph` field feeds the DAG diagram. The `plan` field feeds the
plan table. The `readme` field feeds the markdown walkthrough panel.

**3. `POST /api/scaffold/topologies/{name}/deploy`** — Deploy and create case

Two actions in sequence:

1. `TopologyCompilerService.deploy(tenancyId, basePath, name)` —
   compiles the YAML and starts the reconciliation loop
2. `EngineCaseApi.startCase(request, tenancyId)` — creates an
   `application-lifecycle` case with context:
   ```json
   {
     "namespace": "scaffold",
     "name": "application-lifecycle",
     "version": "1.0.0",
     "context": {
       "topologyName": "01-hello-llm",
       "applicationId": "<generated-uuid>"
     }
   }
   ```

Returns `{ "caseId": "<uuid>", "loopKey": "topology:<tenancyId>:01-hello-llm" }`.

The case owns the lifecycle from this point — drift detection,
scaling events, incidents, and approvals all flow through sub-case
bindings defined in `application-lifecycle.yaml`.

### Frontend

**Deploy tab restructured (`deploy.ts`):**

Top section — topology catalogue:
```typescript
dataTable({
  lookup: lookup("topologies", groupBy(null,
    col("name", "Topology"),
    col("description", "Description"),
    col("methodologyStep", "Step"),
  )),
  sortable: true,
  selectionTopic: "selected-topology",
})
```

On selection — detail panel (new component or inline expansion):
- Left: rendered markdown from the `readme` field
- Right top: `graph-renderer` DAG showing nodes and dependency edges
- Right bottom: plan table showing ordered deployment steps
- Deploy button: POST to deploy endpoint, then navigate to Cases tab

Bottom section — existing deployment records and approvals tables
(unchanged).

**New datasets in app.yaml:**
```yaml
topologies:
  endpoint: /api/scaffold/topologies
  source: topology-service
```

The plan and readme are fetched on-demand when a topology is selected
(not preloaded in the dataset).

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
- `TopologyPlan(GraphView graph, PlanView plan, String readme)`
- `GraphView(List<GraphNode> nodes, List<GraphEdge> dependencies)`
- `GraphNode(String id, String type, List<String> dependsOn)`
- `GraphEdge(String from, String to)`
- `PlanView(List<PlanStep> additions, List<PlanStep> removals, boolean empty)`
- `PlanStep(int order, String nodeId, String action, String type)`
- `DeployResult(UUID caseId, String loopKey)`

`EmbeddedTopologyCatalogueService` injects `TopologyCompilerService`,
`TransitionPlanner`, and `EngineCaseApi`. Client impl deferred — the
catalogue is inherently local to the scaffold instance.

### Testing

- `TopologyCatalogueResourceTest` — annotation-based JAX-RS verification
  (follows `CaseResourceTest` pattern)
- `EmbeddedTopologyCatalogueServiceTest` — unit test with mocked
  `TopologyCompilerService` and `EngineCaseApi`, verifies plan
  generation and deploy-with-case-creation flow
- Integration test: compile Level 01 YAML → verify graph has 4 nodes →
  verify plan has 4 additions in dependency order

## References

- `ops/service/src/main/java/io/casehub/ops/service/topology/TopologyCompilerService.java` — existing compiler
- `desiredstate/runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java` — plan generation
- `desiredstate/api/src/main/java/io/casehub/desiredstate/api/TransitionPlan.java` — plan data model
- `desiredstate/api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java` — graph interface
- `scaffold-backend/src/main/resources/casehub/ops/application-lifecycle.yaml` — case definition
- `mosaic/packages/graph-renderer/` — DAG visualisation component
- `ops/topology-tests/src/test/resources/topologies/deployment/01-hello-llm/` — Level 01 YAML
- casehubio/scaffold#76
