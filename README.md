# skills

The idea is to maintain skills in this repo an then sync them to `~/.agents/skills/`.

To sync skills run:

```bash
mkdir -p ~/.agents/skills
rsync -a --delete --exclude '.system' skills/ ~/.agents/skills/
```

To create a symlink for claude code:

```bash
rm -rf ~/.claude/skills
ln -s ~/.agents/skills ~/.claude/skills
```

## Sync into the Claude desktop app (Cowork)

This repo also doubles as a Claude plugin marketplace (`.claude-plugin/marketplace.json` + `.claude-plugin/plugin.json`), so Cowork can install the skills directly from GitHub and refresh them on demand — no manual re-upload needed.

One-time setup in Cowork:

1. Customize -> Plugins -> Add marketplace
2. Enter `UrosHCS/skills`
3. Install the `uroshcs-skills` plugin from that marketplace

After editing a `SKILL.md` or adding a new skill folder under `skills/`, commit and push as usual, then click **Update** on the marketplace in Cowork (Customize -> Plugins) to pull the latest version.
