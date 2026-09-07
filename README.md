# Global Superstore Business Intelligence Dashboard

## Project Overview
The Global Superstore Business Intelligence Dashboard represents a complete Power BI analytics solution to evaluate sales performance, profitability, discounting trends, shipping costs, fulfillment efficiency, and customer value across international markets.

Even though Global Superstore pulls in strong overall revenue, several international markets, regions, and product lines suffer from shrinking or negative net profit margins. The main goal of this project was to take raw, messy transactional data and transform it into an interactive business intelligence platform. This lets management quickly pinpoint what is driving top-line revenue—and what is secretly draining profits.

The dashboard answers core business questions around:
* Global sales and profit performance tracking
* Year-over-year executive and regional performance trends
* High-revenue markets that are actually losing money
* Product category and sub-category profitability profiles
* The exact impact of steep discounting on profit margins
* Order-to-ship fulfillment efficiency and bottleneck detection
* Shipping cost efficiency across different delivery modes
* Customer ranking, lifetime value, and unprofitable transaction counts

This project covers the full end-to-end BI pipeline: raw data ingestion, Power Query ETL, dimensional modeling (Star Schema), custom DAX calculations, and user-friendly visual reports.

---

## Dataset

### Source
The dataset was pulled from Kaggle and contains transactional retail data from the Global Superstore dataset.

### Dataset Overview
| Attribute | Details |
| :--- | :--- |
| **Records** | 51,290 |
| **Columns** | 27 |
| **Period** | 2011–2014 |
| **Start Date** | January 1, 2011 |
| **End Date** | December 31, 2014 |
| **Markets** | 7 |
| **Regions** | 13 |
| **Countries** | 147 |
| **Categories** | 3 |
| **Sub-Categories** | 17 |
| **Segments** | 3 |
| **Ship Modes** | 4 |
| **Order Priorities** | 4 |

### Key Metrics
The main quantitative fields analyzed in the dataset:
| Metric | Range |
| :--- | :--- |
| **Sales** | $0 to $22,638 |
| **Quantity** | 1 to 14 units |
| **Discount** | 0% to 85% |
| **Profit** | -$6,599.98 to $8,399.98 |
| **Shipping Cost** | $0.002 to $933.57 |

### Core Business Dimensions
* **Sales & Profit:** Line-item revenue and net financial returns.
* **Volume & Pricing:** Order quantities and applied discount rates.
* **Logistics:** Delivery costs, order priorities, and shipping modes.
* **Entities:** Customer profiles, multi-tier product hierarchies, and geographic breakdowns.

### Data Quality Issues Cleaned
During initial data exploration, I ran into a few data quality problems that had to be fixed before modeling:
1. **Corrupted Encoding:** A broken UTF-8 text string (`è®°å½•æ•°`) representing an obsolete record counter.
2. **Redundant Columns:** Duplicate fields like `Market2`, `weeknum`, and `Year`.
3. **Inconsistent Naming:** Header dots instead of spaces (e.g., `Order.Date`, `Customer.ID`).
4. **Data Types:** Dates formatted as raw text/timestamps rather than proper date objects.
5. **Text Formatting:** Extra spaces and unprintable whitespace characters across text fields.

---

## Tools & Technologies
* **Microsoft Power BI Desktop:** Visual reporting, dashboard creation, and data modeling.
* **Power Query Editor:** ETL process—cleaning, filtering, and structuring raw tables.
* **DAX (Data Analysis Expressions):** Building calculated measures, KPI aggregations, and dynamic logic.
* **GitHub:** Version control and public project documentation.

---

## Data Preparation & Power Query

I used Power Query to transform the flat CSV file into clean, organized relational tables.

### Key Transformation Steps

