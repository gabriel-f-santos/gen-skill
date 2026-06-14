---
name: skill-gen
description: >
  Generate production-grade Agent Skills through a guided interview. Use whenever the user
  wants to create, scaffold, author, or design a new skill or set of skills — phrases like
  "create a skill", "make a skill", "gera uma skill", "skill para X", "turn this workflow
  into a skill", "skill generator". Interviews the user to decide target runtimes (Claude
  Code, OpenAI Codex, opencode), single vs multiple skills, execution shape, tool/permission
  needs, and progressive-disclosure layout. Produces a portable SKILL.md plus
  references/scripts/assets and per-platform sidecars (Claude allowed-tools, Codex
  openai.yaml), then validates the result. Do not use to merely tweak prose in one existing
  file — use for authoring or scaffolding skills.
allowed-tools: Read Write Edit Bash(python3 *) Bash(ls *) Bash(cat *) Bash(mkdir *)
---

# skill-gen — the skill that generates skills

A guided generator for **portable Agent Skills**. It conducts a short interview, picks the
minimum viable shape, and writes a skill that runs across **Claude Code, OpenAI Codex, and
opencode** — isolating each platform's quirks instead of hard-coding one.

> **Core principle.** The base spec recognizes only five frontmatter fields (`name`,
> `description`, `license`, `compatibility`, `metadata`). Everything else
> (`allowed-tools`, `disable-model-invocation`, Codex `openai.yaml`, opencode
> `permission.skill`) is **platform-specific and lives in a different place per runtime**.
> A portable skill keeps its logic in the body + `references/` + `scripts/`, and treats
> per-platform control as a sidecar — never a dependency.

This `SKILL.md` is a **router**. Load only the reference you need for the current step.

| Open when you need to… | Read |
|---|---|
| know exactly what each runtime supports (dirs, frontmatter, tools, invocation) | `references/platforms.md` |
| decide the portable layout and what breaks where | `references/portability.md` |
| look up every frontmatter field and which platform honors it | `references/frontmatter.md` |
| decide what goes in `SKILL.md` vs `references/`/`scripts/`/`assets/` | `references/progressive-disclosure.md` |
| pick an execution shape and decide single vs multiple skills | `references/archetypes-and-splitting.md` |
| set least-privilege tools and avoid injection/exfil risks | `references/security.md` |
| copy a starting skeleton | `assets/templates/` |

---

## Workflow (follow in order)

### Step 0 — Capture intent from context first

