# Assessment Questions & Answers

## Table of Contents
1. [Data Cleaning](#1-data-cleaning)
2. [Data Relationships](#2-data-relationships)
3. [Basic Report](#3-basic-report)
4. [Drill-Down Report](#4-drill-down-report)
5. [KPI and Target Comparison](#5-kpi-and-target-comparison)
6. [Sharing and Collaboration](#6-sharing-and-collaboration)

---

## 1. Data Cleaning

### Task
Clean the data by handling null values, removing duplicates, and correcting data inconsistencies.

### Issues Found in the Dataset

| Issue | Location | Detail |
|---|---|---|
| `'Unknown'` value in `ProductCategory` | SalesTransactions RowID 899 | Auto-resolved by lookup from Products table |
| `'Unknown'` value in `Region` | SalesTransactions RowID 475 | Auto-resolved by lookup from Salesperson table |
| Duplicate product names | Products table | Camera×2, Chair×2, Jeans×2, Dress×2, Dining Table×4 — confirmed as intentional SKU variants at different price points, not duplicates |
| No fully duplicate rows | All tables | Verified — no rows identical across all columns |

**Final row counts after cleaning:**
- Sales Transactions: 2,828 (all rows preserved — no deletions)
- Products: 18 (all rows are distinct SKUs)
- Salesperson: 50

---

### Q: How did you handle missing or inconsistent data?

All data cleaning was performed **at the SQL layer inside Power Query native SQL queries**. The source tables in SQL Server are left completely untouched — all fixes are applied live when Power BI queries the data. This is the correct approach for DirectQuery mode since Power Query transformation steps are not supported in DirectQuery.

Each table uses a **native SQL query** passed via the `[Query = "..."]` option in `Sql.Database()`. This bypasses Power Query transformation steps entirely and is fully DirectQuery-compatible.

**Unknown values:** Both `'Unknown'` values (ProductCategory and Region) are auto-healed using `CASE WHEN` logic that looks up the correct value from the related dimension table. If a lookup also fails, a safe fallback (`'Uncategorized'` / `'Unassigned'`) is used instead of dropping the row.

```sql
-- Auto-heal Unknown ProductCategory
CASE
    WHEN ST.ProductCategory IS NULL
      OR TRIM(ST.ProductCategory) = ''
      OR UPPER(TRIM(ST.ProductCategory)) = 'UNKNOWN'
    THEN COALESCE(
            (SELECT TOP 1 TRIM(P.Category)
             FROM dbo.Products P
             WHERE P.ProductID = ST.ProductID),
            'Uncategorized'
         )
    ELSE TRIM(ST.ProductCategory)
END AS ProductCategory
```

**Precautionary rules** are also applied to guard against future data issues:
- All text columns are `TRIM()`-ed to remove whitespace
- `Region` and `SalesChannel` are validated against fixed allowed value lists
- `DiscountPct` is clamped between 0 and 100
- `QuantitySold` and `Price` are forced positive with `ABS()` and `> 0` filters
- Future-dated transactions are excluded via `SaleDate <= GETDATE()`
- All key columns are checked for `NULL` and blank values

---

### Q: Can you demonstrate how you ensured data integrity across tables?

Data integrity is enforced at two levels:

**1. SQL Server — Foreign Key constraints** (at source):
```sql
-- SalesTransactions → Products
CONSTRAINT FK_Sales_Product
    FOREIGN KEY (ProductID) REFERENCES dbo.Products(ProductID)

-- SalesTransactions → Salesperson
CONSTRAINT FK_Sales_Salesperson
    FOREIGN KEY (SalespersonID) REFERENCES dbo.Salesperson(SalespersonID)
```

**2. Power Query — Defensive SQL** (at query layer):
- Orphan-safe: `Unknown` category and region values are healed by joining back to dimension tables
- The `SalesChannel` column is validated against `('Online', 'In-store')` so invalid channel values are excluded automatically
- Null checks on all key columns (`ProductID`, `SalespersonID`, `TransactionID`) ensure no fact rows with broken references reach the model

---

## 2. Data Relationships

### Task
Establish relationships between Sales Transactions, Products, and Salesperson tables in Power BI.

### Relationships Created

| From Table | From Column | To Table | To Column | Type | Cross-filter |
|---|---|---|---|---|---|
| SalesTransactions | ProductID | Products | ProductID | Many-to-One | Single |
| SalesTransactions | SalespersonID | Salesperson | SalespersonID | Many-to-One | Single |

---

### Q: How did you manage relationships between the tables?

Both relationships were verified to already exist in the model (auto-detected by Power BI from the Foreign Key constraints defined in SQL Server). The relationships were confirmed correct via the MCP Power BI modeling tool:
- Cardinality: Many-to-One ✅
- Cross-filter direction: Single (One Direction) ✅
- Active: Yes ✅

No Many-to-Many relationships exist. There is no direct relationship between `Products` and `Salesperson` — they connect only through `SalesTransactions`, forming a clean **Star Schema**.

Cross-filter direction was kept as **Single** (dimension → fact) rather than Both, which is best practice for DirectQuery — bidirectional filtering can cause ambiguous query paths and significantly degrade performance on live queries.

---

### Q: Can you explain the type of relationship (one-to-many, many-to-many)?

Both relationships are **One-to-Many**:

- One **Product** can appear in many **Sales Transactions**
- One **Salesperson** can appear in many **Sales Transactions**

This forms a classic **Star Schema** where `SalesTransactions` is the central **Fact Table** and `Products` and `Salesperson` are **Dimension Tables**. The clean star schema ensures DAX filter context flows predictably from dimension to fact and measures aggregate correctly at every level.

---

## 3. Basic Report

### Task
Create a report showing total sales and total discount by region, product category, and sales channel.

### DAX Measures Used

```dax
Total Sales =
SUMX(
    SalesTransactions,
    SalesTransactions[QuantitySold] * RELATED(Products[Price])
)
```

```dax
Total Discount Amount =
SUMX(
    SalesTransactions,
    SalesTransactions[QuantitySold]
        * RELATED(Products[Price])
        * SalesTransactions[DiscountPct] / 100
)
```

```dax
Net Sales =
SUMX(
    SalesTransactions,
    SalesTransactions[QuantitySold]
        * RELATED(Products[Price])
        * (1 - SalesTransactions[DiscountPct] / 100)
)
```

### Sales Summary

| Dimension | Breakdown |
|---|---|
| **By Region** | South 17.68M → West 16.66M → East 16.20M → North 16.12M |
| **By Category** | Furniture 34.73M → Electronics 28.01M → Clothing 3.92M |
| **By Channel** | Online 33.84M vs In-store 32.82M (near-equal split) |

### Visuals Built

| Visual | Axis / Rows | Values |
|---|---|---|
| Line Chart | Region | Total Sales, Total Discount Amount |
| Clustered Bar Chart | Product Category | Total Sales, Total Discount Amount |
| Clustered Bar Chart | Sales Channel | Total Sales |
| Slicers | Region, Category, Sales Channel, SalespersonName | Cross-filter all visuals on the page |

---

### Q: Can you build a bar chart/line chart to show total sales by region and category?

Yes. A **Line Chart** was built with Region on the axis and both `Total Sales` and `Total Discount Amount` as values for direct comparison. A separate chart uses `Products[Category]` on the axis. A slicer on `SalesChannel` cross-filters both charts simultaneously.

---

### Q: How did you ensure that the data aggregates correctly?

The key decision was using **SUMX with RELATED** rather than a simple SUM. The SalesTransactions table stores `QuantitySold` and `DiscountPct` but not the actual revenue value — the price must be fetched from the Products table via the relationship.

`SUMX` iterates row by row and `RELATED` fetches the corresponding `Price` for each transaction row, multiplies, and sums. This is the correct approach and works reliably in DirectQuery mode. A simple `SUM(QuantitySold)` would only aggregate units, not revenue.

---

## 4. Drill-Down Report

### Task
Create a report where the user can drill down from Region → Product Category → Salesperson.

### Hierarchy Created

A **user hierarchy** named `Sales Drill-Down` was created on the `SalesTransactions` table with 3 levels:

| Level | Column | Source Table |
|---|---|---|
| 1 | Region | SalesTransactions |
| 2 | Product Category | SalesTransactions |
| 3 | Salesperson | SalesTransactions (via Salesperson relationship) |

Since all 3 columns span different tables, placing them all in the `SalesTransactions` table (which has direct relationships to both dimension tables) ensures the hierarchy works correctly in DirectQuery mode without ambiguity.

### Supporting Drill-Down Measures

| Measure | Purpose |
|---|---|
| `Sales by Region` | Total Sales scoped to Region level only |
| `Sales by Category` | Total Sales scoped to Region + Category |
| `Sales by Salesperson` | Total Sales scoped to all 3 levels |
| `Sales % of Total` | Each level's contribution vs grand total |
| `Sales % of Region` | Each item's contribution vs its parent region |

### Drill-Down Data Highlights

| Region | Top Category | Top Salesperson |
|---|---|---|
| South | Furniture (48.4%) | Isabella Hayes |
| West | Furniture (50.8%) | Olivia Shaw |
| East | Furniture (57.4%) | Caleb Ross |
| North | Furniture (52.0%) | Alexander Foster |

---

### Q: Can you create a drill-down report from Region to Product Category to Salesperson?

Yes. The `Sales Drill-Down` hierarchy is placed on the **Axis** of a Bar Chart visual with `Total Sales` as the value. The visual starts at Region level. Clicking the drill-down arrow navigates into Product Category for a selected region, then into individual Salesperson performance within that category.

---

### Q: Can you show how you enabled drill-down functionality in the report?

The `Sales Drill-Down` hierarchy was created as a formal **user hierarchy** in the semantic model (not just stacked fields in the axis). This means:

1. The hierarchy is reusable across any visual on any report page
2. It is available in the **Fields pane** under SalesTransactions → Hierarchies folder
3. Drag it to the **Axis** field of any bar/column/line chart
4. Click the **↓** (single arrow) on the visual header to enable drill-on-click, or **↓↓** to expand all levels at once
5. The breadcrumb at the top of the visual shows the current level and allows navigating back up

---

## 5. KPI and Target Comparison

### Task
Create a KPI card that compares total sales per region against the sales target.

### DAX Measures Used

```dax
Sales Target =
CALCULATE(
    SUMX(Salesperson, Salesperson[SalesTarget]),
    TREATAS(
        VALUES(SalesTransactions[Region]),
        Salesperson[Region]
    )
)
```

```dax
Sales vs Target =
[Total Sales] - [Sales Target]
```

```dax
Sales vs Target % =
DIVIDE([Total Sales] - [Sales Target], [Sales Target], 0)
```

```dax
Target Achievement % =
DIVIDE([Total Sales], [Sales Target], 0)
```

```dax
KPI Status =
VAR Achievement = DIVIDE([Total Sales], [Sales Target], 0)
RETURN
    IF(Achievement >= 1, 1,
        IF(Achievement >= 0.9, 0, -1)
    )
```

```dax
KPI Color =
VAR Achievement = DIVIDE([Total Sales], [Sales Target], 0)
RETURN
    IF(Achievement >= 1, "#27AE60",
        IF(Achievement >= 0.9, "#F39C12", "#E74C3C")
    )
```

### KPI Results by Region

| Region | Total Sales | Sales Target | Achievement | Status |
|---|---|---|---|---|
| South | 17.68M | 7.13M | 247% | ✅ Above Target |
| West | 16.66M | 6.16M | 271% | ✅ Above Target |
| East | 16.20M | 6.01M | 269% | ✅ Above Target |
| North | 16.12M | 5.37M | 300% | ✅ Above Target |

### Visual Built

A **New Card visual** (Card (new)) was used — one card per region displayed side by side:

| Field Well | Measure |
|---|---|
| Callout value | `Total Sales` |
| Reference label 1 | `Sales Target` |
| Reference label 2 | `Sales vs Target %` |

Conditional formatting was applied using the `KPI Color` measure (returns a hex color string) on both the Callout value font color and the Reference label font color via **Format pane → fx → Field value → KPI Color**.

> **Note:** `KPI Status` (numeric) cannot be used directly for conditional formatting as Power BI requires a text/hex color value for Field value formatting. `KPI Color` was created specifically to return the hex string that Power BI needs.

---

### Q: Can you demonstrate a KPI card that compares sales against the target by region?

Yes. Four **New Card visuals** are shown side by side — one per region — each with a visual-level filter pinning it to its region. The card displays Total Sales as the large callout, with Sales Target and Sales vs Target % shown as reference labels below. The `KPI Color` measure drives conditional formatting so all values turn green when above target, orange when within 10% below, and red when more than 10% below.

### Q: Why TREATAS for Sales Target?

`Salesperson[Region]` and `SalesTransactions[Region]` are two separate columns with no direct relationship between the tables. Without `TREATAS`, the `Sales Target` measure ignores the visual's region filter context and always returns the grand total target.

`TREATAS` maps the current `SalesTransactions[Region]` filter context onto `Salesperson[Region]`, so the target correctly aggregates only the salespersons belonging to the filtered region.

---

## 6. Sharing and Collaboration

### Task
Share the report with others using Power BI's collaboration and sharing features.

---

### Q: How would you share this report with a wider audience while ensuring role-based access to the data?

#### Step 1 — Publish to Power BI Service
1. In Power BI Desktop: **File → Publish → Select Workspace**
2. The report and dataset are uploaded to Power BI Service

#### Step 2 — Share via Workspace
Assign roles in the workspace:
- **Viewer** — can view reports but not edit
- **Contributor** — can edit reports but not manage workspace
- **Admin** — full control

#### Step 3 — Row-Level Security (RLS)

RLS ensures each user only sees data for their region. Since `Salesperson[Region]` and `SalesTransactions[Region]` are separate columns, the RLS filter is applied on the `Salesperson` dimension table and cascades through the relationship to `SalesTransactions`.

**Static RLS (one role per region):**
```dax
-- Role: East Region
Salesperson[Region] = "East"
```
Repeat for North, South, West. After publishing, assign each user's email to their role in the Dataset Security settings.

**Dynamic RLS (single scalable role):**
```dax
-- Role: Dynamic Region Filter
Salesperson[Region] =
    LOOKUPVALUE(
        Salesperson[Region],
        Salesperson[SalespersonID],
        USERPRINCIPALNAME()
    )
```
This matches the logged-in user's principal name against the Salesperson table and automatically filters to their region — no need to manage separate roles per region.

#### Step 4 — DirectQuery + RLS Consideration
Because the report uses **DirectQuery**, RLS filters are passed directly to the SQL Server query at runtime. This means the database itself enforces the data boundary — no region's data is ever loaded into memory for another region's user.

---

*End of Assessment — Thank you for reviewing.*