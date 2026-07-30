# Vendored skills

These skills are copied from [mattpocock/skills](https://github.com/mattpocock/skills),
at commit `2ab9580` (2026-07-28).

They are vendored into the repo rather than loaded through the plugin
marketplace entry in `.claude/settings.json`, because remote Claude Code
sessions (claude.ai/code, GitHub Actions) clone the repo fresh and do not fetch
plugin marketplaces — project skills under `.claude/skills/` load everywhere.

## What's here

The `engineering/` category in full, plus the two `productivity/` skills the
engineering ones invoke:

| Skill | Source category |
| --- | --- |
| `ask-matt` | engineering |
| `code-review` | engineering |
| `codebase-design` | engineering |
| `diagnosing-bugs` | engineering |
| `domain-modeling` | engineering |
| `grill-with-docs` | engineering |
| `implement` | engineering |
| `improve-codebase-architecture` | engineering |
| `prototype` | engineering |
| `research` | engineering |
| `resolving-merge-conflicts` | engineering |
| `setup-matt-pocock-skills` | engineering |
| `tdd` | engineering |
| `to-spec` | engineering |
| `to-tickets` | engineering |
| `triage` | engineering |
| `wayfinder` | engineering |
| `grilling` | productivity (dependency) |
| `handoff` | productivity (dependency) |

The upstream `agents/openai.yaml` files are OpenAI Codex configuration and are
not copied.

Start with `/ask-matt`, which routes between the rest, or
`/setup-matt-pocock-skills` to configure this repo's issue tracker and label
vocabulary before first use.

## Updating

```sh
git clone --depth 1 https://github.com/mattpocock/skills.git /tmp/mp-skills
for d in /tmp/mp-skills/skills/engineering/*/ \
         /tmp/mp-skills/skills/productivity/{grilling,handoff}/; do
  n=$(basename "$d")
  rm -rf ".claude/skills/$n"
  cp -r "$d" ".claude/skills/$n"
  rm -rf ".claude/skills/$n/agents"
done
```

Then update the commit reference at the top of this file.
