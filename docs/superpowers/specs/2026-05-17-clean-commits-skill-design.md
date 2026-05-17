# Clean Commits Skill — Design

**Data:** 2026-05-17
**Autor:** moderacja@smakosz.xyz (z asystą Claude)
**Status:** Approved, ready for implementation plan

---

## 1. Cel

Zbudować skill Claude, który prowadzi użytkownika przez pełny pipeline tworzenia "czystych" commitów: od analizy wszystkich niezacommitowanych zmian, przez zaproponowanie podziału na atomowe commity, po wykonanie serii `git add` + `git commit` z wiadomościami w konwencji Conventional Commits.

Skill jest dystrybuowany jako repozytorium GitHub do skopiowania/zalinkowania do `~/.claude/skills/clean-commits/`. Nie jest pluginem Claude Code.

## 2. Struktura repozytorium

```
clean-commits-skill/
├── skills/
│   └── clean-commits/
│       └── SKILL.md          # właściwa treść skilla (frontmatter + instrukcje)
├── README.md                  # dla GitHub: opis, instalacja, przykład, testing scenarios
├── LICENSE                    # MIT
└── .gitignore                 # standard + .idea/ (PyCharm)
```

**Decyzje:**
- Katalog skilla: `clean-commits` (nie `clean-commits-skill` — uniknięcie redundancji z `skills/`).
- Repo nazwa: `clean-commits-skill` (istniejąca).
- **Brak** `references/`, `scripts/`, `assets/` na start (YAGNI — dodamy gdy SKILL.md urośnie >300 linii lub gdy faktycznie potrzebny będzie zewnętrzny skrypt).
- **Brak** `.claude-plugin/plugin.json` — nie pluginujemy.
- **Wszystkie pliki w repo po angielsku** (SKILL.md, README.md, .gitignore comments). Ten spec jest po polsku, bo to dokument roboczy między userem a Claude.

## 3. Workflow skilla — 4 fazy

### Faza 1 — Discovery

Skill zbiera kontekst zanim cokolwiek zaproponuje:

- `git status --porcelain` — pełen obraz: staged, unstaged, untracked, konflikty, stan merge/rebase
- `git diff` (unstaged) + `git diff --cached` (staged) — treść zmian
- `git log -20 --oneline` — wykryć styl projektu (czy używa Conventional Commits, jakie `scope`-y się pojawiają)
- `git branch --show-current` — sprawdzić branch
- Jeśli istnieje `CONTRIBUTING.md` lub `.gitmessage` → przeczytać, bo reguły projektu nadpisują domyślną konwencję CC

**Branch check (mandatory):** Jeśli aktualny branch to `main`/`master`/`develop`/`release/*` (chronione), skill **przed Fazą 2** pyta: *"Jesteś na `main`. Na pewno commitujemy tutaj, czy stworzyć branch?"* — czeka na jednoznaczną odpowiedź.

### Faza 2 — Propozycja planu

Skill prezentuje plan w formie tabeli:

```
Plan podziału (5 plików → 3 commity):

#1  feat(auth): add OAuth login flow
    └─ src/auth/oauth.ts (new)
    └─ src/auth/index.ts (modified)
    └─ tests/auth/oauth.test.ts (new)

#2  fix(api): handle null user in middleware
    └─ src/api/middleware.ts (modified)
       ⚠ ten plik zawiera też refactor — proponuję hunk-split,
         zostawimy refactor jako #3

#3  refactor(api): extract response helpers
    └─ src/api/middleware.ts (hunks 2-3)
    └─ src/api/helpers.ts (new)
```

**Granularność splitu:**
- **Domyślnie per-plik**: każdy commit obejmuje całe pliki (`git add <file>`).
- **Hunk-split tylko gdy konieczne**: jeśli w jednym pliku wykryte są zmiany należące logicznie do różnych commitów (np. fix + refactor), skill schodzi do `git add -p` dla tego konkretnego pliku.

**Wiadomości:** Conventional Commits (`type(scope): subject` + opcjonalne body wyjaśniające *dlaczego*, nie *co*).

### Faza 3 — Iteracja

User może modyfikować plan:
- "połącz #2 i #3"
- "usuń #2 ze split-a, zostaw jako uncommitted"
- "zmień msg #1 na X"
- "dodaj scope `backend` do #2"
- "podziel #1 na osobne commity dla testu i implementacji"

Skill aktualizuje plan i pokazuje go ponownie. Pętla aż user napisze "OK" / "lecimy" / "zatwierdzam" (lub równoważnie po angielsku w SKILL.md: "ok" / "go" / "approve").

### Faza 4 — Execute

Dla każdego commita po kolei:
1. `git reset` — czyste staging area
2. `git add <pliki>` lub `git add -p <plik>` dla hunk-split
3. `git diff --cached` — weryfikacja że staged matchuje plan
4. `git commit -m "..."` (heredoc dla wieloliniowych wiadomości)

**Plan-execute drift check:** Przed każdym commitem skill weryfikuje że `git diff --cached` faktycznie odpowiada temu, co było w planie. Jeśli rozjazd (user zmienił pliki między approve a execute) — stop, info, ponowna propozycja.

Po wszystkim: `git log --oneline -<N>` żeby user widział rezultat.

## 4. Safety rules (zawsze, nie tylko w workflow)

Sekcja w SKILL.md, którą skill stosuje **niezależnie od fazy** — nawet jeśli user zacznie rozmowę nietypowo (np. "dorzuć ten plik do ostatniego commita" = amend → odmowa).

