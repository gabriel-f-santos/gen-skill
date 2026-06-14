# gen-skill

**A skill that generates skills.** `skill-gen` is a meta-[Agent Skill](https://agentskills.io)
that interviews you and scaffolds a new, **portable** skill that runs across **Claude Code**,
**OpenAI Codex**, and **opencode** — isolating each runtime's quirks instead of hard-coding
one.

It's built the way it teaches: a lean router `SKILL.md` with progressive-disclosure
references, bundled templates, and a validation script.

## Why

The Agent Skills standard is shared, but each runtime diverges on frontmatter, tool
permissions, discovery paths, and dynamic-context syntax. A skill written naively for one
tool silently misbehaves on the others. `skill-gen` encodes those differences so the skill it
produces is portable by construction — and tells you exactly what to configure where.

## What it does

Through a short interview it decides:

- **Purpose & trigger** — a sharp, "pushy" `description` matched on real user phrasing.
- **Single vs multiple skills** — splits when triggers, audiences, lifecycle, or safety
  posture diverge.
- **Target runtimes** — Claude Code / Codex / opencode / all, picking install path(s).
- **Execution shape** — knowledge · reference-backed · script-backed · external orchestrator.
- **Tools & safety** — least privilege per runtime, no secret-prompting, no config poisoning.
- **Progressive disclosure** — keeps `SKILL.md` lean, routing heavy detail into `references/`.

Then it writes the portable core plus per-platform sidecars (Claude `allowed-tools`, Codex
`agents/openai.yaml`, an opencode `permission.skill` note in `PORTABILITY.md`) and runs a
validation + security pass.

## Install

Copy the skill into any runtime's skills directory (the folder name must stay `skill-gen`):

```bash
# Claude Code (also discovered by opencode, which scans .claude/skills/)
cp -r skill-gen ~/.claude/skills/

# or project-local
cp -r skill-gen <your-project>/.claude/skills/

# OpenAI Codex
cp -r skill-gen ~/.codex/skills/
```

Then ask your agent to *"create a skill for …"* / *"gera uma skill pra …"*, or invoke it
explicitly (`/skill-gen` in Claude Code/opencode, `$skill-gen` in Codex).

## Layout

```
skill-gen/
├── SKILL.md                         # router + 6-step interview workflow
├── references/
│   ├── platforms.md                 # Claude Code · Codex · opencode (dirs, frontmatter, tools)
│   ├── portability.md               # what travels / breaks, placement matrix
│   ├── frontmatter.md               # every field + which runtime honors it
│   ├── progressive-disclosure.md    # 3-tier loading, when to split
│   ├── archetypes-and-splitting.md  # execution shapes, 4 archetypes, single vs many
│   └── security.md                  # least privilege, injection/exfil, gotchas
├── assets/templates/                # SKILL.md, openai.yaml, PORTABILITY.md skeletons
└── scripts/
    └── validate_skill.py            # structure + Claude-only-construct checks (stdlib only)
```

## Validate any skill

```bash
python3 skill-gen/scripts/validate_skill.py <path-to-a-skill> [--cross-platform]
```

Checks YAML validity, required fields, `name` ↔ directory match, size limits, broken
reference links, and (in `--cross-platform` mode) flags Claude-only constructs that won't work
in Codex/opencode.

## Platform support (verified June 2026)

| | Claude Code | OpenAI Codex | opencode |
|---|---|---|---|
| Skill dir | `~/.claude/skills/` | `~/.codex/skills/` | `.opencode/`, **`.claude/`**, `.agents/` |
| Tool control | `allowed-tools` (frontmatter) | `dependencies` (`openai.yaml`) | `permission.skill` (`opencode.json`) |
| Block auto-invoke | `disable-model-invocation` | `allow_implicit_invocation: false` | `"deny"`/`"ask"` |
| Explicit call | `/name` | `$name` | `/name` |

## License

[MIT](./LICENSE) © 2026 Gabriel Figueiredo
