# Power BI Data Modeling — Messy Dataset to Galaxy Schema

## Overview

Rebuilt a deliberately messy, 23-sheet raw dataset into a clean, trustworthy dimensional model in Power BI — six conformed dimensions shared across six fact tables (a **galaxy schema / fact constellation**, since more than one fact table is involved), built with Power Query transformations, surrogate keys, DAX measures, and row-level security. Focus is the model itself, not the report layer.

## Source Data Problems (`dataset.xlsx`, 23 sheets)

| Issue | Example |
|---|---|
| Customer data split across 5 sheets | `CUST_MASTER`, `customer_contacts`, `user_details`, `Address`, `cities` |
| Product hierarchy stored as one combined string | `subcategories` → `"electronics\|phones"` |
| Two years of orders, non-identical schemas | `ORDERS_2025` has extra `LegacyRef`/`SourceFile`; `ORDERS_2026` doesn't |
| Channel stored as a numeric code | `OrderChannel` = 10/20/30/40, no readable name |
| Wide, pivoted inventory | One column per month instead of one row per month |
| Multi-value field | `campaign_skus.PromotedSKUs` = comma-separated product codes |
| Order lifecycle fragmented across 4 sheets | `orders`, `shipments`, `INVOICES`, `payments` never joined |
| Planted test/junk records | `CustomerID 9999` ("TEST ACCOUNT"), `ZZZ-000` ("DO NOT USE"), 2 incomplete rows tagged `SAP-099001`/`SAP-099002` |

## Sheets Deliberately Left Out of the Model

5 of the 23 raw sheets are never referenced by any query — they were reviewed and excluded rather than missed:

| Sheet | Why it was dropped |
|---|---|
| `Sheet1` | Exact duplicate of `shipments` (same columns, same data) |
| `dim_order` | Just a bare list of `OrderID` values, no attributes beyond what `order_id` already carries in `fact_sales`/`fact_order_process` |
| `exchange_rates` | Currency/rate table with no multi-currency logic anywhere else in the model — nothing to convert |
| `invoice_lines` | Invoice-level line detail at a different grain from `order_line_items`; `fact_order_process` only needed the invoice header date, not line detail |
| `regions` | Bare list of region names, already carried as an attribute on `dim_customer`, `dim_geo`, and `security` |

## Before: 23 Disconnected Raw Sheets

No relationships exist between these sheets in the source file — they're grouped below only by real-world subject area, not by any actual link in the data.

<img width="1440" height="852" alt="image" src="https://github.com/user-attachments/assets/ff3a18f2-94ab-4319-89f9-128bf7c8d601" />


## After: Galaxy Schema (Connected, 12 Tables)

**Dimensions:** `dim_customer`, `dim_product`, `dim_geo`, `dim_date`, `dim_campaign`, `dim_order_flags`
**Facts:** `fact_sales`, `fact_inventory`, `fact_campaign_spend`, `fact_promotion_coverage`, `fact_order_process`, `fact_sales_targets`
**Other:** `security` (RLS mapping table), `_measures` (home table for all DAX measures)

<img width="1440" height="850" alt="image" src="https://github.com/user-attachments/assets/d3ca9ea3-874a-4967-86dc-842b63b632a5" />


## Power Query Structure: Staging Queries → Dimensions/Facts

Every raw sheet was loaded as its own query first, with **Enable Load turned off** — so it exists in the query list for reuse but never lands in the data model as a table. Each dimension/fact was then built as a **Reference** off one or more staging queries, cleaned and shaped, and only the final `dim_`/`fact_` query has load enabled. This keeps the model to exactly 12 tables instead of 23, with no raw duplicate data sitting in the model alongside the clean version.

Queries are also organized into 4 folders (query groups) in the Power Query pane, so the staging/dimension/fact split is visible at a glance and not just implied by naming:

| Query group | Contains |
|---|---|
| `01_Stage` | All raw-sheet queries with load disabled — `CUST_MASTER`, `products`, `orders`, `CAMPAIGN_LOG`, `inventory`, etc. |
| `02_Dimensions` | `dim_customer`, `dim_product`, `dim_geo`, `dim_date`, `dim_campaign`, `dim_order_flags` |
| `03_Facts` | `fact_sales`, `fact_inventory`, `fact_campaign_spend`, `fact_promotion_coverage`, `fact_order_process`, `fact_sales_targets` |
| `04_Support` | `security`, `_measures` |

