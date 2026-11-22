# Key Interpretations — ERP Data Analytics Dashboard

Business-level interpretations of the metrics, patterns, and design decisions in the ERP Sales Analytics solution.

---

## How to Read the Sales Page

| Metric | What It Tells You |
|---|---|
| Total Sales vs Sales LY | Are we growing year-over-year? |
| Sales YoY % | By how much (%) are we growing or shrinking? |
| Sales MoM % | Is the current month accelerating or decelerating? |
| Sales Running Total | Are we ahead or behind the cumulative pace from last year? |
| Sales MTD / QTD / YTD | Where do we stand right now within the current period? |

**Interpretation rule:** If `Sales YoY %` is positive but `Sales MoM %` is negative, growth is slowing — a potential early warning signal worth investigating.

---

## How to Read the Product Page

| Metric | What It Tells You |
|---|---|
| Category Rank | Which product categories are the revenue engines |
| Gross Profit Margin % | Are we pricing products correctly relative to cost |
| Active Product Count vs Products Sold | How much of the catalog is actually generating revenue |
| Sales % of Total | Which products are disproportionately driving overall revenue |
| Unit Margin % | Which products are most profitable per unit sold |

**Interpretation rule:** A product with high `Sales % of Total` but low `Unit Margin %` is a volume driver with thin margins — a pricing review candidate. A product with high `Unit Margin %` but low `Sales % of Total` is an undermarketed high-margin opportunity.

---

## How to Read the Customer Page

| Metric | What It Tells You |
|---|---|
| Repeat Customer % | Customer loyalty and retention health |
| Customer LTV | Average lifetime value per active customer |
| New Customers | Acquisition effectiveness in the current period |
| Revenue per Customer | Whether revenue growth is broad-based or concentrated |
| Age Bucket breakdown | Which demographic segment is the core buyer |
| Sales Male vs Female | Gender-based purchasing patterns |
| Sales Married vs Single | Lifestyle-based purchasing patterns |

**Interpretation rule:**
- High `Repeat Customer %` + declining `New Customers` = loyal base but acquisition is stalling
- Low `Repeat Customer %` + high `New Customers` = strong acquisition but poor retention — a churn problem
- High `Revenue per Customer` with low `Total Customers` = concentrated revenue risk (few large buyers)

---

## Dynamic RLS — Business Interpretation

The `Dynamic RLS` role means:

- A **regional sales manager** for North America will only see North American customers and their associated sales — even if they open the same report as a global executive
- The **global executive** (if assigned to all regions in `dim_securities`) sees the full picture
- This eliminates the need to maintain separate reports per region, reducing maintenance overhead and ensuring data governance compliance
- Access changes take effect on the next dataset refresh — no Power BI Desktop changes required

---

## SCD Type 2 — What It Means for Analysis

Both `dim_customers` and `dim_products` use Slowly Changing Dimension Type 2:

- `is_current = TRUE` / `is_active = TRUE` → the record reflects the customer/product's current state
- Historical records (where `is_current = FALSE`) preserve what the customer looked like at the time of purchase
- This means **historical sales are always analyzed against the attributes that were true at the time** — not today's attributes
- Example: If a customer moved from Australia to the United States, past Australian sales remain attributed to Australia in historical analysis

---

## Role-Playing Dates — When to Use Each

| Date | Use Case |
|---|---|
| `order_date` (active) | Default — when was the sale placed? Revenue recognition |
| `ship_date` (inactive) | Logistics analysis — when did we fulfill the order? |
| `due_date` (inactive) | SLA analysis — are we shipping before the promised date? |

Comparing `Sales by Ship Date` vs `Total Sales` (by order date) for the same period reveals **fulfillment lag** — the gap between when orders are placed and when they ship.

---

## Weekday vs Weekend Sales

- **Weekday-dominant sales** suggest a B2B or corporate buyer profile
- **Weekend-dominant sales** suggest a B2C or retail buyer profile
- A balanced split may indicate a mixed customer base requiring differentiated marketing strategies

---

## Data Freshness Consideration

The model uses **Import mode** — all data is loaded into Power BI's in-memory Vertipaq engine at refresh time. This means:

- Reports are fast (no live query latency)
- Data is only as current as the last scheduled refresh
- For near-real-time needs, consider migrating to **DirectQuery** or **Hybrid Tables** in Power BI Premium