1. **Removed Corrupted Columns:** Dropped `è®°å½•æ•°` and other redundant columns (`Market2`, `weeknum`, `Year`) to strip out unneeded noise.
2. **Cleaned Header Names:** Renamed dot-separated headers into standard business names (`Order.ID` → `Order ID`, `Customer.ID` → `Customer ID`, `Order.Date` → `Order Date`).
3. **Fixed Date Data Types:** Converted timestamp strings into clean `Date` formats for proper time-intelligence calculations.
4. **Set Financial Precision:** Formatted `Profit` and `Shipping Cost` as `Fixed Decimal Number` (Currency) to stop floating-point rounding errors in DAX.
5. **Calculated Fulfillment Duration:** Created a custom field `Fulfillment Days` using:
   ```m
   Duration.Days([Ship Date] - [Order Date])

* **Operational Turnaround:** This gives us exact operational turnaround time per order.
* **Normalized Dimension Tables:** Split the flat 51,290-row table into normalized dimensions (`DimCustomer`, `DimProduct`, `DimLocation`) and removed duplicate rows to enforce entity integrity.
* **Cleaned String Values:** Applied `Trim` and `Clean` across `City`, `State`, `Product Name`, and `Sub Category` to avoid split categories in visuals.
* **Built Discount Tiers:** Created a `Discount Tier` conditional column for grouping discount depth:

| Discount Tier | Definition |
| :--- | :--- |
| **No Discount** | 0% discount |
| **Low Discount** | 1%–20% discount |
| **High Discount** | > 20% discount |

---

## Data Model
I structured the data model into a clean Star Schema to separate numerical performance facts from descriptive lookup dimensions.

### Model Architecture
* **`FactSales` (Central Fact Table):** Contains 51,290 transaction line items storing key figures (`Sales`, `Profit`, `Quantity`, `Discount`, `Shipping Cost`, `Fulfillment Days`) alongside foreign key links.
* **Dimension Tables:**

| Dimension | Unique Records | Primary Role |
| :--- | :--- | :--- |
| **`DimCustomer`** | 4,873 | Tracks customer details and market segments |
| **`DimProduct`** | 10,292 | Holds product descriptions, categories, and sub-categories |
| **`DimLocation`** | 3,819 | Stores geographic hierarchies (City, State, Country, Region, Market) |
| **`DimDate`** | 1,461 | Dedicated calendar table covering 2011–2014 for time-intelligence DAX |

```text
                 DimCustomer
                      |
                      | (1:*)
DimProduct ----> FactSales <---- DimLocation
   (1:*)              |             (1:*)
                      | (1:*)
                   DimDate
```
### Relationship Design & Direction
* All active connections use **One-to-Many (1:*)** cardinality from the dimension lookup down to `FactSales`.
* **Single Filter Direction** was strictly enforced (`Dimension` filters `FactSales`). This keeps filter contexts clean, speeds up visual rendering, and prevents circular relationship loops.
* **Handling Dual Dates:** `DimDate[Date]` connects directly to `FactSales[Order Date]` as the active relationship. A secondary, inactive relationship connects `DimDate[Date]` to `FactSales[Ship Date]`, which I call on demand using `USERELATIONSHIP()` in DAX when analyzing fulfillment schedules.

---

## DAX Measures
I wrote custom DAX logic to handle performance metrics, period-over-period comparisons, and ranking models.

### Core Metrics
```dax
Total Sales = SUM(FactSales[Sales])

Total Profit = SUM(FactSales[Profit])

Total Quantity = SUM(FactSales[Quantity])

Total Orders = DISTINCTCOUNT(FactSales[Order ID])
```
### Profitability & Financial Ratios
```dax
Profit Margin % = DIVIDE([Total Profit], [Total Sales], 0)

Average Order Value (AOV) = DIVIDE([Total Sales], [Total Orders], 0)

Total Shipping Cost = SUM(FactSales[Shipping Cost])

Freight % of Revenue = DIVIDE([Total Shipping Cost], [Total Sales], 0)

Average Discount % = AVERAGE(FactSales[Discount])
```
### Time Intelligence & Growth
```dax
Prior Year Sales = 
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR(DimDate[Date])
)

