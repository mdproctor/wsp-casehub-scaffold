# Decisions — issue-76-topology-catalogue-deploy

## D1: Playbook/tutorial view

**Choice:** Static README rendering
**Alternatives:**
- Full mosaic playbook-controller — requires playbook authoring and scenario server, overkill for MVP
- Start static, wire playbook later — unnecessary abstraction when we'll align with casehub-examples later
**Rationale:** Just get it running. Tutorial UI will be reworked when aligning with casehub-examples (which has a helpdesk example with proper tutorial UX).
**Trade-offs:** No step-by-step interactivity — the walkthrough is read-only markdown
**Sources:** mosaic playbook-controller.ts, casehub-examples index.html
**Exploration:** quick
**Status:** captured

## D2: Desired-state graph visualisation

**Choice:** Both — table as primary view, DAG diagram as secondary visual
**Alternatives:**
- Table only — fastest but loses the visual graph that makes dependencies intuitive
- DAG only — loses the detailed tabular data (node type, status, action)
**Rationale:** The table shows deploy plan detail (terraform-style), the DAG shows the shape at a glance. Both are useful and the data source is the same.
**Trade-offs:** More frontend work — need to integrate mosaic graph-renderer alongside pages-ui dataTable
**Sources:** mosaic graph-core, graph-renderer, DesiredStateGraph API
**Exploration:** quick
**Status:** captured

## D3: Topology YAML source

**Choice:** Read from filesystem via config property
**Alternatives:**
- Bundle in scaffold resources — self-contained but duplicates ops content
- Fetch from GitHub at runtime — true to the vision but adds HTTP/auth complexity
**Rationale:** Point `casehub.ops.topology.path` at the ops repo checkout. Works for local dev and CI where repos are co-located. Avoids duplication.
**Trade-offs:** Requires ops repo to be cloned alongside scaffold. Not self-contained.
**Sources:** TopologyCompilerService.topologyPath config property (already exists)
**Exploration:** quick
**Status:** captured

## D4: Case creation on deploy

**Choice:** Create application-lifecycle case on deploy
**Alternatives:**
- Reconciliation loop only — simpler MVP but loses the case-managed lifecycle
**Rationale:** The whole point is that deploying creates a case that owns the lifecycle. Deferring this defeats the purpose — the Cases tab, approvals, and sub-case delegation only work with a case.
**Trade-offs:** More wiring — need to start a case via EngineCaseApi alongside the reconciliation loop
**Depends on:** D3 (topology source — need the compiled graph to know what case to create)
**Sources:** application-lifecycle.yaml, EngineCaseApi.startCase(), TopologyCompilerService.deploy()
**Exploration:** quick
**Status:** captured
