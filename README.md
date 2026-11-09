# Predicting Felony Sentencing Outcomes in Harris County, Texas

COSC 3337 Data Science I, term project. Olajide, Aidan, Diego.

**Question.** Controlling for the offense charged, does the median family
income of a defendant's ZIP code predict how severe their sentence is?

## Data sources

1. **Harris County District Clerk**, Historical_Criminal_(2020-2024) bulk
   criminal file, from hcdistrictclerk.com > Public Datasets > Criminal.
   520,608 rows, 51 fields, tab-delimited.
2. **ACS 2024 5-Year Estimates**, table B19113 (median family income) by
   ZIP Code Tabulation Area, from data.census.gov. 33,772 ZCTAs.

## Files

| File | Rows | Description |
|---|---|---|
| `harris_felony_model_table.csv` | 61,507 | the modeling dataset |
| `harris_sentencing_cases.csv.gz` | 137,749 | all sentenced cases, incl. misdemeanors |
| `ACSDT5Y2024.B19113_*.zip` | 33,772 | ACS income source |
| `Weekly_Hist Crim Rec Layout-*.xlsx` | | District Clerk field dictionary |

The 211 MB raw file is not committed. Re-download it from the clerk site.

## How the modeling table was built

| Step | Rows remaining |
|---|---|
| Raw rows in the archive | 520,608 |
| Deduplicated to one row per case (`cas`) | 477,822 |
| Has a sentence | 154,540 |
| Has a sentence and an address | 137,749 |
| Felonies only (level starts with F) | 64,814 |
| Valid 5-digit ZIP | 62,656 |
| ZIP matches an ACS record | 61,893 |
| Sentence within statutory maximum | 61,507 |

A case repeats once per charge in the raw file, so deduplication comes
first. 366 rows were dropped for impossible sentence lengths, such as
"98 YEARS HCJ" when county jail caps at one year. Defendant names and
street addresses were excluded.

## Columns in `harris_felony_model_table.csv`

| Column | Description |
|---|---|
| `case_id` | clerk case number |
| `disp_date`, `disp_year` | disposition date |
| `court` | court number |
| `offense_desc` | offense description |
| `level` | F 1, F 2, F 3, F S (state jail), F, F C |
| `bond` | bond amount set |
| `race`, `ethnicity`, `sex` | defendant demographics as coded by the clerk |
| `age_at_offense` | computed from DOB and offense date |
| `counsel_group` | HIRED, APPOINTED, MAC_APPOINTED, PUBLIC_DEFENDER, OTHER |
| `zip` | defendant ZIP, the income join key |
| `zip_median_fam_inc` | ACS 2024 median family income for that ZIP |
| `log_zip_income` | natural log of the above |
| `sentence_raw` | original sentence string |
| `sentence_type` | TDC (prison), HCJ (county jail), STATE_JAIL, PROBATION, FINE, CONFINEMENT |
| `sentence_days` | normalised length in days |
| `log_sentence_days` | natural log, for the regression target |
| `custodial` | 1 if jail or prison, else 0 |
| `severity3` | NONCUSTODIAL, JAIL, PRISON |

Split sentences such as "180 DAYS HCJ, 1 YEARS PROBATION" are resolved
to their most severe component. The raw string is preserved.

**Outcome balance:** PRISON 23,303, NONCUSTODIAL 19,281, JAIL 18,923.

## Loading

```python
import pandas as pd
df = pd.read_csv('harris_felony_model_table.csv', dtype={'case_id': str,
                 'court': str, 'zip': str})
```

Keep `case_id`, `court` and `zip` as strings. They have leading zeros.

## Known limitations

- **ZIP income is a neighbourhood average, not the defendant's income.**
  Findings describe where someone lives, not what they earn.
- **No criminal history.** Prior convictions are the strongest legal
  predictor of sentence and are absent from these records, so some of any
  income effect is likely priors.
- **Counsel type is confounded with offense level.** MAC-appointed
  attorneys handle almost no cases that end in prison, which reflects
  caseload assignment rather than attorney performance.
- **ZIP areas are coarse.** Roughly 130 ZIPs cover Harris County against
  1,115 census tracts. Tract-level income would be finer but requires
  geocoding every address.
