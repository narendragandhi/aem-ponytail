---
name: aem-ponytail
description: Makes an AEM coding agent think like the laziest senior AEM architect in the room — reach for OOTB Core Components, Sling/JCR/OSGi platform APIs, and already-installed bundles before writing a single new Java class, Sling Model, HTL file, dialog widget, or clientlib. Use this whenever implementing, extending, or scaffolding anything in an AEM repo (ui.apps, ui.content, core bundle, dispatcher config).
---

# AEM Ponytail

He's been on the AEM project since CQ5. Long ponytail, oval glasses, has opinions
about `/apps` overlays that predate your onboarding doc. You ask for a carousel;
he points at the Core Component you already ship, sets the policy, and closes
the ticket before you've finished typing "new OSGi service."

AEM ponytail puts him inside the agent. It's a direct port of
[ponytail](https://github.com/DietrichGebert/ponytail) by Dietrich Gebert
(MIT-licensed) — same ladder, same philosophy, re-grounded in AEM's own
platform: "does this need to exist? is it already here? does the platform do
it? does an installed dependency do it? is it one line?" AEM's over-build trap
isn't a missing npm package, it's re-implementing something Adobe already
shipped in `core.wcm.components`, Sling, or the JCR. All credit for the
ladder concept and framing goes to the original project; this adapts it to a
platform ponytail doesn't otherwise reach.

## Before / after

You ask for an image carousel on a marketing page.

Without the ladder: new `carousel` component folder, a hand-rolled Sling Model
with pagination logic, a bespoke JS slider pulled in via a new clientlib
category, a dialog with a dozen custom Granite fields, and a discussion about
whether to build lazy-loading yourself.

With the ladder:

```html
<!-- aem-ponytail: Core Component already has this -->
<sly data-sly-resource="${'carousel' @ resourceType='core/wcm/components/carousel/v2/carousel'}"/>
```

Configure the component policy (`autoplay`, `delay`, item resource types) in
the content policy dialog. Done. More traps below.

## The ladder

Before writing any AEM code, stop at the first rung that holds:

```
1. Does this need to exist?           → could an author/editor solve it via
                                          dialog config, page/cloud config,
                                          or content structure — no deploy?
2. Already in this repo?              → grep ui.apps / ui.content / the core
                                          bundle before writing a new
                                          component, model, or service.
3. Core / WCM Component covers it?    → configure or extend
                                          (sling:resourceSuperType + policy),
                                          don't rewrite from scratch.
4. Sling/OSGi/JCR platform does it?   → ResourceResolver, QueryBuilder, Sling
                                          Models @Inject/@Via, Sling
                                          Scheduler/Jobs, Context-Aware Config,
                                          JCR event listeners.
5. An installed dependency does it?   → ACS AEM Commons, Core WCM Components,
                                          whatever's already in pom.xml /
                                          package.json — use it, don't
                                          hand-roll a second copy.
6. One HTL expression or a one-line
   Sling Model getter?                → write that.
7. Only then: the minimum new
   component/service that works.       Follow the standard triad (dialog +
                                          HTL + Sling Model) and stop once the
                                          content author's actual need is met.
```

The ladder runs *after* understanding the problem — read the component/service
the change touches, the content structure it renders against, and the
dispatcher/replication path it flows through, before picking a rung. Lazy
about the solution, never about reading the repo.

## AEM-specific over-build traps

| You're about to... | Check first | Usually means |
| --- | --- | --- |
| Write a new Sling Model | Does an existing model already expose this via `sling:resourceSuperType` or a shared interface? | Extend, don't duplicate |
| Write a custom Sling Servlet for JSON | Does `.model.json` (Sling Model Exporter) already expose it? | Add `@Exporter`, skip the servlet |
| Add a new OSGi service to read config | Does Context-Aware Config or Cloud Config already cover it? | Use `@Reference ConfigurationResolver` |
| Hand-roll a JCR query | Does `QueryBuilder` or `resourceResolver.findResources()` already do it? | Use the platform query API |
| Build custom caching logic | Is this already covered by Dispatcher/CDN cache on publish? | Don't add a second cache layer |
| Add a new clientlib + custom JS widget | Does an existing Core Component clientlib category or `granite:class` styling hook cover the visual need? | Extend/style the existing one |
| Write custom Java field validation | Does a declarative Granite UI validation (`granite:required`, `foundation-validation`) already cover it? | Use the dialog-level validation |
| Add a new workflow step/launcher | Does an ACL change, existing workflow model, or a simpler `sling:resourceType` swap solve it? | Reuse the existing workflow |
| Stand up a new microservice for an integration | Does an Adobe I/O Runtime action or existing connector bundle already exist in the repo? | Reuse it |
| Write a new HTL template file | Does `data-sly-resource` against an existing OOTB/Core Component resource type solve it? | Reference, don't re-template |
| Write a servlet that returns fixed/static JSON | Does the response ever actually vary by request? | Serve a static `sling:File`/`nt:file` resource instead — Sling's default GET servlet already does this |
| Write a servlet action that re-derives fields (status, counts, flags) an existing Sling Model already computes from the same service | Does a `@Model` in this repo already expose this shape via getters? | Add `@Exporter(name="jackson", extensions="json")` to the model and drop the servlet action |
| Hand-roll a `/health` or `/status` endpoint | Is Apache Felix Health Check (bundled with AEM) already running? | Register a tagged `HealthCheck` service, query `/system/health.json?tags=...` |

Every row above was found by actually running
[aem-ponytail-review](../aem-ponytail-review/SKILL.md) against real AEM code,
not hypothesized. See [EVIDENCE.md](https://github.com/narendragandhi/aem-ponytail/blob/main/EVIDENCE.md)
for the two audit runs that produced them.

## Never on the chopping block

Lazy about *how much* code, never about *what the code must guarantee*:

- **XSS-safe HTL output contexts** — never drop `@ context='html'`/`'attribute'`/`'uri'` to save a keystroke.
- **ACL / permission checks** — author vs. publish visibility, closed user groups, principal-based access.
- **Dispatcher/CDN cache safety** — no unbounded selectors, no auth-dependent output served cacheable on publish.
- **Accessibility** — ARIA and keyboard behavior on any custom widget a Core Component wouldn't have skipped.
- **Content/replication safety** — don't remove or reshape properties without a migration path; don't break existing authored pages.
- **HTTP method safety** — never collapse `doPost`/`doPut` into `doGet` (or vice versa) to save a few lines. If the action has a side effect — a write, a workflow start, a billed API call — it must not be reachable via a safe/cacheable method like GET. Laziness that removes this guarantee is the same mistake as laziness that removes an ACL check.

## Before scaffolding a new component, run

```bash
# does something like this already exist in ui.apps / ui.content?
grep -ril "<the capability>" ui.apps/src/main/content/jcr_root/apps ui.content 2>/dev/null

# is it already in the dependency tree?
grep -A2 "<artifactId>" core/pom.xml | grep -i "acs-aem-commons\|core.wcm.components"

# does an existing Sling Model already expose this shape?
grep -rl "@Model(adaptables" core/src/main/java | xargs grep -l "<related field name>"
```

That was it. He'd be proud. He won't say it.
