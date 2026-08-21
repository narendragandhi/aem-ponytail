# AEM Ponytail

An AEM port of [ponytail](https://github.com/DietrichGebert/ponytail) —
same ladder ("does this need to exist? is it already here? does the
platform do it? does an installed dependency do it? is it one line?"), same
"laziest senior dev in the room" philosophy, re-grounded in AEM's own
platform. All credit for the ladder concept and framing goes to Dietrich
Gebert's original project (MIT-licensed); this adapts it to a stack ponytail
doesn't otherwise reach.

AEM's over-build trap isn't a missing npm package — it's re-implementing
something Adobe already shipped in Core Components, Sling, or the JCR. This
gives an AI coding agent a specific, checkable ladder to stop at before it
writes a new Sling Model, servlet, OSGi service, or component when the
platform already has the answer.

## Before / after

You ask for an image carousel on a marketing page.

Without the ladder: a new `carousel` component folder, a hand-rolled Sling
Model with pagination logic, a bespoke JS slider pulled in via a new
clientlib category, a dialog with a dozen custom Granite fields.

With the ladder:

```html
<!-- aem-ponytail: Core Component already has this -->
<sly data-sly-resource="${'carousel' @ resourceType='core/wcm/components/carousel/v2/carousel'}"/>
```

Configure the component policy (`autoplay`, `delay`, item resource types) in
the content policy dialog. Done.

## Traps, caught live

These aren't made up. Each one is a real finding from an actual AEM codebase
— see [EVIDENCE.md](EVIDENCE.md) for the unabridged version.

**The servlet that returns the same JSON forever.** Someone needed mock data
for a demo. What exists: an `@Component`, a `sling.servlet.paths` binding, a
`Jackson ObjectMapper`, two little builder methods — a whole deployable OSGi
bundle — to return John Doe's identical $5,500 salary deposit and $120.50
Amazon charge, on every request, forever. The `customerId` param you can
pass it? Read, and ignored.

```json
// aem-ponytail: this JSON never changes. it doesn't need a bundle, it needs a file.
{ "customer": { "id": "CUST-10293", "name": "John Doe" }, "transactions": [ "..." ] }
```

Put it at `/content/.../mock-finance-data.json` as a plain `nt:file`. Sling
already serves static JSON for free. Delete the servlet, the `@Component`,
and the redeploy that comes with changing a fixture.

**The status nobody agreed to write twice.** A servlet's `status` action and
a Sling Model, in the same module, both ask the same OSGi service "are you
live, how many pages, what model" — and both write their own copy of the
answer. Nobody decided this; it just happened, twice.

```java
// aem-ponytail: this method and CostOptimizationModel.getStatus() are the same question, asked twice
private void writeStatus(ObjectNode result) {
    result.put("live", anthropicService.isLive());
    result.put("evalPages", ...);
    result.put("model", "claude-opus-5");
}
```

The model already has these as getters. Add `@Exporter(name="jackson",
extensions="json")` to it, point at `dashboard.model.json`, delete the
method that was asking the same question a second time.

**The GET request that spends money.** `doPost` called `doGet` to save two
lines. Which means a plain, cacheable, prefetchable GET — the kind a bot, a
browser prefetcher, or an overzealous dispatcher rule fires without asking
anyone — now triggers a real call to a paid LLM API and hands back a dollar
figure.

```java
// aem-ponytail: this line means a GET request can spend real money
protected void doPost(SlingHttpServletRequest req, SlingHttpServletResponse resp) throws IOException {
    doGet(req, resp); // <- right here
}
```

That's the one kind of laziness the ladder doesn't allow. Write the POST
handler.

## What's in here

- **[`skills/aem-ponytail/SKILL.md`](skills/aem-ponytail/SKILL.md)** — the
  ladder, an AEM-specific over-build trap table, and a "never on the
  chopping block" list (XSS-safe HTL contexts, ACL checks, dispatcher cache
  safety, accessibility, replication safety, HTTP method safety). Load this
  before writing AEM code.
- **[`skills/aem-ponytail-review/SKILL.md`](skills/aem-ponytail-review/SKILL.md)**
  — runs the ladder backwards: audits existing AEM code/diffs and flags
  what a lazier senior architect would have deleted.
- **[`EVIDENCE.md`](EVIDENCE.md)** — the trap table isn't hypothesized. Both
  skills were run against two real, custom AEM codebases before this was
  written up; this file has the specific findings, including one gap
  (HTTP method safety) the ladder didn't originally anticipate and that was
  added as a direct result of the audit.

## Relationship to Adobe's official skills

Adobe publishes its own agent skills at
[adobe/skills](https://github.com/adobe/skills)
(`plugins/aem/cloud-service/skills/`), including `create-component` and
`code-assessment`. This isn't a competitor to either — it covers a gap
between them.

- **`create-component`** scaffolds AEM components and already checks
  whether a Core Component can be extended (a Tier 1/2/3 project → Core
  Component → ask-user decision table), which overlaps with ladder rung 3.
  But its own documented workflow always creates a Sling Model, a unit
  test, and a dedicated clientlib — Step 3 literally says "Component
  Clientlib — Create for Every Component" — with no earlier gate for
  "does this need a new component at all" or "is a one-line HTL expression
  enough." Ponytail's rungs 1, 2, and 6 are exactly the check that's
  missing before `create-component`'s workflow starts.
- **`code-assessment`** audits AEM code, but for a different problem
  entirely: deprecated APIs, unbounded queries, missing HTTP timeouts,
  `@Inject` modernization, scheduler/replication/event-listener migration.
  None of its pattern rows are about custom code written for something
  Core Components/Sling/an installed dependency already provides — that's
  `aem-ponytail-review`'s whole job.

Practical read: run `aem-ponytail` as the gate before `create-component`
scaffolds anything, and treat `aem-ponytail-review` as covering the
over-engineering pattern `code-assessment` doesn't check for.

## Install

**Claude Code** (verified — this is the only integration actually tested so
far):

```bash
mkdir -p .claude/skills
cp -r skills/aem-ponytail .claude/skills/
cp -r skills/aem-ponytail-review .claude/skills/
```

Claude Code auto-discovers `SKILL.md` files under `.claude/skills/<name>/`
and loads them when a task matches the skill's description — no
restart-and-hope, no lifecycle hooks required.

**Other agents** (Cursor, Codex, Windsurf, Gemini CLI, etc.): the SKILL.md
files are plain markdown with no Claude-specific syntax, so copying the
ladder content into whatever convention your agent reads (`AGENTS.md`,
`.cursor/rules/`, etc.) should work the same way ponytail itself supports 20
platforms — but that portability hasn't been verified here the way the
Claude Code path has. If you try it on another agent, an issue reporting
what worked (or didn't) is useful signal for whoever picks this up next.

## Status

Early. Validated by:
- Running the review skill against two real, custom-code AEM projects (see
  [EVIDENCE.md](EVIDENCE.md)) and confirming findings were specific and
  defensible, not generic noise.
- Confirming the skill doesn't flag correctly-written code that uses real
  AEM extension points (`FormSubmitActionService`, `DataProvider`,
  `@Exporter`) the idiomatic way.

Not yet validated:
- Generalization beyond Sling/OSGi-shaped projects (a pure HTL/dispatcher
  change or a headless/GraphQL-only AEM project hasn't been tested).
- Any platform other than Claude Code.
- Community usage — this is a first release, not a mature tool.

## Community project

This is not affiliated with, endorsed by, or maintained by Adobe. AEM,
Sling, and Core Components are Adobe/Apache projects referenced here for
compatibility, not ownership.

## License

MIT. See [LICENSE](LICENSE). The ladder concept and original ponytail
project are © Dietrich Gebert, also MIT-licensed — see
[github.com/DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail).