YoY Sales Growth % = 
VAR _CurrentSales = [Total Sales]
VAR _PYSales = [Prior Year Sales]
RETURN
DIVIDE(_CurrentSales - _PYSales, _PYSales, 0)
```
### Risk & Discount Diagnostics
```dax
Unprofitable Orders Count = 
CALCULATE(
    [Total Orders],
    FactSales[Profit] < 0
)

High Discount Sales = 
CALCULATE(
    [Total Sales],
    FILTER(
        FactSales,
        FactSales[Discount] >= 0.20
    )
)
```
### Customer Ranking
```dax
Customer Sales Rank = 
IF(
    ISINSCOPE(DimCustomer[Customer Name]),
    RANKX(
        ALL(DimCustomer[Customer Name]),
        [Total Sales],
        ,
        DESC,
        Dense
    ),
    BLANK()
)
```
### DAX Summary Table
| Measure Name | Main Function Used | Purpose |
| :--- | :--- | :--- |
| **Total Sales** | `SUM` | Measures total gross revenue |
| **Total Profit** | `SUM` | Measures net bottom-line financial return |
| **Profit Margin %** | `DIVIDE` | Calculates financial efficiency percentage |
| **Prior Year Sales** | `SAMEPERIODLASTYEAR` | Shifts context back 12 months for baseline checks |
| **YoY Sales Growth %** | `VAR / RETURN / DIVIDE` | Calculates growth pace against the previous year |
| **High Discount Sales** | `CALCULATE / FILTER` | Isolates revenue dependent on discounts $\ge$ 20% |
| **Customer Sales Rank** | `RANKX / ISINSCOPE` | Generates dynamic customer rankings without hardcoding |

---

## Dashboard Visualizations
The dashboard combines high-level executive summary cards with interactive deep-dive analytics pages.

### Navigation & Slicers
Users can filter the entire report canvas dynamically by:
* Order Year / Date Ranges
* Geographic Markets, Regions, and Countries
* Product Category and Sub-Category
* Customer Segment
* Shipping Mode & Discount Tiers

## Key Insights
* **Volume Doesn't Guarantee Profit:** Certain international markets consistently hit high sales volumes while ending up in the negative for net profit due to high shipping expenses and unmonitored promotional cuts.
* **Sub-Category Variations:** High-level category stats look fine on the surface, but drilling into sub-categories reveals that specific products (like Tables or Supplies in certain markets) actively drag down total company profit.
* **The 20% Discount Tipping Point:** Orders discounted above 20% show a steep drop into negative margins. High volume from these deals rarely offsets the direct profit loss.
* **Logistics Cost Squeeze:** Shipping costs take up a surprisingly large slice of gross revenue in far-off regional markets, proving that freight options need tighter controls.
* **Fulfillment Bottlenecks:** Tracking Fulfillment Days reveals that certain shipping modes suffer from notable order-to-ship delays, opening up clear operational targets for improvement.

## Recommendations
* **Cap Aggressive Discounts:** Re-evaluate promotional guidelines to prevent automatic price drops over 20% unless explicitly cleared by sales leads.
* **Audit Logistics & Freight Choices:** Renegotiate carrier rates or establish minimum order sizes for markets where freight costs eat up a high percentage of revenue.
* **Fix or Drop Loss-Making Lines:** Adjust prices, change suppliers, or sunset sub-categories that continuously fail to turn a profit despite solid sales numbers.
* **Focus on Key Accounts:** Use customer ranking lists to build targeted retention programs for top-tier, high-margin client accounts.

---

## Conclusion
This Power BI dashboard turns 51,290 individual transaction lines into a clear, actionable analytics platform. By combining structured ETL, a solid Star Schema, targeted DAX formulas, and intuitive visual layouts, the dashboard gives management the full story—helping them safeguard profit margins, fix delivery bottlenecks, and grow the business smarter.
