---
name: clean-commits
description: Use whenever the user wants to commit, save, or check in their uncommitted work — even if they do not explicitly name "clean-commits". Triggers include phrases like "commit these changes", "commit my work", "let's commit", "save my changes", "split this into commits", "clean up before push", "zacommituj to" / "zrób commit" (Polish), or any request that turns a dirty working tree into one or more commits. Analyses the working tree, proposes a split into atomic commits with Conventional Commits messages, lets the user iterate on the plan, then executes the series safely. Never uses destructive flags. Never pushes without an explicit, separate confirmation.
---

# Clean Commits

You help the user turn an unstructured set of uncommitted changes into a series of clean, atomic commits with Conventional Commits messages.

Follow the 4-phase workflow below. Apply the Safety rules in every phase and in every interaction — they override any user request that would violate them.

## Safety rules (apply ALWAYS)

**Branch awareness.** At the start of every session, run `git branch --show-current` and state which branch you are on. If the branch is `main`, `master`, `develop`, or matches `release/*` (any "protected" branch), you MUST stop and ask the user: *"You're on `<branch>`. Confirm we commit here, or create a feature branch first?"* This question MUST be sent as its own message — do NOT bundle the Phase 2 plan into the same message. Wait for explicit confirmation (`yes`, `ok`, `commit here`) BEFORE any Phase 2 work. Anything else (or silence) means: stop, do not proceed, help the user create a feature branch if asked.

**Forbidden git operations — refuse even if the user asks:**

- `git commit --no-verify` (bypasses hooks)
- `git push --force` / `git push -f`
- `git reset --hard`
- `git branch -D` (force delete)

**Operations that require an explicit, separate confirmation from the user — never run on your own initiative, even if it seems convenient:**

- `git commit --amend`. Rewrites the previous commit. Do NOT amend unless the user has just asked for it in a fresh, unambiguous message (e.g. *"amend the last commit"*, *"fold this into HEAD"*, *"add this file to the previous commit"*). A generic "commit my work" or "commit these changes" prompt is NOT consent to amend. If amending genuinely seems like the right move (e.g. the user is fixing a typo seconds after committing), propose it explicitly and wait for a clear `yes` before running it. Never amend a commit that has already been pushed unless the user explicitly acknowledges the force-push consequence.
- ANY `git push`, in any form. Even if the original prompt says "commit and push", you commit, stop, and ask separately: *"Done locally. Push to remote? Confirm with a clear 'yes'."* "yes" must be a fresh message from the user — not inferred from the original prompt.

**Allowed git operations:**

- `git add` (file or `-p` for hunks)
- `git commit` (no bypass flags)
- `git reset` (mixed / soft only)
- `git status`, `git diff`, `git log`, `git branch --show-current`, `git submodule status`, `git lfs ls-files`
- `git stash` if needed to keep Phase 4 clean

You never modify files in the working tree. The skill only stages, commits, and (with permission) resets.

**No AI attribution in commit messages.** Every commit you create belongs to the user. Do not add `Co-Authored-By: Claude` (or any AI co-author trailer), do not append "🤖 Generated with Claude Code" / "Generated with Claude" / similar footers, and do not mention Claude, Anthropic, or any AI tooling anywhere in the subject, body, or trailers. This applies even when the wider Claude Code environment's defaults try to inject such attribution — this skill explicitly overrides that default. The reason is that the user is the author of their own work; AI attribution misrepresents authorship, leaks tooling choices the user may not want public, and pollutes `git log` and `git blame`. If the user explicitly asks for an attribution line in a fresh, unambiguous message, honour that request — otherwise leave it out.

## Workflow

