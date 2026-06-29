# Spring Boot Notes — Spring Data JPA (Repositories, Query Methods, @Query)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The layers: JPA vs Hibernate vs Spring Data JPA

These three names confuse every beginner. Here's the clean breakdown:

| Layer | What it is |
|-------|-----------|
| **JPA** | A **specification** (a set of rules/interfaces) for mapping Java objects to DB tables. Just the standard — no actual code that runs. |
| **Hibernate** | An **implementation** of JPA — the real library that does the work (the most popular one). |
| **Spring Data JPA** | A Spring layer **on top of** JPA/Hibernate that removes boilerplate — you write interfaces, it generates the code. |

```
Spring Data JPA   ← you write interfaces (least code)
      ↓ uses
     JPA (spec)    ← the standard interfaces
      ↓ implemented by
   Hibernate       ← actually talks to the DB
      ↓
   Database
```

**Interview one-liner:** JPA is the specification, Hibernate is the implementation that does the work, and Spring Data JPA sits on top to eliminate boilerplate repository code.

---

## 2. Entity — mapping a class to a table

*Definition:* An **`@Entity`** is a Java class mapped to a **database table**. Each object = one row; each field = one column.

```java
@Entity
@Table(name = "users")
class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)   // auto-increment id
    private Long id;

    @Column(nullable = false)
    private String name;

    private String email;
    // getters/setters
}
```

| Annotation | Meaning |
|-----------|---------|
| `@Entity` | This class maps to a table |
| `@Table(name=...)` | Table name (optional; defaults to class name) |
| `@Id` | The primary key |
| `@GeneratedValue` | Auto-generate the id |
| `@Column` | Column settings (name, nullable, length) |

**Interview one-liner:** An `@Entity` maps a class to a table; `@Id` marks the primary key and `@GeneratedValue` auto-generates it.

---

## 3. Repository — the magic interface

*Definition:* A **Repository** is an **interface** you write to access the database. You don't write any implementation — Spring Data **auto-generates** it at runtime, giving you CRUD methods for free.

```java
@Repository
interface UserRepository extends JpaRepository<User, Long> {
    //  <User, Long> = entity type, primary key type
}
```

That single interface gives you, for free:
```java
userRepo.save(user);            // create/update
userRepo.findById(1L);          // read one
userRepo.findAll();             // read all
userRepo.deleteById(1L);        // delete
userRepo.count();               // count
userRepo.existsById(1L);        // check exists
```

### The repository hierarchy
```
Repository (marker, empty)
  └── CrudRepository       → basic CRUD (save, findById, delete...)
        └── PagingAndSortingRepository  → + pagination & sorting
              └── JpaRepository  → + JPA extras (flush, batch, findAll(Sort)...)
```
**Use `JpaRepository`** — it includes everything below it.

**Interview one-liner:** A Spring Data repository is an interface extending `JpaRepository<Entity, Id>`; Spring auto-implements it, giving CRUD, paging, and sorting with zero code.

---

## 4. Query Methods — queries from method names (the cool part)

*Definition:* **Derived query methods** let you create a query just by **naming a method** in a certain pattern. Spring reads the method name and builds the SQL/JPQL automatically — no query written by you.

```java
interface UserRepository extends JpaRepository<User, Long> {

    User findByEmail(String email);
    List<User> findByName(String name);
    List<User> findByNameAndEmail(String name, String email);
    List<User> findByAgeGreaterThan(int age);
    List<User> findByNameContaining(String part);     // LIKE %part%
    List<User> findByNameOrderByAgeDesc(String name);
    long countByName(String name);
    boolean existsByEmail(String email);
    void deleteByEmail(String email);
}
```

### The naming grammar
```
findBy + FieldName + Keyword
   |         |          |
 action   property   condition (And, Or, GreaterThan, Like, Between, In, Containing...)
```

| Keyword | Generates |
|---------|-----------|
| `findByName` | `WHERE name = ?` |
| `findByAgeGreaterThan` | `WHERE age > ?` |
| `findByNameContaining` | `WHERE name LIKE %?%` |
| `findByNameAndEmail` | `WHERE name = ? AND email = ?` |
| `findByAgeBetween` | `WHERE age BETWEEN ? AND ?` |
| `OrderByAgeDesc` | `ORDER BY age DESC` |

