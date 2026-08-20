# Evidence

The trap table in [`skills/aem-ponytail/SKILL.md`](skills/aem-ponytail/SKILL.md)
isn't hypothesized — every row came from actually running
`aem-ponytail-review` against real, custom AEM code (not vendored/OOTB
code, not archetype boilerplate) and checking whether the findings held up.
Two independent codebases were audited; only two ladder rows needed nothing
new (rungs 1-5 held from the original YAGNI port), and both runs also
surfaced correctly-clean code — the ladder doesn't flag everything, which is
the actual bar for a review tool to be useful rather than noisy.

## Run 1 — an AEM Forms archetype (Adobe Sign integration, headless submission, observability)

**Findings:**
- A mock/demo data servlet returned a fully static, hardcoded JSON payload on
  every GET, ignoring the query parameter callers passed it. → added the
  "static servlet response" trap row (rung 1: this didn't need to be a
  servlet at all).
- A health-check servlet hand-rolled a `{"status": "UP", ...}` response. →
  added the "hand-rolled health endpoint" trap row (rung 4: Apache Felix
  Health Check ships with AEM and already covers this).

**Correctly at the right rung (no finding):** a headless-submit servlet doing
genuine RPC over a `WorkflowSession` (not resource-backed — an exporter
wouldn't apply), two services implementing real AEM Forms extension points
(`FormSubmitActionService`, `DataProvider`) the idiomatic way, a small
dependency-free circuit breaker (justified — AEM has no OSGi-friendly
resilience library bundled), and a webhook receiver with correct signature
validation.

## Run 2 — a small OSGi module (an LLM-cost-tracking dashboard servlet + Sling Model + service)

**Findings:**
- A servlet's `status` action manually re-derived three fields (`live`,
  page count, `model` name) from a backing OSGi service — while a Sling
  Model in the same module already computed the identical shape via
  `@PostConstruct` and getters. Two parallel implementations of the same
  projection, found by reading both files, not by pattern-matching a known
  trap. → added the "servlet re-deriving what a model already computes"
  trap row (rung 2: duplicated capability already in the repo).
- A `lab` action branch was a stub — it didn't run anything, just returned a
  static note saying the real path requires CLI mode. Rung-1 logic applies
  to a dead branch inside a file, not just to whole files.
- **A gap in the ladder itself, not an overbuild finding:** `doPost`
  delegated straight to `doGet`, so an action that calls a paid LLM API and
  reports a dollar cost was reachable via a plain GET with query params — a
  safe/cacheable HTTP method carrying a billed side effect. The ladder as
  written only looks for *too much* code; this is *too little* — a
  guarantee cut to save a few lines. Added as a new "HTTP method safety"
  item under "Never on the chopping block," and as rung 7 in the review
  skill's checklist, since it's the same root cause as every other trap
  (something got cut for convenience) just running in the opposite
  direction.

**Correctly at the right rung (no finding):** the OSGi service wrapping the
LLM client with retry/backoff and a mock/live switch via metatype config —
justified custom code with no platform equivalent.

## What this evidence does and doesn't prove

It proves the ladder discriminates — it found real, specific, defensible
issues in both runs and left real, specific, defensible code alone in both
runs, including issues (the HTTP-method gap) the original table didn't
anticipate. It does not prove the trap table is complete, or that it
generalizes beyond Sling/OSGi-style AEM projects (both audited codebases fit
that shape; a pure HTL/dispatcher-only change, or a headless/GraphQL-only
AEM project, hasn't been tested yet). Treat the table as a growing, unproven
draft that has survived two real reviews — not a finished checklist.
