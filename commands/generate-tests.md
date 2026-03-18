---
name: angular-test:generate
description: >
  Generate complete, ready-to-run .spec.ts files targeting 100% coverage for any
  Angular source file. Auto-detects Angular version (14–19) and applies the correct
  testing patterns: module vs standalone, decorator vs signal inputs/outputs, classic
  control flow vs @if/@for/@switch/@defer, class vs functional guards/resolvers/
  interceptors, DestroyRef, resource(), zoneless, and NgRx. Accepts a single file,
  a directory, or --all for the entire project.
argument-hint: <file-path|directory|--all> [--dry-run] [--skip-existing] [--overwrite] [--only=<types>] [--threshold=<n>] [--runner=jasmine|jest] [--ng=<14|15|16|17|18|19>]
---

# /angular-test:generate

You are an expert Angular test engineer. Generate **complete, runnable `.spec.ts` files**
targeting 100% code coverage, applying patterns that match the detected Angular version.

---

## STEP 0 — DETECT ANGULAR VERSION

Before writing any test, detect the Angular version from `package.json`:

```
Read package.json → dependencies["@angular/core"] → strip semver prefix → major version
Examples: "^14.3.0" → 14 | "~17.1.0" → 17 | "18.2.0" → 18 | "^19.0.0" → 19
```

If `--ng=<version>` flag is passed, use that value and skip detection.
If `package.json` is not readable, default to **17** (broadest compatibility).

Store the detected version as `NG_VERSION` and use it throughout.

---

## VERSION FEATURE MATRIX

Use this matrix to decide which API patterns to apply:

| Feature | 14 | 15 | 16 | 17 | 18 | 19 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Module-based components (`declarations[]`) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Standalone components (`imports[]`) | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| Functional guards (`CanActivateFn`) | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| Functional resolvers (`ResolveFn`) | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| Functional interceptors (`HttpInterceptorFn`) | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| `inject()` in injection context | — | — | ✓ | ✓ | ✓ | ✓ |
| `@Input({ required: true })` | — | — | ✓ | ✓ | ✓ | ✓ |
| `DestroyRef` + `takeUntilDestroyed()` | — | — | ✓ | ✓ | ✓ | ✓ |
| `signal()`, `computed()`, `effect()` | — | — | preview | ✓ | ✓ | ✓ |
| `input()` signal inputs | — | — | preview | ✓ | ✓ | ✓ |
| `output()` signal outputs | — | — | — | ✓ | ✓ | ✓ |
| `model()` two-way signal | — | — | — | ✓ | ✓ | ✓ |
| `@if` / `@for` / `@switch` control flow | — | — | — | ✓ | ✓ | ✓ |
| `@defer` blocks | — | — | — | ✓ | ✓ | ✓ |
| `afterNextRender` / `afterRender` | — | — | — | ✓ | ✓ | ✓ |
| `linkedSignal()` | — | — | — | — | preview | ✓ |
| `resource()` / `httpResource()` | — | — | — | — | preview | ✓ |
| Zoneless (`provideExperimentalZonelessChangeDetection`) | — | — | — | — | preview | ✓ |
| Signal-based forms | — | — | — | — | — | preview |

---

## MODE DETECTION

| Input pattern | Mode |
|---|---|
| A `.ts` file path (not `.spec.ts`) | **Single-file** |
| A directory path | **Directory** |
| `--all` | **Project-wide** |

### Flags

| Flag | Default | Meaning |
|---|---|---|
| `--dry-run` | false | Print plan only — write nothing |
| `--skip-existing` | true | Skip source files that already have a spec |
| `--overwrite` | false | Regenerate even when spec exists |
| `--only=<types>` | all | `component,service,pipe,directive,guard,resolver,interceptor,reducer,effect,selector` |
| `--threshold=<n>` | 100 | Coverage target % for emitted config snippets |
| `--runner=jasmine\|jest` | jasmine | Output syntax preference |
| `--ng=<version>` | auto | Override detected Angular version |

---

## ARTIFACT CLASSIFICATION

Detect type by scanning the source file:

| Indicator | Type |
|---|---|
| `@Component` | Component |
| `@Injectable` + `HttpClient` in constructor | Service (HTTP) |
| `@Injectable`, no `HttpClient` | Service (pure) |
| `@Pipe` | Pipe |
| `@Directive` | Directive |
| `CanActivate` / `CanActivateFn` | Guard |
| `Resolve` / `ResolveFn` | Resolver |
| `HttpInterceptor` / `HttpInterceptorFn` | Interceptor |
| `createReducer` / `on(` | NgRx Reducer |
| `createEffect` | NgRx Effect |
| `createSelector` | NgRx Selector |

---

## ANGULAR 14 — TEST PATTERNS

### Component (module-based only in v14)

```typescript
import { ComponentFixture, TestBed, fakeAsync, tick, waitForAsync } from '@angular/core/testing';
import { By } from '@angular/platform-browser';
import { SimpleChange } from '@angular/core';

// v14: always use declarations[] — no standalone
describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  const mockUserService = jasmine.createSpyObj('UserService', {
    getUsers: of([]),
    deleteUser: of(null)
  });

  beforeEach(waitForAsync(() => {
    TestBed.configureTestingModule({
      declarations: [UserListComponent],  // module-based
      imports:      [ReactiveFormsModule, RouterTestingModule],
      providers:    [{ provide: UserService, useValue: mockUserService }]
    }).compileComponents();
  }));

  beforeEach(() => {
    fixture   = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
  });

  // @Input with decorator (v14 style)
  it('should reflect @Input() title in template', () => {
    component.title = 'Users';          // direct property assignment
    fixture.detectChanges();
    expect(fixture.debugElement.query(By.css('h1')).nativeElement.textContent).toBe('Users');
  });

  // @Input with ngOnChanges (v14 style)
  it('should respond to ngOnChanges', () => {
    component.ngOnChanges({
      title: new SimpleChange(null, 'Users', true)
    });
    expect(component.displayTitle).toBe('Users');
  });

  // @Output with EventEmitter (v14 style)
  it('should emit userDeleted on delete click', () => {
    const spy = spyOn(component.userDeleted, 'emit');
    component.users = [mockUser];
    fixture.detectChanges();
    fixture.debugElement.query(By.css('.delete-btn')).triggerEventHandler('click', null);
    expect(spy).toHaveBeenCalledWith(mockUser);
  });

  // *ngIf (classic control flow)
  it('should show empty state when users array is empty', () => {
    component.users = [];
    fixture.detectChanges();
    expect(fixture.debugElement.query(By.css('.empty-state'))).toBeTruthy();
  });

  // *ngFor
  it('should render one row per user', () => {
    component.users = [mockUser, mockUser2];
    fixture.detectChanges();
    expect(fixture.debugElement.queryAll(By.css('tr.user-row')).length).toBe(2);
  });

  // Subscription cleanup via takeUntil + destroy$
  it('should complete destroy$ on ngOnDestroy', () => {
    let completed = false;
    (component as any).destroy$.subscribe({ complete: () => (completed = true) });
    fixture.destroy();
    expect(completed).toBeTrue();
  });
});
```

### Guard — class-based (v14)

```typescript
// v14: CanActivate class
describe('AuthGuard', () => {
  let guard: AuthGuard;
  const mockAuthService = jasmine.createSpyObj('AuthService', ['isAuthenticated']);
  const mockRouter      = jasmine.createSpyObj('Router', ['navigate', 'createUrlTree']);

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        AuthGuard,
        { provide: AuthService, useValue: mockAuthService },
        { provide: Router,      useValue: mockRouter }
      ]
    });
    guard = TestBed.inject(AuthGuard);
  });

  it('should allow access when authenticated', () => {
    mockAuthService.isAuthenticated.and.returnValue(true);
    expect(guard.canActivate({} as any, {} as any)).toBeTrue();
  });

  it('should redirect when not authenticated', () => {
    mockAuthService.isAuthenticated.and.returnValue(false);
    guard.canActivate({} as any, {} as any);
    expect(mockRouter.navigate).toHaveBeenCalledWith(['/login']);
  });
});
```

### Interceptor — class-based (v14)

