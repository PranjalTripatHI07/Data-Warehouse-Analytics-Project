# Project Requirements

## Table of Contents
1. Overview
2. Data Engineering Requirements
3. Data Analysis Requirements
4. Technical Specifications
5. Deliverables

---

## Overview

This document outlines the comprehensive requirements for building a modern data warehouse and analytics solution using SQL Server. The project aims to consolidate sales data from multiple source systems into a unified analytical platform following the Medallion Architecture (Bronze, Silver, and Gold layers).

---

## Data Engineering Requirements

### 1. Data Sources

#### Source Systems
- **CRM System** (datasets/source_crm/)
  - `cust_info.csv`: Customer demographic information
  - `prd_info.csv`: Product catalog and pricing
  - `sales_details.csv`: Sales transaction records

- **ERP System** (datasets/source_erp/)
  - `CUST_AZ12.csv`: Customer birth dates and gender
  - `LOC_A101.csv`: Customer location data
  - `PX_CAT_G1V2.csv`: Product category hierarchy and maintenance flags

### 2. Bronze Layer Requirements

#### Objective
Store raw data as-is from source systems without any transformations.

#### Specifications
- Implement tables in the `bronze` schema as defined in `scripts/bronze/ddl_bronze.sql`
- Load data directly from CSV files maintaining original structure and data types
- Table naming convention: `<sourcesystem>_<entity>` (e.g., `crm_cust_info`, `erp_cust_az12`)
- No data validation or cleansing at this layer
- Include loading procedures in `scripts/bronze/proc_load_bronze.sql`

#### Tables Required
- `bronze.crm_cust_info`
- `bronze.crm_prd_info`
- `bronze.crm_sales_details`
- `bronze.erp_loc_a101`
- `bronze.erp_cust_az12`
- `bronze.erp_px_cat_g1v2`

### 3. Silver Layer Requirements

#### Objective
Cleanse, standardize, and transform raw data for analytical consumption.

#### Specifications
- Implement tables in the `silver` schema as defined in `scripts/silver/ddl_silver.sql`
- Apply data quality rules and transformations as specified in `scripts/silver/proc_load_silver.sql`
- Add technical columns: `dwh_create_date` (DATETIME2, default GETDATE())
- Maintain same naming convention as Bronze layer

#### Data Quality Requirements

**Customer Data (`erp_cust_az12`)**
- Handle missing gender values (convert empty strings to NULL or 'n/a')
- Validate birthdates: Must be between 1924-01-01 and current date
- Standardize gender values to: 'Male', 'Female', or 'n/a'
- Reference: `tests/quality_checks_silver.sql`

**Product Data (`crm_prd_info`)**
- Extract category ID from product key (first 5 characters, replace '-' with '_')
- Standardize product line codes:
  - 'M' → 'Mountain'
  - 'R' → 'Road'
  - 'S' → 'Other Sales'
  - 'T' → 'Touring'
  - NULL/Other → 'n/a'
- Handle NULL costs: Replace with 0
- Calculate end dates using LEAD function (one day before next start date)

**Sales Data (`crm_sales_details`)**
- Convert integer dates to DATE format (YYYYMMDD)
- Recalculate sales amount if NULL or ≤ 0: `sales = quantity × price`
- Recalculate price if NULL or ≤ 0: `price = sales ÷ quantity`
- Ensure no negative or zero values for sales, quantity, or price

**Location Data (`erp_loc_a101`)**
- Trim whitespace from country names
- Standardize country values

**Product Categories (`erp_px_cat_g1v2`)**
- Trim whitespace from all text fields
- Standardize maintenance values to 'Yes' or 'No'

### 4. Gold Layer Requirements

#### Objective
Create business-ready dimensional model (star schema) optimized for analytics.

#### Specifications
- Implement tables in the `gold` schema as defined in `scripts/gold/ddl_gold.sql`
- Use meaningful business names following conventions in `docs/naming_conventions.md`
- Implement surrogate keys for dimension tables
- Add technical columns: `dwh_create_date` and `dwh_update_date`

