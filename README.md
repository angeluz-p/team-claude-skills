# team-claude-skills

Shared Claude Code skills for the team. Drop any skill here under `skills/<name>/SKILL.md` and teammates pick it up on the next `git pull`.

## Install

Assumes you already have Claude Code, `gh` (logged in), and optionally `codex`.

```bash
git clone <this-repo> ~/team-claude-skills
ln -s ~/team-claude-skills/skills/pr-reviewer ~/.claude/skills/pr-reviewer
```

On Windows (PowerShell, admin):

```powershell
git clone <this-repo> $HOME\team-claude-skills
New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\pr-reviewer" -Target "$HOME\team-claude-skills\skills\pr-reviewer"
```

Or just copy if symlinks are a hassle:

```bash
cp -r ~/team-claude-skills/skills/pr-reviewer ~/.claude/skills/
```

## Update

```bash
cd ~/team-claude-skills && git pull
```

Symlink users: done. Copy users: re-run the `cp`.

## Skills

- **pr-reviewer** — `/pr-reviewer <number>` reviews a GitHub PR with Claude, optionally adds a Codex second opinion, and posts to GitHub only after your explicit approval. See `skills/pr-reviewer/SKILL.md`.

_Add new skills by committing `skills/<name>/SKILL.md` and a line here._

## Adding a new skill

1. Drop the skill at `skills/<name>/SKILL.md` in this repo.
2. Add a bullet under **Skills** above describing what it does.
3. Commit and push. Symlink users get it on next `git pull`; copy users re-run the `cp`.

