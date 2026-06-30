# Spring Data JPA Sorting & Pagination - Interview Guide 2025-2026

This guide contains the most frequently asked Spring Data JPA interview questions focused on Pagination and Sorting, tailored for both freshers and experienced (2-5 years) developers.

## 🟢 Beginner Level (Freshers)

**1. What is pagination?**
Pagination is the process of dividing a large dataset into smaller, manageable chunks (pages) to improve performance and user experience, rather than loading thousands of records at once.

**2. What is sorting?**
Sorting is the process of ordering the dataset based on one or more properties (columns) in either Ascending (ASC) or Descending (DESC) order.

**3. What is `Pageable` in Spring Data JPA?**
`Pageable` is an interface in Spring Data that encapsulates pagination and sorting information (like page number, page size, and sort direction). It is passed to repository methods to fetch a specific page of data.

**4. What is `Page`?**
`Page` is a sub-interface of `Slice`. It represents a chunk of data but also contains metadata about the entire dataset, such as `getTotalElements()` (total records) and `getTotalPages()` (total pages). It triggers a `COUNT` query under the hood.

**5. What is `Slice`?**
`Slice` is similar to `Page` but without the total count metadata. It only knows if there is a `hasNext()` page. It is much faster than `Page` because it avoids executing the expensive `COUNT` query on the database.

**6. What is `PageRequest`?**
`PageRequest` is a concrete implementation of the `Pageable` interface. We use its static factory method `PageRequest.of(page, size, sort)` to create a `Pageable` object.

**7. What is the default page index in Spring Data JPA?**
The default page index is **0** (Zero-based pagination). So page 1 is index 0.

**8. How do you implement pagination in Spring Boot?**
By accepting a `Pageable` object or `page` and `size` request parameters in the Controller, creating a `PageRequest` object, and passing it to a repository method that extends `JpaRepository` or `PagingAndSortingRepository`.

**9. How do you sort ascending / descending?**
```java
Sort.by("price").ascending(); // OR Sort.by(Sort.Direction.ASC, "price")
Sort.by("price").descending(); // OR Sort.by(Sort.Direction.DESC, "price")
```

**10. What is `JpaRepository` vs `PagingAndSortingRepository`?**
`PagingAndSortingRepository` provides methods to do pagination and sorting (`findAll(Pageable)` and `findAll(Sort)`). `JpaRepository` extends `PagingAndSortingRepository` and adds JPA-specific methods like `flush()`, `saveAllAndFlush()`, etc.

**11. How does Spring bind `page`, `size`, and `sort` parameters?**
Spring Boot's web support automatically detects query parameters like `?page=0&size=10&sort=name,asc` and injects a fully populated `Pageable` object directly into the Controller method parameter.

**12. What is `@PageableDefault`?**
It is an annotation used in the Controller to provide default pagination settings if the client doesn't send any parameters. Example: `@PageableDefault(size = 20, page = 0, sort = "id") Pageable pageable`

---

## 🟡 Intermediate Level (2-5 Years Experience)

**13. How do you combine sorting and paging?**
By passing a `Sort` object inside the `PageRequest.of()` method:
`PageRequest.of(pageNumber, pageSize, Sort.by("price").descending());`

**14. How do you do multi-column sorting?**
By chaining `Sort` objects using `.and()`:
`Sort.by("price").descending().and(Sort.by("title").ascending());`

**15. Why is `Page` slower than `Slice`?**
To calculate total pages and total elements, `Page` executes a secondary `SELECT COUNT(*)` query. For massive tables (millions of rows), this count query becomes a major performance bottleneck. `Slice` fetches `size + 1` records to check if a next page exists, avoiding the count query.

**16. How do you add pagination to a custom Derived Query?**
Simply append `Pageable pageable` as the last parameter of your method signature. Spring Data JPA automatically handles the rest.
`List<Product> findByTitleContaining(String title, Pageable pageable);`

