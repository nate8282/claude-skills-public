# claude-skills

A small collection of [Claude](https://claude.ai/) skills I've written and want to share. Practical, opinionated, and shaped by real bugs from real builds.

## What's a Claude skill?

A skill is a folder containing a `SKILL.md` (plus optional reference files / scripts) that Claude reads when the context matches the skill's description. It's how you teach Claude project-specific patterns, conventions, and gotchas without having to repeat them in every prompt.

Skills work in Claude Code, Claude Cowork, and other Claude products that support the skills system.

## Skills in this repo

| Skill | What it does |
|-------|--------------|
| [`live-dashboard/`](./live-dashboard/) | Build a live multi-panel dashboard as a Claude Artifact: one that re-queries connectors on a schedule, persists state, and survives reloads. Enforces a discovery-first workflow (probe data shapes before designing UI) so you don't ship a dashboard that looks polished and renders nothing. |

More to come.

## Installing a skill

### Claude Code

```bash
mkdir -p ~/.claude/skills
cp -R live-dashboard ~/.claude/skills/
```

Restart Claude Code and it'll pick the skill up automatically.

### Cowork

Drop the skill folder into your Cowork skills directory (path varies by install; check Settings → Skills, or your `~/.claude/` folder).

Each skill folder also has its own `README.md` with a more detailed install + trigger guide.

## Contributing

If you spot a bug in a skill, have a real-world example that would sharpen one, or want to add a related skill of your own, issues and pull requests are welcome.

## License

MIT. See [LICENSE](./LICENSE).
