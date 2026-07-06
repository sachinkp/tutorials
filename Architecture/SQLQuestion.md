⭐ SQL Practice Question — Joins + Pagination + Aggregation

Scenario

You have three tables:
```
Users

id	name	email	


Orders

| id | user_id | total_amount | created_at |

OrderItems

| id | order_id | product_name | quantity | price |
```
---

Your Task

Write a SQL query to:

1. Fetch all orders for a given user (user_id = 42)
2. Include:• Order ID
• Order total amount
• Order created date
• Total number of items in the order
• Total value of items (sum of quantity × price)

3. Sort results by created_at descending
4. Return results using cursor-based pagination• Input: cursor_created_at
• Output: next 10 rows after that cursor

---

Here’s the full SQL solution, written exactly the way a senior engineer should present it in an interview — clean, correct, and optimized. I’ll also explain why each part is needed.

---

⭐ SQL Solution — Joins + Aggregation + Cursor-Based Pagination

SELECT
    o.id AS order_id,
    o.total_amount,
    o.created_at,
    COUNT(oi.id) AS total_items,
    SUM(oi.quantity * oi.price) AS total_value
FROM orders o
LEFT JOIN order_items oi
    ON oi.order_id = o.id
WHERE 
    o.user_id = 42
    AND o.created_at < :cursor_created_at
GROUP BY 
    o.id, o.total_amount, o.created_at
ORDER BY 
    o.created_at DESC
LIMIT 10;


---

🧩 Breakdown of the Solution

1. Join Orders → OrderItems

You need a join because each order can have multiple items.

LEFT JOIN order_items oi ON oi.order_id = o.id


• LEFT JOIN ensures orders with zero items still appear.


---

2. Filter by User

WHERE o.user_id = 42


This restricts results to the specific user.

---

3. Cursor-Based Pagination

AND o.created_at < :cursor_created_at


This returns the next page of results after the cursor.

Why cursor pagination?

• Faster than OFFSET
• Avoids skipping rows
• Works well with large datasets


---

4. Aggregation

COUNT(oi.id) AS total_items
SUM(oi.quantity * oi.price) AS total_value


This gives:

• Total number of items in the order
• Total value of the order


---

5. Grouping

GROUP BY o.id, o.total_amount, o.created_at


You must group by all non-aggregated fields.

---

6. Sorting

ORDER BY o.created_at DESC


Newest orders first.

---

7. Limit

LIMIT 10


Return only the next page.

---

⭐ Indexes You Should Mention in an Interview

To show senior-level thinking, mention these indexes:

CREATE INDEX idx_orders_userid_createdat 
ON orders(user_id, created_at DESC);

CREATE INDEX idx_orderitems_orderid 
ON order_items(order_id);


This makes the query fast, especially under load.

---

⭐ Want the next challenge?

I can give you:

• A harder SQL question (window functions, partitioning)
• A multi-tenant SQL question
• A query optimization challenge
• A live mock SQL interview


Choose one:
Hard SQL,
Multi-tenant SQL,
Query optimization, or
Mock interview.