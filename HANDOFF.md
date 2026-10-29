# HANDOFF — scaffold

## Last Session

Implemented case definitions (#68), triaged stale issues (#24-27), landed three small issues (#28, #74, #75), designed and planned the topology catalogue (#76).

- **#68** — 8 YAML case definitions composing ops' domain cases via subCase bindings with CBR, milestones, and SLA tracking. BFF layer with CaseService + ApprovalService dual-impl pattern. Frontend cases tab + inline status indicators. Merged to main.
- **#28** — closed as already working (Quarkus dev mode restarts on resource changes)
- **#74** — getCaseDetail sub-case decomposition via PlanItemView.subCaseIds() (unblocked by engine#1260). Merged to main.
- **#75** — remaining 3 YAML case definitions (application-lifecycle, scaling-event, service-upgrade). Merged to main.
- **#76** — topology catalogue + deploy. Design spec written and reviewed (3 rounds, 15 issues, all resolved). Implementation plan written (3 batches, 7 tasks).

## Immediate Next Step

`work continue 76` → execute the implementation plan at `plans/2026-10-28-topology-catalogue-deploy.md`. Start with Batch 1: Tasks 1-2 are ops cross-repo changes (extract `compileTopology()`, add metadata/playbook files to Level 01). Check if ops is free before starting. Task 3 onwards is pure scaffold.

## Key Context

- Spec: `specs/issue-76-topology-catalogue-deploy/2026-10-28-topology-catalogue-deploy-design.md`
- Review: `/Users/mdproctor/reviews/casehub-scaffold/issue-76-topology-catalogue-deploy-20261028-112518/tracker.md`
- Settled decisions: `/api/scaffold/catalogue` path, layered `List<List<>>` plan structure, `compileTopology()` extraction, `pages-step-viewer` + `playbook.yaml`, split view with graphical default + MD editor
- 8 YAML case definitions in `scaffold-backend/src/main/resources/casehub/ops/`
- engine#1260 landed — PlanItemView has `subCaseIds()` and `capabilityName()`
- All scaffold issues closed except #76 (branch `issue-76-topology-catalogue-deploy`)

## References

- `plans/2026-10-28-topology-catalogue-deploy.md`
- `specs/issue-76-topology-catalogue-deploy/2026-10-28-topology-catalogue-deploy-design.md`
- `specs/issue-76-topology-catalogue-deploy/decisions.md`
- `blog/2026-10-22-mdp01-case-composition-layer.md`
