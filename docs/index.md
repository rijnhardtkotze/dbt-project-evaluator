# dbt_project_evaluator

This package highlights areas of a dbt project that are misaligned with dbt Labs' best practices.
Specifically, this package tests for:

1. __[Modeling](rules/modeling)__ - your dbt DAG for modeling best practices
2. __[Testing](rules/testing)__ - your models for testing best practices
3. __[Documentation](rules/documentation)__ - your models for documentation best practices
4. __[Structure](rules/structure)__ - your dbt project for file structure and naming best practices
5. __[Performance](rules/performance)__ - your model materializations for performance best practices
6. __[Governance](rules/governance)__ - your model governance feature best practices


In addition to tests, this package creates the model `int_all_dag_relationships` which holds information about your DAG in a tabular format and can be queried using SQL in your Warehouse.

Currently, the following adapters are supported:

- BigQuery
- Databricks/Spark
- PostgreSQL
- Redshift
- Snowflake
- DuckDB
- Trino (tested with Iceberg connector)
- AWS Athena (tested manually)
- Greenplum (tested manually)
- ClickHouse (tested manually)

## Using This Package

### Cloning via dbt Package Hub
  
Check [dbt Hub](https://hub.getdbt.com/dbt-labs/dbt_project_evaluator/latest/) for the latest installation instructions, or [read the docs](https://docs.getdbt.com/docs/package-management) for more information on installing packages.

### Additional setup for Databricks/Spark/DuckDB/Redshift

In your `dbt_project.yml`, add the following config:

```yaml title="dbt_project.yml"
dispatch:
  - macro_namespace: dbt
    search_order: ['dbt_project_evaluator', 'dbt']
```

This is required because the project currently overrides a small number of dbt core macros in order to ensure the project can run across the listed adapters. The overridden macros are in the [cross_db_shim directory](https://github.com/dbt-labs/dbt-project-evaluator/tree/main/macros/cross_db_shim/).
  
### How It Works

This package will:

1. Parse your [graph](https://docs.getdbt.com/reference/dbt-jinja-functions/graph) object and write it into your warehouse as a series of models (see [models/marts/core](https://github.com/dbt-labs/dbt-project-evaluator/tree/main/models/marts/core))
2. Create another series of models that each represent one type of misalignment in your project (below you can find a full list of each misalignment and its accompanying model)
3. Test those models to alert you to the presence of the misalignment

Once you've installed the package, all you have to do is run a `dbt build --select package:dbt_project_evaluator`

Each test warning indicates the presence of a type of misalignment. To troubleshoot a misalignment:

1. Locate the related documentation
2. Query the associated model to find the specific instances of the issue within your project or set up an [`on-run-end` hook](https://docs.getdbt.com/reference/project-configs/on-run-start-on-run-end) to display the rules violations in the dbt logs (see [displaying violations in the logs](customization/issues-in-log.md))
3. Either fix the issue(s) or [customize](customization/exceptions.md) the package to exclude them

----

# Project Structure Best Practices

## Overview

**What is this guide?**: A comprehensive reference for organizing your dbt project with clear directory structures and naming conventions that support maintainability and pass evaluator checks.

**Who is this for?**: dbt analytics engineers and data teams looking to establish or improve their project organization.

**Time to read**: 10-15 minutes

---

## Why Structure Matters

A well-organized dbt project makes it easier to:
- **Navigate and understand** the data transformation flow
- **Collaborate** with team members who can quickly locate models
- **Maintain** code as your project grows
- **Pass validation** from dbt-project-evaluator rules

The dbt-project-evaluator includes specific [structure rules](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#structure) that check for common organizational issues. Following these best practices helps you avoid those issues from the start.

---

## Canonical Directory Structure

Here's the recommended directory layout for a dbt project:

```
my_dbt_project/
├── dbt_project.yml
├── packages.yml
├── models/
│   ├── staging/
│   │   ├── stripe/
│   │   │   ├── _stripe__sources.yml
│   │   │   ├── _stripe__models.yml
│   │   │   ├── stg_stripe__customers.sql
│   │   │   ├── stg_stripe__invoices.sql
│   │   │   └── stg_stripe__payments.sql
│   │   ├── salesforce/
│   │   │   ├── _salesforce__sources.yml
│   │   │   ├── _salesforce__models.yml
│   │   │   ├── stg_salesforce__accounts.sql
│   │   │   ├── stg_salesforce__opportunities.sql
│   │   │   └── stg_salesforce__users.sql
│   │   └── _staging__models.yml
│   ├── intermediate/
│   │   ├── finance/
│   │   │   ├── _int_finance__models.yml
│   │   │   ├── int_payments_pivoted_to_orders.sql
│   │   │   └── int_invoice_line_items.sql
│   │   └── marketing/
│   │       ├── _int_marketing__models.yml
│   │       └── int_campaign_attribution.sql
│   ├── marts/
│   │   ├── finance/
│   │   │   ├── _finance__models.yml
│   │   │   ├── orders.sql
│   │   │   ├── revenue_by_customer.sql
│   │   │   └── monthly_revenue.sql
│   │   ├── marketing/
│   │   │   ├── _marketing__models.yml
│   │   │   ├── customers.sql
│   │   │   ├── campaign_performance.sql
│   │   │   └── customer_acquisition_cost.sql
│   │   └── core/
│   │       ├── _core__models.yml
│   │       ├── dim_customers.sql
│   │       ├── dim_products.sql
│   │       └── fct_orders.sql
│   └── metrics/
│       └── metrics.yml
├── tests/
│   └── generic/
│       └── test_valid_discount.sql
├── macros/
│   ├── _macros.yml
│   ├── cents_to_dollars.sql
│   └── generate_schema_name.sql
├── seeds/
│   └── country_codes.csv
├── snapshots/
│   └── snap_orders.sql
└── analyses/
    └── exploratory_revenue_analysis.sql
```

---

## Layer-by-Layer Breakdown

### Staging Layer (`models/staging/`)

**Purpose**: Clean and standardize raw source data. One model per source table.

**Structure**:
- Organize by source system: `staging/<source_name>/`
- Each source gets its own subdirectory

**Naming Convention**:
- Models: `stg_<source>__<entity>.sql`
- Sources YML: `_<source>__sources.yml` (underscore prefix)
- Models YML: `_<source>__models.yml`

**Examples**:
```
✅ GOOD:
  staging/stripe/stg_stripe__customers.sql
  staging/stripe/stg_stripe__payments.sql
  
❌ BAD:
  staging/stripe_customers.sql (missing stg_ prefix)
  staging/customer_data.sql (unclear source)
  staging/stripe_stg_customers.sql (prefix in wrong place)
```

**Why this structure?**
- **Prefixes** (`stg_`, `_`) make model types immediately clear
- **Double underscores** (`__`) separate the source from the entity
- **Source subdirectories** prevent folder clutter as you add more sources
- **Triggers evaluator rule**: [Missing Primary Key Tests](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#missing-primary-key-tests) - staging models should have unique/not_null tests on their primary key

### Intermediate Layer (`models/intermediate/`)

**Purpose**: Business logic transformations that don't directly serve end users. These are "in between" staging and marts.

**Structure**:
- Organize by business domain or process: `intermediate/<domain>/`
- Can be optional for smaller projects

**Naming Convention**:
- Models: `int_<entity>_<verb/action>.sql` or `int_<domain>__<entity>.sql`
- Schema files: `_int_<domain>__models.yml`

**Examples**:
```
✅ GOOD:
  intermediate/finance/int_payments_pivoted_to_orders.sql
  intermediate/finance/int_invoice_line_items.sql
  intermediate/marketing/int_campaign_attribution.sql
  
❌ BAD:
  intermediate/payment_pivot.sql (missing int_ prefix)
  intermediate/int_pivoted_payments.sql (unclear what it's pivoting to)
  models/int_something.sql (not in intermediate folder)
```

**Why this structure?**
- **Prefixes** (`int_`) distinguish intermediate models from marts
- **Descriptive suffixes** (`_pivoted_to_orders`) explain the transformation
- **Domain folders** group related transformations together
- **Triggers evaluator rule**: [Undocumented Models](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#undocumented-models) - intermediate models should have descriptions

### Marts Layer (`models/marts/`)

**Purpose**: Business-defined entities ready for end users and BI tools.

**Structure**:
- Organize by business domain: `marts/<domain>/`
- Common domains: `finance/`, `marketing/`, `product/`, `core/`

**Naming Convention**:
- Models: Simple business entity names like `customers.sql`, `orders.sql`, or dimensional modeling: `dim_customers.sql`, `fct_orders.sql`
- Schema files: `_<domain>__models.yml`

**Examples**:
```
✅ GOOD:
  marts/finance/orders.sql
  marts/finance/revenue_by_customer.sql
  marts/marketing/customers.sql
  marts/core/dim_customers.sql
  marts/core/fct_orders.sql
  
❌ BAD:
  marts/mart_orders.sql (no mart_ prefix needed)
  marts/final_orders.sql (vague, not business-oriented)
  marts/orders_mart.sql (suffix instead of folder)
  models/orders.sql (not in marts subdirectory)
```

**Why this structure?**
- **Clear names** match how business users think about the data
- **Domain folders** organize by business area, not technical structure
- **No prefix required** - location in `marts/` is sufficient
- **Triggers evaluator rule**: [Marts or Staging Models Dependent on Other Models](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#models-dependent-on-downstream-models) - marts shouldn't depend on other marts

---

## Schema Files (YML) Placement

There are two common approaches for schema files:

### Approach 1: Per-Folder Schema Files (Recommended)

Place a schema file in each subdirectory:

```
staging/stripe/
  ├── _stripe__sources.yml      # Source definitions
  ├── _stripe__models.yml        # Model documentation & tests
  ├── stg_stripe__customers.sql
  └── stg_stripe__payments.sql
```

**Pros**:
- Schema file is always next to the models it documents
- Easy to find and maintain
- Clear ownership

### Approach 2: Domain-Level Schema Files

One schema file per domain:

```
marts/finance/
  ├── _finance__models.yml       # Documents all finance models
  ├── orders.sql
  ├── revenue_by_customer.sql
  └── monthly_revenue.sql
```

**Pros**:
- Fewer files to manage
- All documentation in one place

**Best Practice**: Use underscore prefix (`_stripe__models.yml`) to distinguish schema files from model files at a glance.

**Triggers evaluator rule**: [Undocumented Models](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#undocumented-models) - all public models should have descriptions in schema YML files.

---

## Naming Conventions Summary

| Layer | Prefix | Pattern | Example |
|-------|--------|---------|---------|
| **Staging** | `stg_` | `stg_<source>__<entity>` | `stg_stripe__customers.sql` |
| **Intermediate** | `int_` | `int_<entity>_<verb>` or `int_<domain>__<entity>` | `int_payments_pivoted_to_orders.sql` |
| **Marts** | none | `<entity>` or `dim_<entity>` / `fct_<entity>` | `customers.sql`, `dim_customers.sql` |
| **Schema files** | `_` | `_<source>__models.yml` | `_stripe__models.yml` |

### Do's and Don'ts

**DO:**
- ✅ Use consistent prefixes (`stg_`, `int_`)
- ✅ Use double underscores (`__`) to separate source from entity
- ✅ Group by source system (staging) or business domain (marts)
- ✅ Give marts simple, business-friendly names
- ✅ Document all models in schema YML files

**DON'T:**
- ❌ Mix staging and marts in the same folder
- ❌ Use abbreviations that aren't universally understood
- ❌ Create deeply nested folder structures (3+ levels)
- ❌ Omit prefixes in staging/intermediate layers
- ❌ Reference marts from other marts (creates tangled dependencies)

---

## Bad Structure Example: How to Refactor

### ❌ Before: Poor Organization

```
models/
├── stripe_customers.sql          # No stg_ prefix
├── salesforce_accounts.sql       # No stg_ prefix
├── customer_cleaned.sql          # Unclear layer
├── customer_final.sql            # Vague name
├── order_data.sql                # Generic name
└── schema.yml                    # Everything in one file
```

**Problems**:
1. **No layer separation** - all models in root `models/` folder
2. **Missing prefixes** - can't tell staging from marts
3. **No source folders** - stripe and salesforce mixed together
4. **Unclear data flow** - which models depend on which?
5. **Single schema file** - hard to maintain

**What evaluator rules fail**:
- [Root Models](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#root-models) - models shouldn't be in the root models folder
- [Missing Primary Key Tests](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#missing-primary-key-tests) - without organization, tests are often forgotten

### ✅ After: Refactored Structure

```
models/
├── staging/
│   ├── stripe/
│   │   ├── _stripe__sources.yml
│   │   ├── _stripe__models.yml
│   │   └── stg_stripe__customers.sql     # Renamed from stripe_customers.sql
│   └── salesforce/
│       ├── _salesforce__sources.yml
│       ├── _salesforce__models.yml
│       └── stg_salesforce__accounts.sql  # Renamed from salesforce_accounts.sql
├── intermediate/
│   └── crm/
│       ├── _int_crm__models.yml
│       └── int_customers_unified.sql     # Renamed from customer_cleaned.sql
└── marts/
    ├── core/
    │   ├── _core__models.yml
    │   └── customers.sql                 # Renamed from customer_final.sql
    └── finance/
        ├── _finance__models.yml
        └── orders.sql                    # Renamed from order_data.sql
```

**Improvements**:
1. **Clear layering** - staging → intermediate → marts flow is obvious
2. **Proper prefixes** - `stg_` and `int_` make model types clear
3. **Source isolation** - stripe and salesforce in separate folders
4. **Data lineage** - you can trace how data flows through layers
5. **Organized documentation** - schema files next to their models

**Refactoring checklist**:
- [ ] Create layer folders: `staging/`, `intermediate/`, `marts/`
- [ ] Create source subfolders within `staging/`
- [ ] Rename models with proper prefixes
- [ ] Move models to appropriate layer folders
- [ ] Split schema.yml into per-folder files
- [ ] Update model references in SQL files
- [ ] Add primary key tests to staging models
- [ ] Document all models with descriptions

---

## How Structure Rules Help You

The dbt-project-evaluator includes specific rules that check your project structure:

### [Root Models](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#root-models)
**What it checks**: Models shouldn't live directly in the `models/` folder  
**Why it matters**: Forces you to organize models into layers  
**How to fix**: Move models into `staging/`, `intermediate/`, or `marts/` subdirectories

### [Model Directories](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#model-directories)
**What it checks**: Models are organized in approved directories  
**Why it matters**: Ensures consistent project structure across your team  
**How to fix**: Follow the canonical structure above

### [Model Naming Conventions](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#model-naming-conventions)
**What it checks**: Models follow naming patterns (stg_, int_, etc.)  
**Why it matters**: Makes model purpose immediately clear  
**How to fix**: Rename models to follow the prefix patterns

### [Models Dependent on Downstream Models](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#models-dependent-on-downstream-models)
**What it checks**: Staging models don't depend on marts, marts don't depend on other marts  
**Why it matters**: Prevents circular dependencies and tangled lineage  
**How to fix**: Refactor dependencies to flow staging → intermediate → marts

### [Undocumented Models](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#undocumented-models)
**What it checks**: Public models have descriptions in schema YML files  
**Why it matters**: Documentation is critical for collaboration  
**How to fix**: Add descriptions to all models in schema YML files

### [Missing Primary Key Tests](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#missing-primary-key-tests)
**What it checks**: Staging models have unique + not_null tests on their primary key  
**Why it matters**: Ensures data quality at the foundation  
**How to fix**: Add tests to your schema YML files

---

## Quick Start Checklist

Setting up a new project? Follow these steps:

1. **Create the folder structure**:
   ```
   mkdir -p models/{staging,intermediate,marts}
   ```

2. **For each data source**:
   - Create `staging/<source_name>/` folder
   - Add `_<source>__sources.yml` file
   - Create `stg_<source>__<table>.sql` models
   - Add `_<source>__models.yml` with tests

3. **For business logic**:
   - Create domain folders in `intermediate/` or `marts/`
   - Name models appropriately for their layer
   - Document in `_<domain>__models.yml` files

4. **Run evaluator checks**:
   ```
   dbt run -s package:dbt_project_evaluator
   ```

5. **Fix any warnings** by referencing the [structure rules](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/#structure)

---

## Additional Resources

- [dbt Best Practices Guide](https://docs.getdbt.com/guides/best-practices)
- [dbt Project Evaluator Rules](https://dbt-labs.github.io/dbt-project-evaluator/latest/rules/)
- [Structure Rules Documentation](https://github.com/rijnhardtkotze/dbt-project-evaluator/blob/main/docs/rules/structure.md)
- [dbt Discourse: Project Structure Discussions](https://discourse.getdbt.com/)

---

## Summary

A well-structured dbt project:
- Uses clear **layer separation**: staging → intermediate → marts
- Follows **naming conventions**: `stg_`, `int_`, and business names for marts
- Groups models by **source system** (staging) or **business domain** (marts/intermediate)
- Keeps **schema YML files** close to the models they document
- Passes **dbt-project-evaluator** structure rules automatically

Remember: Good structure is an investment. It takes a bit more effort upfront, but pays dividends as your project grows and your team expands.

---

## Limitations

### BigQuery and Databricks

BigQuery current support for recursive CTEs is limited and Databricks SQL doesn't support recursive CTEs.

For those Data Warehouses, the model `int_all_dag_relationships` needs to be created by looping CTEs instead. The number of loops is configured with `max_depth_dag` and defaulted to 9. This means that dependencies between models of more than 9 levels of separation won't show in the model `int_all_dag_relationships` but tests on the DAG will still be correct. With a number of loops higher than 9 BigQuery sometimes raises an error saying the query is too complex.
