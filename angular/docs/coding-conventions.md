# Angular Coding Style Reference

> **Purpose:** AI-consumable style reference for Angular projects.  
> **Scope:** Covers both an Angular SPA (shell application) and a companion Angular component library consumed by it. Patterns are shared unless a section explicitly marks a rule as **App only** or **Library only**.  
> **Stack:** Angular 12–19+ · TypeScript · NgRx · RxJS  
> **Versioning note:** Where a pattern requires a minimum Angular version, that version is noted inline. **Modern syntax** (Angular 14–19+) is shown first and is the preferred default. **Legacy syntax** (Angular 12–13, NgModule-based) is shown second and should be used only when the project targets an older Angular version.

---

## 1. File & Directory Naming

| Artefact | Convention | Example |
|---|---|---|
| Component files | `kebab-case.component.{ts,html,scss}` | `user-profile.component.ts` |
| Service | `kebab-case.service.ts` | `auth.service.ts` |
| Module | `kebab-case.module.ts` | `shared.module.ts` |
| Guard | `kebab-case.guard.ts` | `auth.guard.ts` |
| Pipe | `kebab-case.pipe.ts` | `group-by.pipe.ts` |
| Directive | `kebab-case.directive.ts` | `auto-focus.directive.ts` |
| Model / Interface file | `kebab-case.model.ts` or `kebab-case.interface.ts` | `user-profile.model.ts` |
| Enum file | `kebab-case.enum.ts` (single enum) or grouped as `feature-enums.ts` | `grid-enums.ts` |
| Constants file | `kebab-case.constants.ts` | `app.constants.ts` |
| Utility file | `utility.ts` or `utility-functions.ts` | `date-utils.ts` |
| Routes file (standalone) | `kebab-case.routes.ts` | `user.routes.ts`, `app.routes.ts` |
| Library contracts | Inside a `contracts/` subdirectory per feature | `contracts/grid.ts` |
| Library public API | `public-api.ts` barrel at the library root | `public-api.ts` |

### Component selectors

| Context | Convention | Example |
|---|---|---|
| Shell app components | Consistent prefix set in `angular.json` (`prefix` field). Angular CLI defaults to `app-`; projects may override this. All app components use the same prefix. | `app-user-profile`, `app-nav-bar` |
| Third-party widget wrapper components | Same prefix as app components, with an additional qualifier segment to identify the vendor or widget family | `app-dx-text-box`, `app-dx-grid` |
| Library components | A distinct short prefix chosen for the library, different from the app prefix, set in the library's own `angular.json` entry | `lib-data-grid`, `lib-toast` |

---

## 2. Class & Symbol Naming

### 2.1 Classes and functional symbols

**NgModule-based components, services, guards, interceptors:**

```
Components   → PascalCase + "Component"   → UserProfileComponent
Services     → PascalCase + "Service"     → AuthService, UserDataService
Modules      → PascalCase + "Module"      → SharedModule, CoreModule
Pipes        → PascalCase + "Pipe"        → GroupByPipe
Guards       → PascalCase + "Guard"       → AuthGuard
Models       → PascalCase + "Model"       → ViewSelectionModel
Interceptors → PascalCase + "Interceptor" → AuthInterceptor
```

**Functional guards, interceptors, and resolvers (Angular 15+)** are exported as `const` in camelCase, without a class suffix, because they are plain functions rather than classes:

```typescript
// Functional guard — camelCase const
export const authGuard: CanActivateFn = (route, state) => { ... };

// Functional interceptor — camelCase const
export const authInterceptor: HttpInterceptorFn = (req, next) => { ... };

// Functional resolver — camelCase const
export const userResolver: ResolveFn<IUser> = route => inject(UserService).getUser(+route.paramMap.get('id'));
```

### 2.2 Interfaces

Always prefix with `I` + PascalCase:

```typescript
// ✓ Correct
interface IGridColumn { ... }
interface IValidationRule { ... }
interface IUserProfile { ... }

// ✗ Wrong — missing I prefix
interface GridColumn { ... }
interface UserProfile { ... }
```

### 2.3 Properties and methods

All class properties and methods use **camelCase**. Access level is expressed with TypeScript keywords (`private`, `protected`, `public`). No underscore prefix convention — rely on the `private` keyword rather than naming to signal access level.

```typescript
// ✓ Correct
private isLoading = false;
public userList: IUser[] = [];

// ✗ Avoid
private _isLoading = false;   // underscore prefix — unnecessary
public UserList = [];          // PascalCase on an instance property
```

**Acceptable exception:** An underscore-prefixed backing field is tolerated when it is the private counterpart of a public `@Input()` getter/setter pair, and only in that pairing context.

```typescript
// Acceptable (not required)
@Input() set items(value: IItem[]) { this._items = value; this.process(); }
get items() { return this._items; }
private _items: IItem[] = [];
```

### 2.4 Enums

Enum **type names**: PascalCase + `Enum` suffix.  
Enum **member values**: camelCase.

```typescript
// ✓ Correct
export enum GridEventEnum {
    selectionChanged,
    contentReady,
    rowClick,
}

export enum ToasterTypeEnum {
    success,
    error,
    warning,
}

// ✗ Avoid — wrong casing, missing Enum suffix
export enum gridEvent { ... }
export enum ToasterType { ... }
```

When the string key of a numeric enum member is needed, use the reverse-mapping pattern:

```typescript
const keyName = StorageKeyEnum[StorageKeyEnum.conditionalFormatting];
// → "conditionalFormatting"
```

### 2.5 Constants

Named constant objects use PascalCase:

```typescript
export const APIEndPoints = { ... };
export const DefaultPageSizes = [10, 25, 50, 100];
```

True scalar constants that will never be objects use `SCREAMING_SNAKE_CASE`:

```typescript
export const MAX_RETRY_COUNT = 3;
export const DEFAULT_TIMEOUT_MS = 5000;
```

---

## 3. Component Architecture

### 3.1 Standalone vs NgModule-based components

**Standalone components (Angular 14+, default in Angular 17+)** are the preferred approach for new code. They declare their own dependencies directly in `@Component` and do not belong to an `NgModule`:

```typescript
// ✓ Standalone component — Angular 17+ default
@Component({
    selector: 'app-user-profile',
    standalone: true,
    imports: [CommonModule, RouterLink, UserAvatarComponent],
    templateUrl: './user-profile.component.html',
    styleUrls: ['./user-profile.component.scss'],
    changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserProfileComponent { ... }
```

**NgModule-based components (Angular 12–16, or projects not yet migrated)** declare the component in an `NgModule` and import dependencies through that module:

```typescript
// Legacy NgModule-based component
@Component({
    selector: 'app-user-profile',
    templateUrl: './user-profile.component.html',
    styleUrls: ['./user-profile.component.scss'],
})
export class UserProfileComponent { ... }

// NgModule registers the component
@NgModule({
    declarations: [UserProfileComponent],
    imports: [SharedModule],
})
export class UserProfileModule { }
```

### 3.2 Canonical member ordering

```typescript
@Component({ ... })
export class ExampleComponent implements OnInit, OnDestroy {

    // 1. Static identifier (if used for monitoring/logging — see §3.3)
    static id = 'ExampleComponent';

    // 2. Decorator-based queries
    @ViewChild('dataGrid') grid: DataGridComponent;

    // 3. Signal queries (Angular 17.3+)
    protected readonly panel = viewChild<PanelComponent>('panel');

    // 4. Decorator-based inputs — plain properties first, then getter/setter pairs
    @Input() public title: string;
    @Input() public set options(value: OptionsModel) { ... }

    // 5. Signal inputs (Angular 17.1+)
    protected readonly count = input(0);
    protected readonly isDisabled = input(false, { transform: booleanAttribute });

    // 6. Outputs — no public keyword (Angular convention)
    @Output() selectionChanged = new EventEmitter<SelectionEvent>();

    // 7. Signal outputs (Angular 17.3+)
    protected readonly rowSelected = output<IRow>();

    // 8. Public / protected properties and writable signals
    public layoutList: LayoutModel[] = [];
    protected readonly isLoading = signal(false);

    // 9. Computed values
    protected readonly itemCount = computed(() => this.items().length);

    // 10. Private properties
    private destroy$ = new Subject<void>();
    private readonly destroyRef = inject(DestroyRef);  // Angular 16+

    // 11. Constructor (legacy) or inject() calls declared above as fields (modern)
    constructor(private userService: UserService) { }

    // 12. Lifecycle hooks (in Angular's execution order)
    ngOnInit(): void { ... }
    ngAfterViewInit(): void { ... }
    ngOnDestroy(): void { ... }

    // 13. Public methods
    public onRowClick(event: RowClickEvent): void { ... }

    // 14. Private methods
    private buildColumns(): void { ... }
}
```

