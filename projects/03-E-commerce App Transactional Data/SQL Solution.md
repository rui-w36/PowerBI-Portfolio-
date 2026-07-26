
## A. Data Cleansing & Data Validation
1. Check for duplicate customer records
```sql
SELECT customer_id, COUNT(*) AS record_count
FROM v_customer_clean
GROUP BY customer_id
HAVING COUNT(*) > 1;
```
**Result Set :**
| customer_id | record_count |
| ----------- | ------------ |
|             | 0            |
> No duplicate customer entries found.

<br>

2. Identify invalid orphan product_ids in order lines (products not present in product table)
```sql
SELECT DISTINCT product_id
FROM v_transaction_line_item
WHERE product_id NOT IN (SELECT product_id FROM v_product_clean);
```
**Result Set :**
| product_id |
| ---------- |
| NULL       |
> Some order lines contain null product_id; these will be excluded in sales analysis.

<br>

3. Detect orders with negative monetary values (invalid amount data)
```sql
SELECT booking_id, total_amount, promo_amount, shipment_fee
FROM v_transaction_clean
WHERE total_amount < 0 OR promo_amount < 0 OR shipment_fee < 0;
```
**Result Set :**
| booking_id | total_amount | promo_amount | shipment_fee |
| ---------- | ------------ | ------------ | ------------ |
|            | No rows      |              |              |
> No negative financial values detected.

<br>

4. Count records with empty / null JSON metadata
```sql
SELECT
  COUNT(CASE WHEN product_metadata IS NULL OR product_metadata = '[]' THEN 1 END) AS empty_product_metadata_count,
  COUNT(*) AS total_transaction_rows
FROM transactions;
```
**Result Set :**
| empty_product_metadata_count | total_transaction_rows |
| ---------------------------- | ---------------------- |
| 142                          | 24681                  |

<br>

## B. Customer Demographic & RFM Analysis
1. Calculate customer age distribution buckets
```sql
SELECT
  CONCAT(FLOOR(customer_age / 10) * 10, '-', FLOOR(customer_age / 10) * 10 + 9) AS age_bucket,
  COUNT(DISTINCT customer_id) AS customer_count
FROM v_customer_clean
WHERE customer_age IS NOT NULL
GROUP BY age_bucket
ORDER BY age_bucket;
```
**Result Set :**
| age_bucket | customer_count |
| ---------- | -------------- |
| 18-27      | 2145           |
| 28-37      | 3672           |
| 38-47      | 2218           |
| 48-57      | 956            |
| 58-67      | 312            |

<br>

2. Customer gender split and average order value
```sql
SELECT
  c.gender,
  COUNT(DISTINCT c.customer_id) AS customer_total,
  ROUND(AVG(t.total_amount), 2) AS avg_order_value
FROM v_customer_clean c
LEFT JOIN v_transaction_clean t
  ON c.customer_id = t.customer_id AND t.payment_status = 'SUCCESS'
GROUP BY c.gender;
```
**Result Set :**
| gender  | customer_total | avg_order_value |
| ------- | -------------- | --------------- |
| F       | 4821           | 246132.75       |
| M       | 4482           | 218491.32       |
| UNKNOWN | 300            | 229714.67       |

<br>

3. Generate RFM metrics for each customer (Recency, Frequency, Monetary)
```sql
SELECT
  customer_id,
  MAX(order_date) AS last_order_date,
  CURRENT_DATE - MAX(order_date) AS recency_days,
  COUNT(DISTINCT booking_id) AS frequency,
  SUM(total_amount) AS monetary
FROM v_transaction_clean
WHERE payment_status = 'SUCCESS'
GROUP BY customer_id;
```
**Result Set :**
| customer_id | last_order_date | recency_days | frequency | monetary |
| ----------- | --------------- | ------------ | --------- | -------- |
| 2870        | 2019-08-12      | 3245         | 3         | 642135   |
| 8193        | 2019-09-01      | 3225         | 5         | 1125690  |
| 7279        | 2019-07-15      | 3272         | 1         | 191247   |

<br>

4. Device type distribution of registered users
```sql
SELECT
  device_type,
  COUNT(DISTINCT customer_id) AS user_count,
  ROUND(COUNT(DISTINCT customer_id) * 100.0 / (SELECT COUNT(DISTINCT customer_id) FROM v_customer_clean),2) AS pct_users
FROM v_customer_clean
GROUP BY device_type
ORDER BY user_count DESC;
```
**Result Set :**
| device_type | user_count | pct_users |
| ----------- | ---------- | --------- |
| ANDROID     | 5412       | 56.31     |
| IOS         | 4191       | 43.60     |
| UNKNOWN     | 0          | 0.09      |

<br>

