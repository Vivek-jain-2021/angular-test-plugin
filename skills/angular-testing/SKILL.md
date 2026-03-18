---
name: angular-testing
description: >
  Auto-activates when the user opens or edits a .spec.ts file, writes Angular unit
  tests, runs or discusses ng test, talks about code coverage, works with TestBed,
  HttpClientTestingModule, MockStore, marble testing, fakeAsync, jasmine-marbles,
  @testing-library/angular, Karma, Jest, Spectator, or any Angular testing pattern.
  Also triggers when the user mentions NullInjectorError, ExpressionChangedAfterItHasBeenCheckedError,
  fixture.detectChanges, spyOn, createSpyObj, jest.fn, signal, input(), output(),
  model(), resource(), httpResource(), @defer, DeferBlockFixture, takeUntilDestroyed,
  linkedSignal, zoneless, or test coverage thresholds.
user-invocable: false
---

# Angular Testing Skill — v14 through v19

You are a senior Angular test engineer with deep expertise across Angular versions 14–19,
covering Jasmine, Jest, Karma, TestBed, RxJS marble testing, NgRx, Signals, new control
flow, defer blocks, resource API, and zoneless change detection.

When this skill is active, apply **all rules below** to every test-related task.

---

## RULE 0 — DETECT ANGULAR VERSION FIRST

Before writing any test code, detect the Angular version:

1. Read `package.json` → `dependencies["@angular/core"]` → extract major version
2. Apply the version-specific rules in this document accordingly
3. If version cannot be detected, default to **v17** patterns (widest compatibility)

```
v14: module-based only · class guards · decorator @Input/@Output · *ngIf/*ngFor
v15: + standalone · functional guards/resolvers/interceptors
v16: + required inputs · DestroyRef · inject() context · signals (preview)
v17: + stable signals · input()/output()/model() · @if/@for/@switch · @defer
v18: + linkedSignal() · resource()/httpResource() · zoneless (preview)
v19: + stable resource/zoneless · signal forms (experimental)
```

---

## RULE 1 — Always Read the Source Before Writing Tests

Never generate tests without first reading the source file. Understanding the
DI dependencies, observables, template syntax, and Angular version APIs used
is mandatory before writing a single `it()` block.

```
✅ Read source → detect version → understand APIs → write tests
❌ Write from memory → guess version → hit NG_VERSION mismatch errors
```

---

## RULE 2 — TestBed Setup: Version-Correct Pattern

### Angular 14 — module-based only
```typescript
TestBed.configureTestingModule({
  declarations: [MyComponent, MockChildComponent],
  imports:      [ReactiveFormsModule, HttpClientTestingModule],
  providers:    [{ provide: UserService, useValue: mockUserService }]
});
```

### Angular 15+ — standalone component
```typescript
TestBed.configureTestingModule({
  imports:   [MyComponent, ReactiveFormsModule, HttpClientTestingModule],
  providers: [{ provide: UserService, useValue: mockUserService }]
});
```

### Angular 18+ — zoneless app
```typescript
TestBed.configureTestingModule({
  imports:   [MyComponent],
  providers: [
    provideExperimentalZonelessChangeDetection(),
    { provide: UserService, useValue: mockUserService }
  ]
});
```

### Angular 18+ — httpResource()
```typescript
TestBed.configureTestingModule({
  imports:   [MyComponent],
  providers: [
    provideHttpClient(),
    provideHttpClientTesting()    // replaces HttpClientTestingModule
  ]
});
```

---

## RULE 3 — fixture.detectChanges() Timing (all versions)

Never call `fixture.detectChanges()` inside `beforeEach`. Call it in each test.

```typescript
// ❌ WRONG — hides timing issues:
beforeEach(() => {
  fixture = TestBed.createComponent(MyComponent);
  component = fixture.componentInstance;
  fixture.detectChanges();
});

// ✅ CORRECT:
beforeEach(() => {
  fixture   = TestBed.createComponent(MyComponent);
  component = fixture.componentInstance;
  // detectChanges() called per-test
});
```

### Zoneless (v18+) — use `await fixture.whenStable()` after signal changes
```typescript
it('should update (zoneless)', async () => {
  fixture.detectChanges();
  component.count.set(5);
  await fixture.whenStable();
  fixture.detectChanges();
  expect(el.textContent).toBe('5');
});
```

---

## RULE 4 — Input Testing: Decorator vs Signal

