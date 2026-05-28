---
name: patient-360-builder
description: Build a unified Patient 360 view from BRONZE layer tables in DEMO_DEV.VALUE_BASED_CARE. Analyzes source schemas, designs a patient-centric data model, creates a PATIENT_360 dynamic table (one row per patient), a Cortex Search service for natural language patient lookup, and a semantic view for Cortex Analyst. Use when asked to build, create, or generate a patient 360 view, unified patient profile, patient summary, or patient-centric data model.
Patient 360 Builder Skill
Overview
This skill builds a production-grade, unified Patient 360 data asset from BRONZE layer source tables in `DEMO_DEV.VALUE_BASED_CARE`. It produces:
A schema analysis report
A canonical patient-centric data model design
SQL for a PATIENT_360 dynamic table
A Cortex Search service definition
A Cortex Analyst semantic view
Example natural language queries
Source Tables
All tables reside in `DEMO_DEV.VALUE_BASED_CARE`:
Table	Grain	Primary Key	Join Key
BRONZE_PATIENTS	One row per patient	ID	ID (= PATIENT_ID elsewhere)
BRONZE_MEMBER_MASTER	One row per member enrollment	ID	PATIENT_ID, MEMBER_ID
BRONZE_ELIGIBILITY_HISTORY	One row per eligibility period	ID	PATIENT_ID, MEMBER_ID
BRONZE_DIAGNOSIS_HISTORY	One row per diagnosis event	ID	PATIENT_ID
BRONZE_MEDICATIONS	One row per medication record	ID	PATIENT_ID
BRONZE_RISK_SCORES	One row per risk assessment date	ID	PATIENT_ID
BRONZE_CARE_GAPS	One row per care gap measure	ID	PATIENT_ID
BRONZE_PATIENT_SDOH_PROFILE	One row per patient SDOH profile	ID	PATIENT_ID
BRONZE_PRIOR_AUTHORIZATIONS	One row per prior auth request	ID	PATIENT_ID, MEMBER_ID
BRONZE_CARE_PLANS	One row per care plan	ID	PATIENT_ID
BRONZE_PROVIDER_MASTER	One row per provider	ID	ID (referenced via ASSIGNED_PHYSICIAN_ID, PRESCRIBING_PHYSICIAN_ID, etc.)
BRONZE_PATIENT_STATE	One row per patient (current state)	PATIENT_ID	PATIENT_ID
Key Relationships
`PATIENT_ID` (NUMBER) is the universal join key across all patient-facing tables
`MEMBER_ID` (VARCHAR) links to payer/eligibility domain
`BRONZE_PATIENT_STATE` contains current risk score, risk level, care gap %, and care team assignments
`BRONZE_PATIENT_SDOH_PROFILE.PROFILE_DATA` is a VARIANT containing nested JSON with sdohRadar scores and social determinant details
Provider assignments flow through `BRONZE_PATIENT_STATE.ASSIGNED_PHYSICIAN_ID` -> `BRONZE_PROVIDER_MASTER.ID`
Schema Details
BRONZE_PATIENTS
ID, MRN, FIRST_NAME, LAST_NAME, DATE_OF_BIRTH, GENDER, AGE, PHONE, EMAIL, ADDRESS, INSURANCE, PRIMARY_CONDITIONS (VARIANT), FRAILTY_SCORE, POLYPHARMACY_RISK_SCORE, LAST_VISIT_DATE, LAST_HOSPITAL_DISCHARGE, IS_DEMO_PATIENT, USER_ID, CREATED_AT
BRONZE_MEMBER_MASTER
ID, MEMBER_ID, PATIENT_ID, PAYER, PLAN_NAME, PLAN_TYPE, GROUP_NUMBER, SUBSCRIBER_ID, RELATIONSHIP_CODE, EFFECTIVE_DATE, TERMINATION_DATE, LOB, MBI, HIC_NUMBER, IS_ACTIVE, CREATED_AT, UPDATED_AT
BRONZE_ELIGIBILITY_HISTORY
ID, MEMBER_ID, PATIENT_ID, PAYER, PLAN_TYPE, ELIGIBILITY_START, ELIGIBILITY_END, ELIGIBILITY_STATUS, TERMINATION_REASON, VERIFIED_DATE, BENEFIT_YEAR, PREMIUM_AMOUNT, SOURCE_SYSTEM, CREATED_AT
BRONZE_DIAGNOSIS_HISTORY
ID, PATIENT_ID, ENCOUNTER_ID, ICD10_CODE, ICD10_DESCRIPTION, DIAGNOSIS_TYPE, DIAGNOSIS_DATE, ENCOUNTER_TYPE, HCC_CODE, HCC_DESCRIPTION, CHRONIC_FLAG, RECORDED_BY_ID, CONFIRMED_FLAG, NOTES, CREATED_AT
BRONZE_MEDICATIONS
ID, PATIENT_ID, NAME, GENERIC_NAME, DRUG_CLASS, DOSE, FREQUENCY, ROUTE, PRESCRIBING_PHYSICIAN_ID, ADHERENCE_PCT, INTERACTION_RISK, STATUS, STARTED_DATE, NOTES, CREATED_AT, UPDATED_AT
BRONZE_RISK_SCORES
ID, PATIENT_ID, SCORE, RISK_LEVEL, READMISSION_PROBABILITY, DATE, RISK_DRIVERS
BRONZE_CARE_GAPS
ID, PATIENT_ID, MEASURE_CODE, MEASURE_NAME, STATUS, DUE_DATE, LAST_COMPLETED_DATE, CLOSED_AT, PROPENSITY, NOTES, CREATED_AT
BRONZE_PATIENT_SDOH_PROFILE
ID, PATIENT_ID, PROFILE_DATA (VARIANT - contains sdohRadar with ADI score, Financial/Social/Housing/Food/Transport metrics, and view with immediateNeeds, profile details), UPDATED_AT
BRONZE_PRIOR_AUTHORIZATIONS
ID, PATIENT_ID, MEMBER_ID, PAYER, AUTH_NUMBER, PROCEDURE_NAME, PROCEDURE_CODE, PROCEDURE_CATEGORY, ICD10_PRIMARY, ICD10_SECONDARY, REQUESTING_PROVIDER_ID, FACILITY_NAME, URGENCY, STATUS, SUBMITTED_DATE, DECISION_DATE, EXPIRATION_DATE, APPROVED_UNITS, DENIAL_REASON, APPEAL_STATUS, NOTES, CREATED_AT, UPDATED_AT
BRONZE_CARE_PLANS
ID, PATIENT_ID, PHYSICIAN_ID, CREATED_BY_ID, TITLE, DESCRIPTION, STATUS, SUBMITTED_AT, APPROVED_AT, ACTIVATED_AT, COMPLETED_AT, CREATED_AT, UPDATED_AT
BRONZE_PROVIDER_MASTER
ID, USER_ID, FULL_NAME, ROLE, SPECIALTY, DEPARTMENT, IS_ACTIVE, CREATED_AT
BRONZE_PATIENT_STATE
ID, PATIENT_ID, CURRENT_STATE, ASSIGNED_PHYSICIAN_ID, ASSIGNED_CARE_MANAGER_ID, ASSIGNED_MTM_ID, ASSIGNED_SOCIAL_WORKER_ID, READMISSION_PROBABILITY, RISK_SCORE, RISK_LEVEL, CARE_GAP_PCT, MISSED_DOSES_30D, REFILL_GAPS_COUNT, UPDATED_AT
---
Step 1: PATIENT_360 Dynamic Table
Create a dynamic table with one row per patient. Use the following design:
```sql
CREATE OR REPLACE DYNAMIC TABLE DEMO_DEV.VALUE_BASED_CARE.PATIENT_360
  TARGET_LAG = '1 hour'
  WAREHOUSE = VBC_INTELLIGENCE_WH
AS
SELECT
    -- IDENTIFIERS
    p.ID AS PATIENT_ID,
    p.MRN,
    mm.MEMBER_ID,

    -- DEMOGRAPHICS
    p.FIRST_NAME,
    p.LAST_NAME,
    p.FIRST_NAME || ' ' || p.LAST_NAME AS PATIENT_NAME,
    p.DATE_OF_BIRTH,
    p.GENDER,
    p.AGE,
    p.PHONE,
    p.EMAIL,
    p.ADDRESS,
    p.INSURANCE,

    -- ENROLLMENT & ELIGIBILITY
    mm.PAYER,
    mm.PLAN_NAME,
    mm.PLAN_TYPE,
    mm.LOB,
    mm.IS_ACTIVE AS MEMBER_IS_ACTIVE,
    elig.ELIGIBILITY_STATUS AS LATEST_ELIGIBILITY_STATUS,
    elig.ELIGIBILITY_START AS LATEST_ELIGIBILITY_START,
    elig.ELIGIBILITY_END AS LATEST_ELIGIBILITY_END,

    -- CLINICAL SUMMARY
    p.PRIMARY_CONDITIONS,
    p.FRAILTY_SCORE,
    p.POLYPHARMACY_RISK_SCORE,
    p.LAST_VISIT_DATE,
    p.LAST_HOSPITAL_DISCHARGE,
    dx_agg.CHRONIC_DIAGNOSIS_COUNT,
    dx_agg.TOTAL_DIAGNOSIS_COUNT,
    dx_agg.CHRONIC_CONDITIONS_LIST,
    dx_agg.HCC_CODES_LIST,
    med_agg.ACTIVE_MEDICATION_COUNT,
    med_agg.ACTIVE_MEDICATIONS_LIST,
    med_agg.AVG_ADHERENCE_PCT,
    med_agg.HIGH_INTERACTION_RISK_COUNT,

    -- RISK & QUALITY
    rs.SCORE AS LATEST_RISK_SCORE,
    rs.RISK_LEVEL AS LATEST_RISK_LEVEL,
    rs.READMISSION_PROBABILITY AS LATEST_READMISSION_PROB,
    rs.RISK_DRIVERS AS LATEST_RISK_DRIVERS,
    rs.DATE AS LATEST_RISK_SCORE_DATE,
    rs_agg.RISK_SCORE_COUNT,
    rs_agg.AVG_RISK_SCORE,
    rs_agg.MIN_RISK_SCORE,
    rs_agg.MAX_RISK_SCORE,
    ps.RISK_SCORE AS STATE_RISK_SCORE,
    ps.RISK_LEVEL AS STATE_RISK_LEVEL,
    ps.READMISSION_PROBABILITY AS STATE_READMISSION_PROB,
    ps.CARE_GAP_PCT,
    ps.MISSED_DOSES_30D,
    ps.REFILL_GAPS_COUNT,
    gaps_agg.TOTAL_CARE_GAPS,
    gaps_agg.OPEN_CARE_GAPS,
    gaps_agg.CLOSED_CARE_GAPS,
    gaps_agg.OPEN_GAP_MEASURES,

    -- SDOH
    sdoh.PROFILE_DATA:sdohRadar:adi::FLOAT AS SDOH_ADI_SCORE,
    sdoh.PROFILE_DATA:sdohRadar:adiLevel::VARCHAR AS SDOH_ADI_LEVEL,
    sdoh.PROFILE_DATA:view:profile:foodSecurity::VARCHAR AS SDOH_FOOD_SECURITY,
    sdoh.PROFILE_DATA:view:profile:housing::VARCHAR AS SDOH_HOUSING,
    sdoh.PROFILE_DATA:view:profile:language::VARCHAR AS SDOH_LANGUAGE,

    -- PRIOR AUTHORIZATIONS
    auth_agg.TOTAL_PRIOR_AUTHS,
    auth_agg.PENDING_PRIOR_AUTHS,
    auth_agg.DENIED_PRIOR_AUTHS,

    -- CARE MANAGEMENT
    cp_agg.ACTIVE_CARE_PLANS,
    cp_agg.LATEST_CARE_PLAN_TITLE,
    ps.CURRENT_STATE,
    ps.ASSIGNED_PHYSICIAN_ID,
    prov.FULL_NAME AS PRIMARY_PROVIDER_NAME,
    prov.SPECIALTY AS PRIMARY_PROVIDER_SPECIALTY,
    ps.ASSIGNED_CARE_MANAGER_ID,
    ps.ASSIGNED_SOCIAL_WORKER_ID

FROM DEMO_DEV.VALUE_BASED_CARE.BRONZE_PATIENTS p

LEFT JOIN DEMO_DEV.VALUE_BASED_CARE.BRONZE_MEMBER_MASTER mm
  ON p.ID = mm.PATIENT_ID AND mm.IS_ACTIVE = TRUE

LEFT JOIN (
    SELECT PATIENT_ID, ELIGIBILITY_STATUS, ELIGIBILITY_START, ELIGIBILITY_END
    FROM DEMO_DEV.VALUE_BASED_CARE.BRONZE_ELIGIBILITY_HISTORY
    QUALIFY ROW_NUMBER() OVER (PARTITION BY PATIENT_ID ORDER BY ELIGIBILITY_START DESC) = 1
) elig ON p.ID = elig.PATIENT_ID

LEFT JOIN (
    SELECT
        PATIENT_ID,
        COUNT_IF(CHRONIC_FLAG = TRUE) AS CHRONIC_DIAGNOSIS_COUNT,
        COUNT(*) AS TOTAL_DIAGNOSIS_COUNT,
        LISTAGG(DISTINCT CASE WHEN CHRONIC_FLAG THEN ICD10_DESCRIPTION END, ', ') AS CHRONIC_CONDITIONS_LIST,
        LISTAGG(DISTINCT HCC_CODE, ', ') WITHIN GROUP (ORDER BY HCC_CODE) AS HCC_CODES_LIST
    FROM DEMO_DEV.VALUE_BASED_CARE.BRONZE_DIAGNOSIS_HISTORY
    GROUP BY PATIENT_ID
) dx_agg ON p.ID = dx_agg.PATIENT_ID

LEFT JOIN (
    SELECT
        PATIENT_ID,
        COUNT_IF(STATUS = 'active') AS ACTIVE_MEDICATION_COUNT,
        LISTAGG(DISTINCT CASE WHEN STATUS = 'active' THEN NAME END, ', ') AS ACTIVE_MEDICATIONS_LIST,
        AVG(CASE WHEN STATUS = 'active' THEN ADHERENCE_PCT END) AS AVG_ADHERENCE_PCT,
        COUNT_IF(STATUS = 'active' AND INTERACTION_RISK IN ('high', 'critical')) AS HIGH_INTERACTION_RISK_COUNT
    FROM DEMO_DEV.VALUE_BASED_CARE.BRONZE_MEDICATIONS
    GROUP BY PATIENT_ID
) med_agg ON p.ID = med_agg.PATIENT_ID

LEFT JOIN DEMO_DEV.VALUE_BASED_CARE.BRONZE_PATIENT_STATE ps
  ON p.ID = ps.PATIENT_ID

LEFT JOIN (
    SELECT PATIENT_ID, SCORE, RISK_LEVEL, READMISSION_PROBABILITY, RISK_DRIVERS, DATE
    FROM DEMO_DEV.VALUE_BASED_CARE.BRONZE_RISK_SCORES
    QUALIFY ROW_NUMBER() OVER (PARTITION BY PATIENT_ID ORDER BY DATE DESC) = 1
) rs ON p.ID = rs.PATIENT_ID

LEFT JOIN (
    SELECT
        PATIENT_ID,
        COUNT(*) AS RISK_SCORE_COUNT,
        AVG(SCORE) AS AVG_RISK_SCORE,
        MIN(SCORE) AS MIN_RISK_SCORE,
        MAX(SCORE) AS MAX_RISK_SCORE
    FROM DEMO_DEV.VALUE_BASED_CARE.BRONZE_RISK_SCORES
    GROUP BY PATIENT_ID
) rs_agg ON p.ID = rs_agg.PATIENT_ID

LEFT JOIN (
        COUNT_IF(STATUS = 'open') AS OPEN_CARE_GAPS,
        COUNT_IF(CLOSED_AT IS NOT NULL) AS CLOSED_CARE_GAPS,
        LISTAGG(DISTINCT CASE WHEN STATUS = 'open' THEN MEASURE_NAME END, ', ') AS OPEN_GAP_MEASURES
    FROM DEMO_DEV.VALUE_BASED_CARE.BRONZE_CARE_GAPS
    GROUP BY PATIENT_ID
) gaps_agg ON p.ID = gaps_agg.PATIENT_ID

LEFT JOIN DEMO_DEV.VALUE_BASED_CARE.BRONZE_PATIENT_SDOH_PROFILE sdoh
  ON p.ID = sdoh.PATIENT_ID

LEFT JOIN (
    SELECT
        PATIENT_ID,
        COUNT(*) AS TOTAL_PRIOR_AUTHS,
        COUNT_IF(STATUS = 'pending') AS PENDING_PRIOR_AUTHS,
        COUNT_IF(STATUS = 'denied') AS DENIED_PRIOR_AUTHS
    FROM DEMO_DEV.VALUE_BASED_CARE.BRONZE_PRIOR_AUTHORIZATIONS
    GROUP BY PATIENT_ID
) auth_agg ON p.ID = auth_agg.PATIENT_ID

LEFT JOIN (
    SELECT
        PATIENT_ID,
        COUNT_IF(STATUS = 'active') AS ACTIVE_CARE_PLANS,
        MAX_BY(TITLE, CREATED_AT) AS LATEST_CARE_PLAN_TITLE
    FROM DEMO_DEV.VALUE_BASED_CARE.BRONZE_CARE_PLANS
    GROUP BY PATIENT_ID
) cp_agg ON p.ID = cp_agg.PATIENT_ID

LEFT JOIN DEMO_DEV.VALUE_BASED_CARE.BRONZE_PROVIDER_MASTER prov
  ON ps.ASSIGNED_PHYSICIAN_ID = prov.ID;
```
---
Step 2: Cortex Search Service
After the PATIENT_360 dynamic table is created, build a Cortex Search service for natural language patient lookup:
```sql
CREATE OR REPLACE CORTEX SEARCH SERVICE DEMO_DEV.VALUE_BASED_CARE.PATIENT_360_SEARCH
  ON SEARCH_TEXT
  ATTRIBUTES PATIENT_NAME, MEMBER_ID, MRN, GENDER, PAYER, PLAN_TYPE, LOB, LATEST_RISK_LEVEL, STATE_RISK_LEVEL, SDOH_ADI_LEVEL, CURRENT_STATE, PRIMARY_PROVIDER_NAME, PRIMARY_PROVIDER_SPECIALTY
  WAREHOUSE = VBC_INTELLIGENCE_WH
  TARGET_LAG = '1 hour'
AS (
    SELECT
        -- Composite search text for full-text matching
        COALESCE(PATIENT_NAME, '') || ' ' ||
        COALESCE(MRN, '') || ' ' ||
        COALESCE(MEMBER_ID, '') || ' ' ||
        COALESCE(GENDER, '') || ' ' ||
        COALESCE(LATEST_RISK_LEVEL, '') || ' risk ' ||
        COALESCE(LATEST_RISK_DRIVERS, '') || ' ' ||
        COALESCE(CHRONIC_CONDITIONS_LIST, '') || ' ' ||
        COALESCE(HCC_CODES_LIST, '') || ' ' ||
        COALESCE(ACTIVE_MEDICATIONS_LIST, '') || ' ' ||
        COALESCE(OPEN_GAP_MEASURES, '') || ' ' ||
        COALESCE(PAYER, '') || ' ' ||
        COALESCE(PLAN_TYPE, '') || ' ' ||
        COALESCE(LOB, '') || ' ' ||
        COALESCE(LATEST_ELIGIBILITY_STATUS, '') || ' ' ||
        COALESCE(SDOH_ADI_LEVEL, '') || ' ADI ' ||
        COALESCE(CAST(SDOH_ADI_SCORE AS VARCHAR), '') || ' ' ||
        COALESCE(SDOH_FOOD_SECURITY, '') || ' ' ||
        COALESCE(SDOH_HOUSING, '') || ' ' ||
        COALESCE(PRIMARY_PROVIDER_NAME, '') || ' ' ||
        COALESCE(PRIMARY_PROVIDER_SPECIALTY, '') || ' ' ||
        COALESCE(CURRENT_STATE, '') || ' ' ||
        COALESCE(LATEST_CARE_PLAN_TITLE, '') || ' ' ||
        COALESCE(ADDRESS, '') || ' ' ||
        COALESCE(CAST(PRIMARY_CONDITIONS AS VARCHAR), '') || ' ' ||
        'diagnoses:' || COALESCE(CAST(TOTAL_DIAGNOSIS_COUNT AS VARCHAR), '0') || ' ' ||
        'meds:' || COALESCE(CAST(ACTIVE_MEDICATION_COUNT AS VARCHAR), '0') || ' ' ||
        'risk_score:' || COALESCE(CAST(LATEST_RISK_SCORE AS VARCHAR), '') || ' ' ||
        'readmission:' || COALESCE(CAST(LATEST_READMISSION_PROB AS VARCHAR), '') || ' ' ||
        'state_risk:' || COALESCE(CAST(STATE_RISK_SCORE AS VARCHAR), '') || ' ' ||
        'state_readmission:' || COALESCE(CAST(STATE_READMISSION_PROB AS VARCHAR), '') || ' ' ||
        'missed_doses:' || COALESCE(CAST(MISSED_DOSES_30D AS VARCHAR), '0') || ' ' ||
        'refill_gaps:' || COALESCE(CAST(REFILL_GAPS_COUNT AS VARCHAR), '0')
        AS SEARCH_TEXT,

        -- All columns returned as attributes/results
        CAST(PATIENT_ID AS VARCHAR) AS PATIENT_ID,
        PATIENT_NAME,
        MRN,
        MEMBER_ID,
        FIRST_NAME,
        LAST_NAME,
        GENDER,
        CAST(AGE AS VARCHAR) AS AGE,
        ADDRESS,
        INSURANCE,
        CAST(PRIMARY_CONDITIONS AS VARCHAR) AS PRIMARY_CONDITIONS,
        PAYER,
        PLAN_NAME,
        PLAN_TYPE,
        LOB,
        CAST(MEMBER_IS_ACTIVE AS VARCHAR) AS MEMBER_IS_ACTIVE,
        LATEST_ELIGIBILITY_STATUS,
        CAST(LATEST_ELIGIBILITY_START AS VARCHAR) AS LATEST_ELIGIBILITY_START,
        CAST(LATEST_ELIGIBILITY_END AS VARCHAR) AS LATEST_ELIGIBILITY_END,
        CHRONIC_CONDITIONS_LIST,
        HCC_CODES_LIST,
        CAST(CHRONIC_DIAGNOSIS_COUNT AS VARCHAR) AS CHRONIC_DIAGNOSIS_COUNT,
        CAST(TOTAL_DIAGNOSIS_COUNT AS VARCHAR) AS TOTAL_DIAGNOSIS_COUNT,
        ACTIVE_MEDICATIONS_LIST,
        CAST(ACTIVE_MEDICATION_COUNT AS VARCHAR) AS ACTIVE_MEDICATION_COUNT,
        CAST(AVG_ADHERENCE_PCT AS VARCHAR) AS AVG_ADHERENCE_PCT,
        CAST(HIGH_INTERACTION_RISK_COUNT AS VARCHAR) AS HIGH_INTERACTION_RISK_COUNT,
        CAST(LATEST_RISK_SCORE AS VARCHAR) AS LATEST_RISK_SCORE,
        LATEST_RISK_LEVEL,
        CAST(LATEST_READMISSION_PROB AS VARCHAR) AS LATEST_READMISSION_PROB,
        LATEST_RISK_DRIVERS,
        CAST(LATEST_RISK_SCORE_DATE AS VARCHAR) AS LATEST_RISK_SCORE_DATE,
        CAST(RISK_SCORE_COUNT AS VARCHAR) AS RISK_SCORE_COUNT,
        CAST(AVG_RISK_SCORE AS VARCHAR) AS AVG_RISK_SCORE,
        CAST(MIN_RISK_SCORE AS VARCHAR) AS MIN_RISK_SCORE,
        CAST(MAX_RISK_SCORE AS VARCHAR) AS MAX_RISK_SCORE,
        CAST(STATE_RISK_SCORE AS VARCHAR) AS STATE_RISK_SCORE,
        STATE_RISK_LEVEL,
        CAST(STATE_READMISSION_PROB AS VARCHAR) AS STATE_READMISSION_PROB,
        CAST(CARE_GAP_PCT AS VARCHAR) AS CARE_GAP_PCT,
        CAST(MISSED_DOSES_30D AS VARCHAR) AS MISSED_DOSES_30D,
        CAST(REFILL_GAPS_COUNT AS VARCHAR) AS REFILL_GAPS_COUNT,
        CAST(TOTAL_CARE_GAPS AS VARCHAR) AS TOTAL_CARE_GAPS,
        CAST(OPEN_CARE_GAPS AS VARCHAR) AS OPEN_CARE_GAPS,
        CAST(CLOSED_CARE_GAPS AS VARCHAR) AS CLOSED_CARE_GAPS,
        OPEN_GAP_MEASURES,
        CAST(SDOH_ADI_SCORE AS VARCHAR) AS SDOH_ADI_SCORE,
        SDOH_ADI_LEVEL,
        SDOH_FOOD_SECURITY,
        SDOH_HOUSING,
        SDOH_LANGUAGE,
        CAST(TOTAL_PRIOR_AUTHS AS VARCHAR) AS TOTAL_PRIOR_AUTHS,
        CAST(PENDING_PRIOR_AUTHS AS VARCHAR) AS PENDING_PRIOR_AUTHS,
        CAST(DENIED_PRIOR_AUTHS AS VARCHAR) AS DENIED_PRIOR_AUTHS,
        CAST(ACTIVE_CARE_PLANS AS VARCHAR) AS ACTIVE_CARE_PLANS,
        LATEST_CARE_PLAN_TITLE,
        CURRENT_STATE,
        CAST(ASSIGNED_PHYSICIAN_ID AS VARCHAR) AS ASSIGNED_PHYSICIAN_ID,
        PRIMARY_PROVIDER_NAME,
        PRIMARY_PROVIDER_SPECIALTY,
        CAST(ASSIGNED_CARE_MANAGER_ID AS VARCHAR) AS ASSIGNED_CARE_MANAGER_ID,
        CAST(ASSIGNED_SOCIAL_WORKER_ID AS VARCHAR) AS ASSIGNED_SOCIAL_WORKER_ID,
        CAST(FRAILTY_SCORE AS VARCHAR) AS FRAILTY_SCORE,
        CAST(POLYPHARMACY_RISK_SCORE AS VARCHAR) AS POLYPHARMACY_RISK_SCORE,
        CAST(LAST_VISIT_DATE AS VARCHAR) AS LAST_VISIT_DATE,
        CAST(LAST_HOSPITAL_DISCHARGE AS VARCHAR) AS LAST_HOSPITAL_DISCHARGE
    FROM DEMO_DEV.VALUE_BASED_CARE.PATIENT_360
);
```
Example Search Queries
```sql
-- Find everything about a specific member
SELECT PARSE_JSON(
  SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
    'DEMO_DEV.VALUE_BASED_CARE.PATIENT_360_SEARCH',
    '{"query": "member MEM12345", "columns": ["PATIENT_NAME", "CURRENT_RISK_LEVEL", "PATIENT_ID"], "limit": 5}'
  )
)['results'];

-- High risk diabetic patients with open gaps
SELECT PARSE_JSON(
  SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
    'DEMO_DEV.VALUE_BASED_CARE.PATIENT_360_SEARCH',
    '{"query": "high risk diabetes open care gaps", "columns": ["PATIENT_NAME", "CURRENT_RISK_LEVEL", "PATIENT_ID"], "filter": {"@eq": {"CURRENT_RISK_LEVEL": "high"}}, "limit": 10}'
  )
)['results'];
```
---
Step 3: Semantic View for Cortex Analyst
Create a semantic view enabling natural language queries against the PATIENT_360 table:
```yaml
# Semantic Model YAML (deploy as stage file or semantic view)
name: patient_360_semantic
tables:
  - name: PATIENT_360
    base_table:
      database: DEMO_DEV
      schema: VALUE_BASED_CARE
      table: PATIENT_360
    dimensions:
      # IDENTIFIERS
      - name: PATIENT_ID
        description: "Unique patient identifier"
        expr: PATIENT_ID
        data_type: NUMBER
      - name: MRN
        synonyms: ["medical record number", "chart number"]
        description: "Medical Record Number"
        expr: MRN
        data_type: VARCHAR
      - name: MEMBER_ID
        synonyms: ["member number", "enrollment id"]
        description: "Payer-assigned member identifier"
        expr: MEMBER_ID
        data_type: VARCHAR

      # DEMOGRAPHICS
      - name: FIRST_NAME
        description: "Patient first name"
        expr: FIRST_NAME
        data_type: VARCHAR
      - name: LAST_NAME
        description: "Patient last name"
        expr: LAST_NAME
        data_type: VARCHAR
      - name: PATIENT_NAME
        synonyms: ["member name", "patient", "member", "full name"]
        description: "Full name of the patient (first + last)"
        expr: PATIENT_NAME
        data_type: VARCHAR
      - name: GENDER
        synonyms: ["sex"]
        description: "Patient gender"
        expr: GENDER
        data_type: VARCHAR
      - name: AGE
        description: "Patient age in years"
        expr: AGE
        data_type: NUMBER
      - name: PHONE
        synonyms: ["phone number", "contact number"]
        description: "Patient phone number"
        expr: PHONE
        data_type: VARCHAR
      - name: EMAIL
        synonyms: ["email address"]
        description: "Patient email address"
        expr: EMAIL
        data_type: VARCHAR
      - name: ADDRESS
        synonyms: ["home address", "location"]
        description: "Patient mailing address"
        expr: ADDRESS
        data_type: VARCHAR
      - name: INSURANCE
        synonyms: ["insurance type", "coverage"]
        description: "Patient insurance type"
        expr: INSURANCE
        data_type: VARCHAR

      # ENROLLMENT & ELIGIBILITY
      - name: PAYER
        synonyms: ["insurance company", "health plan", "carrier"]
        description: "Insurance payer name"
        expr: PAYER
        data_type: VARCHAR
      - name: PLAN_NAME
        synonyms: ["plan", "benefit plan"]
        description: "Name of the insurance plan"
        expr: PLAN_NAME
        data_type: VARCHAR
      - name: PLAN_TYPE
        synonyms: ["plan category"]
        description: "Type of insurance plan (HMO, PPO, etc.)"
        expr: PLAN_TYPE
        data_type: VARCHAR
      - name: LOB
        synonyms: ["line of business", "segment"]
        description: "Line of business (Medicare, Medicaid, Commercial)"
        expr: LOB
        data_type: VARCHAR
      - name: MEMBER_IS_ACTIVE
        synonyms: ["active member", "enrolled"]
        description: "Whether the member enrollment is currently active"
        expr: MEMBER_IS_ACTIVE
        data_type: BOOLEAN
      - name: LATEST_ELIGIBILITY_STATUS
        synonyms: ["enrollment status", "eligibility", "coverage status"]
        description: "Most recent eligibility status"
        expr: LATEST_ELIGIBILITY_STATUS
        data_type: VARCHAR

      # CLINICAL SUMMARY
      - name: PRIMARY_CONDITIONS
        synonyms: ["primary diagnoses", "main conditions", "chief complaints"]
        description: "Primary conditions array from patient record (VARIANT)"
        expr: PRIMARY_CONDITIONS
        data_type: VARIANT
      - name: FRAILTY_SCORE
        synonyms: ["frailty", "frailty index", "functional status"]
        description: "Patient frailty score (higher = more frail)"
        expr: FRAILTY_SCORE
        data_type: FLOAT
      - name: POLYPHARMACY_RISK_SCORE
        synonyms: ["polypharmacy risk", "medication complexity score", "poly score"]
        description: "Polypharmacy risk score (higher = more medication complexity)"
        expr: POLYPHARMACY_RISK_SCORE
        data_type: FLOAT
      - name: CHRONIC_DIAGNOSIS_COUNT
        synonyms: ["number of chronic conditions", "comorbidity count"]
        description: "Number of chronic diagnoses for the patient"
        expr: CHRONIC_DIAGNOSIS_COUNT
        data_type: NUMBER
      - name: TOTAL_DIAGNOSIS_COUNT
        synonyms: ["total diagnoses", "all diagnosis count", "number of diagnoses"]
        description: "Total number of diagnoses (chronic + acute) for the patient"
        expr: TOTAL_DIAGNOSIS_COUNT
        data_type: NUMBER
      - name: CHRONIC_CONDITIONS_LIST
        synonyms: ["diagnoses", "chronic diseases", "conditions", "comorbidities"]
        description: "Comma-separated list of chronic conditions"
        expr: CHRONIC_CONDITIONS_LIST
        data_type: VARCHAR
      - name: HCC_CODES_LIST
        synonyms: ["HCC codes", "hierarchical condition categories", "risk adjustment codes"]
        description: "Comma-separated list of HCC codes"
        expr: HCC_CODES_LIST
        data_type: VARCHAR
      - name: ACTIVE_MEDICATIONS_LIST
        synonyms: ["medications", "drugs", "prescriptions", "meds"]
        description: "Comma-separated list of active medications"
        expr: ACTIVE_MEDICATIONS_LIST
        data_type: VARCHAR
      - name: ACTIVE_MEDICATION_COUNT
        synonyms: ["number of medications", "med count", "active meds"]
        description: "Number of currently active medications"
        expr: ACTIVE_MEDICATION_COUNT
        data_type: NUMBER
      - name: HIGH_INTERACTION_RISK_COUNT
        synonyms: ["drug interactions", "interaction risk count"]
        description: "Number of active medications with high or critical interaction risk"
        expr: HIGH_INTERACTION_RISK_COUNT
        data_type: NUMBER

      # RISK & QUALITY
      - name: LATEST_RISK_SCORE
        synonyms: ["risk score", "current score", "patient risk score"]
        description: "Latest risk score value from BRONZE_RISK_SCORES"
        expr: LATEST_RISK_SCORE
        data_type: FLOAT
      - name: LATEST_RISK_LEVEL
        synonyms: ["risk level", "risk category", "risk tier", "risk stratification"]
        description: "Latest risk level from BRONZE_RISK_SCORES (low, moderate, high, critical)"
        expr: LATEST_RISK_LEVEL
        data_type: VARCHAR
      - name: LATEST_READMISSION_PROB
        synonyms: ["readmission probability", "readmission risk", "30-day readmission"]
        description: "Latest readmission probability (0-1 scale)"
        expr: LATEST_READMISSION_PROB
        data_type: FLOAT
      - name: LATEST_RISK_DRIVERS
        synonyms: ["risk factors", "risk reasons", "why high risk"]
        description: "Factors driving the latest risk score"
        expr: LATEST_RISK_DRIVERS
        data_type: VARCHAR
      - name: RISK_SCORE_COUNT
        synonyms: ["number of risk assessments", "assessment count"]
        description: "Total number of risk score assessments for this patient"
        expr: RISK_SCORE_COUNT
        data_type: NUMBER
      - name: AVG_RISK_SCORE_HISTORICAL
        synonyms: ["historical risk average", "mean risk over time"]
        description: "Patient's average risk score across all assessments"
        expr: AVG_RISK_SCORE
        data_type: FLOAT
      - name: MIN_RISK_SCORE
        synonyms: ["lowest risk score", "best risk score"]
        description: "Patient's lowest (best) historical risk score"
        expr: MIN_RISK_SCORE
        data_type: FLOAT
      - name: MAX_RISK_SCORE
        synonyms: ["highest risk score", "worst risk score", "peak risk"]
        description: "Patient's highest (worst) historical risk score"
        expr: MAX_RISK_SCORE
        data_type: FLOAT
      - name: STATE_RISK_SCORE
        synonyms: ["state risk score", "current state score"]
        description: "Risk score from patient state table (operational current value)"
        expr: STATE_RISK_SCORE
        data_type: FLOAT
      - name: STATE_RISK_LEVEL
        synonyms: ["current state risk level"]
        description: "Risk level from patient state table"
        expr: STATE_RISK_LEVEL
        data_type: VARCHAR
      - name: STATE_READMISSION_PROB
        synonyms: ["state readmission probability"]
        description: "Readmission probability from patient state table"
        expr: STATE_READMISSION_PROB
        data_type: FLOAT
      - name: OPEN_GAP_MEASURES
        synonyms: ["open gaps list", "unclosed measures", "HEDIS gaps"]
        description: "Comma-separated list of open care gap measure names"
        expr: OPEN_GAP_MEASURES
        data_type: VARCHAR
      - name: CARE_GAP_PCT
        synonyms: ["gap percentage", "care gap percent", "gap closure rate"]
        description: "Care gap percentage for the patient"
        expr: CARE_GAP_PCT
        data_type: FLOAT
      - name: OPEN_CARE_GAPS
        synonyms: ["number of open gaps", "outstanding gaps"]
        description: "Number of open care gaps for this patient"
        expr: OPEN_CARE_GAPS
        data_type: NUMBER
      - name: CLOSED_CARE_GAPS
        synonyms: ["number of closed gaps", "completed gaps"]
        description: "Number of closed care gaps for this patient"
        expr: CLOSED_CARE_GAPS
        data_type: NUMBER
      - name: TOTAL_CARE_GAPS
        synonyms: ["all gaps count", "total gap count"]
        description: "Total number of care gaps (open + closed) for this patient"
        expr: TOTAL_CARE_GAPS
        data_type: NUMBER
      - name: MISSED_DOSES_30D
        synonyms: ["missed doses", "medication misses", "doses missed"]
        description: "Number of missed medication doses in last 30 days"
        expr: MISSED_DOSES_30D
        data_type: NUMBER
      - name: REFILL_GAPS_COUNT
        synonyms: ["refill gaps", "prescription gaps", "refill delays"]
        description: "Number of medication refill gaps"
        expr: REFILL_GAPS_COUNT
        data_type: NUMBER
      - name: AVG_ADHERENCE_PCT
        synonyms: ["adherence percentage", "medication adherence", "compliance rate"]
        description: "Average medication adherence percentage for this patient"
        expr: AVG_ADHERENCE_PCT
        data_type: FLOAT

      # SDOH
      - name: SDOH_ADI_SCORE
        synonyms: ["ADI score", "area deprivation index", "deprivation score", "social risk score"]
        description: "Area Deprivation Index numeric score (higher = more deprived)"
        expr: SDOH_ADI_SCORE
        data_type: FLOAT
      - name: SDOH_ADI_LEVEL
        synonyms: ["social determinants level", "ADI level", "deprivation level", "social risk"]
        description: "Area Deprivation Index level from SDOH assessment"
        expr: SDOH_ADI_LEVEL
        data_type: VARCHAR
      - name: SDOH_FOOD_SECURITY
        synonyms: ["food security", "food insecurity", "nutrition risk"]
        description: "Food security status from SDOH profile"
        expr: SDOH_FOOD_SECURITY
        data_type: VARCHAR
      - name: SDOH_HOUSING
        synonyms: ["housing status", "living situation"]
        description: "Housing/living situation from SDOH profile"
        expr: SDOH_HOUSING
        data_type: VARCHAR
      - name: SDOH_LANGUAGE
        synonyms: ["preferred language", "language"]
        description: "Patient preferred language"
        expr: SDOH_LANGUAGE
        data_type: VARCHAR

      # CARE MANAGEMENT
      - name: LATEST_CARE_PLAN_TITLE
        synonyms: ["care plan", "active care plan", "plan title"]
        description: "Title of the most recently created care plan"
        expr: LATEST_CARE_PLAN_TITLE
        data_type: VARCHAR
      - name: ACTIVE_CARE_PLANS
        synonyms: ["number of active plans", "care plan count"]
        description: "Number of currently active care plans"
        expr: ACTIVE_CARE_PLANS
        data_type: NUMBER
      - name: CURRENT_STATE
        synonyms: ["patient state", "care state", "workflow state"]
        description: "Current patient workflow state"
        expr: CURRENT_STATE
        data_type: VARCHAR
      - name: PRIMARY_PROVIDER_NAME
        synonyms: ["physician", "doctor", "PCP", "provider", "assigned doctor"]
        description: "Assigned primary care provider name"
        expr: PRIMARY_PROVIDER_NAME
        data_type: VARCHAR
      - name: PRIMARY_PROVIDER_SPECIALTY
        synonyms: ["specialty", "provider specialty"]
        description: "Specialty of the assigned primary provider"
        expr: PRIMARY_PROVIDER_SPECIALTY
        data_type: VARCHAR
      - name: ASSIGNED_PHYSICIAN_ID
        synonyms: ["physician id", "doctor id", "PCP id"]
        description: "ID of the assigned physician"
        expr: ASSIGNED_PHYSICIAN_ID
        data_type: NUMBER
      - name: ASSIGNED_CARE_MANAGER_ID
        synonyms: ["care manager id", "CM id", "case manager id"]
        description: "ID of the assigned care manager"
        expr: ASSIGNED_CARE_MANAGER_ID
        data_type: NUMBER
      - name: ASSIGNED_SOCIAL_WORKER_ID
        synonyms: ["social worker id", "SW id"]
        description: "ID of the assigned social worker"
        expr: ASSIGNED_SOCIAL_WORKER_ID
        data_type: NUMBER

      # PRIOR AUTHORIZATIONS
      - name: TOTAL_PRIOR_AUTHS
        synonyms: ["prior auth count", "total authorizations"]
        description: "Total number of prior authorization requests for this patient"
        expr: TOTAL_PRIOR_AUTHS
        data_type: NUMBER
      - name: PENDING_PRIOR_AUTHS
        synonyms: ["pending auths", "awaiting decision"]
        description: "Number of pending prior authorizations"
        expr: PENDING_PRIOR_AUTHS
        data_type: NUMBER
      - name: DENIED_PRIOR_AUTHS
        synonyms: ["denied auths", "auth denials"]
        description: "Number of denied prior authorizations"
        expr: DENIED_PRIOR_AUTHS
        data_type: NUMBER

    time_dimensions:
      - name: DATE_OF_BIRTH
        synonyms: ["DOB", "birthday", "birth date"]
        description: "Patient date of birth"
        expr: DATE_OF_BIRTH
        data_type: DATE
      - name: LAST_VISIT_DATE
        synonyms: ["last seen", "most recent visit", "last appointment"]
        description: "Date of most recent patient visit"
        expr: LAST_VISIT_DATE
        data_type: DATE
      - name: LAST_HOSPITAL_DISCHARGE
        synonyms: ["discharge date", "last hospitalization", "last inpatient"]
        description: "Date of most recent hospital discharge"
        expr: LAST_HOSPITAL_DISCHARGE
        data_type: DATE
      - name: LATEST_ELIGIBILITY_START
        synonyms: ["eligibility start", "coverage start"]
        description: "Start date of latest eligibility period"
        expr: LATEST_ELIGIBILITY_START
        data_type: DATE
      - name: LATEST_ELIGIBILITY_END
        synonyms: ["eligibility end", "coverage end"]
        description: "End date of latest eligibility period"
        expr: LATEST_ELIGIBILITY_END
        data_type: DATE
      - name: LATEST_RISK_SCORE_DATE
        synonyms: ["risk score date", "last assessed"]
        description: "Date of the most recent risk score assessment"
        expr: LATEST_RISK_SCORE_DATE
        data_type: TIMESTAMP_NTZ

    measures:
      # COUNTS
      - name: PATIENT_COUNT
        synonyms: ["number of patients", "member count", "population size", "total patients"]
        description: "Total count of patients"
        expr: COUNT(PATIENT_ID)
        data_type: NUMBER
      - name: HIGH_RISK_PATIENT_COUNT
        synonyms: ["high risk count", "at-risk patients", "critical patients"]
        description: "Count of patients with high or critical risk level"
        expr: COUNT_IF(LATEST_RISK_LEVEL IN ('High', 'Critical'))
        data_type: NUMBER

      # CLINICAL
      - name: AVG_CHRONIC_DIAGNOSIS_COUNT
        synonyms: ["average conditions", "avg comorbidities"]
        description: "Average number of chronic diagnoses per patient"
        expr: AVG(CHRONIC_DIAGNOSIS_COUNT)
        data_type: FLOAT
      - name: TOTAL_CHRONIC_DIAGNOSES
        synonyms: ["total chronic conditions"]
        description: "Sum of all chronic diagnoses across patients"
        expr: SUM(CHRONIC_DIAGNOSIS_COUNT)
        data_type: NUMBER
      - name: AVG_ACTIVE_MEDICATION_COUNT
        synonyms: ["average medications", "avg meds per patient"]
        description: "Average number of active medications per patient"
        expr: AVG(ACTIVE_MEDICATION_COUNT)
        data_type: FLOAT
      - name: AVG_ADHERENCE
        synonyms: ["medication adherence", "average adherence", "adherence rate"]
        description: "Average medication adherence percentage across patients"
        expr: AVG(AVG_ADHERENCE_PCT)
        data_type: FLOAT
      - name: TOTAL_HIGH_INTERACTION_RISK
        synonyms: ["drug interaction count", "interaction risks"]
        description: "Total count of high/critical interaction risk medications across patients"
        expr: SUM(HIGH_INTERACTION_RISK_COUNT)
        data_type: NUMBER

      # RISK SCORES
      - name: AVG_RISK_SCORE
        synonyms: ["average risk", "mean risk score", "population risk"]
        description: "Average latest risk score across patients"
        expr: AVG(LATEST_RISK_SCORE)
        data_type: FLOAT
      - name: AVG_READMISSION_PROBABILITY
        synonyms: ["readmission rate", "average readmission risk"]
        description: "Average readmission probability across patients"
        expr: AVG(LATEST_READMISSION_PROB)
        data_type: FLOAT
      - name: MAX_RISK_SCORE_ACROSS_PATIENTS
        synonyms: ["highest risk score", "worst risk"]
        description: "Maximum risk score across all patients"
        expr: MAX(LATEST_RISK_SCORE)
        data_type: FLOAT
      - name: AVG_HISTORICAL_RISK_SCORE
        synonyms: ["historical risk average"]
        description: "Average of patients' historical average risk scores"
        expr: AVG(AVG_RISK_SCORE)
        data_type: FLOAT

      # CARE GAPS & QUALITY
      - name: AVG_CARE_GAP_PCT
        synonyms: ["gap closure rate", "average gap percentage", "quality score"]
        description: "Average care gap percentage across patients"
        expr: AVG(CARE_GAP_PCT)
        data_type: FLOAT
      - name: TOTAL_OPEN_GAPS
        synonyms: ["open gaps", "unclosed gaps", "outstanding gaps"]
        description: "Total number of open care gaps across all patients"
        expr: SUM(OPEN_CARE_GAPS)
        data_type: NUMBER
      - name: TOTAL_CLOSED_GAPS
        synonyms: ["closed gaps", "completed gaps"]
        description: "Total number of closed care gaps across all patients"
        expr: SUM(CLOSED_CARE_GAPS)
        data_type: NUMBER
      - name: TOTAL_CARE_GAPS_ALL
        synonyms: ["all gaps", "total gaps"]
        description: "Total number of care gaps (open + closed) across all patients"
        expr: SUM(TOTAL_CARE_GAPS)
        data_type: NUMBER
      - name: AVG_MISSED_DOSES_30D
        synonyms: ["missed doses", "non-adherence"]
        description: "Average missed medication doses in last 30 days"
        expr: AVG(MISSED_DOSES_30D)
        data_type: FLOAT

      # SDOH
      - name: AVG_SDOH_ADI_SCORE
        synonyms: ["average deprivation", "social risk score"]
        description: "Average Area Deprivation Index score across patients"
        expr: AVG(SDOH_ADI_SCORE)
        data_type: FLOAT

      # PRIOR AUTHORIZATIONS
      - name: TOTAL_PRIOR_AUTHS_ALL
        synonyms: ["total authorizations", "all prior auths"]
        description: "Total prior authorization requests across patients"
        expr: SUM(TOTAL_PRIOR_AUTHS)
        data_type: NUMBER
      - name: TOTAL_PENDING_AUTHS
        synonyms: ["pending authorizations", "awaiting decision"]
        description: "Total pending prior authorizations across patients"
        expr: SUM(PENDING_PRIOR_AUTHS)
        data_type: NUMBER
      - name: TOTAL_DENIED_AUTHS
        synonyms: ["denied authorizations", "auth denials"]
        description: "Total denied prior authorizations across patients"
        expr: SUM(DENIED_PRIOR_AUTHS)
        data_type: NUMBER

      # CARE MANAGEMENT
      - name: TOTAL_ACTIVE_CARE_PLANS
        synonyms: ["active plans", "care plans in progress"]
        description: "Total active care plans across patients"
        expr: SUM(ACTIVE_CARE_PLANS)
        data_type: NUMBER
      - name: AVG_FRAILTY_SCORE
        synonyms: ["average frailty", "frailty index"]
        description: "Average frailty score across patients"
        expr: AVG(FRAILTY_SCORE)
        data_type: FLOAT
      - name: AVG_POLYPHARMACY_RISK
        synonyms: ["polypharmacy risk", "medication complexity"]
        description: "Average polypharmacy risk score across patients"
        expr: AVG(POLYPHARMACY_RISK_SCORE)
        data_type: FLOAT

    filters:
      - name: HIGH_RISK_ONLY
        synonyms: ["high risk filter", "at-risk only", "critical only"]
        description: "Filter to high and critical risk patients only"
        expr: LATEST_RISK_LEVEL IN ('High', 'Critical')
      - name: ACTIVE_MEMBERS_ONLY
        synonyms: ["enrolled only", "active enrollment", "current members"]
        description: "Filter to currently active/enrolled members"
        expr: MEMBER_IS_ACTIVE = TRUE
      - name: HAS_OPEN_GAPS
        synonyms: ["patients with gaps", "open gap filter"]
        description: "Filter to patients with at least one open care gap"
        expr: OPEN_CARE_GAPS > 0
      - name: RECENTLY_DISCHARGED
        synonyms: ["recent discharge", "post-acute"]
        description: "Filter to patients discharged in last 30 days"
        expr: LAST_HOSPITAL_DISCHARGE >= DATEADD('day', -30, CURRENT_DATE())
      - name: HIGH_READMISSION_RISK
        synonyms: ["readmission risk", "likely to be readmitted"]
        description: "Filter to patients with readmission probability > 0.5"
        expr: LATEST_READMISSION_PROB > 0.5
      - name: SDOH_AT_RISK
        synonyms: ["social risk", "social determinants at risk"]
        description: "Filter to patients with High or Moderate ADI level"
        expr: SDOH_ADI_LEVEL IN ('High', 'Moderate')
      - name: HIGH_ADI_SCORE
        synonyms: ["high deprivation score", "ADI above 80", "severe social risk"]
        description: "Filter to patients with ADI score above 80 (high deprivation)"
        expr: SDOH_ADI_SCORE > 80
      - name: LOW_ADHERENCE
        synonyms: ["non-adherent", "poor adherence", "medication non-compliance"]
        description: "Filter to patients with medication adherence below 70%"
        expr: AVG_ADHERENCE_PCT < 70
```
---
Step 4: Example Queries for Validation
After building the assets, test with these queries:
```sql
-- Basic patient lookup
SELECT PATIENT_NAME, CURRENT_RISK_LEVEL, OPEN_CARE_GAPS, ACTIVE_MEDICATION_COUNT
FROM DEMO_DEV.VALUE_BASED_CARE.PATIENT_360
WHERE PATIENT_ID = 1;

-- High-risk diabetics with open care gaps
SELECT PATIENT_NAME, CHRONIC_CONDITIONS_LIST, OPEN_CARE_GAPS, CURRENT_RISK_LEVEL
FROM DEMO_DEV.VALUE_BASED_CARE.PATIENT_360
WHERE CURRENT_RISK_LEVEL IN ('high', 'critical')
  AND CHRONIC_CONDITIONS_LIST ILIKE '%diabet%'
  AND OPEN_CARE_GAPS > 0
ORDER BY OPEN_CARE_GAPS DESC;

-- Population risk distribution
SELECT CURRENT_RISK_LEVEL, COUNT(*) AS PATIENT_COUNT, AVG(CARE_GAP_PCT) AS AVG_GAP_PCT
FROM DEMO_DEV.VALUE_BASED_CARE.PATIENT_360
GROUP BY CURRENT_RISK_LEVEL
ORDER BY PATIENT_COUNT DESC;

-- SDOH-impacted patients
SELECT PATIENT_NAME, SDOH_ADI_LEVEL, SDOH_FOOD_SECURITY, CURRENT_RISK_LEVEL
FROM DEMO_DEV.VALUE_BASED_CARE.PATIENT_360
WHERE SDOH_ADI_LEVEL IN ('High', 'Moderate')
ORDER BY SDOH_ADI_SCORE DESC;
```
---
Execution Instructions
When this skill is invoked:
Set context: Run `USE DATABASE DEMO_DEV; USE SCHEMA VALUE_BASED_CARE; USE WAREHOUSE VBC_INTELLIGENCE_WH;` to ensure all objects are created in `DEMO_DEV.VALUE_BASED_CARE`.
Create the PATIENT_360 dynamic table using the SQL in Step 1. Execute it and verify row counts with `SELECT COUNT(*) FROM DEMO_DEV.VALUE_BASED_CARE.PATIENT_360;`.
Create the Cortex Search service using the SQL in Step 2. All objects are created in `DEMO_DEV.VALUE_BASED_CARE`. Wait for initialization and test with a sample query.
Create the semantic model: Upload the YAML in Step 3 to stage `@DEMO_DEV.VALUE_BASED_CARE.PATIENT_360_STAGE/patient_360_semantic.yaml` or create a semantic view in `DEMO_DEV.VALUE_BASED_CARE`.
Validate by running the example queries in Step 4.
Report results including row count, sample data, and confirmation that all objects were created successfully in `DEMO_DEV.VALUE_BASED_CARE`.
Target Schema: All objects (PATIENT_360, PATIENT_360_SEARCH, semantic model/view) MUST be created in `DEMO_DEV.VALUE_BASED_CARE`.
Real-World Value
This solution:
Eliminates 6-12 months of BI effort to integrate claims, EHR, pharmacy, and quality data
Removes need for repeated JOIN logic across teams
Reduces cohort build time from days to minutes
Enables care managers to view unified patient data in one place
Provides natural language access via Cortex Search and Cortex Analyst
