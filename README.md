# Taxi-out time prediction for the PRC Data Challenge 2026

> **This repository is the frozen competition release (final phase, 14 October 2026).** The project continued after the
> challenge closed, with ADS-B ground tracks as per-flight features in a second-stage corrector (15 to 30 s RMSE on held-out
> months). Current code and results: [BlendiFevziu/prc2026-taxi-out-adsb](https://github.com/BlendiFevziu/prc2026-taxi-out-adsb).

Team **upbeat-goblin**. The challenge, run by the EUROCONTROL Performance Review Commission with the
OpenSky Network, asks for the taxi-out time of departures at ten European airports (EDDF, EDDM,
EGLL, EHAM, LEBL, LEMD, LFPG, LIRF, LSZH, LTFM), learned from the 2025 movements and scored by
RMSE. The first phase scored January and July 2026 (344,841 departures); the final phase, one
single submission, scores January, February, June and July 2026 (670,790 departures). This
repository holds the complete code under GPLv3. The competition data is not included; see
`data/README.md` and `SETUP.md`.

## Start here

- **What was submitted:** one file for 670,790 departures; how it is composed is in "Final submission" below, the code
  that builds it in `research_800_final/` (its own `README.md` lists every script).
- **The model in one paragraph:** the airport's block time is mostly the Network Manager's off-block time (`AOBT_3_flt`);
  an ensemble predicts the difference between the two, a separate lane handles departures without it, and a small set of
  rules handles Rome's schedule-copy labels. Open ADS-B tracks refine some airports.
- **How it was judged:** every change had a rule written before the run and was tested on months or days it was not tuned on;
  the research folders keep the rules (`PREREG*.md`), the numbers (`results/`) and the rejected ideas. A map of all 46
  folders is in `RESEARCH_INDEX.md`.
- **How to re-run the pieces** is under "Reproducing the results".

## Results

- **Model score in the first phase:** 245.046 s RMSE (submission v80, 1 October 2026), January and
  July only. Earlier scores are in `results/`.
- **The final submission has two sources** (section "Final submission" below): January and July come
  from v80, February and June from the same frozen design without the January/July-specific
  ADS-B layers, because those layers need aircraft tracks of the scored days and trained
  models for the month. Expect a clear gap between the two pairs of months.

## The key observation

For 98.5 % of the departures to score the flight record carries the Network Manager's actual
off-block time `AOBT_3_flt`, and the movement record carries the runway time `MVT_TIME_UTC_mvt`.
Their difference is already a close first guess of the taxi time. What remains to predict is how
far the airport's own block time differs from the Network Manager's:

    TAXITIME_SEC_mvt - (MVT_TIME_UTC_mvt - AOBT_3_flt)  =  AOBT_3_flt - BLOCK_TIME_UTC_mvt

That residual is a disagreement between two reporting systems rather than a physical movement.
Physical conditions such as weather or the runway direction are already inside the first guess,
while stand, operator and the relationships between planned and actual timestamps carry most of
the remaining signal.

## Model (the base, identical for all four months)

**Main lane** (departures with `AOBT_3_flt`): prediction = first guess + predicted residual, from
91 features (timestamps and their differences, stand, runway, operator, aircraft type, the
turnaround at the stand derived from arrival rows, arrival taxi medians, flight-ID sequence
features, correctly timed runway traffic counts). Ensemble with fixed weights: neighbour sequence
networks 0.35 (five seeds; they replaced the plain network in v47 and read the movements around each
departure), CatBoost 0.39 (three seeds), LightGBM 0.182 (five seeds) and per-airport LightGBM 0.078
(ten airports). The weights follow from the blend order (airport models 0.30 into LightGBM, CatBoost
0.6 against that, networks 0.35 against all) and were set in v50 and v51. Prediction floored at 300 s.
About 4.5 % of the main-lane training departures carry a block time copied from the schedule; a classifier estimates that probability p and the
prediction moves towards `MVT_TIME - SCHED_TIME` with weight p^2.5.

**Fallback lane** (1.5 %, no `AOBT_3_flt`): a classifier for extreme taxi times, a height estimate
for the extremes, and a normal branch anchored on the median `MVT_TIME - AOBT_3` of other departures
at the same airport within 60 or 180 minutes, plus a learned offset (v46).

**Rome rules**, because Rome is where a handful of rows decide the RMSE (48.5 % of its fallback
rows are schedule copies in 2025, and some labels exceed 20 hours):
a copy gate (fallback rows whose copy probability reaches a 0.99 precision chosen inside the
training months get the schedule delay; 8 rows in January/July), a rule for delays above 8 h without
a local-midnight crossing (the delay itself, 10 rows) and, for delays between 1 and 8 h, the mean of
the model value and a label-mechanism classifier's expected value (v70, chosen after seeing its
first test fail and therefore disclosed as post hoc). All three use delay, crossing and 2025 labels
only. `research_800_final/four_month_base.py` applies the whole base to any ranking file and
reproduces the scored v47 and v70 artefacts to better than 0.001 s before it is used.