### 3.3 Component identifier for logging *(App only, optional)*

If the project uses a base component or monitoring service that requires a component identifier for page-view logging, feature components declare a `static id` string matching the class name exactly:

```typescript
static id = 'UserProfileComponent';
```

This is a project-specific requirement — only apply it if the project's monitoring infrastructure expects it. Library components do not use this pattern.

### 3.4 Base component extension *(App only)*

Shell feature components may extend a shared `BaseComponent` that wires up cross-cutting concerns (e.g., monitoring, page-view tracking). The constructor calls `super()` with a human-readable display name:

```typescript
export class DashboardComponent extends BaseComponent implements OnInit {
    static id = 'DashboardComponent';

    constructor(private dataService: DataService) {
        super('Dashboard');
    }
}
```

Library components do **not** extend the application's `BaseComponent` — the library must not depend on application-level infrastructure.

### 3.5 @HostBinding for host element classes

Components that apply a structural container class to their host element use `@HostBinding`:

```typescript
@HostBinding('class.component-container') readonly hostClass = true;
```

### 3.6 @Input patterns (decorator-based, Angular 12+)

Use a plain property when no side-effect is needed on assignment:

```typescript
@Input() public title: string;
@Input() public isDisabled = false;
```

Use a getter/setter when the assignment must trigger logic:

```typescript
@Input() public set dataSource(value: IRecord[]) {
    this._dataSource = value;
    this.rebuildGrid();
}
public get dataSource(): IRecord[] { return this._dataSource; }
private _dataSource: IRecord[];
```

**Do not call lifecycle hooks inside a setter.** Extract shared initialisation logic into a private method and call it from both `ngOnInit()` and the setter.

```typescript
// ✗ Wrong
@Input() set config(val: IConfig) {
    this._config = val;
    this.ngOnInit(); // ← never do this
}

// ✓ Correct
@Input() set config(val: IConfig) {
    this._config = val;
    this.applyConfig(); // ← private method
}
ngOnInit(): void { this.applyConfig(); }
private applyConfig(): void { ... }
```

### 3.7 Signal inputs (Angular 17.1+, preferred for new components)

Signal inputs replace `@Input()` in standalone components on Angular 17.1+. They are read-only signals in the class body:

```typescript
// Basic input with default value
protected readonly title = input<string>('');

// Required input — Angular enforces a value is passed; throws if missing
protected readonly dataSource = input.required<IRecord[]>();

// Input with transform — converts string "true"/"false" to boolean automatically
protected readonly disabled = input(false, { transform: booleanAttribute });

// Use in template or in computed()
protected readonly displayTitle = computed(() => this.title().toUpperCase());
```

### 3.8 @Output patterns (decorator-based, Angular 12+)

No `public` keyword on `@Output()` (Angular convention). Type the `EventEmitter` where the event shape is known:

```typescript
// ✓ Typed
@Output() rowSelected = new EventEmitter<IRow>();
@Output() actionClick = new EventEmitter<{ type: string; row: IRow }>();

// ✗ Avoid untyped
@Output() somethingHappened = new EventEmitter();
```

### 3.9 Signal outputs (Angular 17.3+, preferred for new components)

Signal outputs replace `@Output()` in standalone components on Angular 17.3+:

```typescript
protected readonly rowSelected = output<IRow>();
protected readonly actionClick = output<{ type: string; row: IRow }>();

// Emit with .emit() — same API as EventEmitter
this.rowSelected.emit(row);
```

### 3.10 Signal queries (Angular 17.3+, preferred for new components)

Signal queries replace `@ViewChild` / `@ViewChildren` / `@ContentChild` / `@ContentChildren`:

```typescript
// Optional query — returns Signal<T | undefined>
protected readonly dropdown = viewChild<DxDropDownComponent>('dropdown');

// Required query — throws at runtime if no match is found
protected readonly grid = viewChild.required<DxGridComponent>('grid');

// Multiple children — returns Signal<readonly T[]>
protected readonly panels = viewChildren<PanelComponent>('panel');
```

### 3.11 Smart and Presentational Components

Separate components into two distinct roles to maximise reusability and testability.

**Smart (container) components** own business logic. They inject services, dispatch NgRx actions, manage subscriptions, and know about routing. They pass data down to presentational children via `@Input()` and react to their `@Output()` events.

**Presentational (dumb) components** only render data passed to them. They have no direct service dependencies, emit events for every user action, and are fully controlled by their parent. Because they have no side effects, they are trivially testable and reusable.

```typescript
// ✓ Smart component — knows about the service and routing
@Component({ selector: 'app-user-list-page', ... })
export class UserListPageComponent {
    private readonly userService = inject(UserService);
    private readonly router = inject(Router);
    protected readonly users = toSignal(this.userService.getUsers(), { initialValue: [] as IUser[] });

    onUserSelected(user: IUser): void {
        this.router.navigate(['/users', user.id]);
    }
}

// ✓ Presentational component — no services, fully driven by inputs/outputs
@Component({
    selector: 'app-user-table',
    standalone: true,
    changeDetection: ChangeDetectionStrategy.OnPush,
    ...
})
export class UserTableComponent {
    protected readonly users = input.required<IUser[]>();
    protected readonly userSelected = output<IUser>();
}
```

```html
<!-- Smart component template wires data down and events up -->
<app-user-table
    [users]="users()"
    (userSelected)="onUserSelected($event)">
</app-user-table>
```

As a rule: if a component needs to `inject(SomeFeatureService)` to do its job, it is a smart component and belongs in a feature directory. Presentational components should have no `inject()` calls beyond UI utilities (e.g., a translate pipe or a theme service).

### 3.12 Content Projection (ng-content)

Use `ng-content` to make layout and wrapper components extensible without coupling them to their content. This is preferred over passing large templates or complex objects as `@Input()` properties.

**Single slot — project any content into a container:**

```typescript
// card.component.html
<div class="card-container">
    <ng-content></ng-content>
</div>
```

```html
<!-- Consumer -->
<app-card>
    <h2>Title</h2>
    <p>Any content here is projected into the card wrapper.</p>
</app-card>
```

**Named slots — project content into specific regions of a layout:**

```html
<!-- panel.component.html -->
<div class="panel">
    <div class="panel-header">
        <ng-content select="[slot=header]"></ng-content>
    </div>
    <div class="panel-body">
        <ng-content select="[slot=body]"></ng-content>
    </div>
    <div class="panel-footer">
        <ng-content select="[slot=footer]"></ng-content>
    </div>
</div>
```

```html
<!-- Consumer -->
<app-panel>
    <ng-container slot="header">Panel Title</ng-container>
    <ng-container slot="body">
        <app-data-grid [dataSource]="records" />
    </ng-container>
    <ng-container slot="footer">
        <button (click)="onSave()">Save</button>
    </ng-container>
</app-panel>
```

The layout component's internal structure stays stable while consumers freely control the content — the panel does not need to know what goes inside it.

### 3.13 Component Promotion Strategy

A component starts life in its feature directory. Promote it when it earns broader reuse:

| Stage | Location | Criteria |
|---|---|---|
| **Feature** | Inside the feature directory | Used only within one feature; may have feature-specific service dependencies |
| **Shared** | `SharedModule` or standalone shared component | Used in 2+ features; all data via `@Input()`, all actions via `@Output()`; no feature-specific service dependencies |
| **Library** | Component library (`public-api.ts`) | Used across multiple applications; stable, versioned API; developed and tested independently of any app |

Before promoting, verify the component is genuinely stateless: remove direct service calls, convert internal state to inputs, and convert internal actions to outputs. A component that still injects a feature-specific service is not ready to be shared.

---

## 4. Dependency Injection

### 4.1 inject() function (Angular 14+, preferred)

Use `inject()` in the class body (field initialiser position) as the primary dependency injection pattern for new components. This eliminates constructor boilerplate and works in standalone components, functional guards, interceptors, and resolvers:

```typescript
import { inject } from '@angular/core';

@Component({ standalone: true, ... })
export class UserProfileComponent {
    private readonly userService = inject(UserService);
    private readonly router = inject(Router);
    private readonly destroyRef = inject(DestroyRef);   // Angular 16+
}
```

`inject()` is also valid inside factory functions, functional guards, functional interceptors, and resolvers — any place where an Angular injection context exists.

### 4.2 Constructor injection (Angular 12+, legacy)

Constructor injection remains valid and must be used when the project targets Angular 12–13, or when extending a base class that already has a constructor parameter list:

```typescript
export class UserProfileComponent {
    constructor(
        private readonly userService: UserService,
        private readonly router: Router,
    ) { }
}
```

### 4.3 Injectable scoping rules

| Scope | Declaration | Registration |
|---|---|---|
| Application-wide singleton | `@Injectable({ providedIn: 'root' })` | Automatic, tree-shakeable |
| Component-scoped (new instance per host component) | `@Injectable()` (no `providedIn`) | Listed in `providers: [MyService]` on the component |

