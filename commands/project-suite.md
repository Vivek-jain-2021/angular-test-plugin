---
name: angular-test:suite
description: >
  Full project orchestration command. Runs the complete testing pipeline in one pass:
  discover all Angular source files → generate missing specs → fill coverage gaps →
  fix detectable failures → output a unified project health report. The entry point
  for "set up testing for my entire project". Supports dry-run, module filtering,
  and threshold configuration.
argument-hint: [directory] [--only=<types>] [--threshold=<n>] [--dry-run] [--skip-generate] [--skip-coverage] [--skip-fix]
---

# /angular-test:suite

You are a senior Angular test engineer running a **full project test pipeline**.
This command orchestrates all three sub-commands in sequence for an entire project
or directory — the single command a developer needs to go from zero tests to a
well-covered, passing suite.

---

## ARGUMENT PARSING

Parse `$ARGUMENTS`:

| Input | Meaning |
|---|---|
| *(empty)* | Run on entire project (`src/app/`) |
| `src/app/users/` | Run on a specific directory |
| `--only=component,service` | Filter to specific artifact types |
| `--threshold=<n>` | Coverage % target (default: 90) |
| `--dry-run` | Plan only — write nothing |
| `--skip-generate` | Skip Phase 1 (spec generation) |
| `--skip-coverage` | Skip Phase 2 (gap filling) |
| `--skip-fix` | Skip Phase 3 (failure fixing) |
| `--runner=jasmine\|jest` | Test runner syntax preference (default: jasmine) |

---

## STEP 1 — PROJECT HEALTH CHECK

Before any action, read the project configuration files and produce a health snapshot.

### Read these files (if they exist):
- `angular.json` — project name, source root, test runner config
- `package.json` — Angular version, installed test dependencies
- `karma.conf.js` — current coverage thresholds (Karma)
- `jest.config.js` / `jest.config.ts` — current coverage thresholds (Jest)
- `tsconfig.spec.json` — TypeScript spec compiler options

### Output the health snapshot:

```
╔══════════════════════════════════════════════════════════════════╗
║           ANGULAR TEST SUITE — PROJECT HEALTH CHECK             ║
╚══════════════════════════════════════════════════════════════════╝

Project         : my-angular-app
Angular version : 17.3.0
Source root     : src/app/
Test runner     : Karma + Jasmine
  (jest detected? No)

Dependencies check:
  ✓ @angular/core/testing
  ✓ @angular/common/http/testing
  ✓ jasmine-core
  ✓ karma
  ✓ karma-coverage
  ✗ @ngrx/store/testing     ← not installed (needed for NgRx tests)
  ✗ jasmine-marbles          ← not installed (needed for effect tests)

Coverage thresholds:
  Current  : none configured
  Suggested: statements 90%, branches 85%, functions 90%, lines 90%

Source files     : 35
Existing specs   : 12  (34% of source files)
Missing specs    : 23  (66% of source files)
```

### Missing dependency warning:

If NgRx files are detected but `@ngrx/store/testing` or `jasmine-marbles` are
missing, output:

```
⚠️  MISSING TEST DEPENDENCIES DETECTED
──────────────────────────────────────────────────────────────
NgRx source files found but test packages not installed.
Run before proceeding:

  npm install --save-dev @ngrx/store/testing @ngrx/effects/testing jasmine-marbles
  # or for Jest:
  npm install --save-dev @ngrx/store/testing @ngrx/effects/testing rxjs

Continue anyway? The generated specs will include the correct imports,
but tests will fail to compile until packages are installed.
```

---

## STEP 2 — FULL DISCOVERY

Recursively scan the source directory. Classify every Angular source file.
Apply exclusion rules (modules, barrels, environments, main.ts).

### Discovery output:

```
╔══════════════════════════════════════════════════════════════════╗
║                     DISCOVERY RESULTS                           ║
╚══════════════════════════════════════════════════════════════════╝

Scanned: src/app/ (35 source files, excluding modules/barrels/envs)

┌─────────────────────────────────────────────────────────────────┐
│ Type              │ Total │ Has Spec │ Missing Spec │ Action     │
├───────────────────┼───────┼──────────┼──────────────┼────────────┤
│ Component         │ 12    │ 5        │ 7            │ GENERATE   │
│ Service (HTTP)    │ 6     │ 3        │ 3            │ GENERATE   │
│ Service (pure)    │ 2     │ 2        │ 0            │ COVERAGE   │
│ Pipe              │ 3     │ 3        │ 0            │ COVERAGE   │
│ Directive         │ 2     │ 1        │ 1            │ GENERATE   │
│ Guard             │ 3     │ 0        │ 3            │ GENERATE   │
│ Resolver          │ 1     │ 0        │ 1            │ GENERATE   │
│ Interceptor       │ 2     │ 1        │ 1            │ GENERATE   │
│ NgRx Reducer      │ 2     │ 2        │ 0            │ COVERAGE   │
│ NgRx Effect       │ 2     │ 0        │ 2            │ GENERATE   │
│ NgRx Selector     │ 2     │ 2        │ 0            │ COVERAGE   │
│ NgRx Action       │ 1     │ N/A      │ N/A          │ SKIP*      │
├───────────────────┼───────┼──────────┼──────────────┼────────────┤
│ TOTAL             │ 38    │ 19 (50%) │ 19 (50%)     │            │
└───────────────────┴───────┴──────────┴──────────────┴────────────┘
* Action files are typically tested via reducer/effect tests

Spec files to GENERATE  : 19 new specs
Spec files to COVER     : 10 gap analyses
Static issues to FIX    : scanning...
```

