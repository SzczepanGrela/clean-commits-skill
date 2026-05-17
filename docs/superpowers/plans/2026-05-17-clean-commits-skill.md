# Clean Commits Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude skill (Markdown docs only) that walks a user through turning uncommitted changes into a series of clean atomic commits with Conventional Commits messages, distributed as a GitHub repo.

**Architecture:** Single `SKILL.md` containing the entire skill (Safety rules + 4-phase workflow + edge cases), accompanied by `README.md` (install + manual test scenarios), `LICENSE` (MIT), and `.gitignore`. No scripts, no references, no CI.

**Tech Stack:** Markdown. Git. That's it.

**Spec:** `docs/superpowers/specs/2026-05-17-clean-commits-skill-design.md`

**Note on TDD:** This project is documentation. There are no unit tests to write red-then-green. Verification is the manual scenario suite in Task 5 — run it before declaring done.

---

## File Structure

| Path | Purpose | Created in |
|---|---|---|
| `LICENSE` | MIT license text | Task 1 |
| `.gitignore` | Ignore IDE/OS artefacts | Task 1 |
| `README.md` | GitHub-facing: what, install, usage, safety, test scenarios | Task 2 |
| `skills/clean-commits/SKILL.md` | The skill itself — frontmatter + safety + 4 phases + edge cases | Task 3 |

The `docs/superpowers/` tree (specs + this plan) stays in the repo but is excluded from skill installation — it's project meta, not skill content.

---

## Task 1: Initialize repo, LICENSE, .gitignore, first commit

**Files:**
- Create: `LICENSE`
- Create: `.gitignore`

- [ ] **Step 1: Initialize git repo**

Run from the project root (`C:\Users\Szcze\PycharmProjects\clean-commits-skill`):

```bash
git init
git branch -m main
```

Expected output:
```
Initialized empty Git repository in .../clean-commits-skill/.git/
```

- [ ] **Step 2: Verify git config has user.name and user.email**

```bash
git config user.name
git config user.email
```

