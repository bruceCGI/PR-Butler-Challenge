---
name: pr-butler
description: Automate the complete pre-commit / PR preparation workflow for the PR Butler scaffold (a Vite + TypeScript task manager in scaffold/website) — detect and fix missing French translations and wire them into the UI, format and lint the code, raise test coverage above 80%, add docstrings and README sections, generate CHANGELOG.md and PR_REQUEST.md, enforce quality gates, and prepare a conventional commit. Use when the user says "prepare for PR", "pre-commit check", "make this PR-ready", "fix the scaffold", or "run PR Butler", or asks for any single one of these steps on its own.
---

## Overview

PR Butler takes the scaffold at `scaffold/website/` from "builds, but not ready to ship" to "ready to merge", in six ordered steps followed by a Report Card:

| Step | Outcome | Main files touched |
|---|---|---|
| 0. Preflight | Baseline recorded, test environment working | `package.json` |
| 1. Translation Detection & Fix | All 14 keys translated **and actually rendered** in the UI | `src/translations/fr.json`, `index.html`, `src/i18n.ts`, `src/main.ts` |
| 2. Code Cleanup | Formatted, lint-clean, XSS and error handling fixed | `src/**/*.ts`, `index.html`, lint/format config |
| 3. Test Automation | All tests pass, coverage >= 80%, threshold enforced | `src/tests/*.test.ts`, `vitest.config.ts` |
| 4. Documentation Updates | Docstrings, README, CHANGELOG, PR description | `src/**/*.ts`, `scaffold/website/README.md`, `CHANGELOG.md`, `PR_REQUEST.md` |
| 5. Quality Gates | Every gate green, or a stop with a precise failure report | — |
| 6. PR Preparation | Conventional commit message, finalized PR description | `COMMIT_MSG.txt`, `PR_REQUEST.md` |
| 7. Report Card | Honest self-assessment in the fixed format | — |

**The order is load-bearing.** Step 2 cleanup changes the code Step 3 tests; Step 3 numbers go in the Step 4 documents; Step 5 checks all of it. Never reorder or skip ahead.

### When to Invoke

- "Prepare for PR", "pre-commit check", "make this PR-ready", "fix the scaffold", "run PR Butler" → run every step.
- A request for one step ("fix the French translations", "get coverage up") → run Step 0, the requested step, and the Step 5 gates that apply to it, then print the Report Card with every other step marked `FAIL` and `not attempted` in its details, as the Report Card rules require.
- Re-running on an already-fixed tree is safe: every step checks before it acts, so a second run changes nothing and reports PASS.

All paths below are relative to the repository root. App commands run from `scaffold/website/`.

---

## Instructions

### Ground rules

Apply these throughout. They exist because each one blocks a specific, common way an automated fix goes wrong.

1. **Never weaken a gate to pass it.** Do not lower a coverage threshold, add `eslint-disable`, exclude a file from coverage, `.skip` a test, or delete an assertion. If a gate cannot be met honestly, stop and report it (see Step 5).
2. **Every number comes from tool output.** Coverage, test counts and lint counts in the reports must be read from a command you actually ran in this session, never estimated.
3. **Be idempotent.** Check before acting: do not reinstall a present dependency, overwrite existing config, duplicate a README section, or re-add a CHANGELOG entry.
4. **Behavior changes get a test.** Pure formatting and renames do not need one. Anything that changes what the app does does (for example XSS escaping, input trimming, or storage error handling).
5. **`en.json` is the contract.** Never add, rename or remove keys in `src/translations/en.json`; the brief fixes the set at 14. Hardcoded strings that have no key are reported as follow-ups, not silently added.
6. **Brief vs. reality.** If the brief names a defect the code does not contain (for example `unusedVariable`), confirm its absence with a search and report "not present". Never create a problem so that you can fix it.
7. **Git safety.** Never `git push`, force-push, `reset --hard`, `stash`, or delete tracked files without being asked. Stage explicit paths only, never `git add -A`.
8. **Checkpoint commits.** If the user asked you to commit (for example "commit as you go"), make one conventional commit at the end of each step that changed files, such as `fix(i18n): add 12 missing French translations`. That gives reviewers a step-by-step history and makes each step revertable. Otherwise leave changes uncommitted and say so.
9. **Preserve line endings.** This repo was committed from Windows, so most files are stored with CRLF and `taskManager.ts` with LF. Check with `git ls-files --eol`. Writing LF into a CRLF file makes git report a rewrite of every line, which buries the real change. Keep each file's existing endings, and treat a `git diff --stat` count far larger than your edit as a sign you broke them. Regex edits must match `\r?\n`, not `\n`.

### Run state

Keep `.pr-butler/state.json` at the repo root (add `.pr-butler/` to `.gitignore`). Record the baseline in Step 0 and final numbers as each step completes. The Report Card's "X% → Y%" figures come from here. If the file exists when the skill starts, resume from the first step whose status is not `done`.

```json
{
  "baseline": { "frKeys": 2, "testsPassing": null, "coverage": null, "lintErrors": null, "undocumented": null },
  "final":    { "frKeys": null, "testsPassing": null, "coverage": null, "lintErrors": null, "undocumented": null },
  "steps": { "0": "done", "1": "in_progress" },
  "notes": []
}
```

### Step 0: Preflight

```bash
cd scaffold/website
node --version        # must be >= 18.18 (ESLint 9 needs it); stop and report if not
npm install
npm run build         # must pass before anything changes
npm run test:coverage
```

- **If the build fails at baseline**, stop: the problem predates this run, and fixing it silently would hide it from the reviewer.
- **If vitest prints `MISSING DEPENDENCY  Cannot find dependency 'jsdom'`**, the scaffold's `vitest.config.ts` requests a `jsdom` environment that is not installed. Install it pinned to a Node 18-compatible major, then rerun:
  ```bash
  npm install -D jsdom@^24     # jsdom 24-26 all support Node 18; ^24 is the proven pin
  npm run test:coverage
  ```