```typescript
describe('AuthInterceptor', () => {
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        AuthInterceptor,
        { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
        { provide: TokenService, useValue: { getToken: () => 'test-token' } }
      ]
    });
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should attach Bearer token to outgoing requests', () => {
    TestBed.inject(HttpClient).get('/api/data').subscribe();
    const req = httpMock.expectOne('/api/data');
    expect(req.request.headers.get('Authorization')).toBe('Bearer test-token');
    req.flush({});
  });
});
```

---

## ANGULAR 15 — ADDITIONAL PATTERNS

### Standalone Component

```typescript
// v15+: standalone: true → use imports[], not declarations[]
describe('UserCardComponent (standalone)', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserCardComponent, CommonModule],   // standalone in imports[]
      providers: [{ provide: UserService, useValue: mockUserService }]
    }).compileComponents();
  });
  // ... same fixture/test structure
});
```

### Functional Guard (v15+)

```typescript
// v15+: CanActivateFn — test as a plain function inside injection context
describe('authGuard (functional)', () => {
  let mockAuthService: jasmine.SpyObj<AuthService>;
  let mockRouter: jasmine.SpyObj<Router>;

  beforeEach(() => {
    mockAuthService = jasmine.createSpyObj('AuthService', ['isLoggedIn']);
    mockRouter      = jasmine.createSpyObj('Router', ['createUrlTree']);
    TestBed.configureTestingModule({
      providers: [
        { provide: AuthService, useValue: mockAuthService },
        { provide: Router,      useValue: mockRouter }
      ]
    });
  });

  it('should allow access when user is logged in', () => {
    mockAuthService.isLoggedIn.and.returnValue(true);
    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
    );
    expect(result).toBeTrue();
  });

  it('should return UrlTree to /login when not logged in', () => {
    mockAuthService.isLoggedIn.and.returnValue(false);
    mockRouter.createUrlTree.and.returnValue({ urlTree: '/login' } as any);
    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
    );
    expect(mockRouter.createUrlTree).toHaveBeenCalledWith(['/login']);
  });
});
```

### Functional Interceptor (v15+)

```typescript
// v15+: HttpInterceptorFn — use provideHttpClient(withInterceptors([fn]))
describe('authInterceptor (functional)', () => {
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(withInterceptors([authInterceptor])),
        provideHttpClientTesting(),
        { provide: TokenService, useValue: { getToken: () => 'my-token' } }
      ]
    });
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should attach Authorization header', () => {
    TestBed.inject(HttpClient).get('/api/protected').subscribe();
    const req = httpMock.expectOne('/api/protected');
    expect(req.request.headers.get('Authorization')).toBe('Bearer my-token');
    req.flush({});
  });
});
```

---

## ANGULAR 16 — ADDITIONAL PATTERNS

### Required Inputs (`@Input({ required: true })`)

```typescript
// v16+: required inputs — must set before detectChanges or TestBed will error
it('should display required userId input', () => {
  // Use setInput() — required inputs cannot be set via property assignment
  fixture.componentRef.setInput('userId', '42');
  fixture.detectChanges();
  expect(fixture.debugElement.query(By.css('.user-id')).nativeElement.textContent).toBe('42');
});

it('should throw during compilation if required input is missing', () => {
  // Required input omitted → Angular throws NG0950
  expect(() => fixture.detectChanges()).toThrowError(/NG0950/);
});
```

### Signals — `signal()`, `computed()`, `effect()` (v16 preview, v17+ stable)

```typescript
import { TestBed } from '@angular/core/testing';

// Testing a standalone component that uses signals
describe('CounterComponent (signals)', () => {
  it('should increment count signal', () => {
    fixture.componentRef.setInput('initialCount', 0);
    fixture.detectChanges();

    expect(component.count()).toBe(0);        // read signal value with ()
    component.increment();                    // method calls signal.update()
    expect(component.count()).toBe(1);
    fixture.detectChanges();
    expect(fixture.debugElement.query(By.css('.count')).nativeElement.textContent).toBe('1');
  });

  it('should compute doubled value', () => {
    component.count.set(5);                   // signal.set()
    // computed() updates synchronously
    expect(component.doubled()).toBe(10);
  });

  it('should run effect when signal changes', () => {
    const logSpy = spyOn(console, 'log');
    component.count.set(3);
    TestBed.flushEffects();                   // flush pending effects
    expect(logSpy).toHaveBeenCalledWith('count changed: 3');
  });
});
```

