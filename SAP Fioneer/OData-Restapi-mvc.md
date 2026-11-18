# OData, REST APIs and MVC Pattern – Interview Notes (Java Backend)

> JD lines: *"Spring Boot, REST APIs, Microservices, OData"* and *"UI development framework like React.js / Angular / UI5 using MVC Pattern"*

---

# PART 1: REST APIs

## 1.1 What is REST?

**REST (Representational State Transfer)** is an **architectural style** (not a protocol) for designing networked APIs, introduced by Roy Fielding. Clients interact with **resources** (identified by URIs) using standard HTTP methods, exchanging **representations** (usually JSON).

**Say in interview:** *"REST models everything as resources identified by URIs, manipulated through standard HTTP verbs, stateless between requests, with standard status codes and representations like JSON."*

### Six REST Constraints

| Constraint | Meaning |
|---|---|
| **Client-Server** | UI and data/logic separated; evolve independently |
| **Stateless** | Each request carries everything needed; server keeps no client session state |
| **Cacheable** | Responses declare whether they can be cached |
| **Uniform Interface** | Resource URIs, standard methods, self-descriptive messages, HATEOAS |
| **Layered System** | Proxies, gateways, load balancers can sit between client and server |
| **Code on Demand** | (Optional) Server can send executable code (e.g., JavaScript) |

---

## 1.2 HTTP Methods: Safe and Idempotent

| Method | Purpose | Safe | Idempotent | Typical success code |
|---|---|---|---|---|
| **GET** | Read resource | Yes | Yes | 200 |
| **POST** | Create / non-idempotent action | No | **No** | 201 (with `Location` header) |
| **PUT** | Full replace / create at known URI | No | Yes | 200 / 204 |
| **PATCH** | Partial update | No | Not guaranteed | 200 / 204 |
| **DELETE** | Remove | No | Yes | 204 |
| **HEAD** | Headers only | Yes | Yes | 200 |
| **OPTIONS** | Supported methods (CORS preflight) | Yes | Yes | 200 / 204 |

- **Safe** = does not modify state. **Idempotent** = same request repeated gives the same server state.
- **PUT vs PATCH:** PUT replaces the whole resource; PATCH changes only the sent fields.
- **PUT vs POST:** PUT is idempotent and targets a known URI; POST creates under a collection and the server assigns the ID.

---

## 1.3 HTTP Status Codes (Must Know)

| Range | Codes |
|---|---|
| **2xx Success** | 200 OK, 201 Created, 202 Accepted (async), 204 No Content |
| **3xx Redirect** | 301 Moved Permanently, 304 Not Modified (caching) |
| **4xx Client error** | 400 Bad Request, 401 Unauthorized (not authenticated), 403 Forbidden (not authorized), 404 Not Found, 405 Method Not Allowed, 409 Conflict, 415 Unsupported Media Type, 422 Unprocessable Entity, 429 Too Many Requests |
| **5xx Server error** | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

**401 vs 403:** 401 = "who are you?" (missing/invalid credentials). 403 = "I know you, but you are not allowed."

---

## 1.4 REST API Design Best Practices

1. **Nouns, not verbs**, plural collections: `/accounts`, `/accounts/{id}/transactions` (not `/getAccount`).
2. Use HTTP methods for actions, correct **status codes**.
3. **Versioning:** URI (`/api/v1/accounts`), header, or media type. URI versioning is the most common.
4. **Pagination, sorting, filtering:** `GET /accounts?page=0&size=20&sort=name,asc&status=ACTIVE`.
5. **Consistent error format** (RFC 7807 Problem Details):
```json
{
  "type": "https://bank.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Balance 50.00 is lower than requested 100.00",
  "instance": "/accounts/123/withdrawals"
}
```
6. **Idempotency keys** for POST (payments): client sends `Idempotency-Key` header; server dedupes retries.
7. **Security:** HTTPS, OAuth2/JWT, validate input, rate limiting, don't leak internals in errors.
8. **Caching:** `ETag`/`If-None-Match`, `Cache-Control`.
9. **Optimistic locking:** `ETag` + `If-Match` header (or `version` field) to avoid lost updates, returning 409/412 on conflict.
10. **Documentation:** OpenAPI/Swagger (`springdoc-openapi`).
11. **HATEOAS** (Richardson Maturity Level 3): responses include links to next actions. Often discussed, rarely fully implemented.
12. Use **DTOs**, never expose JPA entities directly.