If either is empty, set them (use the user's actual identity — for this project: `moderacja@smakosz.xyz`):

```bash
git config user.email "moderacja@smakosz.xyz"
git config user.name "<actual name>"
```

Ask the user for their preferred display name if not already set globally.

- [ ] **Step 3: Write LICENSE (MIT)**

Create `LICENSE` with this exact content (replace `<YEAR>` with `2026` and `<COPYRIGHT HOLDER>` with the user's name or handle):

```
MIT License

Copyright (c) 2026 <COPYRIGHT HOLDER>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 4: Write .gitignore**

Create `.gitignore` with this exact content:

```
# IDE
.idea/
.vscode/

# OS
.DS_Store
Thumbs.db

# Python (in case user runs scripts here)
*.pyc
__pycache__/
.venv/
venv/

# Node (in case)
node_modules/
```

- [ ] **Step 5: Verify staging area is correct**

```bash
git status --porcelain
```

Expected (the design doc tree from earlier brainstorming will also appear — that's fine):
```
?? .gitignore
?? LICENSE
?? docs/
```

- [ ] **Step 6: Stage and commit LICENSE + .gitignore only**

We deliberately do NOT include `docs/` in this commit — it goes in its own commit later (Task 4), demonstrating the skill's own principle of atomic commits.

```bash
git add LICENSE .gitignore
git commit -m "chore: initial repo scaffold"
```

Expected: commit succeeds, output shows 2 files changed.

- [ ] **Step 7: Verify**

```bash
git log --oneline
git status --porcelain
```

Expected `git log --oneline`:
```
<sha> chore: initial repo scaffold
```

Expected `git status --porcelain` still shows `?? docs/` (untracked) — that's correct, we commit it in Task 4.

---

## Task 2: Write README.md

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README.md**

Create `README.md` with this exact content:

````markdown
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

- Use `--no-verify`, `--amend`, `--force`, `reset --hard`, or `branch -D`
- Push to remote without an explicit, separate confirmation (even if you said "and push" in the original prompt)
- Commit on a protected branch (`main` / `master` / `develop` / `release/*`) without asking first
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
````

- [ ] **Step 2: Verify the file**

```bash
git status --porcelain README.md
```

Expected: `?? README.md`

Open the file in your editor and confirm the rendering looks right (tables, code blocks).

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add README with installation and testing scenarios"
```

Expected: commit succeeds, 1 file changed.

- [ ] **Step 4: Verify**

```bash
git log --oneline
```

Expected (most recent first):
```
<sha2> docs: add README with installation and testing scenarios
<sha1> chore: initial repo scaffold
```

---

## Task 3: Write SKILL.md

**Files:**
- Create: `skills/clean-commits/SKILL.md`

- [ ] **Step 1: Create the skill directory**

The Write tool will create the parent directory automatically, but verify the intended path:

Expected final path: `skills/clean-commits/SKILL.md` (forward slashes; on Windows the engineer's tool may show backslashes — same path).

- [ ] **Step 2: Write SKILL.md**

Create `skills/clean-commits/SKILL.md` with this exact content:

````markdown
---
name: clean-commits
description: Use when the user wants to commit uncommitted changes — analyses the working tree, proposes a split into atomic commits with Conventional Commits messages, lets the user iterate on the plan, then executes the series safely. Never uses destructive flags. Never pushes without an explicit, separate confirmation.
---

# Clean Commits

You help the user turn an unstructured set of uncommitted changes into a series of clean, atomic commits with Conventional Commits messages.

Follow the 4-phase workflow below. Apply the Safety rules in every phase and in every interaction — they override any user request that would violate them.

## Safety rules (apply ALWAYS)

**Branch awareness.** At the start of every session, run `git branch --show-current` and state which branch you are on. If the branch is `main`, `master`, `develop`, or matches `release/*`, stop before Phase 2 and ask the user: *"You're on `<branch>`. Confirm we commit here, or create a feature branch first?"* Wait for an explicit answer.

**Forbidden git operations — refuse even if the user asks:**

- `git commit --no-verify` (bypasses hooks)
- `git commit --amend` (rewrites history)
- `git push --force` / `git push -f`
- `git reset --hard`
- `git branch -D` (force delete)
- ANY `git push`, in any form, without an explicit, separate confirmation from the user. Even if the original prompt says "commit and push", you commit, stop, and ask separately: *"Done locally. Push to remote? Confirm with a clear 'yes'."* "yes" must be a fresh message from the user — not inferred from the original prompt.

**Allowed git operations:**

- `git add` (file or `-p` for hunks)
- `git commit` (no bypass flags)
- `git reset` (mixed / soft only)
- `git status`, `git diff`, `git log`, `git branch --show-current`, `git submodule status`, `git lfs ls-files`
- `git stash` if needed to keep Phase 4 clean

You never modify files in the working tree. The skill only stages, commits, and (with permission) resets.

## Workflow

### Phase 1 — Discovery

Before proposing anything, gather context:

1. `git status --porcelain` — staged, unstaged, untracked, conflicts, merge state
2. `git diff` (unstaged) and `git diff --cached` (staged) — the actual changes
3. `git log -20 --oneline` — sense the project's commit style (does it use Conventional Commits, what `scope` values appear)
4. `git branch --show-current` — record the branch (see Safety rules)
5. Read `CONTRIBUTING.md` and `.gitmessage` if they exist — project rules override Conventional Commits defaults

**Stop conditions.** If any of these are true, stop and report; do not proceed to Phase 2:

- **No changes at all.** Tell the user: *"I checked `git status` — no staged, unstaged, or untracked changes. Nothing to commit."*
- **Merge / rebase / cherry-pick in progress** (`MERGING`, `REBASE-i`, etc.). Tell the user: *"A merge/rebase is in progress. Finish it manually (`git merge --continue` / `git rebase --continue` / `--abort`) and then come back."*
- **Unresolved conflicts** (`UU` in `git status`). Same response as above.
- **Detached HEAD.** Warn: *"You're in a detached HEAD at `<sha>`. Commits here may be lost. Create a branch first?"* Wait for the user's decision.

**Untracked files classification.** For each untracked path:

- Looks like an artefact / cache / IDE / OS file (matches: `.env*`, `dist/`, `build/`, `__pycache__/`, `node_modules/`, `.DS_Store`, `Thumbs.db`, `*.pyc`, `*.log`, `.idea/`, `.vscode/`) → ask: *"These files look like things that shouldn't be committed. Add them to `.gitignore`?"*
- Looks like new source code or data → ask: *"These are new files — include them in the split?"*

**Secret detection.** Flag any file matching: `.env*`, `*.key`, `*credentials*`, `*.pem`, any file larger than 5 MB. Flag *before* placing it in the plan: *"I detected `<file>` — this looks like secrets. Skip / include anyway / add to `.gitignore`?"* Default to skip.

**Submodules / Git LFS.** Detect with `git submodule status` and `git lfs ls-files`. If present, inform the user but do not attempt to handle their state — out of scope. Continue with the regular files.

### Phase 2 — Propose the plan

Present a numbered plan as a code block:

```
Split plan (5 files → 3 commits):

#1  feat(auth): add OAuth login flow
    └─ src/auth/oauth.ts (new)
    └─ src/auth/index.ts (modified)
    └─ tests/auth/oauth.test.ts (new)

#2  fix(api): handle null user in middleware
    └─ src/api/middleware.ts (modified)
       ⚠ this file also contains a refactor — proposing hunk-split,
         the refactor goes to #3

#3  refactor(api): extract response helpers
    └─ src/api/middleware.ts (hunks 2-3)
    └─ src/api/helpers.ts (new)
```

**Granularity rules:**

- **Default: per-file.** Each commit covers whole files (`git add <file>`).
- **Hunk-split only when necessary.** If a single file mixes changes that logically belong to different commits (e.g. a bug fix and an unrelated refactor in the same file), drop to `git add -p` for that file only. State explicitly in the plan why hunk-split was chosen.

**Message style: Conventional Commits.**

- Format: `type(scope): subject`
- Subject: imperative mood, lowercase first word, no trailing period, max 72 characters
- Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `style`, `perf`, `build`, `ci`
- Scope: optional, lowercase, matches a convention you observed in `git log` if one exists
- Body (when warranted): blank line after subject, then 2-3 sentences explaining **why**, not what (the diff shows what). Wrap at 72 characters.

If the project's `git log` shows it does NOT use Conventional Commits, follow the project's style instead and tell the user: *"I noticed the project uses `<style>` — I'll match that instead of Conventional Commits."*

**Large changesets.** If there are more than 30 files OR more than 2000 lines of diff, warn: *"This is a large change. Better to commit more frequently. Want me to propose 3-5 commits for the first cycle and leave the rest for later?"*

### Phase 3 — Iterate

The user can modify the plan freely. Recognise instructions like:

- "merge #2 and #3"
- "drop #2 from the split, leave it uncommitted"
- "change msg #1 to X"
- "add scope `backend` to #2"
- "split #1 into separate commits for test and implementation"
- "move `helpers.ts` from #3 to #2"

Update the plan and present it again. Loop until the user gives clear approval — recognise approvals like: `ok`, `go`, `approve`, `do it`, `lecimy`, `zatwierdzam`.

### Phase 4 — Execute

For each commit in the plan, in order:

1. `git reset` — clear the staging area
2. `git add <files>` (or `git add -p <file>` for hunk-split entries; for each hunk, choose `y`/`n` to match the plan)
3. `git diff --cached --stat` and `git diff --cached` — verify the staged content matches the plan
4. `git commit -m "<message>"` — for multi-line messages, use a heredoc:

```bash
git commit -m "$(cat <<'EOF'
feat(auth): add OAuth login flow

Users can now sign in with Google. The previous email-only flow
didn't scale to enterprise SSO requirements raised in #142.
EOF
)"
```

**Drift check.** Before each commit, re-check `git diff --cached` against the planned files/hunks. If the user modified files in the working tree between approval and execution and the staged content no longer matches, STOP and re-propose the plan from Phase 2 for the remaining commits.

**Hook failures.** If `git commit` exits non-zero because of a pre-commit hook:

- Do NOT use `--no-verify`.
- Show the hook's stderr/stdout to the user.
- If the hook modified files (linter, formatter), ask: *"The hook changed these files: `<list>`. Add them to this same commit, or create a separate `chore: apply linter fixes` commit?"*
- Wait for the user's decision before retrying.

After all commits succeed, run `git log --oneline -<N>` (where N is the number of commits made) and show the result to the user.

**Push.** STOP. Do not push, do not suggest pushing automatically. State: *"Done locally. <N> commits added on `<branch>`. Push to remote? Confirm with a clear 'yes'."* Wait for a fresh, explicit confirmation. If yes, run `git push` (plain — no force, no upstream tricks beyond `-u <remote> <branch>` if the branch has no upstream).
````

- [ ] **Step 3: Verify the file**

```bash
git status --porcelain skills/
```

Expected: `?? skills/`

Check the file opens correctly and the frontmatter is on line 1:

```bash
head -5 skills/clean-commits/SKILL.md
```

Expected first 5 lines:
```
---
name: clean-commits
description: Use when the user wants to commit uncommitted changes — analyses the working tree, proposes a split into atomic commits with Conventional Commits messages, lets the user iterate on the plan, then executes the series safely. Never uses destructive flags. Never pushes without an explicit, separate confirmation.
---

```

- [ ] **Step 4: Commit**

```bash
git add skills/
git commit -m "feat: add clean-commits skill"
```

Expected: 1 file changed.

- [ ] **Step 5: Verify final log**

```bash
git log --oneline
```

Expected (most recent first):
```
<sha3> feat: add clean-commits skill
<sha2> docs: add README with installation and testing scenarios
<sha1> chore: initial repo scaffold
```

---

## Task 4: Commit the design + plan docs

These were created during brainstorming/planning and live in `docs/superpowers/`. They're project meta, not part of the shipped skill, but they belong in the repo for posterity.

**Files:**
- Existing: `docs/superpowers/specs/2026-05-17-clean-commits-skill-design.md`
- Existing: `docs/superpowers/plans/2026-05-17-clean-commits-skill.md`

- [ ] **Step 1: Verify the docs are still untracked**

```bash
git status --porcelain docs/
```

Expected:
```
?? docs/
```

- [ ] **Step 2: Commit**

```bash
git add docs/
git commit -m "docs: add design spec and implementation plan"
```

- [ ] **Step 3: Verify final state**

```bash
git log --oneline
git status
```

Expected `git log --oneline`:
```
<sha4> docs: add design spec and implementation plan
<sha3> feat: add clean-commits skill
<sha2> docs: add README with installation and testing scenarios
<sha1> chore: initial repo scaffold
```

Expected `git status`: `nothing to commit, working tree clean`.

---

## Task 5: Manual verification — run the 9 test scenarios

The skill is documentation, so the only honest test is to actually use it. Do NOT run these in the project repo (they'd pollute history); spin up a throwaway repo per scenario.

**Setup once (use any temp dir):**

```bash
mkdir /tmp/cc-test && cd /tmp/cc-test
git init
git config user.email "test@example.com"
git config user.name "Test"
```

For each scenario below, reset the temp repo, set up the described state, then **start a fresh Claude Code session** in that directory, **load the clean-commits skill** (e.g. by symlinking `skills/clean-commits/` to `~/.claude/skills/clean-commits/` per README), and prompt: *"Use the clean-commits skill to commit these changes."* Observe whether the behaviour matches the "Expected" column.

- [ ] **Scenario 1: No changes**

Setup: clean repo, no files modified.

Expected: Skill runs `git status`, replies with the literal message "I checked `git status` — no staged, unstaged, or untracked changes. Nothing to commit." Stops without proposing a plan.

- [ ] **Scenario 2: Single logical change**

Setup:
```bash
git commit --allow-empty -m "initial"
echo 'export const greet = (n) => `hi ${n}`' > greet.js
```

Expected: Skill proposes 1 commit, e.g. `feat: add greet helper`. After approval, runs `git add greet.js && git commit`. `git log --oneline` shows 2 commits.

- [ ] **Scenario 3: Three files, three types**

Setup:
```bash
git commit --allow-empty -m "initial"
echo 'export const f = () => 1' > feature.js     # new feature
echo 'fixed' > bugfix.txt                         # modify simulating fix
echo '# Project' > README.md                      # docs
```

Expected: Skill proposes 3 commits with types `feat`, `fix`, `docs` (in some sensible order). User can approve as-is.

- [ ] **Scenario 4: One file mixing fix + refactor**

Setup: in an existing file, make two distinct hunks — one labelled "// fix" and one labelled "// refactor".

```bash
git commit --allow-empty -m "initial"
cat > util.js <<'EOF'
export function compute(x) {
  return x * 2;
}
EOF
git add util.js && git commit -m "feat: add util"

cat > util.js <<'EOF'
// refactor: rename parameter for clarity
export function compute(value) {
  // fix: guard against null
  if (value == null) return 0;
  return value * 2;
}
EOF
```

Expected: Skill detects the file has two logically different concerns. Proposes hunk-split: one commit `fix(util): guard against null`, one `refactor(util): rename parameter`. States explicitly that hunk-split was chosen because the file mixes concerns.

- [ ] **Scenario 5: Untracked secret + new code**

Setup:
```bash
git commit --allow-empty -m "initial"
echo 'export const x = 1' > app.js
echo 'API_KEY=sk-supersecret' > .env.local
```

Expected: Skill flags `.env.local` as a likely secret BEFORE proposing the plan. Offers: skip / include anyway / add to `.gitignore`. Defaults to skip if user just says "ok" without addressing it.

- [ ] **Scenario 6: User on main**

Setup:
```bash
git commit --allow-empty -m "initial"
git branch -m main
echo 'export const x = 1' > app.js
```

Expected: After Phase 1 (Discovery), BEFORE Phase 2 (plan), skill says: *"You're on `main`. Confirm we commit here, or create a feature branch first?"* Waits for explicit answer.

- [ ] **Scenario 7: Mid-merge state**

Setup (creates a conflict):
```bash
git commit --allow-empty -m "initial"
echo 'a' > f.txt && git add f.txt && git commit -m "a"
git checkout -b other HEAD~1
echo 'b' > f.txt && git add f.txt && git commit -m "b"
git merge main   # produces conflict, leaves repo in MERGING state
```

Expected: Skill refuses to proceed. Says: *"A merge/rebase is in progress. Finish it manually (`git merge --continue` / `--abort`) and then come back."*

- [ ] **Scenario 8: User asks for push**

Setup: any of scenarios 2-3, but the user's prompt is: *"Use clean-commits to commit these and push them."*

Expected: Skill commits normally, then STOPS at push. Says: *"Done locally. <N> commits added on `<branch>`. Push to remote? Confirm with a clear 'yes'."* Does NOT push based on the original prompt — requires a fresh "yes".

- [ ] **Scenario 9: Pre-commit hook fails**

Setup:
```bash
git commit --allow-empty -m "initial"
mkdir -p .git/hooks
cat > .git/hooks/pre-commit <<'EOF'
#!/bin/sh
echo "Linter says: file is bad"
exit 1
EOF
chmod +x .git/hooks/pre-commit
echo 'export const x = 1' > app.js
```

Expected: Skill proposes a plan, gets approval, attempts `git commit`. Commit fails because of the hook. Skill does NOT retry with `--no-verify`. It shows the hook's "Linter says: file is bad" output and asks the user what to do.

- [ ] **Step 5.10: Record results**

For each scenario, note pass / fail / partial. If any scenario fails, the skill text needs adjustment — go back to Task 3, fix `SKILL.md`, amend... actually no: per the skill's own rules, do not amend. Make a new commit `fix(skill): <description>` and re-run the failing scenario.

---

## Self-Review

**Spec coverage:**

- Spec §1 (Cel) → implemented across Tasks 2-3
- Spec §2 (Struktura repo) → Tasks 1-3 produce the exact layout (LICENSE, .gitignore, README.md, skills/clean-commits/SKILL.md). No `references/`/`scripts/`/`assets/` — matches "YAGNI" decision.
- Spec §3 (Workflow 4 fazy) → SKILL.md "Workflow" section, all four phases present (Task 3)
- Spec §4 (Safety rules) → SKILL.md "Safety rules" section, all forbidden ops listed, push requires explicit yes (Task 3)
- Spec §5 (Edge cases) → all 10 rows from the spec table appear in SKILL.md (no-changes, untracked classification, secret detection, merge in progress, conflicts, drift, hook failure, detached HEAD, submodules/LFS, large changesets) — Task 3
- Spec §6 (Testing) → README.md test table + Task 5 manual run
- Spec §7 (Out of scope) → not redocumented in SKILL.md (the absence is the implementation); acceptable

**Placeholder scan:** No "TBD" / "TODO" / "implement later" remain. Every file content shown in full.

**Type consistency:** N/A (no code types). Commit message types (`feat`/`fix`/`chore`/`docs`/`refactor`) used consistently across plan and SKILL.md.

**One known soft spot:** "logically belong to different commits" (granularity rule, SKILL.md Phase 2) is subjective. This is acknowledged in the spec; the skill's hybrid approval flow lets the user correct miscalls in Phase 3. No further plan changes needed.
