# Where does a tech paycheck go the furthest?

Project 02 proposal: Logan (working solo)

## Problem statement

- **Who has the problem:** graduating CS students (me included, since I finish in December 2026) and the campus career centers that advise them. New grads have to decide which cities to apply to and which offers to take.
- **The decision:** big tech hubs like San Jose, San Francisco, and Seattle post the highest salaries, but they also cost the most to live in. A $150k offer in San Jose and a $115k offer in Raleigh aren't really comparable until you adjust for prices.
- **Why it matters:** new grads usually compare offers by the headline number. If high-cost hubs don't pay enough extra to cover their prices, students could be turning down offers that would leave them with more money in practice, or skipping whole regions they never looked at.
- **What I expect to find:** nominal pay will be highest in the coastal hubs, but once wages are adjusted for cost of living I expect several mid-cost metros (Texas, the Southeast, the Mountain West) to match or beat them. I also expect metros with more tech jobs per capita to still pay more in real terms, but by less than the headline numbers suggest.

## Research question

> Is a metro area's concentration of software developer jobs (jobs per 1,000) associated with its **cost-of-living-adjusted** median annual wage for software developers, after accounting for census region and metro size?

- **Response:** real (cost-adjusted) median annual wage for software developers
- **Main predictor:** software developer jobs per 1,000 total jobs in the metro
- **Other predictors:** census region (categorical) and metro size class (categorical)

## Data acquisition

I'll download two public government datasets and join them by metro area code (CBSA code): the BLS **Occupational Employment and Wage Statistics (OEWS), May 2025** metro-area file for wages and employment, and the BEA **Regional Price Parities (RPPs) by metro area, 2024** for cost of living.

## Confirm viability

Both are free downloads with no login, and both are U.S. federal government works in the public domain.

**1. OEWS, May 2025 (wages and employment)**