### Richardson Maturity Model
- **L0:** single endpoint, POX/RPC style (like SOAP)
- **L1:** resources (separate URIs)
- **L2:** HTTP verbs + status codes (**most "REST" APIs**)
- **L3:** HATEOAS

---

## 1.5 REST with Spring Boot

```java
@RestController
@RequestMapping("/api/v1/accounts")
@RequiredArgsConstructor
public class AccountController {

    private final AccountService service;

    @GetMapping("/{id}")
    public ResponseEntity<AccountDto> get(@PathVariable Long id) {
        return ResponseEntity.ok(service.findById(id));
    }

    @GetMapping
    public Page<AccountDto> list(@RequestParam(required = false) String status,
                                 Pageable pageable) {
        return service.search(status, pageable);
    }

    @PostMapping
    public ResponseEntity<AccountDto> create(@Valid @RequestBody CreateAccountRequest req) {
        AccountDto created = service.create(req);
        URI location = URI.create("/api/v1/accounts/" + created.id());
        return ResponseEntity.created(location).body(created);       // 201 + Location
    }

    @PutMapping("/{id}")
    public AccountDto update(@PathVariable Long id, @Valid @RequestBody UpdateAccountRequest req) {
        return service.update(id, req);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        service.delete(id);
    }
}
```