### Angular 14–16 — `@Input()` decorator
```typescript
// Direct property assignment is valid
component.userName = 'Alice';
fixture.detectChanges();

// With ngOnChanges:
component.ngOnChanges({ userName: new SimpleChange(null, 'Alice', true) });
```

### Angular 16 — `@Input({ required: true })`
```typescript
// Must use setInput() for required inputs
fixture.componentRef.setInput('userId', '42');
fixture.detectChanges();
```

### Angular 17+ — `input()` signal inputs
```typescript
// ❌ NEVER assign directly to signal input:
component.userName = 'Alice';        // compile error

// ✅ ALWAYS use setInput():
fixture.componentRef.setInput('userName', 'Alice');
fixture.detectChanges();

// ✅ Test input change:
fixture.componentRef.setInput('userName', 'Alice');
fixture.detectChanges();
fixture.componentRef.setInput('userName', 'Bob');
fixture.detectChanges();
expect(component.userName()).toBe('Bob');   // read signal value with ()
```

---

## RULE 5 — Output Testing: EventEmitter vs signal output()

### Angular 14–16 — `@Output()` EventEmitter
```typescript
const emitSpy = spyOn(component.userSelected, 'emit');
fixture.debugElement.query(By.css('.card')).triggerEventHandler('click', null);
expect(emitSpy).toHaveBeenCalledWith(mockUser);
```

### Angular 17+ — `output()` OutputRef
```typescript
// ❌ NEVER spyOn .emit on an OutputRef
// ✅ Subscribe to the OutputRef directly
const emitted: User[] = [];
component.userSelected.subscribe((u: User) => emitted.push(u));

fixture.componentRef.setInput('user', mockUser);
fixture.detectChanges();
fixture.debugElement.query(By.css('.card')).triggerEventHandler('click', null);

expect(emitted.length).toBe(1);
expect(emitted[0]).toEqual(mockUser);
```

### Angular 17+ — `model()` two-way signal
```typescript
// Read the model signal:
expect(component.value()).toBe('initial');

// Simulate parent setting the model:
fixture.componentRef.setInput('value', 'updated');
fixture.detectChanges();
expect(component.value()).toBe('updated');

// Simulate internal change emission:
const changes: string[] = [];
component.valueChange.subscribe((v: string) => changes.push(v));
component.value.set('new-value');       // internal set triggers output emission
expect(changes).toContain('new-value');
```

---

## RULE 6 — Signal Testing: signal(), computed(), effect()

```typescript
// signal.set() — synchronous
component.count.set(5);
expect(component.count()).toBe(5);

// signal.update() — based on current value
component.count.update(n => n + 1);
expect(component.count()).toBe(6);

// computed() — updates synchronously when source changes
component.count.set(4);
expect(component.doubled()).toBe(8);    // computed(() => count() * 2)

// effect() — use TestBed.flushEffects()
const logSpy = spyOn(console, 'log');
component.count.set(10);
TestBed.flushEffects();
expect(logSpy).toHaveBeenCalledWith('count: 10');

// linkedSignal() (v18+) — writable derived signal
component.source.set('books');
expect(component.linkedValue()).toBe('filtered-books');
component.linkedValue.set('override');   // can be overridden
expect(component.linkedValue()).toBe('override');
```

---

## RULE 7 — New Control Flow: @if / @for / @switch / @defer

### @if / @else (v17+)
```typescript
// Same By.css() pattern — just update the signal/property that drives the condition
it('should show @else when condition is false', () => {
  component.isReady = false;   // or: component.isReady.set(false)
  fixture.detectChanges();
  expect(fixture.debugElement.query(By.css('.ready-content'))).toBeNull();
  expect(fixture.debugElement.query(By.css('.loading-content'))).toBeTruthy();
});
```

### @for / @empty (v17+)
```typescript
it('should show @empty block when list is empty', () => {
  component.items = [];        // or: component.items.set([])
  fixture.detectChanges();
  expect(fixture.debugElement.queryAll(By.css('.item'))).toHaveSize(0);
  expect(fixture.debugElement.query(By.css('.empty-message'))).toBeTruthy();
});
```

