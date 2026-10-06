# Ops Perspective Wiring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #53 — wire desiredstate + ops modules and connect perspective tabs to real endpoints
**Issue group:** #52, #53

**Goal:** Connect scaffold's perspective tabs to real ops @McpDomain APIs via GraphQL, completing the ops perspective's data layer.

**Architecture:** Add `casehub-ops-service` as a Maven dependency in scaffold-backend. The @McpDomain APT processor generates REST + GraphQL + MCP endpoints at compile time. The frontend perspective views switch from placeholder dataset lookups to GraphQL queries via the `/graphql` endpoint. No new Java code in scaffold — the ops service layer provides everything.

**Tech Stack:** Maven (dependency wiring), Quarkus SmallRye GraphQL, TypeScript (pages-ui DSL), @McpDomain APT generator

## Global Constraints

- All ops domain logic stays in the ops repo (R1-architecture correction)
- Scaffold contains only: perspective configuration, frontend views, thin delegates
- Use GraphQL where available — ops now has `-AgenerateGraphQL=true`
- `casehub-ops-service` artifact: `io.casehub:casehub-ops-service:0.2-SNAPSHOT`
- Pages-ui data layer has `restSource()` but no `graphqlSource()` — use `fetchSource()` for GraphQL queries
- @McpDomain generates REST at `/api/ops/*` and GraphQL at `/graphql`

---

## Batch 1: Maven dependency + compile verification

### Task 1: Add casehub-ops-service dependency and verify compilation

**Files:**
- Modify: `scaffold-backend/pom.xml`

**Interfaces:**
- Consumes: `casehub-ops-service` Maven artifact from ops repo
- Produces: ops @McpDomain APIs on classpath, APT-generated REST + GraphQL + MCP endpoints

- [ ] **Step 1: Write integration test verifying ops GraphQL schema is available**

```java
package io.casehub.flow.rest;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class OpsGraphQLSchemaIT {

    @Test
    void opsApplicationsQueryIsAvailable() {
        given()
            .contentType(ContentType.JSON)
            .body("{\"query\": \"{ __schema { queryType { fields { name } } } }\"}")
            .when()
            .post("/graphql")
            .then()
            .statusCode(200)
            .body("data.__schema.queryType.fields.name", hasItem(containsString("ops")));
    }
}
```

Test file: `scaffold-backend/src/test/java/io/casehub/flow/rest/OpsGraphQLSchemaIT.java`

- [ ] **Step 2: Run test to verify it fails (dependency missing)**

Run: `mvn --batch-mode test -pl scaffold-backend -am -Dtest=OpsGraphQLSchemaIT`
Expected: FAIL — compilation error, `casehub-ops-service` not on classpath

- [ ] **Step 3: Add casehub-ops-service dependency to scaffold-backend/pom.xml**

Add after the `casehub-platform` dependency (line 98):

```xml
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ops-service</artifactId>
            <version>${project.version}</version>
        </dependency>
```

Use `ide_replace_text_in_file` to insert after the casehub-platform dependency block.

- [ ] **Step 4: Run full build to verify compilation**

Run: `mvn --batch-mode install -pl scaffold-backend -am -DskipTests`
Expected: BUILD SUCCESS — @McpDomain APT generates REST + GraphQL endpoints for all 8 ops domains

- [ ] **Step 5: Run the integration test**

Run: `mvn --batch-mode test -pl scaffold-backend -am -Dtest=OpsGraphQLSchemaIT`
Expected: PASS — GraphQL schema includes ops query fields

- [ ] **Step 6: Verify REST endpoints are also available**

Add a second test method to `OpsGraphQLSchemaIT`:

```java
    @Test
    void opsRestEndpointsAreAvailable() {
        given()
            .when()
            .get("/api/ops/applications")
            .then()
            .statusCode(anyOf(is(200), is(204)));
    }
```

Run: `mvn --batch-mode test -pl scaffold-backend -am -Dtest=OpsGraphQLSchemaIT`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add scaffold-backend/pom.xml scaffold-backend/src/test/java/io/casehub/flow/rest/OpsGraphQLSchemaIT.java
git commit -m "feat(#53): add casehub-ops-service dependency with schema verification

