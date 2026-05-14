---
name: "Boundary Audit"
description: "Audit and remediate architectural boundary violations in a project following the AI Readiness Framework. Use when preparing a codebase for agent work, when business logic is leaking into HTTP handlers, when database queries appear in the wrong layer, when services are reaching into each other's internals, or when agents are producing code that ignores layer boundaries."
---

# Boundary Audit

## What This Skill Does

Finds and fixes two categories of boundary violation that cause agents to replicate bad structure:

1. **Layer boundary violations** — business logic in controllers, database queries outside repositories, HTTP concerns in services
2. **Service boundary violations** — direct database sharing between services, cross-service code calls, undeclared dependencies between packages

Agents learn what correct looks like by reading the code. A controller that contains business logic teaches agents to write controllers with business logic. Fixing violations before running agents is the only reliable way to prevent them from propagating.

---

## Quick Start

1. Determine the established layer structure from `AGENTS.md` or `RULES.md`
2. Sample the codebase to identify the actual layers present
3. Find violations using the patterns below
4. Remediate from the outermost layer inward

---

## Phase 1: Establish the Layer Map

Read `ARCHITECTURE.md` (specifically its **Architectural Pattern** section — framework rule 2.1g), `AGENTS.md`, `RULES.md`, and the directory structure.

**If `ARCHITECTURE.md` does not name a pattern, stop and escalate.** The audit cannot run cleanly without a declared pattern: without one, there is no canonical dependency-direction rule to validate against and "violation" becomes a matter of opinion. Send the user to the `architecture-md` skill first.

**Pattern → layer mapping.** Once the pattern is named, the layers and dependency direction are determined:

| Pattern | Layers | Dependency direction |
|---------|--------|---------------------|
| Hexagonal / ports-and-adapters | Domain core ← Adapters (primary + secondary) | Adapters depend on domain; domain depends on nothing |
| Clean architecture | Entities ← Use cases ← Interface adapters ← Frameworks/drivers | Outer depends on inner; inner depends on nothing outer |
| Layered (n-tier) | Controller → Service → Repository → Model | Top-to-bottom only; lower layer never imports higher |
| Vertical slice | Feature module (own thin layers per feature) | Within slice: top-to-bottom; across slices: shared kernel only |
| MVC | Controller → Model; View ← Controller | Controller orchestrates; View reads model |
| CQRS | Command Handler / Query Handler → Domain → Repository | Handlers depend on domain; domain depends on nothing |
| Action-based (Laravel variant) | Controller → Action → Repository → Model | Top-to-bottom only |

**Where does each layer live?**

Map directories to responsibilities. Example:
```
app/Http/Controllers/   HTTP handlers — layer boundary: HTTP concerns only
app/Actions/            Business logic — layer boundary: no HTTP, no direct DB
app/Repositories/       Data access — layer boundary: all queries here, no business logic
app/Models/             Data structures and relationships — no business logic
```

**What are the cross-service boundaries?**

If this is a monorepo or a system with multiple services:
- Which services exist?
- What is the declared communication mechanism (HTTP API, queue, event bus)?
- Which databases does each service own?

Document this map before proceeding. You will use it to classify every violation found.

---

## Phase 2: Find Layer Boundary Violations

### Controllers / HTTP Handlers

**What should be here:** Request validation, authentication checks, calling one service/action, forming the response.

**What should not be here:** Database queries, business logic, domain calculations, conditional branching based on domain state.

**How to find violations:**

Look for in controller files:
- Direct use of ORM/query builder (`->where(`, `->find(`, `SELECT`, `->query()`, `Model::`)
- Domain logic (price calculation, status transitions, eligibility checks, date arithmetic)
- Multiple service/action calls with conditional logic between them
- Building complex domain objects
- Sending emails, dispatching jobs, or triggering external API calls directly

**Examples of violations:**

```php
// PHP controller — business logic in handler
public function store(Request $request): JsonResponse
{
    $user = User::where('email', $request->email)->first();  // DB query in controller
    if (!$user) {
        $user = User::create([...]);  // DB write in controller
    }
    if ($user->subscription_status === 'trial' && $user->trial_ends_at < now()) {
        $user->subscription_status = 'expired';  // domain logic in controller
        $user->save();
    }
    Mail::to($user)->send(new WelcomeEmail());  // side effect in controller
    return response()->json($user);
}
```