### `DestroyRef` + `takeUntilDestroyed()` (v16+)

```typescript
// v16+: takeUntilDestroyed() uses DestroyRef — fixture.destroy() still triggers it
it('should stop subscription on component destroy', () => {
  const subject = new Subject<number>();
  let received: number[] = [];

  fixture.detectChanges(); // triggers ngOnInit which subscribes via takeUntilDestroyed
  subject.subscribe(v => received.push(v));

  subject.next(1);
  expect(received).toContain(1);

  fixture.destroy();       // triggers DestroyRef → takeUntilDestroyed completes

  subject.next(2);
  // component's internal handler no longer runs after destroy
  expect(component.processedValues?.length).toBe(1);
});
```

### `inject()` in injection context (v16+)

```typescript
// Testing services that use inject() instead of constructor injection
it('should resolve injected dependency via TestBed.runInInjectionContext', () => {
  const result = TestBed.runInInjectionContext(() => {
    const service = inject(UserService);
    return service.getUser('1');
  });
  // result is an Observable — subscribe and test
});
```

---

## ANGULAR 17 — ADDITIONAL PATTERNS

### Signal Inputs `input()` and `input.required()` (v17+)

```typescript
// v17+: input<T>() — CANNOT assign directly, always use setInput()
describe('UserCardComponent (signal inputs)', () => {
  it('should reflect signal input in template', () => {
    // ✅ Correct: use componentRef.setInput()
    fixture.componentRef.setInput('user', mockUser);
    fixture.detectChanges();
    expect(fixture.debugElement.query(By.css('.name')).nativeElement.textContent)
      .toBe(mockUser.name);
  });

  it('should update when signal input changes', () => {
    fixture.componentRef.setInput('user', mockUser);
    fixture.detectChanges();

    fixture.componentRef.setInput('user', { ...mockUser, name: 'Bob' });
    fixture.detectChanges();

    expect(fixture.debugElement.query(By.css('.name')).nativeElement.textContent).toBe('Bob');
  });
});
```

### Signal Outputs `output()` (v17+)

```typescript
// v17+: output<T>() — subscribe via OutputRef, NOT spyOn .emit
describe('UserCardComponent (signal outputs)', () => {
  it('should emit userSelected via output()', () => {
    const emitted: User[] = [];
    // Subscribe to the OutputRef
    component.userSelected.subscribe((u: User) => emitted.push(u));

    fixture.componentRef.setInput('user', mockUser);
    fixture.detectChanges();
    fixture.debugElement.query(By.css('.card')).triggerEventHandler('click', null);

    expect(emitted.length).toBe(1);
    expect(emitted[0]).toEqual(mockUser);
  });
});
```

### `model()` — Two-way signal binding (v17+)

```typescript
// v17+: model<T>() — read with (), write with .set()
it('should reflect model() value in template', () => {
  fixture.componentRef.setInput('value', 'hello');
  fixture.detectChanges();
  expect(component.value()).toBe('hello');
});

it('should emit model change on user interaction', () => {
  const emitted: string[] = [];
  component.valueChange.subscribe((v: string) => emitted.push(v));

  fixture.componentRef.setInput('value', 'hello');
  fixture.detectChanges();

  const input = fixture.debugElement.query(By.css('input'));
  input.nativeElement.value = 'world';
  input.nativeElement.dispatchEvent(new Event('input'));

  expect(emitted).toContain('world');
});
```

### New Control Flow — `@if` / `@else` (v17+)

```typescript
// v17+: @if block replaces *ngIf — same By.css() queries, same detectChanges() pattern
it('should show content when @if condition is true', () => {
  component.isLoggedIn = true;     // or signal: component.isLoggedIn.set(true)
  fixture.detectChanges();
  expect(fixture.debugElement.query(By.css('.dashboard'))).toBeTruthy();
  expect(fixture.debugElement.query(By.css('.login-prompt'))).toBeNull();
});

it('should show @else branch when condition is false', () => {
  component.isLoggedIn = false;
  fixture.detectChanges();
  expect(fixture.debugElement.query(By.css('.dashboard'))).toBeNull();
  expect(fixture.debugElement.query(By.css('.login-prompt'))).toBeTruthy();
});
```