#### Dimensional Model

**Dimension Tables**

1. **`gold.dim_customers`**
   - Surrogate key: `customer_key` (INT IDENTITY)
   - Combine data from: `silver.crm_cust_info`, `silver.erp_cust_az12`, `silver.erp_loc_a101`
   - Attributes:
     - customer_id (business key)
     - customer_number
     - first_name
     - last_name
     - full_name (computed: first_name + ' ' + last_name)
     - marital_status
     - gender
     - birthdate
     - country
     - create_date
   - Reference: `docs/data_catalog.md`

2. **`gold.dim_products`**
   - Surrogate key: `product_key` (INT IDENTITY)
   - Combine data from: `silver.crm_prd_info`, `silver.erp_px_cat_g1v2`
   - Attributes:
     - product_id (business key)
     - product_number
     - product_name
     - category_id
     - category
     - subcategory
     - maintenance_required
     - cost
     - product_line
     - start_date
     - end_date
   - Handle Slowly Changing Dimension (SCD Type 2) based on start_date

**Fact Table**

3. **`gold.fact_sales`**
   - Foreign keys: `product_key`, `customer_key`
   - Measures:
     - order_number
     - order_date
     - shipping_date
     - due_date
     - sales_amount
     - quantity
     - price
   - Grain: One row per sales order line item

### 5. ETL Process Requirements

#### Stored Procedures
- Create stored procedures following naming convention: `load_<layer>`
  - `load_bronze`: Load data from CSV to Bronze layer
  - `load_silver`: Transform Bronze to Silver layer
  - `load_gold`: Build dimensional model in Gold layer
- Reference: `docs/naming_conventions.md`

#### Performance Requirements
- Log execution time for each transformation step
- Print progress messages during execution
- Handle errors gracefully with appropriate error messages

#### Data Refresh Strategy
- Full refresh (TRUNCATE and INSERT) for all layers
- No incremental loading required (latest snapshot only)

---

## Data Analysis Requirements

### 1. Customer Analytics

#### Customer Segmentation
- Analyze customer demographics by gender and age groups
- Identify top customers by sales volume and revenue
- Calculate customer lifetime value (CLTV)
- Analyze customer distribution by country

#### Deliverables
- SQL queries for customer analysis
- Summary of key customer segments
- Insights on high-value customer profiles

### 2. Product Performance Analytics

#### Product Analysis
- Identify top-selling products by revenue and quantity
- Analyze product performance by category and subcategory
- Compare product line performance (Mountain, Road, Touring, Other Sales)
- Evaluate products requiring maintenance vs. maintenance-free products

#### Deliverables
- SQL queries for product analysis
- Product performance rankings
- Category-wise sales breakdown

### 3. Sales Trend Analysis

#### Temporal Analysis
- Analyze sales trends over time (monthly, quarterly, yearly)
- Identify seasonality patterns
- Calculate year-over-year growth rates
- Analyze order fulfillment metrics (order to ship duration)

#### Deliverables
- SQL queries for sales trend analysis
- Time-series visualizations (if applicable)
- Insights on sales patterns and trends

### 4. Business Intelligence Reports

#### Required Reports
- Executive Summary Dashboard metrics
- Sales Performance Report
- Product Category Analysis
- Customer Demographics Report
- Top Performers (Customers & Products)

---

## Technical Specifications

### 1. Database Platform
- **Database:** SQL Server (Express Edition or higher)
- **Management Tool:** SQL Server Management Studio (SSMS)
- **Schema:** Three schemas required: `bronze`, `silver`, `gold`

### 2. Naming Conventions

Follow guidelines in `docs/naming_conventions.md`:

**General Principles**
- Use snake_case (lowercase with underscores)
- Use English for all names
- Avoid SQL reserved words

**Table Names**
- Bronze/Silver: `<sourcesystem>_<entity>`
- Gold: `<category>_<entity>` (e.g., `dim_customers`, `fact_sales`)

