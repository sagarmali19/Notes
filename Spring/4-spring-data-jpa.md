# JDBC, Hibernate & Spring Data — Interview-Ready Notes
### For Java Backend Developer Interviews

---

## Table of Contents
1. [JDBC (Java Database Connectivity)](#1-jdbc-java-database-connectivity)
2. [Hibernate (ORM)](#2-hibernate-orm)
3. [Spring JDBC](#3-spring-jdbc)
4. [Spring Data JPA](#4-spring-data-jpa)
5. [Quick-Fire Cross-Topic Comparison](#5-quick-fire-cross-topic-comparison)

---

# 1. JDBC (Java Database Connectivity)

## 1.1 What It Is
JDBC is a **standard Java API** (`java.sql`, `javax.sql`) for connecting to and executing SQL against relational databases. It's the lowest-level database access mechanism in the Java ecosystem — everything else (Spring JDBC, Hibernate, JPA) is ultimately built **on top of** JDBC.

JDBC itself is just an **interface specification**; each database vendor ships a **JDBC driver** (a `.jar`) that implements these interfaces for their specific database (MySQL Connector/J, PostgreSQL driver, Oracle ojdbc, etc.).

### JDBC Architecture
```
Application Code
      |
      v
  JDBC API (java.sql.*)         <- vendor-neutral interfaces
      |
      v
  JDBC Driver (vendor-specific) <- e.g. mysql-connector-j.jar
      |
      v
  Database (MySQL / PostgreSQL / Oracle...)
```

### Driver Types (conceptual, rarely used beyond Type 4 today)
| Type | Description |
|---|---|
| Type 1 | JDBC-ODBC bridge (obsolete, removed from modern Java) |
| Type 2 | Native-API driver — partly Java, calls native DB client libraries |
| Type 3 | Network-protocol driver — talks to a middleware server |
| Type 4 | **Pure Java driver** — talks directly to the DB over its native network protocol. **This is what's used in practice today** (e.g., `mysql-connector-j`, `postgresql` driver). |

---

## 1.2 Core JDBC Components

| Interface/Class | Purpose |
|---|---|
| `DriverManager` | Factory that manages a list of registered JDBC drivers and creates `Connection`s |
| `Connection` | Represents an active session/connection to the database |
| `Statement` | Executes static SQL (no parameters) |
| `PreparedStatement` | Executes **precompiled**, parameterized SQL — the standard choice |
| `CallableStatement` | Executes stored procedures |
| `ResultSet` | A cursor over the rows returned by a query |
| `SQLException` | Checked exception thrown for DB access errors |

### Basic JDBC Workflow (the 5 canonical steps)
```java
import java.sql.*;

public class JdbcDemo {
    public static void main(String[] args) {
        String url = "jdbc:mysql://localhost:3306/shopdb";
        String user = "root";
        String password = "secret";

        // 1. Register driver — implicit since JDBC 4.0 via Service Provider mechanism
        //    (driver jar on classpath is auto-discovered; Class.forName() is legacy but still seen)

        // 2. Establish connection
        try (Connection conn = DriverManager.getConnection(url, user, password)) {

            // 3. Create a statement
            String sql = "SELECT id, name, price FROM products WHERE price > ?";
            try (PreparedStatement ps = conn.prepareStatement(sql)) {
                ps.setDouble(1, 100.0);

                // 4. Execute and process the ResultSet
                try (ResultSet rs = ps.executeQuery()) {
                    while (rs.next()) {
                        long id = rs.getLong("id");
                        String name = rs.getString("name");
                        double price = rs.getDouble("price");
                        System.out.printf("%d: %s ($%.2f)%n", id, name, price);
                    }
                }
            }
        } catch (SQLException e) {
            e.printStackTrace(); // 5. Handle exceptions; try-with-resources auto-closes everything
        }
    }
}
```

**Interview note:** `try-with-resources` is critical here — `Connection`, `Statement`, and `ResultSet` all implement `AutoCloseable`. Forgetting to close them is a classic **resource leak** bug (exhausts DB connection pool over time).

---

## 1.3 Statement vs PreparedStatement vs CallableStatement

### `Statement` — static SQL, no parameters
```java
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM products WHERE id = 5"); // hardcoded value
```
⚠️ **Never build SQL by string concatenation with user input** — this is the classic **SQL Injection** vulnerability:
```java
// DANGEROUS — never do this
String sql = "SELECT * FROM users WHERE username = '" + userInput + "'";
// userInput = "' OR '1'='1" breaks out of the string and returns all rows
```

### `PreparedStatement` — parameterized, precompiled SQL (always prefer this)
```java
String sql = "SELECT * FROM products WHERE id = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setLong(1, 5L); // parameter is escaped/bound safely by the driver, not concatenated
ResultSet rs = ps.executeQuery();
```
**Why preferred:**
- **Prevents SQL injection** — parameters are sent separately from the SQL text, never interpreted as SQL syntax.
- **Performance** — the DB can cache/reuse the compiled execution plan across calls with different parameter values.
- Cleaner handling of types (dates, nulls, binary data) via typed setters (`setString`, `setInt`, `setDate`, `setNull`, etc.).

### `CallableStatement` — for stored procedures
```java
CallableStatement cs = conn.prepareCall("{call getEmployeeById(?, ?)}");
cs.setLong(1, 101L);                          // IN parameter
cs.registerOutParameter(2, Types.VARCHAR);    // OUT parameter
cs.execute();
String name = cs.getString(2);
```

### Comparison Table
| Aspect | `Statement` | `PreparedStatement` | `CallableStatement` |
|---|---|---|---|
| SQL | Static | Parameterized (`?`) | Calls stored procedures |
| SQL Injection risk | High | Low (parameters bound safely) | Low |
| Precompiled | No | Yes | Yes |
| Reused with different params | Inefficient | Efficient | Efficient |
| Extends | — | `Statement` | `PreparedStatement` |

---

## 1.4 Executing Statements: `executeQuery` vs `executeUpdate` vs `execute`

| Method | Use For | Returns |
|---|---|---|
| `executeQuery()` | `SELECT` statements | `ResultSet` |
| `executeUpdate()` | `INSERT` / `UPDATE` / `DELETE` / DDL | `int` — number of rows affected |
| `execute()` | Unknown/mixed statement types (e.g., stored procs that may return either) | `boolean` — `true` if a `ResultSet` was produced |

```java
PreparedStatement ps = conn.prepareStatement("UPDATE products SET price = ? WHERE id = ?");
ps.setDouble(1, 149.99);
ps.setLong(2, 5L);
int rowsAffected = ps.executeUpdate(); // returns 1 if a row was updated, 0 if id didn't exist
```

---

## 1.5 ResultSet

A `ResultSet` is a **cursor** pointing to a row in the query results — you must call `.next()` to advance before reading the first row.

```java
while (rs.next()) {
    long id = rs.getLong("id");         // by column name (preferred — readable)
    String name = rs.getString(2);      // by 1-based column index (faster but fragile)
}
```

### ResultSet Types
| Type | Behavior |
|---|---|
| `TYPE_FORWARD_ONLY` (default) | Cursor moves only forward — most efficient |
| `TYPE_SCROLL_INSENSITIVE` | Can move forward/backward; doesn't see concurrent DB changes |
| `TYPE_SCROLL_SENSITIVE` | Can move forward/backward; reflects concurrent DB changes |

```java
Statement stmt = conn.createStatement(
    ResultSet.TYPE_SCROLL_INSENSITIVE,
    ResultSet.CONCUR_READ_ONLY);
ResultSet rs = stmt.executeQuery("SELECT * FROM products");
rs.last();                     // jump to last row
int totalRows = rs.getRow();
rs.beforeFirst();              // reset cursor
```

**Interview note:** In practice, forward-only, read-only ResultSets are the overwhelming default — scrollable ResultSets are rarely used in modern apps (pagination is done via SQL `LIMIT`/`OFFSET` instead).

---

## 1.6 Transaction Management in JDBC

By default, a JDBC `Connection` is in **auto-commit mode** — every individual statement is committed immediately as its own transaction. For multi-statement atomicity, you must disable auto-commit and manage the transaction manually.

```java
Connection conn = DriverManager.getConnection(url, user, password);
try {
    conn.setAutoCommit(false); // start manual transaction

    PreparedStatement debit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance - ? WHERE id = ?");
    debit.setDouble(1, 500);
    debit.setLong(2, 1L);
    debit.executeUpdate();

    PreparedStatement credit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance + ? WHERE id = ?");
    credit.setDouble(1, 500);
    credit.setLong(2, 2L);
    credit.executeUpdate();

    conn.commit(); // both updates succeed together
} catch (SQLException e) {
    conn.rollback(); // if either update fails, undo both — atomicity preserved
} finally {
    conn.setAutoCommit(true);
    conn.close();
}
```

### Transaction Isolation Levels
| Level | Prevents |
|---|---|
| `READ_UNCOMMITTED` | Nothing — allows dirty reads |
| `READ_COMMITTED` | Dirty reads |
| `REPEATABLE_READ` | Dirty reads + non-repeatable reads |
| `SERIALIZABLE` | Dirty reads + non-repeatable reads + phantom reads (strictest, slowest) |

```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```

**Interview one-liner:** *"By default JDBC auto-commits every statement; for atomic multi-statement operations you disable auto-commit, execute all statements, then explicitly `commit()` or `rollback()` in a try/catch."*

---

## 1.7 Batch Processing

Executing many similar statements **one round-trip at a time** is slow. Batching groups them into a single network round-trip.

```java
String sql = "INSERT INTO products (name, price) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    conn.setAutoCommit(false);
    for (Product p : products) {
        ps.setString(1, p.getName());
        ps.setDouble(2, p.getPrice());
        ps.addBatch();               // queue this set of params
    }
    int[] results = ps.executeBatch(); // sends all at once
    conn.commit();
}
```
**Why it matters:** dramatically reduces round-trips for bulk inserts/updates (e.g., importing 10,000 rows) — can be an order of magnitude faster than executing statements individually.

---

## 1.8 Connection Pooling

Creating a raw JDBC `Connection` (TCP handshake + DB auth) is **expensive**. Connection pools pre-create and reuse a set of connections instead of opening/closing one per request.

- **HikariCP** — the modern standard, and **Spring Boot's default** connection pool (fast, lightweight, minimal overhead).
- Older alternatives: Apache DBCP2, C3P0, Tomcat JDBC Pool.

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/shopdb");
config.setUsername("root");
config.setPassword("secret");
config.setMaximumPoolSize(10);
HikariDataSource dataSource = new HikariDataSource(config);

try (Connection conn = dataSource.getConnection()) { // borrowed from the pool, not newly created
    // use connection
} // "closing" it here actually just returns it to the pool
```

**Interview note:** Calling `conn.close()` on a pooled connection doesn't physically close the TCP connection — it returns the connection object to the pool for reuse. This is transparent because the pool wraps the real `Connection` in a proxy.

---

## 1.9 Common JDBC Interview Q&A

**Q: What's the difference between `Statement.execute()`, `executeQuery()`, and `executeUpdate()`?**
A: See table in §1.4 — `executeQuery` for SELECT (`ResultSet`), `executeUpdate` for DML/DDL (`int` row count), `execute` for ambiguous/mixed cases (`boolean`).

**Q: Why is `PreparedStatement` preferred over `Statement`?**
A: SQL injection prevention (parameters bound, not concatenated) and better performance (precompiled/cached execution plans for repeated calls).

**Q: What is a `SQLException` and how do you handle multiple chained exceptions?**
A: `SQLException` is a checked exception representing a DB access error. JDBC drivers can chain multiple exceptions; iterate with `getNextException()` to see all of them, not just the first.

**Q: What problems does JDBC have that ORMs like Hibernate solve?**
A: Verbose boilerplate (manual `ResultSet` → object mapping), manual connection/resource management, no object-graph/relationship handling, no built-in caching, database-specific SQL scattered through code, and no automatic dirty-checking for updates.

---

# 2. Hibernate (ORM)

## 2.1 What It Is
Hibernate is an **Object-Relational Mapping (ORM)** framework — it maps Java objects (entities) to relational database tables, letting you work with **objects and their relationships** instead of hand-writing SQL and manually mapping `ResultSet` rows to objects.

Hibernate is the most popular **implementation of JPA** (Jakarta Persistence API — a specification). You can code against the JPA API and swap Hibernate for another provider (EclipseLink, OpenJPA) in theory, though Hibernate dominates in practice and also exposes its own native, richer API beyond plain JPA.

```
Your Code (Entities, Repositories)
        |
        v
   JPA (Specification / Interfaces — javax.persistence / jakarta.persistence)
        |
        v
   Hibernate (Implementation — does the actual ORM work)
        |
        v
   JDBC
        |
        v
   Database
```

### Why ORM?
- Eliminates repetitive `ResultSet` → object mapping boilerplate.
- Manages object relationships (one-to-many, many-to-many) automatically.
- **Automatic dirty checking** — Hibernate tracks changes to managed entities and generates `UPDATE` SQL only for changed fields, without you writing it.
- Built-in caching (first-level, optional second-level) reduces DB round-trips.
- Database portability — Hibernate generates dialect-specific SQL (`MySQLDialect`, `PostgreSQLDialect`, etc.), so switching databases mostly means changing config, not rewriting queries.

---

## 2.2 Core Building Blocks

| Concept | Purpose |
|---|---|
| `SessionFactory` | Heavyweight, thread-safe factory for `Session`s — created **once per application** (expensive to build) |
| `Session` | Lightweight, **not thread-safe**, represents a single unit of work / conversation with the DB — analogous to a JDBC `Connection` but object-oriented |
| `Transaction` | Represents a unit of work to be committed/rolled back atomically |
| `Entity` | A POJO mapped to a DB table via annotations (`@Entity`) |
| `Configuration` / `hibernate.cfg.xml` | Bootstraps Hibernate — DB connection info, dialect, mapped classes |

### Bootstrapping Hibernate (Plain, non-Spring)
```java
Configuration cfg = new Configuration();
cfg.configure("hibernate.cfg.xml"); // reads DB connection + mappings
SessionFactory sessionFactory = cfg.buildSessionFactory(); // build once, reuse app-wide

Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();
try {
    Employee emp = new Employee("Alice", "alice@example.com");
    session.persist(emp);    // INSERT generated automatically
    tx.commit();
} catch (Exception e) {
    tx.rollback();
} finally {
    session.close();
}
```

```xml
<!-- hibernate.cfg.xml -->
<hibernate-configuration>
    <session-factory>
        <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/shopdb</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">secret</property>
        <property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
        <property name="hibernate.hbm2ddl.auto">update</property>
        <property name="hibernate.show_sql">true</property>
        <mapping class="com.example.Employee"/>
    </session-factory>
</hibernate-configuration>
```

**Interview note:** In a Spring Boot app you almost never touch `SessionFactory`/`Session` directly — Spring Data JPA and `EntityManager` abstract this away entirely (see §4). Knowing raw Hibernate is still commonly asked to test fundamentals.

---

## 2.3 Entity Mapping Annotations

```java
import jakarta.persistence.*;

@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String name;

    @Column(unique = true)
    private String email;

    @Enumerated(EnumType.STRING)   // stores enum as readable string, not ordinal int
    private EmployeeStatus status;

    @Temporal(TemporalType.DATE)    // for legacy java.util.Date; not needed for java.time types
    private Date joiningDate;

    @Transient                      // excluded from persistence entirely
    private String cachedDisplayName;

    // no-arg constructor required by Hibernate (via reflection/proxying)
    protected Employee() {}

    public Employee(String name, String email) {
        this.name = name;
        this.email = email;
    }
    // getters/setters
}
```

### `@GeneratedValue` Strategies
| Strategy | Behavior |
|---|---|
| `IDENTITY` | DB auto-increment column generates the ID (e.g., MySQL `AUTO_INCREMENT`) |
| `SEQUENCE` | Uses a DB sequence object (common on PostgreSQL/Oracle) — allows batching ID allocation |
| `TABLE` | Simulates a sequence using a dedicated table — portable but slower, rarely used |
| `AUTO` | Hibernate picks a strategy based on the DB dialect |

**Interview note:** `IDENTITY` disables Hibernate's JDBC batch inserts for that entity (since the ID isn't known until after the INSERT executes), which can hurt bulk-insert performance — `SEQUENCE` avoids this limitation.

---

## 2.4 Entity Relationships

### One-to-Many / Many-to-One
```java
@Entity
class Department {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();
}

@Entity
class Employee {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id") // FK column lives on the "many" side
    private Department department;
}
```
- `mappedBy` marks the **inverse (non-owning)** side of the relationship — the owning side (with `@JoinColumn`) is what actually controls the foreign key in the DB.
- **Always set both sides** when adding to a bidirectional relationship, or the DB won't reflect the change (Hibernate persists based on the owning side):
```java
public void addEmployee(Employee e) {
    employees.add(e);
    e.setDepartment(this); // keep both sides in sync
}
```

### Many-to-Many
```java
@Entity
class Student {
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();
}

@Entity
class Course {
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

### One-to-One
```java
@Entity
class Employee {
    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "parking_spot_id")
    private ParkingSpot parkingSpot;
}
```

### Cascade Types
| Type | Effect |
|---|---|
| `PERSIST` | Saving the parent also saves the child |
| `MERGE` | Merging the parent also merges the child |
| `REMOVE` | Deleting the parent also deletes the child |
| `REFRESH` | Refreshing the parent also refreshes the child |
| `DETACH` | Detaching the parent also detaches the child |
| `ALL` | All of the above |

`orphanRemoval = true` additionally deletes a child automatically when it's **removed from the parent's collection**, even without deleting the parent — distinct from `CascadeType.REMOVE`, which only fires when the parent itself is deleted.

---

## 2.5 Fetch Types — Lazy vs Eager

| Fetch Type | Behavior | Default For |
|---|---|---|
| `LAZY` | Related data loaded **only when accessed** (proxy object returned initially) | `@OneToMany`, `@ManyToMany` |
| `EAGER` | Related data loaded **immediately**, in the same query (or a follow-up join) | `@ManyToOne`, `@OneToOne` |

```java
@ManyToOne(fetch = FetchType.LAZY) // override the EAGER default — usually a good idea
private Department department;
```

**Interview tip — always prefer `LAZY` by default**, and explicitly fetch what you need via `JOIN FETCH` in queries. Defaulting everything to `EAGER` silently pulls in large object graphs and tanks performance.

### The N+1 Query Problem (very common interview topic)
```java
List<Department> departments = session.createQuery("FROM Department", Department.class).list();
for (Department d : departments) {
    System.out.println(d.getEmployees().size()); // triggers a SEPARATE query per department!
}
// 1 query for departments + N queries for each department's employees = N+1 queries total
```
**Fixes:**
```java
// Fix 1 — JOIN FETCH in JPQL/HQL, loads everything in ONE query
"SELECT DISTINCT d FROM Department d JOIN FETCH d.employees"

// Fix 2 — @BatchSize, groups lazy loads into batches instead of one-by-one
@OneToMany(mappedBy = "department")
@BatchSize(size = 20)
private List<Employee> employees;

// Fix 3 — EntityGraph (see §4) to declaratively specify what to fetch eagerly per-query
```

### LazyInitializationException
Occurs when you try to access a lazy-loaded association **after** the `Session`/`EntityManager` that loaded it has closed (e.g., accessing `employee.getDepartment().getName()` in a view/controller layer, after the service method's transaction has ended).
**Fixes:** fetch what you need within the transaction (`JOIN FETCH`), use DTO projections instead of exposing entities, or (last resort, generally discouraged) use the Open Session In View pattern.

---

## 2.6 Entity Lifecycle States

| State | Meaning |
|---|---|
| **Transient** | A plain new object, not yet associated with a `Session`/`EntityManager`, no DB row |
| **Persistent (Managed)** | Associated with an active `Session`; changes are auto-tracked (dirty checking) and flushed to DB |
| **Detached** | Was persistent, but the `Session` is now closed — object still has data but changes aren't tracked |
| **Removed** | Marked for deletion; DELETE issued on flush/commit |

```java
Employee emp = new Employee("Bob", "bob@example.com"); // Transient
session.persist(emp);                                   // now Persistent — INSERT on flush
session.close();                                         // now Detached
emp.setName("Robert");                                   // change NOT tracked (detached)

Session newSession = sessionFactory.openSession();
Employee merged = newSession.merge(emp);                 // re-attaches (as a NEW managed copy); merged is now Persistent
```

### Key Session Methods
| Method | Behavior |
|---|---|
| `persist(obj)` | Makes a transient object persistent (schedules INSERT) |
| `merge(obj)` | Copies state of a detached object onto a managed instance; returns the managed instance |
| `remove(obj)` | Marks a managed entity for deletion |
| `find(Class, id)` / `get(Class, id)` | Retrieves by primary key; returns `null` if not found (JPA `find`) |
| `getReference(Class, id)` | Returns a **lazy proxy** without hitting the DB immediately — throws if accessed and the row doesn't actually exist |
| `flush()` | Forces pending changes to be synced to the DB (without committing the transaction) |
| `clear()` | Detaches all currently managed entities from the session |

---

## 2.7 HQL / JPQL and Criteria API

### HQL (Hibernate Query Language) / JPQL (JPA Query Language)
Object-oriented query language — queries **entities and their fields**, not tables/columns directly.

```java
// HQL
List<Employee> highEarners = session.createQuery(
    "FROM Employee e WHERE e.salary > :minSalary ORDER BY e.salary DESC", Employee.class)
    .setParameter("minSalary", 50000)
    .getResultList();

// Named/positional parameters, joins, aggregate functions all supported
String hql = "SELECT d.name, COUNT(e) FROM Department d JOIN d.employees e GROUP BY d.name";
```

### Criteria API — type-safe, programmatic query building (useful for dynamic queries)
```java
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<Employee> cq = cb.createQuery(Employee.class);
Root<Employee> root = cq.from(Employee.class);

cq.select(root).where(cb.greaterThan(root.get("salary"), 50000));
List<Employee> result = session.createQuery(cq).getResultList();
```
**When to use Criteria over HQL:** when the query structure (filters, sorts) is built dynamically at runtime based on user input (e.g., a flexible search form), rather than being a fixed query string.

### Native SQL (escape hatch)
```java
List<Employee> results = session.createNativeQuery(
    "SELECT * FROM employees WHERE department_id = :deptId", Employee.class)
    .setParameter("deptId", 3)
    .getResultList();
```

---

## 2.8 Caching

### First-Level Cache (L1) — always on, per-Session
- Automatic, cannot be disabled.
- Scoped to a single `Session`/`EntityManager` — cleared when the session closes.
- Guarantees that within one session, requesting the same entity by ID twice returns the **same object instance** without hitting the DB twice.

```java
Employee e1 = session.get(Employee.class, 1L); // hits DB
Employee e2 = session.get(Employee.class, 1L); // returns from L1 cache — no DB hit; e1 == e2
```

### Second-Level Cache (L2) — optional, SessionFactory-scoped
- Shared **across sessions**, needs an external cache provider (Ehcache, Caffeine, Redis via a Hibernate integration).
- Must be explicitly enabled per entity.

```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product { ... }
```
```properties
hibernate.cache.use_second_level_cache=true
hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
```
**Good for:** read-heavy, rarely-changing reference/lookup data (categories, config, static lists). **Bad for:** frequently-updated data, or data requiring absolute freshness (risk of stale reads across app instances).

### Query Cache (used alongside L2)
Caches the **result set of a query** (specifically the IDs returned), not the entity data itself — must be used together with L2 cache to be useful.

---

## 2.9 Common Hibernate Interview Q&A

**Q: `save()` vs `persist()`?**
A: `save()` is Hibernate-native, returns the generated ID immediately, and can be called outside a transaction (delays INSERT). `persist()` is the JPA standard method, returns `void`, and guarantees the operation only executes within a transaction context — `persist()` is generally preferred as the more portable, spec-compliant choice.

**Q: `get()` vs `load()`?**
A: `get()` hits the DB immediately and returns `null` if not found. `load()` returns a lazy proxy without hitting the DB immediately and throws `ObjectNotFoundException` only when the proxy is actually accessed and no row exists.

**Q: What is dirty checking?**
A: Hibernate compares a managed entity's current state to its loaded snapshot at flush time and automatically generates `UPDATE` SQL for any changed fields — you never need to call an explicit `update()` for a still-managed entity.

**Q: What's the difference between HQL and native SQL?**
A: HQL operates on entity objects/fields (portable across DB dialects); native SQL is raw, DB-specific SQL — an escape hatch for queries HQL can't express efficiently (complex vendor-specific functions, hints).

**Q: What causes a `LazyInitializationException` and how do you avoid it?**
A: Accessing a lazy association after its session has closed. Avoid via `JOIN FETCH`, DTO projections, or `@EntityGraph`.

---

# 3. Spring JDBC

## 3.1 What It Is
Spring JDBC is a **thin abstraction over raw JDBC** that eliminates the repetitive boilerplate (opening/closing connections, statements, result sets, exception handling) while still giving you full control over SQL — unlike Hibernate/JPA, it's **not** an ORM; you still write SQL yourself.

**When to reach for Spring JDBC over JPA:** when you want precise control over SQL (complex reporting queries, bulk operations, performance-critical paths) without the overhead of an ORM's object-graph management, caching, and dirty-checking machinery.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

---

## 3.2 JdbcTemplate — The Core Class

`JdbcTemplate` wraps raw JDBC and handles: acquiring/releasing connections, opening/closing statements and result sets, and translating `SQLException` into Spring's unified `DataAccessException` hierarchy (unchecked — no more forced try/catch everywhere).

```java
@Repository
public class ProductRepository {

    private final JdbcTemplate jdbcTemplate;

    public ProductRepository(JdbcTemplate jdbcTemplate) { // auto-configured bean if starter-jdbc + a DataSource are present
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<Product> findAll() {
        return jdbcTemplate.query(
            "SELECT id, name, price FROM products",
            (rs, rowNum) -> new Product(rs.getLong("id"), rs.getString("name"), rs.getDouble("price"))
        );
    }

    public Product findById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT id, name, price FROM products WHERE id = ?",
            (rs, rowNum) -> new Product(rs.getLong("id"), rs.getString("name"), rs.getDouble("price")),
            id
        );
    }

    public int save(Product p) {
        return jdbcTemplate.update(
            "INSERT INTO products (name, price) VALUES (?, ?)",
            p.getName(), p.getPrice()
        );
    }

    public int deleteById(Long id) {
        return jdbcTemplate.update("DELETE FROM products WHERE id = ?", id);
    }

    public int countAll() {
        return jdbcTemplate.queryForObject("SELECT COUNT(*) FROM products", Integer.class);
    }
}
```

### Key `JdbcTemplate` Methods
| Method | Use For |
|---|---|
| `query(sql, RowMapper, args...)` | SELECT returning multiple rows → `List<T>` |
| `queryForObject(sql, RowMapper/Class, args...)` | SELECT returning exactly one row/value |
| `update(sql, args...)` | INSERT/UPDATE/DELETE → returns affected row count |
| `batchUpdate(sql, List<Object[]>)` | Bulk INSERT/UPDATE in one round trip |
| `execute(sql)` | DDL or arbitrary statements |

---

## 3.3 RowMapper

Maps one `ResultSet` row to one domain object. Can be a **lambda** (as above) or a standalone reusable class.

```java
public class ProductRowMapper implements RowMapper<Product> {
    @Override
    public Product mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new Product(
            rs.getLong("id"),
            rs.getString("name"),
            rs.getDouble("price")
        );
    }
}
```
```java
List<Product> products = jdbcTemplate.query("SELECT * FROM products", new ProductRowMapper());
```
**Interview note:** unlike raw JDBC, you never manually call `rs.next()` in a loop — `JdbcTemplate` drives the iteration and invokes your `RowMapper` once per row, catching `SQLException` for you internally.

---

## 3.4 NamedParameterJdbcTemplate

Standard `JdbcTemplate` uses **positional `?` placeholders**, which get error-prone and hard to read with many parameters. `NamedParameterJdbcTemplate` uses **named placeholders** instead.

```java
@Repository
public class OrderRepository {
    private final NamedParameterJdbcTemplate namedTemplate;

    public OrderRepository(NamedParameterJdbcTemplate namedTemplate) {
        this.namedTemplate = namedTemplate;
    }

    public List<Order> findByStatusAndMinAmount(String status, double minAmount) {
        String sql = "SELECT * FROM orders WHERE status = :status AND amount >= :minAmount";
        Map<String, Object> params = Map.of("status", status, "minAmount", minAmount);
        return namedTemplate.query(sql, params,
            (rs, rowNum) -> new Order(rs.getLong("id"), rs.getString("status"), rs.getDouble("amount")));
    }

    // Using a bean-backed parameter source instead of a raw Map
    public int save(Order order) {
        String sql = "INSERT INTO orders (status, amount) VALUES (:status, :amount)";
        SqlParameterSource params = new BeanPropertySqlParameterSource(order); // reads status/amount via getters
        return namedTemplate.update(sql, params);
    }
}
```
**Why prefer it:** clearer SQL when there are many parameters, and parameter order in the SQL string no longer needs to match the order they're set in code — reduces a common source of bugs.

---

## 3.5 SimpleJdbcInsert (convenience for simple inserts)

```java
SimpleJdbcInsert insert = new SimpleJdbcInsert(dataSource)
    .withTableName("products")
    .usingGeneratedKeyColumns("id");

Map<String, Object> params = Map.of("name", "Laptop", "price", 999.99);
Number generatedId = insert.executeAndReturnKey(params);
```
Removes even more boilerplate for straightforward single-table inserts, and conveniently retrieves auto-generated keys.

---

## 3.6 Exception Translation — `DataAccessException` Hierarchy

Spring translates vendor-specific `SQLException`/error codes into a **consistent, unchecked exception hierarchy**, so your code doesn't need to know which database it's talking to.

```
DataAccessException (unchecked, RuntimeException)
 ├── DataIntegrityViolationException     (constraint violations)
 ├── DuplicateKeyException               (unique constraint violation)
 ├── EmptyResultDataAccessException      (queryForObject found 0 rows)
 ├── IncorrectResultSizeDataAccessException (expected 1 row, got 0 or >1)
 ├── CannotGetJdbcConnectionException     (connection/pool problems)
 ├── DeadlockLoserDataAccessException     (transaction deadlock)
 └── ... many more, mapped from SQLState/vendor error codes
```

**Why it matters:**
- These are **unchecked** — no forced `try/catch SQLException` scattered everywhere.
- Same exception types regardless of whether you're on MySQL, PostgreSQL, or Oracle — makes error-handling code portable and testable.
- Integrates cleanly with `@RestControllerAdvice` (Section 10 of the REST notes) for consistent API error responses.

```java
try {
    jdbcTemplate.update("INSERT INTO products (name) VALUES (?)", duplicateName);
} catch (DuplicateKeyException e) {
    // handle unique constraint violation specifically, regardless of underlying DB
}
```

---

## 3.7 Transaction Management with Spring JDBC

Spring JDBC integrates with Spring's declarative transaction management (`@Transactional`), which wraps the underlying JDBC connection's commit/rollback for you.

```java
@Service
public class OrderService {
    private final JdbcTemplate jdbcTemplate;

    public OrderService(JdbcTemplate jdbcTemplate) { this.jdbcTemplate = jdbcTemplate; }

    @Transactional
    public void placeOrder(Order order) {
        jdbcTemplate.update("INSERT INTO orders (status, amount) VALUES (?, ?)", "PENDING", order.getAmount());
        jdbcTemplate.update("UPDATE inventory SET stock = stock - ? WHERE product_id = ?",
            order.getQuantity(), order.getProductId());
        // if either statement throws, the whole method's changes roll back automatically
    }
}
```
Requires `DataSourceTransactionManager` (or `JpaTransactionManager` if JPA is also in play), auto-configured by Spring Boot when a `DataSource` is present.

---

## 3.8 Common Spring JDBC Interview Q&A

**Q: Why use `JdbcTemplate` instead of raw JDBC?**
A: Removes boilerplate (connection/statement/resultset lifecycle management), converts checked `SQLException` to unchecked `DataAccessException`, and integrates with Spring's declarative transaction management — while still giving full control over the SQL itself.

**Q: `JdbcTemplate` vs `NamedParameterJdbcTemplate`?**
A: Positional `?` params vs named `:param` params — named params are more readable/maintainable with many parameters and decouple parameter order in code from order in SQL.

**Q: `JdbcTemplate` vs Hibernate/JPA — when would you choose one over the other?**
A: `JdbcTemplate` for fine-grained SQL control, complex reporting/analytics queries, or bulk operations where ORM overhead isn't worth it. JPA/Hibernate for standard CRUD-heavy domain models where object-relationship mapping, caching, and dirty-checking save significant boilerplate.

---

# 4. Spring Data JPA

## 4.1 What It Is
Spring Data JPA is a layer **on top of JPA/Hibernate** that eliminates almost all repository boilerplate — you define an **interface**, and Spring generates the implementation (CRUD operations, query derivation from method names, pagination, etc.) at runtime via a dynamic proxy.

```
Your Repository Interface (extends JpaRepository)
        |
        v
Spring Data JPA (generates the implementation via proxy)
        |
        v
JPA (EntityManager)
        |
        v
Hibernate (JPA provider)
        |
        v
JDBC → Database
```

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

---

## 4.2 The Repository Interface Hierarchy

| Interface | Adds |
|---|---|
| `Repository<T, ID>` | Marker interface — no methods |
| `CrudRepository<T, ID>` | Basic CRUD: `save`, `findById`, `findAll`, `deleteById`, `count`, `existsById` |
| `PagingAndSortingRepository<T, ID>` | Adds `findAll(Pageable)`, `findAll(Sort)` |
| `JpaRepository<T, ID>` | Adds JPA-specific extras: batch operations, `flush()`, `saveAndFlush()`, `getReferenceById()` — **this is the one you extend in practice** |

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    // CRUD methods are already provided — no implementation needed!
}
```
```java
@Service
public class EmployeeService {
    private final EmployeeRepository repository;
    public EmployeeService(EmployeeRepository repository) { this.repository = repository; }

    public Employee create(Employee e) { return repository.save(e); }         // INSERT or UPDATE
    public Employee getById(Long id) { return repository.findById(id).orElseThrow(); }
    public List<Employee> getAll() { return repository.findAll(); }
    public void delete(Long id) { repository.deleteById(id); }
}
```
**Interview note:** `save()` does an `INSERT` if the entity's ID is `null`/new, or an `UPDATE`/`merge` if the entity already has an assigned ID — Spring Data determines this via `EntityInformation` (checking the ID field, or `@Version` if present).

---

## 4.3 Query Methods — Derived from Method Names

Spring Data JPA parses method names following a naming convention and auto-generates the underlying JPQL query — **no implementation code needed**.

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    List<Employee> findByName(String name);
    List<Employee> findByNameAndDepartment(String name, String department);
    List<Employee> findBySalaryGreaterThan(double salary);
    List<Employee> findByNameContainingIgnoreCase(String namePart);
    List<Employee> findByJoiningDateBetween(LocalDate start, LocalDate end);
    List<Employee> findByDepartmentOrderBySalaryDesc(String department);
    boolean existsByEmail(String email);
    long countByDepartment(String department);
    Optional<Employee> findFirstByOrderBySalaryDesc();
    void deleteByStatus(String status);
}
```

### Common Keywords
| Keyword | Example | Meaning |
|---|---|---|
| `And` / `Or` | `findByNameAndEmail` | Combine conditions |
| `Between` | `findBySalaryBetween` | Range |
| `LessThan` / `GreaterThan` | `findByAgeGreaterThan` | Comparison |
| `Like` / `Containing` / `StartingWith` | `findByNameContaining` | Pattern match |
| `IgnoreCase` | `findByNameIgnoreCase` | Case-insensitive |
| `OrderBy...Asc/Desc` | `findByDeptOrderBySalaryDesc` | Sorting |
| `In` | `findByIdIn(List<Long> ids)` | `WHERE id IN (...)` |
| `IsNull` / `IsNotNull` | `findByManagerIsNull` | Null checks |
| `True` / `False` | `findByActiveTrue` | Boolean field checks |
| `Distinct` | `findDistinctByDepartment` | De-duplicates results |
| `Top` / `First` | `findTop5ByOrderBySalaryDesc` | Limits result count |

**Interview note:** this is compile-time-**unsafe** — a typo in the method name (e.g., referencing a field that doesn't exist) fails only at **application startup** when Spring Data tries to parse the method into a query, not at compile time. Still much better than typo'd raw SQL strings, which fail at runtime on first execution.

---

## 4.4 `@Query` — Custom JPQL / Native SQL

For queries too complex for method-name derivation.

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // JPQL — operates on entities/fields
    @Query("SELECT e FROM Employee e WHERE e.department.name = :deptName AND e.salary > :minSalary")
    List<Employee> findHighEarnersInDept(@Param("deptName") String deptName, @Param("minSalary") double minSalary);

    // Native SQL — operates on actual tables/columns
    @Query(value = "SELECT * FROM employees WHERE salary > :minSalary", nativeQuery = true)
    List<Employee> findHighEarnersNative(@Param("minSalary") double minSalary);

    // Modifying query (UPDATE/DELETE) — requires @Modifying + a transaction
    @Modifying
    @Transactional
    @Query("UPDATE Employee e SET e.status = :status WHERE e.department.id = :deptId")
    int updateStatusForDepartment(@Param("deptId") Long deptId, @Param("status") String status);
}
```
**Interview note:** `@Modifying` is required for any `@Query` that isn't a `SELECT` (UPDATE/DELETE) — without it, Spring Data JPA expects a result set and throws an exception. These bulk operations also **bypass the persistence context** — entities already loaded in memory won't automatically reflect the change unless you clear/refresh the context.

---

## 4.5 Pagination and Sorting

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    Page<Employee> findByDepartment(String department, Pageable pageable);
}
```
```java
@Service
public class EmployeeService {
    public Page<Employee> getByDepartment(String dept, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("salary").descending());
        return repository.findByDepartment(dept, pageable);
    }
}
```

### `Page<T>` vs `Slice<T>` vs `List<T>`
| Return Type | Behavior |
|---|---|
| `Page<T>` | Includes total element/page count (runs an extra `COUNT` query) — for UI pagination with page numbers |
| `Slice<T>` | Only knows if a **next** page exists (`hasNext()`) — no `COUNT` query — cheaper, good for infinite-scroll UIs |
| `List<T>` | No pagination metadata at all |

```java
Page<Employee> result = repository.findByDepartment("Engineering", pageable);
List<Employee> content = result.getContent();
int totalPages = result.getTotalPages();
long totalElements = result.getTotalElements();
```

**Never use unbounded `findAll()` for large tables in production** — always paginate.

---

## 4.6 Entity Graphs (fixing N+1, declaratively)

```java
public interface DepartmentRepository extends JpaRepository<Department, Long> {

    @EntityGraph(attributePaths = {"employees"})
    List<Department> findAll(); // eagerly fetches "employees" for THIS query only, without changing the entity's default fetch type
}
```
This lets you keep associations `LAZY` by default (good practice — see §2.5) while opting into eager fetching **per-query**, exactly where it's needed.

---

## 4.7 Auditing

Spring Data JPA can automatically populate creation/modification metadata.

```java
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class Auditable {
    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}

@Entity
public class Employee extends Auditable {
    // ... regular fields
}
```
```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.of(SecurityContextHolder.getContext().getAuthentication().getName());
    }
}
```

---

## 4.8 Specifications — Dynamic, Type-Safe Queries

For queries whose filters are only known at runtime (e.g., a flexible search endpoint), `JpaSpecificationExecutor` lets you compose predicates programmatically instead of writing dozens of derived-query-method permutations.

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long>, JpaSpecificationExecutor<Employee> {}

public class EmployeeSpecifications {
    public static Specification<Employee> hasDepartment(String dept) {
        return (root, query, cb) -> dept == null ? null : cb.equal(root.get("department"), dept);
    }
    public static Specification<Employee> salaryAtLeast(Double min) {
        return (root, query, cb) -> min == null ? null : cb.greaterThanOrEqualTo(root.get("salary"), min);
    }
}
```
```java
Specification<Employee> spec = Specification
    .where(EmployeeSpecifications.hasDepartment(dept))
    .and(EmployeeSpecifications.salaryAtLeast(minSalary));
List<Employee> results = repository.findAll(spec); // only non-null conditions are applied
```

---

## 4.9 `@Transactional` and JPA

```java
@Service
public class EmployeeService {
    private final EmployeeRepository repository;

    @Transactional
    public void giveRaise(Long id, double amount) {
        Employee e = repository.findById(id).orElseThrow();
        e.setSalary(e.getSalary() + amount);
        // NO explicit save() call needed — the entity is "managed" within the transaction;
        // Hibernate's dirty checking auto-generates the UPDATE at commit/flush time
    }
}
```
**Interview note:** this is a favorite "gotcha" question — people expect you need `repository.save(e)`, but since `e` is a managed entity fetched within an active transaction, dirty checking handles the UPDATE automatically. `save()` is still good practice for clarity/explicitness, but isn't strictly required here.

`@Transactional(readOnly = true)` on read-only service methods is a useful optimization — Hibernate can skip dirty-checking overhead for entities loaded in that transaction.

---

## 4.10 Projections — Fetching Only What You Need

Avoids loading full entities (and all their lazy associations) when you only need a few fields — improves performance and avoids `LazyInitializationException` risk in read paths.

### Interface-Based Projection
```java
public interface EmployeeSummary {
    String getName();
    String getEmail();
}

public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    List<EmployeeSummary> findByDepartment(String department); // Spring generates a proxy implementing this
}
```

### DTO (Class-Based) Projection
```java
public record EmployeeDto(String name, String email) {}

public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    @Query("SELECT new com.example.dto.EmployeeDto(e.name, e.email) FROM Employee e WHERE e.department = :dept")
    List<EmployeeDto> findSummariesByDepartment(@Param("dept") String dept);
}
```

---

## 4.11 Common Spring Data JPA Interview Q&A

**Q: How does Spring Data JPA implement repository interfaces with no code?**
A: At startup, Spring creates a **dynamic proxy** implementing the repository interface. For inherited methods (`save`, `findById`, etc.), it delegates to a default `SimpleJpaRepository` implementation. For custom query methods, it parses the method name (or `@Query` annotation) into a JPQL/SQL query at startup and generates the execution logic.

**Q: `CrudRepository` vs `JpaRepository`?**
A: `JpaRepository` extends `PagingAndSortingRepository` (itself extending `CrudRepository`) and adds JPA-specific methods like `flush()`, `saveAndFlush()`, and batch delete/save variants — extend `JpaRepository` unless you have a specific reason to use a narrower interface.

**Q: What happens if a derived query method name doesn't match any entity field?**
A: Fails at **application startup** (Spring Data validates all repository method signatures eagerly when building the `ApplicationContext`), not at first invocation — a "fail fast" behavior similar to constructor injection's missing-bean detection.

**Q: How would you handle a search endpoint with many optional filters?**
A: `Specification`/`JpaSpecificationExecutor` for composable, type-safe dynamic queries, rather than writing an exploding number of derived query method combinations.

**Q: Spring Data JPA vs plain Hibernate vs Spring JDBC — how do you choose?**
A: Spring Data JPA for standard entity-centric CRUD apps (least boilerplate). Plain Hibernate/JPA (`EntityManager`) when you need finer control than Spring Data's abstractions provide but still want ORM benefits. Spring JDBC when you want raw SQL control without ORM overhead (reporting, bulk jobs, performance-critical paths).

---

# 5. Quick-Fire Cross-Topic Comparison

| Layer | Abstraction Level | You Write | Best For |
|---|---|---|---|
| **JDBC** | Lowest — raw API | Full SQL + manual `ResultSet` mapping + resource management | Full control, understanding fundamentals, non-Spring apps |
| **Spring JDBC (`JdbcTemplate`)** | Thin wrapper over JDBC | SQL + `RowMapper` | Precise SQL control without JDBC boilerplate; reporting/bulk ops |
| **Hibernate** | Full ORM | Entity mappings + HQL/Criteria (SQL auto-generated) | Object-relational domain models, when using plain JPA/Hibernate directly |
| **Spring Data JPA** | ORM + repository generation | Entity mappings + repository interfaces (implementation auto-generated) | Standard CRUD-heavy apps; minimal boilerplate |

| Topic | One-Line Answer |
|---|---|
| JDBC | Low-level, standard Java DB API; `DriverManager` → `Connection` → `PreparedStatement` → `ResultSet`; manual transaction/resource management |
| Hibernate | ORM mapping Java objects to tables; `SessionFactory`/`Session`, entity lifecycle (transient/persistent/detached), dirty checking, L1/L2 caching, HQL/Criteria |
| Spring JDBC | `JdbcTemplate` removes JDBC boilerplate, converts `SQLException` → unchecked `DataAccessException`, still SQL-first |
| Spring Data JPA | Repository interfaces auto-implemented by Spring; query derivation from method names, `@Query`, pagination (`Page`/`Slice`), specifications, auditing |
| N+1 problem | Lazy collection accessed in a loop triggers one query per item; fix with `JOIN FETCH`, `@EntityGraph`, or `@BatchSize` |
| Transaction management | JDBC: manual `commit()`/`rollback()`. Spring: declarative `@Transactional`, backed by `DataSourceTransactionManager`/`JpaTransactionManager` |