### New Control Flow — `@for` / `@empty` (v17+)

```typescript
// v17+: @for replaces *ngFor, adds @empty block for empty collections
it('should render items in @for loop', () => {
  component.items = [{ id: 1 }, { id: 2 }, { id: 3 }];
  fixture.detectChanges();
  expect(fixture.debugElement.queryAll(By.css('.item')).length).toBe(3);
});

it('should show @empty block when collection is empty', () => {
  component.items = [];
  fixture.detectChanges();
  expect(fixture.debugElement.query(By.css('.empty-message'))).toBeTruthy();
  expect(fixture.debugElement.queryAll(By.css('.item')).length).toBe(0);
});
```

### New Control Flow — `@switch` / `@case` / `@default` (v17+)

```typescript
// v17+: @switch replaces *ngSwitch
it('should render admin view for admin role', () => {
  component.role = 'admin';
  fixture.detectChanges();
  expect(fixture.debugElement.query(By.css('.admin-panel'))).toBeTruthy();
});

it('should render user view for user role', () => {
  component.role = 'user';
  fixture.detectChanges();
  expect(fixture.debugElement.query(By.css('.user-panel'))).toBeTruthy();
});

it('should render @default block for unknown role', () => {
  component.role = 'unknown';
  fixture.detectChanges();
  expect(fixture.debugElement.query(By.css('.default-panel'))).toBeTruthy();
});
```

### `@defer` blocks (v17+)

```typescript
import { DeferBlockFixture, DeferBlockState } from '@angular/core/testing';

describe('ArticleComponent (@defer)', () => {
  it('should show @placeholder initially', async () => {
    fixture.detectChanges();
    const deferBlock = (await fixture.getDeferBlocks())[0];
    await deferBlock.render(DeferBlockState.Placeholder);
    expect(fixture.debugElement.query(By.css('.placeholder'))).toBeTruthy();
  });

  it('should show @loading state', async () => {
    fixture.detectChanges();
    const deferBlock = (await fixture.getDeferBlocks())[0];
    await deferBlock.render(DeferBlockState.Loading);
    expect(fixture.debugElement.query(By.css('.loading-spinner'))).toBeTruthy();
  });

  it('should show @error state on failure', async () => {
    fixture.detectChanges();
    const deferBlock = (await fixture.getDeferBlocks())[0];
    await deferBlock.render(DeferBlockState.Error);
    expect(fixture.debugElement.query(By.css('.error-message'))).toBeTruthy();
  });

  it('should render deferred content when complete', async () => {
    fixture.detectChanges();
    const deferBlock = (await fixture.getDeferBlocks())[0];
    await deferBlock.render(DeferBlockState.Complete);
    expect(fixture.debugElement.query(By.css('app-heavy-chart'))).toBeTruthy();
  });
});
```

### New Lifecycle Hooks `afterNextRender` / `afterRender` (v17+)

```typescript
// afterNextRender runs after the first render cycle
// Test by calling detectChanges() and asserting on side effects
it('should execute afterNextRender callback', fakeAsync(() => {
  const callbackSpy = spyOn(component as any, 'onAfterRender');
  fixture.detectChanges();
  tick();    // flush microtask queue
  expect(callbackSpy).toHaveBeenCalledTimes(1);
}));
```

---

## ANGULAR 18 — ADDITIONAL PATTERNS

### `linkedSignal()` (v18 preview, v19 stable)

```typescript
// linkedSignal() creates a writable signal derived from a source signal
it('should reset linkedSignal when source changes', () => {
  // Set the source signal
  component.selectedCategory.set('books');
  fixture.detectChanges();

  // linkedSignal should reflect derived value
  expect(component.filteredItems()).toEqual(mockBooks);

  // Changing source resets the linked signal
  component.selectedCategory.set('music');
  fixture.detectChanges();
  expect(component.filteredItems()).toEqual(mockMusic);
});
```

### `resource()` — Async resource API (v18 preview, v19 stable)

