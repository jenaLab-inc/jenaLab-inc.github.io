---
name: code
description: >
  Apply language-specific code style, naming, quality, and publishing/performance rules across all supported stacks.
  Use this skill whenever writing or reviewing code in Spring Boot, NestJS, React, Next.js,
  Python, Go, Rust, Flutter, Android (Kotlin), iOS (Swift), C#, C++, or Kotlin.
  Also applies when implementing UI rendering, animations, or platform-specific performance optimization.
---

# Code Style & Quality Skill

## Overview

Consistent code style reduces cognitive load, prevents bikeshedding in reviews, and enables automation. This skill defines the rules for each supported language and framework.

**Foundational Rules (apply to ALL languages):**
1. One formatter + linter preset per language. No personal IDE style deviations.
2. Domain terminology over technical jargon in names.
3. Explicit and concise over clever and compact.
4. CI enforces formatting, linting, and type checking — humans do not review style in PRs.

---

## 1. Naming Rules (Universal)

| Category | Rule |
|----------|------|
| Booleans | `is`, `has`, `can` prefix (e.g., `isActive`, `hasPermission`, `canEdit`) |
| Fetch | `fetch`, `get`, `load` prefix for data retrieval |
| Mutate | `create`, `update`, `delete`, `save`, `publish`, `send` for state changes |
| Classes | `PascalCase` |
| Functions/variables | `camelCase` (JS/TS/Kotlin/Java) or `snake_case` (Python/Go/Rust) |
| Constants | `UPPER_SNAKE_CASE` |
| Vague names — FORBIDDEN | `data`, `info`, `util`, `manager`, `helper`, `handler` (unless domain-specific) |
| Over-abbreviation — FORBIDDEN | `usrMgr`, `cfg`, `tmp`, `cnt` (full words unless universally agreed abbreviation) |

---

## 2. Code Quality Rules (Universal)

- **Early return** preferred over nested if-else
- **Magic strings / numbers** → extract as named constants
- **Wildcard imports** → forbidden
- **One primary export per file** (JS/TS/Kotlin)
- **No dead code** committed (commented-out code must be removed)
- **No console.log / print in production paths** — use structured logging
- **Functions do one thing** — if a function name needs "and", split it
- **Test coverage for business logic** — unit test first. Integration/E2E for end-to-end validation.

---

## 3. Java / Spring Boot

**Tooling:** Checkstyle + SpotBugs / ErrorProne + Google Java Format  
**Key Rules:**

```java
// ✅ Correct: thin controller, delegates to use case
@RestController
public class OrderController {
    private final CreateOrderUseCase createOrder;

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req) {
        var result = createOrder.execute(req.toCommand());
        return ResponseEntity.ok(OrderResponse.from(result));
    }
}

// ❌ Wrong: business logic in controller
@PostMapping("/orders")
public ResponseEntity<?> create(@RequestBody Map<String, Object> body) {
    // DB calls, validation, email sending all in one place
}
```

**Forbidden patterns:**
- Default package usage
- Controller calling Repository directly (skip application layer)
- Serializing JPA entity to API response
- `@Transactional` on controller or entity methods
- `public class OrderUtils` catch-all classes
- Magic strings (use `enum` or `public static final String`)

**Naming:**
- Classes: `PascalCase`
- Methods/variables: `camelCase`
- Constants: `UPPER_SNAKE_CASE`
- Packages: `lowercase.dot.separated`

---

## 4. TypeScript / NestJS

**Tooling:** ESLint (strict) + Prettier + `tsc --strict`  
**Key Rules:**

```typescript
// ✅ Correct: typed use case with clear boundary
export class CreateOrderUseCase {
  constructor(private readonly orderRepo: OrderRepository) {}

  async execute(cmd: CreateOrderCommand): Promise<OrderId> {
    const order = Order.create(cmd);
    return this.orderRepo.save(order);
  }
}

// ❌ Wrong: any-typed, mixed concerns
async createOrder(body: any) {
  const data = await this.db.query(`INSERT INTO orders...`);
  await sendEmail(body.email); // mixed side effect
  return data;
}
```

