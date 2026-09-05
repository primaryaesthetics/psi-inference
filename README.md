# psi-inference

[![PyPI](https://img.shields.io/pypi/v/psi-inference.svg)](https://pypi.org/project/psi-inference/)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22342343.svg)](https://doi.org/10.5281/zenodo.22342343)

Critical values and p-values for population stability testing in credit risk
monitoring: pure numpy and scipy, no pandas dependency, Python 3.11+.

A model built on one population gets used on another one later, and the
question a monitoring report has to answer is whether the two still look
alike. The population stability index (PSI) is the standard tool for this,
and the standard reading of it is a pair of constants: below 0.10 is fine,
above 0.25 needs a rebuild. Those constants do not know the sample size or
the number of bins the PSI was computed on, so the same reading means
different things on a 500-row book and a 500,000-row one. This package
implements the statistics that do know it.

## Example

```python
import numpy as np
from psi_inference import psi_test

# Seven credit grades: the base sample and a later monitoring sample.
base = np.array([253, 302, 204, 134, 72, 26, 8])      # 999 accounts
current = np.array([177, 262, 285, 158, 88, 25, 6])    # 1001 accounts

result = psi_test(base, current, alpha=0.05)
print(result.statistic)        # 0.0677
print(result.critical_value)   # 0.0252
print(result.p_value)          # 7.2e-06
print(result.reject)           # True
```

Read against the 0.10 rule of thumb, 0.068 says nothing has changed. This
sample has seven categories and about a thousand rows on each side. At 5%
significance the critical value here is 0.025. The observed PSI is more than
two and a half times that. The p-value says the same thing in one number: a
shift this large would happen by chance in roughly seven per hundred
thousand comparisons if the population had not moved at all. The rule of
thumb and the correctly sized test disagree here. On this sample size, the
test is the one with a stated error rate.

## Critical values

At alpha = 0.05, from `psi_critical_value(n, m, bins, alpha=0.05)`:

| n = m     | B = 10  | B = 20  |
| --------- | ------- | ------- |
| 1,000     | 0.0338  | 0.0603  |
| 10,000    | 0.0034  | 0.0060  |
| 100,000   | 0.0003  | 0.0006  |

The critical value falls by roughly an order of magnitude for each order of
magnitude of sample size, and rises with the number of bins. A PSI of 0.05
is a strong signal at 100,000 rows per side and unremarkable at 1,000. Fixed
thresholds cannot see either fact.

## Why 0.10 and 0.25 are not thresholds

The 0.10 / 0.25 pair comes from Lewis (1994) as a rule of thumb, with no
stated error rate and no dependence on how much data the PSI was computed on.
Yurdakul and Naranjo (2020) show that under the hypothesis of no population
shift, PSI is approximately distributed as (1/n + 1/m) times a chi-square
random variable with B − 1 degrees of freedom. Here n and m are the two
sample sizes and B is the number of bins. That scale factor is exactly what
the fixed thresholds throw away. The practical consequence is two-sided: a
small monitoring sample crosses 0.10 routinely on pure sampling noise, and a
large one can move by an amount that would fail any honest test while never
approaching 0.25. Reporting a PSI against 0.10 or 0.25 is reporting a number
against constants that were never fit to this sample size, this bin count,
or any stated tolerance for being wrong.

## What this package implements

- **PSI** (`psi_statistic`, `psi_critical_value`, `psi_pvalue`, `psi_test`):
  Yurdakul, B. and Naranjo, J. (2020), "Statistical properties of the
  population stability index", *Journal of Risk Model Validation* 14(4),
  89-100, section 3 (theorems 3.1-3.3) and section 4 (equation 4.1). Two
  other Python packages compute the same asymptotic critical value, `iyipada`
  and `feature-engine`'s `DropHighPSIFeatures(threshold="auto")`; this module
  is checked against both in `tests/test_cross_check.py` and is included here
  for completeness, not as a new result.
- **PRS** (`prs_statistic`, `prs_delta`, `prs_thresholds`, `prs_classify`):
  the Population Resemblance Statistic of Potgieter, C.J., Van Zyl, C.,
  Schutte, W.D. and Lombard, F., "The Population Resemblance Statistic: A
  Chi-Square Measure of Fit for Banking" (arXiv:2307.11878). Compares one
  sample against a fixed reference population rather than two samples
  against each other, and gives sample-size-calibrated critical values from
  the non-central chi-square distribution, plus a three-tier decision
  (acceptable / partial / full discrepancy) instead of a single cutoff.
- **Effect-size test** (`effect_size_statistic`, `effect_size_critical_value`,
  `effect_size_pvalue`) and **overlapping test** (`overlapping_statistic`,
  `overlapping_critical_value`, `overlapping_pvalue`): Du Pisanie, J.,
  Allison, J.S., Budde, C.J. and Visagie, I.J.H., "A critical review of
  existing and new population stability testing procedures in credit risk
  scoring" (arXiv:2303.01227), sections 3.6 and 3.7. The effect-size
  statistic does not grow with sample size, which is the failure mode of the
  classical goodness-of-fit tests on a bank-sized book. The overlapping
  statistic reports the share of probability mass two distributions hold in
  common, which needs no explanation of a p-value to a non-technical
  audience.

None of these three had a Python implementation before this package.
`rpsi` on CRAN implements the Yurdakul-Naranjo critical value in R and
nothing beyond it.

## Installing and running the tests

```
pip install psi-inference
```

To run the tests from a checkout:

```
uv venv --python 3.12
uv pip install -e ".[dev]"
uv run pytest
```

The test suite includes a Monte Carlo gate. It simulates the null directly
for several sample sizes and bin counts, including a small sample (500 rows)
and unequal sample sizes (50,000 against 2,000). It checks that the
empirical 95th percentile of the simulated statistic matches the analytic
critical value within simulation error for the PSI and for the PRS. A
formula recalled wrong from the paper produces a number, not an error. This
is the check that catches it.

## Notes on the sources

Every formula in this package is pinned by a test to a worked example
printed in its source paper: the PSI to Yurdakul and Naranjo's seven-grade
example and to their critical-value tables for B = 10 and B = 20; the PRS to
the four (sample size, bin count) cases of the source paper's critical-value
table; the effect-size and overlapping statistics to the paper's example of
a two-category shift from 50/50 to 50.5/49.5. Each matches to the digits the
source prints, and the test names say which table.

The PSI null agrees everywhere it was checked: Yurdakul and Naranjo's own
equation, the `rpsi` R source, `iyipada` and `feature-engine` all reduce to
the same line, `chi2.ppf(1 - alpha, bins - 1) * (1/n + 1/m)`. `rpsi` also
offers a mode that treats the base distribution as a fixed population rather
than a random sample, which drops the `1/n` term. That mode is not
implemented here. The PRS covers the fixed-reference case with a null that
is built for it.

## Citing

The package is archived on Zenodo. The concept DOI 10.5281/zenodo.22342343
resolves to the latest version; the DOI of version 0.1.0 is
10.5281/zenodo.22342344. `CITATION.cff` in the repository carries the full
record.

## Licence

MIT.