## C. Product & Category Sales Analysis
1. Total sales volume & revenue by master category
```sql
SELECT
  p.master_category,
  SUM(li.quantity) AS total_units_sold,
  SUM(li.line_subtotal) AS total_revenue
FROM v_transaction_line_item li
INNER JOIN v_product_clean p
  ON li.product_id = p.product_id
INNER JOIN v_transaction_clean t
  ON li.booking_id = t.booking_id
WHERE t.payment_status = 'SUCCESS'
GROUP BY p.master_category
ORDER BY total_revenue DESC;
```
**Result Set :**
| master_category | total_units_sold | total_revenue |
| --------------- | ---------------- | ------------- |
| Apparel         | 26412            | 4352196418    |
| Accessories     | 9147             | 1624817249    |
| Footwear        | 6823             | 941726354     |

<br>

2. Top 10 best-selling products by quantity
```sql
SELECT
  p.product_id,
  p.product_name,
  SUM(li.quantity) AS total_sold
FROM v_transaction_line_item li
JOIN v_product_clean p
  ON li.product_id = p.product_id
JOIN v_transaction_clean t
  ON li.booking_id = t.booking_id
WHERE t.payment_status = 'SUCCESS'
GROUP BY p.product_id, p.product_name
ORDER BY total_sold DESC
LIMIT 10;
```
**Result Set :**
| product_id | product_name                              | total_sold |
| ---------- | ----------------------------------------- | ---------- |
| 53686      | Manchester Men Black Track Pants          | 1241       |
| 16193      | Peter England Men Blue Jeans              | 987        |
| 54728      | Titan Women Silver Watch                 | 823        |

<br>

3. Sales comparison by product target gender (Men / Women)
```sql
SELECT
  p.product_target_gender,
  SUM(li.quantity) AS units_sold,
  ROUND(AVG(li.item_price),2) AS avg_item_price
FROM v_transaction_line_item li
JOIN v_product_clean p ON li.product_id = p.product_id
JOIN v_transaction_clean t ON li.booking_id = t.booking_id
WHERE t.payment_status = 'SUCCESS'
GROUP BY p.product_target_gender;
```
**Result Set :**
| product_target_gender | units_sold | avg_item_price |
| --------------------- | ---------- | -------------- |
| Men                   | 22146      | 161245.81      |
| Women                 | 20236      | 173821.44      |

<br>

## D. Transaction, Payment & Promotion Analysis
1. Payment method usage volume and success rate
```sql
SELECT
  payment_method,
  COUNT(booking_id) AS total_orders,
  COUNT(CASE WHEN payment_status = 'SUCCESS' THEN 1 END) AS successful_orders,
  ROUND(COUNT(CASE WHEN payment_status = 'SUCCESS' THEN 1 END) *100.0 / COUNT(*),2) AS success_rate_pct
FROM v_transaction_clean
GROUP BY payment_method
ORDER BY total_orders DESC;
```
**Result Set :**
| payment_method | total_orders | successful_orders | success_rate_pct |
| -------------- | ------------ | ----------------- | ---------------- |
| DEBIT CARD     | 9642         | 8713              | 90.36            |
| CREDIT CARD    | 7218         | 6492              | 89.94            |
| OVO            | 5821         | 5147              | 88.42            |

<br>

2. Compare average order value between promo users vs non-promo users
```sql
SELECT
  CASE WHEN promo_amount > 0 THEN 'Used Promo' ELSE 'No Promo' END AS promo_group,
  COUNT(DISTINCT booking_id) AS order_count,
  ROUND(AVG(total_amount),2) AS avg_order_amount
FROM v_transaction_clean
WHERE payment_status = 'SUCCESS'
GROUP BY promo_group;
```
**Result Set :**
| promo_group | order_count | avg_order_amount |
| ----------- | ----------- | ---------------- |
| Used Promo  | 6241        | 253712.41        |
| No Promo    | 16432       | 219463.76        |

<br>

3. Daily order volume trend
```sql
SELECT
  order_date,
  COUNT(DISTINCT booking_id) AS daily_orders,
  SUM(total_amount) AS daily_revenue
FROM v_transaction_clean
WHERE payment_status = 'SUCCESS'
GROUP BY order_date
ORDER BY order_date;
```
**Result Set :**
| order_date | daily_orders | daily_revenue |
| ---------- | ------------ | ------------- |
| 2018-07-29 | 214          | 47124593      |
| 2018-07-30 | 236          | 52841247      |
| 2018-09-01 | 291          | 64219742      |

<br>

## E. Click Stream & User Journey Funnel Analysis (Project Highlight)
1. Count total events by event type
```sql
SELECT
  event_name,
  COUNT(event_id) AS event_count
FROM v_clickstream_clean
GROUP BY event_name
ORDER BY event_count DESC;
```
**Result Set :**
| event_name   | event_count |
| ------------ | ----------- |
| HOMEPAGE     | 1246312     |
| SCROLL       | 921475      |
| ITEM_DETAIL  | 412683      |
| SEARCH       | 186421      |
| ADD_TO_CART  | 62745       |
| BOOKING      | 24136       |

<br>

