---
name: patient-360-builder-v2
description: Build a production-grade Patient 360 dynamic table and semantic view from Silver layer tables. Use when asked to create a patient 360, build a unified patient view, consolidate patient data, or generate a patient-centric canonical model. Triggers include patient 360, patient summary, unified patient, member view, patient profile, care intelligence, patient analytics.
---

# Patient 360 Builder v2

You are a Snowflake Cortex CoCo agent responsible for building a reusable data engineering + analytics solution called **patient-360-builder**.

---

## Objective

Create a production-grade, reusable Patient 360 solution with STRICT constraints on what can be created.

---

## STRICT OBJECT CREATION RULES (CRITICAL)

You are ONLY allowed to create:

1. ONE Dynamic Table: `SILVER_PATIENT_360`
2. ONE Semantic View: `Domain_Skill_Generated_Semantic_View`

---

## ABSOLUTE RESTRICTIONS

You MUST NOT create:
- Stages
- File formats
- External or internal stages
- Temporary tables
- Intermediate tables
- Extra views (other than the semantic view)
- Duplicate tables or views
- YAML files
- Any unused or helper objects

All logic must be executed inline within the dynamic table.

---

## MANDATORY FIRST STEP: Schema Analysis

Before generating ANY SQL, perform deep schema analysis:

### Inspect all source tables:
- Identify primary / candidate keys
- Detect join keys (PATIENT_ID, MEMBER_ID, USER_ID, etc.)
- Determine table grain (row-level meaning)
- Identify time-based patterns (latest records)
- Detect duplicates and data quality issues
- Map relationships across datasets

### Output MUST include:
- Table-by-table summary
- Join relationships (logical mapping)
- Entity mapping: PATIENT, MEDICATION, RISK, GAP, SDOH, CARE PLAN
- Recommended join strategy

DO NOT skip this step.

---

## Source Tables (Use ONLY These)

All tables are in `DEMO_DEV.VALUE_BASED_CARE`:

- SILVER_CARE_GAPS
- SILVER_COHORTS
- SILVER_LAB_RESULTS
- SILVER_MEDICATIONS
- SILVER_PATIENTS
- SILVER_PATIENT_COHORTS
- SILVER_PATIENT_EVAL_DATA
- SILVER_PATIENT_SDOH_PROFILE
- SILVER_PATIENT_STATE
- SILVER_RISK_SCORES
- SILVER_USERS

---

## Data Modeling Requirements

Design a patient-centric canonical model:

### Core Grain:
- One row per PATIENT

### Domains:
- Demographics
- Medications (aggregated)
- Risk scores (latest + historical aggregates)
- Care gaps
- SDOH
- Care plans / patient state

---

## BUILD STEP 1: Dynamic Table (SILVER_PATIENT_360)

### Requirements:
- MUST be a Dynamic Table (NOT a view)
- One row per patient
- All joins and logic must be inline (no intermediate objects)
- Use TARGET_LAG = '1 hour' and WAREHOUSE = VBC_INTELLIGENCE_WH

### Techniques:
- QUALIFY ROW_NUMBER() for:
  - Latest risk score
  - Latest patient state
  - Latest care plan (if applicable)

### Handle 1-to-many using:
- LISTAGG / ARRAY_AGG for:
  - Medications
  - Care gaps
- Aggregations for:
  - counts, averages, risk metrics

### Include columns for:
- Identifiers (patient_id, mrn)
- Demographics (name, dob, age, gender, insurance)
- Clinical summary (conditions, medication count, active meds list)
- Risk & quality (risk score, risk level, readmission probability, care gap count, open gaps)
- SDOH (profile data)
- Care management (assigned physician, care manager, current state)

### Ensure:
- No duplicate rows
- Efficient joins
- Clean structure

---

## BUILD STEP 2: Semantic View (Domain_Skill_Generated_Semantic_View)

### Requirements:
- Built directly on SILVER_PATIENT_360
- Define dimensions, measures, and time dimensions
- Add business-friendly descriptions
- Add synonyms for NLQ:
  - "member" → patient
  - "risk" → risk score
  - "gap" → care gap

The semantic view must be created directly using SQL (no YAML files or stages).

---

## Example NL Queries to Support

After building, verify the semantic view supports queries like:
- "Show me everything about member 123"
- "High-risk patients with open care gaps"
- "Average risk score by gender"
- "Patients with more than 5 medications"

---

## Real-World Value

- Eliminates repeated joins across teams
- Reduces data engineering effort significantly
- Enables real-time patient insights via dynamic table auto-refresh
- Enables natural language analytics using Cortex Analyst

---

## Execution Rules

- Produce clean, production-ready SQL
- Do NOT create unnecessary objects
- Do NOT stop at explanation — generate executable SQL
- Ensure everything runs in `DEMO_DEV.VALUE_BASED_CARE` schema context
- Run schema analysis FIRST, then build the dynamic table, then create the semantic view
