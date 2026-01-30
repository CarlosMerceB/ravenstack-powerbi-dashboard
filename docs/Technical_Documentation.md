# RavenStack SaaS Sales Analytics Dashboard
## Technical Documentation

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Data Architecture](#data-architecture)
3. [Data Model](#data-model)
4. [DAX Measures Reference](#dax-measures-reference)
5. [Dashboard Pages](#dashboard-pages)
6. [Advanced Features](#advanced-features)
7. [Design Decisions](#design-decisions)
8. [Technical Specifications](#technical-specifications)

---

## 1. Project Overview

### **Objective**
Create a comprehensive financial analytics dashboard for a B2B SaaS company (RavenStack) to analyze sales performance, customer segmentation, geographic distribution, and product portfolio across 2020-2023.

### **Business Context**
- **Company**: RavenStack - Enterprise Cloud & Analytics SaaS Provider
- **Dataset**: AWS SaaS Sales (Kaggle)
- **Time Period**: January 1, 2020 - December 31, 2023
- **Geographic Scope**: Global (48 countries, 262 cities across AMER, EMEA, APJ regions)

### **Target Audience**
- CFO / Finance Leadership
- Business Controllers
- BI Analysts
- Strategic Planning Teams

### **Key Metrics**
- Total Revenue: €2.30M
- Total Profit: €286.4K
- Profit Margin: 12.47%
- CAGR (2020-2023): 14.87%
- Customer Base: 99 customers
- Product Portfolio: 14 SaaS products

---

## 2. Data Architecture

### **Source Data**
- **Original Format**: Single CSV table (AWS_SaaS_Sales.csv)
- **Rows**: ~9,800 transaction records
- **Columns**: 20 attributes
- **Grain**: One row = one product sale transaction

### **Data Quality Validation**

**Tools Used**: SQLite for initial data quality assessment

**Process**:
1. **Initial Load**: Imported CSV into SQLite database for exploration
2. **Quality Checks Performed**:
   - Row count validation (confirmed ~9,800 records)
   - Null value detection across all columns
   - Duplicate transaction identification (Row ID uniqueness)
   - Date range verification (2020-2023 coverage)
   - Customer ID gap analysis (identified missing IDs: 1037, 1069)
   - Referential integrity checks (orphaned records)
   - Numeric field validation (Sales, Profit, Discount ranges)
   - Geographic data completeness (Country, City combinations)
   - Product-Customer associations validation

3. **SQL Validation Queries**:
```sql
-- Check for null values
SELECT COUNT(*) FROM sales WHERE customer_id IS NULL;

-- Identify customer ID gaps
SELECT customer_id FROM sales 
GROUP BY customer_id 
ORDER BY customer_id;

-- Validate date range
SELECT MIN(order_date), MAX(order_date) FROM sales;

-- Check for orphaned transactions
SELECT COUNT(*) FROM sales s
LEFT JOIN customers c ON s.customer_id = c.customer_id
WHERE c.customer_id IS NULL;

-- Validate product-customer relationships
SELECT product, COUNT(DISTINCT customer_id) as customer_count
FROM sales
GROUP BY product
ORDER BY customer_count DESC;
```

**Findings**:
- No null values in critical fields (Sales, Profit, Customer ID, Date)
- All transactions have valid geographic and product associations
- Customer ID range: 1001-1101 (101 theoretical, 99 actual)
- Missing IDs: 1037, 1069 (confirmed as deleted/inactive accounts)
- No data quality issues requiring correction
- Clean dataset ready for Power BI import
- All products have multiple customers (no single-customer products)

### **Data Quality Notes**
- Missing Customer IDs: 1037, 1069 (2 accounts deleted/inactive)
- Active Customer Count: 99 (from ID range 1001-1101)
- Date Range: Complete coverage 2020-2023 with no gaps
- All transactions have valid geography, customer, and product associations
- No duplicate transactions (Row ID is unique primary key)
- Product-customer many-to-many relationship confirmed

### **Data Transformation**

**Tool**: Power Query Editor

**Key Transformations**:
1. **Date Formatting**: Converted Order Date from text to proper date type using locale format (dd/mm/yyyy)
2. **Decimal Separators**: Fixed European format (commas to dots) for Sales, Profit, Discount, Quantity
3. **Data Types**: Set appropriate types for all columns (Text for IDs, Currency for financial metrics, Date for temporal data)
4. **Geography Key**: Created composite GeographyID key (Country + "-" + City) for unique geographic identification

**No data cleaning required** - SQLite validation confirmed data integrity

---

## 3. Data Model

### **Architecture**: Star Schema

```
          DimDate (1,461 rows)
               |
               | (1:Many)
               |
DimCustomer ---|--- FactTable (9,800 rows) ---|- DimProduct
  (99 rows)    |                               |  (14 rows)
               |
               | (1:Many)
               |
          DimGeography
           (262 rows)
```

### **Fact Table: FactTable (Sales)**
**Purpose**: Transaction-level sales data

**Columns**:
- **Keys**: Row ID (PK), Order ID, Customer ID, Product, GeographyID, Order Date
- **Measures**: Sales, Quantity, Discount, Profit
- **Attributes**: Contact Name, Segment, License

**Grain**: One row per product sale

### **Dimension Tables**

#### **DimDate**
**Purpose**: Time intelligence and date hierarchies

**Creation Method**: DAX CALENDAR function

**Columns**:
- Date (PK)
- Year (numeric)
- Quarter (numeric 1-4)
- QuarterYear (text: "Q1 2020")
- Month (numeric 1-12)
- MonthName (text: "January")
- Day (numeric)
- QuarterYearSort (numeric sort key: Year*10 + Quarter)
- MonthSort (equals Month, for sorting MonthName)

**Date Range**: 2020-01-01 to 2023-12-31

**Hierarchy**: Year → QuarterYear → MonthName → Day

**Note**: Marked as Date Table in Power BI for time intelligence functions

**DAX Formula**:
```dax
DimDate = 
ADDCOLUMNS(
    CALENDAR(DATE(2020,1,1), DATE(2023,12,31)),
    "Year", YEAR([Date]),
    "Quarter", QUARTER([Date]),
    "Month", MONTH([Date]),
    "MonthName", FORMAT([Date], "MMMM"),
    "QuarterYear", "Q" & QUARTER([Date]) & " " & YEAR([Date]),
    "QuarterYearSort", (YEAR([Date]) * 10) + QUARTER([Date]),
    "Day", DAY([Date])
)
```

#### **DimCustomer**
**Purpose**: Customer master data

**Columns**:
- Customer ID (PK)
- Customer (company name)
- Industry

**Count**: 99 unique customers

**Business Logic**: Segment excluded from dimension as it's transaction-level (varies by purchase size)

#### **DimGeography**
**Purpose**: Geographic hierarchy and location data

**Columns**:
- GeographyID (PK, composite: Country-City)
- Region (AMER, EMEA, APJ)
- Subregion (e.g., NAMER, EMEA, APJ)
- Country
- City

**Count**: 262 unique geographic combinations

**Hierarchy**: Region → Subregion → Country → City

#### **DimProduct**
**Purpose**: Product catalog

**Columns**:
- Product (PK)

**Count**: 14 SaaS products

**Business Logic**: License excluded from dimension as it's transaction-level (unique per sale)

### **Relationships**

| From (Many) | To (One) | Cardinality | Active | Cross-Filter |
|-------------|----------|-------------|---------|--------------|
| FactTable[Order Date] | DimDate[Date] | Many:1 | Yes | Single |
| FactTable[Customer ID] | DimCustomer[Customer ID] | Many:1 | Yes | Single |
| FactTable[GeographyID] | DimGeography[GeographyID] | Many:1 | Yes | Single |
| FactTable[Product] | DimProduct[Product] | Many:1 | Yes | Single |

---

## 4. DAX Measures Reference

### **Base Measures**

```dax
Total Sales = SUM(FactTable[Sales])
```

```dax
Total Profit = SUM(FactTable[Profit])
```

```dax
Total Quantity = SUM(FactTable[Quantity])
```

```dax
Number of transactions = DISTINCTCOUNT(FactTable[Row ID])
```

```dax
Number of Orders = DISTINCTCOUNT(FactTable[Order ID])
```

```dax
Total Products = DISTINCTCOUNT(FactTable[Product])
```

```dax
Total Countries = DISTINCTCOUNT(DimGeography[Country])
```

```dax
Total Cities = DISTINCTCOUNT(DimGeography[City])
```

---

### **Financial Ratios**

```dax
Profit % = DIVIDE([Total Profit], [Total Sales])
```

```dax
Average Order Value = DIVIDE([Total Sales], [Number of Orders])
```

```dax
Product Profit Margin % = DIVIDE([Total Profit], [Total Sales])
```

---

### **Time Intelligence Measures**

```dax
Sales YTD = CALCULATE(SUM(FactTable[Sales]), DATESYTD(DimDate[Date]))
```

```dax
Sales Previous Year = CALCULATE(SUM(FactTable[Sales]), PREVIOUSYEAR(DimDate[Date]))
```

```dax
Sales Previous Quarter = CALCULATE(SUM(FactTable[Sales]), PREVIOUSQUARTER(DimDate[Date]))
```

```dax
Sales Growth YoY % = DIVIDE([Total Sales] - [Sales Previous Year], [Sales Previous Year])
```

```dax
Sales Growth QoQ % = DIVIDE([Total Sales] - [Sales Previous Quarter], [Sales Previous Quarter])
```

---

### **Advanced Growth Metrics**

**CAGR (Compound Annual Growth Rate):**
```dax
CAGR = 
VAR FirstYear = 
    CALCULATE([Total Sales], DimDate[Year] = 2020)
VAR LastYear = 
    CALCULATE([Total Sales], DimDate[Year] = 2023)
VAR NumberOfYears = 3
RETURN
    IF(
        NOT(ISBLANK(FirstYear)) && NOT(ISBLANK(LastYear)) && FirstYear > 0,
        POWER(LastYear / FirstYear, 1 / NumberOfYears) - 1,
        BLANK()
    )
```

**Annual Growth Rate:**
```dax
Annual Growth Rate = DIVIDE([Total Sales] - [Sales Previous Year], [Sales Previous Year])
```

---

### **Annualized Averages**

**Purpose**: Normalize metrics across different time periods for apple-to-apple comparisons

```dax
Avg Annual Revenue per Product = 
VAR YearsSelected = DISTINCTCOUNT(DimDate[Year])
RETURN
    DIVIDE(DIVIDE([Total Sales], YearsSelected), [Total Products])
```

```dax
Avg Annual Sales per Customer = 
VAR YearsSelected = DISTINCTCOUNT(DimDate[Year])
RETURN
    DIVIDE(DIVIDE([Total Sales], YearsSelected), DISTINCTCOUNT(DimCustomer[Customer ID]))
```

```dax
Avg Annual Sales per Country = 
VAR YearsSelected = DISTINCTCOUNT(DimDate[Year])
RETURN
    DIVIDE(DIVIDE([Total Sales], YearsSelected), [Total Countries])
```

---

### **Comparison Measures (KPI Cards)**

**Design Pattern**: Shows "--" for multi-year selections or when comparison isn't applicable

**vs Last Year (Generic Pattern):**
```dax
Sales vs Last Year % = 
VAR YearsSelected = DISTINCTCOUNT(DimDate[Year])
VAR HasSingleQuarterYear = HASONEVALUE(DimDate[QuarterYear])
VAR CurrentSales = [Total Sales]
VAR PriorYearSales = 
    CALCULATE([Total Sales], SAMEPERIODLASTYEAR(DimDate[Date]))
RETURN
    IF(
        YearsSelected = 1 || HasSingleQuarterYear,
        IF(
            NOT ISBLANK(PriorYearSales) && PriorYearSales > 0,
            DIVIDE(CurrentSales - PriorYearSales, PriorYearSales),
            "--"
        ),
        "--"
    )
```

**Key Design Decision**: Uses SAMEPERIODLASTYEAR (not PREVIOUSYEAR) to preserve quarter/month context

**vs Last Quarter (Generic Pattern):**
```dax
Sales vs Last Quarter % = 
VAR YearsSelected = DISTINCTCOUNT(DimDate[Year])
VAR QuarterLevel = HASONEVALUE(DimDate[Quarter]) || HASONEVALUE(DimDate[Month])
VAR CurrentSales = [Total Sales]
VAR PriorQuarterSales = [Sales Previous Quarter]
RETURN
    IF(
        YearsSelected <> 1 || NOT QuarterLevel,
        "--",
        IF(
            NOT ISBLANK(PriorQuarterSales) && PriorQuarterSales > 0,
            DIVIDE(CurrentSales - PriorQuarterSales, PriorQuarterSales),
            "--"
        )
    )
```

**Note**: Similar patterns applied for:
- Profit vs Last Year/Quarter %
- Avg Sales per Customer vs Last Year/Quarter %
- Avg Revenue per Product vs Last Year/Quarter %
- Avg Product Margin vs Last Year %
- Avg Revenue per Country vs Last Year %

---

### **Concentration & Pareto Analysis**

**Top 20% Countries (Pareto 20/80):**
```dax
Top 20% Countries Revenue % = 
VAR TotalCountries = DISTINCTCOUNT(DimGeography[Country])
VAR Top20PercentCount = ROUNDUP(TotalCountries * 0.2, 0)
VAR TopCountriesRevenue = 
    SUMX(
        TOPN(
            Top20PercentCount,
            SUMMARIZE(
                FactTable,
                DimGeography[Country],
                "CountryRevenue", [Total Sales]
            ),
            [CountryRevenue],
            DESC
        ),
        [CountryRevenue]
    )
VAR TotalRevenue = [Total Sales]
RETURN
    DIVIDE(TopCountriesRevenue, TotalRevenue)
```

```dax
Top 20% Country Count = 
ROUNDUP(DISTINCTCOUNT(DimGeography[Country]) * 0.2, 0)
```

```dax
Top Country Revenue % = 
VAR TopCountryRevenue = 
    MAXX(VALUES(DimGeography[Country]), [Total Sales])
RETURN
    DIVIDE(TopCountryRevenue, [Total Sales])
```

```dax
Top Product Revenue % = 
VAR TopProductRevenue = 
    MAXX(VALUES(FactTable[Product]), [Total Sales])
RETURN
    DIVIDE(TopProductRevenue, [Total Sales])
```

---

### **Product Analysis Measures**

```dax
Avg Product Margin = 
AVERAGEX(
    VALUES(FactTable[Product]),
    DIVIDE(
        CALCULATE([Total Profit]),
        CALCULATE([Total Sales])
    )
)
```

```dax
Highest Product Margin % = 
MAXX(
    VALUES(FactTable[Product]),
    DIVIDE(
        CALCULATE([Total Profit]),
        CALCULATE([Total Sales])
    )
)
```

```dax
Highest Margin Product = 
VAR BestProduct = 
    TOPN(
        1,
        ADDCOLUMNS(
            VALUES(FactTable[Product]),
            "Margin", DIVIDE([Total Profit], [Total Sales])
        ),
        [Margin],
        DESC
    )
RETURN
    MAXX(BestProduct, FactTable[Product])
```

**Product Ranking with "Total" Label:**
```dax
Product Rank = 
IF(
    HASONEVALUE(FactTable[Product]),
    FORMAT(
        RANKX(
            ALL(FactTable[Product]),
            [Total Sales],
            ,
            DESC,
            DENSE
        ),
        "0"
    ),
    "Total"
)
```

---

### **Customer Profile Measures (Drill-Through)**

```dax
First Purchase Date = MIN(FactTable[Order Date])
```

```dax
Last Purchase Date = MAX(FactTable[Order Date])
```

```dax
Days Since Last Purchase = 
DATEDIFF([Last Purchase Date], TODAY(), DAY)
```

```dax
Customer Lifetime Value = [Total Sales]
```

```dax
Selected Customer Name = 
IF(
    HASONEVALUE(DimCustomer[Customer]),
    SELECTEDVALUE(DimCustomer[Customer]),
    "All Customers"
)
```

---

### **Dynamic Context Measures**

**Selected Period Display:**
```dax
Selected Period = 
VAR HasMonth = ISFILTERED(DimDate[Month])
VAR HasQuarter = ISFILTERED(DimDate[Quarter])
VAR HasYear = ISFILTERED(DimDate[Year])
RETURN
    SWITCH(
        TRUE(),
        HasMonth, CONCATENATEX(VALUES(DimDate[Month]), DimDate[Month], ", "),
        HasQuarter, CONCATENATEX(VALUES(DimDate[QuarterYear]), DimDate[QuarterYear], ", ", DimDate[QuarterYear], ASC),
        HasYear, CONCATENATEX(VALUES(DimDate[Year]), DimDate[Year], ", ", DimDate[Year], ASC),
        "All Periods"
    )
```

---

## 5. Dashboard Pages

### **Page 1: Executive Summary**

**Purpose**: High-level financial overview and performance trends

**KPIs**:
- Total Revenue (€2.30M) with vs Last Year/Quarter comparisons
- Profit (€286.4K) with vs Last Year/Quarter comparisons
- CAGR 2020-2023 (14.87%) with industry benchmark (18%)
- Year-over-year growth table (2021: -2.83%, 2022: +29.32%, 2023: +20.62%)

**Visuals**:
1. **Revenue by Segment** (Treemap): SMB (€1.16M), Strategic (€706K), Enterprise (€430K)
2. **Profit & Revenue Trends** (Combo chart): Monthly trends with dual axis showing correlation

**Interactivity**:
- Year filter buttons (2020-2023)
- Dynamic subtitle showing selected period
- Drill-down on trend chart (Year → Quarter → Month)
- KPI reference labels show "--" when multiple years selected

**Key Insights**:
- SMB segment dominates (50.5% of revenue)
- Strong recovery 2021→2022 (+29% growth)
- Consistent profitability with 12.47% margin
- CAGR of 14.87% slightly below industry average (18%)

---

### **Page 2: Customer Analysis**

**Purpose**: Customer segmentation and value analysis

**KPIs**:
- Total Customers (99)
- Orders (5K)
- Avg Annual Revenue per Customer (€5.80K)
- Avg Order Value (€458.61)

**Visuals**:
1. **Revenue Distribution by Segment** (Clustered bar): Sales + Profit by SMB/Strategic/Enterprise
2. **Top 10 Customers** (Table): Sales, Profit %, Orders with conditional formatting on margins
3. **Revenue by Industry** (Pie): Finance (20.6%), Energy (13.3%), Manufacturing (11.5%), etc.

**Key Features**:
- Conditional formatting: Green for margins >15%, Red for <5%
- Profit % column highlights profitable vs unprofitable customers
- Industry diversification visible
- Drill-through enabled: Right-click any customer → See Customer Profile page

**Business Value**:
- Identifies high-value customers (Anthem: €55.7K, 10.68% margin)
- Shows segment profitability (Strategic: highest profit per customer)
- Industry concentration analysis
- Customer-level profitability investigation via drill-through

---

### **Page 3: Geography Analysis**

**Purpose**: Geographic performance and market concentration

**KPIs**:
- Countries (48)
- Cities (262)
- Top Country Revenue % (United States: 19.92%)
- Avg Annual Revenue per Country (€11.96K)
- Revenue Concentration Pareto 20/80: 10 countries = 69.26% of revenue

**Visuals**:
1. **Revenue by Region** (Line chart): AMER, EMEA, APJ trends over time
2. **Top 10 Countries** (Table): Revenue and Profit % with conditional formatting
3. **Top 10 Cities** (Table): Revenue and Profit % breakdown
4. **Revenue by Country** (Map): Bubble size = revenue, geographic distribution

**Interactive Features**:
- Hover over countries in table/map → Custom tooltip shows Top 3 Products sold in that country
- Year filter buttons for temporal analysis
- Dynamic subtitle showing selected period

**Strategic Insights**:
- US market dominance (19.92% of total)
- Geographic diversification across 48 countries
- 20% of countries drive 69% of revenue (concentration risk indicator)
- Regional growth: AMER strongest, APJ growing
- Product-market fit visible via country tooltips

---

### **Page 4: Portfolio Analysis**

**Purpose**: Product performance and portfolio optimization

**KPIs**:
- Total Products (14)
- Highest Margin Product (SaaS Connector Pack - Gold)
- Avg Annual Revenue per Product (€43.46K when filtered to 2022)
- Avg Product Margin (18.47% when filtered to 2022)

**Visuals**:
1. **Product Performance Table** (Ranked): Rank, Product, Revenue, Profit, Margin %, Quantity
   - Conditional formatting on margins
   - Shows "Total" in rank column footer
   - Tooltip enabled: Hover shows Top 3 Customers for that product
2. **Top 5 Products** (Pie chart): ContactMatcher (28.3%), FinanceHub (22.2%), Big OI Database (14.3%)
   - Tooltip enabled: Hover shows Top 3 Customers for that product
3. **Product Portfolio Matrix** (Scatter plot): Revenue (X) vs Profit Margin % (Y), Bubble size = Quantity
   - Identifies stars, cash cows, question marks, dogs
   - Shows product positioning across profitability and revenue dimensions

**Interactive Features**:
- Hover over products in table/pie → Custom tooltip shows Top 3 Customers purchasing that product
- Example: ContactMatcher → General Motors, Bank of America, Kroger
- Year filter affects all metrics and comparisons

**Portfolio Insights**:
- Top 3 products = 64.8% of revenue (concentration)
- Margin range: 1.79% (Big OI Database) to 32.13% (SaaS Connector)
- Product #5 (Big OI Database): High volume (440 units), low margin (1.79%) - pricing opportunity
- Scatter plot reveals portfolio balance: mix of high-margin/low-volume and low-margin/high-volume products
- Customer concentration per product visible via tooltip

---

## 6. Advanced Features

### **Feature 1: Customer Profile Drill-Through Page**

**Purpose**: Individual customer deep-dive analysis

**Access Method**: 
- Right-click any customer in Customer Analysis page
- Select "Drill through" → Customer Profile
- Automatically filters all visuals to selected customer

**Configuration**:
- Drill-through field: DimCustomer[Customer]
- "Keep all filters" enabled
- Page hidden from normal navigation (drill-through only)

**Page Layout**:

**Header Section**:
- Customer Name (dynamic card showing selected customer)
- Industry badge
- Customer Lifetime Value (large prominent number)

**KPI Row**:
- Number of Orders
- Avg Revenue per Order
- (Additional KPIs can include: Profit Margin %, Days Since Last Purchase, First/Last Purchase dates)

**Main Visuals**:
1. **Purchase History Timeline** (Line chart):
   - X-axis: Date (quarterly view)
   - Y-axis: Revenue
   - Shows buying patterns and frequency over time
   
2. **Product Mix** (Horizontal bar chart):
   - Products ordered by this customer
   - Sorted by order count
   - Shows product preferences and cross-sell opportunities

3. **Order History Table**:
   - Columns: Order Date, Product, Revenue, Profit, Profit %
   - Sorted by date descending
   - Conditional formatting on Profit %
   - Shows transaction-level detail

**Navigation**:
- Back button (auto-generated by Power BI) returns to Customer Analysis page

**Business Use Case**:
- Account managers investigating customer profitability
- Identifying upsell/cross-sell opportunities (what products haven't they bought?)
- Analyzing customer purchase patterns and frequency
- Troubleshooting profitability issues

**Key Insight Example**: 
Allianz drill-through reveals multiple orders with negative margins (-70%, -80%), indicating pricing or discount issues requiring investigation.

---

### **Feature 2: Country Product Tooltip**

**Purpose**: Show top 3 products sold in each country on hover

**Page Configuration**:
- Page type: Tooltip
- Size: 320x240 pixels (tooltip optimized)
- Hidden from navigation
- Minimal design (no headers, logos, year filters)

**Content**:
1. **Title**: "Top 3 Products" (simple text)
2. **Bar Chart**: 
   - Y-axis: Product
   - X-axis: Total Sales
   - Visual-level filter: Top 3 by Total Sales
   - Data labels enabled
   - Clean formatting (no gridlines, minimal decoration)

**Application**:
- Applied to: Top 10 Countries table and Top 10 Cities table on Geography Analysis page
- Also applied to: Map visual bubbles

**User Experience**:
- Hover over any country/city → Tooltip appears
- Shows which products drive revenue in that specific geography
- Non-intrusive (doesn't navigate away from main page)
- Context-specific (automatically filters to hovered country)

**Business Value**:
- Product-market fit analysis
- Regional product preferences identification
- Localization opportunities
- Market entry strategy (which products to lead with in new markets)
- Sales team enablement (know which products resonate in each market)

**Example Insights**:
- United States: ContactMatcher, FinanceHub, Site Analytics
- Different countries may show different product preferences, indicating regional needs

---

### **Feature 3: Product Customer Tooltip**

**Purpose**: Show top 3 customers purchasing each product on hover

**Page Configuration**:
- Page type: Tooltip
- Size: 320x240 pixels (tooltip optimized)
- Hidden from navigation
- Minimal design (clean, focused)

**Content**:
1. **Title**: "Top 3 Customers" (simple text)
2. **Bar Chart**:
   - Y-axis: Customer
   - X-axis: Total Sales (or Order Count)
   - Visual-level filter: Top 3 by Total Sales
   - Data labels showing revenue amounts
   - Clean formatting

**Application**:
- Applied to: Product Performance Table on Portfolio Analysis page
- Applied to: Top 5 Products pie chart
- Applied to: Product Portfolio Matrix scatter plot (on bubbles)

**User Experience**:
- Hover over any product → Tooltip appears
- Shows which customers are the biggest buyers of that specific product
- Example: ContactMatcher → General Motors (€2.7K), Bank of America (€2.6K), Kroger (€2.5K)
- Helps identify customer-product affinities

**Business Value**:
- **Account management**: Know which customers to target for specific product upsells
- **Customer segmentation**: Identify which customer types buy which products
- **Concentration risk**: See if a product relies too heavily on one customer
- **Cross-sell opportunities**: Identify which customers should buy more of a product
- **Competitive analysis**: Understand customer product preferences
- **Sales territory planning**: Assign territories based on product-customer matches

**Example Business Scenario**:
Marketing team planning ContactMatcher campaign sees that General Motors, Bank of America, and Kroger are top buyers. They can:
1. Create case studies featuring these companies
2. Target similar enterprise financial services companies
3. Understand what drives ContactMatcher adoption in large organizations

**Strategic Integration**:
This tooltip, combined with the Country Product Tooltip, creates a **multi-dimensional analysis framework**:
- Geography page: "What products sell in which countries?"
- Portfolio page: "What customers buy which products?"
- Enables complete product-market-customer mapping

---

## 7. Design Decisions

### **Color Scheme**
**Primary**: RavenStack brand colors
- Navy Blue (#12239E): Primary actions, main metrics
- Bright Blue (#118DFF): Accents, secondary metrics
- Gray (#F5F5F5): Canvas background
- Green/Red: Conditional formatting (positive/negative indicators)

**Rationale**: Professional, corporate-appropriate, high contrast for readability

### **Layout Philosophy**
- **Top-down hierarchy**: Most important metrics (KPIs) at top
- **Left-to-right reading**: Summary → Detail flow
- **Consistent structure**: All pages follow same KPI → Visual → Detail pattern
- **White space**: Generous spacing prevents visual clutter
- **Year filters**: Consistent placement (top right) across all main pages

### **Visualization Choices**

| Visual Type | Use Case | Rationale |
|-------------|----------|-----------|
| KPI Cards with Reference Labels | Single metrics with comparisons | Quick scanning, executive focus, context-aware |
| Clustered Bar | Comparisons | Easy to compare multiple series |
| Line Chart | Trends over time | Shows trajectory and seasonality |
| Treemap/Pie | Composition | Part-to-whole relationships |
| Table | Detailed data | Sortable, conditional formatting, drill-through enabled, tooltip-enhanced |
| Scatter Plot | Portfolio analysis | Multi-dimensional comparison (BCG matrix style) |
| Map | Geography | Intuitive spatial understanding |
| Combo Chart | Dual metrics | Sales vs Profit correlation |

### **Interactivity Design**

**Year Filter Buttons**:
- Primary filter mechanism
- Prominent placement (top of each page)
- Single-select and multi-select supported
- Triggers dynamic subtitle updates
- Affects KPI comparison visibility (shows "--" for multi-year)

**Dynamic Subtitles**:
- Context awareness (shows "2023", "Q3 2023", "All Periods", etc.)
- Adapts to drill-down level
- Prevents user confusion about current filter state

**Drill-Down Hierarchies**:
- Date: Year → Quarter → Month → Day
- Geography: Region → Subregion → Country → City
- Progressive disclosure pattern

**Drill-Through**:
- Enabled on customer dimension
- Provides deep-dive analysis without cluttering main pages
- Maintains all filters from source page
- Back button for easy navigation

**Custom Tooltips (3 types)**:
- **Country Product**: Shows top products per geography
- **Product Customer**: Shows top customers per product
- Non-intrusive hover interactions
- Context-specific information
- Enhances main visuals without adding complexity
- Creates multi-dimensional analysis capability

**Conditional "--" Display**:
- Clear indication when comparisons not applicable
- Prevents misleading metrics
- Educates user on data context

**Cross-Filtering**:
- Enabled between related visuals within a page
- Disabled where it creates confusion (e.g., year slicer ↔ trend chart)

### **Tooltip Strategy**

**Three-Tooltip System Design**:

1. **Country → Products**: Answers "What sells where?"
2. **Product → Customers**: Answers "Who buys what?"
3. **(Potential future)** Customer → Products: "What does each customer buy?"

**Design Philosophy**:
- Keep tooltips minimal and focused
- One insight per tooltip
- Top 3 items only (not overwhelming)
- Consistent visual style across all tooltips
- Applied strategically to high-value visuals only

### **Financial Accuracy Decisions**

**Annualized Averages**:
- Decision: Divide by number of years selected
- Rationale: CFOs think in annual terms; enables apple-to-apple comparisons across time periods
- Example: €164K lifetime revenue per product / 4 years = €41K annual average

**Comparison Logic**:
- Shows "--" for multi-year selections or year-level viewing (for quarterly comparisons)
- Rationale: Comparing aggregated periods doesn't make business sense
- Prevents misinterpretation

**SAMEPERIODLASTYEAR vs PREVIOUSYEAR**:
- Decision: Use SAMEPERIODLASTYEAR for all YoY comparisons
- Rationale: Preserves quarter/month context (Q3 2023 vs Q3 2022, not Q3 2023 vs all 2022)
- Critical fix for accurate period-over-period comparisons

**Product Ranking**:
- Shows "Total" text in footer instead of number
- Rationale: Clear semantic meaning, prevents confusion

---

## 8. Technical Specifications

### **Development Process**

**Phase 1: Data Validation (SQLite)** - 2 hours
- Tool: SQLite database
- Purpose: Validate data integrity before Power BI import
- Activities: Quality checks, gap analysis, relationship validation
- Outcome: Confirmed clean dataset, no cleaning required

**Phase 2: Data Modeling (Power BI)** - 3 hours
- Star schema design
- Dimension table creation and cleaning
- Relationship establishment
- Date table generation with proper hierarchies

**Phase 3: DAX Development** - 6 hours
- 50+ measures across 8 categories
- Iterative refinement of comparison logic
- Time intelligence implementation
- Context-aware measure development
- Testing and debugging of complex formulas

**Phase 4: Dashboard Design** - 8 hours
- 4 main pages layout and visual creation
- KPI card configuration with reference labels
- Chart formatting and conditional formatting
- Color scheme and branding application
- Interactivity configuration (cross-filtering, drill-down)

**Phase 5: Advanced Features** - 4 hours
- Drill-through page creation (Customer Profile)
- Tooltip page 1: Country Product breakdown
- Tooltip page 2: Product Customer breakdown
- Testing and refinement of interactive features

**Phase 6: Testing & Refinement** - 2 hours
- Cross-filter behavior validation
- Comparison measure accuracy testing
- Drill-through functionality verification
- Tooltip behavior testing
- Performance optimization
- Final QA and polish

**Total Development Time: ~25 hours**

### **Tools & Versions**
- **Power BI Desktop**: Latest version for development
- **SQLite**: Data quality validation (initial phase)
- **DAX**: Standard DAX functions (no preview features)
- **Power Query (M)**: For data transformations
- **Data Source**: CSV file (AWS_SaaS_Sales.csv from Kaggle)

### **Performance Optimizations**
- **Star schema design**: Optimized for query performance
- **Measure-based calculations**: Minimizes model size vs calculated columns
- **Appropriate data types**: Minimizes memory footprint
- **Filtered relationships**: Single direction cross-filtering where appropriate
- **Date table optimization**: Contiguous date range, marked as date table
- **Visual-level filters**: Top N filters applied at visual level (not report level)
- **Tooltip optimization**: Lightweight pages with minimal elements

### **File Structure**
```
RavenStack_Dashboard.pbix
├── Data Model
│   ├── FactTable (Sales) - 9,800 rows
│   ├── DimDate - 1,461 rows
│   ├── DimCustomer - 99 rows
│   ├── DimGeography - 262 rows
│   └── DimProduct - 14 rows
├── Measures (50+ total)
│   ├── Base Measures (8)
│   ├── Time Intelligence (5)
│   ├── Comparisons (12)
│   ├── Ratios & Averages (8)
│   ├── Advanced Analytics (7)
│   ├── Customer Profile (5)
│   └── Dynamic Context (2)
├── Pages
│   ├── Executive Summary (visible)
│   ├── Customer Analysis (visible, drill-through enabled)
│   ├── Geography Analysis (visible, Country Product tooltip)
│   ├── Portfolio Analysis (visible, Product Customer tooltip)
│   ├── Customer Profile (hidden, drill-through target)
│   ├── Country Product Tooltip (hidden, tooltip page)
│   └── Product Customer Tooltip (hidden, tooltip page)
└── Assets
    └── RavenStack Logo (PNG)
```

### **Data Refresh**
- **Current**: Static dataset (historical 2020-2023)
- **Future enhancement**: Can be connected to live data source with scheduled refresh
- **Incremental refresh**: Not required for this dataset size (~10K rows)

### **Browser Compatibility**
- Optimized for Power BI Desktop and Power BI Service
- Tested on desktop (1920x1080 resolution)
- Responsive design considerations for tablet viewing
- Drill-through and tooltips work in both Desktop and Service

### **Key Technical Achievements**

1. **Complex DAX Logic**: 
   - Context-aware comparison measures with conditional "--" display
   - SAMEPERIODLASTYEAR for accurate period comparisons
   - Annualized calculations for normalized metrics
   - Mixed data type handling (text + numeric in Product Rank)

2. **Advanced Interactivity**:
   - Drill-through for customer deep-dive
   - Three custom tooltips for multi-dimensional analysis
   - Dynamic subtitles reflecting current filter context
   - Strategic cross-filtering configuration

3. **Financial Precision**:
   - Pareto 20/80 analysis
   - CAGR calculation
   - Product portfolio matrix (BCG-style)
   - Annualized averages for fair comparisons
   - Reference labels with year-over-year and quarter-over-quarter comparisons

4. **Professional Design**:
   - Consistent branding and color scheme
   - Strategic use of white space
   - Conditional formatting for quick insights
   - Clean, executive-appropriate layout
   - RavenStack brand integration

5. **Multi-Dimensional Analysis Framework**:
   - Customer ↔ Product relationship (drill-through + tooltip)
   - Geography ↔ Product relationship (tooltip)
   - Time-based analysis (hierarchies + comparisons)
   - Portfolio analysis (scatter plot matrix)

---

## Summary

This dashboard provides comprehensive financial analytics for a B2B SaaS company with:

**Core Deliverables**:
- **4 analytical pages** covering Executive, Customer, Geography, and Product dimensions
- **3 advanced interactive features**: 
  - 1 drill-through page (Customer Profile)
  - 2 custom tooltip pages (Country→Product, Product→Customer)
- **50+ DAX measures** including advanced time intelligence and comparison logic
- **Star schema data model** with 1 fact table and 4 dimension tables
- **Multi-dimensional analysis** framework enabling comprehensive business insights

**Technical Rigor**:
- **SQLite validation** for data quality assurance before import
- **Advanced DAX** with SAMEPERIODLASTYEAR, context-aware logic, mixed data types
- **Interactive features**: Drill-through, custom tooltips, dynamic filtering
- **Financial precision**: Annualized metrics, Pareto analysis, CAGR, portfolio matrix

**Design Excellence**:
- **Professional branding** with RavenStack color scheme and logo
- **User-centric interactivity** with appropriate cross-filtering and drill capabilities
- **Executive-appropriate** layout and metric selection
- **Context awareness** with dynamic subtitles and conditional displays
- **Strategic tooltip placement** for enhanced insights without clutter

**Skills Demonstrated**:
- Data modeling and star schema design
- Advanced DAX formulas and time intelligence
- Financial metrics and KPI development
- Dashboard design and UX principles
- Business analysis and strategic thinking
- Data quality validation with SQL
- Advanced Power BI features (drill-through, tooltips)
- Multi-dimensional analysis framework design

**Business Value Delivered**:
- **Customer Intelligence**: Individual customer profitability and product preferences
- **Geographic Strategy**: Product-market fit by region/country
- **Portfolio Optimization**: Product performance and customer base analysis
- **Concentration Risk**: Pareto analysis of customers, products, and markets
- **Growth Analysis**: CAGR, YoY/QoQ comparisons, trend identification

**Portfolio Differentiation**:
- Goes beyond basic dashboards with drill-through capability
- Multiple tooltip strategies for contextual insights
- SQL validation demonstrates data quality awareness
- 25+ hours of professional development work
- Enterprise-grade financial analytics

---

*Documentation Version 3.0 - Final*  
*Updated: January 2026*  
*Project: RavenStack SaaS Sales Analytics Dashboard*  
*Includes: Customer Drill-Through, Country Product Tooltip, Product Customer Tooltip, SQL Data Validation*  
*Total Development: ~25 hours*
