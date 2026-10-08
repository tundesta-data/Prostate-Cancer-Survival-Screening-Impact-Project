# Prostate Cancer Survival & Treatment Analytics

**Power BI | DAX | Healthcare Analytics**

A two-page Power BI dashboard analysing prostate cancer survival, screening behaviour, tumour grading and treatment outcomes across 75,000 patient records.

> **Note on the data:** this project uses a **synthetic dataset created for educational and portfolio purposes**. The figures illustrate analytical patterns and are not real patient outcomes. Nothing here should be read as clinical guidance.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Tools & Technologies](#tools--technologies)
- [Dataset Overview](#dataset-overview)
- [Data Model](#data-model)
- [Data Preparation](#data-preparation)
- [Key Measures](#key-measures)
- [Dashboard](#dashboard)
- [Key Metrics](#key-metrics)
- [Key Insights](#key-insights)
- [Data Quality Findings](#data-quality-findings)
- [Recommendations](#recommendations)
- [Clinical Notes](#clinical-notes)
- [Dataset Source](#dataset-source)

---

## Project Overview

Prostate cancer is highly survivable when caught early and considerably less so when it isn't. This project quantifies that gap and examines what drives it.

The analysis sets out to:

- Measure how survival changes across cancer stage, tumour grade and clinical T stage
- Quantify the relationship between screening history and early detection
- Compare recurrence rates across seven treatment pathways
- Examine whether diagnostic markers (PSA, PI-RADS, Gleason) align with disease severity
- Present the findings in a way that communicates the screening message at first view

---

## Tools & Technologies

| Tool | Use |
|---|---|
| **Power BI Desktop** | Data model, DAX measures, dashboard design |
| **DAX** | Calculated columns, survival and recurrence measures, dynamic subtitle cards |
| **Power Query** | Data typing, cleaning and load |
| **Custom theme (JSON)** | Consistent colour semantics across both pages |

---

## Dataset Overview

**75,000 patient records**, 31 fields, covering demographics, diagnostics, staging, histology, treatment and outcomes.

| Category | Fields |
|---|---|
| **Demographics** | `Patient_ID`, `Age`, `Gender` |
| **Diagnostics** | `PSA_Level_ng_mL`, `MRI_PI_RADS`, `DRE_Result`, `Gleason_Score`, `Grade_Group` |
| **Biopsy** | `Biopsy_Cores_Sampled`, `Biopsy_Cores_Positive`, `Histology_Confirmation`, `Histology_Subtype` |
| **Staging** | `Cancer_Stage`, `Clinical_T_Stage`, `Clinical_N_Stage`, `Clinical_M_Stage`, `Metastatic_Site`, `Tumor_Size_cm`, `Lesion_Distribution` |
| **Screening** | `Screening_History`, `Early_Detection`, `Symptoms_at_Diagnosis` |
| **Outcomes** | `Treatment_Type`, `Survival_Status`, `Survival_Months`, `Recurrence_Status`, `Recurrence_Months` |

### Sample preview

| Patient_ID | Age | Cancer_Stage | PSA_Level_ng_mL | Gleason_Score | Grade_Group | Clinical_T_Stage | Treatment_Type | Survival_Status | Survival_Months |
|---|---|---|---|---|---|---|---|---|---|
| PC000001 | 68 | Stage II | 9.4 | 7 | 2 | T2 | Radical Prostatectomy | Alive | 74 |
| PC000002 | 74 | Stage IV | 42.1 | 9 | 5 | T4 | Hormone Therapy | Deceased | 18 |

---

## Data Model

A **star schema** with one fact table and one image dimension:

- **`prostate`** — the fact table, one row per patient
- **`prostate_cancer_image_url`** — a 13-row dimension holding a pathology image and stage icon for each `Cancer_Stage` × `Lesion_Distribution` combination, plus a healthy-prostate reference row
- **`Months`** — a generated series used to plot the survival curve

The two tables join on a composite key, since Power BI relationships accept only a single column:

```dax
Stage Lesion Key =
prostate[Cancer_Stage] & " | " & prostate[Lesion_Distribution]
```

The dimension drives the anatomical image that updates as the user filters by stage, with the healthy reference image isolated using `ALL()` so it stays fixed for comparison.

---

## Data Preparation

- Verified `Gender` is 100% male and excluded it as a non-discriminating field
- Grouped `Age` into clinical bands (Under 50, 50–59, 60–69, 70–79, 80+) with an explicit sort column
- Mapped `Grade_Group` to standard ISUP labels (GG1 Low through GG5 Very High)
- Identified blank-string values in `Metastatic_Site` and excluded them from metastatic measures using `KEEPFILTERS`
- Built a `Months` series table to support a step-down survival curve
- Standardised numeric typing on PSA, tumour size and biopsy core counts

---

## Key Measures

```dax
Survival Rate % =
DIVIDE(
    CALCULATE(COUNTROWS(prostate), prostate[Survival_Status] = "Alive"),
    COUNTROWS(prostate)
)
```

```dax
Biopsy Positivity % =
DIVIDE(SUM(prostate[Biopsy_Cores_Positive]), SUM(prostate[Biopsy_Cores_Sampled]))
```

```dax
Metastatic Patients =
CALCULATE(
    COUNTROWS(prostate),
    KEEPFILTERS(prostate[Metastatic_Site] <> "" && prostate[Metastatic_Site] <> "None")
)
```

```dax
Survival Curve % =
VAR m = SELECTEDVALUE(Months[Value])
VAR reached = CALCULATE([Total Patients], prostate[Survival_Months] >= m)
VAR diedBefore = CALCULATE([Total Patients],
        prostate[Survival_Months] < m,
        prostate[Survival_Status] = "Deceased")
RETURN DIVIDE(reached, reached + diedBefore)
```

Dynamic subtitle cards use `TOPN` over a virtual table to surface the best or worst performer in context:

```dax
Best Surviving Stage =
VAR t = ADDCOLUMNS(VALUES(prostate[Cancer_Stage]), "@s", [Survival Rate %])
VAR best = TOPN(1, t, [@s], DESC)
RETURN MAXX(best, prostate[Cancer_Stage] & " (" & FORMAT([@s], "0.0%") & ")")
```

---

## Dashboard

### Page 1 — Survival Overview
*Does early detection change outcomes?*

- Six KPI cards, each with a dynamic context subtitle
- Survival vs mortality by cancer stage
- Survival and mortality by tumour grade group
- Early detection by screening history (100% stacked)
- Patients by age group
- Survival curve over months since diagnosis
- Healthy-reference and stage-specific anatomical imagery

  <img width="740" height="400" alt="Screenshot 2026-10-07 205003" src="https://github.com/user-attachments/assets/95a10b15-da68-4727-a23f-212fb81f9988" />


### Page 2 — Clinical & Treatment Insights
*Grade and spread drive outcomes more than treatment choice.*

- Five diagnostic KPI cards
- PSA level vs tumour size, coloured by grade group
- Biopsy positivity by MRI PI-RADS score
- Survival rate by clinical T stage
- Recurrence rate by treatment type
- Patients by metastatic site
- Survival by grade group and cancer stage

<img width="735" height="400" alt="Screenshot 2026-10-07 205043" src="https://github.com/user-attachments/assets/51a06946-ed33-47bd-b7da-2acabfebd947" />


Both pages share synced slicers for cancer stage and treatment type.

---

## Key Metrics

| Metric | Value |
|---|---|
| Total Patients | 75,000 |
| Survival Rate | 85% |
| Mortality Rate | 15% |
| Recurrence Rate | 19% |
| Average Survival | 70 months |
| Early Detection Rate | 64% |
| Average PSA | 15.56 ng/mL |
| Average Gleason Score | 7.24 |
| Biopsy Positivity | 34% |
| Metastatic Rate | 18% |

---

## Key Insights

### 1. Stage at diagnosis is the single strongest predictor of survival

Survival falls from **98% at Stage I to 52% at Stage IV** — a near-halving driven entirely by how late the disease is caught. The same pattern holds on clinical T stage: 98% at T1, 52% at T4, with the sharpest drop occurring once the tumour breaks through the prostate capsule.

### 2. Screening behaviour maps directly onto early detection

| Screening history | Detected early |
|---|---|
| Regular check-up | 72% |
| Occasional check-up | 61% |
| No check-up | 51% |

Patients detected early survive at **95.9%**, against 85% overall.

### 3. Tumour grade predicts outcome independently of stage

Survival declines across grade groups — **97% at GG1, 92% at GG2, 75% at GG4, 58% at GG5** — confirming that how aggressive the cancer looks under the microscope matters alongside how far it has spread.

### 4. Imaging predicts biopsy yield

Biopsy positivity rises consistently with MRI PI-RADS score: **16% at PI-RADS 2, 26% at 3, 35% at 4, and 56% at PI-RADS 5**. Imaging is doing real diagnostic work rather than simply confirming what is already known.

### 5. Bone dominates metastatic spread

Of metastatic patients, **bone accounts for 8.3K cases** — roughly three times distant lymph nodes (2.8K) and more than lung (1.4K) and liver (1.1K) combined. This matches the known biology of prostate cancer.

### 6. Recurrence varies more by treatment than survival does

Recurrence rates span **11% (Active Surveillance) to 30% (Chemotherapy)**. Interpreting this requires care: treatment assignment correlates with disease severity, so the gap reflects case mix as much as treatment efficacy.

---

## Data Quality Findings

Documenting what the data does *not* support is part of the analysis:

- **Grade Group 3 (Gleason 4+3) is absent** from the dataset entirely. Charts on grade group therefore show a gap between GG2 and GG4, which is a property of the source data rather than a reporting error.
- **`Metastatic_Site` uses empty strings** rather than nulls or a "None" label for non-metastatic patients. Measures that filtered only on "None" initially returned a 100% metastatic rate; this was corrected using `KEEPFILTERS`.
- **Time to recurrence shows no variation** across treatment types (52–54 months throughout), suggesting the field was generated independently of treatment. It was excluded from the final dashboard for that reason.
- **Early detection reaches 51% even among patients with no screening history**, which is high for an unscreened group and is likely an artefact of synthetic generation. It slightly compresses the screening contrast.
- **`Gender` is 100% male**, as expected, and was dropped from the analysis.

---

## Recommendations

1. **Prioritise screening uptake over treatment optimisation.** The survival gap between Stage I and Stage IV (46 percentage points) dwarfs any difference observed between treatment pathways.
2. **Target men aged 50+.** The 60–69 and 70–79 bands account for 50,000 of 75,000 patients.
3. **Use MRI to triage biopsy.** The PI-RADS to positivity gradient supports imaging-led biopsy decisions over systematic sampling.
4. **Monitor grade group alongside stage.** GG5 patients survive at 58% regardless of how early the stage appears.
5. **Track recurrence by treatment with case-mix adjustment.** Raw recurrence comparisons are confounded by disease severity at assignment.

---

## Clinical Notes

For readers unfamiliar with prostate cancer terminology:

- **Grade Group** (ISUP) translates the Gleason score into a 1–5 scale. GG1 (Gleason 6) is the least aggressive form and is often managed by active surveillance rather than immediate treatment — but it is still cancer, not a healthy result.
- **A normal DRE does not rule out cancer.** Many prostate cancers are detected by PSA testing and MRI despite a normal physical examination.
- **Early-stage prostate cancer is frequently asymptomatic.** Risk rises after age 50, and earlier for Black men and those with a family history.

General information: [NHS — Prostate cancer](https://www.nhs.uk/conditions/prostate-cancer/)

---

## Dataset Source
<img width="933" height="457" alt="Screenshot 2026-10-08 101815" src="https://github.com/user-attachments/assets/3d069cff-168e-4673-9d60-3b08a492499d" />

Synthetic dataset generated for educational and portfolio use.

[Download Here](https://docs.google.com/spreadsheets/d/1CXsmw7o6rdjMadvYXIZDlmgmoom1hw0DswyQiGw-2h4/edit?usp=sharing)

---

**Built by Tunde Adebayo** — Data Analyst, London
[LinkedIn](#) · [Portfolio](#)
