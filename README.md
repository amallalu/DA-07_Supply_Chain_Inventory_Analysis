# DA-07 · Supply Chain & Inventory Analysis

**Category:** Data Analytics · **Tools:** Power BI Desktop, Power Query, DAX · **Author:** Amal Lalu K K

![Dashboard preview](preview/dashboard_preview.png)

## 1. Business problem

A UAE distributor with three warehouses keeps running out of fast movers while other stock sits for months. Operations wants to see fill rate, stock-outs and weeks of cover by product and warehouse, and to know which suppliers deliver late.

**Questions this dashboard answers**

- What share of customer demand do we actually ship (fill rate)?
- Which products and warehouses run out of stock most often?
- How many weeks of cover do we hold right now, and what is it worth?
- Which suppliers have long or unreliable lead times?

> **About the data:** Synthetic data from a week-by-week inventory simulation (reorder-point policy with real supplier delays), so stock-outs happen for real reasons.

## 2. What's in this folder

| Path | What it is |
|---|---|
| `data/weeklystock.csv` | Fact table: one row per product, warehouse and week (9,450 rows) |
| `data/purchaseorders.csv` | Fact table: one row per purchase order to a supplier (2,334 rows) |
| `data/products.csv` | One row per SKU (30 rows) |
| `data/warehouses.csv` | One row per warehouse (3 rows) |
| `powerquery/*.m` | Power Query code for each table, ready to paste into the Advanced Editor |
| `dax/measures.dax` | Every DAX measure used in the report |
| `dax/date_table.dax` | The calendar (Date) table |
| `theme/AmalAnalytics_theme.json` | Report colour theme (colourblind-safe palette) |
| `preview/dashboard_preview.png` | What the finished dashboard looks like |
| `STEPS.html` | This guide, formatted to open in any browser |

## 3. Data dictionary

### WeeklyStock — `data/weeklystock.csv`

Fact table: one row per product, warehouse and week

| Column | Type | Description |
|---|---|---|
| `WeekStart` | Date | Monday of the week |
| `SKU` | Text |  |
| `WarehouseID` | Text | Unique identifier / key column |
| `UnitsDemanded` | Whole number | Units customers ordered |
| `UnitsSold` | Whole number | Units shipped (less than demand when stock ran out) |
| `ClosingStock` | Whole number | Units on hand at the end of the week |

### PurchaseOrders — `data/purchaseorders.csv`

Fact table: one row per purchase order to a supplier

| Column | Type | Description |
|---|---|---|
| `POID` | Text | Unique identifier / key column |
| `SKU` | Text |  |
| `WarehouseID` | Text | Unique identifier / key column |
| `Supplier` | Text |  |
| `OrderDate` | Date | Date of the record |
| `ExpectedDate` | Date | Promised delivery date |
| `ReceivedDate` | Date | Actual delivery date (blank = still open) |
| `Quantity` | Whole number | Units ordered |

*Raw file contains deliberate data-quality issues to clean in Power Query: 2,334 raw rows → 2,277 clean rows.*

### Products — `data/products.csv`

One row per SKU

| Column | Type | Description |
|---|---|---|
| `SKU` | Text |  |
| `ProductName` | Text |  |
| `Category` | Text |  |
| `Supplier` | Text |  |
| `UnitCost` | Decimal number | Purchase cost per unit in AED |

### Warehouses — `data/warehouses.csv`

One row per warehouse

| Column | Type | Description |
|---|---|---|
| `WarehouseID` | Text | Unique identifier / key column |
| `WarehouseName` | Text |  |
| `City` | Text |  |

## 4. Build it step by step

### Step 1 · Set up a new report

