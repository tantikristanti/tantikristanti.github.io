---
title: 'Learning to Embrace Imperfection: Building Resilient ETL Pipelines with CIQUAL'
date: 2026-06-02
collection: posts
permalink: /posts/2026/06/learning-to-embrace-imperfection-building-resilient-etl-pipelines-with-ciqual/
toc: true
tags:
  - etl
  - data-engineering
  - postgresql
  - ciqual
  - data-quality
---
Real-world datasets rarely match the clean, perfectly normalized schemas found in textbooks. If you've ever worked with production data, you've probably learned an uncomfortable truth: *structured* data is not necessarily *clean* data. Many ETL (Extract, Transform, Load) pipelines are built around an idealized view of the world. They assume source systems are complete, relationships are valid, and every foreign key points to an existing record. Under those assumptions, the workflow is straightforward: extract the data, transform it into the target schema, and load it into the database.

However, reality is rarely that cooperative. Datasets collected over years, or even decades, often contain missing references, placeholder values, deprecated identifiers, and inconsistencies spread across multiple files. These issues may remain hidden until the loading phase, when database constraints finally expose them. A single foreign key violation can cause an entire batch import to fail, preventing thousands of otherwise valid records from being loaded.

This article explores how to build resilient ETL pipelines for [CIQUAL](https://ciqual.anses.fr/#/cms/download/node/20), the French food composition database, without resorting to disabling foreign key constraints or dismissing data quality issues. Although CIQUAL is highly structured and well documented, it still contains many of the imperfections commonly found in large public datasets. Foods may reference missing food groups, nutritional composition records may point to foods that were never imported, and placeholder codes can appear where valid relationships are expected. Loading it directly into PostgreSQL with strict foreign key constraints quickly exposes orphan records, missing references, and inconsistencies across related tables. These problems are not unique to CIQUAL. They appear in open datasets, enterprise systems, government records, and third-party data feeds alike. The challenge is not simply loading data into a database. It's reconciling imperfect source data with the strict integrity guarantees of a relational schema.

In this article, we'll build a resilient ETL pipeline that embraces this reality. Instead of disabling foreign key constraints or ignoring data quality issues, we'll introduce a validation layer between parsing and loading. Raw records will be staged, relationships will be verified before insertion, and problematic rows will be isolated for review while valid data continues through the pipeline.

---

## Introduction

Traditional ETL pipelines are built around a simple assumption: source data is complete, consistent, and ready to fit neatly into a relational database. The process is straightforward: extract the data, transform it into the target schema, and load it into the destination system. However, when the data contains broken relationships or missing references, database constraints can quickly expose those issues. A single foreign key violation may be enough to halt the entire load process, leaving valid records stranded alongside problematic ones.

A more resilient approach treats data quality as an expected challenge rather than an exceptional failure. Instead of loading records directly into constrained tables, the pipeline introduces a validation layer between parsing and persistence. Raw data is staged first, relationships are verified before insertion, and records that fail validation are isolated for review. Valid data continues through the pipeline, while inconsistencies are logged and reconciled separately. This allows the database to maintain strict referential integrity without sacrificing the ability to ingest imperfect datasets.

In this guide, we'll apply that approach to [CIQUAL](https://ciqual.anses.fr/#/cms/download/node/20)[[1][1]], the French food composition database. Although CIQUAL is well-structured and extensively documented, it still exhibits many of the data quality issues. Foods may reference group codes that no longer exist, nutrient composition records may point to missing foods, and placeholder values such as `00`, `0000`, or `000000` may appear where valid identifiers are expected.

Rather than weakening database constraints or ignoring these inconsistencies, we'll build an ETL pipeline capable of detecting, documenting, and handling them systematically. Along the way, we'll learn how to:

- Parse CIQUAL XML files into staging tables.
- Validate foreign key relationships before loading data.
- Handle placeholder and unknown category codes.
- Detect and log orphan records for later review.
- Load validated data into PostgreSQL while preserving referential integrity.
- Generate reconciliation reports that make data quality issues visible.

Ultimately, the objective is not simply to load data into PostgreSQL. It is to build an ETL process that can accommodate the imperfections of real-world data while preserving the consistency guarantees that make relational databases reliable in the first place.

---

## Why CIQUAL? A Dataset That Reflects Reality

Before exploring the pipeline architecture, it's worth understanding why CIQUAL is such an effective dataset for studying data engineering challenges. Maintained by the French Agency for Food, Environmental and Occupational Health & Safety (ANSES), CIQUAL contains nutritional information for approximately 3,500 foods and 74 nutritional components. The data is distributed in a structured XML format with clearly defined relationships between foods, food groups, nutrients, and reference sources.

The dataset is not broken. The codes `00`, `0000`, `000000` are documented as placeholders meaning *unknown hierarchy* or *not applicable*. The problem is not the data; it is the assumption that every foreign key must point to an existing row. A resilient pipeline must understand and respect the dataset’s documented semantics.

---

## Use Cases That Break Naïve ETL Pipelines

Before designing a resilient ETL pipeline, it's useful to understand where traditional approaches fail. The following examples come directly from CIQUAL and illustrate how seemingly well-structured datasets can contain relationships that violate the assumptions of a straightforward extract-transform-load workflow.

These examples are not signs of a broken dataset. Rather, they demonstrate the kinds of inconsistencies, edge cases, and historical artifacts that naturally emerge in long-lived information systems. The most instructive data-quality issue encountered during the CIQUAL import process involved a chain of foreign key failures spanning multiple tables.

Consider the following food record:

```xml
<ALIM>
    <alim_code> 24999 </alim_code>
    <alim_nom_fr> Dessert (aliment moyen) </alim_nom_fr>
    <alim_nom_eng> Dessert (average) </alim_nom_eng>
    <alim_nom_sci missing=" " />
    <alim_grp_code> 00 </alim_grp_code>
    <alim_ssgrp_code> 0000 </alim_ssgrp_code>
    <alim_ssssgrp_code> 000000 </alim_ssssgrp_code>
    <facteur_Jones> 6.25 </facteur_Jones>
</ALIM>
```

When loading the data into PostgreSQL, this record fails validation because no corresponding entry exists in the `food_groups` table for the hierarchy (`00`, `0000`, `000000`). A naïve ETL pipeline rejects the food record and moves on. Unfortunately, the failure does not stop there. The `composition` dataset contains nutritional measurements linked to the same food identifier: `alim_code = 24999`.

Once the food record has been excluded from the foods table, those composition records lose their parent reference and trigger a second foreign key violation during loading. What initially appears to be a single missing relationship therefore cascades through the entire schema: `food_groups → foods → composition`.

This example illustrates a broader lesson about real-world data engineering. Data quality issues rarely exist in isolation. A missing reference in one table often propagates downstream, creating additional failures that are symptoms rather than root causes.

The objective of a resilient ETL pipeline is not simply to detect these failures. It is to identify the original source of the inconsistency, reconcile it appropriately, and prevent valid downstream data from being discarded unnecessarily.

---

## Traditional ETL vs. a Resilient Pipeline

The examples from the previous section highlight a common weakness in traditional ETL workflows: they assume the source data is already consistent.

In a typical pipeline, records are extracted from the source system, transformed into the target schema, and inserted directly into relational tables protected by foreign key constraints. This approach works well when the source data is perfectly aligned with the database model. However, when a record contains a missing reference or an unexpected placeholder value, the loading process can fail immediately.

In the case of CIQUAL, a food record associated with the placeholder hierarchy `(00, 0000, 000000)` triggers a foreign key violation because no corresponding entry exists in the `food_groups` table. Once that food record is rejected, any composition records linked to the same `alim_code` become orphaned as well. What begins as a single missing relationship quickly propagates through the rest of the schema.

The result is a brittle pipeline in which a small number of problematic records can prevent large amounts of valid data from being loaded.

A more resilient approach treats data validation as a dedicated stage rather than an afterthought. Instead of loading records directly into constrained tables, the pipeline is divided into three logical layers ([Figure 1](#fig1)):

<figure id="fig1">
  <img src="/images/posts/2026-06-02-learning-to-embrace-imperfection-building-resilient-etl-pipelines-with-ciqual/1-logical-layers.png" alt="ETL pipeline architecture" width="100%">
  <figcaption>Figure 1: Three‑layer staging architecture for CIQUAL ETL.</figcaption>
</figure>

The first layer is responsible for ingesting the raw XML data exactly as it appears in the source files. No assumptions are made about data quality, and no foreign key constraints are enforced at this stage.

The second layer performs validation and reconciliation. Placeholder values are normalised, relationships are checked, and records with missing references are identified. Rather than causing the pipeline to fail, problematic records are logged for later review.

Only after validation records are loaded into the final relational schema. Because invalid relationships have already been addressed, foreign key constraints can remain fully enabled in the production tables.

This design shifts the pipeline's objective from rejection to reconciliation. Instead of discarding an entire batch, valid data continues through the process while inconsistencies are documented and isolated.

The result is a database that preserves referential integrity without sacrificing resilience. Valid records are loaded successfully, data-quality issues remain visible, and the pipeline continues to operate even when the source data is imperfect.

---

## Building the Pipeline

With the architecture defined, it's time to implement the pipeline. The goal is to load CIQUAL into PostgreSQL while preserving referential integrity and preventing data-quality issues from causing the entire import process to fail.

The examples in this article are based on the same issue discussed earlier: *food* records such as `alim_code = 24999` reference placeholder that do not exist in the *food groups* hierarchy. A resilient pipeline should detect these inconsistencies, record them for review, and continue loading valid data.

---

### Prerequisites

The implementation requires the following components:

- Python 3.9+
- PostgreSQL 14+ [[2]]
- Docker (recommended) [[3]]
- The PostgreSQL pgvector extension (optional if semantic search is planned later) [[4]]

---

### Project Setup

Create a virtual environment in a project folder:

```bash

cd ciqual-food

python -m venv .venv
# or with uv:
uv venv .venv

source .venv/bin/activate   # Linux/macOS
# .venv\Scripts\activate   on Windows

```

Install the required dependencies:

```bash
pip install pgvector pyarrow psycopg2-binary sqlalchemy lxml python-dotenv matplotlib seaborn pandas numpy

# or with uv:
uv add install pgvector pyarrow psycopg2-binary sqlalchemy lxml python-dotenv matplotlib seaborn pandas numpy
```

---

### Environment Configuration

Create a `.env` file in the project root:

```env
DB_NAME=ciqual_db
DB_USER=ciqual
DB_PASSWORD=ciqual
DB_HOST=localhost
DB_PORT=5432
```

Load these values in Python:

```python
from dotenv import load_dotenv
import os

load_dotenv()

DB_NAME = os.getenv("DB_NAME")
DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")
DB_HOST = os.getenv("DB_HOST")
DB_PORT = os.getenv("DB_PORT")
```

---

### Starting PostgreSQL

The easiest way to create a local PostgreSQL instance is with Docker:

```bash
docker run -d --name pgvector \
  -e POSTGRES_USER=ciqual \
  -e POSTGRES_PASSWORD=ciqual \
  -e POSTGRES_DB=ciqual_db \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

Verify that the database is running:

```bash
docker ps
```

Connect to PostgreSQL container:

```bash
docker exec -it pgvector psql -U ciqual -d ciqual_db
```

---

### Step 1: Create Raw Staging Tables

The first layer of the pipeline stores data exactly as it appears in the XML files. At this stage, no primary keys, foreign keys, or unique constraints are enforced.

To keep the implementation manageable, the examples below focus on creating the `staging_foods` table.

```sql
self.cur.execute("""
    CREATE TABLE IF NOT EXISTS staging_foods (
        alim_code INTEGER,
        alim_nom_eng TEXT,
        alim_nom_fr TEXT,
        alim_nom_sci TEXT,
        facteur_jones NUMERIC(5,3),
        alim_grp_code TEXT,
        alim_ssgrp_code TEXT,
        alim_ssssgrp_code TEXT
    )
""")

self.conn.commit()

```

The remaining staging tables follow the same design principle: mirror the XML structure as closely as possible and avoid foreign key constraints or validation logic at this stage. The following diagram shows the complete staging layer that will be populated during ingestion ([Figure 2](#fig2)).

<figure id="fig2">
  <img src="/images/posts/2026-06-02-learning-to-embrace-imperfection-building-resilient-etl-pipelines-with-ciqual/2-staging-tables.png" alt="CIQUAL staging tables" width="100%">
  <figcaption>Figure 2: CIQUAL staging tables.</figcaption>
</figure>

The purpose of staging is simple: ingest first, validate later. This prevents malformed records from causing failures during parsing.

---

### Step 2: Create Clean Relational Tables

The final schema enforces referential integrity. Unlike the staging layer, these tables are intentionally strict.

```sql
self.cur.execute("""
    CREATE TABLE IF NOT EXISTS food_groups (
        alim_grp_code TEXT NOT NULL,
        alim_ssgrp_code TEXT NOT NULL,
        alim_ssssgrp_code TEXT NOT NULL,
        alim_grp_nom_eng TEXT,
        alim_ssgrp_nom_eng TEXT,
        alim_ssssgrp_nom_eng TEXT,
        alim_grp_nom_fr TEXT,
        alim_ssgrp_nom_fr TEXT,
        alim_ssssgrp_nom_fr TEXT,
        PRIMARY KEY (alim_grp_code, alim_ssgrp_code, alim_ssssgrp_code)
    )
""")

self.cur.execute("""
    CREATE TABLE IF NOT EXISTS foods (
        alim_code INTEGER PRIMARY KEY,
        alim_nom_eng TEXT,
        alim_nom_fr TEXT,
        alim_nom_sci TEXT,
        facteur_jones NUMERIC(5,3),
        alim_grp_code TEXT,
        alim_ssgrp_code TEXT,
        alim_ssssgrp_code TEXT,
        image_front_url TEXT,
        image_small_url TEXT,
        image_thumb_url TEXT,
        off_product_name TEXT,
        off_brand TEXT,
        off_last_updated TIMESTAMP,
        FOREIGN KEY (alim_grp_code, alim_ssgrp_code, alim_ssssgrp_code)
        REFERENCES food_groups(alim_grp_code, alim_ssgrp_code, alim_ssssgrp_code)
    )
""")

self.conn.commit()

```

The SQL statements above define two of the tables used in the pipeline. The remaining tables can be created following the same pattern. [Figure 3](#fig3) presents the complete entity–relationship (ER) diagram, showing the final schema and the relationships between all tables.

<figure id="fig3">
  <img src="/images/posts/2026-06-02-learning-to-embrace-imperfection-building-resilient-etl-pipelines-with-ciqual/3-ciqual-er-diagram.png" alt="CIQUAL ER-Diagram" width="100%">
  <figcaption>Figure 3: CIQUAL ER-Diagram.</figcaption>
</figure>

---

### Step 3: Normalize Placeholder Codes

Before validation, placeholder hierarchies should be standardized by inserting a special `UNKNOWN` group to absorb placeholder codes.

```sql
self.cur.execute("""
    INSERT INTO food_groups
        (alim_grp_code, alim_ssgrp_code, alim_ssssgrp_code,
        alim_grp_nom_eng, alim_ssgrp_nom_eng, alim_ssssgrp_nom_eng,
        alim_grp_nom_fr, alim_ssgrp_nom_fr, alim_ssssgrp_nom_fr)
    VALUES ('UNKNOWN', 'UNKNOWN', 'UNKNOWN',
            'Unknown', 'Unknown', 'Unknown',
            'Inconnu', 'Inconnu', 'Inconnu')
    ON CONFLICT (alim_grp_code, alim_ssgrp_code, alim_ssssgrp_code) DO NOTHING
""")
```

---

### Step 4: Validate Foreign Key Relationships

Build a set of valid food groups and validate each food before insertion. Then map placeholder values while loading clean tables with validation. This allows food records such as `alim_code = 24999` to remain part of the dataset instead of being discarded.Instead of crashing, invalid rows are isolated for review.

The function returns the list of successfully inserted foods, which is then used to filter composition records.

```python
def insert_foods(self, food_groups: List[FoodGroup], foods: List[Food]) -> List[Food]:
  
    # Build set of valid group keys from food_groups
    valid_groups: Set[Tuple[str, str, str]] = {
        (fg.alim_grp_code, fg.alim_ssgrp_code, fg.alim_ssssgrp_code)
        for fg in food_groups
    }
    # Ensure UNKNOWN is in the set (inserted by insert_unknown_food_groups)
    if ('UNKNOWN', 'UNKNOWN', 'UNKNOWN') not in valid_groups:
        valid_groups.add(('UNKNOWN', 'UNKNOWN', 'UNKNOWN'))

    valid_foods = []
    orphan_foods = []

    for food in foods:
        # Normalize placeholder codes to UNKNOWN
        grp = 'UNKNOWN' if food.alim_grp_code in ('00', '') else food.alim_grp_code
        ssgrp = 'UNKNOWN' if food.alim_ssgrp_code in ('0000', '') else food.alim_ssgrp_code
        ssssgrp = 'UNKNOWN' if food.alim_ssssgrp_code in ('000000', '') else food.alim_ssssgrp_code

        key = (grp, ssgrp, ssssgrp)
        if key in valid_groups:
            # Update food with normalized codes (optional, but keeps consistency)
            food.alim_grp_code = grp
            food.alim_ssgrp_code = ssgrp
            food.alim_ssssgrp_code = ssssgrp
            valid_foods.append(food)
        else:
            orphan_foods.append(food)

    # Insert valid foods
    if valid_foods:
        data = [(f.alim_code, f.alim_nom_eng, f.alim_nom_fr, f.alim_nom_sci, f.facteur_jones, f.alim_grp_code, f.alim_ssgrp_code, f.alim_ssssgrp_code) for f in valid_foods]

        execute_values(self.cur, """
            INSERT INTO foods
            (alim_code, alim_nom_eng, alim_nom_fr, alim_nom_sci,
            facteur_jones, alim_grp_code, alim_ssgrp_code, alim_ssssgrp_code)
            VALUES %s
            ON CONFLICT (alim_code) DO NOTHING
        """, data)
    return valid_foods
```

---

### Step 5: Validate Composition Records

Before loading nutritional composition data, verify that every referenced food exists.

```python
def insert_composition(self, valid_foods: List[Food], compositions: List[Composition]) -> None:

    # Build set of alim_codes from valid foods
    food_codes = {f.alim_code for f in valid_foods}

    clean_compositions = []
    orphan_compositions = []

    for comp in compositions:
        if comp.alim_code in food_codes:
            clean_compositions.append(comp)
        else:
            orphan_compositions.append(comp)

    if clean_compositions:
        data = [(c.alim_code, c.const_code, c. teneur, c.min_val, c.max_val, c.code_confiance, c.source_code) for c in clean_compositions]

        execute_values(self.cur, """
            INSERT INTO composition
            (alim_code, const_code, teneur, min_val, max_val, code_confiance, source_code)
            VALUES %s
            """, data)
```

---

### Step 6: Log Orphan Records

Create an audit table:

```sql
self.cur.execute("""
    CREATE TABLE IF NOT EXISTS orphan_foods (
        alim_code INTEGER,
        reason TEXT,
        created_at TIMESTAMP DEFAULT NOW()
    )
""")

self.cur.execute("""
    CREATE TABLE IF NOT EXISTS orphan_compositions (
        alim_code INTEGER,
        const_code INTEGER,
        reason TEXT,
        created_at TIMESTAMP DEFAULT NOW()
    )
    """)

self.conn.commit()

```

Store problematic records (orphan data):

```python
# Log orphan foods
if orphan_foods:
    for food in orphan_foods:
        self.cur.execute("""
            INSERT INTO orphan_foods (alim_code, reason)
            VALUES (%s, %s)
        """, (food.alim_code, f"Missing group ({food.alim_grp_code}, {food.alim_ssgrp_code}, {food.alim_ssssgrp_code})"))

# Log orphan compositions
if orphan_compositions:
    for comp in orphan_compositions:
        self.cur.execute("""
            INSERT INTO orphan_compositions (alim_code, const_code, reason)
            VALUES (%s, %s, %s)
        """, (comp.alim_code, comp.const_code, f"Missing food code {comp.alim_code} in foods table"))

```

The pipeline continues even when data quality issues are encountered.

---

### Step 7: Generate a Reconciliation Report

After running the validation pipeline, a reconciliation report is generated to show how many records were successfully loaded and how many were rejected due to missing references.

```
============================================================
RECONCILIATION REPORT
============================================================
Foods:
  - Source (staging):        3,484
  - Loaded (clean):          2,035
  - Orphan (logged):         1,449
  - Rejected rate:           41.6%

Composition rows:
  - Source (staging):        257,816
  - Loaded (clean):          150,590
  - Orphan (logged):         107,226
  - Rejected rate:           41.6%

Top 5 reasons for orphan foods:
  - Missing group (04, 0406, 000000): 108
  - Missing group (07, 0709, 000000): 94
  - Missing group (06, 0601, 000000): 88
  - Missing group (07, 0706, 000000): 80
  - Missing group (04, 0409, 000000): 75

Top 5 reasons for orphan compositions:
  - Missing food code 1032 in foods table: 74
  - Missing food code 1034 in foods table: 74
  - Missing food code 3000 in foods table: 74
  - Missing food code 3002 in foods table: 74
  - Missing food code 4000 in foods table: 74
```

---

**Understanding the Reconciliation Report** 

The CIQUAL 2025 dataset contains 3,484 food records. Of these, 2,035 foods were successfully loaded into the clean relational schema, while 1,449 were classified as orphan records and logged for further investigation. This represents a rejection rate of 41.6%([Figure 4](#fig4): Top‑left, Foods pie).

In other words, only 58.4% of foods could be linked to a valid food groups and inserted into the final database. The remaining foods referenced group-code combinations that were not present in the `food_groups` table after validation.

The same pattern appears in the nutritional composition data ([Figure 4](#fig4): Top‑right, Composition pie). Out of 257,816 composition records, 150,590 were successfully loaded, while 107,226 were rejected because their associated food records were not present in the clean `foods` table. Since composition records depend on foods through a foreign key relationship, any orphan food automatically creates orphan composition records.

The charts clearly illustrate this cascading effect. When a food record cannot be loaded because its group reference is invalid, all composition records associated with that food are also excluded from the clean schema.

<figure id="fig4">
  <img src="/images/posts/2026-06-02-learning-to-embrace-imperfection-building-resilient-etl-pipelines-with-ciqual/4-reconciliation-charts.png" alt="Reconciliation charts" width="100%">
  <figcaption>Figure 4: CIQUAL Reconciliation charts.</figcaption>
</figure>

---

**Most Common Reasons for Orphan Foods**
The most frequent causes of orphan food records are missing food groups. The five most common group combinations are:

- (04, 0406, 000000) → 108 foods
- (07, 0709, 000000) → 94 foods
- (06, 0601, 000000) → 88 foods
- (07, 0706, 000000) → 80 foods
- (04, 0409, 000000) → 75 foods

These combinations account for a substantial portion of the rejected food records. These records could not be matched to a corresponding entry in the loaded `food_groups` table and therefore were classified as orphans. Their presence suggests that additional investigation is needed to determine whether these group codes were omitted during parsing, represent deprecated classifications, or correspond to hierarchy levels that require special handling.

---

**Most Common Reasons for Orphan Composition Records**

The composition report shows a different pattern. The most common orphan reasons are:

- Missing food code 1032 in foods
- Missing food code 1034 in foods
- Missing food code 3000 in foods
- Missing food code 3002 in foods
- Missing food code 4000 in foods

Each of these missing food codes appears exactly ***74 times***.

This number is significant because CIQUAL contains 74 nutritional components. A food with a complete nutrient profile typically generates one composition record per component. Therefore, a single missing food can produce dozens of orphan composition records.

The report reveals an important characteristic of relational datasets: data quality issues often propagate through dependent tables. A missing reference in a parent table can create a much larger number of downstream inconsistencies.

---

## Conclusion

Building a resilient ETL pipeline for CIQUAL taught us a fundamental lesson about real‑world data engineering: **perfectly clean datasets are rare, but that does not mean they are broken**. The placeholders `00`, `0000`, and `000000` are not errors; they are the dataset’s honest way of saying ***unknown***. The missing group references and orphan composition rows are not signs of a flawed source. They are the natural by‑products of a living, evolving knowledge base maintained over decades.

Traditional ETL pipelines assume the source data is already consistent. When that assumption fails, the pipeline crashes, and valuable data is lost. The three‑layer staging architecture we built turns that fragility into resilience:

1. **Raw staging** preserves the original data exactly as parsed, no information is discarded.

2. **Validation** normalises placeholders, checks foreign keys, and logs orphans, problems are documented, not silenced.

3. **Clean relational tables** enforce strict referential integrity, but only after the data has been reconciled.

This design transforms foreign key errors from roadblocks into actionable data quality signals. The reconciliation report tells us exactly what was loaded, what was rejected, and why. 

The reconciliation report reveals that 2,035 food records (58.4%) were successfully loaded into the clean schema, while 1,449 records (41.6%) were classified as orphans because they could not be linked to a valid food groups. At the composition level, 150,590 records were successfully imported, whereas 107,226 records were orphaned as a cascade.

These figures are not a failure. They are a transparent audit of the dataset’s internal consistency. With this report, we can decide whether to enrich the `food_groups` table, adjust normalisation rules, or accept the loss as a documented trade‑off.

More importantly, this pattern is not limited to CIQUAL. It applies to any dataset where foreign keys are aspirational. The principles are universal:

- **Stage first, validate later**: never let parsing failures block ingestion.
- **Log, don’t crash**: every rejected row is a data quality signal, not a fatal error.
- **Reconcile**: a reconciliation report turns ETL into a monitoring tool.

---

## References

1. Anses. 2025. **Ciqual French food composition table**, https://doi.org/10.5281/zenodo.17550133".
2. The PostgreSQL Global Development Group. (2026). **PostgreSQL: The World&#39;s Most Advanced Open Source Relational Database**.
3. Docker Inc. (2026). **Docker Documentation**.
4. pgvector. (2026). **pgvector**. 

---

[1]: https://ciqual.anses.fr/#/cms/download/node/20
[2]: https://www.postgresql.org/
[3]: https://docs.docker.com/
[4]: https://github.com/pgvector/pgvector
