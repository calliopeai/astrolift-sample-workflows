# astrolift-sample-workflows

The **workflow-definitions fixture** (R9) for the Astrolift core-loop validation
campaign — one workflow definition per pattern the platform supports. These are
**declarative definition files, not a running app**: there is no source, no
Dockerfile, and no image to build. Each definition choreographs the
[`astrolift-sample-agent`](../astrolift-sample-agent) (R8) across one or more
stages.

> [!IMPORTANT]
> **Authored now / exercised later.** These definitions are exercised in the
> campaign's **Domain C**, which **DEPENDS on the Workflows-module build
> (#964) — not yet built**. Authoring the fixtures up front is cheap and keeps
> them ready; the *exercising* follows the campaign's gated order
> (Domain A = apps/crons/function/task/statefulset/managed-services,
> Domain B = `sample-agent`, **Domain C = these workflows**). See
> spec [42](../../../astrolift-spec/specs/42-sample-fixtures.md) §5 + §7.

## The format (spec 40 §5.4)

Each `workflows/*.toml` is a **workflow manifest** authored verbatim against the
**spec 40 §5.4** grammar — the canonical declarative representation of a
`WorkflowDefinition` (tier 1). A manifest is one `[workflow]` table plus an
ordered array of `[[stage]]` tables (the array index **is** the stage `order`).

```toml
[workflow]
slug        = "feature-dev"     # WorkflowDefinition.slug
name        = "Feature Dev"
pattern     = "chained"         # WorkflowDefinition.pattern_kind
description = "..."

[[stage]]                       # -> one WorkflowStage (order = array index)
kind       = "agent_dispatch"   # agent_dispatch | human_gate | checkpoint | aggregation
role       = "implementer"      # human-readable role label for the stage
agent      = "sample-agent"     # local agent slug; OMIT for globals (role-only, bound at configure-time)
skills     = ["write-tests", "./skills/foo", "acme/dev-skills/lint@v2"]   # spec 39 skill-ref grammar
on_failure = "retry"            # fail | retry | skip | escalate
timeout    = 600                # seconds
fan_out    = 0                  # 0/omit = none; >0 static; "dynamic" = one child per prior-stage output item

[[stage]]
kind      = "human_gate"
prompt    = "Approve the implementation?"
approvers = ["team:reviewers"]
timeout   = 86400               # 24h default
```

### Stage kinds (spec 40 §2/§5.1)

| kind | purpose | fields used here |
|---|---|---|
| `agent_dispatch` | dispatch an agent run | `role`, `agent`, `skills`, `on_failure`, `timeout`, `fan_out` |
| `human_gate` | pause for a human approve/reject decision (Temporal signal, 24h default) | `prompt`, `approvers`, `timeout` |
| `aggregation` | merge the outputs of the preceding `fan_out` stage | (no agent/skills — merge target is structural, see ambiguities) |
| `checkpoint` | a labeled marker (label only — spec 40 §5.1) | not exercised by these six (no required pattern needs one) |

### `on_failure` routing (spec 40 §2 / §5.1)

`fail` (default — surface and stop) · `retry` · `skip` (proceed without this
stage's output) · `escalate` (raise for intervention; the engine handles an
`escalation_cleared` signal, spec 40 §1).

### `skills` references (spec 39 grammar, reused verbatim by §5.4)

| Form | Meaning |
|---|---|
| `"file-ops"` | built-in catalogue skill (latest bundled) |
| `"pr-review@1.2.0"` | catalogue skill, pinned |
| `"./skills/foo"` | local skill (folder in the agent's repo) |
| `"acme/dev-skills/lint@v2"` | org skill repo (`<alias>/<skill-path>@<ref>`) |

These fixtures reference **catalogue skills by bare name only**. The skills used
across the six definitions: `file-ops`, `data-transform`, `csv-process`,
`export`, `http-request` (spec 39 v0.1 nine) and `codebase-qa`, `web-research`
(Tier-2 catalogue slugs — they resolve once those entries are added; non-fatal
per spec 39 resolution).

## Variant -> pattern map

Mirrors the spec 42 §5 `sample-workflows` variant matrix. Each file is one
`pattern_kind`; each choreographs the `sample-agent`.

| File | `pattern` | Stages (order) | What it demonstrates |
|---|---|---|---|
| [`workflows/single.toml`](workflows/single.toml) | `single` | agent_dispatch | The minimal workflow — one agent dispatch. |
| [`workflows/chained.toml`](workflows/chained.toml) | `chained` | agent_dispatch -> agent_dispatch -> agent_dispatch | Sequential stage I/O chaining (each output feeds the next). |
| [`workflows/fan_out.toml`](workflows/fan_out.toml) | `fan_out` | agent_dispatch (`fan_out=3`) -> aggregation | Static fan-out to N parallel children, then merge. |
| [`workflows/supervisor_worker.toml`](workflows/supervisor_worker.toml) | `supervisor_worker` | agent_dispatch (supervisor) -> agent_dispatch (`fan_out="dynamic"`) -> aggregation | Decompose -> dispatch N workers -> aggregate (the multi-agent showcase). |
| [`workflows/review_loop.toml`](workflows/review_loop.toml) | `review_loop` | agent_dispatch -> human_gate -> agent_dispatch | Work -> human approve/reject gate -> iterate on reject. |
| [`workflows/advisor.toml`](workflows/advisor.toml) | `advisor` | agent_dispatch (read-only) | A single read-only advisory run (mutates nothing). |

All six `pattern_kind` values from spec 40 §2 / spec 42 R9 are covered. Trigger
variants (manual / `WorkflowSchedule` cron / `WorkflowWebhook`) are a property of
the **configured `Workflow`** (tier 2), not of these tier-1 definitions
(spec 40 §2.2/§3) — so they are not expressed in these files.

## How these map to the backend (spec 40)

A `*.toml` here is a **tier-1 `WorkflowDefinition`** (template/shape). At
exercise time (Domain C) it is parsed by `parse_workflow_manifest` (spec 40
§5.4), persisted as a `WorkflowDefinition` + ordered `WorkflowStage` rows, and a
tenant creates a runnable **`Workflow`** (tier 2) from it — binding a concrete
agent per `agent_dispatch` stage and choosing a trigger. `runWorkflow` then
starts the existing `WorkflowDefinitionRunWorkflow` Temporal engine (spec 40 §1,
§3). These fixtures use `agent = "sample-agent"` directly (a local slug), so they
are self-contained samples rather than role-only catalogue globals.

## Spec 40 §5.4 ambiguities (noted, authored conservatively)

Per the build mandate (spec 40 §0: *implement verbatim, report anything
uncovered*), the §5.4 grammar leaves these under-specified for some patterns.
The fixtures are authored to the documented grammar and the choices are flagged:

1. **`aggregation` merge target.** §5.1's builder panel says aggregation picks
   "which prior fan-out stage it merges," but the §5.4 TOML grammar shows **no
   key** for that target (and no worked `aggregation`/`checkpoint` stage
   example). Authored on the documented structural rule — the aggregation stage
   merges the **immediately preceding `fan_out` stage** by order (edges are
   "structural/derived," spec 40 §5.1). Multi-fan-out-then-pick-one would need a
   grammar field that does not yet exist.
2. **`checkpoint` label.** §5.1 says checkpoint is "label only," but §5.4
   defines no `label`/`name` key for a stage. Not exercised by the six required
   patterns, so omitted; flagged for when checkpoint is authored.
3. **`supervisor_worker` has no distinct "supervisor" stage kind.** Modeled as a
   decompose `agent_dispatch` (role `supervisor`) -> `fan_out="dynamic"` workers
   -> `aggregation`, matching the spec 40 §4.1 catalogue description
   ("decompose -> dispatch N workers -> aggregate"). The supervisor semantic is
   carried by `pattern_kind` + the role, not a dedicated kind.
4. **`review_loop` has no loop-back / max-iterations key.** §5.4 exposes no
   per-stage loop target or iteration cap; the iterate-on-reject behavior is a
   property of the `review_loop` `pattern_kind` + the human_gate approve/reject
   branch (spec 40 §5.1), handled by the engine — not expressible as a §5.4
   field.
5. **`advisor` has no "read-only" stage flag.** §5.4 has no mechanism to mark a
   stage read-only or to enforce it. The read-only intent is conveyed by the
   `advisor` `pattern_kind`, an advisory `role`, and read-only `skills` (no
   `git`/`commit-pr`/`export`-write). There is no grammar-level enforcement.
6. **`role` placement.** §5.4 shows `role` only on the `agent_dispatch` example,
   though the model (spec 40 §2.4) adds `role` to `WorkflowStage` generally
   ("optional for org definitions"). Authored with `role` on `agent_dispatch`
   stages only.

## Validating locally

These are plain TOML; any tomllib/`tomli` parser loads them. The authoritative
shape check is the backend `parse_workflow_manifest` (spec 40 §5.4) once #964
lands — or `astro workflow validate <file.toml>` (spec 40 §5.5) thereafter.