1. Copy this whole folder somewhere permanent, e.g. `C:\PowerBI\DA-07_Supply_Chain_Inventory_Analysis\`.
2. Open **Power BI Desktop** → **Blank report**.
3. **File → Options and settings → Options → Current file → Data Load** → untick **Auto date/time** (we use our own Date table). Also untick **Autodetect new relationships after data is loaded** under *Relationships*; you will create them yourself in a later step.
4. **View → Themes → Browse for themes** → pick `theme/AmalAnalytics_theme.json`.
5. **File → Save as** → `DA-07_Supply_Chain_Inventory_Analysis.pbix` in this folder. Save often (Ctrl+S).

### Step 2 · Create the folder path parameter

All queries read files relative to one parameter, so the report works on any PC.

1. **Home → Transform data** to open the Power Query Editor.
2. **Home → Manage Parameters → New Parameter**.
3. Name: `FolderPath` · Type: **Text** · Current Value: the full path of this folder **ending with a backslash**, e.g. `C:\PowerBI\DA-07_Supply_Chain_Inventory_Analysis\`.
4. Click **OK**.

### Step 3 · Load and clean the data in Power Query

Repeat for each table below: **Home → New Source → Blank Query** → **Home → Advanced Editor** → delete what's there → paste the code → **Done** → in *Query Settings* on the right, rename the query to the name shown.

#### Query `WeeklyStock`  (file `powerquery/WeeklyStock.m`)

What it does:

- reads `data\weeklystock.csv` and promotes the first row to headers
- sets the data type of every column (using the en-US locale so dates and decimals parse the same on any PC)

```powerquery
let
    Source = Csv.Document(File.Contents(FolderPath & "data\weeklystock.csv"), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {{"WeekStart", type date}, {"SKU", type text}, {"WarehouseID", type text}, {"UnitsDemanded", Int64.Type}, {"UnitsSold", Int64.Type}, {"ClosingStock", Int64.Type}}, "en-US")
in
    #"Changed Type"
```

#### Query `PurchaseOrders`  (file `powerquery/PurchaseOrders.m`)

What it does:

- reads `data\purchaseorders.csv` and promotes the first row to headers
- sets the data type of every column (using the en-US locale so dates and decimals parse the same on any PC)
- trims stray spaces in `Supplier`
- fixes inconsistent capitalisation (e.g. `north`, `NORTH` → `North`) in `Supplier`
- removes incomplete rows where `Quantity` is blank
- removes exact duplicate rows

```powerquery
let
    Source = Csv.Document(File.Contents(FolderPath & "data\purchaseorders.csv"), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {{"POID", type text}, {"SKU", type text}, {"WarehouseID", type text}, {"Supplier", type text}, {"OrderDate", type date}, {"ExpectedDate", type date}, {"ReceivedDate", type date}, {"Quantity", Int64.Type}}, "en-US"),
    #"Trimmed Text" = Table.TransformColumns(#"Changed Type", {{"Supplier", Text.Trim, type text}}),
    #"Capitalized Each Word" = Table.TransformColumns(#"Trimmed Text", {{"Supplier", Text.Proper, type text}}),
    #"Removed Blank Quantity" = Table.SelectRows(#"Capitalized Each Word", each [Quantity] <> null),
    #"Removed Duplicates" = Table.Distinct(#"Removed Blank Quantity")
in
    #"Removed Duplicates"
```

#### Query `Products`  (file `powerquery/Products.m`)

What it does:

- reads `data\products.csv` and promotes the first row to headers
- sets the data type of every column (using the en-US locale so dates and decimals parse the same on any PC)

```powerquery
let
    Source = Csv.Document(File.Contents(FolderPath & "data\products.csv"), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {{"SKU", type text}, {"ProductName", type text}, {"Category", type text}, {"Supplier", type text}, {"UnitCost", type number}}, "en-US")
in
    #"Changed Type"
```

#### Query `Warehouses`  (file `powerquery/Warehouses.m`)

What it does:

- reads `data\warehouses.csv` and promotes the first row to headers
- sets the data type of every column (using the en-US locale so dates and decimals parse the same on any PC)

```powerquery
let
    Source = Csv.Document(File.Contents(FolderPath & "data\warehouses.csv"), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {{"WarehouseID", type text}, {"WarehouseName", type text}, {"City", type text}}, "en-US")
in
    #"Changed Type"
