---
name: add-persona
description: Author or select the agent's personality — interview the user from scratch, or copy from a prebuilt persona under .templates/.
user_invocable: true
---

# /add-persona — Author the Agent's Personality

Produces `workspace/personality.md` (and optionally seeds `workspace/skills/`). Two modes: **interview** (build from scratch) or **template** (copy from `.templates/<name>/`).

This skill is normally invoked by `/setup-nanopilot`, but can be run standalone to re-author or replace a persona.

## Pre-flight

```bash
ls .templates/ 2>/dev/null               # what prebuilt personas are available?
cat workspace/personality.md 2>/dev/null # is there already a persona?
```

If `workspace/personality.md` already exists and is non-stub (more than just `{{DISPLAY_NAME}}` placeholders), STOP and ask the user:

> A persona already exists at `workspace/personality.md`. Do you want to:
> 1. Keep it (skip this skill)
> 2. Replace it (overwrites the file)
> 3. Show me what's there first

Do not silently overwrite an existing persona.

## Mode Selection

If the user (or `/setup-nanopilot`) already specified a mode, skip the question. Otherwise ask:

> How do you want to set up the persona?
> 1. **Interview** — I'll ask you about role, tone, and scope
> 2. **Template** — pick from prebuilt personas (Spark, ProxMan, etc.)
> 3. **Bring your own** — you have a personality.md ready, just point me at it

## Mode A: Interview

Ask these in a single message — collecting all answers at once is faster than one question at a time:

1. **Name and one-line description.** (e.g., "Spark — homelab infrastructure ops")
2. **Primary role.** What's the agent's main job? (one sentence)
3. **Tone.** Pick or describe: terse / friendly / formal / playful / dry / professional / blunt
4. **Scope — what it does.** 3–5 bullets of the kinds of asks it should handle.
5. **Scope — what it doesn't do.** 1–3 bullets of things it should refuse or punt on.
6. **Hard rules / safety.** Anything it must always or never do? (optional)
7. **Skills.** Any specific capabilities it needs (file ops, SSH, Proxmox, web search, etc.)? (optional — informs which other sub-skills should run)

Synthesize into `workspace/personality.md` using this structure:

```markdown
# <Name>

You are <Name>, <one-line description>. Powered by NanoPilot.

## Role

<primary role expanded into 2–3 sentences>

## Tone

<tone description, with one example phrasing if useful>

## What you do

- <bullet>
- <bullet>
- <bullet>

## What you don't do

- <bullet>
- <bullet>

## Rules

- <hard rule>
- <hard rule>
```

Show the draft to the user before writing it. Accept edits before committing the file.

## Mode B: Template

Run `ls .templates/`. If empty or missing:

> No prebuilt personas are available in `.templates/`. Switching to interview mode.

→ Go to Mode A.

If templates are available, list them with their one-line descriptions and
surface any safety warnings (lines starting with `WARNING:` or wrapped in
`**WARNING: …**`) so the user sees them before picking:

```bash
for dir in .templates/*/; do
  name=$(basename "$dir")
  desc=$(head -3 "$dir/personality.md" 2>/dev/null | tail -1 || echo "(no description)")
  warn=$(grep -m1 -E '^(\*\*)?WARNING:' "$dir/personality.md" 2>/dev/null | sed 's/\*\*//g')
  if [ -n "$warn" ]; then
    echo "  $name — $desc"
    echo "    ⚠ $warn"
  else
    echo "  $name — $desc"
  fi
done
```

If the user picks a template that surfaces a WARNING, **read the warning back
to them and ask for explicit confirmation** before copying. Don't silently
accept "naughty" or any other testing/risky persona.

Then:

```bash
# Copy persona
cp .templates/<name>/personality.md workspace/personality.md

# Copy any bundled skills (don't clobber user-authored skills)
mkdir -p workspace/skills
for skill in .templates/<name>/skills/*.md; do
  base=$(basename "$skill")
  if [ ! -f "workspace/skills/$base" ]; then
    cp "$skill" "workspace/skills/$base"
  else
    echo "Skipping $base — already exists in workspace/skills/"
  fi
done

# Copy any bundled rules
if [ -f ".templates/<name>/rules.md" ] && [ ! -f "workspace/rules.md" ]; then
  cp ".templates/<name>/rules.md" "workspace/rules.md"
fi

# (Per-agent bulkhead-policy.json was retired 2026-04-26 — the runtime
# selects policy by name via NANOPILOT_BULKHEAD_POLICY=strict|moderate. A
# per-agent policy file loader is tracked separately in issue #55.)
```

Show what was copied. Ask the user if they want to customize the copied persona before moving on (they'll often want to tweak tone or scope to fit their environment).

## Mode C: Bring Your Own

Ask:

> Where is your personality.md? (path or paste contents)

If a path: `cp <path> workspace/personality.md`.
If pasted contents: write directly to `workspace/personality.md`.

Read it back and confirm it looks right.

## Post-write

After any mode:

1. **Sanity check the file**:
   ```bash
   wc -l workspace/personality.md   # expect 10–80 lines for a sensible persona
   grep -c '{{' workspace/personality.md  # any unresolved placeholders?
   ```
   If unresolved `{{PLACEHOLDER}}` tokens remain, ask the user to fill them in.

2. **Update `agent.json` displayName** (if `agent.json` exists in the workspace):
   ```bash
   # Use jq if available, else manual edit
   jq --arg name "<Name>" '.displayName = $name' agent.json > /tmp/agent.json && mv /tmp/agent.json agent.json
   ```

3. **Report**: tell the user what was written, where, and how many lines/skills.

## Outputs (for the orchestrator)

When invoked by `/setup-nanopilot`, return:
- `persona_name`: the agent's display name
- `persona_source`: `interview | template:<name> | byo`
- `skills_added`: list of skill files copied (if template mode)

## Notes for Template Authors

If you want to seed a new prebuilt persona, drop it into `.templates/<name>/`:

```
.templates/<name>/
├── personality.md        # required — the persona itself
├── rules.md              # optional — hard rules for /reflect
└── skills/               # optional — pre-bundled skills
    └── *.md
```

(Note: `bulkhead-policy.json` was retired 2026-04-26 — runtime policy is
selected by name via `NANOPILOT_BULKHEAD_POLICY`. See issue #55 for the
future per-agent policy-file loader.)

The first line of `personality.md` should be `# <Name>` and the third line should be a one-line description (this is what gets shown in the picker).
