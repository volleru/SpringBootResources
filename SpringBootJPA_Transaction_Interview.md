give # Spring Boot JPA — Transaction Interview Questions & Answers

> 20 scenario-based easy questions covering `@Transactional`, propagation, rollback, isolation, and common pitfalls.

---

## Q1. What happens if you don't add `@Transactional` to a method that saves data?

**Scenario:** You call `repository.save(entity)` inside a service method with no `@Transactional`.

**Answer:**
Each `save()` call runs in its own auto-committed transaction (Spring Data JPA wraps each repository method in its own transaction by default). If you call `save()` multiple times in the same method, each runs independently — if the second fails, the first is already committed and **not rolled back**.

```java
// BAD — no shared transaction, partial saves possible
public void saveOrder(Order order, Payment payment) {
    orderRepo.save(order);    // commits immediately
    paymentRepo.save(payment); // if this fails, order is already saved
}

// GOOD — both save in one transaction
@Transactional
public void saveOrder(Order order, Payment payment) {
    orderRepo.save(order);
    paymentRepo.save(payment); // fails here → both rolled back
}
```

---

## Q2. What is the default rollback behavior of `@Transactional`?

**Answer:**
Spring rolls back only on **unchecked exceptions** (`RuntimeException` and its subclasses) and `Error` by default. Checked exceptions (`IOException`, `SQLException`) do **not** trigger a rollback unless you explicitly configure it.

```java
@Transactional
public void process() throws IOException {
    repo.save(entity);
    throw new IOException("checked"); // NOT rolled back by default!
}

// To rollback on checked exceptions:
@Transactional(rollbackFor = IOException.class)
public void process() throws IOException {
    repo.save(entity);
    throw new IOException("now rolled back");
}
```

---

## Q3. What is `REQUIRED` propagation and why is it the default?

**Answer:**
`REQUIRED` means: use the existing transaction if one exists, otherwise create a new one. It is the default because it ensures all operations within a call chain share the same transaction without needing explicit coordination.

```java
@Transactional(propagation = Propagation.REQUIRED) // default
public void methodA() {
    repo.save(a);
    methodB(); // joins the same transaction
}

@Transactional(propagation = Propagation.REQUIRED)
public void methodB() {
    repo.save(b); // same transaction as methodA
}
// If methodB throws → entire transaction (a + b) rolls back
```

---

## Q4. What does `REQUIRES_NEW` do and when would you use it?

**Answer:**
`REQUIRES_NEW` suspends the current transaction and starts a brand new one. Use it when you want an operation to commit independently — for example, writing an audit log even if the main transaction fails.

```java
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);
    auditService.log("Order placed"); // runs in its own transaction
    throw new RuntimeException("Order failed");
    // order rolled back, but audit log already committed
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void log(String message) {
    auditRepo.save(new AuditLog(message)); // commits independently
}
```

---

## Q5. What is the self-invocation problem with `@Transactional`?

**Answer:**
When a method within the same class calls another `@Transactional` method directly, Spring's proxy is bypassed and the transaction annotation is **ignored**.

```java
@Service
public class OrderService {

    public void placeOrder() {
        processPayment(); // calls directly — proxy bypassed, NO new transaction!
    }

    @Transactional
    public void processPayment() {
        // @Transactional has NO effect when called from same class
    }
}

// Fix: inject the bean and call through it
@Service
public class OrderService {

    @Autowired
    private OrderService self; // Spring injects the proxy

    public void placeOrder() {
        self.processPayment(); // goes through proxy — transaction works
    }

    @Transactional
    public void processPayment() { }
}
```

---

## Q6. What is `@Transactional(readOnly = true)` and when should you use it?

**Answer:**
Marks the transaction as read-only. This gives Hibernate a hint to skip dirty checking (no need to track changes), which can improve performance for queries. Use it on any method that only reads data.

```java
@Transactional(readOnly = true)
public List<Product> getAllProducts() {
    return productRepo.findAll(); // Hibernate skips dirty checking
}

@Transactional // readOnly = false (default) — needed for writes
public void updateProduct(Product p) {
    productRepo.save(p);
}
```

> Never use `readOnly = true` on methods that insert, update, or delete — writes may silently fail or throw exceptions.

---

## Q7. What happens if a `@Transactional` method calls another `@Transactional` method in a different class?

**Answer:**
The called method **joins the existing transaction** (because default propagation is `REQUIRED`). Both run in the same transaction. If either throws a `RuntimeException`, the entire transaction rolls back.

```java
@Service
public class OrderService {
    @Autowired InventoryService inventoryService;

    @Transactional
    public void placeOrder(Order order) {
        orderRepo.save(order);
        inventoryService.deductStock(order); // joins same transaction
    }
}

@Service
public class InventoryService {
    @Transactional // joins existing transaction from OrderService
    public void deductStock(Order order) {
        inventoryRepo.update(order.getProductId());
        // throws here → entire transaction (order + stock) rolls back
    }
}
```