**Forbidden zawsze:**
- `git commit --no-verify` (bypass hooków)
- `git commit --amend` (modyfikacja historii)
- `git push --force` / `git push -f`
- `git reset --hard`
- `git branch -D` (force delete)
- Żaden `push` bez **wyraźnego, osobnego** potwierdzenia usera (jednoznaczne "tak"/"yes"). Nawet jeśli user powie "i wypchnij" w pierwotnym promcie — skill commituje, zatrzymuje się, pyta osobno.

**Dozwolone:**
- `git add`, `git commit` (bez bypass flag)
- `git reset` (mixed/soft)
- `git status`, `git diff`, `git log`, `git branch --show-current`
- `git stash` (jeśli konieczne do czystej Fazy 4)

**Branch awareness:** Skill zawsze wie i komunikuje na którym branchu pracuje. Każdy commit na chronionym branchu wymaga potwierdzenia.

## 5. Edge cases

| Sytuacja | Reakcja |
|---|---|
| Brak zmian | Skill mówi explicite: *"Sprawdziłem `git status` — nie wykryłem żadnych zmian (staged/unstaged/untracked). Nie ma co commitować."* Stop. |
| Tylko untracked files | Skill listuje i klasyfikuje: jeśli wyglądają na artefakty/cache/sekrety (`.env*`, `dist/`, `__pycache__/`, `.idea/`, `node_modules/`, `*.log`, `.DS_Store`, `*.pyc`) → pyta: *"Te pliki wyglądają na rzeczy, które nie powinny być commitowane. Dodać do `.gitignore`?"* Jeśli wyglądają na nowy kod → pyta czy włączyć do splitu. |
| Pliki z sekretami | Heurystyka: `.env*`, `*.key`, `*credentials*`, `*.pem`, pliki >5MB. Skill flaguje **przed** dodaniem do planu, proponuje opcje: pominąć / dodać mimo to / dopisać do `.gitignore`. Domyślnie pomija. |
| Merge / rebase / cherry-pick w toku | Skill odmawia operacji. *"Trwa merge/rebase. Dokończ go ręcznie (`--continue` / `--abort`), potem wróć."* |
| Konflikty (`UU` w status) | Jak wyżej — odmowa, info dla usera. |
| Plan-execute drift | Skill wykrywa przez ponowne `git diff`, zatrzymuje się, mówi co się zmieniło, pyta czy przeplanować. |
| Hook failure | Skill **nie** używa `--no-verify`. Pokazuje output hooka, pyta. Jeśli hook zmodyfikował pliki (linter/formatter) — pyta czy włączyć te zmiany do tego samego commita czy do osobnego. |
| Detached HEAD | Ostrzeżenie: *"Jesteś w detached HEAD na `<sha>`. Commity tutaj mogą zostać utracone. Stworzyć branch?"* Czeka na decyzję. |
| Submoduły / LFS | Skill wykrywa (`git submodule status`, `git lfs ls-files`) i tylko informuje — sam nie obsługuje. |
| >30 plików lub >2000 linii diff | Skill ostrzega o rozmiarze, proponuje podział na 3-5 pierwszych commitów + resztę do późniejszego cyklu. |

## 6. Testing

Skill to dokument — brak unit testów. Walidacja przez **manualne scenariusze** (do README.md jako sekcja "Testing the skill"):

| # | Scenariusz | Oczekiwanie |
|---|---|---|
| 1 | Czyste repo, brak zmian | Skill mówi *"nie wykryłem żadnych zmian"*, stop |
| 2 | 1 plik zmieniony, jedna logiczna zmiana | Skill proponuje 1 commit z poprawnym CC msg |
| 3 | 3 pliki: nowy feature + bugfix + docs | Skill proponuje 3 osobne commity z różnymi typami |
| 4 | 1 plik z fix + refactor wewnątrz | Skill flaguje konflikt logik, proponuje hunk-split |
| 5 | Untracked `.env.local` + nowy kod | Skill flaguje sekret, pyta o `.gitignore` dla `.env.local` |
| 6 | User na branchu `main` | Skill pyta o potwierdzenie commit-u na main |
| 7 | Trwa merge | Skill odmawia, prosi o dokończenie mergea |
| 8 | User pisze "i wypchnij" na końcu | Skill commituje, ale przy push czeka na osobne `tak` |
| 9 | Pre-commit hook failuje | Skill nie używa `--no-verify`, pokazuje output, pyta |

Brak CI. Jeśli kiedyś dojdą `scripts/`, wtedy CI ma sens.

## 7. Out of scope

Eksplicytnie **nie robimy** w v1:
- Generowania PR description / changelog (osobny skill)
- Integracji z GitHub/GitLab API (issues, PR-y) — tylko lokalny git
- `git rebase`, `git cherry-pick`, naprawa historii
- Multi-repo / monorepo workspace coordination
- Konfiguracji per-user (custom prompts, custom rules) — wszystko hardcoded w SKILL.md, user forkuje jeśli chce zmieniać
- Wsparcia dla innych konwencji niż Conventional Commits (gitmoji, jira-style itd.)
- Push do remote — nawet po potwierdzeniu user musi pushować sam (skill może maksymalnie zasugerować komendę)

## 8. Następne kroki

1. Implementation plan (writing-plans skill)
2. Implementacja: SKILL.md, README.md, LICENSE, .gitignore
3. Manualny test 9 scenariuszy z sekcji 6
4. Pierwszy commit na branchu, push do GitHub