```typescript
// resource() manages async data loading with status signals
it('should expose loading status while fetching', fakeAsync(() => {
  fixture.detectChanges();
  expect(component.userResource.status()).toBe(ResourceStatus.Loading);

  tick();
  fixture.detectChanges();
  expect(component.userResource.status()).toBe(ResourceStatus.Resolved);
  expect(component.userResource.value()).toEqual(mockUser);
}));

it('should expose error status on failure', fakeAsync(() => {
  mockUserService.getUser.and.returnValue(throwError(() => new Error('Not found')));
  fixture.detectChanges();
  tick();
  fixture.detectChanges();
  expect(component.userResource.status()).toBe(ResourceStatus.Error);
  expect(component.userResource.error()).toBeTruthy();
}));
```

### `httpResource()` — HTTP-backed resource (v18 preview, v19 stable)

```typescript
// httpResource() wraps an HTTP call as a reactive resource
describe('UsersComponent (httpResource)', () => {
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [UsersComponent],
      providers: [provideHttpClient(), provideHttpClientTesting()]
    });
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should load users via httpResource', fakeAsync(() => {
    fixture.detectChanges();

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);

    tick();
    fixture.detectChanges();

    expect(component.usersResource.value()).toEqual(mockUsers);
    expect(component.usersResource.status()).toBe(ResourceStatus.Resolved);
  }));

  it('should handle HTTP error in httpResource', fakeAsync(() => {
    fixture.detectChanges();

    const req = httpMock.expectOne('/api/users');
    req.flush('Server error', { status: 500, statusText: 'Internal Server Error' });

    tick();
    fixture.detectChanges();

    expect(component.usersResource.status()).toBe(ResourceStatus.Error);
  }));
});
```

### Zoneless Change Detection (v18 preview, v19+)

```typescript
// v18+: provideExperimentalZonelessChangeDetection()
// detectChanges() still works; async ops need explicit change marking
describe('CounterComponent (zoneless)', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [CounterComponent],
      providers: [
        provideExperimentalZonelessChangeDetection()  // zoneless
      ]
    }).compileComponents();
  });

  it('should update view when signal changes (zoneless)', async () => {
    fixture.detectChanges();
    component.count.set(5);
    await fixture.whenStable();          // wait for scheduler
    fixture.detectChanges();
    expect(fixture.debugElement.query(By.css('.count')).nativeElement.textContent).toBe('5');
  });
});
```

---

## ANGULAR 19 — ADDITIONAL PATTERNS

### Signal-based Forms (v19 experimental)

```typescript
// v19: experimental signal forms
import { FormField, SignalForm } from '@angular/forms';  // future API placeholder

it('should validate signal form field', () => {
  component.form.fields.email.set('invalid-email');
  expect(component.form.fields.email.errors()).toContain('email');

  component.form.fields.email.set('user@example.com');
  expect(component.form.fields.email.errors()).toBeNull();
  expect(component.form.valid()).toBeTrue();
});
```

### Stable `resource()` and `httpResource()` (v19)

Same patterns as v18 above — in v19 these APIs are stable (not prefixed with Experimental).
Remove any `Experimental` prefix from imports and providers:

```typescript
// v19: stable — no "Experimental" prefix needed
providers: [
  provideZonelessChangeDetection()  // replaces provideExperimentalZonelessChangeDetection
]
```

---

## VERSION-AWARE TESTBED BUILDER

When generating a spec file, apply the correct TestBed setup based on `NG_VERSION`:

```typescript
// VERSION-AWARE TestBed setup decision tree:

if (NG_VERSION >= 15 && component has standalone: true) {
  // Standalone component
  TestBed.configureTestingModule({
    imports: [ComponentUnderTest, ...neededImports],
    providers: [mockProviders]
  });
} else {
  // Module-based component (all versions, default in v14)
  TestBed.configureTestingModule({
    declarations: [ComponentUnderTest, ...stubChildren],
    imports: [...neededImports],
    providers: [mockProviders]
  });
}

if (NG_VERSION >= 18 && source uses zoneless) {
  providers.push(provideExperimentalZonelessChangeDetection());
}

if (source uses httpResource()) {
  providers.push(provideHttpClient(), provideHttpClientTesting());
} else if (source uses HttpClient) {
  imports.push(HttpClientTestingModule);
}

if (source uses Store) {
  providers.push(provideMockStore({ initialState }));
}
```

---

## MOCK CREATION — VERSION-AWARE

