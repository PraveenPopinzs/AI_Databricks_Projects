# Key Insights — ERP Data Analytics Dashboard

Derived from the semantic model, DAX measures, and report structure of the ERP Sales Power BI project.

---

## Sales Performance

- **Total Sales** is the primary revenue KPI, aggregated from `fact_sale[sales_amount]`
- **Sales YoY %** compares current period revenue against the same period last year using `SAMEPERIODLASTYEAR`, enabling quick identification of growth or decline trends
- **Sales MoM %** tracks month-over-month momentum — useful for spotting seasonal spikes or sudden drops
- **Sales Running Total** provides a cumulative view of revenue over time, helping visualize whether the business is on track against annual targets
- **Sales MTD / QTD / YTD** give real-time progress snapshots at different time granularities for executive reporting

---

## Profitability

- **Gross Profit Margin %** = `(Total Sales - Total Cost) / Total Sales` — the single most important indicator of product pricing health
- **Unit Margin** = Average selling price minus average product cost — highlights which products are most profitable per unit
- **Unit Margin %** normalizes unit margin against price, making cross-product comparisons fair regardless of price tier
- High-volume products with low unit margin may still drive significant gross profit — the `Sales % of Total` measure helps identify these contributors

---

## Product Intelligence

- **Category Rank** uses `RANKX` over all categories by Total Sales — instantly surfaces the top-performing product categories
- **Active Product Count** vs **Products Sold** gap reveals catalog utilization — a large gap means many active products are not generating revenue
- **Sales per Product** measures revenue efficiency per SKU — declining values may indicate catalog bloat or poor product-market fit
- Products with `maintenance_required` flag can be cross-referenced with sales data to assess whether high-maintenance products justify their revenue contribution

---

## Customer Analytics

- **Repeat Customer %** is a key loyalty indicator — a higher percentage signals strong customer retention and brand stickiness
- **Customer LTV (Lifetime Value)** = `Total Sales / Active Customers` — tracks the average revenue generated per active customer
- **New Customers** (filtered by `create_date` year) vs **Total Customers** shows acquisition rate relative to the existing base
- **Revenue per Customer** and **Orders per Customer** together reveal whether growth is driven by acquiring new customers or deepening existing relationships
- **Age Bucket** segmentation (Under 25, 25–34, 35–44, 45–54, 55+) identifies which demographic drives the most revenue

---

## Date & Time Intelligence

- The model uses **role-playing dates** — `order_date` is the active relationship; `ship_date` and `due_date` are inactive and activated on-demand via `USERELATIONSHIP`
- **Sales by Ship Date** vs **Sales by Order Date** comparison can reveal fulfillment lag and logistics performance
- **Sales Weekday vs Weekend** split helps understand whether the business is B2B (weekday-heavy) or B2C (weekend-heavy)

---

## Security & Access Control

- **Dynamic RLS** ensures each user sees only data for their authorized regions — no manual report duplication needed per region
- The `dim_securities` table acts as the access control list (ACL) — adding or removing a user's access requires only a data refresh, not a model change
- Users not present in `dim_securities` will see **zero data**, which is a secure-by-default posture

---

## Data Model Quality

- **SCD Type 2** is implemented on both `dim_customers` and `dim_products` via `start_date`, `end_date`, and `is_current` / `is_active` flags — enabling historical analysis without data loss
- Centralizing all measures in the `_Measures` table improves discoverability and prevents measure sprawl across multiple tables
- The star schema design (1 fact + 4 dimensions) ensures optimal query performance in Power BI's Vertipaq engine
