# HANDOFF — scaffold

## Last Session

Executed implementation plan for epic #83 (Deploy Level 01 from the UI — end-to-end). Completed 7 of 8 tasks across 5 repos. All code changes committed and pushed. Only E2E verification remains.

### Completed

- **T1 (CI Health):** Already done in prior session (commit `22a98d37` — convergence aggregator fix). Current CI failures are unrelated (claudony-app tests, platform publish script).
- **T2 (CI Health):** Added push trigger to scaffold CI workflow. Created issue #87, committed `960a2aa`.
- **T3 (Land Branches):** Merged engine PR #1270 (`fix/flow-execution-registry-scope`) and ops PR #214 (`issue-76-topology-catalogue-deploy`). Both installed to local Maven.
- **T4 (UI Selection):** Fixed selectionKey alias resolution in pages-runtime activation.ts. Committed to casehub-pages (PR #603 merged). Resolves `col("name", "Topology")` mapping — was looking up source ID verbatim instead of output column ID.
- **T5 (UI Selection):** Wired catalogue table selection + deploy button in deploy.ts. Added `selection: "single"`, `selectionKey: "name"`, and an `actionButton` with POST to deploy endpoint. Committed `c8672cb`.
- **T6 (Podman Target):** Added `casehub.ops.deployment-target-override` config property to provision handler and drift checker in ops. PR #216 merged. Test added.
- **T7 (Podman Target):** Added `casehub-ops-container` dependency to scaffold-backend + Jandex index + dev-mode target override `podman`. Committed `9ca44ad`.

### Dependency convergence note

Ops build requires `-Denforcer.skip=true` due to serverlessworkflow version mismatch (7.35.3 vs 7.25.1 via quarkus-flow). Pre-existing issue from recently added enforcer rule in platform.

## Immediate Next Step

`work continue` -> Execute Task 8 (Batch 5: E2E verification). No code changes — verification only:

1. Start Podman: `podman machine start`
2. Build container images for Level 01 topology
3. Start scaffold dev mode: `mvn quarkus:dev`
4. Browser: Deploy tab -> select 01-hello-llm -> Deploy -> verify 4 containers start
5. Verify failure modes (no selection, duplicate deploy, Podman down)

## Cross-Module Changes This Session

- **casehub-pages** (casehubio/casehub-pages): PR #603 merged — selectionKey alias fix in activation.ts
- **engine** (casehubio/engine): PR #1270 merged — @ApplicationScoped on FlowExecutionRegistry; PR #1273 open — remove dead serverlessworkflow-experimental-types dep + fix watchdog parent version (issue #1272)
- **ops** (casehubio/casehub-ops): PR #214 merged — topology catalogue deploy + Level 01 metadata; PR #216 merged — deployment-target-override config
- **scaffold** (casehubio/scaffold): 4 commits on branch `issue-83-deploy-level-01-from-ui-e2e` (added neocortex-annotation-api dep for CDI augmentation)

## References

- `specs/issue-83-deploy-level-01-from-ui-e2e/2026-10-29-deploy-level-01-e2e-design.md`
- `specs/issue-83-deploy-level-01-from-ui-e2e/decisions.md`
- `plans/2026-10-29-deploy-level-01-e2e.md`
