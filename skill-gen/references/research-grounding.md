# Research & Grounding — finding the content the skill will contain

A skill is only as good as the domain knowledge curated into it. Structure (progressive
disclosure, frontmatter) is necessary but not sufficient — the reason `sentry-security-review`
is good is its **content**: per-language vulnerability patterns, OWASP checklists, organized so
the agent loads only what it needs. That content came from research, not pre-training. This step
makes skill-gen go find it.

## When to run (gate it)

Run research when the skill encodes **domain knowledge that must be correct and current**:
a library/framework/API/CLI, a security or compliance domain, a methodology with established
best practice, anything with per-variant (language/platform/framework) detail. **Skip** it for
a thin procedural wrapper with no external knowledge (e.g. "stage and format a commit"). If
unsure, do a quick source scan and decide.

## Source types, in priority order

Highest signal first. Ground in reality before reaching for the open web.

1. **Local codebase (the target repo)** — the real conventions, naming, and **gotchas** the
   skill must respect. `Grep`/`Glob` for actual patterns; read existing docs/PRDs; read sibling
   skills for house style. This is what makes the skill fit *this* project, not a generic one.
   - **Check installed versions first** (`package.json`, lockfiles, `pyproject.toml`…). Pin all
     downstream research to versions actually in use — don't research an API the repo can't run.
2. **context7 (MCP)** — current, version-specific docs for any named library/framework/API/CLI.
   Prefer this over pre-training; your knowledge may be stale. (Resolve the library id, then
   query the exact topic + version.) If context7 isn't available, fall back to official docs via
   web.
3. **Primary sources** — official docs, RFCs/specs, the library's own repo/changelog. These win
   over blogs. Avoid generic blog posts as a primary source.
4. **Recent web (`WebSearch`/`WebFetch`)** — current best practices, benchmarks, deprecations,
   "X vs Y in <year>". Date-bias your queries; libraries and advice go stale.
5. **History & failure modes** — issues, PRs, incident/postmortem notes, regressions. The
   non-obvious "this looks right but breaks because…" facts are the highest-value gotchas.

## Fan out with subagents (when available)

Spawn one `Task` subagent **per dimension** — per language, per framework variant, per
sub-domain, per competing option — each researching independently and returning **structured
findings** (claims + source + confidence). Then synthesize. This is exactly the Sentry pattern:
one reference file per language, each a checklist of patterns to look for.

- **Graceful degradation:** if subagents/`Task` aren't available in the runtime, do the same
  research inline, sequentially. If `WebSearch`/context7 aren't available, say so and rely on
  local sources + flagged pre-training (mark lower confidence).
- Cap the fan-out to the dimensions that matter; log what you deliberately didn't cover.

## Verify before you bake it in (adversarial pass)

Borrowed from deep-research: don't enshrine a single-source claim. For any **non-obvious or
load-bearing** claim, confirm it against a second independent source (or the official doc). If
sources conflict, present both and mark it unresolved rather than picking silently. Tag each
fact: `verified` / `single-source` / `from-pre-training (confirm)`.

## Synthesize into progressively-disclosed references

Turn findings into the skill's `references/`, the Sentry way:
- **One file per domain/variant** (`references/<language>.md`, `references/<topic>.md`), each a
  tight **checklist of what to look for / patterns**, not prose.
- Route every file from `SKILL.md` with an "open when…" reason (see
  `progressive-disclosure.md`). A reference with no router entry is dead weight.
- **Capture provenance.** End each reference (or a `SOURCES.md`) with the sources used and the
  date — so the skill can be re-grounded later when things change.
- Put only what the agent **doesn't already know** — omit generic concepts, keep the
  project/domain-specific nuance (the gotchas).

## Honest synthesis (from the research skill)

- Recommend, don't force. If two options are equivalent for this stack/scope, say so.
- Justify by **project context**, not generic preference.
  - ❌ "JWT é mais moderno." / "Todo mundo usa bcrypt."
  - ✅ "JWT + refresh no DB permite rotação (RFC 9700) sem adicionar Redis, já que Postgres já
    está no stack."

## Coverage / gap check

Before authoring, state: which dimensions were researched, which sources were primary, and
**what's still a gap** (a variant not covered, a claim unverified, a doc not found). Gaps go
into the skill as explicit "(a confirmar)" notes — never silently omitted.

## Output of this step

- A short **research log**: dimensions covered, key findings (with sources + confidence), and
  gaps.
- The drafted `references/` content (grounded, cited) ready for the authoring step.
