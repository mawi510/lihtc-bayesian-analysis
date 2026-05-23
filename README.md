# Does Building Affordable Housing Raise Local Rents?

**A Bayesian hierarchical analysis of America's largest affordable-housing program**

Housing affordability has become a major political talking point, and the debate usually
centers on market-rate housing. When affordable housing programs do come up, they tend to be
criticized with little empirical evidence behind the claim. I wanted to measure the actual
relationship between the Low-Income Housing Tax Credit (LIHTC) and local housing costs. LIHTC
is the largest affordable-housing program in the country (about $10.5B/year and 3.7M+ units
since 1986), which makes it the natural place to ask the question: as a county builds more
LIHTC units, what happens to its home values and rents?

The effect turns out to be local rather than national. State coefficients vary enough that
"the national effect of LIHTC" is the wrong question to ask. After controlling for population
growth and deflating prices with CPI-less-shelter, I find a modest positive association.

| Outcome | Association per 1% more LIHTC per capita | 95% HDI |
|---|---|---|
| Home values (ZHVI) | **+0.13%** | [0.09%, 0.17%] |
| Rents (ZORI) | **+0.32%** | [0.29%, 0.36%] |

I model this with a **Bayesian hierarchical (partial-pooling) regression**: county-level
intercepts and state-level slopes, so each county borrows strength from its state instead of
collapsing into one national average. The positive sign is the part worth sitting with. More
rental supply should lower rents, so the result more likely reflects where credits get
allocated (high-demand, high-growth markets) than a price effect of the units themselves.

<p align="center">
  <img src="reports/figures/housing_plot_forest.jpg" width="520"><br>
  <em>Each row is a state's posterior LIHTC slope on home values, 95% HDI. Several states sit
  on zero (OH, OK), others run well past it (HI, DC, CA, CO).</em>
</p>

📄 **[Read the full write-up (PDF)](reports/LIHTC_Bayesian_Analysis_Report.pdf)**

---

## What this project demonstrates

**Bayesian / statistical modeling**
- Hierarchical (partial-pooling) model with county intercepts and **state-varying slopes**,
  built in **PyMC**.
- Priors set from the data where it helps. `β0 ~ Normal(12, τ=0.25)` comes from the empirical
  log-price distribution, and a tighter `τ_α` prior stabilizes the sparse rental panel.
- **MCMC diagnostics** for every parameter: trace plots, R-hat near 1.00, and bulk/tail ESS.
- Formal **model comparison via LOO** (leave-one-out cross-validation, ELPD), which picks the
  inflation-adjusted hierarchical model over the pooled and nominal alternatives.
- The rental panel is sparse (many counties with only a few years), which is exactly where
  partial pooling and a data-informed prior earn their keep. The model still recovers credible
  county and state effects.

**End-to-end data engineering**
- Joined **six public datasets** into two clean analysis panels on county **FIPS**:
  - HUD LIHTC project database (54k+ projects, 1987 to 2023)
  - Zillow Home Value Index (ZHVI) and Observed Rent Index (ZORI), monthly to annual
  - Census Bureau population estimates across three vintages (intercensal 2000 to 2009,
    2010 to 2019, and 2020 to 2023 via USDA ERS)
  - FRED CPI "All Items Less Shelter", chosen to deflate prices without conditioning on the
    outcome I'm trying to measure
- Feature engineering: **cumulative** LIHTC units (supply persists across years) scaled
  **per 1,000 residents**, plus real (inflation-deflated) price indices.
- Documented the data-quality calls. About 21% of LIHTC projects dropped for invalid county
  FIPS, and ZORI is restricted to counties with 5+ valid years.

---

## Results at a glance

- **Inflation drives most of the raw signal.** The nominal home-value association (about 0.40%)
  drops to 0.13% once prices are deflated. LOO strongly prefers the inflation-adjusted model
  (weight near 0.99).
- **State heterogeneity is real.** High-cost, high-demand states (HI, CA, CO) show the
  strongest positive slopes. Several lower-cost states (OH, OK) have 95% credible intervals
  overlapping zero. A single pooled slope hides all of this.
- **Rents respond more than home values** (0.32% vs. 0.13%), which fits LIHTC being a rental
  program. The positive sign still points to allocation and demand confounding, not a supply
  effect.
- **Not causal.** This is an observational association. The most likely story is that credits
  flow to markets with demonstrated need (job growth, wage growth, zoning) beyond what
  population growth captures.

---

## Repository structure

```
.
├── notebooks/                       # run in numeric order
│   ├── 01_build_population_data.ipynb   # Census vintages into a unified county panel
│   ├── 02_build_inflation_data.ipynb    # FRED CPI into an annual deflator
│   ├── 03_build_panels.ipynb            # join LIHTC + Zillow + pop into housing & rental panels
│   ├── 04_housing_value_models.ipynb    # ZHVI: pooled vs. hierarchical, LOO, diagnostics
│   └── 05_rental_models.ipynb           # ZORI: same workflow on the sparse rental panel
├── data/
│   ├── raw/                         # source files from HUD, Zillow, Census, FRED
│   └── processed/                   # analysis-ready panels built by 01 to 03
├── reports/
│   ├── LIHTC_Bayesian_Analysis_Report.pdf
│   └── figures/                     # trace, forest, and posterior plots
└── requirements.txt
```

## Reproducing

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter lab            # run notebooks/ 01 to 05 in order
```

Notebooks read from `data/raw/`, write intermediate panels to `data/processed/`, and save
figures to `reports/figures/`.

## Methods & tools

`PyMC`, `ArviZ`, `pandas`, `NumPy`, `matplotlib`. Bayesian hierarchical modeling, MCMC (NUTS),
LOO/ELPD model comparison, posterior and HDI inference.

## Next steps

- **Push toward causality.** Instrument LIHTC allocation (Qualified Census Tract eligibility,
  per-capita credit ceilings), or run a difference-in-differences design around project
  completion dates.
- **Add real demand controls.** Job and wage growth, vacancy rates, and zoning/permitting
  measures, so the slope stops absorbing unobserved demand.
- **Model space.** Spatial random effects or distance decay, so neighboring counties inform
  each other rather than only same-state counties.
- **Separate dose from timing.** A distributed-lag specification to split construction-period
  effects from the persistent supply effect.

## Data sources

HUD LIHTC Database, Zillow Research (ZHVI, ZORI), Census Bureau Population Estimates, USDA ERS,
and FRED (St. Louis Fed, series `CUUR0000SA0L2`). Full citations are in the
[report](reports/LIHTC_Bayesian_Analysis_Report.pdf).
