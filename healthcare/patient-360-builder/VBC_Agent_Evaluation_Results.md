# VBC Patient Agent - Evaluation Results

**Agent:** `DEMO_DEV.VALUE_BASED_CARE.VBC_PATIENT_AGENT`
**Export Date:** 2026-05-28

---

## Summary

| Run Name | Total Records | Avg Answer Correctness | Avg Logical Consistency |
|---|---|---|---|
| Eval_CoCo_Generated_Run_1 | 25 questions | 0.65 | 0.95 |
| Eval_Domain_Skill_Generated_Run_1 | 25 questions | 0.75 | 0.97 |

---

## Eval_CoCo_Generated_Run_1

### Scores by Question

| # | Input | Answer Correctness | Logical Consistency | Duration (ms) | LLM Calls |
|---|---|---|---|---|---|
| 1 | How many active medications does Martha Jones have? | 0.33 | 1.00 | 38,264 | 5 |
| 2 | Write me a Python script to scrape Amazon prices | 1.00 | 1.00 | 5,019 | 1 |
| 3 | What is the gap closure rate by risk level? | 1.00 | 1.00 | 43,047 | 5 |
| 4 | Which patients have food insecurity issues? | 0.00 | 1.00 | 58,625 | 7 |
| 5 | What are the risk drivers for patient Eleanor Rigby? | 1.00 | 1.00 | 25,411 | 3 |
| 6 | What percentage of patients are high risk? | 1.00 | 1.00 | 18,178 | 3 |
| 7 | Which patients were recently discharged from the hospital? | 0.67 | 0.67 | 28,153 | 4 |
| 8 | Give me a population health summary | 0.67 | 0.67 | 48,108 | 5 |
| 9 | What are the top care management recommendations by severity? | 0.67 | 1.00 | 57,346 | 5 |
| 10 | Show me patients over age 75 | 0.67 | 1.00 | 19,604 | 3 |
| 11 | How do I change the oil in a 2022 Honda Civic? | 1.00 | 1.00 | 5,761 | 1 |
| 12 | Which patients have medication adherence below 60%? | 0.00 | 1.00 | 58,630 | 7 |
| 13 | Show me patients with high social determinants of health burden | 0.00 | 1.00 | 94,740 | 7 |
| 14 | What is the best laptop for gaming under $1500? | 1.00 | 1.00 | 3,700 | 1 |
| 15 | List patients with high drug interaction risk | 0.33 | 1.00 | 33,915 | 3 |
| 16 | What are the trending fashion styles for summer 2025? | 1.00 | 1.00 | 7,235 | 1 |
| 17 | Show me patients whose risk score has increased over time | 0.00 | 1.00 | 60,851 | 5 |
| 18 | What is the average medication adherence percentage across all patients? | 0.00 | 1.00 | 42,017 | 5 |
| 19 | Which provider has the most high-risk patients assigned? | 1.00 | 1.00 | 16,226 | 3 |
| 20 | Which patients have abnormal lab results in the last 30 days? | 0.00 | 1.00 | 35,846 | 4 |
| 21 | Which patients have the most open care gaps? | 1.00 | 1.00 | 44,974 | 5 |
| 22 | Show me the care team for patient ID 1 | 0.67 | 1.00 | 14,899 | 3 |
| 23 | Who won the NBA finals last year? | 1.00 | 1.00 | 4,080 | 1 |
| 24 | Which patients have a readmission probability above 0.7? | 0.67 | 1.00 | 18,269 | 3 |
| 25 | How many patients are in each risk level category? | 1.00 | 1.00 | 17,536 | 3 |
| 26 | Which patients have the highest risk scores? | 1.00 | 1.00 | 21,533 | 3 |
| 27 | How many patients are enrolled in Medicare vs Medicaid? | 0.67 | 1.00 | 19,697 | 3 |
| 28 | Show me patients in the identified state who need outreach | 0.67 | 1.00 | 20,670 | 3 |
| 29 | Which patients have active care plans? | 0.67 | 1.00 | 25,435 | 4 |
| 30 | Show me overdue care gaps across the population | 0.33 | 1.00 | 74,478 | 6 |

### Detailed Results

#### Q1: How many active medications does Martha Jones have?

**Ground Truth:** Returns the count of active medications and the medication list for patient Martha Jones.

**Agent Output:**
> The semantic model does not include a dedicated active medications count field for patients. However, here is the available medication-related data for Martha Jones (Patient ID: 3, MRN: MRN-1120):
> - Polypharmacy Risk Score: 9.1
> - Missed Doses (Last 30 Days): 8
> - Medication Refill Gaps: 1
>
> An exact count of active medications is not available in the current data model.

**Scores:** Correctness: 0.33 | Consistency: 1.00

---

#### Q4: Which patients have food insecurity issues?

**Ground Truth:** Returns patients whose SDOH food security status indicates risk, including patient name, food security status, ADI level, and risk level.

