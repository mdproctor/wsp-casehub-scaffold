# HANDOFF — scaffold

## Last Session

Implemented and landed #76 (topology catalogue + deploy). Fixed engine CDI bug (#1265). Created follow-up epics #77 and #83. Verified catalogue running in dev mode via Playwright.

- **#76** — Topology catalogue with compile-plan-deploy pipeline. 4 commits landed on main: service interface + 9 DTOs, EmbeddedTopologyCatalogueService (compile, plan, deploy with case lifecycle compensation), TopologyCatalogueResource (3 REST endpoints at `/api/scaffold/catalogue`), frontend catalogue table in Deploy tab + dev config.
- **Cross-repo (ops)** — extracted `compileTopology()` from `deploy()`, added `topology-metadata.yaml` + `playbook.yaml` for Level 01. Branch `issue-76-topology-catalogue-deploy` in ops — not yet merged to main.
- **Cross-repo (engine)** — fixed `FlowExecutionRegistry` missing `@ApplicationScoped` (casehubio/engine#1265). Branch `fix/flow-execution-registry-scope` — not yet merged to main.
- **Blog** — "From YAML to Running Infrastructure" with 2 SVG diagrams.
- **Issues filed** — Epic #77 (post-MVP follow-ups: #78-#82), Epic #83 (deploy Level 01 end-to-end: #84-#86).

## Immediate Next Step

`work start 83` → Deploy Level 01 from the UI. #84 and #85 are parallel:
- **#85** (XS) — land the ops + engine branches (mechanical merge + install)
- **#84** (S) — fix `dataTable` selection crash in pages-ui, wire deploy button

Then **#86** (M) — container images + deployment target routing for Podman.

## Key Context

- Catalogue API verified working: `curl http://localhost:8082/api/scaffold/catalogue` returns 13 topologies, `/01-hello-llm/plan` returns compiled DAG + layered plan
- `selection: "single"` on `dataTable` crashes with `TYPE_MISMATCH: Column "name" is NULL` — investigation notes in #84
- Level 01 YAML hardcodes `deploymentTarget: claudony` — needs override for Podman (#86)
- Pre-existing `LoaderIntegrationTest` failure: engine's `FlowExecutionRegistry` — fixed on branch, not yet merged
- Container images: `casehubio/manifest:latest`, `casehubio/eidos:latest`, `casehubio/engine:latest`, `casehubio/my-llm-app:latest`

## References

- `specs/issue-76-topology-catalogue-deploy/2026-10-28-topology-catalogue-deploy-design.md`
- `plans/2026-10-28-topology-catalogue-deploy.md`
- `blog/2026-10-29-mdp02-topology-catalogue.md`
- `blog/2026-10-22-mdp01-case-composition-layer.md`