| Staging query (load disabled) | Referenced by |
|---|---|
| `CUST_MASTER`, `customer_contacts`, `user_details`, `Address` | `dim_customer` |
| `products` | `dim_product` |
| `subcategories` | `dim_product` |
| `cities` | `dim_customer`, `dim_geo` |
| `ORDERS_2025` + `ORDERS_2026` → `orders` (combined) | `dim_order_flags`, `fact_sales`, `fact_order_process` |
| `channels` | `dim_order_flags` |
| `CAMPAIGN_LOG` | `dim_campaign`, `fact_campaign_spend` |
| `campaign_skus` | `fact_promotion_coverage` |
| `inventory` | `fact_inventory` |
| `shipments`, `INVOICES`, `payments` | `fact_order_process` |
| `sales_targets` | `fact_sales_targets` |

## Build Steps, by Table

| Table | Built from | Key Power Query steps |
|---|---|---|
| `dim_customer` | `CUST_MASTER` + `customer_contacts` + `user_details` + `Address` + `cities` | Filter out `CustomerID 9999` → merge in contacts, credit/phone details, street, city, region (`NestedJoin` + `ExpandTableColumn`) |
| `dim_product` | `products` + `subcategories` | Filter out `ZZZ-000` and `source_id` = `SAP-099001`/`SAP-099002` → split `subcategories` pipe-delimited string into `category`/`subcategory` → merge in → generate `product_key` (`AddIndexColumn`) |
| `dim_geo` | `cities` | Generate `geo_key` (`AddIndexColumn`) |
| `dim_date` | — | Built with `CALENDARAUTO()` |
| `dim_campaign` | `CAMPAIGN_LOG` | Drop daily spend columns → deduplicate to one row per campaign → generate `campaign_key` |
| `dim_order_flags` | `orders` (combined) + `channels` | Drop order-level columns, keep only channel/status/priority → deduplicate → generate `flag_key` → decode channel code via `channels` lookup |
| `fact_sales` | `order_line_items` + `orders` + `dim_customer` + `dim_product` + `dim_order_flags` + `dim_geo` | Merge line items to orders, then look up each dimension's surrogate key (customer, product, flag, ship-to city, bill-to city) and keep only the keys + measures |
| `fact_inventory` | `inventory` | Unpivot monthly columns (`UnpivotOtherColumns`) → look up `product_key` |
| `fact_campaign_spend` | `CAMPAIGN_LOG` | Look up `campaign_key` → keep daily impressions/clicks/spend |
| `fact_promotion_coverage` | `campaign_skus` | Split comma-separated `PromotedSKUs` into rows (`SplitTextByDelimiter` + `ExpandListColumn`) → look up `campaign_key` and `product_key` |
| `fact_order_process` | `orders` + `shipments` + `INVOICES` + `payments` | Chain of merges on `OrderID`/`InvoiceID` to bring order, ship, invoice, and pay dates onto one row → look up `customer_id` |
| `fact_sales_targets` | `sales_targets` | Rename columns only |
| `security` | `security` sheet | Rename columns only — feeds RLS |

**Two modeling patterns worth calling out:**
- **Junk dimension:** `channel`/`status`/`priority` were pulled off `fact_sales` into their own `dim_order_flags` table instead of sitting as degenerate columns on the fact.
- **Role-playing dimension:** `fact_sales` connects to `dim_geo` twice — `ship_to_city_key` (active) and `bill_to_city_key` (inactive). A measure analyzing by bill-to city needs `USERELATIONSHIP()` to activate that second path.

## DAX

**Measures** (`_measures` table):

```dax
total_sales = SUM(fact_sales[line_total])

total_active_customers = DISTINCTCOUNT(dim_customer[customer_id])

base_total_customers = COUNT(dim_customer[customer_id])

avg_order_to_pay = AVERAGE(fact_order_process[order_to_pay])
```

**Calculated column** — order-to-cash cycle time, in days, on `fact_order_process`:

```dax
order_to_pay = DATEDIFF(fact_order_process[order_date], fact_order_process[pay_date], DAY)
```

**Row-level security** — role `regional_access`, filter on `dim_customer`:

```dax
dim_customer[region] = LOOKUPVALUE(security[region], security[user_email], USERPRINCIPALNAME())
```

Each viewer's email (`USERPRINCIPALNAME()`) is looked up against the `security` table to find their assigned region, and `dim_customer` is filtered to that region — so each user only sees data for customers in their own region.

## Files in This Repository

| File | Description |
|---|---|
| `dataset.xlsx` | Raw source data — 23 sheets simulating a fragmented, real-world data environment |
| `powerbi-data-modeling-project1.pbix` | Power BI file containing the full Power Query transformations, galaxy schema model, DAX measures, and RLS role |

## Tools Used

Power BI Desktop, Power Query, DAX, Excel