**Agent Output:**
> The current data model (COCO_GENERATED_SEMANTIC_VIEW) does not include a food insecurity field. The available patient data covers demographics, risk scores, medication adherence, care gaps, care team, and visit history. Food insecurity is a Social Determinants of Health (SDOH) metric and is not captured in the current patient summary view.

**Scores:** Correctness: 0.00 | Consistency: 1.00

---

#### Q12: Which patients have medication adherence below 60%?

**Ground Truth:** Returns patients with average medication adherence percentage below 60%, including patient name, adherence percentage, and active medication count.

**Agent Output:**
> The dataset does not contain a direct medication adherence % column. The available medication-related fields are refill_gaps_count and missed_doses_30d. Neither provides a percentage-based adherence score.

**Scores:** Correctness: 0.00 | Consistency: 1.00

---

#### Q13: Show me patients with high social determinants of health burden

**Ground Truth:** Returns patients with high or moderate ADI level, including patient name, ADI score, food security status, housing status, and language.

**Agent Output:**
> The data model does not include a dedicated SDOH score or flag. The closest available proxy indicators are high risk score, frailty score, care gap %, readmission probability, and polypharmacy risk. [Provided proxy results using high-risk patients]

**Scores:** Correctness: 0.00 | Consistency: 1.00

---

#### Q17: Show me patients whose risk score has increased over time

**Ground Truth:** Returns patients where the latest risk score is higher than their historical average risk score, indicating a worsening trend.

**Agent Output:**
> The current data model provides a point-in-time snapshot per patient rather than historical time-series risk scores. However, surfaced the 8 High-risk patients as the population most likely to have experienced risk escalation.

**Scores:** Correctness: 0.00 | Consistency: 1.00

---

#### Q18: What is the average medication adherence percentage across all patients?

**Ground Truth:** Returns the average medication adherence percentage across all patients.

**Agent Output:**
> The semantic model does not include a dedicated medication adherence percentage column. The available medication-related metrics are missed_doses_30d and refill_gaps_count. There is no medication_adherence_pct or equivalent field.

**Scores:** Correctness: 0.00 | Consistency: 1.00

---

#### Q20: Which patients have abnormal lab results in the last 30 days?

**Ground Truth:** Returns patients with abnormal lab results within the last 30 days, including patient ID, test name, result value, and abnormal flag.

**Agent Output:**
> The current data model (COCO_GENERATED_SEMANTIC_VIEW) does not contain lab result records. The available patient summary data includes demographics, risk scores, care gaps, medication adherence, frailty scores, and readmission probability but no discrete lab result entries.

**Scores:** Correctness: 0.00 | Consistency: 1.00

---

## Eval_Domain_Skill_Generated_Run_1

### Scores by Question

| # | Input | Answer Correctness | Logical Consistency | Duration (ms) | LLM Calls |
|---|---|---|---|---|---|
| 1 | How many patients are in each risk level category? | 1.00 | 1.00 | 23,947 | 3 |
| 2 | Show me patients with high social determinants of health burden | 0.33 | 1.00 | 32,627 | 4 |
| 3 | Which patients have the most open care gaps? | 1.00 | 1.00 | 20,696 | 3 |
| 4 | How many patients are enrolled in Medicare vs Medicaid? | 0.67 | 1.00 | 17,910 | 3 |
| 5 | Who won the NBA finals last year? | 1.00 | 1.00 | 6,666 | 1 |
| 6 | What are the trending fashion styles for summer 2025? | 1.00 | 1.00 | 6,527 | 1 |
| 7 | Show me overdue care gaps across the population | 0.33 | 1.00 | 48,999 | 5 |
| 8 | What are the risk drivers for patient Eleanor Rigby? | 0.67 | 1.00 | 23,255 | 3 |
| 9 | Which patients have active care plans? | 0.67 | 1.00 | 32,245 | 4 |
| 10 | Show me patients in the identified state who need outreach | 1.00 | 1.00 | 30,133 | 3 |
| 11 | What percentage of patients are high risk? | 1.00 | 1.00 | 16,628 | 3 |
| 12 | Which patients have food insecurity issues? | 0.00 | 1.00 | 35,132 | 5 |
| 13 | Which patients have a readmission probability above 0.7? | 0.67 | 1.00 | 25,495 | 3 |
| 14 | Which patients were recently discharged from the hospital? | 0.67 | 1.00 | 31,460 | 4 |
| 15 | Which patients have abnormal lab results in the last 30 days? | 0.33 | 1.00 | 23,193 | 3 |
| 16 | List patients with high drug interaction risk | 0.33 | 1.00 | 32,726 | 3 |
| 17 | Write me a Python script to scrape Amazon prices | 1.00 | 1.00 | 5,832 | 1 |
| 18 | What is the best laptop for gaming under $1500? | 1.00 | 1.00 | 4,559 | 1 |
| 19 | Show me the care team for patient ID 1 | 0.67 | 1.00 | 18,158 | 3 |
| 20 | What is the average medication adherence percentage across all patients? | 1.00 | 1.00 | 14,869 | 3 |
| 21 | Give me a population health summary | 1.00 | 1.00 | 64,727 | 6 |
| 22 | How many active medications does Martha Jones have? | 1.00 | 1.00 | 17,643 | 3 |
| 23 | Show me patients whose risk score has increased over time | 0.67 | 1.00 | 74,635 | 5 |
| 24 | Show me patients over age 75 | 0.67 | 1.00 | 20,269 | 3 |
| 25 | Which provider has the most high-risk patients assigned? | 1.00 | 1.00 | 17,719 | 3 |
| 26 | What are the top care management recommendations by severity? | 0.67 | 1.00 | 49,025 | 5 |
| 27 | What is the gap closure rate by risk level? | 1.00 | 1.00 | 59,758 | 7 |
| 28 | How do I change the oil in a 2022 Honda Civic? | 1.00 | 1.00 | 5,675 | 1 |
| 29 | Which patients have medication adherence below 60%? | 0.67 | 1.00 | 19,352 | 3 |
| 30 | Which patients have the highest risk scores? | 1.00 | 1.00 | 26,293 | 3 |

