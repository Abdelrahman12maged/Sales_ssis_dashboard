# 📊 Sales Analytics — Full-Stack Enterprise BI Solution

<div align="center">

![SQL Server](https://img.shields.io/badge/SQL%20Server-CC2927?style=for-the-badge&logo=microsoftsqlserver&logoColor=white)
![SSIS](https://img.shields.io/badge/SSIS-ETL%20Pipeline-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![SSAS](https://img.shields.io/badge/SSAS-Tabular%20Model-FF8C00?style=for-the-badge&logo=microsoft&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-Measures-6264A7?style=for-the-badge&logo=microsoft&logoColor=white)

**An enterprise-grade Business Intelligence pipeline — from raw transactional data to executive dashboards — built on the Microsoft BI stack.**

</div>


## 🎯 Project Overview

This project architects and implements a **production-grade Business Intelligence solution** focused on analyzing corporate sales patterns, financial health, and multi-regional customer behavior.

The system ingests raw operational transaction logs, processes them through an **isolated three-layer staging architecture**, and serves clean analytical arrays optimized for **sub-second executive reporting** via Power BI dashboards connected live to an SSAS Tabular cube.

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        SOURCE LAYER                             │
│                    Excel / Flat Files                           │
└───────────────────────────┬─────────────────────────────────────┘
                            │ SSIS ETL
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      STAGING LAYER                              │
│   sales_ODS  ──►  sales_STG  ──►  sales_DWH (Star Schema)      │
└───────────────────────────┬─────────────────────────────────────┘
                            │ SSAS Tabular
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ANALYTICAL LAYER                              │
│              In-Memory SSAS Tabular Cube + DAX                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Live Connection
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                  PRESENTATION LAYER                             │
│           Power BI — Executive Dark Theme Dashboards            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📁 Data Source & Profile

The source dataset consists of **raw corporate sales transaction logs** exported from an Excel file, containing:

| Category | Fields |
|---|---|
| **Financial Metrics** | Gross Sales, Quantities Shipped, Net Profits, Shipping Costs, Discount Margins |
| **Customer Dimensions** | Unique Customer Profiles, Corporate Segments (Consumer / Corporate / Home Office) |
| **Product Dimensions** | Product IDs, Names, Top-Level Categories, Sub-Categories |
| **Geographic Dimensions** | Regional Markets, Sub-Regions |
| **Operational Dimensions** | Order Priority Levels, Order & Ship Dates |

### Data Quality Enforcement
- Type-casting pipelines for uniform text and numeric structures
- Deterministic NULL-handling to block corrupted strings from propagating downstream
- Postal code extraction via derived column logic from Order IDs
- Surrogate key replacement on all business keys for JOIN performance

---

## 🗄 Data Warehouse Design

The warehouse follows a **Star Schema** architecture decoupling descriptive attributes from numeric transaction fields, maximizing caching efficiency within SSAS.

### Conceptual Model

```
         Dim_Customer
              │
Dim_Location──┼──► Fact_Sales ◄──── Dim_Product
              │
          Dim_Date
     Dim_Market, Dim_Category, Dim_Order_Priority
```

All dimension tables connect to `Fact_Sales` via **strict One-to-Many (1:M)** relationships.

### Logical Model

| Table | Primary Key | Key Columns |
|---|---|---|
| `Dim_Customer` | Customer_ID | Customer_Name, Corporate_Segment |
| `Dim_Location` | Location_ID | Geographic Market, Sub-Region |
| `Dim_Product` | Product_ID | Product_Name, Category, Sub-Category |
| `Dim_Date` | Date_Key | Year, Month, Quarter (auto-generated) |
| `Dim_Market` | Market_ID | Market Name, Territory |
| `Dim_Category` | Category_ID | Category, Sub-Category |
| `Dim_Order_Priority` | Priority_ID | Priority Level |
| `Fact_Sales` | Surrogate FK chain | Sales, Profit, Quantity, Shipping Cost, Discount |

### Physical Model
- **Surrogate Keys**: All business keys replaced with `INT` surrogate keys for optimized multi-table JOINs
- **Text Normalization**: All variable-character fields standardized to `NVARCHAR(100)` — no BLOBs
- **Database Engine**: Microsoft SQL Server (DWH_Sales database)
<img width="500" height="400" alt="Screenshot 2026-05-28 195835" src="https://github.com/user-attachments/assets/285fc160-28da-44f9-aa4d-ea32629a2c85" />

---

## ⚙️ ETL Layer — SSIS

The SSIS layer implements a **three-package sequential pipeline** across three isolated databases.

### Pipeline Overview

```
sales_ODS  ──►  sales_STG  ──►  sales_DWH
  (Raw)         (Cleaned)       (Dimensional)
```

---

### Package A — ODS (Raw Ingestion)

**Objective:** Ingest raw Excel data, apply basic type conversions, and load into the ODS layer.

| Step | Component | Detail |
|---|---|---|
| Truncate | Execute SQL Task | Clears existing ODS records |
| Type Cast | Data Conversion | String → Numeric for measure columns |
| Postal Extraction | Derived Column | Extracts postal code from Order_ID |
| Load | OLE DB Destination | Inserts into `sales_ODS` |

**SSIS Components:** Excel Source · Data Conversion · Derived Column · Data Flow Task · OLE DB Destination
<img width="400" height="343" alt="Screenshot 2026-05-28 202724" src="https://github.com/user-attachments/assets/b1e82577-c40c-466d-b846-ca764d029976" />
<img width="400" height="367" alt="Screenshot 2026-05-28 202751" src="https://github.com/user-attachments/assets/88225e59-39a9-4644-9e08-0e2820824a4c" />

---

### Package B — STG (Staging & Transformation)

**Objective:** Organize cleaned ODS data into dimension and OBT structures for warehousing.

| Step | Task |
|---|---|
| Startup Cleanup | Truncate OBT and all Dim tables |
| Dynamic Date | Auto-populate `DimDate` with calendar logic |
| ODS → OBT | Stage initial transactional data |
| Dim Loads | Dim_Customer · Dim_Location · Dim_Category · Dim_Product · Dim_Market · Dim_Order_Priority |

**SSIS Components:** Sequence Container · Data Flow Tasks · Derived Column · OLE DB Destination
<img width="400" height="350" alt="Screenshot 2026-05-28 204148" src="https://github.com/user-attachments/assets/58c78f15-07af-488d-b218-a9aedddeabc8" />

---

### Package C — DWH (Warehouse Load & Lookup Resolution)

**Objective:** Populate the final star schema with fully resolved surrogate keys.

| Phase | Detail |
|---|---|
| Startup Cleanup | Truncate all DWH dimension tables |
| Load Dimensions | DIM_CUSTOMER · DIM_LOC · DIM_CATEGORY · DIM_PRODUCT · DIM_MARKET · DIM_ORDER_PRIORITY · DIM_DATE |
| Lookup Chain | lkp_order_date · lkp_ship_date · lkp_customer · lkp_product · lkp_market · lkp_order_priority |
| Load Fact | Bulk-insert fully mapped records into `Fact_Sales` |

**SSIS Components:** Two Sequence Containers (Dims / Fact) · Lookup · Data Flow Tasks · OLE DB Destination
<img width="400" height="350" alt="Screenshot 2026-05-28 205957" src="https://github.com/user-attachments/assets/bf696e18-47f2-4963-b3b6-ef66d1d15f1f" />
<img width="400" height="350" alt="Screenshot 2026-05-28 205848" src="https://github.com/user-attachments/assets/9aa54f04-328c-4f24-b551-d8f395156a24" />

---

## 🧠 Analytical Layer — SSAS Tabular

An **in-memory SSAS Tabular model** is deployed on top of the DWH layer, providing sub-second aggregation performance for executive reporting.

The model caches dimension hierarchies and fact relationships into a columnar in-memory store, enabling Power BI Live Connection without DirectQuery latency.

---

## 📐 DAX Measures 

<img width="600" height="500" alt="Screenshot 2026-06-02 233906" src="https://github.com/user-attachments/assets/eefe8bb4-7a80-45c2-bcba-997d878c51de" />

---

## 📊 Visualization Layer — Power BI

The Power BI frontend is built on an **Executive Dark Theme UI/UX** framework, structured across three focused reporting views connected live to the SSAS Tabular model.

### Dashboard Pages

#### Page 1 — Overview & Sales
High-level KPIs, financial health trends, monthly margin evolution, and corporate sales velocity across the full time dimension.

#### Page 2 — Product & Category Performance
Deep-dives into inventory items, sub-category sales distributions, category-level profit contribution rankings, and discount impact analysis.

#### Page 3 — Customer & Market Behavior
Behavioral analytics across consumer segments, geographical markets, and order priority distributions.

### Advanced Engineering Techniques

| Feature | Implementation |
|---|---|
| **Page Navigation** | Native Page Navigator — eliminates UI flicker via local cache and hardware acceleration |
| **Regional Bubble Plot** | Custom scatter: X=Sales, Y=Profit, bubble size = Total Sales volume |
| **Dynamic Top-N** | Filter layer locks to Top 10 revenue clients, responding live to slicer context |
<img width="900" height="400" alt="Screenshot 2026-05-28 212703" src="https://github.com/user-attachments/assets/c106dd9d-a45e-479f-85dc-26ea26aefe70" />
<img width="900" height="400" alt="Screenshot 2026-05-28 212737" src="https://github.com/user-attachments/assets/f9f973c3-c2b1-4b6b-9a48-213a8a6d9b7f" />
<img width="900" height="400" alt="Screenshot 2026-05-28 212827" src="https://github.com/user-attachments/assets/9f190c63-2bc0-45c3-bf76-4277040fde56" />

---



## 👤 Author

**Abdelrahman Abdelmaged Youssef**
Flutter Developer · Data Engineering Student · NTI HireReady Program

[![GitHub](https://img.shields.io/badge/GitHub-Abdelrahman12maged-181717?style=flat-square&logo=github)](https://github.com/Abdelrahman12maged)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin)](http://www.linkedin.com/in/abdelrahman-abdelmaged-b09356249)

---

<div align="center">
<sub>Built with the Microsoft BI Stack · SSIS · SSAS · Power BI · SQL Server</sub>
</div>
