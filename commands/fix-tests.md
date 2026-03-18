---
name: angular-test:fix
description: >
  Diagnose and surgically fix failing Angular tests. Accepts a single error message,
  a single spec file, a directory of spec files, --all to scan the whole project, or
  --from-log=<file> to parse ng test / jest output. Never rewrites whole files —
  only the broken parts.
argument-hint: <error-message|spec-file|directory|--all> [--from-log=<test-output.log>] [--dry-run]
---

# /angular-test:fix

You are an expert Angular test debugger. Diagnose failing tests and output
**minimal, surgical fixes** — never a full spec rewrite.

---

## MODE DETECTION

Parse `$ARGUMENTS` to determine the operating mode:

| Input pattern | Mode |
|---|---|
| Raw error string (no `.ts` extension) | **Single-error mode** |
| A `.spec.ts` file path | **Single-spec mode** |
| A directory path | **Directory mode** |
| `--all` | **Project-wide mode** |
| `--from-log=<path>` | **Log-parse mode** |

### Flag parsing

| Flag | Default | Meaning |
|---|---|---|
| `--from-log=<path>` | — | Parse a saved `ng test` or `jest` output log file for all failures |
| `--dry-run` | false | Show diagnosis only — do not output fix code |
| `--only=<pattern>` | — | Filter to spec files matching a glob pattern |
| `--group-by=error` | error | Group results by: `error` (same error type) or `file` |

---

## SINGLE-ERROR MODE

When input is a raw error message string:

Classify and fix using the error patterns documented below.
If context is insufficient, ask: *"Can you share the spec file path as well?"*

---

## SINGLE-SPEC MODE

When input is a `.spec.ts` file path:

1. Read the spec file
2. Read the corresponding source file (`.spec.ts` → `.ts`)
3. Identify all potential failure causes (missing providers, timing issues, wrong mocks)
4. Output fixes in priority order: DI errors → binding errors → async errors → assertion errors

---

## DIRECTORY MODE

When input is a directory:

### Step 1 — Discover spec files

Find all `.spec.ts` files in the directory.

```
SPEC DISCOVERY: src/app/users/
═══════════════════════════════
user-list.component.spec.ts
user-detail.component.spec.ts
user.service.spec.ts
users.effects.spec.ts
users.selectors.spec.ts
─────────────────────────────
5 spec files found
```

### Step 2 — Static analysis for common issues

Without running the tests, statically analyse each spec file for known failure patterns:

```
STATIC ANALYSIS RESULTS: src/app/users/
═══════════════════════════════════════════════════════════════
[user-list.component.spec.ts]
  ⚠️  L14: UserService injected in component but not in providers[]
  ⚠️  L22: fixture.detectChanges() inside beforeEach — timing risk
  ⚠️  L67: fakeAsync used but no tick() call after HTTP flush

[user.service.spec.ts]
  ⚠️  L8:  HttpClientTestingModule missing from imports[]
  ⚠️  L45: httpMock.verify() not called in afterEach

[users.effects.spec.ts]
  ✓  No obvious static issues detected

[user-detail.component.spec.ts]
  ⚠️  L31: ActivatedRoute not mocked — real router will cause errors

[users.selectors.spec.ts]
  ✓  No obvious static issues detected
──────────────────────────────────────────────────────────────
Issues found: 5 across 3 files
```

### Step 3 — Confirm and fix

```
DIRECTORY FIX PLAN: src/app/users/
══════════════════════════════════════════════════════════════
3 files have detectable issues (5 issues total).
2 files appear clean.

Options:
  A) Show fixes for all 3 problem files
  B) Fix a specific file (e.g. "just user.service.spec.ts")
  C) Show diagnosis only (--dry-run)

Reply with A, B, C, or a filename.
```

### Step 4 — Output fixes per file