```typescript
// Application singleton — available everywhere, one instance
@Injectable({ providedIn: 'root' })
export class NavigationService { }

// Component-scoped — new instance per host, destroyed with host
@Injectable()
export class GridStateService { }

// Component that owns a scoped service
@Component({
    providers: [GridStateService],  // Scoped: new instance per this component's subtree
})
export class GridComponent {
    private readonly gridState = inject(GridStateService);  // modern
    // constructor(private gridState: GridStateService) { } // legacy
}
```

---

## 5. Application Bootstrap

### 5.1 Standalone bootstrap (Angular 14+, preferred)

New applications use `bootstrapApplication()` with functional providers:

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideStore } from '@ngrx/store';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';
import { authInterceptor } from './app/auth.interceptor';

bootstrapApplication(AppComponent, {
    providers: [
        provideRouter(routes),
        provideHttpClient(withInterceptors([authInterceptor])),
        provideStore(appReducers),
    ],
}).catch(err => console.error(err));
```

### 5.2 NgModule bootstrap (Angular 12+, legacy)

Existing applications use `platformBrowserDynamic().bootstrapModule(AppModule)`:

```typescript
// main.ts — NgModule-based
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';

platformBrowserDynamic().bootstrapModule(AppModule)
    .catch(err => console.error(err));
```

---

## 6. Module & Routing Organisation

### 6.1 Standalone routing (Angular 14+, preferred)

Routes are defined in `*.routes.ts` files and reference standalone components. Lazy-loaded routes use `loadComponent` (for a single component) or `loadChildren` (for a sub-route array):

```typescript
// app.routes.ts
export const routes: Routes = [
    {
        path: 'users',
        loadComponent: () => import('./users/user-list.component').then(m => m.UserListComponent),
    },
    {
        path: 'admin',
        loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes),
        canActivate: [authGuard],
    },
];
```

### 6.2 NgModule routing (Angular 12+, legacy)

Feature modules use a `RoutingModule` that references `NgModule`-declared components:

```typescript
// users-routing.module.ts
const routes: Routes = [
    { path: '', component: UserListComponent },
    { path: ':id', component: UserDetailComponent },
];

@NgModule({
    imports: [RouterModule.forChild(routes)],
    exports: [RouterModule],
})
export class UsersRoutingModule { }
```

### 6.3 SharedModule *(NgModule-based projects only)*

`SharedModule` is the single point of entry for all reusable UI declarations — third-party widget modules, wrapper components, pipes, and common directives. It exports everything it declares and contains no providers. Every feature module imports `SharedModule`.

```typescript
@NgModule({
    imports: [CommonModule, FormsModule, /* third-party modules */],
    declarations: [WrapperComponent, GroupByPipe],
    exports: [CommonModule, FormsModule, /* third-party modules */, WrapperComponent, GroupByPipe],
})
export class SharedModule {}
```

### 6.4 CoreModule *(App only, NgModule-based projects only)*

`CoreModule` is a singleton aggregator imported once in `AppModule`. It enforces singleton behaviour with the `@Optional() @SkipSelf()` guard:

```typescript
constructor(@Optional() @SkipSelf() parentModule?: CoreModule) {
    if (parentModule) {
        throw new Error('CoreModule is already loaded. Import it in AppModule only.');
    }
}
```

Contains authentication infrastructure (interceptors, auth service) and no UI declarations.

### 6.5 Library public API *(Library only)*

The library exposes one or more public `NgModule` classes or standalone components as its integration surface. Public API surface is controlled via `public-api.ts` — only what is exported from that file is part of the library's supported contract:

```typescript
// public-api.ts
export * from './lib/grid/grid.module';
export * from './lib/grid/grid.component';
export * from './lib/grid/contracts/grid';
export * from './lib/toast/toast.module';
```

---

## 7. Service Patterns

### 7.1 HTTP abstraction

HTTP calls are centralised through a shared HTTP utility or wrapper service rather than using `HttpClient` directly in feature services. Feature services call the central wrapper and use `pipe()` to transform the response:

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
    private readonly http = inject(HttpWrapper);

    getUsers(): Observable<IUser[]> {
        return this.http.get<IUser[]>(APIEndPoints.users.getAll).pipe(
            map(response => response),
            catchError((err: HttpErrorResponse) => throwError(() => err))
        );
    }
}
```

### 7.2 Error handling in HTTP pipelines

Use `catchError` for error handling. The two-argument form of `map()` silently ignores errors in RxJS 6+ and must not be used.

```typescript
// ✓ Correct
return this.http.get(url).pipe(
    map((response: IUser[]) => response),
    catchError((err: HttpErrorResponse) => throwError(() => err))
);

// ✗ Wrong — second argument to map() is ignored by RxJS 6+; errors are swallowed
return this.http.get(url).pipe(
    map(response => response, error => error)
);
```

### 7.3 HTTP interceptors

**Functional interceptors (Angular 15+, preferred)** are plain functions registered via `withInterceptors()`:

```typescript
// auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
    const token = inject(AuthService).getToken();
    const authReq = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
    return next(authReq);
};

// Registration (bootstrapApplication)
provideHttpClient(withInterceptors([authInterceptor]))
```

**Class-based interceptors (Angular 12+, legacy)** implement `HttpInterceptor` and are registered in the `CoreModule` providers array:

```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
    constructor(private authService: AuthService) { }

    intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
        const authReq = req.clone({
            setHeaders: { Authorization: `Bearer ${this.authService.getToken()}` }
        });
        return next.handle(authReq);
    }
}
```

### 7.4 Subject/Observable state pattern

Services expose state as a public `Observable` derived from a private `Subject` or `BehaviorSubject`. Consumers subscribe to the public observable and never emit or complete the subject directly.

```typescript
@Injectable({ providedIn: 'root' })
export class AppStateService {

    // Private emitter — internal only
    private loadingSubject = new BehaviorSubject<boolean>(false);

    // Public read-only stream — for consumers
    public loadingState$ = this.loadingSubject.asObservable();

    setLoading(value: boolean): void {
        this.loadingSubject.next(value);
    }
}
```

Naming convention: `private xyzSubject` → `public xyz$` or `public xyzState$`.

### 7.5 Signal-based state in services (Angular 16+)

For UI state that components read reactively without subscribing, prefer signals over `Subject`/`Observable`:

```typescript
@Injectable({ providedIn: 'root' })
export class ThemeService {
    private readonly _theme = signal<string>('light');

    // Read-only signal for consumers — cannot be mutated from outside
    readonly theme = this._theme.asReadonly();

    setTheme(theme: string): void {
        this._theme.set(theme);
    }
}
```

### 7.6 EventEmitter vs Subject

`EventEmitter` is for `@Output()` properties on components only. Services must use `Subject` or `BehaviorSubject` for internal event streams.

```typescript
// ✓ In a service
private sessionEndedSubject = new Subject<void>();
public sessionEnded$ = this.sessionEndedSubject.asObservable();

// ✗ In a service — EventEmitter does not belong here
public onSessionEnded = new EventEmitter();
```

### 7.7 Generic HTTP wrapper service

The HTTP wrapper provides a single typed, centralised entry point for all HTTP calls. Feature services call this wrapper — they never inject `HttpClient` directly. The wrapper owns the base URL, common request options, and any cross-cutting concerns such as loader state.

```typescript
// http-wrapper.service.ts
@Injectable({ providedIn: 'root' })
export class HttpWrapperService {
    private readonly http = inject(HttpClient);
    private readonly baseUrl = environment.apiBaseUrl;

    /**
     * HTTP GET — fetches a resource and returns the response body typed as T.
     */
    get<T>(endpoint: string, params?: HttpParams): Observable<T> {
        return this.http.get<T>(`${this.baseUrl}${endpoint}`, { params });
    }

    /**
     * HTTP POST — sends a body and returns the response typed as T.
     */
    post<T>(endpoint: string, body: unknown): Observable<T> {
        return this.http.post<T>(`${this.baseUrl}${endpoint}`, body);
    }

    /**
     * HTTP PUT — replaces a resource and returns the response typed as T.
     */
    put<T>(endpoint: string, body: unknown): Observable<T> {
        return this.http.put<T>(`${this.baseUrl}${endpoint}`, body);
    }

    /**
     * HTTP PATCH — partially updates a resource and returns the response typed as T.
     */
    patch<T>(endpoint: string, body: unknown): Observable<T> {
        return this.http.patch<T>(`${this.baseUrl}${endpoint}`, body);
    }

    /**
     * HTTP DELETE — removes a resource and returns the response typed as T.
     */
    delete<T>(endpoint: string): Observable<T> {
        return this.http.delete<T>(`${this.baseUrl}${endpoint}`);
    }
}
```

