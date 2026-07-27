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