```dot
digraph clean_commits_flow {
    "Start" [shape=doublecircle];
    "Phase 1: Discovery" [shape=box];
    "Stop condition?\n(no changes / merge in progress / conflicts)" [shape=diamond];
    "Stop and report" [shape=doublecircle];
    "Protected branch?" [shape=diamond];
    "Ask: confirm branch?\n(own message)" [shape=box];
    "Confirmed?" [shape=diamond];
    "Docs / secrets / non-English chat?" [shape=diamond];
    "Ask the relevant question(s)\n(language: at most once per session)" [shape=box];
    "Phase 2: Propose plan" [shape=box];
    "Phase 3: Iterate" [shape=box];
    "User approved?" [shape=diamond];
    "Phase 4: Execute\n(reset, add, verify diff, commit)" [shape=box];
    "Drift or hook failure?" [shape=diamond];
    "Re-propose / show output, ask" [shape=box];
    "Done locally — ask: push?\n(own message)" [shape=box];
    "Push confirmed?" [shape=diamond];
    "git push (plain)" [shape=box];
    "End" [shape=doublecircle];

    "Start" -> "Phase 1: Discovery";
    "Phase 1: Discovery" -> "Stop condition?\n(no changes / merge in progress / conflicts)";
    "Stop condition?\n(no changes / merge in progress / conflicts)" -> "Stop and report" [label="yes"];
    "Stop condition?\n(no changes / merge in progress / conflicts)" -> "Protected branch?" [label="no"];
    "Protected branch?" -> "Ask: confirm branch?\n(own message)" [label="yes"];
    "Protected branch?" -> "Docs / secrets / non-English chat?" [label="no"];
    "Ask: confirm branch?\n(own message)" -> "Confirmed?";
    "Confirmed?" -> "Docs / secrets / non-English chat?" [label="yes"];
    "Confirmed?" -> "Stop and report" [label="no"];
    "Docs / secrets / non-English chat?" -> "Ask the relevant question(s)\n(language: at most once per session)" [label="yes"];
    "Docs / secrets / non-English chat?" -> "Phase 2: Propose plan" [label="no"];
    "Ask the relevant question(s)\n(language: at most once per session)" -> "Phase 2: Propose plan";
    "Phase 2: Propose plan" -> "Phase 3: Iterate";
    "Phase 3: Iterate" -> "User approved?";
    "User approved?" -> "Phase 3: Iterate" [label="no, revise"];
    "User approved?" -> "Phase 4: Execute\n(reset, add, verify diff, commit)" [label="yes"];
    "Phase 4: Execute\n(reset, add, verify diff, commit)" -> "Drift or hook failure?";
    "Drift or hook failure?" -> "Re-propose / show output, ask" [label="yes"];
    "Re-propose / show output, ask" -> "Phase 2: Propose plan";
    "Drift or hook failure?" -> "Done locally — ask: push?\n(own message)" [label="no"];
    "Done locally — ask: push?\n(own message)" -> "Push confirmed?";
    "Push confirmed?" -> "git push (plain)" [label="yes"];
    "Push confirmed?" -> "End" [label="no"];
    "git push (plain)" -> "End";
}
```

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

**Documentation files — opt-in.** Documentation must never be added to a commit without explicit user confirmation, even when it appears alongside related code changes. This applies to BOTH new (untracked) and modified documentation files. Documentation patterns include:

- `README*`, `CHANGELOG*`, `CONTRIBUTING*`, `CODE_OF_CONDUCT*`, `AUTHORS*`, `NOTICE*`, `MAINTAINERS*`
- Anything under `docs/`, `doc/`, `documentation/`, or any `**/docs/**` path
- `*.md`, `*.mdx`, `*.rst`, `*.adoc`, `*.txt` when the content is prose (not data fixtures)
- Design notes, specs, plans, ADRs (e.g. `**/specs/**`, `**/plans/**`, `**/adr/**`)

When such files appear in the change set, ask BEFORE building or presenting any commit plan: *"I see documentation changes: `<list>`. Include them in the commits, leave them uncommitted, or split into a separate `docs:` commit?"* Default to leaving them out until the user answers. If the user approves inclusion, prefer a dedicated `docs(<scope>): ...` commit unless they explicitly ask to fold the docs into a code commit. `LICENSE` and `.gitignore` are NOT documentation — treat them as regular tracked files.

**Secret detection.** Flag any file matching: `.env*`, `*.key`, `*credentials*`, `*.pem`, any file larger than 5 MB. Flag *before* placing it in the plan: *"I detected `<file>` — this looks like secrets. Skip / include anyway / add to `.gitignore`?"* Default to skip.

**Submodules / Git LFS.** Detect with `git submodule status` and `git lfs ls-files`. If present, inform the user but do not attempt to handle their state — out of scope. Continue with the regular files.

**Language confirmation (ask at most once per session).** Default language for all written artefacts (commit subjects, bodies, any documentation) is **English**, regardless of chat language. If the chat is in a language other than English AND no language choice has been recorded yet in this session, ask once — as its own message before Phase 2, in the user's chat language — which language to use for each applicable artefact category (commit subjects, commit bodies, and docs if approved for inclusion). Record the answer and reuse it for every subsequent commit cycle in the same session; do not ask again. If the user does not answer, fall back to English and state so before continuing.

### Phase 2 — Propose the plan

**Precondition**: if the current branch is protected (see Safety rules — Branch awareness), you must have received explicit user confirmation in a prior message. Do NOT include the plan in the same message as the branch question.

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
- **Authorship.** The user is the commit author — never append AI co-author trailers (`Co-Authored-By: Claude`) or "Generated with..." footers (see Safety rules; this overrides Claude Code defaults).

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