---

## Q8. What is a `LazyInitializationException` and how do you fix it?

**Answer:**
Occurs when you try to access a lazily-loaded association outside of an active transaction. The session is already closed by the time the lazy field is accessed.

```java
// PROBLEM
public Order getOrder(Long id) {
    Order order = orderRepo.findById(id).get(); // transaction ends here
    return order;
}
// In controller:
order.getItems().size(); // LazyInitializationException — session closed!

// FIX 1: Keep transaction open
@Transactional
public Order getOrder(Long id) {
    Order order = orderRepo.findById(id).get();
    order.getItems().size(); // fetch inside transaction
    return order;
}

// FIX 2: Use JOIN FETCH in the query
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Order findByIdWithItems(@Param("id") Long id);

// FIX 3: Use DTO projections to avoid lazy loading entirely
```

---

## Q9. What is dirty checking in Hibernate and how does it relate to transactions?

**Answer:**
Within a transaction, Hibernate tracks all loaded entities (the "persistence context"). At the end of the transaction, it automatically detects changes and issues `UPDATE` SQL — **even without calling `save()`**.

```java
@Transactional
public void updateUserName(Long id, String name) {
    User user = userRepo.findById(id).get(); // entity is now managed
    user.setName(name); // just set the field — no save() needed!
    // Hibernate detects the change at commit and fires UPDATE automatically
}
```

> This only works inside a transaction. If the entity is detached (outside transaction), changes are not tracked.

---

## Q10. What is the difference between `NESTED` and `REQUIRES_NEW` propagation?

**Answer:**

| | `REQUIRES_NEW` | `NESTED` |
|---|---|---|
| New transaction? | Yes — fully independent | No — savepoint within outer transaction |
| Outer rollback affects it? | No | Yes — outer rollback rolls back nested too |
| Nested rollback affects outer? | No | No — only rolls back to savepoint |
| DB support needed? | No | Yes — savepoints (not all DBs support it) |

```java
@Transactional
public void outer() {
    repo.save(a);
    try {
        inner(); // nested — uses savepoint
    } catch (Exception e) {
        // inner rolled back to savepoint, outer continues
    }
    repo.save(b); // this still commits
}

@Transactional(propagation = Propagation.NESTED)
public void inner() {
    repo.save(c);
    throw new RuntimeException(); // rolls back only 'c', not 'a'
}
```

---

## Q11. What isolation level prevents dirty reads?

**Answer:**
`READ_COMMITTED` and above prevent dirty reads. `READ_UNCOMMITTED` allows them (reads data from uncommitted transactions).

```java
// Allows dirty reads (dangerous — rarely used)
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public Product getProduct(Long id) { ... }

// Default in most DBs — prevents dirty reads
@Transactional(isolation = Isolation.READ_COMMITTED)
public Product getProduct(Long id) { ... }

// Prevents dirty + non-repeatable reads
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Product getProduct(Long id) { ... }

// Prevents dirty + non-repeatable reads + phantom reads
@Transactional(isolation = Isolation.SERIALIZABLE)
public Product getProduct(Long id) { ... }
```

---

## Q12. What happens when you mark a method `@Transactional` in a `private` method?

**Answer:**
Nothing — `@Transactional` is **ignored** on `private` methods. Spring's proxy cannot intercept private methods. The annotation must be on `public` methods.

```java
@Service
public class UserService {

    @Transactional // IGNORED — private method!
    private void saveUser(User user) {
        userRepo.save(user);
    }

    @Transactional // WORKS — public method
    public void registerUser(User user) {
        userRepo.save(user);
    }
}
```

---

## Q13. How do you roll back a transaction manually without throwing an exception?

**Answer:**
Use `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` to mark the transaction for rollback without throwing.

```java
@Transactional
public void processOrder(Order order) {
    orderRepo.save(order);

    if (!isValid(order)) {
        // mark rollback without throwing
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        return;
    }

    paymentRepo.save(order.getPayment());
}
```

---

## Q14. What is `SUPPORTS` propagation?

**Answer:**
`SUPPORTS` means: run within a transaction if one exists, otherwise run **without** a transaction. Useful for read operations that can work with or without a transaction.

```java
@Transactional(propagation = Propagation.SUPPORTS)
public List<User> findAll() {
    // runs in transaction if caller has one, else runs non-transactionally
    return userRepo.findAll();
}
```

---

## Q15. What is an `OptimisticLockException` and how do you handle it?

**Answer:**
Occurs when two transactions try to update the same entity simultaneously and optimistic locking detects a version conflict. Use `@Version` on an entity field to enable optimistic locking.