**TypeScript strict rules:**
- `strict: true` in `tsconfig.json` — no exceptions
- No `any` type unless explicitly justified and commented
- No non-null assertion (`!`) without null check justification
- All public functions have explicit return types

**Naming:**
- Classes / Interfaces / Types: `PascalCase`
- Variables / functions: `camelCase`
- Constants: `UPPER_SNAKE_CASE` or `camelCase` for module-level configs
- Enums: `PascalCase` with `PascalCase` members

**Forbidden:**
- `any` as a shortcut
- `console.log` in production code (use NestJS Logger)
- Barrel files (`index.ts`) that re-export everything indiscriminately

---

## 5. React

**Tooling:** ESLint (eslint-plugin-react-hooks) + Prettier + TypeScript strict  
**Key Rules:**

```tsx
// ✅ Correct: pure functional component
function OrderCard({ orderId, totalAmount }: OrderCardProps) {
  const { data: order } = useOrder(orderId); // data fetching in custom hook
  if (!order) return null;
  return <div>{formatCurrency(totalAmount)}</div>; // pure render
}

// ❌ Wrong: fetch inside leaf component, complex JSX logic
function OrderCard({ orderId }: { orderId: string }) {
  const [order, setOrder] = useState(null);
  useEffect(() => {
    fetch(`/api/orders/${orderId}`).then(r => r.json()).then(setOrder);
  }, [orderId]);
  return order ? (order.status === 'paid' ? <div /> : <span />) : null;
}
```

**Component rules:**
- Function components only — no class components in new code
- Component names: `PascalCase`
- Hook names: `use` + descriptive name (`useOrderDetails`, not `useData`)
- Props interface: `ComponentNameProps` suffix
- No nested ternaries in JSX — extract to variables or functions
- No inline handlers with complex logic — extract to named functions
- `dangerouslySetInnerHTML` → review required comment mandatory

**State rules:**
- Minimize state — derive values instead of storing them
- State separation: server state (React Query / SWR) / UI state (useState) / form state (React Hook Form) / URL state (search params)
- `useReducer` when state transitions become complex

**Forbidden:**
- Leaf components calling `fetch()` directly
- Cross-feature internal file imports (`../../billing/internal/...`)
- `useEffect` without dependency array review

---

## 6. Next.js (App Router)

**Tooling:** Same as React + Next.js ESLint config  
**Key Rules:**