If the current conversation already contains the workflow to capture ("turn this into a
skill"), extract answers from history before asking: the steps taken, tools used,
corrections the user made, input/output formats observed. Only ask what you cannot infer.

### Step 1 — Interview

Ask **one question at a time**, each with your recommended answer, and explore the codebase
instead of asking when the answer is discoverable. Use the `AskUserQuestion` tool when it
fits. Cover these decision axes (full rationale in the linked references):

1. **Purpose & trigger** — What should the skill let the agent do? What user phrases/contexts
   should activate it? What is the expected output? *(The `description` is the trigger — get
   real phrases the user would actually type.)*
2. **Single vs multiple skills** — Is this one cohesive capability, or several with distinct
   triggers/audiences? If triggers diverge, split. → `references/archetypes-and-splitting.md`
3. **Target runtimes** — Claude Code only? + Codex? + opencode? all three? This decides
   install location(s) and which sidecars to emit. → `references/platforms.md`
4. **Execution shape** — pick the simplest adequate one → `references/archetypes-and-splitting.md`:
   - **knowledge** (instructions only)
   - **reference-backed** (heavy domain docs in `references/`)
   - **script-backed** (deterministic automation in `scripts/`)
   - **orchestrator** (drives an external MCP server / API)
5. **Tools & safety** — Read-only, or does it write/run/network? Destructive? Should it be
   barred from auto-invocation? → `references/security.md` + `references/frontmatter.md`
6. **Knowledge volume** — Will the body exceed ~500 lines? If so, plan the
   `references/` split now. → `references/progressive-disclosure.md`

Record the chosen answers as a short spec before writing anything. If the user says "just
vibe with me / skip the interview", collapse to questions 1, 3, and 5 only.

### Step 2 — Decide layout

From the interview, fix:
- the **directory name** (= `name`, 1–64 chars, lowercase, single-hyphen separators);
- the **install path(s)** per target runtime (see `references/portability.md` §placement);
- the **bundled resources** (`references/`, `scripts/`, `assets/`) the shape requires.

Default to `.claude/skills/<name>/` when targeting Claude Code **and** opencode — opencode
reads `.claude/skills/` natively, so one copy covers both. Add a `.codex/skills/<name>/` copy
(or a `config.toml` entry) only when Codex is a target. Use `.agents/skills/<name>/` when the
user wants the broadest cross-client convention.

### Step 3 — Write the portable core

Start from `assets/templates/SKILL.md.tpl`. Rules:
- Frontmatter carries the **five standard fields** + (only if Claude is a target)
  `allowed-tools`. Unknown fields are harmless in opencode/Codex (ignored) but keep them
  minimal. → `references/frontmatter.md`
- Body in **imperative voice**, trigger-rich, explaining the *why* — not heavy-handed MUSTs.
- **No `!`cmd`` load-time execution and no `${CLAUDE_SKILL_DIR}`** unless the skill is
  Claude-only — those don't expand in Codex/opencode. Instead, instruct the agent to *run*
  the command as a workflow step ("run `ls -la` and read the output").
- Keep `SKILL.md` under ~500 lines; route heavy material into `references/` with explicit
  "open when…" pointers. → `references/progressive-disclosure.md`
- Use checklists, tables, and **template anchoring** (embed exact output skeletons) over
  prose.

### Step 4 — Emit per-platform sidecars (only for chosen targets)

- **Claude Code** — `allowed-tools` (and optionally `disable-model-invocation: true`) in the
  frontmatter. Use the scoped syntax `Bash(git add *)` for least privilege.
- **Codex** — write `agents/openai.yaml` from `assets/templates/openai.yaml.tpl`
  (`dependencies`, `allow_implicit_invocation`). Tool/dep control lives **here**, not in
  frontmatter.
- **opencode** — it ignores `allowed-tools`; permission is set in the user's `opencode.json`
  under `permission.skill`. Don't write to their config — instead, if the skill needs
  restriction, emit a short `PORTABILITY.md` telling them the exact `opencode.json` snippet.
- When more than one runtime is targeted, always emit `PORTABILITY.md` listing what to
  configure where (template in `assets/templates/`). → `references/portability.md`

### Step 5 — Validate

Run the bundled checker, then a security pass:

```bash
python3 scripts/validate_skill.py <path-to-generated-skill>
```

It checks: `SKILL.md` exists, valid YAML, `name`/`description` present and within limits,
`name` matches the directory, and flags Claude-only constructs (`!`cmd``,
`${CLAUDE_SKILL_DIR}`) when the skill is declared cross-platform. Then apply
`references/security.md` (least privilege, no secret-prompting, review any `scripts/`). If a
`skill-scanner` skill is available, run it too.

### Step 6 — Report

Return:
1. **Summary** — what was generated and for which runtimes.
2. **Files** — tree of created paths.
3. **Per-platform setup** — the exact steps the user must do in each target (e.g. Codex
   `config.toml` entry, opencode permission snippet).
4. **Validation results** — checker output + any open gaps.

---

## Quick decision shortcuts

- **One runtime, Claude Code, read-only knowledge skill** → single `SKILL.md`, 5 fields +
  `allowed-tools: Read Grep Glob`, no references. Done in one file.
- **All three runtimes** → portable body, no load-time exec, `allowed-tools` for Claude,
  `openai.yaml` for Codex, `PORTABILITY.md` for opencode permissions.
- **Heavy domain knowledge** → lean router `SKILL.md` + `references/<topic>.md`, each with an
  "open when…" entry.
- **Repetitive deterministic step** → bundle a `scripts/` helper so every future run reuses
  it instead of reinventing.