- Record the baseline: fr.json key count, tests passing, the `All files` coverage row, and "no linter configured" if `package.json` has no `lint` script.
- **If `.pr-butler/i18n-audit.mjs` does not exist**, write it now from the Step 1a heredoc. `.pr-butler/` is git-ignored, so a fresh clone never has it, and Gate 6 runs it even when Step 1 is skipped (a single-step or gates-only request).

Commit checkpoint: `build(test): add missing jsdom test environment`.

### Step 1: Translation Detection & Fix

**1a. Detect.** Write this auditor once, then run it. It compares every key, catches values copied untranslated from English, and catches interpolation placeholders that differ between the locales (a dropped `{{count}}` is a runtime bug, not a typo).

```bash
mkdir -p ../../.pr-butler && cat > ../../.pr-butler/i18n-audit.mjs << 'AUDIT'
import { readFileSync, existsSync } from 'node:fs';
// Tokens for i18next {{x}}, ICU {x}, Trans <0>, printf %s — must match across locales.
const TOKEN = /\{\{[^}]+\}\}|\{[^}{]+\}|<\/?\d+>|%[sd]/g;
// Keys deliberately identical across locales (product names, terms of art), one per line.
const IGNORE_FILE = new URL('./i18n-ignore.txt', import.meta.url);
const IGNORE = existsSync(IGNORE_FILE)
  ? new Set(readFileSync(IGNORE_FILE, 'utf8').split('\n').map((l) => l.split('#')[0].trim()).filter(Boolean))
  : new Set();
const flat = (o, p = '', out = new Map()) => {
  for (const [k, v] of Object.entries(o)) {
    const key = p ? `${p}.${k}` : k;
    if (v && typeof v === 'object' && !Array.isArray(v)) flat(v, key, out);
    else out.set(key, v);
  }
  return out;
};
const toks = (v) => (typeof v === 'string' ? (v.match(TOKEN) ?? []).sort() : []);
// Symbols, digits and bare tokens are legitimately identical across locales.
const sameIsFine = (v) => typeof v !== 'string' || !/[a-z]{3}/i.test(v.replace(TOKEN, '').trim());
const [refPath, targetPath] = process.argv.slice(2);
const ref = flat(JSON.parse(readFileSync(refPath, 'utf8')));
const target = flat(JSON.parse(readFileSync(targetPath, 'utf8')));
const d = { missing: [], empty: [], untranslated: [], placeholder: [], orphan: [] };
for (const [k, rv] of ref) {
  if (!target.has(k)) { d.missing.push(k); continue; }
  const v = target.get(k);
  if (typeof v !== 'string' || !v.trim()) { d.empty.push(k); continue; }
  if (v === rv && !sameIsFine(rv) && !IGNORE.has(k)) d.untranslated.push(`${k} = ${JSON.stringify(v)}`);
  if (toks(rv).join('\0') !== toks(v).join('\0'))
    d.placeholder.push(`${k}: expected ${toks(rv).join(' ') || '(none)'}, got ${toks(v).join(' ') || '(none)'}`);
}
for (const k of target.keys()) if (!ref.has(k)) d.orphan.push(k);
let total = 0;
for (const [label, list] of Object.entries(d)) {
  total += list.length;
  if (list.length) console.log(`\n${label} (${list.length})\n  ${list.join('\n  ')}`);
}
console.log(`\n${ref.size} reference keys, ${target.size} target keys, ${total} defect(s)`);
process.exit(total ? 1 : 0);
AUDIT
node ../../.pr-butler/i18n-audit.mjs src/translations/en.json src/translations/fr.json
```

On the unmodified scaffold, expect `missing (12)` and `14 reference keys, 2 target keys`. Keys here are **flat strings that contain dots** (`"priority.low"`), not nested objects, so keep them flat when editing.

**1b. Generate.** Translate each missing key from its English value, and review the two existing values against the same rules:

- **Sentence case.** French UI text does not use English Title Case, so `Mon Gestionnaire de Tâches` becomes `Mon gestionnaire de tâches`. Correcting an existing value counts as a fix; report it as "corrected", separately from "added".
- **Agreement.** *Tâche* is feminine, so filter and status labels that describe tasks agree with it: *Actives*, *Terminées*.
- **UI register.** Use idiomatic interface French rather than word-for-word translation: buttons are verbs (*Ajouter*, *Supprimer*) and placeholders are instructions (*Saisissez…*).
- **Termbase.** Keep terms consistent across keys: task → *tâche*, priority → *priorité* (low/medium/high → *basse / moyenne / haute*), completed → *terminée(s)*, delete → *supprimer*. Brand and technology names such as *TypeScript* stay as they are.
- Preserve any interpolation token verbatim, keep keys in the same order as `en.json`, and save the file as UTF-8 so the accents survive.
- A value that is correctly identical in both languages goes in `.pr-butler/i18n-ignore.txt` with a `# reason` comment, so it does not count as untranslated. The list is for proper nouns, not for unfinished work, and goes into the PR description.

**1c. Wire translations into the UI.** A complete `fr.json` on its own changes nothing a user can see. In the scaffold `t()` is defined but never called, `index.html` hardcodes English, and `switchLanguage()` carries the comment "Translation application is missing". Confirm with `grep -rnE '(^|[^[:alnum:]_.])t\(' src/*.ts index.html`: only the definition in `i18n.ts` should match (a bare `"t("` also matches every `getElementById(` and `set(`, so it proves nothing). Then:

1. **`index.html`**: add `data-i18n="<key>"` to every element whose text is an `en.json` value (h1, the "Add New Task" h2, the three `<option>`s, the submit button, the three filter buttons), and `data-i18n-placeholder="task.placeholder"` to the input. Where a label shares its element with a dynamic value, wrap the label in its own span:
   `<p><span data-i18n="stats.total">Total tasks</span>: <span id="total-count">0</span></p>` and
   `<p><span data-i18n="footer.text">Built with TypeScript</span> • 2026</p>`.