Refs #53"
```

---

## Batch 2: Update perspective.yaml and frontend datasets

### Task 2: Update perspective.yaml dataset endpoints to match ops API paths

**Files:**
- Modify: `scaffold-backend/src/main/resources/META-INF/perspective.yaml`
- Test: `scaffold-backend/src/test/java/io/casehub/flow/perspective/PerspectiveLoaderTest.java`

**Interfaces:**
- Consumes: existing PerspectiveLoader parser
- Produces: updated dataset endpoints matching actual ops @McpDomain REST paths

- [ ] **Step 1: Update PerspectiveLoaderTest to verify correct endpoint paths**

Add a test that validates the dataset endpoints match ops API base paths:

```java
    @Test
    void datasetEndpointsMatchOpsApiPaths() {
        var config = PerspectiveLoader.loadFromClasspath().orElseThrow();
        var datasets = config.datasets();
        assertThat(datasets).anyMatch(d ->
            d.name().equals("applications") && d.endpoint().equals("/api/ops/applications"));
        assertThat(datasets).anyMatch(d ->
            d.name().equals("deployments") && d.endpoint().equals("/api/ops/deployments"));
        assertThat(datasets).anyMatch(d ->
            d.name().equals("clusters") && d.endpoint().equals("/api/ops/clusters"));
        assertThat(datasets).anyMatch(d ->
            d.name().equals("reconciliation") && d.endpoint().equals("/api/ops/reconciliation"));
        assertThat(datasets).anyMatch(d ->
            d.name().equals("cases") && d.endpoint().equals("/api/ops/cases"));
        assertThat(datasets).anyMatch(d ->
            d.name().equals("approvals") && d.endpoint().equals("/api/ops/approvals"));
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl scaffold-backend -am -Dtest=PerspectiveLoaderTest#datasetEndpointsMatchOpsApiPaths`
Expected: FAIL — current endpoints are `/api/ops/deployments` (for "deployments") but dataset names and endpoints don't match the full set

- [ ] **Step 3: Update perspective.yaml datasets**

Replace the datasets section in `scaffold-backend/src/main/resources/META-INF/perspective.yaml`:

```yaml
  datasets:
    applications:
      endpoint: /api/ops/applications
      source: ops-service
    deployments:
      endpoint: /api/ops/deployments
      source: ops-service
    clusters:
      endpoint: /api/ops/clusters
      source: ops-service
    reconciliation:
      endpoint: /api/ops/reconciliation
      source: ops-service
    cases:
      endpoint: /api/ops/cases
      source: ops-service
    approvals:
      endpoint: /api/ops/approvals
      source: ops-service
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl scaffold-backend -am -Dtest=PerspectiveLoaderTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scaffold-backend/src/main/resources/META-INF/perspective.yaml scaffold-backend/src/test/java/io/casehub/flow/perspective/PerspectiveLoaderTest.java
git commit -m "feat(#53): update perspective.yaml datasets to match ops @McpDomain API paths

Refs #53"
```

### Task 3: Wire frontend views to GraphQL queries

**Files:**
- Modify: `scaffold-backend/src/main/webui/src/datasets.ts`
- Modify: `scaffold-backend/src/main/webui/src/views/fleet.ts`
- Modify: `scaffold-backend/src/main/webui/src/views/deploy.ts`
- Modify: `scaffold-backend/src/main/webui/src/views/health.ts`
- Modify: `scaffold-backend/src/main/webui/src/views/drift.ts`
- Modify: `scaffold-backend/src/main/webui/src/views/situations.ts`
- Modify: `scaffold-backend/src/main/webui/src/views/updates.ts`

**Interfaces:**
- Consumes: `fetchSource` from `@casehubio/pages-data`, perspective dataset definitions
- Produces: live data-bound views calling ops GraphQL endpoint

- [ ] **Step 1: Add a graphqlSource helper to datasets.ts**

Add a utility function that creates a fetchSource targeting `/graphql`:

```typescript
import { bind, restSource, fetchSource } from "@casehubio/pages-ui";
import type { DataSetId } from "@casehubio/pages-data";

function graphql(id: string, query: string, dataPath: string, opts?: {
  refreshTime?: string;
}) {
  const binding = bind(id, fetchSource(`/graphql`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query }),
    dataPath,
  } as any));
  return opts?.refreshTime ? { ...binding, refreshTime: opts.refreshTime } : binding;
}
```

- [ ] **Step 2: Update createPerspectiveDatasets to use GraphQL queries**

Replace the REST-based `createPerspectiveDatasets` with GraphQL-backed datasets:

```typescript
export function createPerspectiveDatasets(_defs: PerspectiveDatasetDef[]) {
  return [
    graphql("applications",
      "{ opsApplications { id name status version lastDeployed } }",
      "data.opsApplications", { refreshTime: "10second" }),
    graphql("deployments",
      "{ opsDeployments { applicationId version status timestamp } }",
      "data.opsDeployments", { refreshTime: "10second" }),
    graphql("clusters",
      "{ opsClusters { id name status nodeCount } }",
      "data.opsClusters", { refreshTime: "10second" }),
    graphql("reconciliation",
      "{ opsReconciliationStatus { clusters { clusterId clusterName active nodeCount } } }",
      "data.opsReconciliationStatus", { refreshTime: "10second" }),
    graphql("cases",
      "{ opsCases { caseId eventCount } }",
      "data.opsCases", { refreshTime: "10second" }),
    graphql("approvals",
      "{ opsApprovals { id planId status reviewer } }",
      "data.opsApprovals", { refreshTime: "5second" }),
  ];
}
```

Note: The exact GraphQL field names depend on the @McpDomain generator output. After the build compiles (Task 1), inspect the generated schema at `/graphql/schema.graphql` or via introspection to get exact field names and adjust the queries.

- [ ] **Step 3: Update fleet.ts view columns to match OpsApplicationApi fields**

```typescript
import { rows, dataTable, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";

export const fleetView = rows(
  title("Fleet", "h2"),
  dataTable({
    lookup: lookup("applications", groupBy(null,
      col("id"), col("name"), col("status"),
      col("version"), col("lastDeployed"),
    )),
    sortable: true,
    selectionTopic: "selected-deployment",
  }),
);
```

- [ ] **Step 4: Update deploy.ts with deployment + approval views**

```typescript
import { rows, columns, dataTable, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";

export const deployView = rows(
  columns([6, 6],
    [rows(
      title("Deployments", "h3"),
      dataTable({
        lookup: lookup("deployments", groupBy(null,
          col("applicationId"), col("version"),
          col("status"), col("timestamp"),
        )),
        sortable: true,
      }),
    )],
    [rows(
      title("Approvals", "h3"),
      dataTable({
        lookup: lookup("approvals", groupBy(null,
          col("id"), col("planId"),
          col("status"), col("reviewer"),
        )),
        sortable: true,
      }),
    )],
  ),
);
```

- [ ] **Step 5: Update health.ts with cluster health columns**

```typescript
import { rows, dataTable, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";

export const healthView = rows(
  title("Health", "h2"),
  dataTable({
    lookup: lookup("clusters", groupBy(null,
      col("id"), col("name"), col("status"),
      col("nodeCount"),
    )),
    sortable: true,
  }),
);
```

- [ ] **Step 6: Update drift.ts with reconciliation data**

```typescript
import { rows, dataTable, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";

export const driftView = rows(
  title("Drift", "h2"),
  dataTable({
    lookup: lookup("reconciliation", groupBy(null,
      col("clusterId"), col("clusterName"),
      col("active"), col("nodeCount"),
    )),
    sortable: true,
  }),
);
```

- [ ] **Step 7: Update situations.ts with case data**

```typescript
import { rows, dataTable, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";

export const situationsView = rows(
  title("Situations", "h2"),
  dataTable({
    lookup: lookup("cases", groupBy(null,
      col("caseId"), col("eventCount"),
    )),
    sortable: true,
  }),
);
```

- [ ] **Step 8: Update updates.ts with deployment version data**

```typescript
import { rows, dataTable, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";

export const updatesView = rows(
  title("Updates", "h2"),
  dataTable({
    lookup: lookup("deployments", groupBy(null,
      col("applicationId"), col("version"),
      col("status"), col("timestamp"),
    )),
    sortable: true,
  }),
);
```

- [ ] **Step 9: Build and verify frontend compiles**

Run: `mvn --batch-mode install -pl scaffold-backend -am -DskipTests`
Expected: BUILD SUCCESS — webui compiles with updated views

- [ ] **Step 10: Commit**

```bash
git add scaffold-backend/src/main/webui/src/datasets.ts scaffold-backend/src/main/webui/src/views/
git commit -m "feat(#53): wire perspective views to ops GraphQL API

Connect Fleet, Deploy, Health, Drift, Situations, Updates tabs to
real ops @McpDomain data via GraphQL queries at /graphql.

Refs #53"
```

---

## Batch 3: End-to-end verification

### Task 4: End-to-end integration test — perspective tabs serve real ops data

**Files:**
- Create: `scaffold-backend/src/test/java/io/casehub/flow/rest/OpsPerspectiveIT.java`

**Interfaces:**
- Consumes: perspective REST endpoint, ops GraphQL endpoint, ops REST endpoints
- Produces: verified end-to-end data flow from ops service through to perspective tab discovery

- [ ] **Step 1: Write integration test**

```java
package io.casehub.flow.rest;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class OpsPerspectiveIT {

    @Test
    void perspectiveEndpointReturnsTabs() {
        given()
            .when()
            .get("/api/perspective")
            .then()
            .statusCode(200)
            .body("tabs.size()", greaterThanOrEqualTo(6))
            .body("tabs.id", hasItems("fleet", "deploy", "health",
                "drift", "situations", "updates"));
    }

    @Test
    void perspectiveEndpointReturnsDatasets() {
        given()
            .when()
            .get("/api/perspective")
            .then()
            .statusCode(200)
            .body("datasets.size()", greaterThanOrEqualTo(6))
            .body("datasets.name", hasItems("applications", "deployments",
                "clusters", "reconciliation", "cases", "approvals"));
    }

    @Test
    void opsGraphQLReturnsApplications() {
        given()
            .contentType(ContentType.JSON)
            .body("{\"query\": \"{ opsApplications { id name } }\"}")
            .when()
            .post("/graphql")
            .then()
            .statusCode(200)
            .body("errors", nullValue());
    }
}
```

- [ ] **Step 2: Run test**

Run: `mvn --batch-mode test -pl scaffold-backend -am -Dtest=OpsPerspectiveIT`
Expected: PASS — perspective returns tabs/datasets, GraphQL responds without errors

- [ ] **Step 3: Commit**

```bash
git add scaffold-backend/src/test/java/io/casehub/flow/rest/OpsPerspectiveIT.java
git commit -m "test(#53): end-to-end ops perspective integration tests

Verify perspective tabs, datasets, and GraphQL endpoint serve real
ops data through the full stack.

Closes #53"
```

---

## References

- [perspective.yaml](scaffold-backend/src/main/resources/META-INF/perspective.yaml) — perspective tab + dataset config
- [datasets.ts](scaffold-backend/src/main/webui/src/datasets.ts) — frontend data binding
- [index.ts](scaffold-backend/src/main/webui/src/index.ts) — view registry + perspective loading
- [OpsApplicationApi](ops/service/src/main/java/io/casehub/ops/service/api/OpsApplicationApi.java) — fleet management API
- [OpsDeploymentApi](ops/service/src/main/java/io/casehub/ops/service/api/OpsDeploymentApi.java) — deployment lifecycle API
- [OpsReconciliationApi](ops/service/src/main/java/io/casehub/ops/service/api/OpsReconciliationApi.java) — drift detection API
- [Phase 1 review corrections](plans/2026-09-21-ops-perspective-phase1-review-corrections.md) — R1-architecture constraint
- [Decisions](specs/issue-52-ops-perspective/decisions.md) — 12 validated design decisions
- [GitHub #52](https://github.com/casehubio/scaffold/issues/52) — parent ops perspective issue
- [GitHub #53](https://github.com/casehubio/scaffold/issues/53) — this issue
- [casehub-ops#118](https://github.com/casehubio/casehub-ops/issues/118) — service layer extraction (landed)
- [casehub-ops#119](https://github.com/casehubio/casehub-ops/issues/119) — GraphQL generation enabled (landed)