```typescript
// Express route handler — business logic in handler
router.post('/orders', async (req, res) => {
    const items = await db.query('SELECT * FROM cart_items WHERE user_id = $1', [req.user.id]);
    let total = 0;
    for (const item of items) {
        total += item.price * item.quantity;  // calculation in handler
        if (item.stock < item.quantity) {
            return res.status(400).json({ error: 'insufficient stock' });  // domain rule in handler
        }
    }
    const order = await db.query('INSERT INTO orders ...', [req.user.id, total]);
    await emailService.sendConfirmation(req.user.email, order);  // side effect in handler
    res.json(order.rows[0]);
});
```

### Services / Business Logic Layer

**What should be here:** Domain rules, state transitions, orchestration of repository calls, business decisions.

**What should not be here:** HTTP request/response objects, raw SQL, framework-specific HTTP clients (unless wrapped), presentation logic.

**How to find violations:**

Look for in service/action files:
- `$request->` or `Request $request` parameters
- Raw SQL or direct ORM queries (if repositories are the established pattern)
- `response()`, `redirect()`, `view()` calls
- HTTP status codes
- Framework HTTP client calls without going through a declared client wrapper

### Repositories / Data Access Layer

**What should be here:** All database queries. Methods that return domain objects or collections.

**What should not be here:** Business logic, HTTP concerns, domain rules, conditional branching that is not about query construction.

**How to find violations:**

Look for in repository files:
- Sending notifications or emails
- Calling external HTTP APIs
- Domain logic that determines what to query based on business rules (this belongs in the service)
- Methods that do multiple unrelated database operations in one call

---

## Phase 2b: Find Pattern-Direction Violations

Beyond per-layer responsibility, the declared pattern imposes a **dependency direction**. Validate it.

**How to find violations:**

- Build the import graph for the codebase (or, at minimum, grep for cross-layer imports).
- For each import edge `A → B`, check it against the pattern's allowed direction.
- Flag any edge going the wrong way as a pattern violation distinct from a layer-responsibility violation.

**Examples:**

- Hexagonal: `src/domain/order.ts` imports from `src/infra/database/...` — domain depends on infra. **Violation.** Introduce a port (trait/interface) in domain and an adapter in infra implementing it.
- Layered: `app/Repositories/UserRepository.php` imports from `app/Http/Controllers/...` — lower depends on higher. **Violation.** Whatever data the repository needs must be passed in by the controller.
- Vertical slice: `features/orders/...` imports from `features/billing/...` — cross-slice import. **Violation** unless the import is from a declared shared kernel.

**Preferred tooling** (configure once, run in CI for framework rule 1.6c):

- PHP: `qossmic/deptrac`
- TypeScript/JavaScript: `dependency-cruiser`
- Java/Kotlin: ArchUnit
- Rust: `cargo-modules` graph + custom check; or test-based import assertions
- Python: `import-linter`

Report every pattern-direction violation as its own finding, distinct from layer-responsibility violations.

---

## Phase 3: Find Service Boundary Violations

### Shared Database Access

A shared database is when two services query the same database tables directly. This is one of the highest-risk couplings.

**How to find it:**

- Compare database connection strings or database names across services
- Look for the same table name appearing in query code in more than one service
- Check for shared ORM models that are imported across service boundaries

**What to do:** Document as a known architectural issue. Full remediation (introducing an API boundary) is a large change — flag it, do not silently fix it.

### Direct Cross-Service Code Calls

A direct cross-service call is when one service imports and calls code from another service's internal codebase (not through a declared API).

**How to find it:**

In a monorepo, look for:
- Import statements that cross top-level service directories
- `require('../other-service/...')` or equivalent
- Shared model objects defined in one service but imported by another

**What to do:** Replace with the declared communication mechanism (HTTP API call, queue message, shared contract package).

### Undeclared Package Dependencies

In a monorepo, a package depends on another package that is not declared in its manifest.