### Detailed Results

#### Q2: Show me patients with high social determinants of health burden

**Ground Truth:** Returns patients with high or moderate ADI level, including patient name, ADI score, food security status, housing status, and language.

**Agent Output:**
> Only 1 patient in the dataset is classified with a High SDOH burden (Area Deprivation Index level = High):
> - Patient ID: 8, David Nguyen, Age 70 (Male), Risk Level: Low, ADI Score: 85, ADI Level: High, Open Care Gaps: 2 (Annual Wellness Visit, HbA1c Test), Medication Adherence: 76.5%, Readmission Probability: 19%

**Scores:** Correctness: 0.33 | Consistency: 1.00

---

#### Q12: Which patients have food insecurity issues?

**Ground Truth:** Returns patients whose SDOH food security status indicates risk, including patient name, food security status, ADI level, and risk level.

**Agent Output:**
> The current patient data model does not contain a dedicated field for food insecurity. After searching across SDOH indicators, risk drivers, and open care gaps, no patients have food insecurity explicitly documented. The semantic view captures SDOH through the Area Deprivation Index (ADI).

**Scores:** Correctness: 0.00 | Consistency: 1.00

---

#### Q15: Which patients have abnormal lab results in the last 30 days?

**Ground Truth:** Returns patients with abnormal lab results within the last 30 days, including patient ID, test name, result value, and abnormal flag.

**Agent Output:**
> 4 patients have abnormal lab results in the last 30 days (all High risk):
> - Eleanor Rigby: 5 abnormal labs (May 1, 2026)
> - Martha Jones: 3 abnormal labs (May 3, 2026)
> - Irene Kelly: 2 abnormal labs (May 2, 2026)
> - Robert Okafor: 2 abnormal labs (Apr 29, 2026)

**Scores:** Correctness: 0.33 | Consistency: 1.00

---

#### Q20: What is the average medication adherence percentage across all patients?

**Ground Truth:** Returns the average medication adherence percentage across all patients.

**Agent Output:**
> The average medication adherence percentage across all patients is 72.14%.

**Scores:** Correctness: 1.00 | Consistency: 1.00

---

#### Q22: How many active medications does Martha Jones have?

**Ground Truth:** Returns the count of active medications and the medication list for patient Martha Jones.

**Agent Output:**
> Martha Jones currently has 9 active medications: Tiotropium, Azithromycin, Furosemide, Lisinopril, Calcium + Vitamin D, Amlodipine, Fluticasone/Salmeterol, Omeprazole, Prednisone

**Scores:** Correctness: 1.00 | Consistency: 1.00

---

## Comparative Analysis

### Key Differences

| Question | CoCo Score | Domain Skill Score | Winner |
|---|---|---|---|
| How many active medications does Martha Jones have? | 0.33 | 1.00 | Domain Skill |
| What is the average medication adherence percentage? | 0.00 | 1.00 | Domain Skill |
| Which patients have abnormal lab results in the last 30 days? | 0.00 | 0.33 | Domain Skill |
| Show me patients with high SDOH burden | 0.00 | 0.33 | Domain Skill |
| Give me a population health summary | 0.67 | 1.00 | Domain Skill |
| What are the risk drivers for patient Eleanor Rigby? | 1.00 | 0.67 | CoCo |
| What is the gap closure rate by risk level? | 1.00 | 1.00 | Tie |
| Which patients have food insecurity issues? | 0.00 | 0.00 | Tie |
| Show me patients whose risk score has increased over time | 0.00 | 0.67 | Domain Skill |

### Token Usage

| Run | Total Input Tokens | Total Output Tokens | Total LLM Calls |
|---|---|---|---|
| Eval_CoCo_Generated_Run_1 | 1,003,295 | 33,103 | 108 |
| Eval_Domain_Skill_Generated_Run_1 | 926,750 | 33,898 | 98 |