Code: `main_model.py`, `research_250/` (the 91-column schema), `research_260/` (sequence networks),
`research_240/` (fallback anchor), `research_280/` (fallback and Rome screens; the copy gate itself is
`rome_copy_gate` in `research_800_final/four_month_base.py`), `research_500_midnight/`,
`research_510_rome_classifier/`, `research_800_final/` (the four-month pipeline).

## ADS-B layers (open data, January and July)

On January and July 2026 the base is refined by layers built on open ADS-B tracks (adsb.lol, ODbL)
near the airports: stand detectors that read the off-block moment from where an aircraft was parked
and when it moved (Munich, Amsterdam), partial-track residual corrections, a stand model, an
Amsterdam January add-on, drift-robust variants and detectors for rows without `AOBT_3_flt` at four
airports. They are in `research_290`, `research_300`, `research_320`, `research_360_integration`,
`research_400_cells`, `research_440_fresh_july`, `research_450_stand_january`, `research_480_weights`,
`research_550_review`, `research_590_stand_new`, `research_640_tail_rule`, `research_650_eham_cat`.
Layers were pre-registered (rules written before the run) and tested on dates they were not tuned on;
each had to hold for January and July separately and without its most favourable rows (the top 100 rows,
later an airport-day bootstrap). Two exceptions are recorded: the v70 Rome slice was chosen after its first
test failed, and on 1 October the Paris and Amsterdam January cells were admitted retroactively under the
bootstrap rule (`research_640_tail_rule/POLICY_tail_rule.md`). Together with the Rome rules and the later weight settings they took the January/July score
from 266.94 s (v47, the last scored version with the base alone) to 245.05 s (v80).
Row ownership of every layer is documented in `research_360_integration/PLAN.md`.

## Final submission (14 October 2026)

`research_800_final/assemble_final.py` builds `upbeat-goblin_final.parquet`:

| rows | months | source |
|---|---|---|
| 344,841 | January, July | v80 (base plus ADS-B layers) plus the known shift of `MVT_TIME_UTC_mvt` |
| 325,949 | February, June | the base above; on 11,320 rows (Munich 7,443, Amsterdam 3,877) replaced by the stand detector's taxi time |

The final dataset dithered `MVT_TIME_UTC_mvt` by up to 12 s on 92 % of the January/July rows (every
`_flt` column and `SCHED_TIME_UTC_mvt` are unchanged), so v80's predictions move with the shift.
The base reproduces the January/July plain-base rows of v80 (47 % of rows agree within 15 s, as
the coverage audit of 2 October predicted for rows no layer touches).

Why February and June get no partial-track and stand layers: those layers are trained on aircraft
tracks of the same months in 2025 together with out-of-sample base predictions for those months, and
need the 2026 tracks of every scored day (about 3.3 GB per day). The tracks for February and June
were fetched on 13 October; the stand detectors of Munich and Amsterdam, which need no
training, are checked on February and June 2025 days before they are used
(`research_800_final/detector_feb_jun.py`).