Feature services consume the wrapper and add their own `pipe()` transforms. They never construct URLs manually — all endpoint strings come from the `APIEndPoints` constants object (see §18):

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
    private readonly http = inject(HttpWrapperService);

    getUsers(): Observable<IUser[]> {
        return this.http.get<IUser[]>(APIEndPoints.users.getAll);
    }

    getUserById(id: number): Observable<IUser> {
        return this.http.get<IUser>(APIEndPoints.users.getById(id));
    }

    createUser(user: IUser): Observable<IUser> {
        return this.http.post<IUser>(APIEndPoints.users.create, user);
    }

    updateUser(id: number, changes: Partial<IUser>): Observable<IUser> {
        return this.http.put<IUser>(APIEndPoints.users.update(id), changes);
    }

    deleteUser(id: number): Observable<void> {
        return this.http.delete<void>(APIEndPoints.users.delete(id));
    }
}
```

### 7.8 Global error handling

Two complementary layers handle errors globally. Both should be present — they catch different categories of failure.

**Layer 1 — HTTP interceptor** catches all HTTP error responses before they reach feature services. It handles known status codes centrally and re-throws so that feature-level `catchError` calls can still handle domain-specific cases (e.g., a 404 on a search endpoint is not an error — it means no results).

```typescript
// global-error.interceptor.ts
export const globalErrorInterceptor: HttpInterceptorFn = (req, next) => {
    const router = inject(Router);
    const toastService = inject(ToastService);

    return next(req).pipe(
        catchError((error: HttpErrorResponse) => {
            switch (error.status) {
                case 401:
                    // Session expired — redirect to login
                    router.navigate(['/login']);
                    break;
                case 403:
                    toastService.error('You do not have permission to perform this action.');
                    break;
                case 404:
                    // Do not intercept — let feature services decide how to handle missing resources
                    break;
                case 500:
                default:
                    toastService.error('An unexpected server error occurred. Please try again.');
                    break;
            }
            return throwError(() => error);   // always re-throw so callers can still react
        })
    );
};
```

Register the global interceptor after the auth interceptor so tokens are attached before error handling runs:

```typescript
// Standalone bootstrap
provideHttpClient(withInterceptors([authInterceptor, globalErrorInterceptor]))

// NgModule-based (CoreModule providers)
{ provide: HTTP_INTERCEPTORS, useClass: GlobalErrorInterceptor, multi: true }
```

**Layer 2 — Angular ErrorHandler** catches all unhandled runtime errors that escape the component and service layer — template errors, uncaught exceptions in lifecycle hooks, and any `throw` that is never caught. It logs to a monitoring service and shows a fallback message.

```typescript
// global-error-handler.ts
import { ErrorHandler, inject, Injectable } from '@angular/core';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
    private readonly logger = inject(LoggingService);
    private readonly toastService = inject(ToastService);

    handleError(error: unknown): void {
        if (error instanceof HttpErrorResponse) {
            // HTTP errors are already handled by the interceptor — avoid double-logging
            return;
        }
        console.error('Unhandled application error:', error);
        this.logger.logError(error);
        this.toastService.error('An unexpected application error occurred.');
    }
}
```

Register the `GlobalErrorHandler` at the application root so it catches errors from all modules:

```typescript
// Standalone bootstrap
bootstrapApplication(AppComponent, {
    providers: [
        { provide: ErrorHandler, useClass: GlobalErrorHandler },
        provideHttpClient(withInterceptors([authInterceptor, globalErrorInterceptor])),
    ]
});

// NgModule-based (AppModule providers)
@NgModule({
    providers: [{ provide: ErrorHandler, useClass: GlobalErrorHandler }]
})
export class AppModule { }
```

---

## 8. Signals

Angular 16+ introduces the Signal as a reactive primitive and an alternative to RxJS for UI state. Use signals for values that drive template rendering; use Observables for async data pipelines (HTTP, WebSockets, router events).

### 8.1 Core primitives (Angular 16+)

```typescript
import { signal, computed, effect } from '@angular/core';

// Writable signal
const count = signal(0);
count.set(1);              // replace the value
count.update(n => n + 1); // derive from the current value

// Computed — derived, read-only, lazy; only re-evaluates when dependencies change
const doubled = computed(() => count() * 2);

// Effect — runs whenever its signal dependencies change; use sparingly for side effects
effect(() => {
    console.log('count changed:', count());
});
```

### 8.2 RxJS interop (Angular 16+)

```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';

// Observable → Signal (unsubscribes automatically on destroy)
protected readonly users = toSignal(this.userService.getUsers(), { initialValue: [] as IUser[] });

// Signal → Observable (passes a signal's value into an RxJS pipeline)
private readonly searchObs$ = toObservable(this.searchTerm);

this.searchObs$.pipe(
    debounceTime(300),
    switchMap(term => this.userService.search(term)),
    takeUntilDestroyed(this.destroyRef)
).subscribe(results => this.searchResults.set(results));
```

### 8.3 When to use signals vs Observables

| Scenario | Recommended approach |
|---|---|
| Template-bound UI state (loading flag, selected item, toggle) | `signal()` |
| Derived or computed display value | `computed()` |
| HTTP result displayed directly in template | `toSignal(observable$)` |
| Async pipeline (debounce, retry, combine, switchMap chain) | Observable + RxJS |
| Cross-component event bus (fire-and-forget, no shared state) | `Subject` in a service |
| Simple shared UI state across components | `signal()` in a service |

### 8.4 effect() rules

`effect()` is for side effects that must react to signal changes (e.g., syncing to localStorage, logging). Avoid mutating other signals inside an `effect()`; if unavoidable, pass `{ allowSignalWrites: true }` explicitly. Prefer `computed()` for value derivation and restrict `effect()` to I/O operations.

---

## 9. RxJS Usage

### 9.1 Subscription teardown — modern pattern (Angular 16+, preferred)

Use `takeUntilDestroyed(this.destroyRef)` from `@angular/core/rxjs-interop`. It automatically unsubscribes when the component is destroyed, without requiring `ngOnDestroy`:

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DestroyRef, inject } from '@angular/core';

export class MyComponent {
    private readonly destroyRef = inject(DestroyRef);

    constructor() {
        this.dataService.records$.pipe(
            takeUntilDestroyed(this.destroyRef)
        ).subscribe(records => this.records = records);
    }
}
```

### 9.2 Subscription teardown — legacy pattern (Angular 12+)

Use `takeUntil(this.destroy$)` as the standard teardown mechanism. Every component that subscribes must implement `OnDestroy`:

```typescript
export class MyComponent implements OnInit, OnDestroy {
    private destroy$ = new Subject<void>();

    ngOnInit(): void {
        this.dataService.records$
            .pipe(takeUntil(this.destroy$))
            .subscribe(records => this.records = records);
    }

    ngOnDestroy(): void {
        this.destroy$.next();
        this.destroy$.complete();
    }
}
```

Avoid managing multiple named `Subscription` variables and calling `.unsubscribe()` individually — the pattern scales poorly and is easy to forget.

### 9.3 Operators — when to use each

| Operator | Use for |
|---|---|
| `map` | Transform an emitted value |
| `catchError` | Handle errors in a stream — the only correct place for HTTP error handling |
| `tap` | Side effects (logging, state update) without transforming the value |
| `distinctUntilChanged` | Suppress emissions when the value has not changed |
| `debounceTime` | Delay and coalesce rapid emissions (search input, resize events) |
| `take(1)` | One-shot subscriptions that complete after the first emission |
| `takeUntil` | Component teardown (legacy) — see §9.2 |
| `takeUntilDestroyed` | Component teardown (modern) — see §9.1 |
| `filter` | Conditionally suppress emissions |
| `switchMap` | Dependent HTTP calls — cancels the prior inner observable on new emission |
| `combineLatest` | Combine the latest values from multiple streams |
| `finalize` | Cleanup that must run regardless of completion or error (e.g., hide loader) |

### 9.4 Avoiding nested subscriptions

```typescript
// ✗ Nested subscribe — do not do this
this.service.getUser(id).subscribe(user => {
    this.service.getOrders(user.id).subscribe(orders => {
        this.orders = orders;
    });
});

// ✓ Use switchMap
this.service.getUser(id).pipe(
    switchMap(user => this.service.getOrders(user.id)),
    takeUntilDestroyed(this.destroyRef)  // modern
    // or: takeUntil(this.destroy$)      // legacy
).subscribe(orders => this.orders = orders);
```

### 9.5 Router event subscriptions

When subscribing to `Router.events`, use `filter()` to narrow to only the events that have logic. Remove empty branches:

```typescript
// ✓ Correct
this.router.events.pipe(
    filter(event => event instanceof NavigationEnd),
    takeUntilDestroyed(this.destroyRef)
).subscribe(event => this.onNavigationEnd(event as NavigationEnd));

// ✗ Avoid — empty branches add noise and signal incomplete cleanup
this.router.events.subscribe(event => {
    if (event instanceof NavigationStart) {
        // nothing
    } else if (event instanceof NavigationEnd) {
        this.onNavigationEnd(event);
    } else if (event instanceof NavigationError) {
        // nothing
    }
});
```

---

## 10. Template Style

### 10.1 Binding syntax

```html
<!-- Property binding -->
<app-grid [dataSource]="records"></app-grid>

<!-- Event binding -->
<button (click)="onSave()">Save</button>

<!-- Two-way binding -->
<input [(ngModel)]="searchText" />

<!-- Custom two-way binding (requires @Output() valueChange EventEmitter) -->
<app-filter-bar [(filterValue)]="activeFilter"></app-filter-bar>

<!-- ✗ Avoid interpolation in attribute values -->
<a href="{{url}}">Link</a>

<!-- ✓ Use property binding -->
<a [href]="url">Link</a>
```

### 10.2 Control flow

Angular 17+ introduces built-in control flow (`@if`, `@for`, `@switch`). **Use the new syntax by default.** Fall back to the legacy structural directives only when the project targets Angular 16 or below.

#### New syntax (Angular 17+ — preferred)

```html
<!-- Conditional rendering -->
@if (isVisible) {
    <div>Content</div>
} @else {
    <div>Fallback</div>
}

<!-- Iteration — track is required and replaces trackBy -->
@for (item of items; track item.id) {
    <div>{{ item.name }}</div>
} @empty {
    <div>No items found</div>
}

<!-- Multiple branches -->
@switch (activeSection) {
    @case ('overview') { <app-overview /> }
    @case ('settings') { <app-settings /> }
    @default { <div>Not found</div> }
}
```

The `track` expression in `@for` is mandatory — use a unique identifier (`item.id`) rather than `$index` wherever possible, so Angular can correctly reconcile DOM nodes when the list changes.

#### Legacy syntax (Angular 16 and below — use only if required by the project's Angular version)

```html
<!-- Conditional rendering -->
<div *ngIf="isVisible">Content</div>

<!-- Iteration — always include trackBy -->
<div *ngFor="let item of items; trackBy: trackByItem">{{ item.name }}</div>

<!-- Multiple branches -->
<div [ngSwitch]="activeSection">
    <app-overview *ngSwitchCase="'overview'"></app-overview>
    <app-settings *ngSwitchCase="'settings'"></app-settings>
    <div *ngSwitchDefault>Not found</div>
</div>
```

When using the legacy syntax, always provide a `trackBy` function on `*ngFor`:

```typescript
// In the component class
trackByItem(index: number, item: IItem): number | string {
    return item.id;
}
```

### 10.3 Deferred loading (Angular 17+)

Use `@defer` to lazily render non-critical content, reducing initial bundle size and improving LCP:

```html
@defer (on viewport) {
    <app-heavy-chart [data]="chartData" />
} @placeholder {
    <div class="chart-placeholder">Loading chart…</div>
} @loading (minimum 500ms) {
    <app-spinner />
} @error {
    <div>Failed to load chart.</div>
}
```

Common triggers: `on viewport` (element enters the viewport), `on idle` (browser idle), `on interaction` (user clicks/focuses), `on timer(2s)` (fixed delay).

### 10.4 Optimised images (Angular 15+)

Use the `NgOptimizedImage` directive (`ngSrc`) instead of the native `src` attribute to get automatic lazy loading, dimension enforcement, and LCP preload hints:

```html
<!-- Add NgOptimizedImage to the component's imports array -->
<img ngSrc="assets/logo.png" width="200" height="60" alt="Logo" priority />
<img ngSrc="assets/avatar.jpg" width="48" height="48" alt="User avatar" />
```

`priority` marks above-the-fold images for preloading. `width` and `height` are required and prevent cumulative layout shift.

### 10.5 ng-container

Use `<ng-container>` to apply structural directives without introducing a DOM element:

```html
<!-- Legacy structural directive — no extra DOM wrapper -->
<ng-container *ngIf="isReady">
    <app-grid [dataSource]="records"></app-grid>
</ng-container>

<!-- Modern control flow makes ng-container less necessary, but still valid -->
@if (isReady) {
    <app-grid [dataSource]="records" />
}
```

---

## 11. Type System

### 11.1 Interface naming

All interfaces use the `I` prefix + PascalCase:

```typescript
interface IGridColumn { dataField: string; caption: string; }
interface IValidationRule { type: string; message: string; }
interface IUserProfile { id: number; name: string; email: string; }
```

### 11.2 Server contract types

Types that represent server API response shapes (DTOs) should be declared in a dedicated namespace or a shared `types/` directory and clearly separated from UI-layer component contracts:

```typescript
// Server DTO types — in a shared namespace or types file
namespace API {
    interface IUserResponse { UserId: number; UserName: string; }
}

// Component contract — local to the feature
interface IUserRow { id: number; displayName: string; }
```

Library components define their own local interfaces in a `contracts/` directory and do not depend on application-level DTO types.

### 11.3 Minimise `any`

Prefer specific types. Use `unknown` where the type is genuinely unknown and must be narrowed before use. Reserve `any` for third-party library event arguments where the vendor's type definitions are impractical.

```typescript
// ✓ Typed
@Input() public options: GridOptionsModel = null;

// Acceptable where vendor types are unavailable
onEditorPreparing?: (args: any) => void;

// ✗ Avoid gratuitous any
public processData(data: any): any { ... }
```

### 11.4 Return type annotations

Declare explicit return types on all public service methods and all public library API members. Private methods may omit them when the type is obvious from the return expression.

```typescript
// ✓ Public service method — explicit return type required
public getUser(id: number): Observable<IUserProfile> { ... }

// ✓ Public event handler — return type helps the compiler
public onSelectionChanged(event: SelectionChangedEvent): void { ... }

// ✓ Private method — may omit when the type is obvious
private buildColumns() { return this.schema.map(s => new GridColumn(s)); }
```

---

## 12. Comments & Documentation

### 12.1 JSDoc — standard for public methods

All public service methods and all public library API members must have JSDoc comments. Component event handlers and private methods are encouraged but not mandatory.

```typescript
/**
 * Fetches the paginated list of users for a given role.
 * @param {string} role The role identifier to filter by.
 * @param {number} page Zero-based page index.
 * @returns Observable that emits the page of user records.
 */
public getUsersByRole(role: string, page: number): Observable<IPage<IUserProfile>> { ... }
```

### 12.2 Region comments

Large files may use `#region` / `#endregion` comments to group related members. Useful in files with many `@Input()` / `@Output()` declarations or in utility modules.

```typescript
//#region Inputs
@Input() public dataSource: IRecord[];
@Input() public pageSize: number;
//#endregion Inputs

//#region Lifecycle
ngOnInit(): void { ... }
ngOnDestroy(): void { ... }
//#endregion Lifecycle
```

### 12.3 Inline comments

- One-liner above the relevant code: `// Double-encode to escape proxy URL decoding`
- Ticket references are acceptable for non-obvious workarounds: `// PROJ-1234: suppress vendor error E1047`
- Do not commit commented-out code — delete it. Git history preserves it.

### 12.4 TODO / FIXME markers

Mark unresolved technical debt explicitly rather than leaving silent stubs:

```typescript
// TODO: replace with reactive form validation — ticket PROJ-456
// FIXME: this fires on every CD cycle — switch to ngOnChanges
```

---

## 13. Lifecycle Hook Conventions

| Hook | Canonical use |
|---|---|
| `ngOnChanges` | React when one or more `@Input()` values change, especially when multiple inputs interact |
| `ngOnInit` | HTTP calls, subscription setup, one-time initialisation after inputs are set |
| `ngAfterViewInit` | `@ViewChild` / `@ViewChildren` access; third-party widget instance registration (legacy — prefer `afterNextRender()` in Angular 17+) |
| `ngAfterViewChecked` | Post-render checks — use sparingly; runs on every change detection cycle |
| `ngOnDestroy` | Teardown: `destroy$.next()` / `destroy$.complete()`; clear session/local storage entries owned by this component |
| `ngDoCheck` | **Avoid** — runs on every change detection cycle regardless of what changed. Use `ngOnChanges` or reactive forms instead |

**Angular 17+ render hooks** (`afterNextRender`, `afterRender`) are standalone functions, not lifecycle interface methods — call them in the constructor or field initialisers:

```typescript
import { afterNextRender, afterRender } from '@angular/core';

export class ChartComponent {
    constructor() {
        // Runs once after the next render — preferred for third-party DOM setup
        afterNextRender(() => {
            this.initChart();
        });

        // Runs after every render cycle — use sparingly (e.g., canvas/chart redraws)
        afterRender(() => {
            this.refreshCanvasSize();
        });
    }
}
```