```tsx
// ✅ Correct: Server Component by default
// app/orders/page.tsx
export default async function OrdersPage() {
  const orders = await getOrders(); // server-only call
  return <OrderList orders={orders} />;
}

// Only interactive leaf uses "use client"
// app/orders/_components/OrderFilter.tsx
'use client';
export function OrderFilter({ onFilter }: OrderFilterProps) {
  const [query, setQuery] = useState('');
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

**Rules:**
- `use client` → smallest possible interactive leaf only
- `server-only` package for modules that must never reach the browser
- Route Handlers (`app/api/*/route.ts`): HTTP adapter pattern only — validate + auth + delegate to use case
- `generateMetadata` for all pages (SEO)
- Image optimization: always use `next/image`

**Forbidden:**
- Full page components marked `use client` by default
- Secret keys / DB credentials accessible in client bundle
- Business logic in route files
- Uncontrolled `fetch` without caching strategy

---

## 7. Python

**Tooling:** Black + Ruff (replaces flake8/isort/pyflakes) + mypy (strict)  
**Key Rules:**

```python
# ✅ Correct: typed, explicit, no import-time side effects
from typing import Optional
from uuid import UUID

def get_order_total(order_id: UUID, *, include_tax: bool = False) -> Decimal:
    """Calculate the total for a given order.

    Args:
        order_id: The unique identifier of the order.
        include_tax: Whether to include tax in the total.
    Returns:
        Total amount as Decimal.
    """
    ...

# ❌ Wrong: untyped, mixed None/False return, import-time side effect
import db  # db.connect() called at import time!

def get_total(id):
    result = db.query(id)
    if not result:
        return False  # mixing None / False / dict
    return result
```

**Naming:**
- Functions / variables: `snake_case`
- Classes: `PascalCase`
- Constants: `UPPER_SNAKE_CASE`
- Private: leading underscore `_private_method`

**Rules:**
- All public functions/methods: type hints mandatory
- All public modules, classes, functions: docstrings required
- No import-time side effects (no network, DB, or env init on import)
- Exceptions must be explicit — no mixed return type conventions

**Forbidden:**
- Giant `utils.py` or `helpers.py`
- `**kwargs` as a lazy catch-all for typed functions
- `except Exception: pass` (swallowing exceptions)

---

## 8. Go

**Tooling:** `gofmt` + `goimports` + `golangci-lint`  
**Key Rules:**

```go
// ✅ Correct: error as return value, short package name
package order

func (r *Repository) FindByID(ctx context.Context, id OrderID) (*Order, error) {
    row := r.db.QueryRowContext(ctx, "SELECT * FROM orders WHERE id = $1", id)
    return scanOrder(row)
}

// ❌ Wrong: panic for normal errors, meaningless package name
package util

func GetOrderOrPanic(id string) *Order {
    o, err := db.Find(id)
    if err != nil {
        panic(err) // never for business failures
    }
    return o
}
```

**Naming:**
- Packages: short, lowercase, meaningful (`order`, `billing`, not `util`, `common`)
- Exported: `PascalCase`
- Unexported: `camelCase`
- No `Get` prefix on getters: `order.Status()` not `order.GetStatus()`
- Error variables: `ErrNotFound`, `ErrInvalidInput` (sentinel errors)
- Interfaces: behavior-describing names (`Reader`, `Storer`, not `IRepository`)

**Forbidden:**
- `panic` for business failures
- Exporting packages that don't need to be public
- Logging errors and continuing as if they didn't happen

---

## 9. Rust

**Tooling:** `rustfmt` + `clippy` (warn level: all)  
**Key Rules:**

```rust
// ✅ Correct: Result for recoverable, descriptive error type
pub fn find_order(id: OrderId) -> Result<Order, OrderError> {
    order_repo::find(id).map_err(OrderError::NotFound)
}

// ❌ Wrong: unwrap in production path
pub fn find_order(id: OrderId) -> Order {
    order_repo::find(id).unwrap() // panics on error
}
```

**Naming:**
- Types / Traits / Enums: `PascalCase`
- Functions / variables: `snake_case`
- Constants: `UPPER_SNAKE_CASE`

**Rules:**
- `Result<T, E>` for recoverable errors — always
- `panic!` only for invariant violations or unrecoverable bootstrap errors
- Prefer traits for abstraction (ports). Adapters implement traits.
- `unsafe` blocks require a mandatory safety comment

**Forbidden:**
- `unwrap()` / `expect()` in production code paths without explicit justification
- Unreviewed `unsafe` blocks
- Single massive module containing all application logic

---

## 10. Kotlin / Android

**Tooling:** ktlint + Detekt + Android Lint  
**Key Rules:**

```kotlin
// ✅ Correct: immutable data, nullable expressed in type
data class Order(
    val id: OrderId,
    val status: OrderStatus,
    val userId: UserId,
)

// ❌ Wrong: mutable, meaningless type name, lost nullability
data class OrderData(
    var id: String?,
    var status: String,
)
```

**Naming:**
- Classes / Interfaces / Objects: `PascalCase`
- Functions / variables: `camelCase`
- Constants: `UPPER_SNAKE_CASE`
- Files: named after primary content (`OrderRepository.kt`, not `Util.kt`)

**Rules:**
- `val` and immutable data classes preferred over `var`
- Nullability expressed explicitly in type signatures — never assume nullable-by-default
- Extension functions only when they genuinely improve readability
- Android projects: Kotlin naming is secondary to Android architecture rules

**Forbidden:**
- Meaningless file/class suffixes: `Util.kt`, `Helper.kt`, `Manager.kt` (without domain context)
- Giant extension file dumps
- Nullable-by-default public API unless genuinely optional

---

## 11. Swift / iOS

**Tooling:** SwiftFormat + SwiftLint  
**Key Rules:**

```swift
// ✅ Correct: structured concurrency, clear ownership
func fetchOrder(id: OrderID) async throws -> Order {
    try await orderRepository.find(id: id)
}

// ❌ Wrong: completion callback chaos, optional misuse
func fetchOrder(id: String, completion: @escaping (Any?, Error?) -> Void) {
    URLSession.shared.dataTask(with: ...) { data, _, error in
        completion(data, error) // no type safety
    }.resume()
}
```

**Naming:**
- Types / Protocols: `UpperCamelCase`
- Variables / functions: `lowerCamelCase`
- Boolean: `is`, `has`, `can` prefix
- Methods: name reads naturally at call site (`order.cancel()` not `order.performCancellation()`)

**Rules:**
- Structured concurrency (`async/await`) by default — avoid callback-based APIs for new code
- `Sendable` for types that cross concurrency boundaries
- `protocol`-first for abstraction
- Public API must have documentation comments

**Forbidden:**
- Networking / persistence inside View or ViewController
- Force unwrapping (`!`) without explicit nil-check justification
- Shared mutable state across concurrency boundaries without synchronization

---

## 12. Flutter

**Tooling:** `dart format` + `flutter analyze`  
**Key Rules:**

```dart
// ✅ Correct: pure build method
class OrderCard extends StatelessWidget {
  const OrderCard({super.key, required this.order});
  final Order order;

  @override
  Widget build(BuildContext context) {
    // Only render — no API calls, no state mutation
    return Card(child: Text(order.totalAmount.formatted));
  }
}

// ❌ Wrong: API call inside build
@override
Widget build(BuildContext context) {
  ApiClient.getOrder(orderId).then((order) { // never do this
    setState(() => _order = order);
  });
  return ...;
}
```

**Naming:**
- Classes: `UpperCamelCase`
- Variables / functions: `lowerCamelCase`
- Private: `_privateField`

**Per-feature structure:**
```
features/<feature>/
  view/       ← Widgets (pure, stateless where possible)
  view_model/ ← State + intent handling
  repository/ ← Combines local/remote sources
  service/    ← Low-level HTTP/DB adapter
  model/      ← Data models
```

**Forbidden:**
- Direct API calls inside `build()` or widget constructors
- Single global provider managing all screen state
- Feature boundaries dissolved into a shared `lib/` dump

---

## 13. C# / .NET

**Tooling:** `.editorconfig` + Roslyn Analyzers + `dotnet format`  
**Key Rules:**

```csharp
// ✅ Correct: async suffix, thin controller
public class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> CreateAsync(CreateOrderRequest request)
    {
        var result = await _createOrder.ExecuteAsync(request.ToCommand());
        return Ok(OrderResponse.From(result));
    }
}

// ❌ Wrong: sync, EF entity leaking, no separation
[HttpPost]
public IActionResult Create(OrderEntity entity) // EF entity as input!
{
    _context.Orders.Add(entity);
    _context.SaveChanges();
    return Ok(entity); // leaks internal DB structure
}
```

**Naming:**
- Classes / Methods / Properties: `PascalCase`
- Local variables / parameters: `camelCase`
- Private fields: `_camelCase`
- Async methods: `MethodAsync` suffix

**Forbidden:**
- Domain layer depending on EF Core or ASP.NET types
- Service Locator pattern
- Swallowing exceptions with `null`/`false` returns

---

## 14. C++

**Tooling:** `clang-format` + `clang-tidy` (cppcoreguidelines-* rules in CI)  
**Key Rules:**

```cpp
// ✅ Correct: RAII, value semantics, no raw new/delete
class OrderProcessor {
public:
    explicit OrderProcessor(std::unique_ptr<OrderRepository> repo)
        : repo_(std::move(repo)) {}

    std::expected<OrderId, ProcessError> process(const OrderCommand& cmd);
private:
    std::unique_ptr<OrderRepository> repo_;
};

// ❌ Wrong: raw pointer ownership, manual memory management
class OrderProcessor {
    OrderRepository* repo; // who deletes this?
public:
    OrderProcessor(OrderRepository* r) : repo(r) {}
};
```

**Rules:**
- Modern C++ (C++17 minimum, C++20 preferred)
- RAII for all resource management — no raw `new`/`delete` in business code
- Rule of Zero first; define special members only when owning resources directly
- Library/target boundaries define domain separation
- Executables kept thin (entry point + wiring only)

**Forbidden:**
- Business logic in macros
- Giant monolithic header files
- Raw pointer APIs without clear ownership semantics

---

## 15. Publishing & Performance Rules

Core principle: **"Build small, read late, render only what is needed, keep state as close as possible."**

These rules apply to all UI code — web, mobile, and cross-platform. Detailed per-platform rules are in the `ref/` directory.

### Universal Publishing Principles

- Split every screen into **small rendering units**.
- State lives **as close to the reader as possible** — lift only to the lowest common ancestor.
- UI renders **state results only** — side effects belong in lifecycle/effect scopes.
- **DO NOT render invisible content** — defer or skip it.
- Long lists: render **only the visible window**, never the full dataset.
- Animate **only properties that skip layout recalculation** (transform, opacity).
- **Cancel** network requests, listeners, and streams when the screen is removed.

### Platform-Specific References

| Platform | Reference |
|---|---|
| HTML / CSS | [`ref/publishing-html-css.md`](ref/publishing-html-css.md) |
| Flutter | [`ref/publishing-flutter.md`](ref/publishing-flutter.md) |
| Android Compose | [`ref/publishing-compose.md`](ref/publishing-compose.md) |
| iOS SwiftUI | [`ref/publishing-swiftui.md`](ref/publishing-swiftui.md) |
| Profiling Tools | [`ref/profiling-tools.md`](ref/profiling-tools.md) |

### Platform-Specific Forbidden Patterns

**Web:**
- Layout animation abuse
- Eager rendering thousands of DOM nodes
- Synchronous layout read/write on every scroll event
- SPA without listener cleanup
- Giant global mutable state

**Flutter:**
- Heavy computation inside `build()`
- `setState` on entire top-level tree
- Large lists with plain `ListView`
- Unnecessary `Opacity` / `Clip` / `saveLayer`
- Missing `dispose()` calls

**Compose:**
- Heavy work inside composable body
- Over-hoisted state
- Lazy lists without stable keys
- Flow collection ignoring lifecycle
- High-level composable directly subscribing to rapidly-changing state

**SwiftUI:**
- Expensive computation inside `body`
- Broad `@EnvironmentObject` overuse
- Unstable identity in `ForEach`
- Async tasks outliving view lifecycle
- Cramming different state dependencies into one `body`

---

## 16. PR Code Review Checklist

Use this during every code review. **Do NOT comment on formatting** — CI handles that.

```
[ ] Does the code solve the right problem?
[ ] Are naming conventions followed (domain terms, not technical slang)?
[ ] Is business logic in the correct layer (not leaking into controllers/views)?
[ ] Are all states handled (error, empty, loading, edge cases)?
[ ] Are tests present and testing behavior (not implementation details)?
[ ] Are there any magic strings/numbers that should be constants?
[ ] Does the change introduce any cross-layer dependency violations?
[ ] Are all new public APIs documented?
[ ] Is error handling explicit and consistent?
[ ] Does the PR description include before/after context (for frontend: screenshots)?
[ ] Does the screen render only what is visible? (lazy/windowed lists)
[ ] Do animations avoid triggering layout recalculation?
[ ] Are async tasks cleaned up when the screen disappears?
[ ] Is state subscription scope minimal for high-frequency state?
```