> Powerful but careful: very long names get unreadable. For complex queries, use `@Query` instead.

**Interview one-liner:** Spring Data builds queries automatically from method names like `findByNameAndAgeGreaterThan` — no SQL needed for common cases.

---

## 5. @Query — write your own query

*Definition:* **`@Query`** lets you write a custom query (in **JPQL** — works on entities/objects, or native SQL) when method names aren't enough.

```java
interface UserRepository extends JpaRepository<User, Long> {

    // JPQL — uses entity & field names, not table & column names
    @Query("SELECT u FROM User u WHERE u.email = ?1")
    User findByEmailCustom(String email);

    // Named parameters (clearer)
    @Query("SELECT u FROM User u WHERE u.name = :name AND u.age > :age")
    List<User> search(@Param("name") String name, @Param("age") int age);

    // Native SQL — real table/column names
    @Query(value = "SELECT * FROM users WHERE email = ?1", nativeQuery = true)
    User findByEmailNative(String email);

    // Modifying query (UPDATE/DELETE) needs @Modifying + @Transactional
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :date")
    int deactivateOldUsers(@Param("date") LocalDate date);
}
```

| | JPQL | Native SQL |
|--|------|-----------|
| Works on | Entities & fields (`User`, `u.name`) | Tables & columns (`users`, `name`) |
| Database-independent? | ✅ Yes | ❌ No (DB-specific) |
| `nativeQuery=true`? | No | Yes |

> `@Modifying` is required for UPDATE/DELETE queries (otherwise Spring expects a SELECT). Wrap them in `@Transactional`.

**Interview one-liner:** `@Query` writes custom queries — JPQL (on entities, DB-independent) or native SQL (`nativeQuery=true`). Use `@Modifying` + `@Transactional` for UPDATE/DELETE.

---

## 6. Pagination & sorting (built-in)

```java
Page<User> findByName(String name, Pageable pageable);

// usage
Pageable page = PageRequest.of(0, 10, Sort.by("age").descending()); // page 0, size 10
Page<User> result = userRepo.findByName("Danish", page);
result.getContent();        // the 10 users
result.getTotalElements();  // total count
result.getTotalPages();
```
Pass a `Pageable`, get back a `Page` with data + metadata. No manual `LIMIT`/`OFFSET`.

**Interview one-liner:** Pass a `Pageable` (page number, size, sort) and return a `Page` — Spring Data handles `LIMIT`/`OFFSET` and gives total counts automatically.

---

## 7. Relationships (entity associations)

```java
@Entity
class Order {
    @ManyToOne                          // many orders → one user
    @JoinColumn(name = "user_id")
    private User user;
}

@Entity
class User {
    @OneToMany(mappedBy = "user")       // one user → many orders
    private List<Order> orders;
}
```

| Annotation | Meaning |
|-----------|---------|
| `@OneToOne` | One ↔ one (user ↔ profile) |
| `@OneToMany` | One → many (user → orders) |
| `@ManyToOne` | Many → one (orders → user) |
| `@ManyToMany` | Many ↔ many (students ↔ courses) |

> **Watch out for the N+1 problem:** lazily loading a list and then looping triggers one query per item. Fix with `@Query` + `JOIN FETCH` or an `@EntityGraph`. Common interview gotcha.

---

## Quick Revision Cheat Sheet

- **JPA** = spec, **Hibernate** = implementation, **Spring Data JPA** = boilerplate-remover on top.
- **@Entity** maps class→table; **@Id** + **@GeneratedValue** = primary key.
- **Repository** = interface extending **`JpaRepository<Entity, Id>`**; auto-gives CRUD, paging, sorting.
- Hierarchy: `CrudRepository` → `PagingAndSortingRepository` → `JpaRepository` (use this one).
- **Query methods**: `findByNameAndAgeGreaterThan` → query built from the method name.
- **@Query**: custom JPQL (entities, DB-independent) or native SQL (`nativeQuery=true`); `@Modifying`+`@Transactional` for UPDATE/DELETE.
- **Pagination**: pass `Pageable`, get `Page` (data + totals).
- **Relationships**: `@OneToOne/@OneToMany/@ManyToOne/@ManyToMany`; beware the **N+1 problem** (use JOIN FETCH).