Only declare lifecycle hook interfaces (`implements OnInit`, `implements OnDestroy`, etc.) for hooks that are actually used. Do not call lifecycle hooks manually — Angular invokes them; calling them yourself produces double-initialisation and interacts badly with change detection.

---

## 14. NgRx State Management

### 14.1 Store structure

Each feature slice has its own directory with `actions/`, `reducers/`, and `selectors/` files. The root store's `AppState` interface references each slice by feature key:

```typescript
export interface AppState {
    auth: fromAuth.AuthState;
    userProfile: fromUserProfile.UserProfileState;
}
```

### 14.2 Modern action style (NgRx 8+, preferred)

Use `createAction` with `props<>()` for typed payloads:

```typescript
import { createAction, props } from '@ngrx/store';

export const loadUser = createAction(
    '[UserProfile] Load User',
    props<{ id: number }>()
);
export const loadUserSuccess = createAction(
    '[UserProfile] Load User Success',
    props<{ user: IUserProfile }>()
);
export const loadUserFailure = createAction(
    '[UserProfile] Load User Failure',
    props<{ error: string }>()
);
```

### 14.3 Modern reducer style (NgRx 8+, preferred)

Use `createReducer` with `on()` handlers:

```typescript
import { createReducer, on } from '@ngrx/store';

export const userProfileReducer = createReducer(
    initialState,
    on(loadUserSuccess, (state, { user }) => ({ ...state, user, isLoaded: true })),
    on(loadUserFailure, (state, { error }) => ({ ...state, error, isLoaded: false })),
);
```

### 14.4 Modern selector style (NgRx 8+, preferred)

Use `createSelector` for memoised selectors:

```typescript
import { createSelector, createFeatureSelector } from '@ngrx/store';

const selectUserProfileState = createFeatureSelector<UserProfileState>('userProfile');
export const selectUser = createSelector(selectUserProfileState, state => state.user);
export const selectIsLoaded = createSelector(selectUserProfileState, state => state.isLoaded);
```

### 14.5 Modern effect style (NgRx 8+, preferred)

Use `createEffect` inside an injectable effects class:

```typescript
import { createEffect, Actions, ofType } from '@ngrx/effects';

@Injectable()
export class UserProfileEffects {
    private readonly actions$ = inject(Actions);
    private readonly userService = inject(UserService);

    loadUser$ = createEffect(() =>
        this.actions$.pipe(
            ofType(loadUser),
            switchMap(({ id }) => this.userService.getUser(id).pipe(
                map(user => loadUserSuccess({ user })),
                catchError(error => of(loadUserFailure({ error: error.message })))
            ))
        )
    );
}
```

### 14.6 Legacy class-based action style (pre-NgRx 8)

Class-based actions remain in place in projects that have not migrated. Do not mix styles within the same feature slice:

```typescript
export const Names = {
    LoadUser: '[UserProfile] Load User',
    LoadUserSuccess: '[UserProfile] Load User Success',
    LoadUserFailure: '[UserProfile] Load User Failure',
};

export class LoadUser implements Action {
    readonly type = Names.LoadUser;
    constructor(public payload: number) { }
}

export class LoadUserSuccess implements Action {
    readonly type = Names.LoadUserSuccess;
    constructor(public payload: IUserProfile) { }
}

export type UserProfileActions = LoadUser | LoadUserSuccess | LoadUserFailure;
```

### 14.7 Legacy reducer style (pre-NgRx 8)

```typescript
export function userProfileReducer(
    state: UserProfileState = initialState,
    action: UserProfileActions
): UserProfileState {
    switch (action.type) {
        case Names.LoadUserSuccess: {
            return { ...state, user: action.payload, isLoaded: true };
        }
        default:
            return state;
    }
}
```

**Always wrap `case` bodies in braces** when local variables are declared inside them — TypeScript's `const`/`let` in an unwrapped `case` block shares the switch scope and causes lint errors.

---

## 15. Guards and Route Resolvers

### 15.1 Functional guards (Angular 15+, preferred)

Functional guards are plain functions of type `CanActivateFn`, `CanMatchFn`, or `CanDeactivateFn`. They use `inject()` to access services:

```typescript
// auth.guard.ts
import { CanActivateFn } from '@angular/router';
import { inject } from '@angular/core';

export const authGuard: CanActivateFn = (route, state) => {
    const authService = inject(AuthService);
    const router = inject(Router);
    return authService.isLoggedIn()
        ? true
        : router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};
```

### 15.2 Functional resolvers (Angular 15+, preferred)

```typescript
// user.resolver.ts
import { ResolveFn } from '@angular/router';
import { inject } from '@angular/core';

export const userResolver: ResolveFn<IUser> = route => {
    return inject(UserService).getUser(+route.paramMap.get('id'));
};
```

### 15.3 Class-based guards (Angular 12+, legacy)

```typescript
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
    constructor(private authService: AuthService, private router: Router) { }

    canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): boolean {
        if (this.authService.isLoggedIn()) { return true; }
        this.router.navigate(['/login'], { queryParams: { returnUrl: state.url } });
        return false;
    }
}
```

---

## 16. Third-Party Widget Wrapper Components

When wrapping a third-party UI widget (e.g., a grid, date-picker, or select-box), the wrapper component pattern applies:

- Use `ViewEncapsulation.None` on the wrapper so the parent can style the widget from outside.
- Keep the component's selector within the project's chosen prefix scheme (`app-dx-*`, `app-ag-*`, etc.).
- Use `#region` comments to organise `@Input()` / `@Output()` declarations in large wrapper files.
- Where the vendor uses `(onXxx)` naming for events (non-Angular convention), disable the relevant lint rule and add a comment explaining why:

```typescript
// eslint-disable-next-line @angular-eslint/no-output-on-prefix
@Output() onCellClick = new EventEmitter<CellClickEvent>();
// eslint-disable-next-line @angular-eslint/no-output-on-prefix
@Output() onSelectionChanged = new EventEmitter<SelectionChangedEvent>();
```

For legacy projects still using TSLint:

```typescript
// tslint:disable:no-output-on-prefix
@Output() onCellClick = new EventEmitter<CellClickEvent>();
@Output() onSelectionChanged = new EventEmitter<SelectionChangedEvent>();
// tslint:enable:no-output-on-prefix
```

---

## 17. Angular-Specific Best Practices

### 17.1 Change detection strategy

Use `ChangeDetectionStrategy.OnPush` as the default for new components. OnPush components only re-render when an input reference changes, a signal changes, or an event fires inside the component — drastically reducing unnecessary rendering cycles.

```typescript
@Component({
    selector: 'app-user-card',
    standalone: true,
    changeDetection: ChangeDetectionStrategy.OnPush,
    ...
})
export class UserCardComponent { ... }
```

For components that cannot easily move to OnPush (e.g., those that mutate objects in-place or depend on external DOM events), document the reason explicitly rather than silently leaving the strategy on `Default`.

### 17.2 Avoiding business logic in AppModule / bootstrapApplication

Route registration, menu building, and application-level data preparation do not belong in `AppModule` or the bootstrap call. These should live in dedicated services wired up via `APP_INITIALIZER`:

```typescript
// ✓ Extract to a service and use APP_INITIALIZER
export function initApp(initService: AppInitService) {
    return () => initService.initialize();
}

@NgModule({
    providers: [{
        provide: APP_INITIALIZER,
        useFactory: initApp,
        deps: [AppInitService],
        multi: true
    }]
})
export class AppModule { }
```

### 17.3 Documenting scoped services

When a service is in a component's `providers` array (creating a new instance per component), add a comment to make the intent clear — it is otherwise easy to assume the service is a singleton:

```typescript
@Component({
    providers: [GridStateService],  // Scoped: new instance per GridComponent subtree
})
```

### 17.4 Named exceptions to conventions