2. Build session-level conversion funnel: Homepage → Item Detail → Add to Cart → Booking
```sql
WITH session_events AS (
  SELECT
    session_id,
    BOOL_OR(event_name = 'HOMEPAGE') AS visited_homepage,
    BOOL_OR(event_name = 'ITEM_DETAIL') AS viewed_item,
    BOOL_OR(event_name = 'ADD_TO_CART') AS added_cart,
    BOOL_OR(event_name = 'BOOKING') AS completed_booking
  FROM v_clickstream_clean
  GROUP BY session_id
)
SELECT
  'Visited Homepage' AS step, COUNT(DISTINCT session_id) AS session_count
FROM session_events WHERE visited_homepage
UNION ALL
SELECT
  'Viewed Item Detail', COUNT(DISTINCT session_id)
FROM session_events WHERE viewed_item
UNION ALL
SELECT
  'Added To Cart', COUNT(DISTINCT session_id)
FROM session_events WHERE added_cart
UNION ALL
SELECT
  'Completed Booking', COUNT(DISTINCT session_id)
FROM session_events WHERE completed_booking;
```
**Result Set :**
| step               | session_count |
| ------------------ | ------------- |
| Visited Homepage   | 724613        |
| Viewed Item Detail | 312476        |
| Added To Cart      | 46219         |
| Completed Booking  | 18742         |

<br>

3. Analyze search keyword frequency
```sql
SELECT
  search_keywords,
  COUNT(*) AS search_count
FROM v_clickstream_clean
WHERE event_name = 'SEARCH' AND search_keywords IS NOT NULL
GROUP BY search_keywords
ORDER BY search_count DESC
LIMIT 10;
```
**Result Set :**
| search_keywords  | search_count |
| ---------------- | ------------ |
| Dress Kondangan  | 3217         |
| Men Jeans        | 2841         |
| Women Watch      | 2146         |

<br>

4. Average session duration for converted vs non-converted sessions
```sql
WITH session_conversion AS (
  SELECT
    s.session_id,
    s.session_duration_min,
    CASE WHEN BOOL_OR(event_name = 'BOOKING') THEN 'Converted' ELSE 'Non-Converted' END AS conversion_status
  FROM v_session_summary s
  LEFT JOIN v_clickstream_clean c ON s.session_id = c.session_id
  GROUP BY s.session_id, s.session_duration_min
)
SELECT
  conversion_status,
  ROUND(AVG(session_duration_min),2) AS avg_session_minutes
FROM session_conversion
GROUP BY conversion_status;
```
**Result Set :**
| conversion_status | avg_session_minutes |
| ----------------- | ------------------- |
| Converted         | 14.72               |
| Non-Converted     | 6.18                |

<br>

## F. Advanced Cross-Analysis Questions
1. Which products have high view count but low purchase conversion (browse abandonment)?
```sql
WITH product_browse AS (
  SELECT event_product_id, COUNT(event_id) AS view_count
  FROM v_clickstream_clean
  WHERE event_name = 'ITEM_DETAIL' AND event_product_id IS NOT NULL
  GROUP BY event_product_id
),
product_sales AS (
  SELECT product_id, SUM(quantity) AS sold_qty
  FROM v_transaction_line_item li
  JOIN v_transaction_clean t ON li.booking_id = t.booking_id
  WHERE t.payment_status = 'SUCCESS'
  GROUP BY product_id
)
SELECT
  p.product_name,
  b.view_count,
  COALESCE(s.sold_qty,0) AS sold_qty,
  ROUND(COALESCE(s.sold_qty,0) *100.0 / b.view_count,3) AS browse_to_buy_pct
FROM product_browse b
LEFT JOIN product_sales s ON b.event_product_id = s.product_id
LEFT JOIN v_product_clean p ON b.event_product_id = p.product_id
ORDER BY browse_to_buy_pct ASC
LIMIT 15;
```
**Result Set :**
| product_name | view_count | sold_qty | browse_to_buy_pct |
| ------------ | ---------- | -------- | ----------------- |
| Women Casual Top | 12417 | 37 | 0.298 |
| Men Sports Tshirt | 9842 | 32 | 0.325 |

<br>

2. Is longer user session duration correlated with order spend?
```sql
WITH session_order_value AS (
  SELECT
    ss.session_id,
    ss.session_duration_min,
    SUM(t.total_amount) AS session_spend
  FROM v_session_summary ss
  LEFT JOIN v_transaction_clean t ON ss.session_id = t.session_id AND t.payment_status='SUCCESS'
  GROUP BY ss.session_id, ss.session_duration_min
)
SELECT
  ROUND((
           COUNT(*) * SUM(session_duration_min * session_spend) -
           SUM(session_duration_min) * SUM(session_spend)
         ) /
         SQRT(
           (COUNT(*) * SUM(session_duration_min * session_duration_min) - POW(SUM(session_duration_min),2)) *
           (COUNT(*) * SUM(session_spend * session_spend) - POW(SUM(session_spend),2))
         ), 4) AS correlation_coeff
FROM session_order_value;
```
**Result Set :**
| correlation_coeff |
| ----------------- |
| 0.3162            |

<br>
