# Data Dictionary

Covers `cleaned_analysis_data.csv` (the final analysis-ready file). Columns inherited unchanged from earlier pipeline stages are marked with their origin.

## File lineage

```
boxoffice_data_2024.csv                (raw: Year, Title, Gross)
        ↓  enrich_with_budget_ratings_fast.py  (+ TMDB/OMDb columns)
enriched_with_budget_ratings.csv
        ↓  analyze_box_office.py  (+ derived columns)
cleaned_analysis_data.csv              (final, documented below)
```

## Columns

| Column | Type | Source | Description |
|---|---|---|---|
| `Year` | int | Raw | Theatrical release year, 1984–2024 |
| `Title` | string | Raw | Film title as listed by BoxOfficeMojo |
| `Gross` | string | Raw | Domestic box office gross, `$`-formatted (e.g. `"$234,760,478"`) — nominal (not inflation-adjusted), not yet numeric |
| `tmdb_matched_title` | string | TMDB | Title as matched on TMDB; may differ slightly from `Title` (e.g. punctuation, subtitle differences) — useful for spot-checking match quality |
| `budget` | float | TMDB | Production budget in nominal USD, as recorded by TMDB. `NaN` where TMDB has no budget on file (~31% of rows) — **not** the same as a $0 budget |
| `tmdb_revenue` | float | TMDB | Worldwide box office revenue per TMDB (may differ from `Gross`, which is domestic-only from BoxOfficeMojo) |
| `rating` | float | TMDB | TMDB audience vote average, 0–10 scale |
| `vote_count` | float | TMDB | Number of TMDB user votes backing `rating` — low vote counts make `rating` less reliable |
| `genres` | string | TMDB | Comma-separated genre list (e.g. `"Comedy, Crime, Action"`). **Not mutually exclusive** — a film can and often does carry multiple genre tags |
| `studio` | string | TMDB (derived) | Canonical studio name. If any of the film's TMDB-listed production companies matches a major-studio keyword (see `MAJOR_STUDIO_LOOKUP` in the enrichment script), that canonical name is used (e.g. "20th Century Studios"); otherwise falls back to the first-listed production company name |
| `is_independent` | bool | TMDB (derived) | `True` if no major-studio keyword matched anywhere in the film's production company list; `False` otherwise. This is a **name-matching heuristic**, not a verified financing/distribution classification |
| `rotten_tomatoes_critic_score` | float | OMDb (optional) | Rotten Tomatoes Tomatometer, 0–100. Populated only if `USE_OMDB = True` during enrichment — blank in the default pipeline run |
| `metacritic_score` | float | OMDb (optional) | Metacritic critic score, 0–100. Same optional-population note as above |
| `imdb_rating` | float | OMDb (optional) | IMDb audience rating, 0–10. Same optional-population note as above |
| `gross_nominal` | float | Derived (Step 2) | `Gross` parsed from string to numeric float, still in nominal (non-inflation-adjusted) dollars |
| `cpi` | float | Derived (Step 2) | Annual average CPI-U index value for the film's `Year`, per BLS historical tables. Used as the inflation-adjustment basis |
| `real_gross` | float | Derived (Step 2) | `gross_nominal` adjusted to constant 2024 dollars: `gross_nominal × (CPI_2024 / cpi)` |
| `real_budget` | float | Derived (Step 2) | `budget` adjusted to constant 2024 dollars using the same method. `NaN` wherever `budget` is `NaN` |
| `budget_tier` | string | Derived (Step 2) | One of four categories, assigned from `real_budget`: `"Micro (<$20M)"`, `"Mid ($20-70M)"`, `"Upper-mid ($70-150M)"`, `"Blockbuster ($150M+)"`. `NaN`/blank if `real_budget` is missing |
| `decade` | int | Derived (Step 2) | `Year` floored to the decade, e.g. `1987` → `1980` |
| `roi` | float | Derived (Step 2) | Return on investment: `(real_gross − real_budget) / real_budget`. Only computed where both `real_gross` and `real_budget` are present and `real_budget > 0` |
| `breakeven_estimate` | float | Derived (Step 2) | `real_budget × 2.0` — a rough industry rule-of-thumb breakeven threshold (production budget + marketing spend, roughly approximated). See Limitations in README |
| `is_hit` | float (0.0/1.0/NaN) | Derived (Step 2) | `1.0` if `real_gross ≥ breakeven_estimate`, `0.0` otherwise, `NaN` where ROI inputs are missing. Stored as float rather than boolean for compatibility with aggregation functions (mean = hit rate) |

## Notes on missingness

| Column | Approx. % missing | Why |
|---|---|---|
| `budget` / `real_budget` / `budget_tier` / `roi` / `is_hit` | ~31% | TMDB has no budget on file for many older or smaller films |
| `rotten_tomatoes_critic_score`, `metacritic_score`, `imdb_rating` | 100% by default | Only populated if `USE_OMDB = True` was set during enrichment |
| `is_independent` / `studio` | <1% | TMDB has no production company data for a small number of titles |

## Known caveats worth remembering when querying this data

1. **2024 has unusually low budget coverage (33% vs. ~69% typical)** and is excluded from trend/share charts in `analyze_box_office.py` by default (`EXCLUDE_YEARS_FROM_TRENDS`). If you query `cleaned_analysis_data.csv` directly, remember this exclusion isn't baked into the file itself — you'll need to filter `Year < 2024` yourself for trend-safe analysis.
2. **`genres` requires exploding before genre-level aggregation.** A naive `groupby('genres')` will treat `"Comedy, Drama"` as its own distinct category rather than counting toward both Comedy and Drama. Split on `", "` and explode first (see `analyze_box_office.py` for the pattern used throughout this project).
3. **`is_hit` is a heuristic, not ground truth.** Real studio profitability depends on marketing spend, ancillary revenue (streaming, merchandising), and international vs. domestic splits — none of which this dataset captures precisely.