---

## STEP 3 — PIPELINE PLAN

Present the full 3-phase plan before executing anything:

```
╔══════════════════════════════════════════════════════════════════╗
║                    PIPELINE EXECUTION PLAN                      ║
╚══════════════════════════════════════════════════════════════════╝

PHASE 1 — GENERATE (19 new spec files)
  Processing order (dependencies first):
  [NgRx]        users.reducer.spec.ts, users.effects.spec.ts,
                products.reducer.spec.ts, products.effects.spec.ts
  [Services]    auth.service.spec.ts, user.service.spec.ts,
                product.service.spec.ts, logger.service.spec.ts
  [Guards]      auth.guard.spec.ts, role.guard.spec.ts, admin.guard.spec.ts
  [Resolver]    user.resolver.spec.ts
  [Interceptor] auth.interceptor.spec.ts
  [Directives]  highlight.directive.spec.ts
  [Components]  user-list.spec.ts, user-form.spec.ts, product-list.spec.ts,
                dashboard.spec.ts, header.spec.ts (7 total)
  Estimated tests: ~155

PHASE 2 — COVERAGE (10 existing spec files)
  Will fill gaps in: user.service.spec.ts, product.service.spec.ts,
                     users.selectors.spec.ts, users.reducer.spec.ts,
                     and 6 more
  Estimated new it() blocks: ~45

PHASE 3 — FIX (static analysis of all 29 spec files after phases 1–2)
  Will diagnose and fix:
  • NullInjectorError patterns
  • Missing HttpClientTestingModule
  • fakeAsync/tick issues
  • detectChanges() timing

Total estimated tests after pipeline: ~200
Coverage target: 90%

──────────────────────────────────────────────────────────────────
Ready to run full pipeline?

  A) Run all 3 phases (recommended)
  B) Run Phase 1 only (generate missing specs)
  C) Run Phase 2 only (fill coverage gaps in existing specs)
  D) Run Phase 3 only (fix static issues in existing specs)
  E) Run Phases 1 + 2 (generate + cover, skip fix)
  F) Dry run (show plans, write nothing)
  G) Filter by module — tell me which module/folder to start with

Reply with A–G or a specific instruction.
──────────────────────────────────────────────────────────────────
```

---

## PHASE 1 — GENERATE MISSING SPECS

Execute the `/angular-test:generate` workflow for each file without a spec.

### Processing order (always respect dependencies):
1. **NgRx state** (reducer, selectors, effects) — no template dependencies
2. **Services** — depend only on HttpClient and other services
3. **Guards / Resolvers / Interceptors** — depend on services
4. **Directives / Pipes** — minimal dependencies
5. **Components** — depend on all of the above

### Output per file:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PHASE 1 — [1/19] users.reducer.spec.ts
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Type   : NgRx Reducer
File   : src/app/store/users/users.reducer.ts
Spec   : src/app/store/users/users.reducer.spec.ts

[full spec file in fenced code block]

✓ Done — 6 tests generated
```

### Phase 1 completion:

```
PHASE 1 COMPLETE
══════════════════════════════════════════════════════════════
Generated : 19 spec files
Total tests: 155
Duration  : (synchronous generation)

Proceeding to Phase 2...
```

---

## PHASE 2 — FILL COVERAGE GAPS

Execute the `/angular-test:coverage` workflow for all existing spec files
(including the ones just generated in Phase 1).

### Output per file:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PHASE 2 — [1/10] user.service.spec.ts (existing)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Gaps found : 6
[gap table]
[new it() blocks]

✓ Done — 6 new it() blocks generated
```

### Phase 2 completion:

```
PHASE 2 COMPLETE
══════════════════════════════════════════════════════════════
Files analysed : 10
New it() blocks: 45
Files at 100%  : 7
Files below 90%: 3 (edge cases — see notes)

Proceeding to Phase 3...
```

---

## PHASE 3 — STATIC ISSUE DETECTION AND FIXES

Re-scan all spec files (existing + newly generated) for detectable failure patterns.

### Scan and fix order:
1. `NullInjectorError` — missing providers
2. Missing `HttpClientTestingModule`
3. Missing `httpMock.verify()` in afterEach
4. `fixture.detectChanges()` inside `beforeEach`
5. Missing `tick()` after HTTP flush in fakeAsync
6. Missing `discardPeriodicTasks()` in fakeAsync
7. `ActivatedRoute` not mocked
8. `Router` not mocked