### Centralized Exception Handling
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AccountNotFoundException.class)
    public ProblemDetail handleNotFound(AccountNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Account not found");
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Validation failed");
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage()).toList());
        return pd;
    }
}
```
(`ProblemDetail` is built in from Spring 6 / Boot 3.)

### Key Annotations

| Annotation | Use |
|---|---|
| `@RestController` | `@Controller` + `@ResponseBody` (returns JSON directly) |
| `@RequestMapping`, `@GetMapping`, ... | Map URL and HTTP method |
| `@PathVariable` | Value from URI path `/accounts/{id}` |
| `@RequestParam` | Query parameter `?status=ACTIVE` |
| `@RequestBody` | Deserialize JSON body into object |
| `@Valid` / `@Validated` | Bean Validation (`@NotNull`, `@Size`, `@Email`) |
| `@ResponseStatus` | Set HTTP status |
| `@RestControllerAdvice` + `@ExceptionHandler` | Global error handling |
| `@CrossOrigin` | CORS config |

### Calling Other REST APIs
`RestClient` (Spring 6.1+, modern), `WebClient` (reactive/non-blocking), `RestTemplate` (legacy, maintenance mode), OpenFeign (declarative). Always set **timeouts**, and add retry/circuit breaker (Resilience4j).

### Testing
- `@WebMvcTest` + `MockMvc` for controller slice tests.
- `@SpringBootTest` + `TestRestTemplate` / Testcontainers for integration tests.

---

## 1.6 REST vs SOAP (Quick Recap)
REST: architectural style, JSON, lightweight, HTTP semantics, cacheable. SOAP: protocol, XML only, WSDL contract, WS-* standards. (See SOAP notes.)

---

# PART 2: OData

## 2.1 What is OData?

**OData (Open Data Protocol)** is an **OASIS/ISO standard** that defines a uniform way to build and consume **RESTful APIs** for querying and manipulating data. It standardizes URL conventions, query options, metadata, and payload formats so clients don't need custom code for every API.

- Built on REST/HTTP; payload typically **JSON** (also Atom/XML in older V2).
- Self-describing via **`$metadata`** (like a WSDL/schema for data).
- Strongly used in the **SAP ecosystem** (SAP Gateway, S/4HANA, SAP UI5/Fiori, SAP CAP, SAP Fioneer products) and Microsoft (Dynamics, Power Platform, SharePoint).

**Say in interview:** *"OData is a standardized REST-based protocol that adds a common query language (`$filter`, `$select`, `$expand`, ...), a metadata document, and standard payload conventions, so generic clients such as UI5 can consume any OData service without custom integration code."*

### OData vs Plain REST

| Aspect | Plain REST | OData |
|---|---|---|
| Standardization | Convention only, each API differs | Formal standard |
| Querying | Custom params per API | Built-in `$filter`, `$orderby`, `$top`, `$skip`, `$select`, `$expand`, `$count` |
| Metadata | Optional (OpenAPI) | Mandatory **`$metadata`** (EDM) |
| Relationships | Custom links/nesting | Navigation properties, `$expand` |
| Batching | Custom | Standard `$batch` |
| Tooling | Generic | Generic OData clients, UI5 models, Power BI/Excel integration |
| Overhead | Lightweight | More rigid, heavier spec |

---

## 2.2 Core Concepts

| Concept | Description |
|---|---|
| **Service root** | Base URL, e.g., `/odata/v4/AccountService/` |
| **Entity Data Model (EDM)** | Schema describing the data |
| **Entity Type** | Structured type with a **key** (e.g., Account) |
| **Entity Set** | Collection of entities (e.g., `Accounts`) |
| **Property** | Attribute of an entity (primitive or complex) |
| **Navigation Property** | Relationship to other entities (Account → Transactions) |
| **Complex Type** | Structured type without key |
| **Function / Action** | Custom operations. **Function** = read-only (GET); **Action** = has side effects (POST) |
| **Service Document** | Lists available entity sets (`GET /`) |
| **Metadata Document** | `GET /$metadata` returns the full model (XML CSDL) |

---

## 2.3 OData Query Options (Core Knowledge)

| Option | Purpose | Example |
|---|---|---|
| `$select` | Return only some properties | `/Accounts?$select=id,name,balance` |
| `$filter` | Filter rows | `/Accounts?$filter=balance gt 1000 and status eq 'ACTIVE'` |
| `$orderby` | Sort | `/Accounts?$orderby=name asc,balance desc` |
| `$top` | Limit rows (page size) | `/Accounts?$top=10` |
| `$skip` | Offset | `/Accounts?$top=10&$skip=20` |
| `$count` | Total count | `/Accounts?$count=true` (V4) / `$inlinecount=allpages` (V2) |
| `$expand` | Include related entities | `/Accounts(1)?$expand=transactions` |
| `$search` | Free-text search | `/Accounts?$search=savings` |
| `$apply` | Aggregation/grouping (V4) | `/Sales?$apply=groupby((region),aggregate(amount with sum as total))` |
| `$format` | Response format | `?$format=json` |

### Filter Operators and Functions
- **Comparison:** `eq`, `ne`, `gt`, `ge`, `lt`, `le`
- **Logical:** `and`, `or`, `not`
- **String functions:** `contains(name,'abc')`, `startswith`, `endswith`, `length`, `tolower`
- **Date functions:** `year()`, `month()`, `day()`
- **Lambda:** `Orders/any(o: o/amount gt 100)`, `all(...)`

### Addressing
```
GET /Accounts                      → all accounts
GET /Accounts(1)                   → account with key 1
GET /Accounts(1)/name              → single property
GET /Accounts(1)/transactions      → navigation (related collection)
GET /Accounts(1)/$ref              → reference only
POST /Accounts                     → create
PATCH /Accounts(1)                 → partial update
PUT /Accounts(1)                   → replace
DELETE /Accounts(1)                → delete
```

### Batch Requests
`POST /$batch` bundles multiple operations in one HTTP call (reduces round trips); V4 supports JSON batch; **changesets** make a group of changes atomic.

### Sample V4 Response
```json
{
  "@odata.context": "$metadata#Accounts",
  "@odata.count": 120,
  "value": [
    { "id": 1, "name": "Savings", "balance": 2500.00 }
  ],
  "@odata.nextLink": "Accounts?$skiptoken=10"
}
```

**Server-driven paging:** server limits page size and returns `@odata.nextLink`; client follows it.

---

## 2.4 OData V2 vs V4 (Frequently Asked, SAP Context)

| Feature | V2 | V4 |
|---|---|---|
| Payload | Atom/XML or JSON (`d` wrapper: `{"d":{"results":[]}}`) | JSON by default, cleaner (`value`) |
| Count | `$inlinecount=allpages`, `$count` | `$count=true` |
| Actions/Functions | Function Imports | Actions and Functions (bound/unbound) |
| Aggregation | Limited | `$apply` |
| Batch | Multipart/mixed | JSON batch supported |
| Query features | Basic | `$search`, lambda operators, `$compute`, richer |
| Metadata | EDMX | CSDL (JSON option), vocabularies/annotations |
| Standard | Microsoft/community | **OASIS & ISO standard** |
| SAP | Gateway / many legacy Fiori apps | S/4HANA, RAP, CAP, newer Fiori |

---

## 2.5 OData in Java

| Library / Framework | Notes |
|---|---|
| **Apache Olingo** | Main Java OData library (V2 and V4). Server + client. Used in SAP Cloud SDK / SAP BTP Java apps |
| **SAP Cloud Application Programming (CAP) Java** | Generates OData V4 services from CDS models |
| **SAP Cloud SDK** | Typed OData clients for consuming SAP services |
| **Spring Boot + Olingo** | Custom `ODataHttpHandler` exposed via servlet / controller |
| **Spring Data REST / GraphQL** | Alternatives, not OData |

### Implementing OData V4 with Apache Olingo (Concept Flow)
1. Define the **EDM** via `CsdlEdmProvider` (entity types, sets, container).
2. Implement processors: `EntityCollectionProcessor`, `EntityProcessor` (read, create, update, delete).
3. Use `ODataHttpHandler` to route requests to the service:
```java
@RestController
@RequestMapping("/odata")
public class ODataController {