```java
@Entity
public class Product {
    @Id Long id;
    String name;
    int stock;

    @Version
    int version; // Hibernate checks this on every update
}

// If two threads load version=1 and both try to save,
// the second update throws OptimisticLockException
@Transactional
public void updateStock(Long id, int qty) {
    try {
        Product p = productRepo.findById(id).get();
        p.setStock(p.getStock() - qty);
        productRepo.save(p);
    } catch (OptimisticLockException e) {
        // retry logic or inform user of conflict
        throw new RuntimeException("Product was updated by another user. Please retry.");
    }
}
```

---

## Q16. What does `@Transactional` on a class level do?

**Answer:**
Applies the transaction settings to **all public methods** in that class. Method-level annotations override the class-level one.

```java
@Service
@Transactional(readOnly = true) // default for all methods
public class ProductService {

    public List<Product> getAll() { // readOnly = true (inherits)
        return productRepo.findAll();
    }

    @Transactional // overrides to readOnly = false for this method
    public void save(Product p) {
        productRepo.save(p);
    }
}
```

---

## Q17. What is a phantom read and which isolation level prevents it?

**Answer:**
A phantom read occurs when a transaction re-executes a query and gets **different rows** because another transaction inserted or deleted rows in between.

```
Transaction A: SELECT * FROM orders WHERE amount > 100  → 5 rows
Transaction B: INSERT INTO orders (amount=200) → commits
Transaction A: SELECT * FROM orders WHERE amount > 100  → 6 rows (phantom!)
```

`SERIALIZABLE` isolation prevents phantom reads by locking the entire range.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public List<Order> getLargeOrders() {
    return orderRepo.findByAmountGreaterThan(100);
    // same result on every read within this transaction
}
```

---

## Q18. How does Spring handle transactions with multiple data sources?

**Answer:**
You need a `JtaTransactionManager` (distributed transaction manager like Atomikos) to coordinate transactions across multiple data sources. The default `JpaTransactionManager` only manages one data source.

```java
// application.yml
spring:
  jta:
    atomikos:
      enabled: true

// Entity for DS1
@Transactional("transactionManager1")
public void saveToDb1(User user) {
    db1UserRepo.save(user);
}

// Entity for DS2
@Transactional("transactionManager2")
public void saveToDb2(Account account) {
    db2AccountRepo.save(account);
}

// Distributed transaction across both — requires JTA
@Transactional // JtaTransactionManager coordinates both
public void transferData(User user, Account account) {
    db1UserRepo.save(user);
    db2AccountRepo.save(account); // both commit or both rollback
}
```

---

## Q19. What happens if you catch an exception inside a `@Transactional` method and don't rethrow it?

**Answer:**
The transaction is **not rolled back**. Spring only rolls back if the exception propagates out of the method. Swallowing the exception means the transaction commits normally, potentially saving corrupt data.

```java
// BAD — transaction commits even after the error
@Transactional
public void processPayment(Payment payment) {
    try {
        paymentRepo.save(payment);
        externalService.charge(); // throws RuntimeException
    } catch (Exception e) {
        log.error("Payment failed", e);
        // exception swallowed — transaction commits with bad state!
    }
}

// GOOD — rethrow so Spring can roll back
@Transactional
public void processPayment(Payment payment) {
    try {
        paymentRepo.save(payment);
        externalService.charge();
    } catch (Exception e) {
        log.error("Payment failed", e);
        throw e; // rethrow → Spring rolls back
    }
}
```

---

## Q20. What is the `NOT_SUPPORTED` propagation and when would you use it?

**Answer:**
`NOT_SUPPORTED` suspends any existing transaction and runs the method **without** a transaction. Use it for operations that must not run inside a transaction — for example, sending an email or calling an external API where a long transaction would hold DB locks unnecessarily.

```java
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);
    notificationService.sendConfirmationEmail(order); // suspends transaction
    inventoryRepo.update(order); // resumes transaction after email
}

@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void sendConfirmationEmail(Order order) {
    // runs without a transaction
    // DB locks are released during this slow external call
    emailClient.send(order.getEmail(), "Your order is confirmed!");
}
```

---

## Quick Reference Cheat Sheet

| Topic | Key Point |
|---|---|
| Default propagation | `REQUIRED` — join existing or create new |
| Default rollback | `RuntimeException` and `Error` only |
| Checked exception rollback | Use `rollbackFor = Exception.class` |
| Self-invocation | Proxy bypassed — inject self or refactor |
| `private` methods | `@Transactional` ignored |
| `readOnly = true` | Skips dirty checking — use for queries |
| `REQUIRES_NEW` | New independent transaction (audit logs) |
| `NESTED` | Savepoint within outer transaction |
| Dirty checking | No need to call `save()` on managed entity |
| Lazy loading | Access inside `@Transactional` or use JOIN FETCH |
| Swallowed exception | Transaction does NOT roll back — rethrow it |
| `@Version` | Enables optimistic locking |
| Multiple datasources | Requires JTA (Atomikos) |
