# sfmastech-domain-skills

A repository of domain-specific Snowflake Cortex Code skills spread across **Healthcare**, **Retail**, and **Financial Services & Payments (FSP)**. Each skill is a self-contained, production-ready agent instruction set that leverages Snowflake-native capabilities — including Cortex AI, Dynamic Tables, Cortex Search, and Cortex Analyst — to accelerate data engineering and analytics workflows within a specific industry vertical.

---

## What Are Cortex Code Skills?

Cortex Code skills are structured `SKILL.md` files that guide a Cursor AI agent through multi-step Snowflake data tasks. Each skill encodes domain knowledge, source table schemas, SQL patterns, and execution instructions, enabling the agent to autonomously build data assets, semantic models, and AI-powered services without manual scaffolding.

---

## Domains & Skills

### Healthcare

Skills targeting value-based care, clinical data platforms, and population health management.

| Skill | Description |
|---|---|
| [`patient-360-builder`](healthcare/patient-360-builder/SKILL.md) | Builds a unified Patient 360 view from BRONZE layer tables in `DEMO_DEV.VALUE_BASED_CARE`. Creates a `PATIENT_360` dynamic table (one row per patient), a Cortex Search service for natural language patient lookup, and a semantic view for Cortex Analyst. Covers demographics, clinical summary, risk scores, care gaps, SDOH, medications, prior authorizations, and care management. |

### Retail

Skills targeting competitor intelligence, pricing analytics, and category performance for retail businesses.

| Skill | Description |
|---|---|
| [`retail-competitor-benchmarker`](retail/retail-competitor-benchmarker/SKILL.md) | Builds a retail competitor benchmarking solution from Gold/Silver layer tables in `DEMO_DEV.RETAIL_CATEGORY_ANALYTICS_AGENT`. Creates a `COMPETITOR_BENCHMARK_360` dynamic table, a `BRAND_VS_COMPETITOR_SCORECARD` for head-to-head pricing analysis, a Cortex Search service for natural language competitor queries, and a semantic view for Cortex Analyst. Use for benchmarking retailers, comparing competitors, analyzing competitive positioning, or assessing pricing threats. |


### Financial Services & Payments (FSP)

> Skills coming soon.

---

## Repository Structure

```
sfmastech-domain-skills/
├── healthcare/
│   └── patient-360-builder/
│       └── SKILL.md
├── retail/
│   └── retail-competitor-benchmarker/
│       └── SKILL.md
└── fsp/
```

---

## How to Use a Skill

1. Open the relevant `SKILL.md` file in Cursor.
2. Invoke the Cortex Code agent skill (or reference the skill path in your agent configuration).
3. The agent will read the skill, execute the steps against your Snowflake environment, and report results.

Each skill file contains:
- **Source table schemas** and key relationships
- **Step-by-step SQL** to build dynamic tables, search services, and semantic models
- **Validation queries** to confirm successful execution
- **Execution instructions** including warehouse and schema context setup

---

## Prerequisites

- A Snowflake account with Cortex AI features enabled
- Appropriate warehouse and database/schema permissions as specified in each skill
- Cursor IDE with the [Cortex Code skill](https://github.com/sfmastech) configured