If a section of code intentionally breaks a convention (e.g., PascalCase method names that mirror a vendor library's API), document this with a comment so collaborators do not "correct" it:

```typescript
// Method names are PascalCase to mirror the oidc-client library's API.
// Do not rename — it would break 1-to-1 parity with vendor documentation.
public Login(): void { ... }
public Logout(): void { ... }
```

### 17.5 Linting

New projects use `@angular-eslint`. TSLint is deprecated and should be migrated:

```bash
ng add @angular-eslint/schematics
ng g @angular-eslint/schematics:convert-tslint-to-eslint
```

Disable lint rules only when unavoidable. Always comment why a rule is suppressed:

```typescript
// eslint-disable-next-line @angular-eslint/no-output-on-prefix
@Output() onCellClick = new EventEmitter<CellClickEvent>();
```

---

## 18. Constants and API Endpoints

API endpoint strings are centralised in a single constants file under a nested object, not scattered across service files:

```typescript
// ✓ Centralised
export const APIEndPoints = {
    users: {
        getAll: `${baseUrl}/api/users`,
        getById: (id: number) => `${baseUrl}/api/users/${id}`,
    },
    orders: {
        getByUser: (userId: number) => `${baseUrl}/api/orders?userId=${userId}`,
    },
};

// ✗ Avoid — URL strings scattered inside services
@Injectable()
export class UserService {
    private readonly url = 'https://my-api.example.com/api/users'; // not here
}
```

---

## 19. Styling (SCSS)

### 19.1 SCSS file organisation

Global styles live in a `src/styles/` directory organised as SCSS partials (underscore-prefixed files — they are never compiled independently):

```
src/
  styles/
    _variables.scss     // design tokens — colours, spacing, typography, z-index
    _mixins.scss        // reusable mixins and functions
    _breakpoints.scss   // responsive breakpoint map and mixin
    _typography.scss    // font-face declarations, heading scale
    _reset.scss         // normalise / box-sizing reset
  styles.scss           // root entry — @forward all partials, global element styles
```

Component-level `.component.scss` files import only what they need from the shared partials:

```scss
// user-profile.component.scss
@use '../../../styles/variables' as vars;
@use '../../../styles/mixins' as mix;
```

Use `@use` and `@forward` throughout — `@import` is deprecated in Dart Sass and will be removed in a future release. Never use `@import` in new code.

### 19.2 Variables and design tokens

Define all design tokens in `_variables.scss` as SCSS variables and mirror them as CSS custom properties on `:root` for runtime access (theming, dark mode):

```scss
// _variables.scss

// Spacing scale
$spacing-xs:  4px;
$spacing-sm:  8px;
$spacing-md:  16px;
$spacing-lg:  24px;
$spacing-xl:  40px;

// Colour palette
$color-primary:       #0066cc;
$color-primary-dark:  #004999;
$color-surface:       #ffffff;
$color-on-surface:    #1a1a1a;
$color-error:         #d32f2f;
$color-success:       #388e3c;
$color-warning:       #f57c00;

// Typography
$font-family-base:    'Inter', system-ui, sans-serif;
$font-size-base:      14px;
$font-size-sm:        12px;
$font-size-lg:        18px;
$font-weight-regular: 400;
$font-weight-bold:    600;

// Z-index scale — all z-index values live here; never use arbitrary numbers elsewhere
$z-base:      1;
$z-dropdown:  100;
$z-sticky:    200;
$z-overlay:   300;
$z-modal:     400;
$z-toast:     500;

// Expose tokens as CSS custom properties for runtime theming
:root {
    --color-primary:    #{$color-primary};
    --color-surface:    #{$color-surface};
    --color-on-surface: #{$color-on-surface};
    --color-error:      #{$color-error};
    --color-success:    #{$color-success};
    --font-family-base: #{$font-family-base};
    --font-size-base:   #{$font-size-base};
    --spacing-md:       #{$spacing-md};
}
```

**Rule:** Use CSS custom properties (`var(--color-primary)`) in component styles when the value must be overridable at runtime (theme switching, user preferences). Use SCSS variables (`$color-primary`) when the value is fixed at build time and never changes.

### 19.3 Breakpoints and responsive design

Define breakpoints in a Sass map and expose them through a mixin. Write styles **mobile-first** — base rules cover the smallest viewport; `respond-to()` adds overrides for larger sizes:

```scss
// _breakpoints.scss
@use 'sass:map';

$breakpoints: (
    'xs':  320px,
    'sm':  576px,
    'md':  768px,
    'lg':  992px,
    'xl':  1200px,
    'xxl': 1400px,
);

/// Applies styles at the given breakpoint and above (mobile-first).
/// @param {String} $breakpoint - Key from the $breakpoints map.
@mixin respond-to($breakpoint) {
    $value: map.get($breakpoints, $breakpoint);
    @if $value == null {
        @error 'Unknown breakpoint `#{$breakpoint}`. Available: #{map.keys($breakpoints)}.';
    }
    @media screen and (min-width: $value) {
        @content;
    }
}
```

Usage in a component:

```scss
@use '../../../styles/breakpoints' as bp;

.user-list {
    display: grid;
    grid-template-columns: 1fr;          // single column on mobile

    @include bp.respond-to('md') {
        grid-template-columns: 1fr 1fr;  // two columns on tablet+
    }

    @include bp.respond-to('xl') {
        grid-template-columns: repeat(3, 1fr);  // three on desktop
    }
}
```

Use `clamp()` for fluid values that scale continuously between viewport sizes without breakpoints:

```scss
.page-title {
    // Fluid: 1.25rem at narrow, scales to 2rem at wide, relative to viewport width
    font-size: clamp(1.25rem, 2.5vw + 0.5rem, 2rem);
}
```

### 19.4 Mixins and functions

Centralise reusable style patterns in `_mixins.scss`. Use `@use 'sass:math'` for arithmetic rather than the deprecated `/` division operator:

```scss
// _mixins.scss
@use 'sass:math';
@use 'variables' as vars;