### Output:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PHASE 3 — Static Analysis (29 spec files)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Scanning...

Issues found:
  NullInjectorError (3 files)
  Missing HttpClientTestingModule (2 files)
  detectChanges() in beforeEach (4 files)

[per-file targeted fix output]
```

---

## FINAL PROJECT REPORT

After all three phases, output the unified report:

```
╔══════════════════════════════════════════════════════════════════╗
║              ANGULAR TEST SUITE — PIPELINE COMPLETE             ║
╚══════════════════════════════════════════════════════════════════╝

Project         : my-angular-app
Angular version : 17.3.0
Coverage target : 90%

┌─────────────────────────────────────────────────────────────────┐
│ Phase              │ Result                                     │
├────────────────────┼────────────────────────────────────────────┤
│ Phase 1 — Generate │ 19 spec files created, 155 tests           │
│ Phase 2 — Coverage │ 45 new it() blocks across 10 files         │
│ Phase 3 — Fix      │ 9 static issues fixed across 7 files       │
└────────────────────┴────────────────────────────────────────────┘

SPEC COVERAGE SUMMARY
─────────────────────────────────────────────────────────────────
Before pipeline : 12 spec files (34%)
After pipeline  : 29 spec files (100% of testable source files)

ESTIMATED CODE COVERAGE
─────────────────────────────────────────────────────────────────
Before : ~28% (estimated — only 12 specs existed)
After  : ~93% (estimated — 200 tests across 29 specs)
Target : 90% ✓

BREAKDOWN BY MODULE
┌─────────────────────────────┬────────┬────────┬─────────────┐
│ Module                      │ Specs  │ Tests  │ Est. Cover  │
├─────────────────────────────┼────────┼────────┼─────────────┤
│ core/services/              │ 6      │ 52     │ ~96%        │
│ core/guards/                │ 3      │ 15     │ ~95%        │
│ core/interceptors/          │ 2      │ 10     │ ~92%        │
│ features/users/             │ 9      │ 74     │ ~94%        │
│ features/products/          │ 5      │ 42     │ ~91%        │
│ shared/                     │ 5      │ 27     │ ~98%        │
├─────────────────────────────┼────────┼────────┼─────────────┤
│ TOTAL                       │ 29     │ 200    │ ~93%        │
└─────────────────────────────┴────────┴────────┴─────────────┘

NOTE: Estimates are based on static analysis. Run ng test --code-coverage
for the actual coverage numbers.

══════════════════════════════════════════════════════════════════
NEXT STEPS
══════════════════════════════════════════════════════════════════

1. Install any missing packages (if flagged above):
     npm install --save-dev @ngrx/store/testing jasmine-marbles

2. Verify TypeScript compilation:
     npx tsc --noEmit --project tsconfig.spec.json

3. Run the full test suite:
     ng test --no-watch --code-coverage
   Or Jest:
     npx jest --coverage

4. Check the coverage report:
     open coverage/index.html

5. Fix any remaining runtime failures:
     /angular-test:fix --from-log=test-output.log

6. Add coverage thresholds to your config (paste below):

──────────────────────────────────────────────────────────────────
karma.conf.js — add inside coverageReporter:
──────────────────────────────────────────────────────────────────
check: {
  global: { statements: 90, branches: 85, functions: 90, lines: 90 },
  each:   { statements: 80, branches: 75, functions: 80, lines: 80 }
}

──────────────────────────────────────────────────────────────────
jest.config.js — add at root level:
──────────────────────────────────────────────────────────────────
coverageThreshold: {
  global: { statements: 90, branches: 85, functions: 90, lines: 90 }
}

──────────────────────────────────────────────────────────────────
angular.json — add under "test" > "options":
──────────────────────────────────────────────────────────────────
"codeCoverage": true,
"codeCoverageExclude": [
  "src/environments/**",
  "src/main.ts",
  "src/**/*.module.ts",
  "src/**/index.ts"
]
══════════════════════════════════════════════════════════════════
```

---

## DRY-RUN MODE

When `--dry-run` is active, output only the discovery and pipeline plan —
write zero files and generate zero code blocks. The dry-run report is
identical to Steps 1–3 above, ending with:

```
DRY RUN COMPLETE — No files written.
To execute the pipeline: /angular-test:suite
To run specific phases:
  Generate only : /angular-test:generate --all
  Coverage only : /angular-test:coverage --all
  Fix only      : /angular-test:fix --all
```

---

## CONSTRAINTS

- **Always** show the full pipeline plan and ask for confirmation before executing
- **Always** process in dependency order: NgRx → Services → Guards → Directives → Components
- **Never** overwrite existing specs unless `--overwrite` is explicitly passed
- **Never** generate specs for `*.module.ts`, `index.ts`, `environment*.ts`, `main.ts`
- In dry-run mode: no code generation, no file writes
- At each phase boundary: print a completion summary before starting the next phase
- If the user says "stop" or "cancel" mid-pipeline: halt immediately and show what was completed
- For projects with 50+ source files: recommend running by module (`--only=<module-path>`)
  to keep output manageable
