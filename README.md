# The Case for the Comeback: Mid-Budget Movies vs. the Blockbuster Strategy

A data-driven analysis of the decline of the $20–70M "mid-budget" film in Hollywood, 1984–2024, built to test whether the popular "studios have abandoned the middle for blockbusters" narrative holds up against real box-office data.

## TL;DR

- **8,145** U.S. theatrical releases analyzed, 1984–2024, sourced from BoxOfficeMojo and enriched with TMDB budget/studio/rating data
- Mid-budget's share of releases fell from **~50% (1984) to ~34% (2023)** — real, statistically significant (R²=0.57)
- Top 5 studios control **75%** of blockbuster releases vs. only **47%** of mid-budget releases — the middle is where independents still compete
- Counter-finding: blockbusters actually post a **higher** hit rate and median ROI than mid-budget in this dataset — the popular "mid-budget is safer" narrative doesn't hold on raw numbers; the stronger argument is market access and genre diversity, not ROI superiority
- Genre composition is strongly tied to budget tier in every decade tested (chi-square p < 0.001 throughout, strongest in the 2010s)

## Project structure

```
├── boxoffice_data_2024.csv              # Raw input: Year, Title, Gross (BoxOfficeMojo, 1984-2024)
├── enrich_with_budget_ratings.py        # Step 1 (sequential): TMDB API enrichment
├── enrich_with_budget_ratings_fast.py   # Step 1 (concurrent + checkpointed, recommended)
├── enriched_with_budget_ratings.csv     # Output of Step 1: adds budget, studio, rating, is_independent
├── analyze_box_office.py                # Step 2: cleaning, inflation adjustment, tiering, ROI/hit-rate, charts
├── cleaned_analysis_data.csv            # Output of Step 2: final analysis-ready dataset
├── summary_stats.txt                    # Key summary numbers from Step 2
├── charts/                              # Generated PNG charts (10 total)
├── mid_budget_movie_comeback.pptx       # 17-slide presentation deck with embedded speaker notes
├── presenter_talking_points.md          # Standalone presenter cheat sheet (deck + Power BI dashboard)
├── powerbi_dashboard_and_dax_guide.md   # DAX measures + dashboard page layout guide
├── powerbi_dark_marquee_theme.json      # Custom Power BI theme (dark, cherry/navy palette)
├── DATA_DICTIONARY.md                   # Column-by-column reference for cleaned_analysis_data.csv
└── README.md                            # This file
```

## Data pipeline

**Step 1 — Enrichment** (`enrich_with_budget_ratings_fast.py`)
Takes the raw `Year, Title, Gross` file and queries the [TMDB API](https://www.themoviedb.org/documentation/api) for each title to pull budget, revenue, audience rating, genres, and production studio. Classifies each film's studio against a major-studio keyword list to derive `is_independent`. Uses concurrent requests (~6 workers) with a token-bucket rate limiter and checkpointing every 200 rows, so an interrupted run can resume rather than restart.

Requires a free [TMDB API key](https://www.themoviedb.org/settings/api). Optionally also uses [OMDb](https://www.omdbapi.com/apikey.aspx) for Rotten Tomatoes/Metacritic critic scores (`USE_OMDB = True`); off by default.

```bash
pip install requests pandas --break-system-packages
python3 enrich_with_budget_ratings_fast.py
```

**Step 2 — Cleaning & analysis** (`analyze_box_office.py`)
Takes the enriched CSV and:
- Parses the `$`-formatted `Gross` string into numeric
- Inflation-adjusts budget and gross to 2024 dollars using annual BLS CPI-U
- Buckets films into 4 budget tiers (Micro/Mid/Upper-mid/Blockbuster), based on real (inflation-adjusted) budget
- Computes ROI and an estimated hit/flop flag (gross ≥ 2× budget heuristic)
- Excludes years with unreliable budget-data coverage (2024 by default — see Limitations) from trend charts
- Generates 5 core charts and a written summary

```bash
pip install pandas numpy matplotlib --break-system-packages
python3 analyze_box_office.py
```

## Key findings

See `summary_stats.txt` for the full numeric output and `presenter_talking_points.md` for narrative framing of each chart. Highlights:

| Finding | Direction |
|---|---|
| Mid-budget share of releases, 1984→2023 | ~50% → ~34% (declining, significant) |
| Top-5 studio concentration, Blockbuster tier | 75.4% |
| Top-5 studio concentration, Mid tier | 46.7% |
| Median ROI, Blockbuster vs. Mid | 1.39 vs. 0.72 (Blockbuster higher) |
| Hit rate, Blockbuster vs. Mid | 61% vs. 44% (Blockbuster higher) |
| Genre × budget-tier chi-square, every decade | p < 0.001 (significant), strongest in 2010s (Cramér's V = 0.286) |
| Comedy's share of Mid-budget releases, trend | −0.11 pct pts/yr (p=0.008, significant but low R²) |

## Limitations

- **Not a full release census.** Source data is BoxOfficeMojo's tracked releases (~200/year), which skews toward films with meaningful theatrical distribution — the smallest/most limited releases are likely underrepresented.
- **Budget data is incomplete.** ~69% of films have a TMDB-sourced budget; the rest are excluded from tier/ROI/hit-rate analysis. Coverage is notably worse for 2024 (33%) and 2020 (37.5%, plausibly a real COVID effect) — 2024 is excluded from trend charts by default.
- **Hit-rate is a heuristic**, not real studio financial data. It uses a rough industry rule of thumb (gross ≥ 2× budget) to proxy for marketing spend and breakeven — treat as directional, not precise.
- **"Independent" is inferred**, not verified. It's derived from matching the studio name against a fixed list of major studios, not from actual financing/distribution records.
- **Genre tags are not mutually exclusive.** A film tagged both Comedy and Drama counts toward both categories in genre-share analysis, so tier-level genre percentages can sum to more than 100%.
- **The popular "mid-budget is safer" narrative is only partially supported.** This dataset's own ROI and hit-rate numbers actually favor blockbusters on average — see `presenter_talking_points.md`, Slide 9, for the full honest breakdown.

## Data sources

- Box office data: [BoxOfficeMojo](https://www.boxofficemojo.com/), via the Kaggle dataset [Box Office Data (1984–2024)](https://www.kaggle.com/datasets/harios/box-office-data-1984-to-2024-from-boxofficemojo)
- Budget, studio, genre, rating: [TMDB API](https://www.themoviedb.org/documentation/api)
- Inflation adjustment: U.S. Bureau of Labor Statistics, [CPI-U historical tables](https://www.bls.gov/cpi/tables/)

## License / attribution

This is a personal/educational analysis project. Box office and metadata are sourced from third-party APIs and datasets under their respective terms of use (TMDB, BoxOfficeMojo/Kaggle). This repository does not redistribute TMDB's raw API responses beyond the derived, aggregated figures used in analysis.
