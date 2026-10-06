# Phase 1 Plan — Review Corrections

Mandatory corrections from light review. The executor MUST apply these
when implementing the plan. Grouped by affected task.

## Architecture: Ops Domain Logic Placement (affects Tasks 8–11)

**Correction:** Tasks 8–11 place ops domain logic (`DeploymentRegistry`,
`HealthPoller`, `DriftDetectionService`, `ProvisioningCaseManager`) in
`scaffold-backend`. This violates the 4-tier architecture. **All ops
domain logic goes in the ops repo** — either `ops/app` or a new
`ops/fleet` module. Scaffold contains only:
- Perspective configuration (`PerspectiveLoader`, `PerspectiveResource`)
- Frontend views (pages DSL)
- Thin REST resources that delegate to ops services

The existing `ops/app` module already has `ApplicationResource`,
`ApplicationEntity` (JPA), `ApplicationLifecycleService`,
`ApplicationCaseDescriptor`, and `ApplicationGoalCompiler` — the new
deployment management extends these, not replaces them.

## Task 1: Add CDI Producer (R1-09)

Add a `PerspectiveProducer` CDI bean:

```java
@ApplicationScoped
public class PerspectiveProducer {
    @Produces
    Optional<PerspectiveConfig> perspectiveConfig() {
        return PerspectiveLoader.loadFromClasspath();
    }
}
```

## Task 1: Module Validation (R1-11)

The perspective loader must expand the probe list beyond the current 6
modules (engine, work, planning, ledger, ops, iot) to include all
perspective-declared modules. Use a generic classpath probe with the
module's well-known class, or switch to a service-loader / CDI
extension pattern where each module self-registers.

## Task 3: Navigation Must Be Dynamic (R1-13)

Do NOT add hardcoded entries to `TAB_TOPICS` and `SLUG_TO_LABEL`.
Instead, derive navigation mappings from the perspective response:
```typescript
const tabTopics: Record<string, string> = {};
const slugToLabel: Record<string, string> = {};
for (const tab of perspective.tabs) {
  tabTopics[tab.id] = tab.id;
  slugToLabel[tab.id] = tab.label;
}
```

## Task 3: Warn on Unknown Views (R1-12)

Replace silent `.filter()` with a console warning:
```typescript
.map((t) => {
  const view = VIEW_REGISTRY[t.view];
  if (!view) console.warn(`Perspective tab "${t.label}" references unknown view "${t.view}"`);
  return [t.label, view];
})
.filter(([, v]) => v != null);
```

## Task 4: DatabaseContainerSpec.image() (R1-14)

Make `image` a record component with compact constructor default:
```java
public record DatabaseContainerSpec(
    String name, String image, String databaseName,
    String username, String password, String dataVolume
) implements ContainerNodeSpec {
    public DatabaseContainerSpec {
        image = image != null ? image : "postgres:16";
    }
    @Override public NodeType nodeType() { return ContainerNodeTypes.DATABASE; }
}
```

## Task 5: GoalCompiler Test API Fix (R1-04)

`CompilationResult` is sealed with `SingleGraph` and `Lifecycle`
variants. Fix test:
```java
var result = compiler.compile(descriptor, factory);
assertThat(result).isInstanceOf(CompilationResult.SingleGraph.class);
var graph = ((CompilationResult.SingleGraph) result).graph();
// graph.nodes() returns Map<NodeId, DesiredNode>
var appNode = graph.nodes().values().stream()
    .filter(n -> n.type().equals(ContainerNodeTypes.APP_CONTAINER))
    .findFirst().orElseThrow();
```

## Task 6: Unix Socket Client (R1-06)

`java.net.http.HttpClient` does NOT support Unix domain sockets.
Use **Vert.x WebClient** with `SocketAddress.domainSocketAddress()`:

```java
var socketAddress = SocketAddress.domainSocketAddress(
    System.getenv("XDG_RUNTIME_DIR") + "/podman/podman.sock");
var client = WebClient.create(vertx, new WebClientOptions());
client.get(socketAddress, 80, "localhost", "/v5.0.0/libpod/containers/json")
    .send()
    .onSuccess(response -> { /* parse JSON */ });
```

This is native Quarkus/Vert.x — no additional dependencies needed.

## Task 7: NodeProvisioner Contract (R1-07)

1. `provision()` returns `ProvisionResult` (sealed: `Success`, `Failed`,
   `PendingApproval`). Not void.
2. Dispatch by `NodeType` via `handledTypes()`, not by sealed-type on
   NodeSpec. Declare:
   ```java
   Set<NodeType> handledTypes() {
       return Set.of(APP_CONTAINER, DATABASE, NETWORK, VOLUME);
   }
   ```
3. Must implement `deprovision(DesiredNode, DeprovisionContext)`
   returning `DeprovisionResult`. Container cleanup: stop + remove.
4. Follow `InfraNodeProvisioner` / `DeploymentNodeProvisioner` patterns.

## Task 7: Use ThresholdFaultPolicy (R1-05)

Do NOT create `ContainerFaultPolicy`. Use the existing
`ThresholdFaultPolicy`:
```java
ThresholdFaultPolicy.builder()
    .faultTypes(Set.of(FaultType.PROVISION_FAILED))
    .nodeTypes(Set.of(ContainerNodeTypes.APP_CONTAINER,
                      ContainerNodeTypes.DATABASE))
    .tier(3, FaultPolicy.addReviewNode(reviewSpecFactory))
    .build();
```

## Tasks 9 + 11: Use ReconciliationLoop (R1-02)

Do NOT create `HealthPoller` or `DriftDetectionService` with
`@Scheduled`. Use the existing `ReconciliationLoop`:

1. Start one `ReconciliationLoop` instance per deployment:
   ```java
   reconciliationLoop.start(deploymentId, desiredGraph);
   ```
2. Register `PodmanNodeProvisioner` via `NodeProvisionerRouter`
3. Register `PodmanActualStateAdapter` via `ActualStateAdapterRouter`
4. Drift detection, health, fault feedback, and CBR all come for free
5. Manual re-convergence: `reconciliationLoop.requestReconciliation(deploymentId)`
6. Health tracking: implement `ReconciliationListener.onReconciliationCycleCompleted()`
   to update deployment health status
7. Drift events: listen for `NODE_DRIFTED`, `NODE_RECOVERED` CloudEvents

## Task 8: Use JPA Persistence (R1-15)

Do NOT use in-memory `DeploymentRegistry`. Extend the existing
`ApplicationEntity` pattern in `ops/app` with JPA persistence.
If the ReconciliationLoop + ops/app patterns are followed, state
persistence comes from `ReconciliationStateStore` and the existing
`ApplicationEntity` / `EntityManager` infrastructure.

## Task 1: Fix Typo (R1-16)

`usesPerpsectiveDeclaredModulesWhenPresent` →
`usesPerspectiveDeclaredModulesWhenPresent`
