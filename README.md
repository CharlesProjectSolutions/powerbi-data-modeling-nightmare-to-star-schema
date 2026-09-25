# Enterprise B2B Sales & Supply Chain Semantic Model: Re-Architecting bad models into a Star Schema

## Imagine you are working as a Data & Solution Analyst for a global B2B manufacturing and distribution firm. 

The executive leadership, regional sales heads, and supply chain managers want to make critical, data-driven daily decisions and need to answer questions like:
*   *Which regional accounts are reaching their credit limits, and where do we have outstanding risk?*
*   *How many days does it actually take from the moment a B2B order is placed to when the cash is collected in our bank account?*
*   *Which of our product lines are currently over-stocked in our warehouses month-over-month?*
*   *Which marketing campaigns are driving actual wholesale product sales, and which are simply wasting budget on empty clicks?*

Instead of having a single, unified source of truth to answer these questions, your team is handed a fragmented, chaotic staging database of 15 disconnected tables. Direct many-to-many loops and dual-filter directions make reports incredibly sluggish, and worst of all, they calculate incorrect numbers that destroy corporate trust.

This project documents how I stepped into this data "nightmare", drove a structured **4-Phase engineering lifecycle** governed by **5 core architectural rules**, and re-engineered the entire transactional workspace into a secure, high-performance **Galaxy and Star Schema** semantic model in Power BI.

---

## 🛑 The Challenge: Staging Chaos & The "Before" Spider-Web Model
At the start of the migration, the raw database was in a state of "beautiful chaos". Rushing directly to build report visuals without prior data modeling had resulted in a system plagued by severe architectural flaws:
*   **Identical Staging Duplicates:** System integration errors resulted in identical tables imported twice.
*   **Mismatched Table Grains:** Transaction logs and descriptive master tables were joined directly at conflicting granularities, creating a "fan-out" effect that duplicated sales figures.
*   **Heavy Garbage Columns:** Tables were packed with heavy, string-based system hash keys, redundant product descriptions, and pre-aggregated totals that bloated file storage and choked refresh times.
*   **Lack of Development Standards:** Columns featured cryptic abbreviations, inconsistent casing, and mixed business terminology (referring to "customers" in some tables and "users" in others).

### 🕸️ The "Before" Relationship Workspace
```
       +------------------+                   +------------------+
       |   orders_2025    |<-(Many-to-Many)-->|   orders_2026    |
       +--------+---------+                   +--------+---------+
                |                                      |
         (Dual-Filter)                           (Dual-Filter)
                |                                      |
       +--------v---------+                   +--------v---------+
       |   product_master |<-(Many-to-Many)-->|     shipments    |
       +------------------+                   +--------+---------+
                                                       |
                                                 (Dual-Filter)
                                                       |
                                              +--------v---------+
                                              |     sheet_1      |
                                              |   (Duplicate!)   |
                                              +------------------+
```

---

## 🛠️ The 5 Core Modeling Rules (Our Architectural Guardrails)
To prevent the model from sliding back into chaos, every engineering step was strictly governed by five database rules:
1.  **Build a Star Schema:** Place central fact tables (transactional event logs) in the middle, surrounded exclusively by descriptive dimension tables. **Never connect two fact tables directly**; always route filter context through shared dimensions using single-direction, one-to-many relationships.
2.  **Understand the Grain:** Always clearly define and "say out loud" what a single row represents in a table (e.g., one company vs. one contact person) before performing any merges or transforms.
3.  **Make Every Column Earn Its Place:** Drop columns not required for analytics (such as raw descriptions or system hash keys) to optimize file storage, speed up data refreshes, and prevent user confusion.
4.  **Protect the Numbers:** Know the baseline totals of core transactional metrics by heart (specifically the true **Total Sales** card) and validate them after every relationship change to ensure numbers never break silently.
5.  **Enforce Strict Naming Standards:** Define standard naming conventions before writing code, including:
    *   Strict use of a single language (**English**).
    *   Enforced **snake_case** for all tables and columns.
    *   Prefixing tables with `dim_` for dimensions and `fact_` for facts.
    *   Suffixing custom-created surrogate keys with `_key` to differentiate them from source system natural keys (`_id`).
    *   Applying capital casing to text columns and renaming cryptic source abbreviations to friendly, intuitive business terms.

