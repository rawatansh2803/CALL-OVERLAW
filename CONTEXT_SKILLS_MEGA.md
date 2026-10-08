# MEGA FILE — Context + Skills + Master Prompt
## Project: Dynamic Index Call Overlay (systematic NIFTY 50 call overwriting) on a diversified, non-index Indian equity portfolio

This single file is the **complete handoff**. It merges (a) **Context** — everything done from start to
finish, every decision, every result, every mistake and correction — and (b) **Skills** — the exact
environment, data, formulas, code conventions and verification steps a new AI agent needs to **re-implement
this more accurately and more rigorously**. Read it fully before writing any code. Section 0 is a short
**master prompt** you can paste to kick off a fresh agent.

---

## 0. MASTER PROMPT (paste this to start a new agent)

> You are a senior quantitative developer re-implementing, from scratch and with maximum rigor, an empirical
> study of a **Dynamic Index Call Overlay**: systematically selling NIFTY 50 index call options over a
> diversified, **non-index** Indian equity portfolio, evaluated strictly **after Indian taxes and costs**.
> First read `/home/user/CONTEXT_SKILLS_MEGA.md` in full — it is the source of truth for environment, data,
> formulas, conventions, past mistakes and verification targets. Then:
>
> 1. **Restore the environment** exactly as Section 2 dictates (deps in `.local/pkgs`, `--no-deps`, never
>    shadow system packages).
> 2. **Ingest data faithfully**: build the 2010–2020 traded set from the Drive `processed/` files, AND build a
>    **real 2020–2026** traded set by stream-parsing the Drive `raw/` daily bhavcopies for NIFTY CE rows
>    (do NOT model 2020–2026; do NOT keep the multi-GB raw on disk; do NOT recreate `data/raw_gdrive_options`).
> 3. **Re-implement** the modules in Section 5 with the exact formulas in Section 6 and dated regulatory rules
>    in Section 7. Use **integer lot sizing** and **walk-forward estimation** with an **untouched 2025–26 holdout**.
> 4. **Run controls**: for every risk metric (beta, tail-beta κ⁺), also compute it on an index-like top-5
>    mega-cap basket and on NIFTY-vs-NIFTY; the method must return ≈1.0 on the controls or it is broken.
> 5. **Never write a number from memory.** Read every reported figure from an on-disk artifact; verify against
>    the targets in Section 9 (tolerances given). Flag anything that cannot be verified.
> 6. **Report honestly**: the two samples have historically disagreed in sign (+ on 2021–26, − on 2010–20).
>    Re-derive both on real data; if they still disagree, say "regime-dependent, not deployable", do not pick a
>    side. Deliver: a results report (PDF+MD), all figures, and a verification log.
>
> Definition of done: Section 9's verification script prints PASSED on real data; the report states, per
> pre-committed criterion, pass/fail/inconclusive with measured values; all gaps are flagged, none fabricated.

---

## 1. CONTEXT — what this project is and what happened, start to finish

### 1.1 The idea (plain words)
Hold a diversified Indian equity book; each month **sell** some NIFTY 50 call options to collect premium
("rent"). Rent cushions flat/falling markets; but in strong rallies the short call gives up upside. Question:
after Indian taxes (F&O at 35.88% vs equity LTCG 10–12.5%), STT, and 50%-cash-margin drag, does the rent win?

### 1.2 Timeline of what was done
1. Reconstructed a 25-name, non-mega-cap, value-momentum equity book (₹2 Cr, top-16 equal-weight, semi-annual
   rebalance) from yfinance prices → `data/reconstructed_portfolio.parquet`.
2. Built the dated regulatory/tax schedule (`src/data_fetcher.get_historical_contract_spec`).
3. Implemented risk analytics: rolling 252-day beta, upside/downside beta, OOS beta stability, upside tail-beta
   κ⁺, Yang–Zhang RV + overnight-gap split, HAR/EWMA/GARCH vol forecasts, CR(δ) friction screen
   (`src/mapping_and_richness.py`, `src/volatility_engine.py`).