    @RequestMapping("/**")
    public void process(HttpServletRequest req, HttpServletResponse resp) {
        OData odata = OData.newInstance();
        ServiceMetadata edm = odata.createServiceMetadata(new AccountEdmProvider(), new ArrayList<>());
        ODataHttpHandler handler = odata.createHandler(edm);
        handler.register(new AccountEntityCollectionProcessor(accountService));
        handler.process(req, resp);
    }
}
```
4. In processors, read query options from `UriInfo` (`FilterOption`, `TopOption`, `SkipOption`, `ExpandOption`, `OrderByOption`) and **translate them into JPA/SQL queries** (e.g., `$filter` → JPA Criteria/Specification, `$top/$skip` → `Pageable`).
5. Serialize results with `ODataSerializer`.

**Design tips**
- Never apply `$filter` in-memory on big datasets: push it to the DB.
- Enforce **max page size** and limit `$expand` depth to prevent expensive queries.
- Secure with OAuth2/JWT, row-level authorization.
- Whitelist filterable/sortable properties; validate `$filter` to avoid injection (use parameterized queries).
- Keep `$metadata` stable; version through service path (`/v1/`, `/v2/`).

### Consuming OData
- **UI5:** `sap.ui.model.odata.v4.ODataModel` / `v2.ODataModel` bind controls directly to entity sets.
- **Java:** Olingo client, SAP Cloud SDK, or simple `RestClient` with the query string.

---

## 2.6 When to Use OData vs REST vs GraphQL

| Use | Choose |
|---|---|
| Data-centric CRUD with rich filtering, SAP/UI5/Fiori front ends, generic tooling (Excel/Power BI) | **OData** |
| Simple, flexible, lightweight service APIs, microservices, mobile | **REST** |
| Client-controlled shape of data, aggregating many sources, avoiding over/under-fetching | **GraphQL** |

---

# PART 3: MVC Pattern

## 3.1 What is MVC?

**Model-View-Controller** separates an application into three parts, improving **separation of concerns, testability and maintainability**.

| Part | Responsibility |
|---|---|
| **Model** | Data and business logic/state (entities, DTOs, services) |
| **View** | What the user sees (HTML/Thymeleaf, JSON, UI5 XML view, React component) |
| **Controller** | Handles user input/requests, calls the model, selects the view/response |

```
User → Controller → Model (business logic/data)
          │              │
          └──── View ◄───┘   → Response to user