```typescript
// All versions — Jasmine:
const mockService = jasmine.createSpyObj('ServiceName', {
  method1: of(mockData),
  method2: of(null)
});

// All versions — Jest:
const mockService = {
  method1: jest.fn().mockReturnValue(of(mockData)),
  method2: jest.fn().mockReturnValue(of(null))
};

// v15+ functional guards/resolvers — use TestBed.runInInjectionContext():
TestBed.runInInjectionContext(() => myGuard(route, state));

// v16+ inject()-based services — they resolve from TestBed automatically
// No special mock setup needed; the provider[] handles DI

// v17+ signal inputs — NEVER assign directly:
// ❌ component.name = 'Alice';
// ✅ fixture.componentRef.setInput('name', 'Alice');

// v17+ signal outputs — NEVER spyOn .emit, subscribe to OutputRef:
// ❌ spyOn(component.nameChange, 'emit')
// ✅ component.nameChange.subscribe(v => captured = v);
```

---

## COVERAGE PLAN TAXONOMY (all versions)

### Components
- Every `@Input()` / `input()` / `input.required()` — binding test + DOM assertion
- Every `@Output()` / `output()` / `model()` — emission test
- Every public method
- All lifecycle hooks: `ngOnInit`, `ngOnChanges`, `ngOnDestroy`, `ngAfterViewInit`
- New hooks (v17+): `afterNextRender`, `afterRender`
- Every `*ngIf` / `@if` branch (truthy + falsy + `@else`)
- Every `*ngFor` / `@for` (populated + empty + `@empty` block)
- Every `*ngSwitch` / `@switch` case + default
- Every `@defer` state: `@placeholder`, `@loading`, `@error`, `@complete`
- Every `signal()` — `.set()`, `.update()`, `.mutate()`
- Every `computed()` — derived value test
- Every `effect()` — `TestBed.flushEffects()`
- Every `resource()` / `httpResource()` — loading, resolved, error states
- Every service call — happy + error paths
- Every router navigation
- Every form control — valid, invalid, submission

### Services
- Every public method — happy path + error path
- All HTTP calls: URL, method, headers, body
- HTTP error responses (4xx, 5xx)
- Retry/caching logic
- Constructor + `inject()` side effects
- `takeUntilDestroyed()` cleanup

### NgRx
- Reducer: initial state + every `on()` handler
- Effect: marble test happy path + error path per effect
- Selector: `projector()` with direct inputs + MockStore

---

## BATCH MODES (Directory / --all)

### Directory mode output

```
DISCOVERY: src/app/users/ (Angular 17 detected)
═══════════════════════════════════════════════════════
File                         Type          NG APIs      Spec
─────────────────────────────────────────────────────────
user-list.component.ts       Component     standalone   ✗
user.service.ts              Service(HTTP) inject()     ✗
users.effects.ts             NgRx Effect   —            ✓
user.guard.ts                Guard         CanActivateFn✗
```

### Project-wide confirmation prompt

```
PROJECT-WIDE GENERATION PLAN (Angular 17)
══════════════════════════════════════════
Total files  : 23 specs to generate
Skipping     : 12 (already have specs)
Order        : NgRx → Services → Guards → Directives → Components

A) Generate ALL 23 files
B) Generate by module (one at a time)
C) Specific types only (e.g. "only components")
D) Show full file list

Reply with A, B, C, D, or an instruction.
```

---

## CONSTRAINTS

- Detect Angular version before writing a single test — never assume a version
- For NG >= 15 + standalone: use `imports[]` not `declarations[]`
- For NG >= 17 signal inputs: use `setInput()` — never direct property assignment
- For NG >= 17 signal outputs: use `subscribe()` — never `spyOn(.emit)`
- For NG >= 17 `@defer`: use `DeferBlockFixture` / `getDeferBlocks()`
- For NG >= 18 zoneless: add `provideExperimentalZonelessChangeDetection()`
- For NG >= 18 `httpResource()`: use `provideHttpClient()` + `provideHttpClientTesting()`
- Never use `NO_ERRORS_SCHEMA` without a comment
- Never use `any` in assertions
- Always call `httpMock.verify()` in `afterEach`
- In project-wide mode: always confirm before generating
- If `--dry-run`: plan only, zero files written