4. Built two backtesters: 2021–26 integer-lot engine (`src/backtest_engine.py`) and a traded-LTP engine for the
   2010–20 Google Drive archive (`src/gdrive_strategy_backtester.py`), plus stress/sensitivity/reporting.
5. Downloaded the Drive `processed/` files → 1,643,069 records / 2,524 days (2010–2020).
6. Ran 8 strategies on both samples; produced figures 1–9, CSVs, an attribution table, and a 19-page PDF report.
7. **Self-audit found an earlier draft had memory-written (wrong) numbers**; every figure was re-read from
   artifacts and corrected (see 1.5).
8. **Discovered the Drive also has a `raw/` subfolder** (daily NSE F&O bhavcopies, 2001→2026, real NIFTY CE
   rows). The 2021–26 leg had been **modelled** (NSE blocks servers); the correct upgrade is to re-run 2020–26
   on the real `raw/` data. This is the main open task for the new agent.

### 1.3 Headline results (as last verified on artifacts)
- **2021–26 (modelled options):** every overlay slightly beats unhedged; best Static High **32.82%** post-tax vs
  **31.53%** unhedged; MaxDD −20.48% → −17.86%; survives the 2025–26 holdout (+3.56 pp).
- **2010–20 (real traded):** every overlay **loses** −₹14.6 L to −₹48.4 L on ₹2 Cr; calls paid out more than
  collected; but crash drawdown cut −38.44% → −12.98%.
- **Conclusion:** sign is **regime-dependent** → not deployable as a standalone alpha strategy; it is a
  drawdown/bear-market cushion. Biggest lever = early roll at delta 0.50 (+8.1 pp/yr).
- **Decision criteria:** tradeable band δ∈[0.10,0.50] PASS; sizing FAIL (realised-q error 42–55%); exposure
  mapping FAIL (stability 0.346, κ⁺ 0.495–0.715); after-tax INCONCLUSIVE (samples disagree).

### 1.4 Why the findings are real (control experiments, re-run with yfinance)
Same computation on three "portfolios": κ⁺(+1%) = book **0.676**, top-5 mega basket **1.142**, NIFTY-vs-NIFTY
**1.004**. β-OOS-stability = book 0.346, mega basket 0.158. ⇒ Method is sound (controls ≈1); low tail-beta is a
property of the non-index book; beta instability is general; sizing coarseness is arithmetic.

