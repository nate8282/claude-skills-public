# live-dashboard

A Claude Code / Cowork skill for building multi-panel live dashboards as Claude Artifacts - the kind that re-query MCP connectors on a schedule, persist user state, and survive reloads.

This skill exists because the same handful of failure modes show up every time someone builds one of these dashboards: UI designed before data shapes were probed, date-awareness drift between panels, cache-state confusion, silent refreshes, timezone parsing off-by-one. The skill encodes the working pattern (data first, UI second, explicit contracts up front, pre-change protocol on edits) so each new dashboard doesn't relearn it from scratch.

## What's in here

`SKILL.md` - the full skill, structured around five phases (discovery → contracts → build → pre-change protocol → verification) plus a recurring-traps section.

## Install

### Claude Code

```
mkdir -p ~/.claude/skills/live-dashboard
cp SKILL.md ~/.claude/skills/live-dashboard/
```

Restart Claude Code and it'll pick up the skill automatically.

### Cowork

Drop the `live-dashboard/` folder into your `.claude/skills/` directory (location varies by install - check Settings → Skills, or your `~/.claude/` folder).

## When it triggers

Designed to trigger on prompts like:

- "Build me a daily ops dashboard…"
- "Make a live artifact that pulls from Slack and Linear…"
- "Add a new panel to my dashboard…"
- "My dashboard panels are showing blank…"
- "Create a status page for my team's services…"

It triggers earlier than the user explicitly says "dashboard" - anywhere the work involves a connector-backed artifact that needs to keep showing fresh data over time.

## What it does

When the skill triggers, Claude works through five phases in order:

1. **Discovery** - call each connector once to see the actual response shape before designing any UI. The number-one cause of failed dashboard builds is assuming a connector returns clean JSON when it actually returns markdown text wrapped in a string field.
2. **Contracts** - write down rules for date awareness, cache state, refresh feedback, and storage versioning at the top of the artifact, so they don't drift across panels.
3. **Build with environment awareness** - `minmax(0, 1fr)` for grids, sandbox capability checks before relying on browser APIs, real error blocks instead of silent spinners.
4. **Pre-change protocol on edits** - list every caller, predict side effects, audit the diff. Multi-panel dashboards regress the same date/cache bugs over and over without this.
5. **Verification** - actually load the artifact and confirm data renders before saying done.

## Contributing

Issues and pull requests welcome. Areas that would benefit from more coverage:

- Examples of probe scripts for additional connectors (Jira, Notion, GitHub, custom MCPs)
- A bundled `scripts/` folder with reusable cache/date helpers
- Reference examples of well-structured panel templates

## License

MIT.