```

When all queries show data, click **Home → Close & Apply**.

### Step 4 · Create the Date table

1. **Modeling → New table** and paste:

```dax
Date =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2024, 1, 1 ), DATE ( 2025, 12, 31 ) ),
    "Year", YEAR ( [Date] ),
    "Quarter", "Q" & QUARTER ( [Date] ),
    "Month Num", MONTH ( [Date] ),
    "Month", FORMAT ( [Date], "MMM" ),
    "Year-Month", FORMAT ( [Date], "YYYY-MM" ),
    "Month Start", DATE ( YEAR ( [Date] ), MONTH ( [Date] ), 1 ),
    "Weekday Num", WEEKDAY ( [Date], 2 ),
    "Weekday", FORMAT ( [Date], "ddd" )
)
```

2. With the table selected: **Table tools → Mark as date table** → choose the `Date` column.
3. Select the `Month` column → **Column tools → Sort by column → Month Num**. Do the same for `Weekday` → **Weekday Num**.

### Step 5 · Create the relationships (Model view)

Open **Model view** (left sidebar). For each row, drag the first column onto the second, or use **Home → Manage relationships → New**. Delete any relationship Power BI created that is not in this list.

| From (many side) | To (one side) | Cardinality | Cross-filter |
|---|---|---|---|
| `WeeklyStock[SKU]` | `Products[SKU]` | Many-to-one (*:1) | Single |
| `WeeklyStock[WarehouseID]` | `Warehouses[WarehouseID]` | Many-to-one (*:1) | Single |
| `WeeklyStock[WeekStart]` | `Date[Date]` | Many-to-one (*:1) | Single |
| `PurchaseOrders[SKU]` | `Products[SKU]` | Many-to-one (*:1) | Single |
| `PurchaseOrders[WarehouseID]` | `Warehouses[WarehouseID]` | Many-to-one (*:1) | Single |
| `PurchaseOrders[OrderDate]` | `Date[Date]` | Many-to-one (*:1) | Single |

Result: a star schema with the fact table(s) in the middle and lookup tables around them.

### Step 6 · Create the measures

1. **Home → Enter data** → name the table `_Measures` → **Load**. (A dedicated table keeps all measures in one place.)
2. Select `_Measures` in the Data pane → **Home → New measure**, paste one measure, press Enter. Repeat for each.
3. After each one, set the format in **Measure tools → Format** as listed.
4. Once you have measures, delete the empty `Column1` from `_Measures`; the table moves to the top of the Data pane.

| Measure | Format | What it means |
|---|---|---|
| `Units Demanded` | Whole number, thousands separator | What customers wanted |
| `Units Sold` | Whole number, thousands separator | What we shipped |
| `Lost Sales Units` | Whole number, thousands separator | Demand we could not meet |
| `Fill Rate` | Percentage, 1 decimal | Share of demand shipped |
| `Stock-out Weeks` | Whole number, thousands separator | Product-warehouse weeks that ended with zero stock |
| `Closing Stock` | Whole number, thousands separator | Units on hand in the latest week in context |
| `Stock Value` | Currency, 0 decimals | Value of closing stock at cost |
| `Avg Weekly Demand` | Whole number, thousands separator | Average units demanded per week |
| `Weeks of Cover` | Decimal number, 1 decimal | How many weeks current stock lasts |
| `Avg Lead Time (days)` | Decimal number, 1 decimal | Order to delivery, received POs only |
| `On-Time Delivery %` | Percentage, 1 decimal | Received on or before the promised date |

```dax
Units Demanded =
    SUM ( WeeklyStock[UnitsDemanded] )

Units Sold =
    SUM ( WeeklyStock[UnitsSold] )

Lost Sales Units =
    [Units Demanded] - [Units Sold]

Fill Rate =
    DIVIDE ( [Units Sold], [Units Demanded] )

Stock-out Weeks =
    COUNTROWS ( FILTER ( WeeklyStock, WeeklyStock[ClosingStock] = 0 ) )

Closing Stock =
    VAR LastWeek = MAX ( WeeklyStock[WeekStart] )
    RETURN
        CALCULATE ( SUM ( WeeklyStock[ClosingStock] ), WeeklyStock[WeekStart] = LastWeek )

Stock Value =
    VAR LastWeek = MAX ( WeeklyStock[WeekStart] )
    RETURN
        SUMX (
            FILTER ( WeeklyStock, WeeklyStock[WeekStart] = LastWeek ),
            WeeklyStock[ClosingStock] * RELATED ( Products[UnitCost] )
        )

Avg Weekly Demand =
    DIVIDE ( [Units Demanded], DISTINCTCOUNT ( WeeklyStock[WeekStart] ) )

Weeks of Cover =
    DIVIDE ( [Closing Stock], [Avg Weekly Demand] )

Avg Lead Time (days) =
    AVERAGEX (
        FILTER ( PurchaseOrders, NOT ISBLANK ( PurchaseOrders[ReceivedDate] ) ),
        DATEDIFF ( PurchaseOrders[OrderDate], PurchaseOrders[ReceivedDate], DAY )
    )

