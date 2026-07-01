# 🛒 TradeZone E-commerce Analytics
## End-to-End SQL Data Analysis for a Fast-Growing Nigerian E-commerce Platform

> A complete data analytics engagement covering **data cleaning**, **SQL business analysis**, and **executive memo writing** for TradeZone — a Nigerian e-commerce platform operating across Lagos, Abuja, Kano, Port Harcourt, and Ibadan. Built entirely in **PostgreSQL**.

---

## 📌 Table of Contents

- [Project Overview](#project-overview)
- [Business Context](#business-context)
- [Dataset Description](#dataset-description)
- [Tools Used](#tools-used)
- [Stage 1 — Data Cleaning](#stage-1--data-cleaning)
- [Stage 2 — SQL Business Analysis](#stage-2--sql-business-analysis)
  - [Query 1 — Customer Acquisition & 30-Day Conversion](#query-1--customer-acquisition--30-day-conversion)
  - [Query 2 — Product Revenue Performance](#query-2--product-revenue-performance)
  - [Query 3 — Seller Fulfilment Efficiency](#query-3--seller-fulfilment-efficiency)
  - [Query 4 — Quarterly Revenue Trends](#query-4--quarterly-revenue-trends)
  - [Query 5 — Customer Spend Segmentation](#query-5--customer-spend-segmentation)
  - [Query 6 — Payment Method Preferences by State](#query-6--payment-method-preferences-by-state)
  - [Query 7 — Review Ratings vs Sales Performance](#query-7--review-ratings-vs-sales-performance)
  - [Query 8 — Top Seller Bonus Qualification](#query-8--top-seller-bonus-qualification)
- [Stage 3 — Analyst Memo](#stage-3--analyst-memo)
- [Key Findings & Insights](#key-findings--insights)
- [Data Gaps & Recommendations](#data-gaps--recommendations)
- [Business Recommendations](#business-recommendations)
- [Project Structure](#project-structure)
- [How to Reproduce This Analysis](#how-to-reproduce-this-analysis)
- [Author](#-author)

---

## Project Overview

This project simulates a real-world data analytics engagement for **TradeZone**, a fast-growing Nigerian e-commerce platform. The work was structured across three professional stages:

```
Stage 1: Data Cleaning
    → Standardise, validate, and prepare raw messy data for analysis

Stage 2: SQL Business Analysis
    → Answer 8 business questions using PostgreSQL queries

Stage 3: Analyst Memo
    → Translate SQL findings into executive-ready recommendations
       for the Head of Growth and Head of Seller Operations
```

The goal was not to produce charts — it was to produce **decisions**. Each query was designed to answer a question that a real leadership team would need answered before their next planning cycle.

---

## Business Context

TradeZone operates as a multi-seller e-commerce marketplace serving customers across **5 Nigerian states**:

| State | City |
|---|---|
| Lagos | Lagos |
| FCT | Abuja |
| Kano | Kano |
| Rivers | Port Harcourt |
| Oyo | Ibadan |

The platform connects **customers**, **sellers**, and **products** through an order management system that tracks purchases, payments, fulfilment, and reviews. At the time of this analysis, TradeZone's leadership needed clarity on:

- Why customers acquired weren't converting to buyers
- Which products and sellers were driving revenue — and the risk this concentration created
- Whether the review system was influencing purchasing behaviour
- How efficiently sellers were fulfilling orders
- Which sellers qualified for performance bonuses

---

## Dataset Description

The raw dataset was a real-world messy file requiring significant cleaning before analysis. The schema covers the core entities of an e-commerce marketplace:

### Core Tables

| Table | Description | Key Fields |
|---|---|---|
| `customers` | Customer registration records | `customer_id`, `name`, `email`, `state`, `city`, `registration_date` |
| `orders` | All orders placed on the platform | `order_id`, `customer_id`, `seller_id`, `order_date`, `order_status`, `payment_method`, `total_amount` |
| `order_items` | Line-item breakdown per order | `item_id`, `order_id`, `product_id`, `quantity`, `unit_price`, `subtotal` |
| `products` | Product catalogue | `product_id`, `product_name`, `category`, `price`, `seller_id` |
| `sellers` | Seller accounts | `seller_id`, `seller_name`, `state`, `registration_date` |
| `reviews` | Customer product reviews | `review_id`, `product_id`, `customer_id`, `rating`, `review_date` |
| `payments` | Payment transaction records | `payment_id`, `order_id`, `payment_method`, `payment_status`, `payment_date` |

### Data Quality Issues Found (Pre-Cleaning)

| Issue Type | Description |
|---|---|
| Inconsistent city names | e.g. `"lagos"`, `"LAGOS"`, `"Lag0s"` all representing Lagos |
| Non-standard product categories | Inconsistent casing and naming conventions across categories |
| NULL values | Present in critical fields including `order_status`, `payment_method`, `total_amount` |
| Duplicate records | Duplicate orders and customer registrations found in raw data |
| Amount validation failures | `total_amount` on orders did not always match sum of `order_items.subtotal` |

---

## Tools Used

| Tool | Purpose |
|---|---|
| **PostgreSQL** | All data cleaning, transformation, and business analysis |
| **pgAdmin / psql** | Query execution environment |
| **Google Drive** | Project file hosting and delivery |
| **Markdown** | Documentation and analyst memo formatting |

---

## Stage 1 — Data Cleaning

All cleaning was performed in PostgreSQL before any analysis queries were run. The cleaned dataset was exported as a SQL dump for reproducibility.

### 1.1 Standardise City Names

```sql
-- Normalise all city name variations to canonical names
UPDATE customers
SET city = CASE
    WHEN LOWER(TRIM(city)) IN ('lagos', 'lag0s', 'lagoss', 'laagos') THEN 'Lagos'
    WHEN LOWER(TRIM(city)) IN ('abuja', 'abujah', 'abj') THEN 'Abuja'
    WHEN LOWER(TRIM(city)) IN ('kano', 'kano state') THEN 'Kano'
    WHEN LOWER(TRIM(city)) IN ('port harcourt', 'portharcourt', 'ph', 'port-harcourt') THEN 'Port Harcourt'
    WHEN LOWER(TRIM(city)) IN ('ibadan', 'ibadaan', 'ibadn') THEN 'Ibadan'
    ELSE city
END;

-- Verify: should return 0 rows after cleaning
SELECT city, COUNT(*) 
FROM customers 
WHERE city NOT IN ('Lagos', 'Abuja', 'Kano', 'Port Harcourt', 'Ibadan')
GROUP BY city;
```

### 1.2 Normalise Product Categories

```sql
-- Standardise category naming conventions
UPDATE products
SET category = INITCAP(LOWER(TRIM(category)));

-- Remove extra whitespace from category strings
UPDATE products
SET category = REGEXP_REPLACE(category, '\s+', ' ', 'g');

-- Review distinct categories post-cleaning
SELECT DISTINCT category 
FROM products 
ORDER BY category;
```

### 1.3 Resolve NULL Values

```sql
-- Flag and handle NULLs in order_status
UPDATE orders
SET order_status = 'unknown'
WHERE order_status IS NULL;

-- Flag NULLs in payment_method
UPDATE payments
SET payment_method = 'not_recorded'
WHERE payment_method IS NULL;

-- Check for NULL total_amount
SELECT order_id, total_amount
FROM orders
WHERE total_amount IS NULL;

-- Set NULL total_amount from order_items where possible
UPDATE orders o
SET total_amount = (
    SELECT SUM(subtotal)
    FROM order_items oi
    WHERE oi.order_id = o.order_id
)
WHERE o.total_amount IS NULL
  AND EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.order_id = o.order_id
  );
```

### 1.4 Remove Duplicate Records

```sql
-- Identify duplicate orders (same customer, same date, same amount)
WITH duplicates AS (
    SELECT
        order_id,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id, order_date, total_amount
            ORDER BY order_id
        ) AS rn
    FROM orders
)
DELETE FROM orders
WHERE order_id IN (
    SELECT order_id FROM duplicates WHERE rn > 1
);

-- Identify duplicate customer registrations (same email)
WITH dup_customers AS (
    SELECT
        customer_id,
        ROW_NUMBER() OVER (
            PARTITION BY LOWER(email)
            ORDER BY registration_date ASC
        ) AS rn
    FROM customers
)
DELETE FROM customers
WHERE customer_id IN (
    SELECT customer_id FROM dup_customers WHERE rn > 1
);
```

### 1.5 Validate Order Amounts Against Line Items

```sql
-- Find orders where total_amount does not match sum of line items
SELECT
    o.order_id,
    o.total_amount AS recorded_total,
    ROUND(SUM(oi.subtotal)::NUMERIC, 2) AS calculated_total,
    ROUND((o.total_amount - SUM(oi.subtotal))::NUMERIC, 2) AS discrepancy
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.total_amount
HAVING ROUND(o.total_amount::NUMERIC, 2) <> ROUND(SUM(oi.subtotal)::NUMERIC, 2)
ORDER BY ABS(o.total_amount - SUM(oi.subtotal)) DESC;

-- Correct discrepancies by overwriting with calculated total
UPDATE orders o
SET total_amount = sub.calculated_total
FROM (
    SELECT order_id, ROUND(SUM(subtotal)::NUMERIC, 2) AS calculated_total
    FROM order_items
    GROUP BY order_id
) sub
WHERE o.order_id = sub.order_id
  AND ROUND(o.total_amount::NUMERIC, 2) <> sub.calculated_total;
```

### Cleaning Summary

| Step | Records Affected |
|---|---|
| City name standardisation | Multiple variants resolved across 5 cities |
| Category normalisation | All product categories standardised to Title Case |
| NULL order_status resolved | Flagged as 'unknown' |
| NULL payment_method resolved | Flagged as 'not_recorded' |
| NULL total_amount recovered | Recalculated from order_items |
| Duplicate orders removed | Based on customer_id + date + amount match |
| Duplicate customers removed | Based on email deduplication (earliest kept) |
| Amount validation corrections | total_amount overwritten where discrepancy found |

---

## Stage 2 — SQL Business Analysis

Eight business queries were written to answer the core questions facing TradeZone's leadership team.

---

### Query 1 — Customer Acquisition & 30-Day Conversion

**Business Question:** Of all customers who registered on the platform, what percentage made their first purchase within 30 days — and how does this vary by state?

```sql
WITH first_orders AS (
    SELECT
        c.customer_id,
        c.state,
        c.registration_date,
        MIN(o.order_date) AS first_order_date
    FROM customers c
    LEFT JOIN orders o
        ON c.customer_id = o.customer_id
        AND o.order_status NOT IN ('cancelled', 'unknown')
    GROUP BY c.customer_id, c.state, c.registration_date
),
conversion AS (
    SELECT
        customer_id,
        state,
        registration_date,
        first_order_date,
        CASE
            WHEN first_order_date IS NOT NULL
             AND first_order_date <= registration_date + INTERVAL '30 days'
            THEN 1 ELSE 0
        END AS converted_in_30_days
    FROM first_orders
)
SELECT
    state,
    COUNT(customer_id)                                      AS total_customers,
    SUM(converted_in_30_days)                               AS converted,
    COUNT(customer_id) - SUM(converted_in_30_days)          AS not_converted,
    ROUND(
        SUM(converted_in_30_days)::NUMERIC
        / COUNT(customer_id) * 100, 2
    )                                                       AS conversion_rate_pct
FROM conversion
GROUP BY state
ORDER BY conversion_rate_pct DESC;
```

**Key Finding:** Conversion within the first 30 days was **significantly below 50% in several states** — meaning TradeZone was spending to acquire customers and losing them before a single purchase was recorded.

---

### Query 2 — Product Revenue Performance

**Business Question:** Which products are generating the most revenue, and how concentrated is revenue across the product catalogue?

```sql
SELECT
    p.product_id,
    p.product_name,
    p.category,
    COUNT(DISTINCT oi.order_id)                             AS total_orders,
    SUM(oi.quantity)                                        AS total_units_sold,
    ROUND(SUM(oi.subtotal)::NUMERIC, 2)                    AS total_revenue,
    ROUND(AVG(oi.unit_price)::NUMERIC, 2)                  AS avg_unit_price,
    ROUND(
        SUM(oi.subtotal)::NUMERIC
        / SUM(SUM(oi.subtotal)) OVER () * 100, 2
    )                                                       AS revenue_share_pct,
    ROUND(
        SUM(SUM(oi.subtotal)::NUMERIC) OVER (
            ORDER BY SUM(oi.subtotal) DESC
        ) / SUM(SUM(oi.subtotal)) OVER () * 100, 2
    )                                                       AS cumulative_revenue_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_status NOT IN ('cancelled', 'unknown')
GROUP BY p.product_id, p.product_name, p.category
ORDER BY total_revenue DESC;
```

**Key Finding:** Revenue was **heavily concentrated in a narrow set of products** — a Pareto-like distribution where a small number of SKUs drove a disproportionate share of total revenue. The loss or disruption of any top product would have an immediately visible impact.

---

### Query 3 — Seller Fulfilment Efficiency

**Business Question:** How efficiently are sellers fulfilling orders, and which sellers have the highest rates of delayed, cancelled, or undelivered orders?

```sql
WITH seller_fulfilment AS (
    SELECT
        s.seller_id,
        s.seller_name,
        s.state,
        COUNT(DISTINCT o.order_id)                          AS total_orders,
        SUM(CASE WHEN o.order_status = 'delivered' THEN 1 ELSE 0 END)   AS delivered,
        SUM(CASE WHEN o.order_status = 'cancelled' THEN 1 ELSE 0 END)   AS cancelled,
        SUM(CASE WHEN o.order_status = 'pending' THEN 1 ELSE 0 END)     AS pending,
        SUM(CASE WHEN o.order_status = 'returned' THEN 1 ELSE 0 END)    AS returned,
        ROUND(SUM(oi.subtotal)::NUMERIC, 2)                AS total_revenue
    FROM sellers s
    JOIN orders o ON s.seller_id = o.seller_id
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY s.seller_id, s.seller_name, s.state
)
SELECT
    *,
    ROUND(delivered::NUMERIC / NULLIF(total_orders, 0) * 100, 2) AS delivery_rate_pct,
    ROUND(cancelled::NUMERIC / NULLIF(total_orders, 0) * 100, 2) AS cancellation_rate_pct
FROM seller_fulfilment
ORDER BY delivery_rate_pct DESC, total_orders DESC;
```

**Key Finding:** Clear performance spread across sellers in delivery rates and cancellation rates — some sellers operate with significantly higher fulfilment reliability than others, directly affecting customer experience and retention.

---

### Query 4 — Quarterly Revenue Trends

**Business Question:** How is revenue trending quarter by quarter, and where are the growth or decline periods?

```sql
SELECT
    EXTRACT(YEAR FROM o.order_date)                         AS order_year,
    EXTRACT(QUARTER FROM o.order_date)                      AS order_quarter,
    'Q' || EXTRACT(QUARTER FROM o.order_date)::TEXT
        || ' ' || EXTRACT(YEAR FROM o.order_date)::TEXT    AS quarter_label,
    COUNT(DISTINCT o.order_id)                              AS total_orders,
    ROUND(SUM(oi.subtotal)::NUMERIC, 2)                    AS total_revenue,
    ROUND(AVG(oi.subtotal)::NUMERIC, 2)                    AS avg_order_value,
    ROUND(
        (SUM(oi.subtotal) - LAG(SUM(oi.subtotal)) OVER (ORDER BY
            EXTRACT(YEAR FROM o.order_date),
            EXTRACT(QUARTER FROM o.order_date)
        )) / NULLIF(LAG(SUM(oi.subtotal)) OVER (ORDER BY
            EXTRACT(YEAR FROM o.order_date),
            EXTRACT(QUARTER FROM o.order_date)
        ), 0) * 100, 2
    )                                                       AS qoq_growth_pct
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_status NOT IN ('cancelled', 'unknown')
GROUP BY
    EXTRACT(YEAR FROM o.order_date),
    EXTRACT(QUARTER FROM o.order_date)
ORDER BY order_year, order_quarter;
```

**Key Finding:** Quarter-over-quarter growth reveals the platform's revenue trajectory and highlights which periods saw demand spikes or drops — critical for planning inventory, marketing spend, and seller support.

---

### Query 5 — Customer Spend Segmentation

**Business Question:** How are customers distributed by total lifetime spend, and what does the high-value customer segment look like?

```sql
WITH customer_spend AS (
    SELECT
        c.customer_id,
        c.name,
        c.state,
        c.city,
        COUNT(DISTINCT o.order_id)                          AS total_orders,
        ROUND(SUM(oi.subtotal)::NUMERIC, 2)                AS lifetime_spend,
        ROUND(AVG(oi.subtotal)::NUMERIC, 2)                AS avg_order_value,
        MIN(o.order_date)                                   AS first_order_date,
        MAX(o.order_date)                                   AS last_order_date
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.order_status NOT IN ('cancelled', 'unknown')
    GROUP BY c.customer_id, c.name, c.state, c.city
),
segmented AS (
    SELECT
        *,
        CASE
            WHEN lifetime_spend >= PERCENTILE_CONT(0.9) WITHIN GROUP
                (ORDER BY lifetime_spend) OVER () THEN 'High Value'
            WHEN lifetime_spend >= PERCENTILE_CONT(0.5) WITHIN GROUP
                (ORDER BY lifetime_spend) OVER () THEN 'Mid Value'
            ELSE 'Low Value'
        END AS spend_segment
    FROM customer_spend
)
SELECT
    spend_segment,
    COUNT(customer_id)                                      AS customer_count,
    ROUND(AVG(lifetime_spend)::NUMERIC, 2)                 AS avg_lifetime_spend,
    ROUND(SUM(lifetime_spend)::NUMERIC, 2)                 AS total_segment_revenue,
    ROUND(AVG(total_orders)::NUMERIC, 2)                   AS avg_orders_per_customer
FROM segmented
GROUP BY spend_segment
ORDER BY avg_lifetime_spend DESC;
```

**Key Finding:** A small high-value customer segment is responsible for a disproportionate share of total revenue — consistent with the product concentration finding. Protecting this segment through loyalty programmes and personalised retention is critical.

---

### Query 6 — Payment Method Preferences by State

**Business Question:** Which payment methods do customers prefer in each state, and are there state-level differences that should inform payment infrastructure investment?

```sql
SELECT
    c.state,
    p.payment_method,
    COUNT(DISTINCT o.order_id)                              AS total_orders,
    ROUND(SUM(oi.subtotal)::NUMERIC, 2)                    AS total_revenue,
    ROUND(
        COUNT(DISTINCT o.order_id)::NUMERIC
        / SUM(COUNT(DISTINCT o.order_id)) OVER (PARTITION BY c.state) * 100, 2
    )                                                       AS method_share_pct
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN payments p ON o.order_id = p.order_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE p.payment_status = 'completed'
  AND o.order_status NOT IN ('cancelled', 'unknown')
GROUP BY c.state, p.payment_method
ORDER BY c.state, total_orders DESC;
```

**Key Finding:** Payment method preferences vary meaningfully by state — understanding these patterns helps TradeZone prioritise payment infrastructure investment (e.g. USSD, card, bank transfer, mobile money) by region rather than applying a blanket strategy nationally.

---

### Query 7 — Review Ratings vs Sales Performance

**Business Question:** Are customers using reviews as a purchasing signal? Do low-rated products sell less than high-rated alternatives at comparable price points?

```sql
WITH product_ratings AS (
    SELECT
        p.product_id,
        p.product_name,
        p.category,
        p.price,
        ROUND(AVG(r.rating)::NUMERIC, 2)                   AS avg_rating,
        COUNT(r.review_id)                                  AS review_count
    FROM products p
    LEFT JOIN reviews r ON p.product_id = r.product_id
    GROUP BY p.product_id, p.product_name, p.category, p.price
),
product_sales AS (
    SELECT
        oi.product_id,
        COUNT(DISTINCT oi.order_id)                         AS total_orders,
        SUM(oi.quantity)                                    AS total_units_sold,
        ROUND(SUM(oi.subtotal)::NUMERIC, 2)                AS total_revenue
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.order_status NOT IN ('cancelled', 'unknown')
    GROUP BY oi.product_id
)
SELECT
    pr.product_id,
    pr.product_name,
    pr.category,
    pr.price,
    pr.avg_rating,
    pr.review_count,
    CASE
        WHEN pr.avg_rating >= 4.0 THEN 'High Rated (4.0+)'
        WHEN pr.avg_rating >= 3.0 THEN 'Mid Rated (3.0–3.9)'
        WHEN pr.avg_rating IS NOT NULL THEN 'Low Rated (<3.0)'
        ELSE 'No Reviews'
    END                                                     AS rating_band,
    ps.total_orders,
    ps.total_units_sold,
    ps.total_revenue
FROM product_ratings pr
LEFT JOIN product_sales ps ON pr.product_id = ps.product_id
ORDER BY pr.avg_rating DESC NULLS LAST, ps.total_revenue DESC;
```

**Key Finding:** Low-rated products (below 3.0) were still generating **meaningful order volume at prices comparable to better-rated alternatives** — a clear sign that the review system was not functioning as a market signal for customers. This points to low review visibility or low customer trust in the review system.

---

### Query 8 — Top Seller Bonus Qualification

**Business Question:** Which sellers qualify for a performance bonus based on revenue generated, fulfilment rate, and order volume thresholds?

```sql
WITH seller_performance AS (
    SELECT
        s.seller_id,
        s.seller_name,
        s.state,
        COUNT(DISTINCT o.order_id)                          AS total_orders,
        SUM(CASE WHEN o.order_status = 'delivered' THEN 1 ELSE 0 END) AS delivered_orders,
        ROUND(SUM(oi.subtotal)::NUMERIC, 2)                AS total_revenue,
        ROUND(
            SUM(CASE WHEN o.order_status = 'delivered' THEN 1.0 ELSE 0 END)
            / NULLIF(COUNT(DISTINCT o.order_id), 0) * 100, 2
        )                                                   AS delivery_rate_pct
    FROM sellers s
    JOIN orders o ON s.seller_id = o.seller_id
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY s.seller_id, s.seller_name, s.state
)
SELECT
    seller_id,
    seller_name,
    state,
    total_orders,
    delivered_orders,
    total_revenue,
    delivery_rate_pct,
    CASE
        WHEN total_revenue >= 500000
         AND delivery_rate_pct >= 85
         AND total_orders >= 50
        THEN '✅ Qualifies for Bonus'
        ELSE '❌ Does Not Qualify'
    END                                                     AS bonus_status,
    CASE
        WHEN total_revenue < 500000
        THEN 'Revenue below threshold (₦500K)'
        WHEN delivery_rate_pct < 85
        THEN 'Delivery rate below 85%'
        WHEN total_orders < 50
        THEN 'Order count below 50'
        ELSE 'All criteria met'
    END                                                     AS qualification_note
FROM seller_performance
ORDER BY total_revenue DESC;
```

**Key Finding:** A clear distinction emerged between high-performing sellers who met all three bonus criteria (revenue, delivery rate, and order volume) and those who fell short on one or more dimensions — giving the Seller Operations team a data-backed framework for reward and intervention.

---

## Stage 3 — Analyst Memo

> The following is a summary of the analyst memo delivered to TradeZone's Head of Growth and Head of Seller Operations, translating SQL findings into business decisions.

---

**TO:** Head of Growth | Head of Seller Operations
**FROM:** Data Analytics Team
**DATE:** 2025
**RE:** TradeZone Platform Analysis — Key Findings and Recommended Actions

---

### Executive Summary

Analysis of TradeZone's transactional data across Lagos, Abuja, Kano, Port Harcourt, and Ibadan has surfaced four priority findings with direct implications for the next planning cycle. This memo translates those findings into decisions.

---

### Finding 1 — Customer Acquisition Is Not Converting to Revenue

**The Data:** First-30-day conversion rates are significantly below 50% in several states. Customers are registering but not purchasing.

**The Implication:** Every naira spent on acquisition in low-converting states is partially wasted. The platform is filling the top of the funnel without closing it.

**Recommended Actions:**
- Introduce a **first-purchase incentive** (discount, free delivery, or bonus credit) triggered at registration and valid for 14 days
- Build a **Day 7 / Day 14 re-engagement email or SMS flow** for registered-but-not-purchased customers
- Investigate whether conversion differences across states are driven by **payment friction**, **product availability**, or **UX/onboarding gaps**

---

### Finding 2 — Revenue Concentration Creates Fragility

**The Data:** A small number of products and sellers account for a disproportionate share of total revenue.

**The Implication:** If one or two top sellers reduce activity or exit the platform, revenue impact would be immediate and significant. This is a single-point-of-failure risk embedded in the current seller mix.

**Recommended Actions:**
- Identify the **top 5 revenue-driving sellers** and place them on a formal **Key Seller Relationship programme** (dedicated account manager, priority support)
- Run a **seller diversification campaign** to onboard new sellers in categories where concentration is highest
- Set an internal KPI target: **no single seller should exceed X% of total platform GMV**

---

### Finding 3 — The Review System Is Not Working as a Market Signal

**The Data:** Products rated below 3.0 are generating comparable order volumes to better-rated alternatives at similar price points.

**The Implication:** Customers are either not seeing reviews, not trusting them, or not being served them at the right moment in the purchase journey. The review system is collecting data that is not influencing behaviour — making it useless as a quality signal and a quality improvement mechanism.

**Recommended Actions:**
- **Surface reviews more prominently** on product listing and product detail pages
- Introduce a **"Verified Purchase" badge** on reviews to increase trust
- Consider **suppressing or flagging products** with ratings below 2.5 and fewer than 5 reviews from high-traffic search results until they accumulate sufficient signal
- Send a **post-delivery review request** via SMS/email within 48 hours of order completion to increase review volume

---

### Finding 4 — Seller Fulfilment Quality Is Uneven

**The Data:** Delivery rates and cancellation rates vary significantly across the seller base.

**The Implication:** Customers experiencing late delivery or cancellation from a low-performing seller are likely to blame the TradeZone platform, not the individual seller. Seller underperformance is a brand and retention risk.

**Recommended Actions:**
- Publish an internal **Seller Scorecard** covering delivery rate, cancellation rate, and review rating
- Implement a **performance improvement plan (PIP)** trigger: any seller whose delivery rate drops below 70% for two consecutive months receives a formal warning and support intervention
- Tie **platform visibility** (search ranking, promotional placement) to fulfilment performance

---

### Priority Data Gap — What the Data Could Not Explain

The current dataset has **no attribution data** (how customers found TradeZone) and **no post-purchase feedback mechanism** beyond product reviews. This means:

- We cannot determine whether low conversion is driven by acquisition quality, onboarding UX, payment friction, or product-market fit
- We cannot understand why customers who purchased did not return
- We cannot connect marketing channel investment to revenue outcome

**Recommendation:** Make **attribution tagging** (UTM parameters or source field at registration) and a **post-purchase NPS or churn survey** a data collection priority for 2025. Without this data, growth decisions remain partially informed.

---

## Key Findings & Insights

| # | Finding | Business Impact |
|---|---|---|
| 1 | 30-day conversion below 50% in multiple states | Acquisition spend partially wasted; retention risk from day 1 |
| 2 | Revenue concentrated in narrow product/seller set | Single-point-of-failure; high fragility if top sellers exit |
| 3 | Low-rated products (<3.0) selling at volumes comparable to better-rated alternatives | Review system not functioning as intended market signal |
| 4 | Significant spread in seller delivery and cancellation rates | Inconsistent customer experience; brand risk from underperforming sellers |
| 5 | No attribution or post-purchase feedback data | Cannot explain churn or connect acquisition to revenue |

---

## Data Gaps & Recommendations

| Gap | Why It Matters | Recommended Fix |
|---|---|---|
| No acquisition source/channel attribution | Cannot assess marketing ROI or identify which channels bring converting customers | Add UTM/source field to customer registration table |
| No post-purchase feedback | Cannot understand why customers don't return after first order | Build a post-delivery NPS survey triggered 48hrs after delivery |
| No session or browse data | Cannot see if customers browsed before buying, or abandoned carts | Implement event logging on product views and cart additions |
| No inventory data | Cannot determine if low conversion is supply-side (out of stock) or demand-side | Connect inventory/stock levels to product and order tables |
| No delivery timestamp detail | DaysToShip is available but no expected delivery date makes SLA measurement imprecise | Add `expected_delivery_date` field to orders table |

---

## Business Recommendations

| Priority | Recommendation | Owner | Timeline |
|---|---|---|---|
| 🔴 Critical | First-purchase incentive programme for new registrants | Head of Growth | Q1 2025 |
| 🔴 Critical | Key Seller Relationship programme for top revenue drivers | Head of Seller Ops | Q1 2025 |
| 🟠 High | 7-day and 14-day re-engagement flow for non-converting registrants | Growth / CRM Team | Q1 2025 |
| 🟠 High | Seller Scorecard and PIP framework for underperforming sellers | Head of Seller Ops | Q1 2025 |
| 🟠 High | Review visibility improvements on product listing pages | Product Team | Q2 2025 |
| 🟡 Medium | Seller diversification campaign (onboard new sellers in concentrated categories) | Seller Ops | Q2 2025 |
| 🟡 Medium | Post-delivery review request via SMS/email | CRM / Product | Q2 2025 |
| 🟡 Medium | Attribution tagging at customer registration | Engineering | Q2 2025 |
| 🟢 Low | Post-purchase NPS / churn survey implementation | Product / Analytics | Q3 2025 |
| 🟢 Low | Inventory data integration into analytics schema | Data Engineering | Q3 2025 |

---

## Project Structure

```
TradeZone-SQL-Analytics/
│
├── README.md                              # This document
│
├── 📁 data/
│   ├── tradezone_raw_dump.sql             # Original raw data (pre-cleaning)
│   └── tradezone_clean_dump.sql           # Cleaned data export (post-cleaning)
│
├── 📁 cleaning/
│   ├── 01_standardise_cities.sql
│   ├── 02_normalise_categories.sql
│   ├── 03_resolve_nulls.sql
│   ├── 04_remove_duplicates.sql
│   └── 05_validate_order_amounts.sql
│
├── 📁 analysis/
│   ├── 01_customer_acquisition_conversion.sql
│   ├── 02_product_revenue_performance.sql
│   ├── 03_seller_fulfilment_efficiency.sql
│   ├── 04_quarterly_revenue_trends.sql
│   ├── 05_customer_spend_segmentation.sql
│   ├── 06_payment_method_by_state.sql
│   ├── 07_review_ratings_vs_sales.sql
│   └── 08_top_seller_bonus_qualification.sql
│
└── 📁 memo/
    └── analyst_memo_tradezone_2025.md     # Full executive memo
```

---

## How to Reproduce This Analysis

### Prerequisites
- PostgreSQL 14+ installed
- pgAdmin or psql command-line access

### Steps

```bash
# 1. Create the database
createdb tradezone_analytics

# 2. Load the cleaned data dump
psql -d tradezone_analytics -f data/tradezone_clean_dump.sql

# 3. Run cleaning scripts in order (if starting from raw data)
psql -d tradezone_analytics -f cleaning/01_standardise_cities.sql
psql -d tradezone_analytics -f cleaning/02_normalise_categories.sql
psql -d tradezone_analytics -f cleaning/03_resolve_nulls.sql
psql -d tradezone_analytics -f cleaning/04_remove_duplicates.sql
psql -d tradezone_analytics -f cleaning/05_validate_order_amounts.sql

# 4. Run analysis queries
psql -d tradezone_analytics -f analysis/01_customer_acquisition_conversion.sql
psql -d tradezone_analytics -f analysis/02_product_revenue_performance.sql
# ... and so on for all 8 queries
```
----
## 👤 Author

*(Yusuf Shotunde, LinkedIn: www.linkedin.com/in/yusuf-shotunde / GitHub: https://github.com/Yusinho/ / CV: https://github.com/Yusinho/my-portfolio/blob/main/Yusuf_Lanre_Shotunde_CV.pdf )*

---

*This project was built as part of an ongoing data analytics portfolio — demonstrating the ability to clean real-world messy data, write production-quality SQL, and translate findings into business decisions.*

Attached is the google drive containing the cleaned dump, solutions to each query using SQL analysis and Analyst Memo.
https://drive.google.com/drive/folders/1rCPOUX8654PF3pi1_LvgW9uW9pCOpuU9?usp=sharing


#DataAnalytics #SQL #PostgreSQL #Ecommerce #BusinessIntelligence #DataDriven #Nigeria #TechInAfrica