---

## 🔄 The 4-Phase Engineering Lifecycle

### 📂 Phase 1: Prepare & Investigate
*   **Table View Exploration:** Analyzed all 15 staging tables to understand the B2B business domain and classify files as dimensions (master data) or facts (events).
*   **Power Query Housekeeping:** Created numbered folders in Power Query to keep referenced queries completely organized and separate from raw source tables: `1. Stage` (raw, untouched sources), `2. Dimensions`, `3. Facts`, and `4. Support`.

### 📐 Phase 2: Dimension Engineering
To eliminate the cluttered schema, 6 disconnected customer-related tables were consolidated into a single dimension, and product directories were scrubbed:
*   **`dim_customer`:** Merged Customer Master, Contacts, Address, City, and Region tables.
    *   *Grain Alignment Solution:* The Contacts table had multiple rows per B2B customer (different employees), which caused sales figures to duplicate during merges. To align the grains to a strict **1 row = 1 customer company** relationship, the table was filtered to keep **primary contacts only**, successfully protecting the unique 60-customer baseline.
*   **`dim_product`:** Resolved duplicates in kitchen and audio accessory records where blank, data-poor rows leaked from the source system and distorted product lookups. Duplicate rows were filtered out, text fields were capitalized, and an indexed **surrogate key** (`product_key`) was generated to ensure stable relationships.
*   **`dim_order_flags` (Junk Dimension):** Standardized transactional parameters (status, priority, and order channel codes) into a single, performance-optimized "junk" dimension. Grouped cryptic source channel codes (10, 20, 30, 40) and mapped them to user-friendly business labels ("Online Store", "Retail Partner", "Wholesale", "Field Sales").
*   **`dim_geo` (Role-Playing Geography):** Extracted geography details from raw city/region logs. Connected this dimension to the fact table twice (once for `ship_to_city_key` and once for `bill_to_city_key`) utilizing active and inactive relationships to model role-playing behavior.

### 📊 Phase 3: Fact Modeling
*   **`fact_sales` (Sales Transactions):** Appended separate transaction tables (Orders 2025 and Orders 2026). Under headers-and-details rules, the fact was modeled at the lowest, finest detail grain (the detailed line items) to enable accurate roll-ups [66]. All text labels were replaced with numeric surrogate keys, and pre-aggregated order totals from headers were dropped to prevent incorrect summation.
*   **`fact_order_process` (Accumulating Snapshot):** Rushing five separate fact tables for each milestone stage (Order, Shipment, Delivery, Invoice, Payment) would duplicate revenue figures and kill dashboard performance. Instead, an **Accumulating Snapshot Fact Table** was engineered, placing all milestone dates side-by-side in a single row per order, allowing easy process duration tracking.
*   **`fact_promotion_coverage` (Factless Fact):** Exploded comma-separated lists of product SKUs mapped to marketing campaigns, trimming whitespace to resolve character joins. Since this table contains no numeric measures, it was modeled as a **Factless Fact Table** to track campaign-product coverage.

### 🔒 Phase 4: Polish, Secure & Validate
*   **Format Standardization:** Enforced a uniform, compact date format (`YYYY-MM-DD`) and turned off default summarization for non-aggregable numeric parameters.
*   **Shared Date Calendar:** Generated a continuous date table (`dim_date`) using DAX `CALENDARAUTO()`, which dynamically scans the model for the minimum and maximum dates to enable chronological slicing across multiple facts.
*   **Row-Level Security (RLS) Implementation:** Connected regional security mapping tables to `dim_customer`. Implemented a strict regional role using a dynamic DAX filter:
    ```dax
    [region] = LOOKUPVALUE(
        security[region], 
        security[user_email], 
        USERPRINCIPLENAME()
    )
    ```
    *Verified using "View As" to ensure regional managers (e.g., Nora for North America) are automatically restricted to their specific territorial scope.*

---

