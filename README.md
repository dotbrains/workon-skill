# workon-skill

> [!IMPORTANT]
> **This repo has moved.** The canonical home for the `/workon` skill is now
> [**dotbrains/skills**](https://github.com/dotbrains/skills), alongside the
> rest of the dotbrains agent skills.
>
> Install with:
>
> ```bash
> npx skills@latest add dotbrains/skills
> ```
>
> Or copy the file directly:
>
> ```bash
> mkdir -p ~/.claude/skills/workon
> curl -fsSL https://raw.githubusercontent.com/dotbrains/skills/main/skills/workon/SKILL.md \
>   -o ~/.claude/skills/workon/SKILL.md
> ```
>
> This repository is archived and will not receive further updates. File
> issues and PRs against [dotbrains/skills](https://github.com/dotbrains/skills).

---

Portable `/workon` skill for Linear-driven ticket execution:

1. Create an isolated worktree.
2. Implement and open a PR.
3. Watch PR health in a loop (AI review comments, CI, merge conflicts).
4. Tear down the worktree after merge.

See the canonical [`SKILL.md`](https://github.com/dotbrains/skills/blob/main/skills/workon/SKILL.md) in dotbrains/skills.
