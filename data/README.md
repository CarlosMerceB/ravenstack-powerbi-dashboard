# Data Information

## Dataset Source
This project uses the **AWS SaaS Sales** dataset from Kaggle (public dataset).

**Dataset characteristics:**
- 9,800 transaction records
- 20 attributes
- Time period: January 2020 - December 2023
- Geography: 48 countries, 262 cities
- Customers: 99 B2B companies
- Products: 14 SaaS offerings

## Why Data is Not Included

The actual `.pbix` file and raw data are **not included** in this repository for the following reasons:

1. **File Size:** Power BI files can be large (50-100MB+)
2. **Best Practice:** GitHub is optimized for code, not binary files
3. **Portfolio Focus:** This repo showcases analytical skills, not raw data
4. **Privacy:** Even with public datasets, best practice is documentation over data

## Dataset Access

If you'd like to recreate this analysis:
1. Visit [Kaggle - AWS SaaS Sales Dataset](https://www.kaggle.com/) (search for AWS SaaS)
2. Download the CSV file
3. Follow the data validation steps in [Technical Documentation](../docs/Technical_Documentation.md)
4. Use the documented DAX measures to rebuild the dashboard

## Data Model Summary

The analysis uses a **star schema** with:
- **Fact Table:** FactTable (Sales) - 9,800 rows
- **Dimensions:**
  - DimDate - 1,461 rows (generated)
  - DimCustomer - 99 rows
  - DimGeography - 262 rows
  - DimProduct - 14 rows

See [Technical Documentation](../docs/Technical_Documentation.md) for complete data model specifications.