## 📈 The Final Semantic Data Model (Galaxy / Star Schema)
In our finalized architecture, all relationship filter propagation is strictly unidirectional (flowing outwards from dimensions to facts), preventing filter loops and protecting your aggregate calculations.

```
        +------------------+         +------------------+
        |     dim_date     |         |   dim_product    |
        +--------+---------+         +--------+---------+
                 |                            |
      1:N (Order/Ship/Pay Dates)             1:N (Product Key)
                 |                            |
       +---------v---------+        +---------v---------+
       |fact_order_process |        |    fact_sales     |
       +---------^---------+        +---------^---------+
                 |                            |
           1:N (Customer ID)            1:N (Customer ID)
                 |                            |
        +--------+---------+                  |
        |   dim_customer   <------------------+
        +--------^---------+
                 |
         1:1 (Region Filter)
                 |
        +--------+---------+
        |     security     |
        |  (Row-Level Sec) |
        +------------------+
```

---

## 🧠 Architectural Deep Dives: Engineering Deciphered

### 🔍 Deep Dive 1: The Accumulating Snapshot vs. Slow Fact Joins
In legacy architectures, developers often create five separate fact tables to track different operational milestones: `fact_orders`, `fact_shipments`, `fact_deliveries`, `fact_invoices`, and `fact_payments` [118, 119]. 

To calculate operational cycle times (such as "Order to Payment" duration), they are forced to run massive many-to-many joins across these five heavy fact tables in Power BI [65, 121]. This results in two catastrophic issues:
1.  **Metric Fan-Out:** Joining multiple transactional facts together at different grains duplicates values, fanning out your core revenue metrics [65, 120].
2.  **Resource Exhaustion:** Processing multiple fact-to-fact joins forces the Power BI engine to hold millions of nested relations in active memory, causing query time-outs and dashboard lag [65].

**The Solution:** I engineered an **Accumulating Snapshot Fact Table** (`fact_order_process`) [122]. By using the unique `order_id` as a single master spine, I merged these milestones side-by-side during the ETL phase [122, 124, 125]. This allows us to store the entire order lifecycle in a single row containing five native date fields [125]. Now, calculating cycle durations is a simple, high-performance row-level DAX subtraction (`DATEDIFF`) that requires zero database joins at runtime [154].

### 🔍 Deep Dive 2: Solving B2B Contact Grain Alignment & Fanning
When merging B2B customer master directories with their corresponding contacts, a critical grain mismatch arises [30]. The customer table grain is **1 row = 1 customer company**, whereas the contacts table grain is **1 row = 1 contact person** (with multiple contact people representing a single company) [11, 30].

If you merge these tables directly without correcting the grain, the customer company rows duplicate to accommodate every contact person [29, 30]. When this merged dimension is joined to your sales facts, it duplicates your transactional lines—causing your baseline total sales to skyrocket and report incorrect numbers [29, 30, 84].

**The Solution:** In Power Query, I analyzed the contact records and identified a Boolean flag column: `is_primary` [11, 31]. By filtering the staging query to keep **only rows where `is_primary = TRUE`**, I forced a strict **1-to-1 grain alignment** [31, 32]. This preserved the exact 60 unique customer accounts baseline, ensuring that subsequent merges kept our transaction metrics 100% accurate and protected [28, 32, 79].

### 🔍 Deep Dive 3: Creating Surrogate Keys over Volatile Business Keys
The raw source data linked transactional orders to product files using text-based product names or volatile, alphanumeric business codes [49, 81, 83]. In database design, relying on business keys or text names for relationships is highly risky:
*   Text comparisons are highly sensitive to white-spaces, casing, and trailing characters, leading to broken lookup joins and null values [51, 84, 112].
*   Business keys are prone to change in source systems (e.g., during product re-brandings or database migrations), which breaks historical data connections.

**The Solution:** During Phase 2, I generated an independent, sequential integer **surrogate key** (`product_key`) inside our `dim_product` query using an index column starting at 1 [53]. I then mapped this numeric key into our sales facts during ETL, removing all volatile text names and source keys from the fact table [53, 98]. Because integers require significantly less RAM to store than long text strings, this surrogate key pattern dramatically compressed the final model size and accelerated query performance.