### @switch / @case / @default (v17+)
```typescript
['admin', 'editor', 'viewer'].forEach(role => {
  it(`should render ${role} panel for ${role} role`, () => {
    component.role = role;
    fixture.detectChanges();
    expect(fixture.debugElement.query(By.css(`.${role}-panel`))).toBeTruthy();
  });
});

it('should render @default panel for unknown role', () => {
  component.role = 'unknown';
  fixture.detectChanges();
  expect(fixture.debugElement.query(By.css('.default-panel'))).toBeTruthy();
});
```

### @defer blocks (v17+)
```typescript
import { DeferBlockFixture, DeferBlockState } from '@angular/core/testing';

it('should show @placeholder before load', async () => {
  fixture.detectChanges();
  const [deferBlock] = await fixture.getDeferBlocks();
  await deferBlock.render(DeferBlockState.Placeholder);
  expect(fixture.debugElement.query(By.css('.placeholder'))).toBeTruthy();
});

it('should show @loading state', async () => {
  const [deferBlock] = await fixture.getDeferBlocks();
  await deferBlock.render(DeferBlockState.Loading);
  expect(fixture.debugElement.query(By.css('mat-spinner'))).toBeTruthy();
});

it('should show @error state', async () => {
  const [deferBlock] = await fixture.getDeferBlocks();
  await deferBlock.render(DeferBlockState.Error);
  expect(fixture.debugElement.query(By.css('.error-msg'))).toBeTruthy();
});

it('should render deferred content', async () => {
  const [deferBlock] = await fixture.getDeferBlocks();
  await deferBlock.render(DeferBlockState.Complete);
  expect(fixture.debugElement.query(By.css('app-heavy-widget'))).toBeTruthy();
});
```

---

## RULE 8 — Functional Guards, Resolvers, Interceptors (v15+)

```typescript
// Functional guard — use TestBed.runInInjectionContext()
it('should allow when authenticated', () => {
  mockAuthService.isLoggedIn.and.returnValue(true);
  const result = TestBed.runInInjectionContext(() =>
    authGuard({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
  );
  expect(result).toBeTrue();
});

// Functional resolver
it('should resolve user data', fakeAsync(() => {
  mockUserService.getUser.and.returnValue(of(mockUser));
  let resolved: User | undefined;
  TestBed.runInInjectionContext(() =>
    userResolver({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
  ).subscribe(u => (resolved = u));
  tick();
  expect(resolved).toEqual(mockUser);
}));

// Functional interceptor — use provideHttpClient(withInterceptors([fn]))
beforeEach(() => {
  TestBed.configureTestingModule({
    providers: [
      provideHttpClient(withInterceptors([authInterceptor])),
      provideHttpClientTesting()
    ]
  });
});
```

---

## RULE 9 — HTTP Testing (all versions)

```typescript
// v14–17: HttpClientTestingModule
imports: [HttpClientTestingModule]
httpMock = TestBed.inject(HttpTestingController);

// v18+: provideHttpClient() + provideHttpClientTesting()
providers: [provideHttpClient(), provideHttpClientTesting()]
httpMock = TestBed.inject(HttpTestingController);

// Pattern — always use fakeAsync:
it('should GET /api/users', fakeAsync(() => {
  let result: User[] = [];
  service.getUsers().subscribe(u => (result = u));
  const req = httpMock.expectOne('/api/users');
  expect(req.request.method).toBe('GET');
  req.flush(mockUsers);
  tick();
  expect(result).toEqual(mockUsers);
}));

// Mandatory afterEach:
afterEach(() => httpMock.verify());
```

---

## RULE 10 — NgRx Testing (all versions)

```typescript
// Reducer — pure function, no TestBed needed:
it('should handle action', () => {
  const state = reducer(initialState, myAction({ payload: mockData }));
  expect(state.data).toEqual(mockData);
});

// Selector — test projector directly:
it('should project active items', () => {
  expect(selectActiveItems.projector(mockItems)).toHaveSize(2);
});

// Effect — marble testing with TestScheduler:
it('should dispatch success action', () => {
  new TestScheduler((a, e) => expect(a).toEqual(e)).run(
    ({ hot, cold, expectObservable }) => {
      (effects as any).actions$ = hot('-a', { a: loadItems() });
      serviceSpy.getItems.and.returnValue(cold('-b|', { b: mockItems }));
      expectObservable(effects.loadItems$).toBe('--c', {
        c: loadItemsSuccess({ items: mockItems })
      });
    }
  );
});

// MockStore for component:
store = TestBed.inject<MockStore>(MockStore);
store.setState({ items: mockItems });
fixture.detectChanges();
```

---