2. **`src/i18n.ts`**: add and export `applyTranslations(root: ParentNode = document)`. It sets `textContent` on every `[data-i18n]` element and the `placeholder` on every `[data-i18n-placeholder]` element, and sets `document.documentElement.lang` to the current language.
3. **`src/main.ts`**: `switchLanguage()` calls `applyTranslations()` and then `taskManager.render()`, so task rows re-render in the new language. Remove the "missing" comment.
4. **`src/taskManager.ts`**: the delete button uses `t('button.delete')` instead of the literal `'Delete'`.
5. **Leave as they are, and list them as follow-ups**: the language buttons *English* / *Français* (each language's own name for itself is correct), the "Your Tasks" heading and the uppercase priority badges (neither has a key, and Ground rule 5 forbids adding keys).

**1d. Verify.**

```bash
node ../../.pr-butler/i18n-audit.mjs src/translations/en.json src/translations/fr.json   # exit 0, target key count equals reference key count
npm run build
```

**Pass when** the auditor exits 0 with the target key count equal to the reference key count, the build passes, and switching to French re-renders every keyed element. Step 3's tests prove the last point automatically.

Commit checkpoint: `fix(i18n): add 12 missing French translations and apply them to the UI`.

### Step 2: Code Cleanup

**2a. Tooling (only if absent).** The scaffold ships no formatter, no linter and no `lint` script, so "zero lint errors" means nothing until one exists. Pin majors that support Node 18:

```bash
npm install -D prettier@^3 eslint@^9 @eslint/js@^9 typescript-eslint@^8 eslint-plugin-jsdoc@^50
npm pkg set scripts.lint="eslint src" scripts.format="prettier --write ." scripts.format:check="prettier --check ." scripts.typecheck="tsc --noEmit"
```

`.prettierrc.json` matches the style already dominant in the codebase (no semicolons, single quotes, no trailing commas, bare arrow params), so the formatting diff only touches genuinely inconsistent code. `endOfLine: "auto"` is essential here: Prettier's default of `"lf"` would rewrite every line of every CRLF file (Ground rule 9).

```json
{ "semi": false, "singleQuote": true, "trailingComma": "none", "arrowParens": "avoid", "endOfLine": "auto" }
```

`.prettierignore`: `dist`, `coverage`, `node_modules`, `package-lock.json`.

`eslint.config.js`:

```js
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import jsdoc from 'eslint-plugin-jsdoc'

export default tseslint.config(
  { ignores: ['dist', 'coverage', 'node_modules'] },
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    files: ['src/**/*.ts'],
    plugins: { jsdoc },
    rules: {
      '@typescript-eslint/no-unused-vars': [
        'error',
        { argsIgnorePattern: '^_' }
      ],
      'no-console': ['error', { allow: ['warn', 'error'] }],
      // User-controlled text reaching innerHTML is stored XSS; clear with replaceChildren().
      'no-restricted-syntax': [
        'error',
        {
          selector: "AssignmentExpression[left.property.name='innerHTML']",
          message:
            'innerHTML is an XSS risk. Use textContent, or replaceChildren() to clear.'
        }
      ],
      // Warn, not error: docstrings are Step 4's job, and Step 5 enforces --max-warnings 0.
      // enableFixer is off because the autofix inserts empty /** */ stubs that silence the rule.
      'jsdoc/require-jsdoc': [
        'warn',
        {
          require: {
            FunctionDeclaration: true,
            MethodDefinition: true,
            ClassDeclaration: true
          },
          checkConstructors: false,
          enableFixer: false
        }
      ],
      // A docstring must say something, and must document the parameters it claims to.
      'jsdoc/require-description': 'warn',
      'jsdoc/require-param': ['warn', { enableFixer: false }],
      'jsdoc/check-param-names': 'warn'
    }
  },
  // Tests set static DOM fixtures with innerHTML; no user input reaches them.
  {
    files: ['src/tests/**'],
    rules: { 'jsdoc/require-jsdoc': 'off', 'no-restricted-syntax': 'off' }
  }
)
```

**Why `enableFixer: false` matters:** by default `jsdoc/require-jsdoc` "autofixes" a missing docstring by inserting an empty `/** */`. Step 2c runs `lint --fix`, so without this setting the documentation gate would pass with no documentation written. `require-description` and `require-param` then reject empty or incomplete stubs. Tests are exempt only from *requiring* docstrings: a test helper that does have one must still document its parameters.

Run `npm run lint` once **before** fixing anything, and record the error count and the `jsdoc/require-jsdoc` warning count as the baselines for lint errors and undocumented functions. On the unmodified scaffold, expect `2 errors` (both `innerHTML`) and `20 warnings`.

**2b. Format.**

```bash
npm run format        # fixes handleSubmit() in main.ts, plus any other drift
npm run format:check  # must exit 0
```

**2c. Fix lint violations.** Run `npm run lint -- --fix`, then fix what remains by hand. Fix the **cause** every time:

- `innerHTML` in `render()`: user input goes into `text.innerHTML = task.text`, is saved to localStorage, and replays on every load, which makes it stored XSS. Use `textContent`, and clear the list with `taskList.replaceChildren()`.
- Unused variable or import: delete it. An unused parameter required by a callback signature gets a `_` prefix instead of being removed.
- `any` types: infer the real type. Fall back to `unknown` with a narrowing check, never to a cast.

**2d. Known items in the brief.**

- **`unusedVariable`**: run `grep -rn unusedVariable src`. `tsconfig.json` sets `noUnusedLocals`, so if `npm run build` passes, no unused local exists. Report "not present" (Ground rule 6).
- **`render()` is too long**: split it with no change in logic. Extract `getFilteredTasks(): Task[]` (public, since it is useful and testable on its own) and a private `createTaskElement(task: Task): HTMLLIElement`, so that `render()` reads as clear, build, append, update stats.
- **Repeated priority union**: `'low' | 'medium' | 'high'` appears in three files. Export `type Priority = Task['priority']` from `types.ts` and use it everywhere.

**2e. Hygiene and error handling.** These are small and each one is covered by a Step 3 test:

- **Secrets scan.** Search the source for anything that looks like a credential or an internal endpoint:
  ```bash
  grep -rnEi 'sk_(test|live)_|api[_-]?key|secret|passw(or)?d|bearer |BEGIN [A-Z ]*PRIVATE KEY|https?://[a-z0-9.-]*\.internal' src index.html
  ```
  The scaffold's `loadFromStorage()` contains a `temp auth: sk_test_…` comment and an internal API URL. Delete both lines. In reports, name the file and line but never repeat the value, and recommend rotating anything that was ever real.
- **Corrupt storage**: `loadFromStorage()` calls `JSON.parse` unguarded, so malformed or non-array data in localStorage crashes the app on load. Wrap it in `try/catch`, accept only an array, and otherwise start empty.
- **Unhandled startup rejection**: change `init()` to `init().catch(err => console.error('Failed to initialise app', err))`.
- **`handleSubmit()`**: return early if the input or select element is missing, and pass the **trimmed** text to `addTask`. The current code checks `trim()` and then stores the untrimmed value.

**2f. Verify.**

```bash
npm run format:check && npm run lint && npm run build && npm run test
```

**Pass when** the format check passes, lint reports 0 errors (the jsdoc warnings are left for Step 4), the build passes, and the existing tests still pass.

Commit checkpoint: `style: add prettier and eslint, fix formatting and lint violations`, then `fix: escape task text, guard storage parsing, refactor render()` if you commit the behavior fixes separately.

### Step 3: Test Automation

**3a. Run the existing suite.** Run `npm run test`. Fix failures before writing new tests: say whether each one was a regression from Steps 1–2 (fix the code) or a stale assertion (fix the test).

**3b. Enforce the threshold in config** so the gate outlives this run. In `vitest.config.ts`, under `test.coverage`:

```ts
reporter: ['text', 'json', 'json-summary', 'html'],
thresholds: { statements: 80, branches: 80, functions: 80, lines: 80 }
```

`json-summary` writes `coverage/coverage-summary.json`, which is where the reported numbers are read from. The thresholds make `npm run test:coverage` exit non-zero below 80%, so the gate is enforced by the tool and not just by the agent's say-so. Add `coverage/` to `.gitignore`.

**3c. Write tests that would fail if the behavior broke.** Use three files:

| File | Covers |
|---|---|
| `src/tests/taskManager.test.ts` (extend) | `addTask`, `toggleTask`, `deleteTask`, `setFilter`, `getFilteredTasks`, `render`, `saveToStorage`, `loadFromStorage`, stats |
| `src/tests/i18n.test.ts` (new) | `t`, `setLanguage`, `getCurrentLanguage`, `applyTranslations`, key parity |
| `src/tests/main.test.ts` (new) | `init`, `setupEventListeners`, `handleSubmit`, `switchLanguage` |

Required cases:

- **toggleTask**: flips `completed`; toggling twice restores it; an unknown id changes nothing.
- **deleteTask**: removes only the matching task; an unknown id is a no-op.
- **setFilter / getFilteredTasks**: `active`, `completed` and `all` each render exactly the right rows.
- **render**: returns quietly when `#tasks` is missing; builds one `li.task-item` per task with the `completed` class when done; checkbox `change` toggles the task; the delete button `click` removes it; the stats counters update; the delete label shows `Supprimer` under `fr`.
- **XSS regression**: a task whose text is `<img src=x onerror=alert(1)>` renders as literal text, and no `<img>` element appears.
- **saveToStorage / loadFromStorage** (private): test them through their observable effects, never with `as any`. Adding a task writes `localStorage.tasks`; a new `TaskManager` restores those tasks and continues ids from the highest one; malformed JSON and non-array JSON both start empty without throwing.
- **i18n parity**: every `en.json` key exists in `fr.json` with a non-empty value. This locks Step 1 into CI.
- **main.ts**: it calls `init()` when imported. Load the real `index.html` with `readFileSync(resolve(__dirname, '../../index.html'), 'utf8')`, parse it with `DOMParser`, and copy its body into `document`. Do **not** use `new URL(..., import.meta.url)`: under the jsdom environment `import.meta.url` is not a `file:` URL, and `readFileSync` throws `ERR_INVALID_URL_SCHEME`. Then call `vi.resetModules()`, `await import('../main')`, then flush microtasks with `await new Promise(r => setTimeout(r, 0))`. Assert that submitting adds a trimmed task and clears the input; that whitespace-only input is ignored; that clicking `#lang-fr` translates the heading, the placeholder and existing rows and sets `html[lang="fr"]`; and that clicking a filter button moves the `active` class and filters the list. Using the real `index.html` means a missing `data-i18n` attribute fails a test.

Never write a test that only asserts "renders without crashing". It raises coverage without guarding anything.

**3d. Measure and iterate.**

```bash
npm run test:coverage
node -e "const t=require('./coverage/coverage-summary.json').total; for (const k of ['statements','branches','functions','lines']) console.log(k, t[k].pct + '%')"
```

Target the uncovered lines listed in the text table until every metric is at least 80%.

**3e. Prove the tests catch regressions.** Coverage shows which lines ran, not whether any assertion would notice them breaking. Back up each file, reintroduce one original bug at a time, run `npx vitest run`, and restore the file:

| Reintroduced bug | Edit | Expect |
|---|---|---|
| Stored XSS | `text.textContent = task.text` → `text.innerHTML = task.text` in `taskManager.ts` | at least 1 failure |
| Translations not applied | delete the `applyTranslations()` call in `switchLanguage()` | at least 1 failure |
| Missing French key | delete the `"button.delete"` line from `fr.json` | at least 1 failure |

If a bug survives, the tests for that behavior are too weak: strengthen them and repeat. Confirm the suite is fully green after restoring. Report the results in the PR description.

**Pass when** all tests pass, all four metrics are at least 80%, the thresholds are committed in `vitest.config.ts`, and every reintroduced bug in 3e fails the suite.

Commit checkpoint: `test: cover task, i18n and UI behavior; enforce 80% coverage threshold`.

### Step 4: Documentation Updates

**4a. Docstrings.** Add JSDoc to every function and method in `src/` (tests excluded), starting with the nine the brief names:

- `taskManager.ts`: `addTask`, `toggleTask`, `deleteTask`, `setFilter`, `render`
- `main.ts`: `init`, `setupEventListeners`, `handleSubmit`, `switchLanguage`

Also document the class, `getTasks`, `getCompletedCount`, `getFilteredTasks`, the private helpers, and every export in `i18n.ts`. Each docstring states the purpose, `@param`, `@returns` and side effects (persists to storage, re-renders, mutates the DOM). Do not write docstrings that only repeat the signature.

```ts
/**
 * Flips a task between active and completed, then persists and re-renders.
 * Does nothing if no task has the given id.
 *
 * @param id - Id of the task to toggle.
 */
```

Verify with `npx eslint src --max-warnings 0`. It must exit 0: no missing, empty or parameter-less docstrings anywhere in `src/`, including test helpers that have one. Record the count you documented. On the scaffold it is 22: the 20 from the Step 2 baseline plus the two helpers extracted from `render()`.

**4b. `scaffold/website/README.md`.** Keep Setup and Build, and add **Features** (task CRUD, priorities, filters, stats, English/French switching, localStorage persistence), **Testing** (`npm run test`, `npm run test:coverage`, the 80% threshold, where the HTML report is written) and **Contributing** (setup, the gate commands a change must pass, Conventional Commits, adding a language: keep `en.json` as the reference and run the parity test). If a section already exists, update it in place rather than duplicating it.

Then check the README against reality, one claim at a time. **Run** every command it documents. **Trace** every behavioral claim to the code that implements it, because plausible-sounding prose is where docs go wrong. Two claims that were easy to write here and turned out false: "a missing translation fails the build" (the parity check is a *test*, and `npm run build` does not run tests), and "to add a language, register it in `i18n.ts` and add a button" (`setupEventListeners()` wires the language buttons by id, so a new one also needs code in `main.ts`).

**4c. `CHANGELOG.md` (repo root).** Use the Keep a Changelog format, under `## [Unreleased]` grouped into `Added / Changed / Fixed / Security`. The brief asks it to summarize **all** fixes, so list internal changes too (refactors, shared types, reformatting, docstrings) under `Changed`, not only what users notice. Word each entry for someone who has not read the diff. Build it from `git diff main --stat` and the actual hunks, not from memory. If the file exists, merge into `[Unreleased]` instead of adding a second entry.

**4d. `PR_REQUEST.md` (repo root).** Draft it now with the sections below. Write every number you already have (translations, coverage, test counts, all read from `state.json` and `coverage-summary.json`). Write the literal word `PENDING` wherever the value comes later: the gate results, the checklist and the commit message. Step 6 replaces every `PENDING`.

| Section | Contents |
|---|---|
| `# <title>` | The Step 6 commit subject |
| `## Summary` | 2–4 sentences, user-visible effect first |
| `## Changes` | One bullet each for Translations, Code quality, Security, Reliability, Tests and Docs, with counts |
| `## Coverage report` | A table of Statements, Branches, Functions, Lines and Tests, before and after, plus the 3e regression-check results |
| `## Quality gates` | One row per Step 5 gate, with its command and result |
| `## Checklist` | One `- [x]` line per Success Criteria item, ticked only if verified |
| `## How to verify` | The install and gate commands, then manual checks: switch to **Français** and confirm everything translates; add a task named `<img src=x onerror=alert(1)>` and confirm it shows as literal text |
| `## Follow-ups` | Strings with no translation key, `npm audit` advisories, and anything else deliberately left out of scope |
| `## Commit message` | The Step 6 message, verbatim, in a code block |

**4e. Re-format.** Step 4 adds text after Step 2's format pass, and long docstring or README lines will fail the Step 5 format gate. From `scaffold/website/`, run `npm run format`, then `npx prettier --write ../../CHANGELOG.md ../../PR_REQUEST.md` for the root deliverables, which sit outside the app's Prettier scope. Rerun `npx eslint src --max-warnings 0` and `npm run test` afterwards.

**Pass when** eslint reports zero docstring warnings, the README has all three new sections and every claim in it checks out, both root files exist, and everything is formatted.

Commit checkpoint: `docs: add JSDoc, expand README, add CHANGELOG and PR description`.

### Step 5: Quality Gates

From `scaffold/website/`, run **every** gate even after one fails, so the report is complete, and record each exit code:

| # | Gate | Command | Passes when |
|---|---|---|---|
| 1 | Build | `npm run build` | exit 0 |
| 2 | Format | `npm run format:check` | exit 0 |
| 3 | Lint | `npm run lint -- --max-warnings 0` | exit 0: 0 errors and 0 warnings, which also proves docstrings are complete |
| 4 | Tests | `npm run test` | exit 0, 0 failed, and no skipped tests you added |
| 5 | Coverage | `npm run test:coverage` | exit 0, **and** every metric >= 80, read from `coverage-summary.json` (or from the `All files` row if the summary reporter is not configured). Exit 0 alone is not enough, because without configured thresholds vitest exits 0 at any coverage |
| 6 | Translations | `node ../../.pr-butler/i18n-audit.mjs src/translations/en.json src/translations/fr.json` | exit 0, target key count equals reference key count |
| 7 | Secrets | the Step 2e `grep` | no matches |
| 8 | Deliverables | `grep -c '^## \(Features\|Testing\|Contributing\)' README.md` and `ls ../../CHANGELOG.md ../../PR_REQUEST.md` | 3, and both files exist |

**Advisory (reported, not blocking): `npm audit`.** Compare the advisory count against the original lockfile, so the report separates what this branch introduced from what was already there:

```bash
npm audit --package-lock-only --json | node -e "console.log(JSON.parse(require('fs').readFileSync(0,'utf8')).metadata.vulnerabilities)"
# baseline: run the same command in a temp dir holding `git show main:scaffold/website/package.json`
# and `git show main:scaffold/website/package-lock.json`
```

An advisory **this branch introduced** blocks: pick a different version of the tool you added. Pre-existing advisories go under Follow-ups in `PR_REQUEST.md` with their severity and fix version. Fixing them usually means major upgrades (on this scaffold, Vite 5 → 8 and Vitest 1 → 5), and that belongs in its own PR, not in pre-commit cleanup.

**If any gate fails: report and stop.** Do not start Step 6. For each failing gate, print its name, the command, the relevant lines of its output, and the most likely cause (for example: "Coverage: branches 76.9% < 80. `main.ts` lines 42–47, the missing-element guard in `handleSubmit`, is untested."). Then print the Report Card with Step 5 as `FAIL` and Step 6 as `FAIL (not attempted)`, and end the run. Never edit a threshold, config or test to turn a gate green at this stage.

Record each gate's result in `state.json`, and fill in the `PR_REQUEST.md` gate table.

### Step 6: PR Preparation

**6a. Review the diff.** Run `git status` and `git diff main --stat`. Confirm the change set contains no `dist/`, `coverage/`, `node_modules/`, `.pr-butler/`, `.env` or debug edits, and that `.gitignore` covers `coverage/` and `.pr-butler/`.

**6b. Write the commit message** in Conventional Commits form, and save it as `COMMIT_MSG.txt` at the repo root. It is a deliverable, so it gets committed and is not stored under the git-ignored `.pr-butler/`. It is the message for the PR as a whole, for use with `git commit -F` or as the squash-merge message:

```
<type>(<scope>): <imperative summary, at most 72 chars, no trailing period>

<body: what changed and why, wrapped at 72 columns>

<footer: BREAKING CHANGE: ... | Refs: #...>
```

Choose one type, the one that best describes the change as a whole: `fix` for corrected behavior, `feat` for a new capability, `test` or `docs` for changes that touch only tests or only docs. This run fixes broken translations and security defects, so it is a `fix`. The tests, docs and tooling go in the body, not the type.

Check the shape before moving on. The subject must be at most 72 characters, the second line blank, and no body line over 72 characters:

```bash
awk 'NR==1 && length($0)>72 {print "subject too long"} NR==2 && $0!="" {print "line 2 not blank"} NR>2 && length($0)>72 {print "line " NR " too long"}' ../../COMMIT_MSG.txt   # must print nothing
```

**6c. Finalize `PR_REQUEST.md`.** Replace every `PENDING` with the Step 5 gate results, tick only the checklist items you verified in Step 5, and paste the contents of `COMMIT_MSG.txt` verbatim into the `## Commit message` code block. The two copies must stay identical. Confirm nothing is left unfilled: `grep -n PENDING ../../PR_REQUEST.md` must print nothing. Re-run Prettier on the file afterwards.

**6d. Confirm all deliverables** exist and are current: `fr.json` (same key count as `en.json`), the source docstrings, `scaffold/website/README.md`, `CHANGELOG.md`, `PR_REQUEST.md` and `COMMIT_MSG.txt`. Check that the commit message matches the copy in the PR description:

```bash
diff <(tr -d '\r' < ../../COMMIT_MSG.txt) <(tr -d '\r' < ../../PR_REQUEST.md | sed -n '/^## Commit message/,$p' | sed -n '/^```$/,/^```$/p' | sed '1d;$d')   # must print nothing
```

**6e. Commit** only under Ground rule 8, staging explicit paths and never pushing. Tell the user the branch is ready for them to push.

- **Checkpoint mode** (a commit per step): commit `PR_REQUEST.md`, `CHANGELOG.md` and `COMMIT_MSG.txt` as a final `docs:` checkpoint. The earlier checkpoints already hold the code, so `COMMIT_MSG.txt` stays as the squash-merge message for the PR as a whole.
- **Single-commit mode**: stage every changed path, including `COMMIT_MSG.txt`, and run `git commit -F ../../COMMIT_MSG.txt`.

Then produce the Step 7 Report Card as the final output.

---

## Examples

### Example 1: Full PR Preparation

**Input:** "Make the scaffold PR-ready"

**Expected output:**

```
Step 0: Preflight
  Node v24.20.0 · npm install ok · baseline build passes
  vitest: MISSING DEPENDENCY 'jsdom' → installed jsdom@^24 (Node 18-compatible)
  Baseline: 2/2 tests · coverage 26.02% stmts, 69.23% branches, 57.14% funcs, 26.02% lines
  ✔ commit build(test): add missing jsdom test environment

Step 1: Translation Detection & Fix
  Audit: missing (12): task.placeholder, priority.low, priority.medium, priority.high,
         button.add, filter.all, filter.active, filter.completed, stats.total,
         stats.completed, button.delete, footer.text
  Added 12 French values; corrected 2 existing values from Title Case to sentence case
  Found t() never called, so translations were not being applied → added data-i18n
    markup, applyTranslations(), and a re-render in switchLanguage()
  Left as they are (no key; en.json is fixed at 14): "Your Tasks", priority badges, <title>
  Audit: 14 reference keys, 14 target keys, 0 defect(s) · build passes
  ✔ commit fix(i18n): add 12 missing French translations and apply them to the UI

Step 2: Code Cleanup
  No formatter or linter configured → added Prettier and ESLint (pinned, Node 18-compatible)
  Baseline lint: 2 errors (innerHTML ×2), 20 warnings (missing JSDoc)
  Formatted 5 source files; handleSubmit() now indented
  Fixed: stored XSS (innerHTML → textContent) · render() split into getFilteredTasks()
    and createTaskElement() · shared Priority type · corrupt-storage guard · init().catch
  Removed: credential-style comment at taskManager.ts:113, and the internal URL above it
  unusedVariable: not present (grep finds nothing; noUnusedLocals build passes)
  Lint: 0 errors · format:check passes · build passes · 2/2 tests
  ✔ commit style: add prettier and eslint, fix formatting and lint violations

Step 3: Test Automation
  Thresholds (80% on all four metrics) added to vitest.config.ts; run now exits 1 below them
  Tests: 2 → 45 across taskManager.test.ts, i18n.test.ts (new), main.test.ts (new)
  Coverage: statements 100%, branches 100%, functions 100%, lines 100%
  Regression check: XSS reintroduced → 1 failed · applyTranslations removed → 1 failed ·
    French key deleted → 5 failed · restored → 45/45 pass
  ✔ commit test: cover task, i18n and UI behavior; enforce 80% coverage threshold

Step 4: Documentation Updates
  JSDoc: 22 documented (all 9 from the brief, plus the class and the helpers)
  eslint --max-warnings 0: clean
  README: + Features, Testing, Contributing (every command run; claims traced to code)
  CHANGELOG.md: [Unreleased] Added / Changed / Fixed / Security
  PR_REQUEST.md: drafted, with gate results and commit message PENDING
  ✔ commit docs: add JSDoc, expand README, add CHANGELOG and PR description

Step 5: Quality Gates (8 run, 8 passed)
  Build ✅  Format ✅  Lint ✅ 0/0  Tests ✅ 45/45  Coverage ✅ 100%
  Translations ✅ 14/14  Secrets ✅  Deliverables ✅ 3/3 sections, 2/2 files
  Advisory: npm audit 8 (2 critical, 4 high, 2 moderate), identical to the original
    lockfile, so none were introduced · recorded as a follow-up (needs Vite 8 / Vitest 5)

Step 6: PR Preparation
  Diff reviewed: no dist/, coverage/, node_modules/ or .pr-butler/ tracked
  COMMIT_MSG.txt: fix: complete French UI, close stored XSS and restore quality gates
    (subject 67 chars, body at most 72 per line · identical to the PR_REQUEST.md copy)
  PR_REQUEST.md finalized: 0 PENDING left
  ✔ commit docs: finalize PR description and add commit message
  Branch ready to push; nothing pushed.

═══════════════════════════════════════════════
  PR BUTLER — REPORT CARD
═══════════════════════════════════════════════

  📋 Step 1: Translation Detection & Fix
     Status:  PASS
     Details: 12 of 14 French keys added to fr.json (2 existing corrected; all 14 applied to the UI)

  📋 Step 2: Code Cleanup
     Status:  PASS
     Details: 5 files formatted, 2 lint violations fixed

  📋 Step 3: Test Automation
     Status:  PASS
     Details: Coverage: 26.02% → 100%, 43 new test cases added

  📋 Step 4: Documentation Updates
     Status:  PASS
     Details: 22 functions documented, README updated: Y,
               CHANGELOG.md: Y, PR_REQUEST.md: Y

  📋 Step 5: Quality Gates
     Status:  PASS
     Details: Coverage ≥ 80%: Y, Lint clean: Y,
               All tests pass: Y

  📋 Step 6: PR Preparation
     Status:  PASS
     Details: Commit message: Y, PR_REQUEST.md finalized: Y

  ─────────────────────────────────────────────
  OVERALL:   6 / 6 steps passed
  GRADE:     A
═══════════════════════════════════════════════
```

### Example 2: Translation-Only Run

**Input:** "Fix the missing French translations"

**Expected output:**

```
Step 0: Preflight
  Baseline build passes · jsdom missing → installed jsdom@^24 · 2/2 tests

Step 1: Translation Detection & Fix
  Audit: missing (12) · 14 reference keys, 2 target keys
  Added 12, corrected 2 (sentence case) · applied to the UI via data-i18n and applyTranslations()
  Audit: 0 defect(s) · build passes · 2/2 tests

Step 5: Quality Gates (only the gates that apply to this step)
  Build ✅  Translations ✅ 14/14  Tests ✅ 2/2
  Not run, because out of scope for this request: Format, Lint, Coverage, Secrets, Deliverables

Only Step 1 was requested. Run "make the scaffold PR-ready" for the full workflow.

═══════════════════════════════════════════════
  PR BUTLER — REPORT CARD
═══════════════════════════════════════════════

  📋 Step 1: Translation Detection & Fix
     Status:  PASS
     Details: 12 of 14 French keys added to fr.json (2 existing corrected; applied to the UI)

  📋 Step 2: Code Cleanup
     Status:  FAIL
     Details: Not attempted (translation-only run)

  📋 Step 3: Test Automation
     Status:  FAIL
     Details: Not attempted (coverage still 26.02% from baseline, 0 new test cases)

  📋 Step 4: Documentation Updates
     Status:  FAIL
     Details: Not attempted: 0 functions documented, README updated: N,
               CHANGELOG.md: N, PR_REQUEST.md: N

  📋 Step 5: Quality Gates
     Status:  FAIL
     Details: Coverage ≥ 80%: N, Lint clean: N (no linter configured),
               All tests pass: Y

  📋 Step 6: PR Preparation
     Status:  FAIL
     Details: Not attempted: Commit message: N, PR_REQUEST.md finalized: N

  ─────────────────────────────────────────────
  OVERALL:   1 / 6 steps passed
  GRADE:     F
═══════════════════════════════════════════════
```

### Example 3: A Gate Fails

**Input:** "Run the PR Butler quality gates" on the unmodified scaffold. The gate results below are the real output of that run.

**Expected output:**

```
Step 0: Preflight
  Baseline build passes · jsdom missing → installed jsdom@^24 · 2/2 tests

Step 5: Quality Gates (8 run, 2 passed, 6 failed)
  1 Build         ✅ PASS  vite build ✓ built
  2 Format        ❌ FAIL  npm error Missing script: "format:check"
  3 Lint          ❌ FAIL  npm error Missing script: "lint"
  4 Tests         ✅ PASS  2 passed (2)
  5 Coverage      ❌ FAIL  exit 0, but statements 26.02%, branches 69.23%, functions 57.14%,
                           lines 26.02%. No thresholds are configured, so exit 0 proves nothing;
                           read the numbers.
  6 Translations  ❌ FAIL  missing (12) · 14 reference keys, 2 target keys, 12 defect(s)
  7 Secrets       ❌ FAIL  src/taskManager.ts:112 internal API URL
                           src/taskManager.ts:113 credential-style token (value withheld)
  8 Deliverables  ❌ FAIL  README sections 0/3 · CHANGELOG.md missing · PR_REQUEST.md missing

STOPPED at Step 5: 6 of 8 gates failed, so Step 6 was not attempted.
Likely causes, and the step that fixes each:
  Format, Lint    No formatter or linter is configured                    → Step 2
  Coverage        Only addTask and getCompletedCount have tests           → Step 3
  Translations    fr.json has 2 of 14 keys                                → Step 1
  Secrets         A stale TODO comment holds a token and an internal URL  → Step 2e;
                  rotate the token if it was ever real
  Deliverables    Docs not yet written                                    → Step 4
No threshold, config or test was changed to make a gate pass.

═══════════════════════════════════════════════
  PR BUTLER — REPORT CARD
═══════════════════════════════════════════════

  📋 Step 1: Translation Detection & Fix
     Status:  FAIL
     Details: 0 of 14 French keys added to fr.json (not attempted; 12 missing)

  📋 Step 2: Code Cleanup
     Status:  FAIL
     Details: 0 files formatted, 0 lint violations fixed (not attempted)

  📋 Step 3: Test Automation
     Status:  FAIL
     Details: Coverage: 26.02% → 26.02%, 0 new test cases added (not attempted)

  📋 Step 4: Documentation Updates
     Status:  FAIL
     Details: 0 functions documented, README updated: N,
               CHANGELOG.md: N, PR_REQUEST.md: N

  📋 Step 5: Quality Gates
     Status:  FAIL
     Details: Coverage ≥ 80%: N, Lint clean: N,
               All tests pass: Y

  📋 Step 6: PR Preparation
     Status:  FAIL
     Details: Commit message: N, PR_REQUEST.md finalized: N (not attempted: gates failed)

  ─────────────────────────────────────────────
  OVERALL:   0 / 6 steps passed
  GRADE:     F
═══════════════════════════════════════════════
```

---

## Success Criteria

Tick an item only once the named command confirms it in this run.

**Step 0: Preflight**
- [ ] Baseline build passes; baseline metrics recorded in `.pr-butler/state.json`
- [ ] Test environment runs (`jsdom` present, Node 18-compatible version)

**Step 1: Translations**
- [ ] All 14 French translation keys present in `fr.json`; the auditor exits 0
- [ ] French is sentence-cased, grammatically agreed, and uses consistent terms
- [ ] Translations are applied to the UI: `data-i18n` markup, `applyTranslations()`, and `switchLanguage()` re-renders

**Step 2: Code Cleanup**
- [ ] Code formatted consistently: `npm run format:check` exits 0; `handleSubmit()` is indented
- [ ] No lint violations: `npm run lint` reports 0 errors
- [ ] `render()` split into focused helpers; `unusedVariable` confirmed present-and-removed or reported absent
- [ ] Stored XSS fixed; credential comment removed; storage and startup errors handled

**Step 3: Tests**
- [ ] All tests pass
- [ ] Test coverage >= 80% on statements, branches, functions and lines, with thresholds in `vitest.config.ts`
- [ ] Every function named in the brief has tests that assert behavior
- [ ] Each bug reintroduced in the 3e regression check fails the suite

**Step 4: Documentation**
- [ ] JSDoc/TSDoc on all public functions: `npx eslint src --max-warnings 0` is clean
- [ ] `README.md` has Features, Testing and Contributing sections; each command was run and each claim traced to the code
- [ ] `CHANGELOG.md` generated
- [ ] `PR_REQUEST.md` generated with summary, checklist and coverage report

**Step 5: Quality Gates**
- [ ] All 8 gates pass in one run, after a fresh `npm install`
- [ ] `npm audit` shows no advisory introduced by this branch; pre-existing ones are listed as follow-ups

**Step 6: PR Preparation**
- [ ] Conventional commit message prepared in `COMMIT_MSG.txt` at the repo root and committed: subject at most 72 characters, line 2 blank, body wrapped at 72 (the 6b check prints nothing)
- [ ] `COMMIT_MSG.txt` is identical to the `## Commit message` block in `PR_REQUEST.md` (the 6d `diff` prints nothing)
- [ ] `PR_REQUEST.md` finalized with no placeholders left
- [ ] Diff contains only intended files; nothing pushed

---

<!-- ⚠️ DO NOT MODIFY ANYTHING BELOW THIS LINE ⚠️ -->
<!-- ================================================ -->
<!-- REPORT CARD — Auto-grading step                  -->
<!-- This step runs automatically after your Skill    -->
<!-- completes. It generates a progress report card   -->
<!-- so you can track how well your Skill is doing.   -->
<!-- ================================================ -->

## Step 7: Report Card (Do Not Modify)

After completing all steps above, generate a **Report Card** summarizing the results. Output the report card in the following exact format:

```
═══════════════════════════════════════════════
  PR BUTLER — REPORT CARD
═══════════════════════════════════════════════

  📋 Step 1: Translation Detection & Fix
     Status:  [PASS / FAIL]
     Details: [X of 14 French keys added to fr.json]

  📋 Step 2: Code Cleanup
     Status:  [PASS / FAIL]
     Details: [X files formatted, X lint violations fixed]

  📋 Step 3: Test Automation
     Status:  [PASS / FAIL]
     Details: [Coverage: X% → Y%, X new test cases added]

  📋 Step 4: Documentation Updates
     Status:  [PASS / FAIL]
     Details: [X functions documented, README updated: Y/N,
               CHANGELOG.md: Y/N, PR_REQUEST.md: Y/N]

  📋 Step 5: Quality Gates
     Status:  [PASS / FAIL]
     Details: [Coverage ≥ 80%: Y/N, Lint clean: Y/N,
               All tests pass: Y/N]

  📋 Step 6: PR Preparation
     Status:  [PASS / FAIL]
     Details: [Commit message: Y/N, PR_REQUEST.md finalized: Y/N]

  ─────────────────────────────────────────────
  OVERALL:   [X / 6 steps passed]
  GRADE:     [A / B / C / F]
             A = 6/6 passed
             B = 5/6 passed
             C = 4/6 passed
             F = 3 or fewer passed
═══════════════════════════════════════════════
```

**Grading rules:**
- A step passes only if ALL its success criteria are met
- Do not skip any step in the report — mark it FAIL if not attempted
- Be honest in the details — the evaluator will verify against actual file contents
- Output this report card as the very last thing your Skill does
