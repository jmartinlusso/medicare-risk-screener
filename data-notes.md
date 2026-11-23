# Data notes: CMS Part D 2019 + OIG LEIE

First look at the two raw sources for the Medicare Provider Risk Screener (Day 4).
Exploration: `notebooks/01_explore.ipynb`.

## Sources

| Source | File | Rows | Downloaded |
|---|---|---|---|
| CMS Medicare Part D Prescribers by Provider, data year 2019 | `data/raw/partd_2019.csv` (641 MB) | 1,240,595 | 2026-11-21 |
| OIG List of Excluded Individuals/Entities (LEIE), full file | `data/raw/leie.csv` (11 MB) | 84,224 | 2026-11-21 |

- Part D: one row per prescriber (1,240,595 distinct NPIs), 84 columns. CMS has published data years 2013–2024.
- LEIE: one row per exclusion, 18 columns. Record layout: http://oig.hhs.gov/exclusions/files/leie_record_layout.pdf
- Join key: `Prscrbr_NPI` (Part D) = `NPI` (LEIE). Both load as BIGINT.

## Key counts

| Step | Six positive codes | With 1128b4 |
|---|---|---|
| LEIE rows | 84,224 | 84,224 |
| …with a real NPI (not 0) | 9,023 (10.7%) | 9,023 |
| …with a positive exclusion code | 5,392 | 8,083 |
| …matching a 2019 Part D prescriber | 916 rows / 915 NPIs | 1,404 rows / 1,391 NPIs |
| **…excluded in the label window (2020–2021)** | **151** | **193** |

Matched prescribers by exclusion date (six codes): before 2019: 2 · during 2019: 39 · 2020–2021: 151 · after 2021: 724.

**Class imbalance:** 151 positives out of 1,240,595 prescribers = **0.012%, about 1 in 8,200**.
A model that always predicts "negative" is 99.99% accurate, so accuracy is useless here; the project uses precision@500.

## Draft label definition (finalize on Day 6)

- **Feature year t = 2019. Label = excluded within the next 2 years (2020–2021).**
- **Positive exclusion codes:** fraud- and drug-related:
  `1128a1` (program-related conviction), `1128a3` (felony health care fraud), `1128a4` (felony controlled substances),
  `1128b1` (misdemeanor health care fraud), `1128b3` (misdemeanor controlled substances), `1128b7` (fraud, kickbacks).
- **Open question:** `1128b4` (license revoked/suspended). It adds 42 positives (+28%), and a license is sometimes lost for
  overprescribing, but often for reasons unrelated to fraud.
- **Drop** prescribers already excluded by the end of the feature year (41 here): they're neither positive nor negative.

## Quirks

1. **Most LEIE rows have no NPI.** 75,201 of 84,224 rows have `NPI = 0`. They're mostly old exclusions (median Oct 2008 vs
   Aug 2019 for real NPIs; NPIs were issued from 2005 and required from 2007), but no-NPI exclusions are still added today,
   likely businesses and workers who never bill as prescribers. None of them can be matched to Part D.
2. **Reinstated providers are missing.** `REINDATE` is 0 in every row: the full file only lists current exclusions, and
   reinstated providers are removed. Labels therefore miss everyone who was excluded and later reinstated (label bias).
3. **Exclusion is not fraud.** 22 exclusion types; the most common, `1128b4` (license revocation, 33,463 rows), often has
   nothing to do with fraud. `1128b14` (defaulted student loan) and `1128a2` (patient abuse) are also not fraud.
4. **Duplicate NPIs in the LEIE.** 1,475 matched rows but 1,453 distinct NPIs: one prescriber can have several exclusion
   rows. Build labels from distinct NPIs, or the join duplicates prescribers.
5. **Excluded but still prescribing.** 2 prescribers were excluded before 2019 yet have 2019 Part D claims.
   My explanation: TODO
6. **Dates and codes stored as numbers.** `EXCLDATE`, `REINDATE`, `WAIVERDATE` and `DOB` are YYYYMMDD integers: they sort
   like dates, but need converting for date arithmetic. `ZIP` loses leading zeros (02139 → 2139).
7. **Later exclusions count as negatives.** 724 matched prescribers were excluded after 2021, outside the 2-year window, so
   they are negatives for 2019 features. A longer horizon would turn some into positives: revisit H on Day 9.
8. **Specialties vary widely.** Nurse practitioners (187,939) and dentists (131,165) are the largest groups, and residents
   have their own category (58,640). Features must compare prescribers within their specialty.
9. **Privacy suppression in Part D.** Counts between 1 and 10 are left empty, so NULL in a count column usually means
   "small", not "zero".

## Next

- Day 6: finalize the positive codes and the review capacity in `PROBLEM.md`.
- Day 9: download 4–5 more years, build labels with real dates, drop already-excluded prescribers, split by year.