### 1.5 Mistakes already made and corrected (do NOT regress these)
Wrong→Right: No-Overlay post-tax CAGR 30.54%→**31.53%** (vol 13.43→18.44, Sharpe 1.790→1.404, MaxDD
−16.59→−20.48); 2021–26 overlay "underperforms"→**outperforms**; holdout vol 11.23→18.20, MaxDD −10.42→−19.18;
attribution omitted **equity LTCG −₹1,50,02,058 (−53.11%)**; net deriv contrib 1.77L→**₹1,71,240**; initial book
2.00Cr→**₹2,82,48,578 pre-tax**; margin drag "−2.36% annualised"→**−2.36% cumulative over 5.54y**; "60-day beta"→
**252-day**; overnight fraction 32.8→**33.14%**; VRP "8/11"→**9/11, median +1.02**; κ⁺0.648 is the
**high-rally-concentration** conditional (N=103), unconditional 0.495–0.715; CR<0.10 band [0.12,0.40]→**first met at
δ=0.10**; melt-ups "lose money"→**all net positive, cost 0.24–2.83 pp forgone upside**; sizing ±24.3%→**42–55%
mean**; lot history 50→25→75→50→**75→50→25→65**; criterion-4 FAILS→**inconclusive**.
Also: the **P&L attribution does not reconcile** (components ₹96,937,068 vs stated ₹101,154,432; residual
₹42.17L because the cash account isn't a line) — keep it flagged, don't silently "fix".

---

## 2. SKILLS — Environment

- Python 3.13; workdir `/home/user`.
- System site-packages ALREADY has: numpy, pandas, matplotlib, seaborn, scipy, fontTools, lxml, bs4, requests,
  PIL. It LACKS: pyarrow, gdown, reportlab, statsmodels, yfinance (+deps).
- Install non-system deps to an **excluded** path (`.local` is snapshot-excluded and **does not persist** between
  sessions — reinstall at session start):
  ```
  pip install --target=/home/user/.local/pkgs --no-deps \
    reportlab pyarrow gdown statsmodels yfinance \
    patsy formulaic interface_meta multitasking websockets peewee protobuf filelock PySocks curl_cffi \
    beautifulsoup4 soupsieve tqdm filelock
  export PYTHONPATH=/home/user/.local/pkgs
  ```
- `--no-deps` is essential: installing numpy/pandas/PIL into `.local/pkgs` creates broken partial copies that
  shadow system ones (`ImportError: _imaging`). If seen, delete that package dir from `.local/pkgs`.
- For PDF: register **DejaVu Sans** (has ₹ U+20B9). No `DejaVuSans-Oblique.ttf` — alias italic to regular.
- **Workspace cap ~128 MB.** Keep tracked dirs small: the 24.5 MB parquet + figures + outputs ≈ 45 MB is fine.
  NEVER put big caches in tracked dirs; put downloads in `.cache/` or `.local/` (both excluded). NEVER recreate
  `data/raw_gdrive_options/` (127 MB dead cache, was deleted, is not read by any script).

## 3. SKILLS — Data

### 3.1 Drive layout (folder `1tqqUoPsRpFB6w1Wom6df84E8Gkz_V0gw`)
- `processed/`: `spot_prices.csv`, `ATM_CE_options.csv`, `CLEANED_CE_options.csv` (2014–18),
  `NIFTY_CE_2014_2018.csv` (2010–13 + 2019–20), plus category-mapped/BANKNIFTY variants. → use for 2010–2020.
- `raw/`: **daily NSE F&O CE bhavcopies, 2001→2026** (filenames `fo_YYMMDD_CE.csv`). Each has real NIFTY CE rows.
  Columns: `INSTRUMENT,Symbol,Expiry,Strike Price,Option Type,Open,High,Low,Close,Settlement Price,Volume,
  Turnover,Open Interest,CHG_IN_OI,Date,Unnamed: 15,LTP`. → use for a REAL 2020–2026 set.
- `logs/`: `errors.log`, `scraper.log`, `validation_report.txt`.

### 3.2 Building the 2010–2020 traded set (already done; reproduce via `download_and_process_gdrive_data.py`)
Filter `Symbol=='NIFTY'`, dedupe on (Date,Expiry,Strike), map spot, derive Moneyness/Zero_Volume, write
`data/nifty_historical_options_2010_2020.parquet` (1,643,069 rows). Verify counts.

### 3.3 Building a REAL 2020–2026 set from `raw/` (the key rigor upgrade)
Stream: download `raw/` into `.cache/raw` (excluded), and **as each daily file lands**, read it, keep rows with
`Symbol=='NIFTY'` & `Option Type=='CE'`, keep the needed columns, append to a parquet, then **delete the file**
so the multi-GB raw never accumulates. Parse `Expiry`/`Date` (dd-Mon-yyyy), `Strike Price`, OHLC/Settlement/LTP,
`Volume`,`Turnover`,`OI`. Map a spot series for NIFTY (from `processed/spot_prices.csv` if it extends past 2020,
else yfinance `^NSEI`). This yields real traded prints for 2020–2026 to replace the modelled leg.

### 3.4 What is NOT available (flag, never fabricate)
NSE live quotes (HTTP 403 to servers) — so any pre-raw 2021–26 option price is modelled; real bid-ask (we used a
(High−Low)/mid proxy); actual client ledger (book is a reconstruction); broker margin statements; the stress
matrix's `Upside_Gap_G_INR` (0.0 in all 20 rows).

## 4. SKILLS — The portfolio (names = assumption; weights = rule)
Universe (25, chosen, non-mega-cap): BEL, CUMMINSIND, DIXON, HAL, PERSISTENT, POLYCAB, TRENT, COALINDIA, NTPC,
SUNPHARMA, BAJFINANCE, M&M, BSE, ONGC, POWERGRID, VEDL, FEDERALBNK, AUROPHARMA, ASHOKLEY, CHOLAFIN, APOLLOHOSP,
VOLTAS, BHARATFORG, OBEROIRLTY, COFORGE.
Rule: every 6 months score = (12-1 momentum)/(1-yr vol); hold top-16 equal-weight; 0.15% rebalance cost. Output:
`reconstructed_portfolio.parquet` (1,662 days 2020-01-02→2026-09-21), `portfolio_weights.parquet`.
Latest weights (computed): AUROPHARMA 7.16, APOLLOHOSP 6.71, ONGC 6.53, BAJFINANCE 6.53, FEDERALBNK 6.52,
SUNPHARMA 6.51, ASHOKLEY 6.46, M&M 6.40, COALINDIA 6.35, BEL 6.22, BHARATFORG 6.07, POWERGRID 6.00, NTPC 5.98,
CUMMINSIND 5.85, POLYCAB 5.58, BSE 5.21.

## 5. SKILLS — Modules (what each must do)
`data_fetcher`(regulatory schedule+ingest) · `portfolio_reconstruction`(book+rally concentration) ·
`volatility_engine`(Yang–Zhang RV, gap split, forecasts) · `mapping_and_richness`(betas, stability, κ⁺, CR(δ)) ·
`backtest_engine`(2021–26 integer-lot, costs, 8.5% margin drag, taxes) · `gdrive_strategy_backtester`(traded-LTP) ·
`stress_testing`, `sensitivity_analysis`, `reporting`(metrics, holdout@2025-01-01, attribution, regime, figs1–3) ·
`generate_extended_figures`(figs4–7) · `empirical_gdrive_validation`(fig8, surface/VRP) ·
`run_gdrive_strategy`(fig9, 2010–20 metrics) · `build_pdf_report`.
Strategies (8): NoOverlay; StaticLow q=.20 K=1.03S; StaticHigh q=.50 K=1.03S; FixedDelta q=.25 K=S(1+.20σ√T);
RichnessGated q=.25 if IV/RV≥1.15 else 0; TailConstrained q=min(.25,κ⁺·.40) K=1.04S; Dynamic (gate IV/RV≥1.05; if
σ21>.22 q=.35 K=1.05S else q=.25 K=1.03S); CallSpread (short .20δ + long wing).

## 6. SKILLS — Formulas (exact)
- Beta (252d): `Cov(rp,rm)/Var(rm)`. Upside/downside: same restricted to rm>0 / rm<0. OOS stability: corr(252d
  beta, forward-63d realised beta). Tail-beta κ⁺: OLS slope of rp on rm restricted to rm≥threshold (p75,p90,
  +1%,+1.5%,+2%); also condition on high rally-concentration days.
- Yang–Zhang daily var: `var_o + k·var_c + (1−k)·var_rs`, `k=0.34/(1.34+(n+1)/(n−1))`; annualise ×252; gap_share
  = var_o/var_yz.
- VRP = median(ATM IV − forward RV) by year. IV = Black–Scholes implied (invert via brentq).
- CR(δ) = (fixed costs + spread + proportional taxes)/premium; tradeable if <0.10.
- Lot sizing: `lots = floor(q·(V·β)/(S·lot_size))`. Integer only.
- Metrics (`reporting.calculate_performance_metrics`, rf=0.0564): years=n/252; CAGR=(end/start)^(1/years)−1;
  vol=dailyσ·√252; Sharpe=(CAGR−rf)/vol; Sortino uses downside σ; MaxDD from cummax; CVaR95 weekly.

## 7. SKILLS — Dated regulatory schedule (apply per date)
Lot 75(<2021-05-01)→50(→2024-04-26)→25(→2025-11-20)→65. Expiry Thu(<2025-09-01)→Tue. STT sale 0.0625%
(<2024-10-01)→0.10%(→2026-04-01)→0.15%. Min contract ₹5L→₹15L(2024-11-20)+2% expiry ELM. Calendar-spread-on-expiry
withdrawn 2025-02-10. F&O tax 35.88%. Equity LTCG 10%→12.5%(2024-07-23), STCG 15→20%, exemption 1L→1.25L. Margin
SPAN/ELM 11%, 50% min cash, 20% pledge haircut. Exchange .0495%+GST18%+SEBI .0001%+stamp .003% buy+₹20/order.
Margin funding drag 8.5%/yr on cash margin.

## 8. SKILLS — Reproduce (ordered, idempotent)
```
export PYTHONPATH=/home/user/.local/pkgs
python3 download_and_process_gdrive_data.py      # 2010-20 parquet
python3 src/empirical_gdrive_validation.py       # fig8 + surface/VRP
python3 run_gdrive_strategy.py                   # fig9 + 2010-20 metrics
python3 -m src.reporting                         # figs1-3, CSVs, attribution
python3 src/generate_extended_figures.py         # figs4-7
python3 build_pdf_report.py                      # PDF
```

## 9. SKILLS — Verification targets (assert within tol)
2021–26: NoOverlay post-tax CAGR .3153 / vol .1844 / Sharpe 1.404 / MaxDD −.2048; StaticHigh CAGR .3282; opening
pre-tax book 28,248,578.18. Holdout: NoOverlay .0481, StaticHigh .0837. 2010–20: NoOverlay CAGR .0936 Sharpe .159
MaxDD −.3844; FixedDelta net-deriv −4,843,760; StaticHigh −4,581,646. Micro: stability .3462; κ⁺high-conc .6483
(N103); gap_share .3314; VRP median +1.02, 9/11; zero-vol 61.87% (1.10–1.25) / 76.74% (>1.25); CR<0.10 first at
δ=.10. Attribution total 101,154,432 (components 96,937,068; residual 4,217,365 — flag, don't hide). Sensitivity:
spread 1×→3× CAGR .3202→.3202; delta_stop .4014 vs hold .3202; tax 35.88%→.3151. Parquet rows 1,643,069.
(See `AGENT_HANDOFF_AND_REPRODUCTION_GUIDE.md` §8 for a ready verification script.)

## 10. SKILLS — Rigor checklist for the NEW run (do better than last time)
- [ ] Real 2020–26 from `raw/` (no modelling), streamed to stay under disk/cap.
- [ ] Controls for every risk metric (index-like basket, NIFTY-vs-NIFTY) ≈1.
- [ ] Every reported number read from an artifact; verification script PASSED on real data.
- [ ] Walk-forward + untouched 2025–26 holdout; dated rules; integer lots.
- [ ] Attribution either reconciles or the residual is explicitly shown.
- [ ] Honest verdict: if samples disagree in sign → "regime-dependent", not a picked side.
- [ ] All gaps flagged (403, bid-ask proxy, gap-drag 0.0); nothing fabricated.
- [ ] Workspace kept <128 MB tracked; big caches in `.cache`/`.local` only.

## 11. Companion files
`AGENT_HANDOFF_AND_REPRODUCTION_GUIDE.md` (repro+verify), `Project_Explained_In_Plain_English.md` (plain
explanation), `Dynamic_Index_Call_Overlay_Final_Presentation_Report.md` + `..._Final_Report.pdf` (results),
`Dynamic_Index_Call_Overlay_Research_Report.md` (original study).
