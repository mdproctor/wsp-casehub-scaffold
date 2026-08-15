# Scaffold Console Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #41 — Add generic platform console to scaffold
**Issue group:** #42, #43, #44, #45, #46, #47

**Goal:** Add a generic platform console (web UI) to scaffold that composes casehub-pages and blocks-ui components into a tabbed dashboard covering the full CaseHub platform surface.

**Architecture:** scaffold becomes a multi-module Maven project. The existing single module becomes `scaffold-backend`. A new `scaffold-web` module adds a Lit + TypeScript frontend bundled by Quinoa. The frontend is pure composition — zero new Lit components — using pages' DSL (`page()`, `tabs()`, `hostPanel()`) to wire blocks-ui workbenches to existing REST endpoints.

**Tech Stack:** Java 21 / Quarkus 3.32.2, Lit 3 / TypeScript / esbuild / Quinoa 2.8.3, casehub-pages DSL, blocks-ui web components

## Global Constraints

- Quarkus version: `3.32.2` (match `version.quarkus.platform` in existing pom.xml)
- Quinoa version: `2.8.3` (from casehub-parent `<quarkus-quinoa.version>`)
- blocks-ui-npm version: `0.1-SNAPSHOT`
- pages-npm version: `0.2-SNAPSHOT`
- Zero new Lit web components — composition only via pages DSL
- All REST endpoints called directly from the browser — no BFF
- Frontend file pattern follows devtown exactly (esbuild.config.mjs, index.html, src/index.ts, src/datasets.ts, src/views/*.ts)
- Tab navigation via `site.navigate(tabLabel)` — `LiveSite.navigate(path)` walks the component tree using slot names
- Events via `emitPagesEvent(target, topic, payload)` from `@casehubio/pages-component` — first arg is an `EventTarget` (use the app container element)

---

### Task 1: Convert scaffold to multi-module Maven project (#42)

**Files:**
- Create: `scaffold/pom.xml` (new parent POM)
- Move: existing `scaffold/pom.xml` → `scaffold/scaffold-backend/pom.xml` (manual — not a Java source file)
- Move: existing `scaffold/src/` → `scaffold/scaffold-backend/src/` (manual — directory move)
- Move: existing `scaffold/mvnw`, `scaffold/mvnw.cmd` → keep at root

**Interfaces:**
- Produces: Multi-module project structure that builds with `mvn install`

- [ ] **Step 1: Create the parent POM**

Create `scaffold/pom.xml` as a `pom` parent with two modules:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>io.casehub</groupId>
    <artifactId>scaffold</artifactId>
    <version>0.2-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>CaseHub Scaffold</name>
    <description>Deployable microservice for CaseHub platform — backend + optional web console</description>

    <modules>
        <module>scaffold-backend</module>
    </modules>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <quarkus.platform.version>3.32.2</quarkus.platform.version>
        <quarkus-quinoa.version>2.8.3</quarkus-quinoa.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>io.quarkiverse.quinoa</groupId>
                <artifactId>quarkus-quinoa</artifactId>
                <version>${quarkus-quinoa.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <repositories>
        <repository>
            <id>github</id>
            <name>casehubio GitHub Packages</name>
            <url>https://maven.pkg.github.com/casehubio/*</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>

    <distributionManagement>
        <repository>
            <id>github</id>
            <name>casehubio GitHub Packages</name>
            <url>https://maven.pkg.github.com/casehubio/flow</url>
        </repository>
        <snapshotRepository>
            <id>github</id>
            <name>casehubio GitHub Packages</name>
            <url>https://maven.pkg.github.com/casehubio/flow</url>
            <uniqueVersion>false</uniqueVersion>
        </snapshotRepository>
    </distributionManagement>

    <scm>
        <connection>scm:git:https://github.com/casehubio/flow.git</connection>
        <developerConnection>scm:git:https://github.com/casehubio/flow.git</developerConnection>
        <url>https://github.com/casehubio/flow</url>
        <tag>HEAD</tag>
    </scm>
</project>
```

- [ ] **Step 2: Create scaffold-backend directory and move sources**

```bash
mkdir scaffold-backend
git mv src scaffold-backend/
```

- [ ] **Step 3: Create scaffold-backend/pom.xml from existing pom.xml**

Copy the existing `pom.xml` content into `scaffold-backend/pom.xml`. Change:
- Add `<parent>` pointing to the new parent POM
- Change `<artifactId>` from `scaffold` to `scaffold-backend`
- Remove `<groupId>` and `<version>` (inherited from parent)
- Remove the `<repositories>` section (inherited from parent)
- Remove `<distributionManagement>` (inherited from parent)
- Remove `<scm>` (inherited from parent)
- Remove `<properties>` that are now in the parent (`maven.compiler.release`, `project.build.sourceEncoding`, `project.reporting.outputEncoding`, `quarkus.platform.version`)
- Keep `<skipITs>`, `<compiler-plugin.version>`, `<surefire-plugin.version>` in child properties
- Keep all dependencies, plugins, and profiles as-is
- `<packaging>` stays as `quarkus`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>scaffold</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>scaffold-backend</artifactId>
    <packaging>quarkus</packaging>

    <properties>
        <compiler-plugin.version>3.15.0</compiler-plugin.version>
        <skipITs>false</skipITs>
        <surefire-plugin.version>3.5.4</surefire-plugin.version>
    </properties>

    <!-- Keep ALL existing <dependencyManagement>, <dependencies>, <build>, <profiles> sections
         exactly as they are in the current pom.xml -->
</project>
```

The comment above is a placeholder — the actual file must contain the full dependency management, dependencies, build plugins, and profiles sections copied verbatim from the current `pom.xml`. Do not abbreviate.

- [ ] **Step 4: Replace the root pom.xml with the parent POM**

Delete the original `pom.xml` and replace with the parent POM from Step 1.

- [ ] **Step 5: Update CI workflow**

Update `.github/workflows/maven.yml`:
- Surefire reports path: change `**/target/surefire-reports/*` (already uses glob, should still work)
- Add Node.js setup step before build (for Quinoa):

```yaml
      - name: Set up Node.js
        if: github.event_name != 'pull_request' || github.event.action != 'closed' || github.event.pull_request.merged == true
        uses: actions/setup-node@v4
        with:
          node-version: '20'
```

- [ ] **Step 6: Verify the backend builds**

Run: `mvn --batch-mode install -pl scaffold-backend -am -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 7: Run existing tests**

Run: `mvn --batch-mode test -pl scaffold-backend`
Expected: All existing tests pass

- [ ] **Step 8: Commit**

```bash
git add pom.xml scaffold-backend/ .github/
git commit -m "chore: convert scaffold to multi-module — move backend to scaffold-backend

Refs casehubio/scaffold#42"
```

---

### Task 2: Add ModuleRegistryResource to scaffold-backend (#43)

**Files:**
- Create: `scaffold-backend/src/main/java/io/casehub/flow/rest/ModuleRegistryResource.java`
- Create: `scaffold-backend/src/test/java/io/casehub/flow/rest/ModuleRegistryResourceTest.java`

**Interfaces:**
- Produces: `GET /api/modules` → `{"modules": ["engine", "work", ...]}` based on classpath probing

- [ ] **Step 1: Write the failing test**

Use `ide_create_file` to create `scaffold-backend/src/test/java/io/casehub/flow/rest/ModuleRegistryResourceTest.java`:

```java
package io.casehub.flow.rest;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import static org.hamcrest.Matchers.*;

@QuarkusTest
class ModuleRegistryResourceTest {

    @Test
    void returnsAvailableModules() {
        RestAssured.given()
            .when().get("/api/modules")
            .then()
            .statusCode(200)
            .body("modules", hasItem("engine"))
            .body("modules", hasItem("work"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl scaffold-backend -Dtest=ModuleRegistryResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — 404 (endpoint does not exist)

- [ ] **Step 3: Write the implementation**

Use `ide_create_file` to create `scaffold-backend/src/main/java/io/casehub/flow/rest/ModuleRegistryResource.java`:

```java
package io.casehub.flow.rest;

import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@Path("/api/modules")
@Produces(MediaType.APPLICATION_JSON)
public class ModuleRegistryResource {

    private final List<String> modules;

    @Inject
    public ModuleRegistryResource() {
        var detected = new ArrayList<String>();
        probeClass("io.casehub.engine.CaseHubRuntime", "engine", detected);
        probeClass("io.casehub.work.runtime.service.WorkItemService", "work", detected);
        probeClass("io.casehub.engine.plan.DagPlan", "planning", detected);
        probeClass("io.casehub.ledger.LedgerService", "ledger", detected);
        probeClass("io.casehub.ops.app.OpsApplication", "ops", detected);
        probeClass("io.casehub.iot.webapp.IoTApplication", "iot", detected);
        this.modules = List.copyOf(detected);
    }

    @GET
    public Map<String, List<String>> getModules() {
        return Map.of("modules", modules);
    }

    private static void probeClass(String className, String moduleName, List<String> target) {
        try {
            Class.forName(className, false, Thread.currentThread().getContextClassLoader());
            target.add(moduleName);
        } catch (ClassNotFoundException ignored) {
        }
    }
}
```

Note: The probe class names are best-guesses. Use `ide_find_class` during implementation to find the actual canonical class for each module (search for the module's main runtime class or CDI producer). The pattern is correct — the class names may need adjustment.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl scaffold-backend -Dtest=ModuleRegistryResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Run diagnostics**

Run: `ide_diagnostics` on `ModuleRegistryResource.java`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add scaffold-backend/src/main/java/io/casehub/flow/rest/ModuleRegistryResource.java
git add scaffold-backend/src/test/java/io/casehub/flow/rest/ModuleRegistryResourceTest.java
git commit -m "feat: add /api/modules endpoint for runtime module detection

Closes casehubio/scaffold#43"
```

---

### Task 3: Create scaffold-web Maven module with Quinoa (#44)

**Files:**
- Create: `scaffold-web/pom.xml`
- Create: `scaffold-web/src/main/webui/package.json`
- Create: `scaffold-web/src/main/webui/tsconfig.json`
- Create: `scaffold-web/src/main/webui/esbuild.config.mjs`
- Create: `scaffold-web/src/main/webui/index.html`
- Create: `scaffold-web/src/main/webui/src/index.ts` (minimal placeholder)
- Create: `scaffold-web/src/main/webui/.gitignore`
- Modify: `scaffold-backend/src/main/resources/application.properties` (add Quinoa config)

**Interfaces:**
- Consumes: `casehub-pages-npm` (0.2-SNAPSHOT), `casehub-blocks-ui-npm` (0.1-SNAPSHOT) via maven-dependency-plugin
- Produces: A Quarkus JAR that serves the bundled frontend at `/`

- [ ] **Step 1: Create scaffold-web/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>scaffold</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>scaffold-web</artifactId>
    <name>Scaffold :: Web Console</name>
    <description>Generic CaseHub platform console — pages + blocks-ui composition</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>scaffold-backend</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkiverse.quinoa</groupId>
            <artifactId>quarkus-quinoa</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-npm</artifactId>
            <version>0.2-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-blocks-ui-npm</artifactId>
            <version>0.1-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <executions>
                    <execution>
                        <id>unpack-casehub-packages</id>
                        <phase>initialize</phase>
                        <goals><goal>unpack</goal></goals>
                        <configuration>
                            <artifactItems>
                                <artifactItem>
                                    <groupId>io.casehub</groupId>
                                    <artifactId>casehub-pages-npm</artifactId>
                                    <version>0.2-SNAPSHOT</version>
                                    <outputDirectory>${project.basedir}/src/main/webui/.casehub-packages</outputDirectory>
                                </artifactItem>
                                <artifactItem>
                                    <groupId>io.casehub</groupId>
                                    <artifactId>casehub-blocks-ui-npm</artifactId>
                                    <version>0.1-SNAPSHOT</version>
                                    <outputDirectory>${project.basedir}/src/main/webui/.casehub-packages</outputDirectory>
                                </artifactItem>
                            </artifactItems>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>io.quarkus.platform</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <version>${quarkus.platform.version}</version>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals>
                            <goal>build</goal>
                            <goal>generate-code</goal>
                            <goal>generate-code-tests</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Create package.json**

Create `scaffold-web/src/main/webui/package.json`:

```json
{
  "name": "scaffold-console",
  "private": true,
  "scripts": {
    "build": "node esbuild.config.mjs",
    "dev": "node esbuild.config.mjs --watch",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@casehubio/blocks-ui-approval-gate": "file:.casehub-packages/tarballs/approval-gate.tgz",
    "@casehubio/blocks-ui-audit-trail-viewer": "file:.casehub-packages/tarballs/audit-trail-viewer.tgz",
    "@casehubio/blocks-ui-case-explorer": "file:.casehub-packages/tarballs/case-explorer.tgz",
    "@casehubio/blocks-ui-compliance-summary": "file:.casehub-packages/tarballs/compliance-summary.tgz",
    "@casehubio/blocks-ui-core": "file:.casehub-packages/tarballs/blocks-ui-core.tgz",
    "@casehubio/blocks-ui-detail-pane": "file:.casehub-packages/tarballs/detail-pane.tgz",
    "@casehubio/blocks-ui-execution-monitor": "file:.casehub-packages/tarballs/execution-monitor.tgz",
    "@casehubio/blocks-ui-kpi-metric-row": "file:.casehub-packages/tarballs/kpi-metric-row.tgz",
    "@casehubio/blocks-ui-list-pane": "file:.casehub-packages/tarballs/list-pane.tgz",
    "@casehubio/blocks-ui-notification-inbox": "file:.casehub-packages/tarballs/notification-inbox.tgz",
    "@casehubio/blocks-ui-orchestration-workbench": "file:.casehub-packages/tarballs/orchestration-workbench.tgz",
    "@casehubio/blocks-ui-preferences-editor": "file:.casehub-packages/tarballs/preferences-editor.tgz",
    "@casehubio/blocks-ui-routing-rationale": "file:.casehub-packages/tarballs/routing-rationale.tgz",
    "@casehubio/blocks-ui-session-workbench": "file:.casehub-packages/tarballs/session-workbench.tgz",
    "@casehubio/blocks-ui-split-workbench": "file:.casehub-packages/tarballs/split-workbench.tgz",
    "@casehubio/blocks-ui-trust-workbench": "file:.casehub-packages/tarballs/trust-workbench.tgz",
    "@casehubio/blocks-ui-work-item-workbench": "file:.casehub-packages/tarballs/work-item-workbench.tgz",
    "@casehubio/pages-component": "file:.casehub-packages/tarballs/pages-component.tgz",
    "@casehubio/pages-data": "file:.casehub-packages/tarballs/pages-data.tgz",
    "@casehubio/pages-primitives": "file:.casehub-packages/tarballs/pages-primitives.tgz",
    "@casehubio/pages-runtime": "file:.casehub-packages/tarballs/pages-runtime.tgz",
    "@casehubio/pages-table": "file:.casehub-packages/tarballs/pages-table.tgz",
    "@casehubio/pages-ui": "file:.casehub-packages/tarballs/pages-ui.tgz",
    "@casehubio/pages-ui-components": "file:.casehub-packages/tarballs/pages-ui-components.tgz",
    "@casehubio/pages-ui-tokens": "file:.casehub-packages/tarballs/pages-ui-tokens.tgz",
    "@casehubio/pages-viz": "file:.casehub-packages/tarballs/pages-viz.tgz"
  },
  "devDependencies": {
    "esbuild": "^0.25.0",
    "typescript": "^5.6.0"
  }
}
```

- [ ] **Step 3: Create tsconfig.json**

Create `scaffold-web/src/main/webui/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "useDefineForClassFields": false,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": false,
    "noEmit": true
  },
  "include": ["src"]
}
```

- [ ] **Step 4: Create esbuild.config.mjs**

Create `scaffold-web/src/main/webui/esbuild.config.mjs`:

```javascript
import { build, context } from "esbuild";
import { copyFileSync, mkdirSync } from "fs";

const isWatch = process.argv.includes("--watch");

const options = {
  entryPoints: ["src/index.ts"],
  bundle: true,
  outfile: "dist/app.js",
  format: "esm",
  target: "es2020",
  minify: !isWatch,
  sourcemap: isWatch,
};

mkdirSync("dist", { recursive: true });
copyFileSync("index.html", "dist/index.html");

if (isWatch) {
  const ctx = await context(options);
  await ctx.watch();
  console.log("Watching for changes...");
} else {
  await build(options);
}
```

- [ ] **Step 5: Create index.html**

Create `scaffold-web/src/main/webui/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>CaseHub Console</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body { height: 100%; }
    #app {
      min-height: 100%;
      font-family: var(--casehub-font, system-ui, sans-serif);
      background: var(--casehub-bg, #fff);
      color: var(--casehub-text, #333);
    }
  </style>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="app.js"></script>
</body>
</html>
```

- [ ] **Step 6: Create minimal index.ts**

Create `scaffold-web/src/main/webui/src/index.ts`:

```typescript
import { loadSite } from "@casehubio/pages-runtime";
import { page, tabs, title, rows } from "@casehubio/pages-ui";

async function start() {
  const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;

  const app = page("CaseHub Console",
    tabs(
      ["Home", rows(title("CaseHub Console", "h2"))],
    ),
    { settings: { mode: prefersDark ? "dark" : "light" } },
  );

  const container = document.getElementById("app");
  if (container) {
    const site = await loadSite(container, app);
    site.setTheme(prefersDark ? "dark" : "light");

    window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change", (e) => {
      site.setTheme(e.matches ? "dark" : "light");
    });
  }
}

start().catch(console.error);
```

- [ ] **Step 7: Create .gitignore**

Create `scaffold-web/src/main/webui/.gitignore`:

```
node_modules/
dist/
.casehub-packages/
```

- [ ] **Step 8: Add scaffold-web module to parent POM**

Add `<module>scaffold-web</module>` to the `<modules>` list in the root `pom.xml`:

```xml
    <modules>
        <module>scaffold-backend</module>
        <module>scaffold-web</module>
    </modules>
```

- [ ] **Step 9: Add Quinoa config to application.properties**

Add to `scaffold-backend/src/main/resources/application.properties`:

```properties
# Quinoa (scaffold-web frontend)
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=false
quarkus.quinoa.enable-spa-routing=true
quarkus.http.auth.permission.quinoa-static.paths=/,/index.html,/app.js
quarkus.http.auth.permission.quinoa-static.policy=permit
```

- [ ] **Step 10: Verify the full build**

Run: `mvn --batch-mode install -DskipTests`
Expected: BUILD SUCCESS — both modules build, Quinoa bundles the frontend

- [ ] **Step 11: Verify the app serves the UI**

Run: `mvn --batch-mode quarkus:dev -pl scaffold-web`
Open: `http://localhost:8080`
Expected: Page renders with "CaseHub Console" title in a tab

- [ ] **Step 12: Commit**

```bash
git add scaffold-web/ scaffold-backend/src/main/resources/application.properties
git commit -m "feat: add scaffold-web module — Quinoa + pages shell

Closes casehubio/scaffold#44"
```

---

### Task 4: Wire datasets and all 10 tab views (#45)

**Files:**
- Create: `scaffold-web/src/main/webui/src/datasets.ts`
- Create: `scaffold-web/src/main/webui/src/views/cases.ts`
- Create: `scaffold-web/src/main/webui/src/views/work-items.ts`
- Create: `scaffold-web/src/main/webui/src/views/queues.ts`
- Create: `scaffold-web/src/main/webui/src/views/orchestration.ts`
- Create: `scaffold-web/src/main/webui/src/views/trust-audit.ts`
- Create: `scaffold-web/src/main/webui/src/views/operations.ts`
- Create: `scaffold-web/src/main/webui/src/views/iot.ts`
- Create: `scaffold-web/src/main/webui/src/views/notifications.ts`
- Create: `scaffold-web/src/main/webui/src/views/sessions.ts`
- Create: `scaffold-web/src/main/webui/src/views/system.ts`
- Modify: `scaffold-web/src/main/webui/src/index.ts`

**Interfaces:**
- Consumes: `GET /api/modules` from Task 2, all existing platform REST endpoints
- Produces: Full 10-tab console with conditional OPS/IoT tabs

- [ ] **Step 1: Create datasets.ts**

Create `scaffold-web/src/main/webui/src/datasets.ts`:

```typescript
import { bind, restSource } from "@casehubio/pages-ui";
import type { DataSetId } from "@casehubio/pages-data";

function rest(id: string, url: string, opts?: {
  dataPath?: string;
  expression?: string;
  refreshTime?: string;
}) {
  const binding = bind(id, restSource(url, id as DataSetId, opts));
  return opts?.refreshTime ? { ...binding, refreshTime: opts.refreshTime } : binding;
}

export function createDatasets(modules: string[]) {
  const datasets = [
    rest("cases", "/api/v1/cases", { refreshTime: "10second" }),
    rest("case-definitions", "/api/v1/definitions"),
    rest("workers", "/api/v1/workers", { refreshTime: "10second" }),
    rest("work-items", "/workitems", { refreshTime: "5second" }),
    rest("queues", "/queues", { refreshTime: "10second" }),
    rest("actors", "/workitems/actors", { refreshTime: "30second" }),
    rest("audit", "/audit"),
    rest("plan-items", "/api/v1/cases/#{row.caseId}/plan-items"),
    rest("goals", "/api/v1/cases/#{row.caseId}/goals"),
    rest("case-context", "/api/v1/cases/#{row.caseId}/context"),
  ];

  if (modules.includes("ops")) {
    datasets.push(
      rest("applications", "/api/applications", { refreshTime: "10second" }),
      rest("approvals", "/api/approvals", { refreshTime: "5second" }),
    );
  }

  if (modules.includes("iot")) {
    datasets.push(
      rest("devices", "/api/devices", { refreshTime: "5second" }),
      rest("situations", "/api/situations", { refreshTime: "10second" }),
      rest("providers", "/api/providers", { refreshTime: "30second" }),
    );
  }

  return datasets;
}
```

- [ ] **Step 2: Create cases.ts**

Create `scaffold-web/src/main/webui/src/views/cases.ts`:

```typescript
import { hostPanel } from "@casehubio/pages-ui";
import { registerPanel } from "@casehubio/pages-runtime";
import "@casehubio/blocks-ui-case-explorer";

registerPanel("case-explorer", "blocks-case-explorer");

export const casesView = hostPanel("case-explorer", {
  entityTypes: "all-presets",
  endpoint: "/api/v1",
});
```

- [ ] **Step 3: Create work-items.ts**

Create `scaffold-web/src/main/webui/src/views/work-items.ts`:

```typescript
import { hostPanel } from "@casehubio/pages-ui";
import { registerPanel } from "@casehubio/pages-runtime";
import "@casehubio/blocks-ui-work-item-workbench";

registerPanel("work-item-workbench", "blocks-work-item-workbench");

export const workItemsView = hostPanel("work-item-workbench", {
  endpoint: "/workitems",
});
```

- [ ] **Step 4: Create queues.ts**

Create `scaffold-web/src/main/webui/src/views/queues.ts`:

```typescript
import { rows, dataTable, hostPanel } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";
import { registerPanel } from "@casehubio/pages-runtime";
import "@casehubio/blocks-ui-kpi-metric-row";

registerPanel("kpi-metric-row", "blocks-kpi-metric-row");

export const queuesView = rows(
  hostPanel("kpi-metric-row", { endpoint: "/queues/health" }),
  dataTable({
    lookup: lookup("queues", groupBy(null,
      col("id"), col("name"), col("pendingCount"),
      col("activeCount"), col("completedCount"),
    )),
    sortable: true,
    filter: { enabled: true },
  }),
);
```

- [ ] **Step 5: Create orchestration.ts**

Create `scaffold-web/src/main/webui/src/views/orchestration.ts`:

```typescript
import { rows, dataTable, hostPanel } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";
import { registerPanel } from "@casehubio/pages-runtime";
import "@casehubio/blocks-ui-orchestration-workbench";

registerPanel("orchestration-workbench", "blocks-orchestration-workbench");

export const orchestrationView = rows(
  dataTable({
    lookup: lookup("cases", groupBy(null,
      col("id"), col("name"), col("status"),
      col("namespace"), col("createdAt"),
    )),
    sortable: true,
    filter: { enabled: true },
    selectionTopic: "active-case",
  }),
  hostPanel("orchestration-workbench", {
    endpoint: "/api/v1",
    "case-id": "#{row.id}",
  }),
);
```

- [ ] **Step 6: Create trust-audit.ts**

Create `scaffold-web/src/main/webui/src/views/trust-audit.ts`:

```typescript
import { columns, rows, dataTable, hostPanel } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";
import { registerPanel } from "@casehubio/pages-runtime";
import "@casehubio/blocks-ui-trust-workbench";
import "@casehubio/blocks-ui-audit-trail-viewer";
import "@casehubio/blocks-ui-compliance-summary";
import "@casehubio/blocks-ui-routing-rationale";

registerPanel("trust-workbench", "blocks-trust-workbench");
registerPanel("audit-trail-viewer", "blocks-audit-trail-viewer");
registerPanel("compliance-summary", "blocks-compliance-summary");
registerPanel("routing-rationale", "blocks-routing-rationale");

export const trustAuditView = columns([4, 8],
  [dataTable({
    lookup: lookup("actors", groupBy(null,
      col("actorId"), col("actorType"), col("trustScore"),
    )),
    sortable: true,
    selectionTopic: "selected-actor",
  })],
  [rows(
    hostPanel("trust-workbench", {
      endpoint: "/workitems/actors",
      "actor-id": "#{row.actorId}",
    }),
    hostPanel("audit-trail-viewer", {
      endpoint: "/audit",
      "subject-id": "#{row.actorId}",
    }),
    hostPanel("compliance-summary", {
      endpoint: "/audit/compliance",
    }),
    hostPanel("routing-rationale", {
      endpoint: "/workitems/actors",
      "actor-id": "#{row.actorId}",
    }),
  )],
);
```

- [ ] **Step 7: Create operations.ts**

Create `scaffold-web/src/main/webui/src/views/operations.ts`:

```typescript
import { rows, columns, dataTable, hostPanel, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";
import { registerPanel } from "@casehubio/pages-runtime";
import "@casehubio/blocks-ui-kpi-metric-row";
import "@casehubio/blocks-ui-approval-gate";

registerPanel("approval-gate", "blocks-approval-gate");

export const operationsView = rows(
  hostPanel("kpi-metric-row", { endpoint: "/api/applications/health" }),
  columns([4, 8],
    [dataTable({
      lookup: lookup("applications", groupBy(null,
        col("id"), col("name"), col("status"),
        col("version"), col("lastDeployed"),
      )),
      sortable: true,
      selectionTopic: "selected-app",
    })],
    [rows(
      title("Deployments", "h3"),
      dataTable({
        lookup: lookup("applications", groupBy(null,
          col("deploymentId"), col("version"),
          col("status"), col("timestamp"),
        )),
      }),
      hostPanel("approval-gate", {
        endpoint: "/api/approvals",
      }),
    )],
  ),
);
```

- [ ] **Step 8: Create iot.ts**

Create `scaffold-web/src/main/webui/src/views/iot.ts`:

```typescript
import { rows, columns, dataTable, title } from "@casehubio/pages-ui";
import { lookup, col, groupBy } from "@casehubio/pages-ui";

export const iotView = columns([4, 8],
  [dataTable({
    lookup: lookup("devices", groupBy(null,
      col("id"), col("name"), col("type"),
      col("status"), col("provider"),
    )),
    sortable: true,
    filter: { enabled: true },
    selectionTopic: "selected-device",
  })],
  [rows(
    title("Device State", "h3"),
    dataTable({
      lookup: lookup("devices", groupBy(null,
        col("property"), col("value"),
        col("lastUpdated"),
      )),
    }),
    title("Situations", "h3"),
    dataTable({
      lookup: lookup("situations", groupBy(null,
        col("id"), col("name"), col("severity"),
        col("status"), col("detectedAt"),
      )),
      sortable: true,
    }),
  )],
);
```

- [ ] **Step 9: Create notifications.ts**

Create `scaffold-web/src/main/webui/src/views/notifications.ts`:

```typescript
import { tabs, hostPanel } from "@casehubio/pages-ui";
import { registerPanel } from "@casehubio/pages-runtime";
import "@casehubio/blocks-ui-notification-inbox";

registerPanel("notification-inbox", "blocks-notification-inbox");
registerPanel("notification-preferences", "blocks-notification-preferences");

export const notificationsView = tabs(
  ["Inbox", hostPanel("notification-inbox", {
    endpoint: "/notifications",
  })],
  ["Preferences", hostPanel("notification-preferences", {
    endpoint: "/subscriptions",
  })],
);
```

- [ ] **Step 10: Create sessions.ts**

Create `scaffold-web/src/main/webui/src/views/sessions.ts`:

```typescript
import { hostPanel } from "@casehubio/pages-ui";
import { registerPanel } from "@casehubio/pages-runtime";
import "@casehubio/blocks-ui-session-workbench";

registerPanel("session-workbench", "blocks-session-workbench");

export const sessionsView = hostPanel("session-workbench", {
  endpoint: "/api/sessions",
});
```

- [ ] **Step 11: Create system.ts**

Create `scaffold-web/src/main/webui/src/views/system.ts`:

```typescript
import { tabs, hostPanel } from "@casehubio/pages-ui";
import { registerPanel } from "@casehubio/pages-runtime";
import "@casehubio/blocks-ui-case-explorer";
import "@casehubio/blocks-ui-preferences-editor";
import "@casehubio/blocks-ui-kpi-metric-row";

registerPanel("case-definition-browser", "blocks-case-definition-browser");
registerPanel("preferences-editor", "blocks-preferences-editor");

export const systemView = tabs(
  ["Definitions", hostPanel("case-definition-browser", {
    endpoint: "/api/v1/definitions",
  })],
  ["Preferences", hostPanel("preferences-editor", {
    endpoint: "/api/preferences",
  })],
  ["Health", hostPanel("kpi-metric-row", {
    endpoint: "/q/health",
  })],
);
```

- [ ] **Step 12: Update index.ts to compose all tabs with module detection**

Replace `scaffold-web/src/main/webui/src/index.ts`:

```typescript
import { loadSite } from "@casehubio/pages-runtime";
import { page, tabs } from "@casehubio/pages-ui";
import type { Component } from "@casehubio/pages-component";
import { createDatasets } from "./datasets";
import { casesView } from "./views/cases";
import { workItemsView } from "./views/work-items";
import { queuesView } from "./views/queues";
import { orchestrationView } from "./views/orchestration";
import { trustAuditView } from "./views/trust-audit";
import { operationsView } from "./views/operations";
import { iotView } from "./views/iot";
import { notificationsView } from "./views/notifications";
import { sessionsView } from "./views/sessions";
import { systemView } from "./views/system";

type TabEntry = [string, Component];

async function start() {
  const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;

  const { modules } = await fetch("/api/modules").then(r => r.json());

  const allTabs: TabEntry[] = [
    ["Cases", casesView],
    ["Work Items", workItemsView],
    ["Queues", queuesView],
    ["Orchestration", orchestrationView],
    ["Trust & Audit", trustAuditView],
    ...(modules.includes("ops") ? [["Operations", operationsView] as TabEntry] : []),
    ...(modules.includes("iot") ? [["IoT", iotView] as TabEntry] : []),
    ["Notifications", notificationsView],
    ["Sessions", sessionsView],
    ["System", systemView],
  ];

  const app = page("CaseHub Console",
    tabs(...allTabs),
    { settings: { mode: prefersDark ? "dark" : "light" }, datasets: createDatasets(modules) },
  );

  const container = document.getElementById("app");
  if (container) {
    const site = await loadSite(container, app);
    site.setTheme(prefersDark ? "dark" : "light");

    window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change", (e) => {
      site.setTheme(e.matches ? "dark" : "light");
    });
  }
}

start().catch(console.error);
```

- [ ] **Step 13: Verify build**

Run: `mvn --batch-mode install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 14: Verify all tabs render**

Run: `mvn --batch-mode quarkus:dev -pl scaffold-web`
Open: `http://localhost:8080`
Expected: 8 tabs visible (OPS and IoT hidden since modules not on classpath). Click each tab — components render without JS errors.

- [ ] **Step 15: Commit**

```bash
git add scaffold-web/src/main/webui/src/
git commit -m "feat: wire all 10 tab views — cases, work items, queues, orchestration, trust, ops, iot, notifications, sessions, system

Closes casehubio/scaffold#45"
```

---

### Task 5: Add URL hash navigation for cross-tab deep-linking (#46)

**Files:**
- Create: `scaffold-web/src/main/webui/src/navigation.ts`
- Modify: `scaffold-web/src/main/webui/src/index.ts`

**Interfaces:**
- Consumes: `LiveSite.navigate(path)` from pages-runtime (walks component tree using slot names / tab labels)
- Consumes: `emitPagesEvent(target: EventTarget, topic: string, payload: T)` from `@casehubio/pages-component`
- Produces: URL hash routing — `#/cases/abc-123` navigates to Cases tab with entity selected

- [ ] **Step 1: Create navigation.ts**

Create `scaffold-web/src/main/webui/src/navigation.ts`:

```typescript
import { emitPagesEvent } from "@casehubio/pages-component";
import type { LiveSite } from "@casehubio/pages-runtime";

interface NavigationState {
  tab: string;
  entityId?: string;
}

const TAB_TOPICS: Record<string, string> = {
  "cases": "case-explorer",
  "work-items": "work-item",
  "queues": "queue",
  "orchestration": "active-case",
  "trust-audit": "selected-actor",
  "operations": "selected-app",
  "iot": "selected-device",
  "notifications": "notification",
  "sessions": "session",
  "system": "system",
};

const SLUG_TO_LABEL: Record<string, string> = {
  "cases": "Cases",
  "work-items": "Work Items",
  "queues": "Queues",
  "orchestration": "Orchestration",
  "trust-audit": "Trust & Audit",
  "operations": "Operations",
  "iot": "IoT",
  "notifications": "Notifications",
  "sessions": "Sessions",
  "system": "System",
};

export function parseHash(hash: string): NavigationState | null {
  const match = hash.match(/^#\/([a-z-]+)(?:\/(.+))?$/);
  if (!match) return null;
  return { tab: match[1], entityId: match[2] };
}

export function buildHash(tab: string, entityId?: string): string {
  return entityId ? `#/${tab}/${entityId}` : `#/${tab}`;
}

export function setupNavigation(site: LiveSite, appContainer: HTMLElement) {
  function navigate() {
    const state = parseHash(location.hash);
    if (!state) return;
    const label = SLUG_TO_LABEL[state.tab];
    if (label) {
      site.navigate(label);
    }
    if (state.entityId) {
      const topic = TAB_TOPICS[state.tab];
      if (topic) {
        setTimeout(() => {
          emitPagesEvent(appContainer, `${topic}:selected`, { id: state.entityId });
        }, 100);
      }
    }
  }

  window.addEventListener("hashchange", navigate);
  if (location.hash) navigate();
}
```

- [ ] **Step 2: Wire navigation into index.ts**

Add import to the top of `index.ts`:

```typescript
import { setupNavigation } from "./navigation";
```

Add after `const site = await loadSite(container, app);`:

```typescript
    setupNavigation(site, container);
```

- [ ] **Step 3: Verify hash navigation**

Run: `mvn --batch-mode quarkus:dev -pl scaffold-web`
Open: `http://localhost:8080#/orchestration`
Expected: Orchestration tab is active on load

Open: `http://localhost:8080#/sessions`
Expected: Sessions tab is active on load

Change URL to `#/cases` — tab switches without page reload.

- [ ] **Step 4: Commit**

```bash
git add scaffold-web/src/main/webui/src/navigation.ts
git add scaffold-web/src/main/webui/src/index.ts
git commit -m "feat: add URL hash navigation for cross-tab deep-linking

Closes casehubio/scaffold#46"
```

---

### Task 6: Final verification and build cleanup (#47)

**Files:**
- Modify: `scaffold-web/src/main/webui/.gitignore` (if needed)
- Modify: root `.gitignore` (if needed)

**Interfaces:**
- Consumes: Everything from Tasks 1-5

- [ ] **Step 1: Full clean build**

Run: `mvn --batch-mode clean install`
Expected: BUILD SUCCESS for both modules, all tests pass

- [ ] **Step 2: Headless build verification**

Run: `mvn --batch-mode install -pl scaffold-backend -am`
Expected: BUILD SUCCESS — backend builds without scaffold-web

- [ ] **Step 3: Verify .gitignore covers generated files**

Ensure these are gitignored:
- `scaffold-web/src/main/webui/node_modules/`
- `scaffold-web/src/main/webui/dist/`
- `scaffold-web/src/main/webui/.casehub-packages/`

Run `git status` — none of the above should appear as untracked.

- [ ] **Step 4: Run full test suite**

Run: `mvn --batch-mode test -pl scaffold-backend`
Expected: All existing tests pass unchanged

- [ ] **Step 5: Update CLAUDE.md build commands**

Update the Build Commands section in CLAUDE.md to reflect multi-module:

```markdown
## Build Commands

```bash
# Build all modules
mvn --batch-mode install

# Build backend only (headless — no web console)
mvn --batch-mode install -pl scaffold-backend -am

# Skip tests
mvn --batch-mode install -DskipTests
```
```

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "chore: final verification — gitignore, build docs, CI cleanup

Closes casehubio/scaffold#47"
```
