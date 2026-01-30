# RavenStack SaaS Financial Analytics Dashboard

> Comprehensive Power BI solution analyzing €2.3M in B2B SaaS revenue across 99 customers, 48 countries, and 14 products (2020-2023)

![Executive Summary](docs/Screenshots/01_Executive_Summary.jpg)

---

## 🎯 Project Overview

**Challenge:** Build a CFO-ready financial analytics dashboard to identify business risks, growth opportunities, and strategic recommendations for a B2B SaaS company.

**Key Finding:** Discovered critical customer retention issue with **0% year-over-year consistency** in top revenue customers, representing >€200K in potential lost revenue annually.

**Impact:** Delivered 3 strategic priorities with **€400K+ projected profit improvement** over 3 years.

---

## 📊 Dashboard Components

### 1. Executive Summary
Financial KPIs, growth trends, revenue by segment
- CAGR: 14.87% (2020-2023)
- Profit margin: 12.47%
- Dynamic YoY/QoQ comparisons with conditional logic

### 2. Customer Analysis
Customer segmentation, retention analysis, top accounts
- 99 customers, €5.8K average annual revenue per customer
- Industry distribution analysis
- **Interactive drill-through** to individual customer profiles

### 3. Geography Analysis
Regional performance, market concentration, expansion opportunities
- 48 countries, 262 cities
- Pareto 20/80: 10 countries = 69% revenue
- **Custom tooltip:** Hover over country → See top 3 products

### 4. Portfolio Analysis
Product profitability, portfolio matrix, strategic positioning
- 14 products analyzed
- Alchemy: 40% margin, 473% growth (star product)
- ContactMatcher: Loss leader strategy (-0.3% margin, top revenue)
- **Custom tooltip:** Hover over product → See top 3 customers

---

## 🔧 Technical Implementation

### Data Architecture
- **Source:** AWS SaaS Sales dataset (Kaggle) - 9,800 transactions
- **Validation:** SQL quality checks before Power BI import
- **Model:** Star schema (1 fact table, 4 dimension tables)
- **Grain:** Transaction-level sales data

### Advanced Features

**1. Star Schema Design**
```
FactTable (9,800 rows)
    ├── DimDate (1,461 rows) - Time intelligence
    ├── DimCustomer (99 rows) - Customer master
    ├── DimGeography (262 rows) - Location hierarchy
    └── DimProduct (14 rows) - Product catalog
```

**2. DAX Measures (50+ total)**
- Base metrics (Sales, Profit, Orders)
- Time intelligence (YoY, QoQ, YTD, CAGR)
- Context-aware comparisons (shows "--" when inappropriate)
- Annualized averages (normalized cross-period)
- Pareto analysis (20/80 rule)
- Product ranking with conditional logic

**3. Advanced DAX Examples**

**SAMEPERIODLASTYEAR for accurate comparisons:**
```dax
Sales vs Last Year % = 
VAR PriorYearSales = 
    CALCULATE([Total Sales], SAMEPERIODLASTYEAR(DimDate[Date]))
RETURN
    DIVIDE([Total Sales] - PriorYearSales, PriorYearSales)
```

**Context-aware comparison logic:**
```dax
Sales vs Last Quarter % = 
VAR YearsSelected = DISTINCTCOUNT(DimDate[Year])
VAR QuarterLevel = HASONEVALUE(DimDate[Quarter])
RETURN
    IF(YearsSelected <> 1 || NOT QuarterLevel, "--", [Calculation])
```

**4. Interactive Features**
- **Customer Drill-Through Page:** Right-click any customer → Full profile with purchase history, product mix, order timeline
- **Geographic Product Tooltip:** Hover over country → Top 3 products sold
- **Product Customer Tooltip:** Hover over product → Top 3 customers purchasing
- **Dynamic Period Display:** Adapts to selected filters (year/quarter/month)

---

## 💡 Business Impact & Strategic Insights

### Critical Findings

**1. Customer Retention Crisis** 🚨
- Top 5 customers change completely year-over-year
- Example: Anthem went from €38K (2020) to minimal activity (2021-2022)
- **Impact:** Unstable revenue base, increased acquisition costs

**2. Product Portfolio Insights**
- **Alchemy** (Star Product): 40% margin, 473% growth 2020-2023, #1 profit contributor
- **ContactMatcher** (Loss Leader): Top revenue but -0.3% margin in 2023
- Strategic opportunity: Bundle ContactMatcher + Alchemy

**3. Geographic Expansion Opportunity**
- Southeast Asia: >25% margins (vs 12.47% average)
- Underutilized high-margin market
- Japan: -15% margin requires fix-or-exit decision

### Strategic Recommendations