- Citation: U.S. Bureau of Labor Statistics. (2026). *Occupational Employment and Wage Statistics, May 2025: Metropolitan and nonmetropolitan area estimates* [Data set]. <https://www.bls.gov/oes/tables.htm>
- Direct file: [oesm25ma.zip](https://www.bls.gov/oes/special-requests/oesm25ma.zip) (contains `MSA_M2025_dl.xlsx`)
- Documentation: [OEWS technical notes, May 2025](https://www.bls.gov/oes/2025/may/oes_tec.htm)
- How it was collected: OEWS is an employer survey run by BLS together with state workforce agencies. About 1.1 million nonfarm establishments are sampled over a three-year cycle in six semiannual panels (Nov 2022 to May 2025 for this release). Employers report employment and wages by occupation by web, mail, email, phone, or in person. Metro areas follow the 2023 OMB (Bulletin 23-01) definitions.

**2. BEA Regional Price Parities, 2024 (cost of living)**

- Citation: U.S. Bureau of Economic Analysis. (2026). *Regional price parities by state and metro area, 2024* [Data set]. <https://www.bea.gov/data/prices-inflation/regional-price-parities-state-and-metro-area>
- Direct file: the `MARPP` zip from [BEA regional downloads](https://apps.bea.gov/regional/downloadzip.htm)
- Documentation: [RPP news release and methodology](https://www.bea.gov/newsreleases/regional/rpp/rpp_newsrelease.htm)
- How it was collected: BEA builds the RPPs from existing data, mainly BLS Consumer Price Index price quotes plus Census Bureau American Community Survey housing rents. An RPP of 100 is the national average, so 120 means prices are 20% above average.

**Expected size:** OEWS covers about 390 metro areas. After dropping metros where software developer data are suppressed and metros that don't match BEA, I expect roughly 300 to 380 rows, well above the 200 minimum. If the count comes in lower, my backup is to widen the unit to metro × tech occupation (software developers, web developers, QA, data scientists, information security analysts, and so on), which would give 1,000+ rows and add occupation as another categorical variable.

## Variables of interest

One row = one metro area (software developers, SOC 15-1252).

| Variable | Source | Type | Description |
|---|---|---|---|
| `area` | OEWS | ID (not analyzed) | CBSA metro code, used as the join key |
| `area_title` | OEWS | ID (not analyzed) | Metro name, e.g. "San Jose-Sunnyvale-Santa Clara, CA" |
| `prim_state` | OEWS | Categorical | Primary state of the metro |
| `region` | Derived from `prim_state` | Categorical | Census region: Northeast, Midwest, South, West |
| `size_class` | Derived from total metro employment | Categorical | Small (< 250k jobs), Medium (250k to 1M), Large (> 1M) |
| `tot_emp` | OEWS | Quantitative | Number of software developers employed in the metro |
| `jobs_1000` | OEWS | Quantitative | Software developer jobs per 1,000 total jobs (tech concentration) |
| `loc_quotient` | OEWS | Quantitative | Location quotient: metro concentration vs. the national concentration (1 = same as U.S.) |
| `a_median` | OEWS | Quantitative | Nominal median annual wage for software developers ($) |
| `a_pct10` | OEWS | Quantitative | 10th percentile annual wage, a rough stand-in for entry-level pay ($) |
| `rpp_all` | BEA | Quantitative | Regional price parity, all items (U.S. = 100) |
| `rpp_housing` | BEA | Quantitative | Regional price parity for housing rents (U.S. = 100) |
| `real_median` | Derived | Quantitative | `a_median / (rpp_all / 100)`, the median wage in "average-U.S." dollars |
| `real_pct10` | Derived | Quantitative | `a_pct10 / (rpp_all / 100)` |

## Data example

The data are downloaded, so [`data_management.qmd`](data_management.qmd) imports both raw files and runs `glimpse()`. The rendered version is [`data_management.pdf`](data_management.pdf).

## EDA

**Import and wrangle**

- `readxl::read_excel()` for the OEWS file and `readr::read_csv()` for the BEA file, then `janitor::clean_names()` so the column names are consistent.
- `filter(occ_code == "15-1252")` to keep software developers, plus a second pull of `occ_code == "00-0000"` (all occupations) to get total metro employment for `size_class`.
- OEWS stores numbers as text and uses `*`, `**`, and `#` for suppressed or top-coded values. I'll use `na_if()` and `parse_number()` to convert them, and flag `#` (top-coded) wages separately so they aren't silently dropped.
- The BEA file is wide (one column per year) with quoted, padded codes. I'll use `str_trim()`/`str_remove_all()` on `GeoFIPS`, keep 2024, and `pivot_wider()` so all-items and housing RPPs are separate columns.
- `left_join()` OEWS to BEA on the CBSA code, then `anti_join()` to list metros that don't match so I can report what was lost and why.
- `mutate()` + `case_when()` to build `region` (via `state.abb`/`state.region`, with DC set to South by hand), `size_class`, and the real-wage columns.

**Visualize and summarize**

- Histograms or density plots of nominal vs. real median wage, to see how much the spread shrinks after adjusting.
- Scatterplot of `a_median` vs. `rpp_all` with `geom_smooth(method = "lm")`, to see whether pay keeps up with prices.
- Scatterplot of `real_median` vs. `jobs_1000`, colored by `region`, for the main research question.
- Side-by-side boxplots of `real_median` by `region` and by `size_class`.
- Bar chart (`slice_max()`) comparing the top 15 metros by nominal pay with the top 15 by real pay, which is the chart new grads will actually care about.
- `gtsummary::tbl_summary(by = region)` and `group_by()` + `summarise()` tables for group means, medians, and counts.

## Modeling

- **Model:** multiple linear regression.

  `real_median ~ jobs_1000 + region + size_class`

- **Why:** the response is quantitative and continuous, and I want the association with tech concentration while holding region and metro size constant. Region and size are categorical, so they enter as indicator variables.
- **Comparison:** I'll also fit the simple linear model `real_median ~ jobs_1000` to see how much region and size change the slope and the adjusted R².
- **Checking the model:** residual-vs-fitted plots and a histogram of residuals for the linear model assumptions; adjusted R²; an 80/20 train/test split to compare RMSE on held-out metros; and a check of high-leverage metros (San Jose will almost certainly be one).
- **Using it:** metros with large positive residuals pay more in real terms than the model predicts for their size and tech concentration. Those are the "hidden value" metros I'd recommend new grads add to their lists.
- **Limitations I'll address in the critique:** these are wages for all software developers, not new grads (I'll repeat the analysis with the 10th percentile as a rough entry-level proxy); RPPs average prices over all households, not a young renter's budget; OEWS wages are base pay only and leave out stock and bonuses, which matter a lot at big tech firms; and the BEA metro definitions may not match the 2023 OMB definitions OEWS uses for every metro.

## Communicate

- A Quarto HTML report published as a website (GitHub Pages), written for a graduating CS student rather than a statistician.
- It will lead with the recommendation (a short ranked table of metros that give the most real pay for tech workers) and then show the supporting plots, the model, and an honest limitations section.

## LLM disclaimer
- I plan on using LLMs to confirm my understanding of topics throughout the project and assist in cross cutting concerns (grammer, accuracy, etc...)