**How to find it:**

- Check `package.json` / `Cargo.toml` / `composer.json` dependencies for each package
- Search for imports that reference sibling packages not listed as dependencies

---

## Remediation

### Move Business Logic Out of a Controller

The pattern: extract the logic into the appropriate layer, then call it from the controller.

```php
// Before — logic in controller
public function store(Request $request): JsonResponse
{
    $user = User::where('email', $request->email)->first();
    if (!$user) { $user = User::create([...]); }
    if ($user->trial_ends_at < now()) { $user->subscription_status = 'expired'; }
    return response()->json($user);
}

// After — controller delegates to action
public function store(CreateUserRequest $request): JsonResponse
{
    $user = $this->createOrUpdateUser->execute(
        new CreateUserData($request->validated())
    );
    return UserResource::make($user);
}

// New action class
class CreateOrUpdateUser
{
    public function execute(CreateUserData $data): User
    {
        $user = $this->userRepository->findByEmail($data->email)
            ?? $this->userRepository->create($data);
        if ($user->trial_ends_at < now()) {
            $user = $this->userRepository->markSubscriptionExpired($user);
        }
        return $user;
    }
}
```

### Move Database Queries Into a Repository

```typescript
// Before — query in service
class OrderService {
    async createOrder(userId: string, items: CartItem[]): Promise<Order> {
        const cartItems = await db.query(
            'SELECT * FROM cart_items WHERE user_id = $1', [userId]
        );
        // ...
    }
}

// After — service calls repository
class OrderService {
    constructor(private cartRepository: CartRepository) {}

    async createOrder(userId: string): Promise<Order> {
        const cartItems = await this.cartRepository.findByUserId(userId);
        // ...
    }
}

class CartRepository {
    async findByUserId(userId: string): Promise<CartItem[]> {
        return db.query('SELECT * FROM cart_items WHERE user_id = $1', [userId]);
    }
}
```

### Replace Cross-Service Code Import with API Call

```typescript
// Before — direct import across service boundary
import { UserRepository } from '../../auth-service/repositories/UserRepository';

// After — call through the declared HTTP client
import { AuthServiceClient } from '../clients/AuthServiceClient';

class SomeService {
    constructor(private authClient: AuthServiceClient) {}

    async getUser(userId: string): Promise<User> {
        return this.authClient.getUser(userId);
    }
}
```

---

## Commit Strategy

One commit per layer:

```
refactor: extract business logic from order controllers into actions

Moved domain logic from OrderController, CheckoutController into
CreateOrder, ProcessRefund, and CancelOrder action classes.
Controllers now handle HTTP concerns only.
```

```
refactor: move raw queries from services into repository layer

Extracted 18 inline queries from OrderService and PaymentService
into OrderRepository and PaymentRepository.
```

---

## Quality Check

After remediation:

- [ ] `ARCHITECTURE.md` names the architectural pattern (framework rule 2.1g); audit's findings reference it
- [ ] No controller file contains database queries or domain logic
- [ ] No service file contains HTTP request/response objects or raw SQL (if repositories are the pattern)
- [ ] No direct cross-service code imports
- [ ] All pattern-direction violations (Phase 2b) are remediated or documented
- [ ] A pattern-enforcement tool or test is configured to run in CI (framework rule 1.6c) — deptrac, dependency-cruiser, ArchUnit, import-linter, or equivalent
- [ ] Any shared database access is documented in `ARCHITECTURE.md` as a known issue
- [ ] Tests pass

---

## Escalation Conditions

Stop and ask before proceeding if:

- `ARCHITECTURE.md` does not name an architectural pattern — stop and route the user to the `architecture-md` skill first; resuming this audit without a declared pattern produces opinion-based findings, not violations
- The established layer structure cannot be determined — there is no `AGENTS.md`, no `RULES.md`, and the code itself is inconsistent — ask the user which structure to standardise on
- A violation is deeply embedded and the extraction would require changes to more than 10 files — agree on scope before proceeding
- A shared database is found — this is an architectural decision that requires human judgement about the remediation path
- The violation appears to be intentional — e.g. a deliberate shortcut with a documented reason — confirm before refactoring