**Priority 1: Customer Retention Program**
- Implement customer health scoring
- Assign account managers to top accounts
- Build customer success team
- **Projected Impact:** €400K+ profit improvement

**Priority 2: Product Portfolio Optimization**
- Reposition ContactMatcher as strategic loss leader
- Scale Alchemy aggressively (2x revenue target)
- Implement bundling strategy

**Priority 3: Geographic Expansion**
- Enter high-margin SEA markets
- Diversify UK beyond London (currently 90% concentrated)
- Address Japan profitability

---

## 📁 Documentation

### [📄 Technical Documentation](docs/Technical_Documentation.md)
Complete technical reference covering:
- Data validation methodology (SQL)
- Star schema architecture
- All 50+ DAX formulas with explanations
- Dashboard page specifications
- Advanced features implementation
- Design decisions and rationale
- 25-hour development timeline

**Size:** 1,167 lines | 36KB

### [📊 Business Insights & Strategic Recommendations](docs/Business_Insights.md)
CFO-ready analysis including:
- Current state financial performance
- Key findings (retention crisis, product opportunities)
- Risk assessment (prioritized)
- 3 detailed strategic recommendations with action plans
- Implementation timeline (Q1-Q4 2026)
- Expected financial impact (€1.1-1.2M revenue, 16-17% margins)

**Size:** 486 lines | 21KB

---

## 📸 Dashboard Visuals

<details>
<summary><b>Click to view all dashboard pages</b></summary>

### Executive Summary
![Executive Summary](docs/Screenshots/01_Executive_Summary.jpg)
*Financial KPIs, growth trends, segment analysis*

### Customer Analysis
![Customer Analysis](docs/Screenshots/02_Customer_Analysis.jpg)
*Customer segmentation, top accounts, industry distribution*

### Geography Analysis
![Geography Analysis](docs/Screenshots/03_Geography_Analysis.jpg)
*Regional performance, Pareto analysis, market opportunities*

### Portfolio Analysis
![Portfolio Analysis](docs/Screenshots/04_Portfolio_Analysis.jpg)
*Product profitability, portfolio matrix, strategic positioning*

### Customer Drill-Through (Interactive Feature)
![Customer Profile](docs/Screenshots/05_Customer_Drillthrough.jpg)
*Individual customer deep-dive with purchase history*

### Multi-Dimensional Tooltips (Interactive Feature)
![Tooltip Demo](docs/Screenshots/06_Tooltip_Demo.jpg)
*Hover interactions showing product-customer relationships*

</details>

---

## 🎓 Skills Demonstrated

### Technical Skills
- **Power BI:** Advanced dashboard development, visual design, UX optimization
- **DAX:** Time intelligence, context-aware logic, complex calculations (50+ measures)
- **SQL:** Data validation, quality assurance, integrity checks
- **Data Modeling:** Star schema design, relationship optimization
- **Power Query:** ETL transformations, data type handling

### Business Skills
- **Financial Analysis:** Revenue analysis, profitability metrics, margin analysis
- **Strategic Thinking:** Risk assessment, opportunity identification, prioritization
- **Business Intelligence:** KPI development, executive reporting, insight generation
- **Stakeholder Communication:** CFO-ready deliverables, actionable recommendations

### Analytical Skills
- Customer retention analysis (cohort behavior)
- Product portfolio strategy (BCG matrix approach)
- Geographic market analysis (Pareto principle)
- Competitive positioning and pricing strategy

---

## 🚀 Key Technologies

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-F2C811?style=for-the-badge)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=for-the-badge&logo=postgresql&logoColor=white)
![Power Query](https://img.shields.io/badge/Power%20Query-217346?style=for-the-badge&logo=microsoft-excel&logoColor=white)

---

## 📊 Project Statistics

| Metric | Value |
|--------|-------|
| **Development Time** | ~25 hours |
| **Dashboard Pages** | 7 (4 visible, 3 hidden) |
| **DAX Measures** | 50+ |
| **Data Points** | 9,800 transactions |
| **Customers Analyzed** | 99 |
| **Countries Covered** | 48 |
| **Documentation** | 1,653 lines (57KB) |
| **Strategic Impact** | €400K+ projected profit |

---

## 💼 About This Project

This project was developed as part of my career transition into Business Intelligence and Financial Analysis roles. It demonstrates end-to-end capabilities from data validation through strategic business recommendations.

**Project Type:** Portfolio / Case Study  
**Completed:** January 2026  
**Dataset:** AWS SaaS Sales (Kaggle - public dataset)

---

## 🙏 Acknowledgments

- Dataset: AWS SaaS Sales from Kaggle
- Tools: Power BI Desktop, SQLite, VS Code
- Design inspiration: Enterprise SaaS dashboards and financial reporting best practices

---

*If you found this project helpful, please ⭐ star the repository!*
