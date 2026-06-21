# Sophisticated E-Commerce BI Data Modeling using Looker (LookML)

## 📌 Project Overview
This repository features an enterprise-grade **LookML (Looker Modeling Language)** data model designed to structure raw e-commerce data from Google BigQuery into a centralized, user-friendly semantic layer. 

In high-growth business environments, allowing non-technical stakeholders to build their own reports safely requires a robust data model. By defining the business logic, financial metrics (such as Net Profit and AOV), and table joins within LookML, we eliminate data discrepancies and ensure a single source of truth across all Looker dashboards.

---

## 🛠️ LookML Architecture & File Structure
* **The View File (`orders.view.lkml`):** Defines the dimensions (raw columns) and measures (calculated metrics like running revenue or profit margins) for business users.
* **The Model File (`ecommerce_operations.model.lkml`):** Handles the complex data relationships, structural joins, and global caching rules (datagroups) connecting Looker directly to Google BigQuery.

---

## 💻 1. The View File (`orders.view.lkml`)
*This file transforms raw transactional rows into clear business definitions inside Looker:*

```lookml
view: orders {
  sql_table_name: `ecommerce-data-warehouse.transactions.raw_sales_logs` ;;

  # --- DIMENSIONS (Attributes) ---
  dimension: order_id {
    type: string
    primary_key: yes
    sql: ${TABLE}.order_id ;;
    description: "The unique identifier for each customer transaction."
  }

  dimension: sku {
    type: string
    sql: ${TABLE}.sku ;;
  }

  dimension: marketplace_platform {
    type: string
    sql: ${TABLE}.marketplace_platform ;;
    description: "E-commerce channel: Shopify, Amazon FBA, or D2C Store."
  }

  # LookML Timeframes for Cohort & Velocity Tracking
  dimension_group: order_date {
    type: time
    timeframes: [raw, date, week, month, quarter, year]
    convert_tz: no
    datatype: date
    sql: ${TABLE}.order_timestamp ;;
  }

  # --- MEASURES (Aggregations / KPIs) ---
  measure: total_gross_revenue {
    type: sum
    sql: ${TABLE}.gross_sales_amount ;;
    value_format_name: usd
    description: "Total gross sales before fees and COGS are subtracted."
  }

  measure: total_logistics_cost {
    type: sum
    sql: COALESCE(${TABLE}.fba_fulfillment_fees, 0) + COALESCE(${TABLE}.long_term_storage_fees, 0) ;;
    value_format_name: usd
  }

  # Dynamic Net Profit Calculation Layer
  measure: true_net_profit {
    type: number
    sql: ${total_gross_revenue} - ${total_logistics_cost} ;;
    value_format_name: usd
    description: "Net profitability calculated safely across dynamic warehouse data."
  }

  # Conversion Matrix Metric
  measure: average_order_value {
    type: number
    sql: ${total_gross_revenue} / NULLIF(${order_count}, 0) ;;
    value_format_name: usd
  }

  measure: order_count {
    type: count
    drill_fields: [order_id, order_date_date, marketplace_platform, true_net_profit]
  }
}