**17. What is `countQuery` used for in `@Query`?**
When writing complex custom JPQL or Native queries with `@Query`, Spring might fail to automatically generate the `COUNT` query for pagination. We explicitly provide it using `countQuery`.
```java
@Query(value = "SELECT p FROM Product p WHERE p.status = ?1", 
       countQuery = "SELECT count(p) FROM Product p WHERE p.status = ?1")
Page<Product> findByStatus(String status, Pageable pageable);
```

**18. How do you sort by nested properties?**
You use property path dot-notation. For example, if a `User` has an `Address`, and `Address` has a `city`, you can sort by: `Sort.by("address.city")`.

**19. Why should we avoid returning Entities in paginated APIs?**
Returning entities directly can lead to `LazyInitializationException`, unwanted data exposure (passwords), and recursive JSON serialization issues. Always map `Page<Entity>` to `Page<DTO>`.

**20. How do you handle custom page response DTOs?**
Using the `.map()` method provided by the `Page` interface.
```java
Page<Product> productPage = repository.findAll(pageable);
Page<ProductDTO> dtoPage = productPage.map(product -> new ProductDTO(product));
```

**21. What happens if the page size is too large?**
It can cause OutOfMemory (OOM) errors and slow down the database. Best practice is to set a global maximum page size in `application.properties`:
`spring.data.web.pageable.max-page-size=100`

---

## 🔴 Advanced Level & Tricky Scenarios

**22. What is Offset Pagination and why does it become slow on large tables?**
Offset pagination uses SQL `LIMIT` and `OFFSET`. If you request page 500 (`OFFSET 5000`), the database still has to scan and discard the first 5000 rows before returning the next 10. This makes deep pagination extremely slow (O(N) time complexity).

**23. When should you prefer Cursor Pagination over Offset Pagination?**
For infinite scrolling features or massive datasets (like social media feeds) where deep pagination is required. Cursor pagination uses a unique identifier (like the last seen `id` or `timestamp`) to fetch the next set of records (`WHERE id > last_id LIMIT 10`).

**24. Why can pagination break with `JOIN FETCH`?**
Using `JOIN FETCH` on a One-To-Many collection along with pagination causes Spring Data to issue a warning: *"firstResult/maxResults specified with collection fetch; applying in memory!"*. Hibernate fetches ALL records into memory and paginates them in RAM, causing severe memory crashes.

**25. How do you prevent unstable pagination (duplicates across pages)?**
If you sort by a column that has duplicate values (e.g., `price`), SQL doesn't guarantee the order of ties. A row might appear on Page 1 and again on Page 2. 
**Fix:** Always add a unique, deterministic tie-breaker column (like `id`) to your sort: `ORDER BY price DESC, id ASC`.

**26. Can you use `Pageable` with Native SQL Queries?**
Yes, but you MUST provide a explicit `countQuery` because Spring Data often cannot parse complex native SQL to derive the count query automatically.
```java
@Query(value = "SELECT * FROM products WHERE status = :status", 
       countQuery = "SELECT count(*) FROM products WHERE status = :status", 
       nativeQuery = true)
Page<Product> findByStatusNative(@Param("status") String status, Pageable pageable);
```

**27. A frontend sends `page=1` for the first page, but your API uses zero-based pages. How to fix this globally?**
You can configure Spring Boot to accept 1-based pagination by adding this to `application.properties`:
`spring.data.web.pageable.one-indexed-parameters=true`

**28. How do you handle pagination with dynamic Filtering + Sorting?**
For dynamic queries (where filters change based on user input), using `@Query` is difficult. Instead, use **JPA Specifications** or **QueryDSL**.
`Page<Product> findAll(Specification<Product> spec, Pageable pageable);`

**29. What is `Sort.unsorted()`?**
It returns an empty `Sort` instance, meaning the data will be returned in whatever order the database defaults to (usually insertion order, but not guaranteed).

**30. How do you implement custom pagination for aggregated data?**
When writing complex `GROUP BY` aggregations, return the results as a `List` or use `Slice` instead of `Page`, because generating a generic `countQuery` for grouped data is often inaccurate and highly inefficient.

---
*Created dynamically for quick interview preparation and revision.*