/// Truncate overflowing text to a single line with an ellipsis.
@mixin text-ellipsis {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

/// Visually hidden but accessible to screen readers and keyboard navigation.
@mixin visually-hidden {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
}

/// Flex row centred on both axes.
@mixin flex-center {
    display: flex;
    align-items: center;
    justify-content: center;
}

/// Absolute fill — stretch an element to cover its nearest positioned ancestor.
@mixin absolute-fill {
    position: absolute;
    inset: 0;   // shorthand for top/right/bottom/left: 0
}

/// Convert px to rem using the base font size.
@function rem($px) {
    @return math.div($px, 16px) * 1rem;
}
```

Usage in a component:

```scss
@use '../../../styles/mixins' as mix;
@use '../../../styles/variables' as vars;

.filter-label {
    @include mix.visually-hidden;
}

.card-title {
    @include mix.text-ellipsis;
    font-size: mix.rem(18px);
    font-weight: vars.$font-weight-bold;
}

.loading-overlay {
    @include mix.absolute-fill;
    @include mix.flex-center;
    background: rgba(255, 255, 255, 0.7);
    z-index: vars.$z-overlay;
}
```

### 19.5 BEM naming convention

Use **BEM (Block–Element–Modifier)** for all CSS class names. All class names are **kebab-case**. In Angular, the component is the Block — the BEM block name should match the component's selector name.

```
.block { }
.block__element { }
.block--modifier { }
.block__element--modifier { }
```

```scss
// user-card.component.scss
@use '../../../styles/variables' as vars;
@use '../../../styles/mixins' as mix;

.user-card {                                    // Block
    background: var(--color-surface);
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    padding: vars.$spacing-md;

    &__avatar {                                 // Element
        width: 48px;
        height: 48px;
        border-radius: 50%;
        object-fit: cover;
    }

    &__name {                                   // Element
        font-weight: vars.$font-weight-bold;
        @include mix.text-ellipsis;
    }

    &__badge {                                  // Element
        display: inline-flex;
        align-items: center;
        padding: vars.$spacing-xs vars.$spacing-sm;
        border-radius: 4px;
        font-size: mix.rem(12px);

        &--active   { color: vars.$color-success; background: #e8f5e9; }  // Modifier
        &--inactive { color: vars.$color-error;   background: #ffebee; }  // Modifier
        &--pending  { color: vars.$color-warning; background: #fff3e0; }  // Modifier
    }

    &--selected {                               // Block modifier
        outline: 2px solid var(--color-primary);
        outline-offset: 2px;
    }

    &--loading {                                // Block modifier
        opacity: 0.6;
        pointer-events: none;
    }
}
```

```html
<!-- Template reflects the BEM structure exactly -->
<div class="user-card"
     [class.user-card--selected]="isSelected"
     [class.user-card--loading]="isLoading">
    <img class="user-card__avatar" [src]="user.avatarUrl" [alt]="user.name" />
    <span class="user-card__name">{{ user.name }}</span>
    <span class="user-card__badge"
          [class.user-card__badge--active]="user.status === 'active'"
          [class.user-card__badge--inactive]="user.status === 'inactive'"
          [class.user-card__badge--pending]="user.status === 'pending'">
        {{ user.status }}
    </span>
</div>
```

### 19.6 Dynamic styling — ngClass over inline styles

**Prefer CSS classes over inline styles.** Define all visual states as named CSS classes; use Angular bindings only to toggle which class is active.

**Single conditional class — `[class.modifier]` binding:**

```html
<!-- ✓ Preferred -->
<div class="panel" [class.panel--collapsed]="isCollapsed">...</div>

<!-- ✗ Avoid — hard to override, no reuse, no IDE support -->
<div [style.display]="isCollapsed ? 'none' : 'block'">...</div>
```

**Multiple conditional classes — `[ngClass]` directive:**

```html
<!-- ✓ Object syntax — clear, readable -->
<button [ngClass]="{
    'action-btn':          true,
    'action-btn--primary': isPrimary,
    'action-btn--danger':  isDanger,
    'action-btn--disabled': isDisabled
}">{{ label }}</button>
```

Move complex class logic into a `computed()` or a getter in the component — keep templates declarative:

```typescript
// ✓ In the component
protected readonly cardClasses = computed(() => ({
    'user-card': true,
    'user-card--selected': this.isSelected(),
    'user-card--loading':  this.isLoading(),
}));
```

```html
<div [ngClass]="cardClasses()">...</div>
```

**Inline `[style]` binding is acceptable only when the value cannot be expressed as a precomputed CSS class** — for example, a user-defined hex colour, a pixel position derived from a DOM measurement, or a runtime percentage:

```html
<!-- ✓ Acceptable — value is not knowable at build time -->
<div [style.background-color]="user.brandColour">...</div>
<div [style.top.px]="tooltipTop">...</div>
<div [style.width.%]="progressPercent">...</div>
```

### 19.7 View encapsulation

| Strategy | When to use |
|---|---|
| `Emulated` (default) | All components — Angular adds a unique scoping attribute automatically; no declaration needed |
| `None` | Third-party widget wrappers only — allows parent and global styles to reach widget internals. Always add a comment explaining why |
| `ShadowDom` | Avoid — native Shadow DOM prevents global theming and is not supported in all target environments |

```typescript
// ✓ Default — nothing to declare; Emulated is applied automatically
@Component({ selector: 'app-user-card', ... })
export class UserCardComponent { }

// ✓ Justified use of None for a third-party wrapper
@Component({
    selector: 'app-dx-grid',
    encapsulation: ViewEncapsulation.None,  // DevExtreme styles must pierce encapsulation
    ...
})
export class DxGridWrapperComponent { }
```

### 19.8 :host selector

Use `:host` to style the component's own root element. Every component that participates in layout should declare its `display` value via `:host` — Angular components are `inline` by default, which is almost never what you want:

```scss
:host {
    display: block;     // or flex, grid — set explicitly; never rely on the default
    width: 100%;
}
```

Apply conditional styles to the host using a class or attribute selector:

```scss
:host(.compact) {
    padding: vars.$spacing-xs;
    font-size: mix.rem(12px);
}

:host([disabled]) {
    opacity: 0.5;
    pointer-events: none;
}
```

Use `:host-context()` to adapt a component's appearance based on an ancestor's state (e.g., a theme class on `<body>`):

```scss
// Default light styles on the block
.status-indicator {
    background: vars.$color-surface;
    color: vars.$color-on-surface;
}

// Override when a dark-theme ancestor is present
:host-context(.dark-theme) .status-indicator {
    background: #1e1e1e;
    color: #f5f5f5;
}
```

### 19.9 Avoiding ::ng-deep

**Do not use `::ng-deep`.** Angular has deprecated it — it leaks styles globally (the scoping attribute is not applied) and will eventually be removed.

Use these alternatives instead:

**Style a child component's internals → move the styles into the child component's own SCSS file.** If the style logically belongs to the child, that is the right place for it.

**Style a third-party widget → use `ViewEncapsulation.None` on the wrapper and scope manually with the BEM block class:**

```scss
// ✗ Wrong — leaks globally
::ng-deep .dx-datagrid-header-panel { background: vars.$color-primary; }

// ✓ Correct — wrapper has ViewEncapsulation.None; BEM block class scopes it
// dx-grid-wrapper.component.scss (ViewEncapsulation.None)
.app-dx-grid {
    .dx-datagrid-header-panel { background: var(--color-primary); }
    .dx-datagrid-rowalt       { background: #f9f9f9; }
}
```

**Apply a theme class from the parent → use `:host-context()` in the child's SCSS** (see §19.8).

### 19.10 !important

Never use `!important` as a shortcut for winning a specificity battle. Treat it as a last resort. When you reach for it, fix the root cause instead:

- **Flatten the selector** — overly nested selectors lose to shorter but more specific ones.
- **Move the rule closer to the element** — component-scoped styles take priority over global stylesheet rules.
- **Add a BEM modifier class** — `.button--highlight` is a clean override with zero specificity conflict.

The only acceptable uses:

```scss
// ✓ Utility class — !important is intentional; the class must always win regardless of context
@mixin visually-hidden {
    position: absolute !important;
    width: 1px !important;
    height: 1px !important;
    ...
}

// ✓ Unavoidable third-party override — document it with a comment and a ticket reference
// TODO: PROJ-1042 — remove when vendor upgrades widget to expose a public theming API
.dx-overlay-wrapper { z-index: vars.$z-modal !important; }
```

Whenever `!important` appears, a comment explaining why is mandatory.

---

## 20. Summary of Key Rules

### Core conventions (all Angular versions)

| Rule | Convention |
|---|---|
| Interface names | `I` prefix + PascalCase: `IUserProfile` |
| Enum names | PascalCase + `Enum` suffix: `GridEventEnum` |
| Enum members | camelCase: `GridEventEnum.selectionChanged` |
| Class properties | camelCase; use TypeScript `private`/`public`, no underscore prefix |
| Access on `@Output()` | No `public` keyword: `@Output() rowClick = new EventEmitter()` |
| HTTP error handling | `catchError()` — never the second argument of `map()` |
| EventEmitter | `@Output()` on components only — use `Subject` in services |
| `ngDoCheck` | Avoid — use `ngOnChanges` or reactive forms |
| `this.ngOnInit()` | Never call manually — use a private method instead |
| Lifecycle hooks | Only declare hooks that are actually used |
| JSDoc | Required on all public service methods and public library API members |
| Commented-out code | Delete it — git history is the archive |
| Library public API | Everything exported from `public-api.ts`; nothing else is supported |
| Service in providers[] | Add a comment explaining the scoped-instance intent |
| CSS class names | kebab-case BEM: `.block__element--modifier` |
| Dynamic classes | `[class.x]` or `[ngClass]` — avoid inline `[style]` unless value is runtime-dynamic |
| `!important` | Last resort only — always add a comment explaining why |
| `::ng-deep` | Never use — deprecated by Angular; use `ViewEncapsulation.None` or move styles to the child |
| SCSS imports | `@use` / `@forward` only — `@import` is deprecated in Dart Sass |
| Z-index values | Only from `$z-*` variables in `_variables.scss` — never use arbitrary numbers |

### Modern vs legacy comparison

| Feature | Modern (min. version) | Legacy equivalent |
|---|---|---|
| Dependency injection | `inject(Service)` in field initialiser (14+) | Constructor parameter injection |
| Component inputs | `input()` / `input.required()` (17.1+) | `@Input()` decorator |
| Component outputs | `output()` (17.3+) | `@Output() ... = new EventEmitter()` |
| ViewChild / ContentChild | `viewChild()` / `contentChild()` (17.3+) | `@ViewChild()` / `@ContentChild()` |
| App bootstrap | `bootstrapApplication()` (14+) | `platformBrowserDynamic().bootstrapModule()` |
| Routing | `*.routes.ts` with `loadComponent` (14+) | `*.routing.module.ts` with `loadChildren` pointing to a module |
| Guards / Resolvers | Functional `CanActivateFn` / `ResolveFn` (15+) | Class-based `CanActivate` / `Resolve` |
| HTTP interceptors | Functional `HttpInterceptorFn` + `withInterceptors()` (15+) | Class-based `HttpInterceptor` + `HTTP_INTERCEPTORS` token |
| Control flow | `@if` / `@for` / `@switch` (17+) | `*ngIf` / `*ngFor` / `[ngSwitch]` |
| Deferred loading | `@defer` blocks (17+) | Manual lazy loading / dynamic imports |
| Subscription teardown | `takeUntilDestroyed(destroyRef)` (16+) | `takeUntil(destroy$)` + `ngOnDestroy` |
| Reactive primitive | `signal()` / `computed()` (16+) | `BehaviorSubject` / `Observable` |
| Observable ↔ Signal | `toSignal()` / `toObservable()` (16+) | N/A |
| Post-render DOM access | `afterNextRender()` (17+) | `ngAfterViewInit()` |
| Optimised images | `NgOptimizedImage` / `ngSrc` (15+) | Native `<img src>` |
| NgRx actions | `createAction` / `props<>()` (NgRx 8+) | Class-based `Action` implementations |
| NgRx reducers | `createReducer` / `on()` (NgRx 8+) | `switch` statement reducer function |
| Change detection | `ChangeDetectionStrategy.OnPush` by default | `ChangeDetectionStrategy.Default` |
| Linting | `@angular-eslint` | TSLint (deprecated) |
