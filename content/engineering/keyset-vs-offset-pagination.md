+++
title = "Keyset Pagination vs OFFSET, or Why Your Page 500 Takes 10 Seconds"
date = 2025-06-20
[taxonomies]
tags = ["database", "jpa", "performance"]
+++

I once watched a monitoring dashboard crawl to a halt because someone was paginating through a 2-million-row table using OFFSET. Page 1 was fast. Page 100 was tolerable. Page 5000 was a solid 8-second query. That was the day I learned that OFFSET pagination is a performance lie.

## How OFFSET Works (And Why It's Slow)

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
```

What you think happens: the database jumps to row 10,000 and reads 20 rows.

What actually happens: the database reads rows 1 through 10,020, discards the first 10,000, and returns the last 20. Every row before your offset is fetched and thrown away. The deeper you paginate, the more work the database does for nothing.

At page 1 (OFFSET 0), it reads 20 rows. At page 500 (OFFSET 10,000), it reads 10,020 rows. At page 50,000 (OFFSET 1,000,000), it reads 1,000,020 rows. Linear degradation. Predictable. Painful.

## The Spring Data Default

Spring Data JPA's `Pageable` uses OFFSET pagination by default:

```java
Page<Order> orders = orderRepository.findAll(PageRequest.of(500, 20));
```

This generates `LIMIT 20 OFFSET 10000`. It works. It's simple. And for the first few hundred pages, it's fine. But if your users (or your API consumers, or your batch jobs) go deep, performance tanks.

The `Page` return type also runs a `COUNT(*)` query to know the total number of pages. That's a second expensive query on top of the already expensive offset query. Use `Slice` instead if you don't need the total count.

## The SQL SEEK Method (Keyset Pagination)

Keyset pagination doesn't skip rows. It uses a WHERE clause to start from where the previous page left off:

```sql
-- First page
SELECT * FROM orders ORDER BY created_at DESC, id DESC LIMIT 20;

-- Next page (using the last row's values as the cursor)
SELECT * FROM orders
WHERE (created_at, id) < ('2025-06-15 10:30:00', 'order-abc-123')
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

The database uses the index on `(created_at, id)` to jump directly to the right position. No scanning, no discarding. Page 1 and page 50,000 have the same performance.

The trade-off: you can't jump to "page 347." You can only go forward and backward from a known position. This means keyset pagination works for infinite scrolling, "load more" buttons, and cursor-based APIs. It doesn't work for "click on page number 42."

## Implementing Keyset Pagination in Spring Data

Spring Data doesn't have built-in keyset pagination (as of Spring Data 3.x, `ScrollPosition` support is emerging but limited). Here's how I implement it:

### The Cursor

```java
public record OrderCursor(Instant createdAt, String id) {
    public static OrderCursor from(Order order) {
        return new OrderCursor(order.getCreatedAt(), order.getId());
    }

    public String encode() {
        return Base64.getEncoder().encodeToString(
            (createdAt.toEpochMilli() + ":" + id).getBytes()
        );
    }

    public static OrderCursor decode(String encoded) {
        String decoded = new String(Base64.getDecoder().decode(encoded));
        String[] parts = decoded.split(":");
        return new OrderCursor(Instant.ofEpochMilli(Long.parseLong(parts[0])), parts[1]);
    }
}
```

The cursor encodes the values of the sort columns from the last row of the current page. It's opaque to the client - they just pass it back to get the next page.

### The Repository

```java
@Query("""
    SELECT o FROM Order o
    WHERE (o.createdAt < :createdAt)
       OR (o.createdAt = :createdAt AND o.id < :id)
    ORDER BY o.createdAt DESC, o.id DESC
    """)
List<Order> findNextPage(
    @Param("createdAt") Instant createdAt,
    @Param("id") String id,
    Pageable pageable
);
```

The WHERE clause is the key. It implements the row-value comparison `(created_at, id) < (?, ?)`. I break it into individual comparisons because JPQL doesn't support tuple comparison syntax. Some databases (PostgreSQL) support it natively in SQL; if you're using native queries, use it.

### The Controller

```java
@GetMapping("/orders")
public CursorPage<OrderDto> listOrders(
    @RequestParam(required = false) String cursor,
    @RequestParam(defaultValue = "20") int size
) {
    List<Order> orders;
    if (cursor == null) {
        orders = orderRepository.findFirstPage(PageRequest.of(0, size + 1));
    } else {
        OrderCursor c = OrderCursor.decode(cursor);
        orders = orderRepository.findNextPage(c.createdAt(), c.id(), PageRequest.of(0, size + 1));
    }

    boolean hasMore = orders.size() > size;
    if (hasMore) {
        orders = orders.subList(0, size);
    }

    String nextCursor = hasMore ? OrderCursor.from(orders.getLast()).encode() : null;
    return new CursorPage<>(orders.stream().map(OrderDto::from).toList(), nextCursor);
}
```

The trick of fetching `size + 1` rows tells you if there's a next page without running a COUNT query. If you get 21 rows when you asked for 20, there's more. Return 20 and provide a cursor to the next page.

## The Sort Column Matters

Keyset pagination requires a deterministic sort order. If your sort column has duplicates (e.g., multiple orders with the same `created_at`), you'll skip or duplicate rows when paginating.

The fix: always include a unique column as a tiebreaker. Sort by `(created_at DESC, id DESC)`, not just `(created_at DESC)`. The unique column ensures every row has a distinct position in the sort order.

And make sure you have a composite index on the sort columns:

```sql
CREATE INDEX idx_orders_cursor ON orders (created_at DESC, id DESC);
```

Without this index, the database can't efficiently seek to the cursor position, and you lose the performance advantage.

## When to Use Which

**OFFSET pagination:**
- Admin panels where you need "page X of Y" navigation
- Small datasets (under 10,000 rows)
- Datasets that rarely change during pagination

**Keyset pagination:**
- APIs consumed by mobile apps or SPAs (infinite scroll)
- Large datasets
- Real-time feeds where rows are constantly added
- Batch processing (iterate through all records efficiently)

## The Hybrid Approach

Some applications use keyset pagination for the data query and a separate cached COUNT for the total:

```java
// Cache the total count (refresh every 5 minutes)
@Cacheable(value = "orderCount", key = "#status")
public long countByStatus(OrderStatus status) {
    return orderRepository.countByStatus(status);
}
```

The count is approximate but avoids the expensive COUNT query on every page request. This works well when users want to know "roughly how many results" without needing exact numbers.

## My Default

For any new API, I default to keyset pagination. It's more work to implement initially, but it scales. I've never had to go back and replace keyset with offset. I've replaced offset with keyset three times.
