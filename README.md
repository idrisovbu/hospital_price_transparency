# Reconciling hospital and payer price disclosures

This project combines two public price disclosures for the same three hospitals
and three billing codes into a single table, and reports where the hospital's
price and the insurer's price agree.

* One file is published by insurers and lists negotiated payment rates.
* One file is published by hospitals and lists the rates they accept.

The two use different schemas and report prices at different levels of detail, so
the work is to reshape both onto a shared level of observation, match them, and
state how confident each match is.

## The full write-up

The complete analysis, with every transformation and the reasoning behind it, is
in `price_reconciliation.Rmd` which produced the knitted `price_reconciliation.html`. 
You can just open this link https://htmlpreview.github.io/?https://github.com/idrisovbu/hospital_price_transparency/blob/main/price_reconciliation.html

for the narrative version, or open the `.Rmd` and knit yourself. 

## Files

* `price_reconciliation.Rmd` : the pipeline and the write-up.
* `price_reconciliation.html` : the knitted report (open this to read).
* `unified_output.csv` : the combined table, written by the pipeline.
* `hpt_extract_20250213.csv` : hospital price disclosure (input).
* `tic_extract_20250213.csv` : insurer price disclosure (input).

## How to run

From the project directory, with R installed:

```r
install.packages(c("tidyverse", "MASS", "rmarkdown"))
rmarkdown::render("price_reconciliation.Rmd")
```

This regenerates `price_reconciliation.html` and writes `unified_output.csv`.

## Method in brief

1. Normalize the join keys: tax id to nine digits, billing code, insurer name.
2. Join on tax id, code, and insurer, keeping commercial plans.
3. Put both files on a common full price basis. A service has a facility part and
   a professional part. The insurer file reports both, so we learn the ratio of
   each part to the full price from it, then use those ratios to scale the
   hospital's single reported number up to a full price estimate (facility-to-total
   approach; Dieleman et al. 2025).
4. Benchmark every price against Medicare (using the baseline the payer file
   provides), so different services are comparable and can be checked against
   published national figures.
5. Assess agreement between the two sources with a Bland-Altman method comparison,
   and express match confidence as a price consistency measure in a record
   linkage framing.
6. Compare the two full prices and report the price ratio, the price consistency,
   and a decision.

## Notes

The analysis draws on published work: the RAND Hospital Price Transparency Study
for the Medicare benchmark, Fellegi and Sunter (1969) for the linkage score, and
Bland and Altman (1986) for the agreement analysis, and the facility-to-total
method co-authored by Bulat Idrisov and published in JAMA (Dieleman et al. 2025).
Full citations are in the document.

## Codebook for `unified_output.csv`

| Column | Meaning |
| --- | --- |
| `hospital` | Hospital name, recovered from the tax id (the payer file carries no name). |
| `payer` | Insurer, normalized to a single label (AETNA, CIGNA, UHC). |
| `family` | Code type: CPT (a single service), DRG (a bundled inpatient stay), LOCAL (a hospital's own internal code). |
| `code` | The billing code (43239 endoscopy with biopsy, 99283 emergency visit, 872 sepsis inpatient stay). |
| `status` | Which file the raw rate came from: `both`, `hospital_only`, or `payer_only`. |
| `hospital_rate` | The hospital file's raw negotiated rate (median across its commercial plans), before any adjustment. |
| `reported_part` | Which component the hospital rate is inferred to be: `facility` or `professional`. `NA` when not determinable. |
| `hospital_full` | The hospital rate scaled to a full-encounter estimate (facility plus professional). `NA` when no scaling ratio exists. |
| `payer_full` | The insurer's full-encounter price, its facility and professional parts summed. |
| `payer_x_medicare` | The insurer's price as a multiple of Medicare (2.02 means 202 percent of Medicare). |
| `price_ratio` | `hospital_full` divided by `payer_full` (1.46 means the hospital side is 1.46 times the payer side). `NA` unless matched on both sides. |
| `price_consistency` | How typical the price gap is for a true match, from 0 to 1. Near 1 is central; below about 0.05 is outside the limits of agreement. |
| `decision` | `match`, `non-match`, `review (no professional part to model)`, or `unmatched`. |
