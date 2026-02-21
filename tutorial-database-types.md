# Choosing the Right Database for the Job: A Practical Guide for Data Students

*Published · Stat 386 Portfolio · ~5 min read*

---

## Introduction: Why Does Database Design Matter?

You have learned how to write a `SELECT` statement and maybe even joined a few tables together. But have you ever wondered *why* some databases look the way they do — or why your professor's example table looks nothing like the database schema at your internship?

The answer is that there is no single "correct" way to store data. The structure you choose depends on **what you plan to do with the data**: record transactions, run analytics, store unstructured content, or serve a machine-learning pipeline. Picking the wrong structure means slower queries, higher costs, and a lot of frustrated engineers.

This post walks you through the most common database paradigms you will encounter as a data student, explains the tradeoffs in plain language, and gives you a mental model for deciding which one fits a given problem.

---

## The Core Tension: Writes vs. Reads

Before diving into specific designs, it helps to understand the fundamental tradeoff that drives almost every database decision.

| Goal | Optimized for | Typical user |
|------|--------------|--------------|
| Recording transactions fast | **Writes** (inserts/updates) | Web app, point-of-sale system |
| Answering analytical questions fast | **Reads** (aggregations, scans) | Data analyst, BI dashboard |
| Storing raw, heterogeneous data cheaply | **Volume and flexibility** | Data engineer, ML pipeline |

Most database paradigms sit somewhere on this spectrum. Keep that tension in mind as you read through each type below.

---

## Relational (OLTP) Databases

**OLTP** stands for *Online Transaction Processing*. This is the classic relational database you learned about in your intro course — think PostgreSQL, MySQL, or SQLite.

OLTP databases are built around **normalization**, the process of eliminating redundancy by splitting data into many small, related tables. The standard target is *Third Normal Form (3NF)*, which ensures that every non-key column depends only on the primary key.

```sql
-- A normalized OLTP schema
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name        TEXT,
    email       TEXT UNIQUE
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    order_date  DATE,
    total       NUMERIC(10, 2)
);
```

**When to use it:** Anytime you need to guarantee data integrity for individual transactions — an e-commerce checkout, a hospital patient record, a banking ledger. ACID compliance (Atomicity, Consistency, Isolation, Durability) ensures that a payment either fully succeeds or fully fails; there is no in-between.

**The downside:** Joining 10+ normalized tables to answer "What was our revenue by region last quarter?" is slow and painful. That pain is what motivated the next two paradigms.

---

## Star Schema: Fact and Dimension Tables

When a team needs to run analytical queries on large volumes of historical data, they often move that data into a separate **data warehouse** structured as a *star schema*.

A star schema splits data into two kinds of tables:

- **Fact tables** — store numeric measurements of business events (sales amount, click counts, page views). They are wide, tall, and append-only.
- **Dimension tables** — store descriptive context (product name, customer region, date attributes). They are smaller and updated less frequently.

The schema gets its name because an entity-relationship diagram looks like a star, with the fact table at the center and dimension tables radiating outward.

```
         dim_date
            |
dim_product — fact_sales — dim_customer
            |
        dim_store
```

The math behind aggregations becomes straightforward:

$$\text{Revenue} = \sum_{i} \text{quantity}_i \times \text{unit\_price}_i$$

```sql
-- Typical star schema analytical query
SELECT
    d.month,
    c.region,
    SUM(f.quantity * f.unit_price) AS revenue
FROM fact_sales f
JOIN dim_date     d ON f.date_id     = d.date_id
JOIN dim_customer c ON f.customer_id = c.customer_id
GROUP BY d.month, c.region
ORDER BY d.month;
```

**When to use it:** BI dashboards, executive reporting, any scenario where analysts need fast aggregations over millions of rows. Tools like Snowflake, BigQuery, and Redshift are purpose-built for this pattern.

---

## NoSQL Databases: Three Flavors

Sometimes your data does not fit neatly into rows and columns. NoSQL databases trade rigid schema and ACID guarantees for flexibility and horizontal scalability. There are three main types worth knowing.

### Document Stores