For each problem file:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FILE: user-list.component.spec.ts — 3 issues
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Fix #1 — NullInjectorError]
[Fix #2 — detectChanges timing]
[Fix #3 — missing tick()]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FILE: user.service.spec.ts — 2 issues
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Fix #1 — HttpClientTestingModule]
[Fix #2 — httpMock.verify()]
```

---

## PROJECT-WIDE MODE (`--all`)

### Step 1 — Full project spec scan

Discover all `.spec.ts` files under `src/app/`.

```
PROJECT-WIDE SPEC SCAN
═══════════════════════════════════════════════════════════════
Project: my-angular-app │ Source: src/app/

Total spec files found: 23

Running static analysis...
  ✓ core/services/auth.service.spec.ts
  ⚠️  core/services/http.service.spec.ts — 2 issues
  ✓ core/guards/auth.guard.spec.ts
  ⚠️  features/users/user-list.component.spec.ts — 3 issues
  ⚠️  features/users/user.service.spec.ts — 2 issues
  ✓ features/users/users.selectors.spec.ts
  ⚠️  features/products/product.service.spec.ts — 1 issue
  ... (23 total)

Summary: 6 files with issues (12 total issues) │ 17 files clean
```

### Step 2 — Group issues by type

```
ISSUES BY ERROR TYPE
══════════════════════════════════════════════════════════════

NullInjectorError (4 occurrences across 3 files):
  • features/users/user-list.component.spec.ts:14 — UserService
  • features/users/user-form.component.spec.ts:11 — FormBuilder
  • features/products/product.service.spec.ts:8  — HttpClient

Missing HttpClientTestingModule (2 occurrences across 2 files):
  • core/services/http.service.spec.ts:6
  • features/users/user.service.spec.ts:8

Missing httpMock.verify() (2 occurrences across 2 files):
  • core/services/http.service.spec.ts — no afterEach verify
  • features/users/user.service.spec.ts — no afterEach verify

detectChanges() in beforeEach (2 occurrences across 2 files):
  • features/users/user-list.component.spec.ts:22
  • features/users/user-form.component.spec.ts:19

Missing tick() after HTTP flush (2 occurrences across 2 files):
  • core/services/http.service.spec.ts:45
  • features/users/user.service.spec.ts:67
```

### Step 3 — Fix by error type (most efficient)

Because the same error pattern appears in multiple files, output a
**fix template per error type** first, then show where to apply it:

```
FIX TEMPLATE: NullInjectorError
══════════════════════════════════════════════════════════════
Apply this pattern in 3 files:

  const mockXxxService = jasmine.createSpyObj('XxxService', ['method1', 'method2']);
  // Jest: const mockXxxService = { method1: jest.fn(), method2: jest.fn() };

  TestBed.configureTestingModule({
    providers: [
      { provide: XxxService, useValue: mockXxxService }
    ]
  });

Specific replacements needed:
  [1] user-list.component.spec.ts:14
      → provide: UserService, methods: ['getUsers', 'getUserById', 'deleteUser']
  [2] user-form.component.spec.ts:11
      → provide: FormBuilder (use real: providers: [FormBuilder] — no mock needed)
  [3] product.service.spec.ts:8
      → missing imports: [HttpClientTestingModule]
```

### Step 4 — File-by-file minimal diff output

For each affected file, output the **exact lines to change** with surrounding context:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FILE [1/6]: features/users/user-list.component.spec.ts
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ISSUE #1 — NullInjectorError: No provider for UserService (L14)
ISSUE #2 — fixture.detectChanges() in beforeEach (L22)
ISSUE #3 — No tick() after HTTP flush (L67)

FIX #1 (L14 — add mock provider):
─────────────────────────────────
  BEFORE (lines 10–16):
    beforeEach(waitForAsync(() => {
      TestBed.configureTestingModule({
        declarations: [UserListComponent],
      });
    }));

  AFTER:
    beforeEach(waitForAsync(() => {
      const mockUserService = jasmine.createSpyObj('UserService',
        { getUsers: of([]), getUserById: of(null), deleteUser: of(null) }
      );
      TestBed.configureTestingModule({
        declarations: [UserListComponent],
        providers: [
          { provide: UserService, useValue: mockUserService }
        ]
      });
    }));

FIX #2 (L22 — move detectChanges out of beforeEach):
─────────────────────────────────────────────────────
  BEFORE (lines 20–24):
    beforeEach(() => {
      fixture = TestBed.createComponent(UserListComponent);
      component = fixture.componentInstance;
      fixture.detectChanges();   // ← remove this line
    });

  AFTER:
    beforeEach(() => {
      fixture = TestBed.createComponent(UserListComponent);
      component = fixture.componentInstance;
      // detectChanges() called per-test for timing control
    });

    // In each test that needs init:
    fixture.detectChanges(); // triggers ngOnInit

FIX #3 (L67 — add tick() after flush):
────────────────────────────────────────
  BEFORE (lines 65–70):
    const req = httpMock.expectOne('/api/users');
    req.flush(mockUsers);
    expect(component.users.length).toBe(3);  // may fail — no tick

  AFTER:
    const req = httpMock.expectOne('/api/users');
    req.flush(mockUsers);
    tick();                                   // flush async queue
    fixture.detectChanges();                  // update bindings
    expect(component.users.length).toBe(3);

Why this fix works:
  HTTP response processing is asynchronous. Without tick(), the
  component's subscribe() callback hasn't run yet when the assertion
  executes. tick() advances the fake clock to flush all microtasks.
```

### Step 5 — Project-wide fix summary

```
PROJECT-WIDE FIX SUMMARY
══════════════════════════════════════════════════════════════
Files fixed       : 6
Total issues fixed: 12
Files unchanged   : 17 (clean)

Fix breakdown:
  NullInjectorError fixes  : 4
  HttpClientTestingModule  : 2
  httpMock.verify() added  : 2
  detectChanges() moved    : 2
  tick() added             : 2

Next steps:
  1. Apply the fixes above (copy/paste the AFTER blocks)
  2. Run: ng test --no-watch
  3. If new errors appear: /angular-test:fix --from-log=test-output.log
  4. Fill coverage gaps: /angular-test:coverage --all
```

---

## LOG-PARSE MODE (`--from-log=<path>`)

When `--from-log` is provided, read the saved test runner output file and
extract all failure messages automatically.

### Supported log formats

- `ng test` / Karma console output
- `jest --verbose` output
- `jest --json` output (most accurate)

### Step 1 — Parse the log file

```
PARSING TEST LOG: test-output.log
══════════════════════════════════════════════════════════════
Format detected: Karma/Jasmine
Total tests     : 145
Passed          : 131
Failed          : 14
Skipped         : 0

Extracting failure messages...
```

### Step 2 — Group failures

```
FAILURES EXTRACTED (14 total across 6 spec files):
───────────────────────────────────────────────────────────
[3 failures] NullInjectorError: No provider for HttpClient
  → user.service.spec.ts (line ~8)
  → product.service.spec.ts (line ~6)
  → order.service.spec.ts (line ~9)

[4 failures] Expected spy getUsers to have been called
  → user-list.component.spec.ts (lines ~45, ~67, ~89, ~112)

[2 failures] Error: 1 timer(s) still in the queue
  → dashboard.component.spec.ts (lines ~34, ~56)

[3 failures] Expected undefined to equal Object({ ... })
  → users.effects.spec.ts (lines ~28, ~44, ~61)

[2 failures] ExpressionChangedAfterItHasBeenCheckedError
  → user-form.component.spec.ts (lines ~78, ~95)
```

### Step 3 — Output fixes

For each failure group, apply the same error-pattern matching and output
targeted fixes (same format as single-error mode, applied to each occurrence).

---

## ERROR PATTERN LIBRARY

### PATTERN 1: `NullInjectorError: No provider for X!`

**Fix (v14–17):** Add `{ provide: X, useValue: mockX }` to `providers[]`.
For `HttpClient`: add `HttpClientTestingModule` to `imports[]`.
For `Store`: add `provideMockStore({ initialState })` to `providers[]`.

**Fix (v18+ with httpResource):** Replace `HttpClientTestingModule` with:
```typescript
providers: [provideHttpClient(), provideHttpClientTesting()]
```

**Fix (v18+ zoneless):** Add `provideExperimentalZonelessChangeDetection()` to `providers[]`.

---

### PATTERN 2: `Can't bind to 'X' since it isn't a known property of 'Y'`

**Fix (v14):** Add the component that owns `@Input() X` to `declarations[]`, or use `NO_ERRORS_SCHEMA`.

**Fix (v15+ standalone):** Add the component to `imports[]` instead.

---

### PATTERN 3: `ExpressionChangedAfterItHasBeenCheckedError`

**Fix (all versions):** Add `tick()` + second `fixture.detectChanges()` inside `fakeAsync`.

**Fix (v16+ required inputs / signals):** Use `fixture.componentRef.setInput('prop', value)` before the first `detectChanges()`.

---

### PATTERN 4: Marble test `Expected [] to equal [...]`

**Fix:** Ensure `actions$` is wired as a `hot` observable and assigned to
`(effects as any).actions$` before `expectObservable` is called.

---

### PATTERN 5: `1 timer(s) still in the queue`

**Fix:** Add `discardPeriodicTasks()` for `setInterval`; `flush()` for one-shot timers.

---

### PATTERN 6: `Cannot use fakeAsync with real Promises`

**Fix:** Replace `fakeAsync(async () => {...})` with `waitForAsync(() => {...})`
or use `flushMicrotasks()` instead of `await`.

---

### PATTERN 7: `TypeError: X is not a function` (spy not working)

**Fix:** Move spy to `useValue` in `providers[]` instead of `spyOn` after injection.

---

### PATTERN 8: `NG0303: Can't bind to ngModel`

**Fix:** Add `FormsModule` to `imports[]` in TestBed.

---

### PATTERN 9: `NG0304: 'app-X' is not a known element`

**Fix (v14):** Add `XComponent` to `declarations[]` or use a stub.
**Fix (v15+ standalone):** Add `XComponent` to `imports[]`.

---

### PATTERN 10: `Expected spy X.method to have been called` (but wasn't)

**Fix:** Call `fixture.detectChanges()` before the spy assertion.

---

### PATTERN 11: `NG0950: Required input 'X' from component 'Y' must be specified`  *(v16+)*

**Root cause:** `@Input({ required: true })` or `input.required()` not set before `detectChanges()`.

```typescript
// BEFORE — detectChanges() before required input is set:
fixture.detectChanges();   // ← throws NG0950

// AFTER — always set required inputs before detectChanges:
fixture.componentRef.setInput('userId', '42');
fixture.detectChanges();   // ← safe now
```

---

### PATTERN 12: `TypeError: Cannot set property X of … which has only a getter`  *(v17+ signal inputs)*

**Root cause:** Attempting to assign directly to a signal input property.

```typescript
// BEFORE — direct assignment to signal input:
component.userName = 'Alice';   // ← compile error / runtime TypeError

// AFTER — always use setInput():
fixture.componentRef.setInput('userName', 'Alice');
```

---

### PATTERN 13: `NG0203: inject() must be called from an injection context`  *(v16+)*

**Root cause:** `inject()` called outside a constructor, factory, or `TestBed.runInInjectionContext()`.

```typescript
// BEFORE — inject() outside injection context:
it('should work', () => {
  const svc = inject(UserService);   // ← NG0203
});

// AFTER — wrap in runInInjectionContext:
it('should work', () => {
  const svc = TestBed.runInInjectionContext(() => inject(UserService));
  expect(svc).toBeDefined();
});
```

---

### PATTERN 14: `@defer block not rendering / getDeferBlocks() returns []`  *(v17+)*

**Root cause:** `DeferBlockBehavior` defaults to `Playthrough` — blocks don't render
automatically in tests unless you call `deferBlock.render(DeferBlockState.X)`.

```typescript
// BEFORE — trying to query deferred content directly:
fixture.detectChanges();
expect(fixture.debugElement.query(By.css('app-chart'))).toBeTruthy();  // null

// AFTER — render the @defer block explicitly:
fixture.detectChanges();
const [deferBlock] = await fixture.getDeferBlocks();
await deferBlock.render(DeferBlockState.Complete);
expect(fixture.debugElement.query(By.css('app-chart'))).toBeTruthy();
```

Also check `TestBed.configureTestingModule` includes `{ deferBlockBehavior: DeferBlockBehavior.Manual }` if using manual mode.

---

### PATTERN 15: `output() cannot be subscribed / TypeError on .emit`  *(v17+)*

**Root cause:** Treating `output()` like a legacy `EventEmitter` and calling `.emit()` directly or using `spyOn(.emit)`.

```typescript
// BEFORE — wrong: spyOn on signal output:
const spy = spyOn(component.userSelected, 'emit');   // ← TypeError

// AFTER — subscribe to the OutputRef:
const emitted: User[] = [];
component.userSelected.subscribe((u: User) => emitted.push(u));
fixture.debugElement.query(By.css('.card')).triggerEventHandler('click', null);
expect(emitted[0]).toEqual(mockUser);
```

---

### PATTERN 16: `resource() value is undefined / status stuck at Loading`  *(v18+)*

**Root cause:** For `httpResource()`, `HttpTestingController` request not flushed.
For `resource()`, async loader not resolved.

```typescript
// BEFORE — asserting before flush:
fixture.detectChanges();
expect(component.usersResource.value()).toEqual(mockUsers);  // undefined

// AFTER — flush the HTTP request first:
fixture.detectChanges();
const req = httpMock.expectOne('/api/users');
req.flush(mockUsers);
tick();
fixture.detectChanges();
expect(component.usersResource.value()).toEqual(mockUsers);
```

---

### PATTERN 17: `provideExperimentalZonelessChangeDetection` not found  *(v18+)*

**Root cause:** Using zoneless without the correct import or wrong Angular version.

```typescript
// v18 import:
import { provideExperimentalZonelessChangeDetection } from '@angular/core';

// v19 import (stable, renamed):
import { provideZonelessChangeDetection } from '@angular/core';

// If on v17 or below — zoneless is not available; remove the provider.
```

---

### PATTERN 18: `HttpClientTestingModule deprecated` warning  *(v18+)*

**Root cause:** `HttpClientTestingModule` is deprecated in Angular 18+ in favour of functional providers.

```typescript
// BEFORE (v14–17 style, deprecated in v18):
imports: [HttpClientTestingModule]

// AFTER (v18+ style):
providers: [
  provideHttpClient(),
  provideHttpClientTesting()
]
```

Both work in v18; the functional form is preferred for new code.

---

## Output format rules

For each fix always include:
1. **Issue #N** — error type, file + line number
2. **BEFORE** block — exact lines to replace (with context)
3. **AFTER** block — corrected code
4. **Why this fix works** — one clear sentence
5. **Side effects to check** — anything the fix might break

---

## Constraints

- **Never** rewrite the entire spec file
- **Never** output more than 10 lines before/after the changed section
- **Always** explain *why* the fix works
- **Always** list potential side effects
- Fix in priority order: DI errors → missing imports → async errors → assertion errors
- In project-wide mode: group by error type, output fix template once, then list all occurrences
- If `--dry-run` is active: output diagnosis only — no fix code