---

## 📋 Governance & Architectural Recommendations

To ensure long-term stability and maintain the high performance of this semantic layer, I have established three governance guidelines for the development team:

### 1. Enforce Strict Semantic Model Ownership
To prevent **"measure sprawl"** and calculation discrepancies, report developers must never write raw aggregation formulas (such as `SUM` or `COUNT`) directly inside front-end report files. All business metrics must be centrally defined as DAX measures inside the empty `_measures` table in the master dataset. If a new calculation is required, it must be peer-reviewed and added to the semantic layer, maintaining a single point of truth across all corporate dashboards.

### 2. Implement Automated Data Quality Gates
Our dimension engineering phase revealed that blank, duplicate product rows occasionally leak from the source system, which can distort our lookup joins. To prevent this from breaking production reports, we recommend implementing **Data Quality Gates** in the upstream data warehouse. If a source file contains duplicate business keys with null metadata fields, the pipeline should automatically quarantine those records and trigger an alert, preventing corrupted data from ever reaching the semantic layer.

### 3. Conduct Self-Service Empowerment Training
Because the model contains inactive, role-playing relationships (such as our date dimension connecting to multiple dates in the accumulating snapshot), business users building self-service reports must be trained on how to activate these paths. Rather than duplicating tables, developers should be trained on utilizing DAX time-intelligence functions—like `USERELATIONSHIP`—to cleanly activate billing vs. shipping calculations dynamically inside their visuals.

---

## 🧮 Key DAX Calculations & Enterprise Governance
To eliminate calculation errors, all key business metrics are bundled in a dedicated, empty folder table named `_measures`:

### 1. Total Sales (Transactional Detail Aggregation)
```dax
Total Sales = SUM(fact_sales[line_total])
```
*Aggregates line-item totals. Formatted as a whole number with thousand separators.*

### 2. Total Orders (Distinct Count over Detail Grain)
```dax
Total Orders = DISTINCTCOUNT(fact_sales[order_id])
```
*Ensures correct counts when evaluating transactions across the line-item detail grain, avoiding fanning-out.*

### 3. Active Customers (Dynamic Fact Evaluation)
```dax
Total Active Customers = DISTINCTCOUNT(fact_sales[customer_id])
```
*Evaluates customers with purchasing activity within the selected filter context.*

### 4. Average Fulfillment Duration (Accumulating Snapshot Average)
```dax
// Calculated Row-Level Column inside [fact_order_process]:
order_to_pay_days = DATEDIFF(fact_order_process[order_date], fact_order_process[pay_date], DAY)

// Dynamic Measure inside [_measures]:
Average Order to Pay = AVERAGE(fact_order_process[order_to_pay_days])
```
*Tracks the average cycle days from order placement to cash payment, responding dynamically to slicers.*

---

## 🏆 Business Outcomes & Portfolio Highlights
*   **Zero Filter Chaos:** Eliminated sluggish, bi-directional many-to-many relationship loops, replacing them with a high-performance unidirectional Star Schema.
*   **Reduced Model Size by ~20%:** Stripped heavy, unused string-based system hash keys, redundant columns, and duplicate staging tables, significantly accelerating query refresh times.
*   **100% Data Integrity:** Protected baseline metrics and verified total sales at every stage of the query transformation, restoring corporate trust in BI reporting.
*   **Governed & Self-Service Ready:** Empowered non-technical report developers with a secure semantic layer containing centralized measures and dynamic row-level data security.

---

## 📂 Repository Structure
```
├── data/
│   └── raw/                       # Staging source files (Note: Excluded due to licensing)
├── model/
│   ├── star-schema-migration.pptx # Detailed technical architecture presentation slides
│   └── star-schema-case-study.pdf # Detailed step-by-step PDF Case Study whitepaper
└── src/
    ├── PowerQuery_Transforms.m    # M code queries for Stage, Dimensions, and Facts
    └── Measures.dax               # Centralized DAX calculations
```

---
Feel free to connect with me on [LinkedIn](https://www.linkedin.com/in/amadico/) to discuss advanced Power BI modeling, data warehousing, and semantic layer architectures!* [166]
