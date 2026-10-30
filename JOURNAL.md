# Design Journal — issue-83-deploy-level-01-from-ui-e2e

## 2026-10-29 — Brainstorming + Planning

**Session scope:** Full design cycle for epic #83 (Deploy Level 01 from the UI — end-to-end). Brainstorming through to implementation plan.

### Key Decisions

1. **D1 — selectionKey crash fix:** Fix in pages-runtime activation.ts to resolve selectionKey through column alias mapping (sourceId → columnId). Root cause: `col("name", "Topology")` remaps output column ID to "Topology", but `selectionKey: "name"` looks up source ID verbatim.

2. **D2 — Deploy-time target override:** Add `casehub.ops.deployment-target-override` config property to `CaseHubAppProvisionHandler.resolveTarget()`. Graph stays truthful (shows YAML values); runtime routes to configured target.

3. **D3 — Container images built locally:** Quarkus container-image-podman extension in each source repo via Maven profile. Images not published to registry yet.

4. **D4 — Fix reactor first:** casehub-all reactor is red (Jandex index build-order issue in platform Spring convergence). Missing Quarkus modules in `aggregator-convergence.xml` — `registry-jpa` and 7 other paired modules.

5. **D5 — Scaffold CI trigger gap:** `.github/workflows/maven.yml` lacks `push` trigger. #76 commits pushed to main were never CI-validated.

### Discoveries

- Template interpolation uses `#{selection.<datasetId>.<field>}` — confirmed via context-wiring tests
- Scaffold depends only on `casehub-ops-service`, not `casehub-ops-container` — PodmanDeploymentTarget not on classpath
- Ops uses "prebuilt image" model (`PrebuiltImagePipeline` @DefaultBean) — no image building in ops Maven lifecycle
- `CaseHubAppProvisionHandler` uses `@Any Instance<ApplicationDeploymentTarget>` for target routing via CDI iteration

### Artifacts

- Spec: `specs/issue-83-deploy-level-01-from-ui-e2e/2026-10-29-deploy-level-01-e2e-design.md`
- Decisions: `specs/issue-83-deploy-level-01-from-ui-e2e/decisions.md` (5 decisions)
- Plan: `plans/2026-10-29-deploy-level-01-e2e.md` (5 batches, 8 tasks)

### Next session

Execute the plan starting from Batch 1 (CI Health). Run `work continue` to resume.