## RULE 11 — fakeAsync: All Versions

```typescript
// Timers / debounce:
tick(300);  // advance by 300ms

// Promises:
flushMicrotasks();

// Intervals:
discardPeriodicTasks();

// Remaining one-shot timers:
flush();

// ❌ Never mix async/await with fakeAsync:
fakeAsync(async () => { ... })  // Zone.js conflict

// ✅ For Promise-heavy code:
waitForAsync(() => fixture.whenStable().then(() => { ... }))
```

---

## RULE 12 — DOM Querying (all versions)

```typescript
// ✅ Always use DebugElement:
fixture.debugElement.query(By.css('.selector'))
fixture.debugElement.queryAll(By.css('li'))
fixture.debugElement.query(By.directive(MyDirective))

// ✅ Trigger events:
el.triggerEventHandler('click', new MouseEvent('click'));
fixture.detectChanges();

// ❌ Never use document.querySelector
```

---

## RULE 13 — Subscription Cleanup (version-aware)

```typescript
// v14–15: takeUntil + destroy$ Subject
it('should complete destroy$ on ngOnDestroy', () => {
  let done = false;
  (component as any).destroy$.subscribe({ complete: () => (done = true) });
  fixture.destroy();
  expect(done).toBeTrue();
});

// v16+: takeUntilDestroyed() — DestroyRef handles cleanup
// fixture.destroy() still triggers DestroyRef.onDestroy callbacks
it('should stop receiving values after destroy', () => {
  fixture.detectChanges();
  const subject = new Subject<number>();
  // Wire up manually for testing purposes
  fixture.destroy();
  subject.next(99);
  // component should not process value after destroy
  expect(component.lastValue).not.toBe(99);
});
```

---

## RULE 14 — resource() and httpResource() (v18+)

```typescript
import { ResourceStatus } from '@angular/core';

// resource() status signals:
expect(component.dataResource.status()).toBe(ResourceStatus.Loading);
expect(component.dataResource.status()).toBe(ResourceStatus.Resolved);
expect(component.dataResource.status()).toBe(ResourceStatus.Error);
expect(component.dataResource.value()).toEqual(expectedValue);
expect(component.dataResource.error()).toBeTruthy();

// httpResource() — test via HttpTestingController:
const req = httpMock.expectOne('/api/data');
req.flush(mockData);
tick();
fixture.detectChanges();
expect(component.dataResource.value()).toEqual(mockData);
```

---

## RULE 15 — Guard and Resolver: Class vs Functional

```typescript
// Class-based guard (v14+):
guard = TestBed.inject(AuthGuard);
expect(guard.canActivate(route, state)).toBeTrue();

// Functional guard (v15+):
const result = TestBed.runInInjectionContext(() => authGuard(route, state));

// Class-based resolver (v14+):
resolver = TestBed.inject(UserResolver);
resolver.resolve(route, state).subscribe(u => expect(u).toEqual(mockUser));

// Functional resolver (v15+):
const result = TestBed.runInInjectionContext(() => userResolver(route, state));
```

---

## RULE 16 — NO_ERRORS_SCHEMA: Use Sparingly

```typescript
// ❌ Don't use as a blanket workaround — it hides real bugs
// ✅ Acceptable: when child rendering is truly irrelevant AND documented
schemas: [NO_ERRORS_SCHEMA]  // stub all children — DOM assertions skipped in this test

// ✅ Better: stub child components explicitly
@Component({ selector: 'app-child', template: '' })
class MockChildComponent { @Input() data: any; }
declarations: [ParentComponent, MockChildComponent]
```

---

## RULE 17 — Coverage Thresholds

```javascript
// karma.conf.js:
check: { global: { statements: 90, branches: 85, functions: 90, lines: 90 } }

// jest.config.js:
coverageThreshold: { global: { statements: 90, branches: 85, functions: 90, lines: 90 } }
```

---

## Applied Style Rules (all versions)

1. `const` for all test body declarations
2. Explicit types — no implicit `any`
3. `toEqual` for objects/arrays · `toBe` for primitives
4. `toHaveBeenCalledWith(exactArgs)` not just `toHaveBeenCalled()`
5. Name subjects: `component`, `service`, `pipe`, `guard`
6. One assertion per `it()` where practical
7. One `describe` group per method or feature
8. Always note Angular version in spec file header comment:
   `// Angular version: 17 | TestBed: standalone | Runner: Jasmine`
