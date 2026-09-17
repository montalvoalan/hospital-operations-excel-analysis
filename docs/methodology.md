# Methodology

This document explains how I built and checked the hospital operations workbook. The goal was to keep the analysis refreshable, preserve the meaning of each source table, and avoid calculations that would inflate the results.

## Source Data

The project uses five CSV files from the Maven Analytics Hospital Patient Records dataset:

| Table | Rows | What one row represents |
|---|---:|---|
| Patients | 974 | One patient |
| Encounters | 27,891 | One patient encounter |
| Procedures | 47,701 | One recorded procedure |
| Payers | 10 | One insurance payer |
| Organizations | 1 | One healthcare organization |

I kept the raw files unchanged in `data/raw/`. Before building calculations, I reviewed the fields, checked the intended keys, and defined the grain of each table. This was important because encounter costs and procedure costs are stored at different levels.

## Power Query Preparation

I imported the source files with Power Query instead of editing the CSV files directly. In Power Query, I:

- assigned suitable data types to identifiers, dates, text, and currency fields;
- kept meaningful blanks, such as encounters without a reason code;
- added date fields used for monthly and yearly analysis;
- calculated encounter duration and patient responsibility fields;
- created a lookup query for the formula-driven Encounter Explorer; and
- kept the main data tables separate instead of merging procedures into encounters.

The separate-table approach prevents an encounter with several procedures from repeating its total claim cost several times.

## Data-Quality Checks

I created QA queries to check the relationships and record structure. These checks covered:

- orphan patient, payer, and organization keys in the encounters table;
- orphan encounter and patient keys in the procedures table;
- procedure records whose patient did not match the patient on the related encounter;
- duplicate procedure records; and
- overall row-count and relationship summaries.

The QA queries were used as checks rather than as sources for the dashboard calculations.

## Data Model

I used the Excel Data Model rather than creating one flattened table. The model contains these one-to-many relationships:

```text
Patients[Id]       1 ---- * Encounters[PATIENT]
Organizations[Id]  1 ---- * Encounters[ORGANIZATION]
Payers[Id]         1 ---- * Encounters[PAYER]
Encounters[Id]     1 ---- * Procedures[ENCOUNTER]
```

`Encounters` is the main fact table for utilization, claim cost, and coverage. `Procedures` is a separate fact table for procedure frequency and base cost. I did not add encounter claim cost and procedure base cost together because the source definitions do not establish that they are separate components of one total.

## Core Calculations

The main calculations used throughout the workbook were:

- **Encounter count:** distinct encounter IDs.
- **Unique patients:** distinct patients with an encounter.
- **Encounter duration:** encounter stop time minus start time, reported in hours.
- **Total claim cost:** sum of encounter-level total claim cost.
- **Patient responsibility:** total claim cost minus payer coverage.
- **Weighted coverage rate:** total payer coverage divided by total claim cost.
- **Repeat inpatient patient:** a patient with more than one inpatient encounter during the full analysis period.
- **Procedure count:** number of procedure records.
- **Average procedure cost:** procedure base cost divided by procedure count.

I used Power Pivot measures for model-based summaries and standard Excel formulas for the Encounter Explorer and supporting checks. Formula work included `XLOOKUP`, `COUNTIFS`, `FILTER`, `UNIQUE`, `SORT`, `LET`, and `IFERROR`.

## Analysis Process

The analysis was completed in stages rather than starting with the dashboard:

1. I checked encounter volume over time and by encounter class.
2. I reviewed inpatient duration using both the distribution and long-stay counts.
3. I counted patients with multiple inpatient encounters. I described this as repeat utilization, not a formal readmission rate.
4. I compared total and average claim cost across encounter classes.
5. I measured payer coverage using a weighted rate and investigated zero-coverage encounters.
6. I ranked procedures separately by frequency, total base cost, and average base cost.
7. I recorded candidate findings, evidence, interpretations, and follow-up questions in the Insight Log.

This sequence made it easier to check each result before deciding what belonged on the dashboard.

## Dashboard and Encounter Explorer

The dashboard brings together five headline metrics, five charts, three summary findings, and slicers for year, encounter class, and payer. I tested the slicers individually and in combination, then cleared them before saving the final workbook.

The Encounter Explorer uses a validated encounter-ID selection to display operational and financial details for one synthetic encounter. It also includes a changeable high-cost threshold and a formula check confirming that the filtered row count matches the qualifying encounter count. Patient names and street addresses are not displayed.

## Reconciliation and Final Testing

I reconciled the main totals across Power Query, formulas, PivotTables, and the Data Model. The final controls were:

| Control | Final value |
|---|---:|
| Encounter records | 27,891 |
| Unique patients | 974 |
| Procedure records | 47,701 |
| Total claim cost | $101,514,375.52 |
| Recorded payer coverage | $31,097,506.99 |
| Weighted coverage rate | 30.6% |

Before packaging the project, I ran **Refresh All**, checked the formula validation, tested the dashboard slicers, and confirmed that the workbook returned to the full-population totals after the filters were cleared.

## Interpretation Limits

- The dataset is synthetic and does not represent the performance of an actual hospital.
- The 2022 data ends on February 5, so it is not a complete year.
- Multiple inpatient encounters are not automatically clinical readmissions.
- Payer coverage is a recorded field, not proof that a payment was adjudicated or received.
- A small number of very long stays can raise the average inpatient duration.
- Average procedure costs based on only a few records should be interpreted cautiously.
- Encounter claim cost and procedure base cost use different grains and should not be combined.

## Refreshing the Workbook

1. Open `workbook/hospital_operations_analysis.xlsx` in desktop Excel.
2. Update the Power Query source paths if the repository was downloaded to a different location.
3. Select **Data > Refresh All**.
4. Confirm that the QA checks and headline totals still reconcile.
5. Clear the dashboard slicers before reviewing the complete results.
