# angular-test — Claude Code Plugin

> Generate complete spec files · Fill coverage gaps · Fix failing tests
> Works on a **single file**, a **feature module**, or your **entire project**
> Supports Angular 14 · 15 · 16 · 17 · 18 · 19 · Jasmine · Jest · NgRx

---

## Table of Contents

- [At a Glance](#at-a-glance)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Which Command Do I Use?](#which-command-do-i-use)
- [Scope: Single File vs Directory vs Entire Project](#scope-single-file-vs-directory-vs-entire-project)
- [Commands](#commands)
  - [generate](#angular-testgenerate) — create spec files
  - [coverage](#angular-testcoverage) — fill coverage gaps
  - [fix](#angular-testfix) — repair failing tests
  - [suite](#angular-testsuite) — full project pipeline
- [All Flags — Quick Reference](#all-flags--quick-reference)
- [Supported Angular File Types](#supported-angular-file-types)
- [Auto-Trigger Skill](#auto-trigger-skill)
- [Test Engineer Agent](#test-engineer-agent)
- [Coverage Configuration](#coverage-configuration)
- [Troubleshooting](#troubleshooting)
- [Plugin Structure](#plugin-structure)
- [Changelog](#changelog)

---

## At a Glance

| Command | What it does | Scope |
|---------|-------------|-------|
| `/angular-test:generate` | Creates complete `.spec.ts` files from scratch | file · dir · `--all` |
| `/angular-test:coverage` | Finds untested lines and adds only the missing `it()` blocks | file · dir · `--all` |
| `/angular-test:fix` | Diagnoses and surgically repairs failing tests | error · file · dir · `--all` · log |
| `/angular-test:suite` | Runs all three phases in one pipeline pass | dir · `--all` |

---

## Installation

### Option A — Local project (recommended)

Copy the plugin into your Angular project root, then register it:

```bash
cp -r angular-test-plugin/ /path/to/your-angular-project/
```

**.claude/settings.json** in your project:

```json
{
  "plugins": ["./angular-test-plugin"]
}
```

```
/plugins reload
```

---

### Option B — Global (available in every project)

```bash
# macOS / Linux
cp -r angular-test-plugin/ ~/.claude/plugins/angular-test/

# Windows
xcopy /E /I angular-test-plugin %USERPROFILE%\.claude\plugins\angular-test\
```

**~/.claude/settings.json**:

```json
{
  "plugins": ["~/.claude/plugins/angular-test"]
}
```

---

### Option C — Team-shared via version control

Commit the plugin folder to your repo so every developer gets it automatically:

```
your-angular-project/
├── .claude-plugins/
│   └── angular-test/        ← commit this entire folder
├── .claude/
│   └── settings.json        ← reference it here
└── src/
```

**.claude/settings.json**:

```json
{
  "plugins": ["./.claude-plugins/angular-test"],
  "pluginSettings": {
    "angular-test": {
      "defaultTestRunner": "jest",
      "coverageTarget": 90,
      "batchDefaults": {
        "excludePatterns": ["src/app/legacy/**"]
      }
    }
  }
}
```

---

### Option D — Marketplace (future)

```bash
claude plugin install angular-test
```

---

## Quick Start

```bash
# NEW PROJECT — zero tests — run the full pipeline:
/angular-test:suite --dry-run        # preview what will happen
/angular-test:suite                  # run it for real

# SINGLE FILE — generate a spec:
/angular-test:generate src/app/users/user.service.ts

# FEATURE MODULE — generate all missing specs:
/angular-test:generate src/app/users/

# WHOLE PROJECT — generate everything missing:
/angular-test:generate --all

# EXISTING SPECS — find and fill coverage gaps:
/angular-test:coverage --all --threshold=90

# BROKEN TESTS — paste the error and get a fix:
/angular-test:fix "NullInjectorError: No provider for UserService!"

# BROKEN TESTS — scan and fix an entire module:
/angular-test:fix src/app/users/
```

---

## Which Command Do I Use?

```
I have a source file with no spec yet
  └─→ /angular-test:generate <file>

I have a directory / feature module with some specs missing
  └─→ /angular-test:generate <directory>

I have an entire project with little or no test coverage
  └─→ /angular-test:suite   (runs all three phases)
  OR  /angular-test:generate --all  (generate only)

I have existing specs but coverage is low
  └─→ /angular-test:coverage <file|directory|--all>

A test is failing with an error message
  └─→ /angular-test:fix "<error message>"
  OR  /angular-test:fix <spec-file>

Multiple tests are failing — I have a log file
  └─→ /angular-test:fix --from-log=test-output.log

I want to see what would happen without writing anything
  └─→ add --dry-run to any command
```

---

## Scope: Single File vs Directory vs Entire Project

Every command accepts three scope levels — you choose what you need:

| Scope | How to specify | Example |
|-------|---------------|---------|
| **Single file** | Full `.ts` path | `src/app/users/user.service.ts` |
| **Feature module** | Directory path | `src/app/users/` |
| **Entire project** | `--all` flag | `/angular-test:generate --all` |

### How batch mode behaves

When you pass a directory or `--all`, the plugin always:

1. **Discovers** — scans recursively, classifies every `.ts` file
2. **Plans** — prints exactly what will be processed and estimates test counts
3. **Confirms** — asks before doing anything (you can filter, cancel, or choose a subset)
4. **Executes** — processes files with a `[N/total]` progress counter
5. **Summarizes** — prints a completion report with next steps

### What gets auto-excluded from batch

The following are always skipped (no test value):

| Pattern | Reason |
|---------|--------|
| `*.module.ts` | Not testable units |
| `index.ts` barrel files | Re-exports only |
| `environment*.ts` | Configuration constants |
| `src/main.ts`, `src/polyfills.ts` | Bootstrap/platform code |
| `*.spec.ts` | Already specs |

To add your own exclusions in `.claude/settings.json`:

```json
{
  "pluginSettings": {
    "angular-test": {
      "batchDefaults": {
        "excludePatterns": [
          "src/app/legacy/**",
          "src/app/generated/**"
        ]
      }
    }
  }
}
```

---

## Commands

---

### /angular-test:generate

Generate **complete, ready-to-run `.spec.ts` files** from Angular source files.
Covers every method, branch, lifecycle hook, `@Input`, `@Output`, HTTP call,
NgRx action, and async path. Targets 100% coverage by default.

#### Syntax

```
/angular-test:generate <file-path|directory|--all> [flags]
```

#### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--dry-run` | `false` | Print discovery plan only — write nothing |
| `--skip-existing` | `true` | Skip source files that already have a `.spec.ts` |
| `--overwrite` | `false` | Regenerate specs even when one already exists |
| `--only=<types>` | all | Comma-separated type filter (see table below) |
| `--threshold=<n>` | `100` | Coverage target % written into config snippets |
| `--runner=jasmine\|jest` | `jasmine` | Syntax preference for generated test code |

**`--only` values:** `component`, `service`, `pipe`, `directive`, `guard`,
`resolver`, `interceptor`, `reducer`, `effect`, `selector`

#### Examples

```bash
# ── Single file ────────────────────────────────────────────────
/angular-test:generate src/app/users/user.service.ts
/angular-test:generate src/app/core/guards/auth.guard.ts
/angular-test:generate src/app/store/users/users.effects.ts

# ── Feature module ─────────────────────────────────────────────
/angular-test:generate src/app/users/
/angular-test:generate src/app/store/

# ── Whole project ──────────────────────────────────────────────
/angular-test:generate --all
/angular-test:generate --all --runner=jest
/angular-test:generate --all --dry-run           # preview only
/angular-test:generate --all --only=service,guard
/angular-test:generate --all --only=reducer,effect,selector

# ── Selective overwrite ────────────────────────────────────────
/angular-test:generate src/app/users/ --only=component --overwrite
```

#### Output: single file

```
1. Coverage plan table     — every testable unit listed before a line is written
2. Complete .spec.ts file  — copy-paste ready, Jasmine syntax + Jest notes inline
3. Jest migration notes    — when Jasmine-only APIs are used
4. Karma threshold config  — paste into karma.conf.js
5. Jest threshold config   — paste into jest.config.js
```

#### Output: directory / `--all`

```
1. Discovery table         — every source file with type + spec status
2. Confirmation prompt     — choose all / subset / specific type before anything runs
3. [N/total] spec file     — one fenced code block per file with progress indicator
4. Batch summary           — files created, total tests, skipped, next steps
```

#### Processing order (batch)

Specs are always generated dependency-first so earlier outputs can be referenced
by later ones:

```
1. NgRx state (reducer → selector → effect)
2. Services
3. Guards · Resolvers · Interceptors
4. Pipes · Directives
5. Components  ← depend on all of the above
```

---

### /angular-test:coverage

Analyse **existing** spec files against their source. Maps every untested branch,
method, and condition by line number. Generates **only the missing `it()` blocks** —
never touches, deletes, or rewrites existing tests.

#### Syntax

```
/angular-test:coverage <source-file|directory|--all> [spec-file] [flags]
```

`spec-file` is optional — when omitted, derived automatically
(`user.service.ts` → `user.service.spec.ts`).

#### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--threshold=<n>` | `90` | Minimum acceptable coverage %; files above are skipped |
| `--report=table\|json\|html` | `table` | Output format (`json` useful for CI pipelines) |
| `--only=<types>` | all | Filter to specific artifact types |
| `--skip-complete` | `true` | Skip files already at or above the threshold |
| `--dry-run` | `false` | Show gap plan — output no new `it()` blocks |

#### Examples

```bash
# ── Single file ────────────────────────────────────────────────
/angular-test:coverage src/app/users/user.service.ts
/angular-test:coverage src/app/users/user.service.ts src/app/users/user.service.spec.ts

# ── Feature module ─────────────────────────────────────────────
/angular-test:coverage src/app/users/
/angular-test:coverage src/app/users/ --dry-run   # see gaps, no output

# ── Whole project ──────────────────────────────────────────────
/angular-test:coverage --all
/angular-test:coverage --all --threshold=90
/angular-test:coverage --all --threshold=85 --only=component,service
/angular-test:coverage --all --report=json        # for CI / custom tooling
```

#### Example gap table output

```
COVERAGE GAP ANALYSIS — src/app/users/user.service.ts
══════════════════════════════════════════════════════════════
┌────────────────────────────────┬──────────┬───────────────────────────┐
│ Testable Unit                  │ Status   │ Missing scenario           │
├────────────────────────────────┼──────────┼───────────────────────────┤
│ L24  getUser(id) — happy path  │ COVERED  │ —                         │
│ L24  getUser(id) — HTTP 404    │ MISSING  │ error path                │
│ L24  getUser(id) — null id     │ MISSING  │ null input                │
│ L45  if (this.user) truthy     │ COVERED  │ —                         │
│ L45  if (this.user) falsy      │ MISSING  │ null user branch          │
│ L61  catchError handler        │ MISSING  │ error recovery            │
│ L14  @Output() userSelected    │ MISSING  │ EventEmitter emission     │
└────────────────────────────────┴──────────┴───────────────────────────┘
Summary  6 / 14 covered (42.9%)  →  +8 tests  →  ~100%
```

The generated `it()` blocks include the source line number in a comment so
you know exactly where to paste them:

```typescript
// NEW — L24 getUser(id) — HTTP 404 (paste into describe('UserService'))
it('should propagate HTTP 404 error', fakeAsync(() => {
  let err: any;
  service.getUser('x').subscribe({ error: (e) => (err = e) });
  httpMock.expectOne('/api/users/x').flush({}, { status: 404, statusText: 'Not Found' });
  tick();
  expect(err.status).toBe(404);
}));
```

#### Output: directory / `--all`

```
1. Inventory table         — all source/spec pairs, estimated coverage per file
2. Confirmation prompt     — choose scope before analysis runs
3. Per-file gap tables     — module-by-module with [N/total] progress
4. Module summary table    — current % → tests added → estimated new %
5. Project summary table   — aggregate before/after numbers
6. Threshold configs       — Karma, Jest, angular.json snippets
```

---

### /angular-test:fix

Diagnose and produce **minimal, surgical fixes** for failing Angular tests.
Never rewrites the whole spec — outputs only the exact lines that need changing,
with a BEFORE/AFTER diff and a plain-English explanation.

#### Syntax

```
/angular-test:fix <error-message|spec-file|directory|--all> [flags]
```

#### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--from-log=<path>` | — | Parse a saved `ng test` or `jest --verbose` / `jest --json` log |
| `--dry-run` | `false` | Print diagnosis only — output no fix code |
| `--only=<glob>` | — | Filter to spec files matching a glob pattern |
| `--group-by=error\|file` | `error` | How to organize project-wide output |

#### Examples

```bash
# ── Single error ───────────────────────────────────────────────
/angular-test:fix "NullInjectorError: No provider for UserService!"
/angular-test:fix "Can't bind to 'userId' since it isn't a known property"
/angular-test:fix "ExpressionChangedAfterItHasBeenCheckedError"
/angular-test:fix "Expected [] to equal [Object({ frame: 20 ... })]"

# ── Single spec file ───────────────────────────────────────────
/angular-test:fix src/app/users/user-list.component.spec.ts

# ── Feature module (static analysis) ──────────────────────────
/angular-test:fix src/app/users/
/angular-test:fix src/app/users/ --dry-run

# ── Whole project ──────────────────────────────────────────────
/angular-test:fix --all
/angular-test:fix --all --group-by=error   # group same errors together (default)
/angular-test:fix --all --group-by=file    # group by spec file

# ── From saved test output ─────────────────────────────────────
ng test --no-watch 2>&1 | tee test-output.log
/angular-test:fix --from-log=test-output.log
```

#### Handled error patterns

| # | Error | Root cause | Fix strategy |
|---|-------|-----------|-------------|
| 1 | `NullInjectorError: No provider for X` | Service / module missing from TestBed | Add `{ provide: X, useValue: mockX }` to `providers[]` |
| 2 | `Can't bind to 'X' since it isn't a known property` | Undeclared component or missing module | Stub component or add import |
| 3 | `ExpressionChangedAfterItHasBeenCheckedError` | `detectChanges()` timing | `fakeAsync` + `tick()` + second `detectChanges()` |
| 4 | Marble test `Expected [] to equal [Object...]` | Wrong marble string / unlinked `actions$` | Correct TestScheduler pattern |
| 5 | `1 timer(s) still in the queue` | Missing `tick()` / `discardPeriodicTasks()` | Proper `fakeAsync` cleanup |
| 6 | `Cannot use fakeAsync with real Promises` | `async/await` inside `fakeAsync` | Replace with `waitForAsync` + `flushMicrotasks()` |
| 7 | `TypeError: X is not a function` | Spy set up after injection | Move to `useValue` mock in `providers[]` |
| 8 | `NG0303: Can't bind to ngModel` | `FormsModule` not imported | Add `FormsModule` to `imports[]` |
| 9 | `NG0304: 'app-X' is not a known element` | Component not declared | Add declaration or stub |
| 10 | `Expected spy X to have been called` (but wasn't) | `detectChanges()` not called before assertion | Move `detectChanges()` before spy check |

#### Fix output format

For every issue:

```
ISSUE #1 — NullInjectorError: No provider for UserService
File    : src/app/users/user-list.component.spec.ts
Line    : 14 (inside beforeEach)

BEFORE (lines 10–16):
──────────────────────────────────────────────────────
  TestBed.configureTestingModule({
    declarations: [UserListComponent],
  });

AFTER:
──────────────────────────────────────────────────────
  const mockUserService = jasmine.createSpyObj('UserService',
    { getUsers: of([]), deleteUser: of(null) }
  );
  TestBed.configureTestingModule({
    declarations: [UserListComponent],
    providers: [{ provide: UserService, useValue: mockUserService }]
  });

Why this works:
  UserService is injected by UserListComponent but was never provided
  to the TestBed injector. The spy returns of([]) by default, satisfying
  all tests without making real HTTP calls.

Side effects to check:
  • Tests that call mockUserService.getUsers — verify the spy return value
    matches expected data in each test
```

#### Project-wide batch output

When using `--all`, issues are **grouped by error type** so you get one fix
template per pattern instead of the same fix repeated 8 times:

```
NullInjectorError — 4 occurrences across 3 files
  Fix template: [shown once]
  Apply to:
    features/users/user-list.component.spec.ts:14  (UserService)
    features/users/user-form.component.spec.ts:11  (FormBuilder → use real)
    features/products/product.service.spec.ts:8    (HttpClient → HttpClientTestingModule)
```

---

### /angular-test:suite

**The full pipeline in one command.** Takes a project or directory from its
current state to a well-covered, statically-clean test suite:

```
Phase 1 — Generate   all missing .spec.ts files
Phase 2 — Coverage   fill gaps in all spec files (existing + newly generated)
Phase 3 — Fix        detect and repair static issues across all spec files
         → Unified project health report with before/after coverage
```

#### Syntax

```
/angular-test:suite [directory] [flags]
```

If `directory` is omitted, the full `src/app/` is used.

#### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--only=<types>` | all | Filter to specific artifact types |
| `--threshold=<n>` | `90` | Coverage % target |
| `--dry-run` | `false` | Print full plan — write nothing |
| `--skip-generate` | `false` | Skip Phase 1 (useful when specs exist) |
| `--skip-coverage` | `false` | Skip Phase 2 |
| `--skip-fix` | `false` | Skip Phase 3 |
| `--runner=jasmine\|jest` | `jasmine` | Test runner syntax |
| `--overwrite` | `false` | Regenerate existing specs in Phase 1 |

#### Examples

```bash
# ── Full project ───────────────────────────────────────────────
/angular-test:suite
/angular-test:suite --dry-run            # see the full plan first

# ── One directory only ─────────────────────────────────────────
/angular-test:suite src/app/users/

# ── Filtered runs ──────────────────────────────────────────────
/angular-test:suite --only=component,service
/angular-test:suite --only=reducer,effect,selector

# ── Specific phases ────────────────────────────────────────────
/angular-test:suite --skip-generate      # coverage + fix only
/angular-test:suite --skip-coverage --skip-fix   # generate only

# ── Jest project ───────────────────────────────────────────────
/angular-test:suite --runner=jest --threshold=85
```

#### What `suite` produces step by step

```
Step 1  Project health check
        ├── Angular version detected from package.json
        ├── Test runner detected (Karma / Jest)
        ├── Required test packages verified (NgRx, jasmine-marbles, etc.)
        └── Current spec coverage inventory (source files vs spec files)

Step 2  Full discovery table
        └── Every source file classified by type, spec status shown

Step 3  Pipeline plan with estimates
        ├── Phase 1: N files to generate, ~X tests
        ├── Phase 2: M specs to analyze, ~Y new it() blocks
        ├── Phase 3: static issues to scan for
        └── Confirmation prompt (A = all / B = module-by-module / F = dry-run)

Phase 1  [N/total] progress per generated file

Phase 2  [N/total] progress per gap-analyzed file

Phase 3  Static issue scan → grouped fix output

Final   Unified report
        ├── Before / after table by module
        ├── Aggregate coverage estimate
        ├── Next steps (compile check, run ng test, open coverage/index.html)
        └── Ready-to-paste Karma + Jest + angular.json threshold configs
```

#### Example final report

```
╔══════════════════════════════════════════════════════════════════╗
║              ANGULAR TEST SUITE — PIPELINE COMPLETE             ║
╚══════════════════════════════════════════════════════════════════╝

┌─────────────────────────────┬──────────┬───────┬──────────┐
│ Module                      │ Est. Now │ Added │ Est. New │
├─────────────────────────────┼──────────┼───────┼──────────┤
│ core/services/              │ 62%      │ +12   │ ~95%     │
│ core/guards/                │ 0%       │ +15   │ ~95%     │
│ features/users/             │ 48%      │ +28   │ ~93%     │
│ features/products/          │ 0%       │ +42   │ ~91%     │
│ shared/                     │ 100%     │ +0    │ 100%     │
├─────────────────────────────┼──────────┼───────┼──────────┤
│ TOTAL                       │ ~28%     │ +97   │ ~93%     │
└─────────────────────────────┴──────────┴───────┴──────────┘
```

---

## All Flags — Quick Reference

| Flag | generate | coverage | fix | suite |
|------|:--------:|:--------:|:---:|:-----:|
| `--dry-run` | ✓ | ✓ | ✓ | ✓ |
| `--all` | ✓ | ✓ | ✓ | — |
| `--only=<types>` | ✓ | ✓ | ✓ | ✓ |
| `--runner=jasmine\|jest` | ✓ | — | — | ✓ |
| `--threshold=<n>` | ✓ | ✓ | — | ✓ |
| `--skip-existing` | ✓ | — | — | — |
| `--overwrite` | ✓ | — | — | ✓ |
| `--skip-complete` | — | ✓ | — | — |
| `--report=table\|json\|html` | — | ✓ | — | — |
| `--from-log=<path>` | — | — | ✓ | — |
| `--group-by=error\|file` | — | — | ✓ | — |
| `--skip-generate` | — | — | — | ✓ |
| `--skip-coverage` | — | — | — | ✓ |
| `--skip-fix` | — | — | — | ✓ |

---

## Angular Version Compatibility

The plugin **auto-detects** your Angular version from `package.json` and applies the
correct testing patterns automatically. You can also override it with `--ng=<version>`.

### What changes per version

| Feature | 14 | 15 | 16 | 17 | 18 | 19 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Module-based components (`declarations[]`) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Standalone components (`imports[]` in TestBed) | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| Functional guards (`CanActivateFn`) | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| Functional resolvers (`ResolveFn`) | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| Functional interceptors (`HttpInterceptorFn`) | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| `TestBed.runInInjectionContext()` for `inject()` | — | — | ✓ | ✓ | ✓ | ✓ |
| `@Input({ required: true })` — `setInput()` required | — | — | ✓ | ✓ | ✓ | ✓ |
| `DestroyRef` + `takeUntilDestroyed()` | — | — | ✓ | ✓ | ✓ | ✓ |
| `signal()`, `computed()`, `effect()` + `TestBed.flushEffects()` | — | — | ✓ | ✓ | ✓ | ✓ |
| `input()` signal inputs — `setInput()` only, no direct assign | — | — | — | ✓ | ✓ | ✓ |
| `output()` signal outputs — `subscribe()` not `spyOn(.emit)` | — | — | — | ✓ | ✓ | ✓ |
| `model()` two-way signal binding | — | — | — | ✓ | ✓ | ✓ |
| `@if` / `@for` / `@switch` new control flow | — | — | — | ✓ | ✓ | ✓ |
| `@defer` + `DeferBlockFixture` + `DeferBlockState` | — | — | — | ✓ | ✓ | ✓ |
| `afterNextRender` / `afterRender` lifecycle hooks | — | — | — | ✓ | ✓ | ✓ |
| `linkedSignal()` | — | — | — | — | preview | ✓ |
| `resource()` / `httpResource()` + `ResourceStatus` | — | — | — | — | preview | ✓ |
| `provideHttpClient()` + `provideHttpClientTesting()` | — | — | — | — | ✓ | ✓ |
| Zoneless (`provideExperimentalZonelessChangeDetection`) | — | — | — | — | preview | ✓ |
| Signal-based forms | — | — | — | — | — | preview |

### Version-specific testing rules summary

**Angular 14**
- Always use `declarations[]` — no standalone
- Class-based guards, resolvers, interceptors
- `@Input()` / `@Output()` decorators — direct property assignment
- `takeUntil(destroy$)` pattern for cleanup
- `*ngIf`, `*ngFor`, `*ngSwitch` template syntax
- `HttpClientTestingModule` in `imports[]`

**Angular 15**
- Standalone: use `imports: [MyComponent]` in TestBed
- Functional guards/resolvers: call via `TestBed.runInInjectionContext()`
- Functional interceptors: `provideHttpClient(withInterceptors([fn]))`

**Angular 16**
- Required inputs: always use `fixture.componentRef.setInput()` before `detectChanges()`
- Signal preview: `signal()`, `computed()`, `effect()` — use `TestBed.flushEffects()`
- `DestroyRef`: `fixture.destroy()` triggers `onDestroy` callbacks

**Angular 17**
- Signal inputs `input()`: **never assign directly** — always `setInput()`
- Signal outputs `output()`: **never `spyOn(.emit)`** — use `.subscribe()`
- `model()`: test read with `()`, write with `setInput()` or `.set()`
- `@if` / `@for` / `@switch`: same `By.css()` queries — just signal/property driven
- `@defer`: use `getDeferBlocks()` + `deferBlock.render(DeferBlockState.X)`

**Angular 18**
- `httpResource()`: use `provideHttpClient()` + `provideHttpClientTesting()`; flush via `HttpTestingController`
- `resource()`: assert `.status()` (Loading → Resolved/Error) and `.value()` signals
- Zoneless: add `provideExperimentalZonelessChangeDetection()`; use `await fixture.whenStable()` after signal changes
- `HttpClientTestingModule` is deprecated — migrate to `provideHttpClientTesting()`

**Angular 19**
- `provideZonelessChangeDetection()` (stable, renamed from Experimental)
- `resource()` and `httpResource()` are stable — same test patterns as v18
- Signal forms (experimental) — test via `.fields.x.set()` and `.valid()`

---

## Supported Angular File Types

| File Type | Detection signal | TestBed pattern | Key assertions |
|-----------|-----------------|-----------------|----------------|
| **Component** (module) | `@Component` · `standalone` absent/false | `declarations: [C]` | `By.css()`, `triggerEventHandler`, `Input/Output` |
| **Component** (standalone) | `@Component({ standalone: true })` | `imports: [C]` · `setInput()` | Same + signal-safe APIs |
| **Service** (HTTP) | `@Injectable` + `HttpClient` in constructor | `HttpClientTestingModule` | `expectOne`, `flush`, `verify()` |
| **Service** (pure) | `@Injectable`, no HTTP | Direct `new` or TestBed | Return values, side effects |
| **Pipe** | `@Pipe` | Direct `new Pipe().transform()` | All input categories + edge cases |
| **Directive** | `@Directive` | Host wrapper component | `By.directive()`, host element state |
| **Guard** (class) | `CanActivate` interface | Mock Router + service | Allow / block / redirect paths |
| **Guard** (functional) | `CanActivateFn` | Direct function call | Same paths, no TestBed needed |
| **Resolver** (class) | `Resolve<T>` | Mock service + ActivatedRoute | Resolved value + error |
| **Resolver** (functional) | `ResolveFn<T>` | Direct function call | Same |
| **Interceptor** (class) | `HttpInterceptor` | `HttpClientTestingModule` | Headers added, pass-through, error |
| **Interceptor** (functional) | `HttpInterceptorFn` | `provideHttpClient(withInterceptors([fn]))` | Same |
| **NgRx Reducer** | `createReducer` | None — pure function | Every `on()` handler, initial state |
| **NgRx Effect** | `createEffect` | `TestScheduler` marble testing | Happy path, error path, side effects |
| **NgRx Selector** | `createSelector` | `projector()` direct call | Derived value, memoisation |

---

## Auto-Trigger Skill

The `angular-testing` skill **activates automatically** when you:

- Open or edit any `.spec.ts` file
- Write Angular tests in a conversation
- Run or discuss `ng test` output
- Work with `TestBed`, `HttpClientTestingModule`, `MockStore`, marble testing
- Mention `NullInjectorError`, `fakeAsync`, `fixture.detectChanges`, coverage %

No setup needed. When active it silently enforces **15 best-practice rules**:

| # | Rule |
|---|------|
| 1 | Always read the source file before writing tests |
| 2 | Use `declarations[]` (module) or `imports[]` (standalone) correctly |
| 3 | Call `fixture.detectChanges()` inside each `it()`, never in `beforeEach` |
| 4 | Mock all external deps via `createSpyObj` / `jest.fn()` — no real services |
| 5 | Use `HttpTestingController` for all HTTP — never real network |
| 6 | Call `httpMock.verify()` in every `afterEach` |
| 7 | Use `fakeAsync` + `tick()` for timers and debounce — not `done` callbacks |
| 8 | Use `TestScheduler` marble testing for NgRx effects |
| 9 | Query DOM with `By.css()` / `By.directive()` — not `document.querySelector` |
| 10 | Use `NO_ERRORS_SCHEMA` sparingly and always with a justification comment |
| 11 | Test subscription cleanup in `ngOnDestroy` |
| 12 | Test every `@Input()` binding and every `@Output()` emission |
| 13 | Test all Reactive Forms states: valid · invalid · submitted |
| 14 | Use `fixture.componentRef.setInput()` for Angular 16+ signal-compatible inputs |
| 15 | Enforce coverage thresholds via `karma.conf.js` / `jest.config.js` |

---

## Test Engineer Agent

For deep TDD sessions and whole-module test strategy work, invoke the specialized
`test-engineer` agent (senior Angular test engineer):

```bash
# Full test suite for a feature module
/agent test-engineer write all tests for src/app/users/

# TDD from a user story
/agent test-engineer set up TDD for the checkout feature

# Comprehensive coverage audit
/agent test-engineer full test coverage audit for src/app/store/

# Whole project strategy
/agent test-engineer plan my test suite for the entire project
```

The agent follows a 4-phase workflow:

| Phase | Output |
|-------|--------|
| **Discovery** | Reads all files, builds dependency graph, estimates test count per file |
| **Strategy** | Unit vs integration split, mock strategy, coverage targets, spec ordering |
| **Execution** | Generates spec files in dependency order with full content |
| **CI/CD** | `package.json` test scripts + GitHub Actions workflow config |

**Agent rules:** never guesses at DI dependencies · always tests unhappy paths ·
always verifies HTTP mocks · prefers `fakeAsync` over `done` · uses marble testing
for all effects.

---

## Coverage Configuration

### Karma — `karma.conf.js`

```javascript
module.exports = function(config) {
  config.set({
    reporters: ['progress', 'coverage'],
    coverageReporter: {
      type: 'html',
      dir: require('path').join(__dirname, './coverage'),
      subdir: '.',
      check: {
        global: {
          statements: 90,
          branches:   85,
          functions:  90,
          lines:      90
        },
        each: {
          statements: 80,
          branches:   75,
          functions:  80,
          lines:      80
        }
      }
    }
  });
};
```

```bash
ng test --code-coverage
ng test --no-watch --code-coverage --browsers=ChromeHeadless   # CI
```

### Jest — `jest.config.js`

```javascript
module.exports = {
  preset: 'jest-preset-angular',
  setupFilesAfterFramework: ['<rootDir>/setup-jest.ts'],
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['html', 'lcov', 'text-summary'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.spec.ts',
    '!src/**/*.module.ts',
    '!src/main.ts',
    '!src/polyfills.ts',
    '!src/environments/**',
    '!src/**/index.ts'
  ],
  coverageThreshold: {
    global: {
      statements: 90,
      branches:   85,
      functions:  90,
      lines:      90
    }
  }
};
```

```bash
npx jest --coverage
npx jest --watch
```

### `angular.json`

```json
{
  "projects": {
    "your-app": {
      "architect": {
        "test": {
          "options": {
            "codeCoverage": true,
            "codeCoverageExclude": [
              "src/environments/**",
              "src/main.ts",
              "src/**/*.module.ts",
              "src/**/index.ts"
            ]
          }
        }
      }
    }
  }
}
```

---

## Troubleshooting

### Quick error lookup

| Symptom | Angular | Jump to |
|---------|---------|---------|
| `NullInjectorError: No provider for X` | all | [Issue 1](#issue-1-nullinjectorerror-no-provider-for-x) |
| Tests pass locally, fail in CI | all | [Issue 2](#issue-2-tests-pass-locally-but-fail-in-ci) |
| `1 timer(s) still in the queue` | all | [Issue 3](#issue-3-fakeAsync--1-timers-still-in-the-queue) |
| Marble tests always pass trivially | all | [Issue 4](#issue-4-marble-tests-always-pass-trivially) |
| Coverage shows 0% for some files | all | [Issue 5](#issue-5-coverage-report-shows-0-for-some-files) |
| Batch generates too many files at once | all | [Issue 6](#issue-6-batch-generates-too-many-files-at-once) |
| `NG0950: Required input must be specified` | 16+ | [Issue 7](#issue-7-ng0950-required-input-must-be-specified-v16) |
| `TypeError: Cannot set property` on signal input | 17+ | [Issue 8](#issue-8-typeerror-cannot-set-property-on-signal-input-v17) |
| `NG0203: inject() must be called from injection context` | 16+ | [Issue 9](#issue-9-ng0203-inject-must-be-called-from-injection-context-v16) |
| `getDeferBlocks()` returns `[]` / `@defer` not rendering | 17+ | [Issue 10](#issue-10-getdeferblocks-returns---defer-not-rendering-v17) |
| `output()` cannot be subscribed / TypeError on `.emit` | 17+ | [Issue 11](#issue-11-output-cannot-be-subscribed--typeerror-on-emit-v17) |
| `httpResource()` value stuck at `undefined` / Loading | 18+ | [Issue 12](#issue-12-httpresource-value-undefined--loading-v18) |
| `HttpClientTestingModule` deprecated warning | 18+ | [Issue 13](#issue-13-httpclienttestingmodule-deprecated-v18) |

---

### Issue 1: `NullInjectorError: No provider for X!`

Provide the missing service as a spy object in TestBed:

```typescript
const mockUserService = jasmine.createSpyObj('UserService', {
  getUser: of(mockUser),
  saveUser: of(null)
});

TestBed.configureTestingModule({
  providers: [{ provide: UserService, useValue: mockUserService }]
});
```

| Missing | Solution |
|---------|----------|
| `HttpClient` | Add `HttpClientTestingModule` to `imports[]` |
| `Store` | Add `provideMockStore({ initialState })` to `providers[]` |
| `Router` | Add `jasmine.createSpyObj('Router', ['navigate'])` to `providers[]` |
| Any service | Add `{ provide: X, useValue: mockX }` to `providers[]` |

Automated fix:
```bash
/angular-test:fix "NullInjectorError: No provider for UserService!"
```

---

### Issue 2: Tests pass locally but fail in CI

```javascript
// karma.conf.js — detect CI environment:
browsers: process.env.CI ? ['ChromeHeadless'] : ['Chrome'],
customLaunchers: {
  ChromeHeadless: {
    base: 'Chrome',
    flags: [
      '--no-sandbox',
      '--headless',
      '--disable-gpu',
      '--disable-dev-shm-usage'
    ]
  }
}
```

CI run command:
```bash
ng test --no-watch --code-coverage --browsers=ChromeHeadless
```

---

### Issue 3: `fakeAsync` — `1 timer(s) still in the queue`

```typescript
it('should poll every 5s', fakeAsync(() => {
  fixture.detectChanges();
  tick(5000);
  expect(mockService.poll).toHaveBeenCalledTimes(1);
  discardPeriodicTasks();  // ← clears setInterval
  flush();                  // ← clears remaining one-shot timers
}));
```

---

### Issue 4: Marble tests always pass trivially

The most common cause: `actions$` is not wired to the effect instance.

```typescript
scheduler.run(({ hot, cold, expectObservable }) => {
  const actions = hot('-a', { a: loadUsers() });

  // ← REQUIRED: wire actions$ before expectObservable
  (effects as any).actions$ = actions;

  serviceSpy.getAll.and.returnValue(cold('-b|', { b: mockData }));
  expectObservable(effects.loadUsers$).toBe('--c', {
    c: loadUsersSuccess({ users: mockData })
  });
});
```

---

### Issue 5: Coverage report shows 0% for some files

1. Run `ng test --code-coverage` — the `--code-coverage` flag is required
2. For Jest: verify `collectCoverageFrom` includes the affected paths
3. For Karma: verify `preprocessors` covers source files:
   ```javascript
   preprocessors: { 'src/**/!(*.spec).ts': ['coverage'] }
   ```
4. Check `angular.json` `codeCoverageExclude` — the file may be excluded

---

### Issue 7: `NG0950: Required input must be specified` *(v16+)*

A component has `@Input({ required: true })` or `input.required()` and you called
`detectChanges()` before setting the input.

```typescript
// ❌ WRONG — detectChanges before required input:
fixture.detectChanges();                          // throws NG0950

// ✅ CORRECT — set input first:
fixture.componentRef.setInput('userId', '42');   // required input satisfied
fixture.detectChanges();                          // safe
```

Run the automated fix:
```bash
/angular-test:fix "NG0950: Required input 'userId' from component must be specified"
```

---

### Issue 8: `TypeError: Cannot set property` on signal input *(v17+)*

Signal inputs created with `input()` are read-only properties — you cannot assign
to them directly.

```typescript
// ❌ WRONG — direct assignment to signal input:
component.userName = 'Alice';    // TypeError or compile error

// ✅ CORRECT — use setInput():
fixture.componentRef.setInput('userName', 'Alice');
fixture.detectChanges();
expect(component.userName()).toBe('Alice');  // read signal with ()
```

---

### Issue 9: `NG0203: inject() must be called from an injection context` *(v16+)*

```typescript
// ❌ WRONG — inject() inside it() body:
it('should work', () => {
  const svc = inject(UserService);   // NG0203
});

// ✅ CORRECT — wrap with runInInjectionContext:
it('should work', () => {
  const svc = TestBed.runInInjectionContext(() => inject(UserService));
  expect(svc).toBeDefined();
});

// For functional guards/resolvers — same pattern:
const result = TestBed.runInInjectionContext(() =>
  authGuard({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
);
```

---

### Issue 10: `getDeferBlocks()` returns `[]` / `@defer` not rendering *(v17+)*

`@defer` blocks are not rendered automatically — you must trigger each state manually.

```typescript
// ❌ WRONG — querying deferred content without rendering the block:
fixture.detectChanges();
expect(fixture.debugElement.query(By.css('app-chart'))).toBeTruthy();  // null

// ✅ CORRECT — use DeferBlockFixture:
import { DeferBlockFixture, DeferBlockState } from '@angular/core/testing';

fixture.detectChanges();
const deferBlocks = await fixture.getDeferBlocks();
const [deferBlock] = deferBlocks;
await deferBlock.render(DeferBlockState.Complete);
expect(fixture.debugElement.query(By.css('app-chart'))).toBeTruthy();
```

If `getDeferBlocks()` still returns `[]`, verify the component template actually
contains a `@defer` block and that `fixture.detectChanges()` was called first.

---

### Issue 11: `output()` cannot be subscribed / TypeError on `.emit` *(v17+)*

Signal outputs (`output()`) are `OutputRef` — not `EventEmitter`. Never call `.emit()`
directly or use `spyOn(.emit)`.

```typescript
// ❌ WRONG — treating output() like EventEmitter:
const spy = spyOn(component.userSelected, 'emit');   // TypeError

// ✅ CORRECT — subscribe to the OutputRef:
const emitted: User[] = [];
component.userSelected.subscribe((u: User) => emitted.push(u));

fixture.componentRef.setInput('user', mockUser);
fixture.detectChanges();
fixture.debugElement.query(By.css('.card')).triggerEventHandler('click', null);

expect(emitted.length).toBe(1);
expect(emitted[0]).toEqual(mockUser);
```

---

### Issue 12: `httpResource()` value `undefined` / stuck at Loading *(v18+)*

`httpResource()` goes through `HttpClient` — you must flush the request via
`HttpTestingController`.

```typescript
// ❌ WRONG — asserting before flushing the HTTP request:
fixture.detectChanges();
expect(component.usersResource.value()).toEqual(mockUsers);  // undefined — not flushed

// ✅ CORRECT — flush the HTTP request first:
fixture.detectChanges();                                       // triggers httpResource load
const req = httpMock.expectOne('/api/users');
req.flush(mockUsers);
tick();
fixture.detectChanges();
expect(component.usersResource.value()).toEqual(mockUsers);   // resolved
```

Also ensure you are using `provideHttpClient()` + `provideHttpClientTesting()` (not
`HttpClientTestingModule`) in the TestBed providers for `httpResource()`.

---

### Issue 13: `HttpClientTestingModule` deprecated warning *(v18+)*

Angular 18+ deprecates `HttpClientTestingModule` in favour of functional providers.

```typescript
// BEFORE (still works but deprecated in v18+):
imports: [HttpClientTestingModule]

// AFTER (preferred for v18+):
providers: [
  provideHttpClient(),
  provideHttpClientTesting()
]
```

Run the automated fix across your project:
```bash
/angular-test:fix --all --only="*.spec.ts"
```

---

### Issue 6: Batch generates too many files at once

Filter to one artifact type at a time:

```bash
/angular-test:generate --all --only=service
/angular-test:generate --all --only=component
/angular-test:generate --all --only=reducer,effect,selector
```

Or preview the full list first without writing anything:

```bash
/angular-test:generate --all --dry-run
```

Or process one module at a time:

```bash
/angular-test:suite src/app/core/
/angular-test:suite src/app/features/users/
/angular-test:suite src/app/features/products/
```

---

## Plugin Structure

```
angular-test-plugin/
│
├── .claude-plugin/
│   └── plugin.json              ← plugin metadata, command registry (v2.0.0)
│
├── .claude/
│   └── settings.json            ← ready-to-use project settings
│
├── commands/
│   ├── generate-tests.md        ← /angular-test:generate
│   │                               single file · directory · --all
│   │                               --dry-run · --skip-existing · --only · --runner
│   │
│   ├── coverage-report.md       ← /angular-test:coverage
│   │                               gap analysis · missing it() blocks only
│   │                               --threshold · --report=json · --dry-run
│   │
│   ├── fix-tests.md             ← /angular-test:fix
│   │                               10 error patterns · static analysis
│   │                               --from-log · --group-by · --all
│   │
│   └── project-suite.md         ← /angular-test:suite
│                                   3-phase pipeline · health check
│                                   unified project report · --skip-* flags
│
├── skills/
│   └── angular-testing/
│       └── SKILL.md             ← auto-trigger skill (15 rules)
│                                   activates on *.spec.ts, ng test, TestBed, etc.
│
├── agents/
│   └── test-engineer.md         ← TDD agent
│                                   4-phase workflow · never guesses DI
│
└── README.md                    ← this file
```

---

## Changelog

### v3.0.0
- **New:** Full Angular 14–19 version matrix — auto-detects version from `package.json`
- **New:** `--ng=<version>` flag to override detected version on all commands
- **New:** Angular 15 patterns — standalone TestBed setup, functional guards/resolvers via `TestBed.runInInjectionContext()`, functional interceptors via `provideHttpClient(withInterceptors([]))`
- **New:** Angular 16 patterns — `@Input({ required: true })` + `setInput()`, `signal()`/`computed()`/`effect()` + `TestBed.flushEffects()`, `DestroyRef`, `inject()` context testing
- **New:** Angular 17 patterns — `input()` signal inputs (setInput only), `output()` OutputRef (subscribe not spyOn), `model()`, `@if`/`@for`/`@switch` control flow, `@defer` + `DeferBlockFixture` + `DeferBlockState`, `afterNextRender`/`afterRender`
- **New:** Angular 18 patterns — `linkedSignal()`, `resource()`/`httpResource()` + `ResourceStatus` signals, `provideHttpClient()` + `provideHttpClientTesting()`, zoneless `provideExperimentalZonelessChangeDetection()` + `await fixture.whenStable()`
- **New:** Angular 19 patterns — stable `resource()`/`httpResource()`, `provideZonelessChangeDetection()`, signal forms (experimental)
- **New:** 8 new error patterns in `/angular-test:fix` (NG0950, NG0203, signal input assignment, `output()` misuse, `@defer` rendering, `httpResource()` flushing, `HttpClientTestingModule` deprecation)
- **New:** Version compatibility matrix table in README
- **New:** Version-specific troubleshooting section (Issues 7–13)
- **Updated:** SKILL.md — 17 rules covering all version-specific patterns, `user-invocable: false` only (removed unsupported `triggers`/`autoTrigger` front matter attributes)
- **Updated:** `plugin.json` — added Angular 19, updated description

### v2.0.0
- **New:** all three commands now accept a directory path or `--all` for project-wide batch mode
- **New:** `/angular-test:suite` — full 3-phase pipeline orchestration command
- **New:** `--dry-run` flag on all commands (plan without writing)
- **New:** `--only=<types>` filter on all commands
- **New:** `--from-log=<path>` on `fix` — parses saved `ng test` / `jest` output
- **New:** `--report=json|html` on `coverage` — machine-readable output for CI
- **New:** `--group-by=error|file` on `fix` — groups project-wide issues by error type
- **New:** project health check in `suite` — detects Angular version, test runner, missing packages
- **New:** processing order in batch (NgRx → Services → Guards → Components)
- **Improved:** `fix` now performs static analysis across whole directories without needing a live test run

### v1.0.0
- Initial release: `generate`, `coverage`, `fix` — single file mode only
- Auto-trigger `angular-testing` skill (15 rules)
- `test-engineer` TDD agent

---

## Contributing

Issues and pull requests:
`https://github.com/your-org/angular-test-plugin`

## License

MIT