Document databases (e.g., MongoDB, Firestore) store data as self-contained JSON-like documents. Each document can have a different structure, making them ideal for heterogeneous data.

```json
{
  "user_id": "u_8821",
  "name": "Jordan Lee",
  "preferences": {
    "theme": "dark",
    "notifications": ["email", "sms"]
  },
  "orders": [
    { "order_id": "o_001", "total": 49.99 },
    { "order_id": "o_002", "total": 120.00 }
  ]
}
```

This is the "just nest it" approach — great for user profiles, product catalogs, and CMS content.

### Key-Value Stores

Key-value databases (e.g., Redis, DynamoDB) are the simplest possible structure: a dictionary at massive scale. They are extremely fast for lookups by a single key.

**Best for:** session tokens, caching, leaderboards, real-time counters.

### Graph Databases

Graph databases (e.g., Neo4j) model data as nodes (entities) and edges (relationships). When the *connections between things* matter more than the things themselves, graphs win.

**Best for:** social networks, fraud detection, recommendation engines, knowledge graphs.

---

## Data Lakes: Raw Data at Rest

A **data lake** is a centralized repository — often on cloud object storage like Amazon S3 or Azure Data Lake — where you dump raw data in its original format: CSV files, JSON logs, images, Parquet files, database exports. Nothing is transformed before it lands.

The key insight is *schema-on-read*: you do not impose a structure when writing data, only when querying it. This makes ingestion cheap and fast, but querying requires more upfront work.

Data lakes are foundational to modern ML pipelines because they preserve the raw signal. You can always re-process raw data with a better model; you cannot un-aggregate data that was summarized on the way in.

---

## One Big Table (OBT)

The *One Big Table* pattern is exactly what it sounds like: denormalize everything into a single, flat, wide table. Every dimension value is pre-joined so that analysts never need to write a JOIN.

| order_id | customer_name | customer_region | product_name | category | order_date | quantity | unit_price |
|----------|--------------|----------------|-------------|----------|------------|----------|------------|
| o_001    | Jordan Lee   | West           | Widget Pro  | Hardware | 2024-03-01 | 2        | 24.99      |
| o_002    | Sam Rivera   | East           | Gadget Plus | Software | 2024-03-02 | 1        | 99.00      |

**Pros:** dead-simple queries, no JOINs, easy for non-technical stakeholders.

**Cons:** massive storage footprint, duplicated data, painful to update (changing a product's category means touching thousands of rows).

OBT works best as a *downstream export* — something generated from a star schema for a specific use case — rather than a source of truth.

---

## Choosing the Right Tool: A Quick Decision Guide

Work through these questions in order:

1. **Do you need transaction integrity?** → OLTP relational database.
2. **Do you need fast analytical aggregations on historical data?** → Star schema / data warehouse.
3. **Is your data unstructured or variable-schema?** → Document store (NoSQL).
4. **Do you need sub-millisecond lookups by a single key?** → Key-value store.
5. **Are relationships between entities the primary concern?** → Graph database.
6. **Do you need to store raw data cheaply at scale for future processing?** → Data lake.
7. **Do you need a simple, flat export for a dashboard or downstream consumer?** → OBT.

Real architectures often combine several of these. A common pattern is: OLTP → raw events to a data lake → transformed into a star schema warehouse → a specific OBT exported for a dashboard.

---

## Conclusion and Call to Action

Database design is not about memorizing definitions — it is about understanding tradeoffs. Every paradigm discussed here exists because some team hit a real limitation with the previous approach and needed a better fit for their specific workload.

Here is what to do next:

- **Sketch a schema** for a dataset you already work with. Which paradigm does it currently use? Which would be better?
- **Spin up a free tier** of [Snowflake](https://www.snowflake.com/en/data-cloud/try-snowflake/) or [MongoDB Atlas](https://www.mongodb.com/cloud/atlas/register) and load a small dataset using the pattern described above.
- **Read further** — the [Kimball Group's dimensional modeling resources](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/) are the gold standard for star schema design and are freely available online.

The best database is the one that matches your workload. Now you have the vocabulary to make that call deliberately.

---

*Have a question or want to share what you built? Reach out via the contact page.*