What was applied to February and June 2026: the two training-free stand detectors, nothing else.
Check on 2025 days (six in February, six in June, `research_800_final/results/`): against a single
LightGBM trained without the month (RMSE 220 s in February and 214 s in June on main-lane rows, so
weaker than the production ensemble), the Munich detector has RMSE 60.4 s against 161.8 s on its
1,494 rows and the Amsterdam detector 71.7 s against 160.3 s on 655 rows; both pass the four rules
(pooled gain, each month, without the best 5 % of rows, bootstrap interval above zero) in February
and June separately. Because the comparator is weaker than the ensemble, these gains overstate the
real ones; on January/July the same detectors realised 1.5 to 1.6 times their validated gain over a
held-out full ensemble, which is why they were kept. The detectors reach about a third of Munich's and
under a tenth of Amsterdam's main-lane departures (an estimate needs an aircraft seen parked at its
assigned stand). Tracks were available for 57 of 58 days; 11 June 2026 has no adsb.lol release, so its
rows keep the base prediction. The partial-track residual correction, the stand model and the
Amsterdam January add-on were not extended to February and June.

## Reproducing the results

- **Re-runnable from this repository with the competition data:** the base (`research_800_final/four_month_base.py`,
  after training the models as described in `SETUP.md` sections 3 to 5 and 6b, which took more than a day on the
  reference machine), the replication gates, the February/June detector check and application, the assembly, and every
  screen in the research folders. The gates are the evidence that the code reproduces what was scored.
- **The January/July part of the final file** is the scored submission v80. It was built as a chain of about 30 steps
  (v54 to v80), each of which reads the previous candidate file and cached intermediate tables (track summaries, saved
  layer models). Every step's script, its pre-registered rules and its aggregate results are here (`RESEARCH_INDEX.md`;
  `research_360_integration/PLAN.md` for row ownership). Cached intermediate tables and trained model files are not
  published (the main model is 1.7 GiB).
- **Random elements:** tree and network seeds are fixed in the code, but a new run on other hardware produces different
  hashes and differences at rounding level (`SETUP.md` section 5); the replication gates therefore carry tolerances
  (0.001 to 0.05 s).
- **Data that cannot be redistributed:** the competition files (access keys of the organisers) and the adsb.lol archives
  (several GB per day; the tools to fetch and reduce them are in `adsb-lol/`).

## What did not help

Measured and rejected, with scripts and aggregate results in the research folders: METAR weather
and runway operating direction, arrival-side signals for copied block times, re-tuned fallback
blends, bootstrap bagging of the fallback, month- or day-level drift corrections, more ensemble
seeds, Rome-specific main-lane corrections, and the transfer of Rome's copy logic to the other nine
airports (outside Rome rows with long delays are normal rows; scripts and tables in
`research_800_final/`).

## External data

- OpenStreetMap airport geometry in `geometry/` (ODbL, `geometry/LICENSE.md`).
- adsb.lol daily ADS-B archives (ODbL 1.0, github.com/adsblol/globe_history_YYYY), fetched and
  reduced to airport tracks by `adsb-lol/fetch_days.py`, `fetch_months.py`, `extract_tracks.py`.
- Stand positions from the X-Plane Scenery Gateway (approved sceneries of EDDM, EHAM and EDDF;
  scenery ids in `research_300/geometry/xplane_*.json`). X-Plane's developer terms publish the
  default airport data under the GNU GPL; the exact version for Gateway submissions was not
  confirmed.
- METAR reports from the Iowa Environmental Mesonet (public domain), for weather research only.

## Repository layout and language

    main_model.py, neural_model.py, submission.py, features_*.py   v44 recipe and shared modules
    research_250/ research_260/ research_240/ research_280/        base: schema, sequence nets, fallback, copy gate
    research_500_midnight/ research_510_rome_classifier/          Rome rules
    research_800_final/   four-month base, final assembly, February/June checks
    research_290/ ... research_720/                               ADS-B layers and screens, with results
    RESEARCH_INDEX.md     one line per research folder
    research_700_probe/   leaderboard experiments of 2 October: policy, scripts, ledger
    adsb-lol/             track download and extraction tools
    results/              official score responses and release manifests
    geometry/             OpenStreetMap geometry
    tests/                lightweight tests that need no data
    data/                 place the competition files here (not included)

The older modules use German identifiers and comments; a glossary is at the end of `SETUP.md`. The
research folders are in English.

## License and acknowledgements

GPLv3, see `LICENSE`. Thanks to the EUROCONTROL Performance Review Commission and the OpenSky Network
for running the challenge and providing the data, to the OpenStreetMap contributors, the adsb.lol
volunteers, the X-Plane Scenery Gateway contributors and the Iowa Environmental Mesonet.