```

**Say in interview:** *"MVC splits concerns so UI, business logic and request handling can evolve independently, with each part tested in isolation."*

---

## 3.2 MVC in Spring (Server Side)

**Flow (DispatcherServlet / front controller pattern):**
1. Request hits **DispatcherServlet** (front controller).
2. **HandlerMapping** finds the controller method (`@RequestMapping`).
3. **HandlerAdapter** invokes the **Controller** (arguments bound, validated).
4. Controller calls **Service layer**, which uses **Repository** (Model).
5. Controller returns a **model + view name** (`ModelAndView`) or a response body.
6. **ViewResolver** resolves the view (Thymeleaf/JSP); view renders HTML.
    - In `@RestController`, step 6 is replaced by an **HttpMessageConverter** (Jackson) writing JSON directly.

```java
@Controller
public class AccountPageController {

    @GetMapping("/accounts/{id}")
    public String show(@PathVariable Long id, Model model) {
        model.addAttribute("account", service.findById(id));   // Model
        return "account-details";                               // View name → templates/account-details.html
    }
}
```

**Typical Spring Boot layering:** `Controller → Service → Repository → DB`, with DTO/Entity mapping (MapStruct) between layers.

---

## 3.3 MVC on the Front End

### React
- Not strictly MVC; **component-based, unidirectional data flow** (state → UI via props). Often described as the **View** library; Model/Controller come from state management (Redux/Context/hooks) and services.
- Pattern: **Container (smart)** components handle logic/data; **Presentational (dumb)** components render UI.

### Angular
- Component-based with **MVVM-like** structure: Component class (controller/view-model), Template (view), Services (model/business logic), Dependency Injection, two-way binding.

### SAPUI5 / OpenUI5 (Preferred in JD)
- **True MVC** framework:
    - **Model:** `JSONModel`, `ODataModel` (V2/V4), `ResourceModel` (i18n). Data binding connects model to controls.
    - **View:** declarative **XML views** (also JS/JSON/HTML).
    - **Controller:** JavaScript with event handlers and lifecycle (`onInit`, `onBeforeRendering`, `onAfterRendering`, `onExit`).
- UI5 + OData: `ODataModel` talks to OData backend; controls bind to entity sets (`items="{/Accounts}"`).

```xml
<!-- View -->
<List items="{/Accounts}">
  <StandardListItem title="{name}" description="{balance}" />
</List>
```
```javascript
// Controller
onInit: function () { /* set up model, routing */ },
onPress: function (oEvent) { /* handle user action */ }
```

### MVC vs MVP vs MVVM

| Pattern | Key difference |
|---|---|
| **MVC** | Controller handles input and updates model/selects view; view may observe model |
| **MVP** | Presenter mediates everything; view is passive and talks only to presenter |
| **MVVM** | ViewModel exposes state; view **binds** to it (data binding, two-way); used in Angular, Vue, UI5 binding |

---

# Combined: How They Fit Together

```
UI5 / React / Angular (View + Controller, MVC on client)
        │  HTTP/JSON  (REST or OData)
        ▼
Spring Boot Controller  → Service → Repository (JPA/Hibernate) → PostgreSQL
        └─ Olingo OData processors / @RestController
