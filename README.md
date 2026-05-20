# Clean Commits Skill

A Claude skill that turns a messy working tree into a series of clean, atomic commits with Conventional Commits messages. Plan-first, user-approved, no destructive operations, never pushes without explicit confirmation.

## What it does

Given a project with uncommitted changes, the skill walks you through 4 phases:

1. **Discovery** — reads `git status`, `git diff`, `git log`, current branch, and `CONTRIBUTING.md` if present
2. **Propose** — shows a numbered plan: which files go into which commit, with proposed Conventional Commits messages
3. **Iterate** — you merge, split, reorder, or rename entries until you approve
4. **Execute** — runs `git add` + `git commit` per entry, verifies staged content against the plan, stops before any push

## Installation

Clone the repo and link the skill into your Claude skills folder.

**macOS / Linux:**
```bash
git clone https://github.com/<your-username>/clean-commits-skill.git
ln -s "$(pwd)/clean-commits-skill/skills/clean-commits" ~/.claude/skills/clean-commits
```

**Windows (PowerShell, run as admin for symlinks):**
```powershell
git clone https://github.com/<your-username>/clean-commits-skill.git
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.claude\skills\clean-commits" -Target "$(Get-Location)\clean-commits-skill\skills\clean-commits"
```

Alternatively, just copy the `skills/clean-commits/` directory into `~/.claude/skills/`.

## Usage

In a Claude Code session, with a project that has uncommitted changes:

> "Use the clean-commits skill to commit these changes."

Claude runs through the 4 phases and asks for your approval before any `git commit`.

## Safety guarantees

The skill will **never**:

- Use `--no-verify`, `--force`, `reset --hard`, or `branch -D`
- Run `git commit --amend` on its own initiative — amend requires a fresh, explicit user request ("amend the last commit", "fold this into HEAD"); a generic "commit my changes" is not consent
- Push to remote without an explicit, separate confirmation (even if you said "and push" in the original prompt)
- Commit on a protected branch (`main` / `master` / `develop` / `release/*`) without asking first
- Append `Co-Authored-By: Claude`, "Generated with Claude Code", or any other AI/Claude/Anthropic attribution to commits — the user is the author; this overrides Claude Code's default behavior of injecting such trailers
- Modify your working tree — only `git add`, `commit`, and `reset --mixed`

## Testing the skill

Manual scenarios to run before releasing a change. Create a throwaway git repo, set up the state described in "Setup", invoke the skill, and verify the behaviour.

| # | Setup | Expected behaviour |
|---|---|---|
| 1 | Clean repo, no changes | Skill says "no changes detected", stops |
| 2 | 1 file changed, single logical change | Proposes 1 commit with a valid CC message |
| 3 | 3 files: new feature + bugfix + docs | Proposes 3 commits with different types |
| 4 | 1 file mixing a fix + refactor | Flags the conflict, proposes hunk-split |
| 5 | Untracked `.env.local` + new code | Flags the secret, asks about adding to `.gitignore` |
| 6 | User on `main` branch | Asks for confirmation before proposing the plan |
| 7 | Mid-merge state (artificial conflict) | Refuses, asks user to finish the merge first |
| 8 | User says "and push it" at the end | Commits, stops at push, asks for separate confirmation |
| 9 | Pre-commit hook fails (e.g. linter exits non-zero) | Does NOT use `--no-verify`, shows hook output, asks |

## License

MIT — see [LICENSE](LICENSE).