**Column Names**
- Surrogate keys: `<entity>_key` (e.g., `customer_key`, `product_key`)
- Technical columns:
  - `dwh_create_date`: Record creation timestamp
  - `dwh_update_date`: Record update timestamp

### 3. Data Quality Checks

Implement quality checks as defined in `tests/quality_checks_silver.sql`:
- Completeness checks (NULL values)
- Range validation (dates, numeric values)
- Referential integrity
- Data consistency
- Standardization verification

### 4. Documentation Requirements

#### Data Catalog
Maintain comprehensive data catalog in `docs/data_catalog.md`:
- Table descriptions
- Column definitions with data types
- Business rules and calculations
- Sample values

#### Architecture Documentation
- Data architecture diagram
- Data flow diagram
- ETL process documentation
- Star schema model

---

## Deliverables

### 1. Code Deliverables
- [ ] Bronze layer DDL scripts (`scripts/bronze/ddl_bronze.sql`)
- [ ] Bronze layer loading procedures (`scripts/bronze/proc_load_bronze.sql`)
- [ ] Silver layer DDL scripts (`scripts/silver/ddl_silver.sql`)
- [ ] Silver layer transformation procedures (`scripts/silver/proc_load_silver.sql`)
- [ ] Gold layer DDL scripts (`scripts/gold/ddl_gold.sql`)
- [ ] Gold layer loading procedures
- [ ] Data quality test scripts (`tests/quality_checks_silver.sql`, `tests/quality_checks_gold.sql`)
- [ ] Analytical SQL queries

### 2. Documentation Deliverables
- [ ] Data catalog (`docs/data_catalog.md`)
- [ ] Naming conventions guide (`docs/naming_conventions.md`)
- [ ] Architecture diagrams
- [ ] ETL process documentation
- [ ] Analysis insights and recommendations
- [ ] README with setup instructions

### 3. Testing Deliverables
- [ ] Unit tests for each layer
- [ ] Data quality validation results
- [ ] Performance benchmarks
- [ ] End-to-end integration tests

---

## Success Criteria

The project will be considered successful when:

1. **Data Pipeline**
   - All CSV files successfully loaded into Bronze layer
   - Silver layer contains cleansed, validated data
   - Gold layer provides accurate dimensional model
   - ETL processes execute without errors

2. **Data Quality**
   - No data quality issues in Silver layer
   - All business rules correctly implemented
   - Referential integrity maintained in Gold layer

3. **Analytics**
   - All required analytical queries execute successfully
   - Results provide meaningful business insights
   - Reports accurately reflect source data

4. **Documentation**
   - Complete data catalog with all tables and columns documented
   - Clear architecture and ETL documentation
   - Naming conventions consistently applied

5. **Best Practices**
   - Code follows SQL best practices
   - Proper error handling implemented
   - Performance optimization applied where needed
   - Version control maintained through Git

---

## Timeline and Milestones

1. **Week 1-2:** Environment Setup & Bronze Layer
   - Set up SQL Server and SSMS
   - Create Bronze schema and tables
   - Implement data loading from CSV files

2. **Week 3-4:** Silver Layer Development
   - Implement data cleansing rules
   - Apply transformations
   - Validate data quality

3. **Week 5-6:** Gold Layer Development
   - Design dimensional model
   - Implement SCD Type 2 for products
   - Build fact and dimension tables

4. **Week 7-8:** Analytics & Reporting
   - Develop analytical SQL queries
   - Generate business insights
   - Create documentation

5. **Week 9:** Testing & Documentation
   - Complete quality checks
   - Finalize documentation
   - Prepare project presentation

---

## Notes

- Focus on the latest data snapshot only; historical tracking is not required except for product SCD Type 2
- Prioritize data quality and consistency over volume
- Document any assumptions or design decisions
- Ensure code is modular and reusable
- Follow the Medallion Architecture principles throughout the project

---

*For additional information, refer to the project `README.md` and documentation files in the `docs` directory.*