```

**Interview line:** *"On the front end, a UI5 app follows MVC with an OData model. On the backend, a Spring Boot service exposes REST/OData endpoints through controllers, with business logic in the service layer and persistence via JPA."*

---

# Interview Questions and Crisp Answers

**Q1. What is REST? Name its constraints.**
Architectural style for resource-based APIs over HTTP. Constraints: client-server, stateless, cacheable, uniform interface, layered system, optional code on demand.

**Q2. PUT vs POST vs PATCH?**
POST creates/non-idempotent actions (server picks ID). PUT replaces a resource at a known URI, idempotent. PATCH partially updates.

**Q3. What does idempotent mean? Which methods?**
Repeating the request yields the same server state. GET, PUT, DELETE, HEAD, OPTIONS are idempotent; POST is not. Use idempotency keys for POST in payments.

**Q4. 401 vs 403? 400 vs 422? 404 vs 204?**
401 unauthenticated, 403 unauthorized. 400 malformed request, 422 semantically invalid. 404 resource absent; 204 success with no body.

**Q5. How do you version a REST API?**
URI (`/v1`), header, or content negotiation; prefer non-breaking additive changes; deprecate gradually.

**Q6. How do you handle exceptions in Spring REST?**
`@RestControllerAdvice` with `@ExceptionHandler` returning a consistent `ProblemDetail` (RFC 7807) error body and correct status codes.

**Q7. `@Controller` vs `@RestController`?**
`@RestController` = `@Controller` + `@ResponseBody`; returns data (JSON) instead of view names.

**Q8. How do you make REST APIs secure?**
HTTPS, OAuth2/OIDC with JWT, Spring Security, input validation, rate limiting, CORS config, least privilege, no sensitive data in URLs/logs.

**Q9. How do you implement pagination?**
`Pageable` in Spring Data (`page`, `size`, `sort`) or cursor/keyset pagination for large/real-time datasets (more efficient than offset).

**Q10. What is OData?**
OASIS/ISO standard for RESTful data APIs adding standardized URL query options, `$metadata`, payload formats and batching.

**Q11. Name common OData query options.**
`$select`, `$filter`, `$orderby`, `$top`, `$skip`, `$count`, `$expand`, `$search`, `$apply`, `$batch`.

**Q12. OData vs REST?**
OData is a standardized flavor of REST with built-in querying, metadata and conventions; plain REST leaves these to each API's design.

**Q13. OData V2 vs V4?**
V4 is the OASIS/ISO standard with cleaner JSON, `$count=true`, actions/functions, `$apply` aggregation, `$search`, JSON batch. V2 uses `d.results` wrapper, `$inlinecount`, function imports; still widespread in SAP legacy.

**Q14. How would you implement OData in Java?**
Apache Olingo: define EDM via `CsdlEdmProvider`, implement entity processors, route through `ODataHttpHandler`, translate `$filter/$top/$skip/$orderby/$expand` into JPA queries. In SAP landscape, CAP Java generates OData from CDS.

**Q15. What is `$expand` and what's the risk?**
Inlines related entities in one call (avoids extra round trips). Risk: N+1 queries, large payloads, so limit depth/page size and fetch efficiently (JOIN FETCH/entity graphs).

**Q16. What is `$metadata`?**
Document describing the service's data model (entity types, sets, relationships, functions) in CSDL; enables generic clients and code generation.

**Q17. Explain MVC.**
Model (data/logic), View (presentation), Controller (handles input, coordinates). Separation of concerns improves testability and maintainability.

**Q18. Explain the Spring MVC request flow.**
DispatcherServlet → HandlerMapping → Controller (via HandlerAdapter) → Service/Repository → return ModelAndView → ViewResolver → View; for REST, `HttpMessageConverter` writes JSON.

**Q19. MVC vs MVVM?**
MVC: controller mediates between view and model. MVVM: view binds to a ViewModel via data binding; fewer manual DOM/UI updates.

**Q20. How does UI5 follow MVC?**
Views (XML) declare UI, Controllers hold logic/events, Models (JSON/OData) supply data via binding.

---

# Quick Revision Cheat Sheet

- **REST** = resources + HTTP verbs + status codes + stateless; maturity **L2** typical
- Idempotent: **GET, PUT, DELETE**; **POST** not
- **401** unauthenticated / **403** forbidden / **404** missing / **409** conflict / **422** validation
- Spring: `@RestController`, `@RequestBody`, `@Valid`, `@RestControllerAdvice`, `ProblemDetail`
- Use **DTOs**, pagination, versioning, OpenAPI docs, idempotency keys
- **OData** = REST + standard query language + `$metadata` + `$batch`
- Query options: **`$select $filter $orderby $top $skip $count $expand $search $apply`**
- **V4** = OASIS/ISO standard, `value` wrapper; **V2** = `d.results`, SAP legacy
- Java OData: **Apache Olingo**, **CAP Java**, SAP Cloud SDK; push filters to DB
- **MVC** = Model / View / Controller; Spring flow via **DispatcherServlet**
- **UI5** = true MVC (XML View + JS Controller + OData/JSON Model); React = component/unidirectional; Angular = component/MVVM-like

---

*Next topic suggestions: Spring Boot internals, Hibernate/JPA/EclipseLink, Microservices, PostgreSQL, Design Patterns, Java 8+ features, JavaScript ES6.*