On-Time Delivery % =
    DIVIDE (
        COUNTROWS ( FILTER ( PurchaseOrders, NOT ISBLANK ( PurchaseOrders[ReceivedDate] ) && PurchaseOrders[ReceivedDate] <= PurchaseOrders[ExpectedDate] ) ),
        COUNTROWS ( FILTER ( PurchaseOrders, NOT ISBLANK ( PurchaseOrders[ReceivedDate] ) ) )
    )
```

### Step 7 · Build the report pages

Set the page size first: **View → Page view → Fit to page**, and in **Format page → Canvas settings** use 16:9. Add a text box at the top of each page with the page title.

#### Page: Service Level

| # | Visual | Fields to drag in | Settings |
|---|---|---|---|
| 1 | Card (x5) | Fill Rate · Lost Sales Units · Stock-out Weeks · Weeks of Cover · Stock Value |  |
| 2 | Line chart | X-axis: Date[Date] (or WeeklyStock[WeekStart]) · Y-axis: Units Demanded, Units Sold | The gap between the lines is lost sales |
| 3 | Clustered column chart | X-axis: Products[Category] · Y-axis: Fill Rate |  |
| 4 | Matrix | Rows: Products[ProductName] · Columns: Warehouses[WarehouseName] · Values: Stock-out Weeks | Background colour scale (white → red) highlights problem SKUs |
| 5 | Slicers | Date[Year] · Warehouses[WarehouseName] · Products[Category] |  |

#### Page: Suppliers

| # | Visual | Fields to drag in | Settings |
|---|---|---|---|
| 1 | Clustered bar chart | Y-axis: Products[Supplier] · X-axis: Avg Lead Time (days) | Use Products[Supplier], which reaches PurchaseOrders through the SKU relationship |
| 2 | Clustered bar chart | Y-axis: Products[Supplier] · X-axis: On-Time Delivery % | Add a constant line at 90% |
| 3 | Table | PurchaseOrders[POID], Products[Supplier], OrderDate, ExpectedDate, ReceivedDate, Quantity | Filter ReceivedDate is blank to list open orders |

#### Page: Stock Position

| # | Visual | Fields to drag in | Settings |
|---|---|---|---|
| 1 | Table | Products[ProductName], Warehouses[WarehouseName], Closing Stock, Avg Weekly Demand, Weeks of Cover, Stock Value | Conditional formatting on Weeks of Cover: red < 1, amber 1–3, green > 3 |

### Step 8 · Check your numbers

With **no slicers or filters applied**, your cards should show these values. If one is different, re-check that measure and the cleaning steps.

| Measure | Expected value |
|---|---|
| `Units Demanded` | 223,424 |
| `Units Sold` | 220,512 |
| `Lost Sales Units` | 2,912 |
| `Fill Rate` | 98.7% |
| `Stock-out Weeks` | 376 |
| `Closing Stock` | 4,463 |
| `Stock Value` | AED 806,263 |
| `Avg Weekly Demand` | 2,128 |
| `Weeks of Cover` | 2.1 |
| `Avg Lead Time (days)` | 10.0 |
| `On-Time Delivery %` | 82.0% |

### Step 9 · Save and share

1. **File → Save** — your finished `DA-07_Supply_Chain_Inventory_Analysis.pbix`.
2. **File → Export → Export to PDF** for a static copy, or take a screenshot of each page for your portfolio and GitHub.
3. Optional: **Home → Publish** to Power BI Service (needs a work or school account) to get a shareable link.

## 5. Key findings from this data

- **Average lead time by supplier (days):** highest is Oasis Beauty Imports (16.3), lowest is Emirates Food Distributors (3.4).
- **On-time delivery by supplier:** highest is Emirates Food Distributors (94.2%), lowest is Falcon Tech Supply (67.0%).
- **Fill rate by category:** highest is Grocery (99.1%), lowest is Electronics (97.5%).
- **Weekly units demanded vs sold:** peaks at Apr 25 (2,427); latest value 2,290.

## 6. Skills shown

- DAX: VAR/RETURN, latest-period snapshot measures, AVERAGEX with FILTER, DATEDIFF
- Two fact tables sharing conformed dimensions
- Inventory KPIs: fill rate, stock-outs, weeks of cover, supplier on-time %
- Power Query cleaning
