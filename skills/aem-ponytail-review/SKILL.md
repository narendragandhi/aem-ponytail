---
name: aem-ponytail-review
description: Audits existing AEM code (a diff, a component, a bundle) against the aem-ponytail ladder to find places where custom Java/HTL/OSGi was written for something Core Components, Sling, or an installed dependency already provides. Use after implementing an AEM feature, or when asked to review AEM code for over-engineering.
---

# AEM Ponytail Review

Runs [aem-ponytail](../aem-ponytail/SKILL.md)'s ladder backwards: instead of
guiding new code, it inspects code that already exists and flags what a lazier
senior AEM architect would have deleted.

## When to use

- Right after implementing an AEM component, service, or workflow — before
  opening the PR.
- When asked to review an AEM diff, component, or bundle for bloat.
- When a bundle or `ui.apps` tree feels bigger than the feature it implements.

## What to look for

For each new/changed file, ask which rung of the ladder it actually needed:

1. **Unnecessary existence** — a component, service, or config for something
   an author could do with existing dialog fields or page properties.
2. **Duplicated capability** — a new Sling Model, utility class, or clientlib
   that re-implements something already in this repo (another model, ACS AEM
   Commons, Core WCM Components).
3. **Reinvented platform** — hand-rolled JCR query, caching, scheduling, or
   config-reading code where Sling/OSGi/JCR already has an API for it.
4. **Servlet instead of exporter** — a custom `SlingSafeMethodsServlet`
   returning JSON that a `.model.json` Sling Model Exporter would cover.
5. **Imperative validation instead of declarative** — Java-side field checks
   that `granite:required`/`foundation-validation` already express in the
   dialog.
6. **New template where a reference would do** — an HTL file re-authoring
   markup that `data-sly-resource` against an existing resource type already
   renders.
7. **Method safety collapsed for convenience** — `doPost`/`doPut` delegating
   straight to `doGet` (or vice versa) to save code, when the POST/PUT path
   has a side effect — a write, a workflow start, a billed API call — that
   shouldn't be reachable via a safe/cacheable method. This is the opposite
   direction from the other six (a missing guarantee, not extra code) but
   the same root cause: something got cut to save typing that shouldn't
   have been.

## What NOT to flag

Never suggest cutting: XSS-safe HTL context attributes, ACL/permission
checks, dispatcher cache-safety logic, accessibility attributes/behavior, or
replication/migration safety code. Findings that would remove these are
false positives — drop them even if they technically shrink the diff.

## Output format

Report each finding as: file/location → what's there → what rung of the
ladder it should have stopped at → the specific OOTB/platform/dependency
capability that already covers it. Skip files where the code already sits at
the correct rung — a clean file needs no entry.
