# Installed-Base Propensity Model — Agilent Interview Prep Note

This note backs slide 2, point 3 of `Agilent_Interview_Prework.pptx`
("Over half of Agilent's revenue — consumables and service — tracks
installed-base utilization, not new instrument bookings"). It explains where
the challenge comes from, what a propensity model is, and exactly how I would
apply one to it if I'm in this role.

## 1. Why this is a challenge worth naming

Agilent's own numbers say consumables and service contracts together are
**more than 50% of total revenue**, and that revenue is explicitly tied to
**installed-base utilization** — how much customers who already own an
instrument keep running it — not to new instrument bookings. Tiered service
contracts (Gold/Platinum) renew on their own schedule, and consumables
(columns, sample-prep kits, reagents) get reordered based on lab usage, not
sales-cycle timing.

A demand-planning function that only forecasts "what will we book next" is
blind to this. New-instrument bookings come from a CRM/pipeline forecast;
consumables and service renewals come from a completely different process —
what customers who already bought from us are doing with what they own. If
the two aren't modeled separately, an accuracy miss can't be root-caused
("was it a sales pipeline problem, or are existing customers running their
instruments less?"), and IBP deviation reviews end up guessing.

## 2. What a propensity model is (in one paragraph)

A propensity model estimates the **probability that a specific entity will
take a specific action in a specific window** — here, "will this installed
instrument generate a consumables order (or renew its service contract) in
the next period" — using that entity's own history and profile as features.
It's the same family of model as a churn model or a lead-scoring model; the
label is just binary (did the event happen, yes/no) rather than a continuous
forecast number. The reason it's the right tool here, and a plain time-series
forecast isn't: **consumables demand at the individual-instrument level is
intermittent** — most instrument-months have zero orders — and a regression
trained on mostly-zero data just predicts something close to the average
everywhere, which is wrong everywhere. Propensity (will it happen at all)
has to be modeled separately from magnitude (how much, if it happens).

## 3. How I would apply it here, step by step

### 3.1 Pick the right unit of analysis

The model has to run at **instrument (install) × customer/site × month** —
not customer-level, not product-family-level. An instrument is the thing
that actually consumes reagents and columns; two instruments at the same
customer can have completely different usage. Forecasting at a coarser grain
throws away exactly the signal the model needs.

### 3.2 Split into two sub-problems, each with its own model shape

**A. Consumables demand — a hurdle (two-part) model**, because the data is
zero-inflated:

- **Stage 1 — propensity (classification):** will this install place a
  consumables order this month? Label = binary. Features:
  - *RFM*: months since last order, historical order frequency, cumulative
    spend to date
  - *Instrument profile*: model/family, months since install (usage often
    ramps up after validation/method development, then plateaus), direct
    vs. distributor-sold
  - *Customer segment*: pharma QC lab (near-continuous running) vs.
    academic (semester/grant-cycle driven) vs. CRO/CDMO
  - *Compatibility*: which consumable SKUs are even usable on this
    instrument model (a bill-of-materials join that constrains Stage 2)
  - Model: gradient-boosted trees (LightGBM) — pools across all installs so
    low-volume instrument types borrow signal from high-volume ones
- **Stage 2 — amount (regression, fit only on the positive cases):** given
  they order, how much? Same feature set, target = order value/volume.
- **Combine:** expected demand for that instrument-month =
  `P(order) × E[amount | order]`. Sum across every install in a
  product-family × region × month cell to get an **install-base-driven
  consumables baseline** — the piece of total demand that's independent of
  new bookings.

**B. Service-contract renewal — survival framing, not plain classification**,
because a contract that hasn't reached its renewal date yet is *censored*,
not a negative label:

- Either a proper survival model (Cox proportional hazards) or a logistic
  model evaluated only at each contract's actual renewal checkpoint.
- Features: contract tier, tenure, recent service-ticket volume, instrument
  utilization trend, plus the same kind of engagement/health signal I used
  building **Strikedeck at Verisk** — a model that predicted customer health
  and fired automated at-risk alerts. This is the same architecture pointed
  at contract renewal instead of relationship health: a risk score per
  contract, an alert ahead of the renewal date, surfaced to the CrossLab
  commercial/service team so they can intervene before it lapses.

### 3.3 Handle cold start explicitly

A newly installed instrument has no purchase history to condition on. I'd
blend an individual estimate with a **cohort-level prior** (the average
ramp-up curve for that instrument family/segment) and let individual signal
take over as real history accumulates — the same technique I used for the
`alpha` curve-shape parameter in my response-curve budget-optimization
project: fit what you can from data, fall back to a structural prior when
data is thin, and blend rather than switching abruptly.

### 3.4 Validate the way that actually matches production

Walk-forward, not a random train/test split: train on everything known as of
month T, predict month T+1, compare, roll forward. A random split would leak
future purchase behavior into training and make the model look better than
it would ever be live — this is the same validation discipline behind the
WMAPE/Tracking-Signal numbers in my forecasting-ensemble project.

### 3.5 Roll it into the bigger forecast, and use the split itself

`Total demand = new-bookings forecast (CRM/pipeline-driven) + install-base
consumables/service forecast (this model)`. Keeping them separate isn't just
more accurate — it's more actionable: when the total forecast misses, this
split immediately tells you whether it was a bookings problem (sales
pipeline) or a utilization problem (existing customers running their
instruments less than expected, which might itself be an early churn
signal) — the same root-cause-first instinct behind the bias-correction
layer in my forecasting project.

### 3.6 Production shape

Needs an ETL pipeline joining install-base, order-history, and
service-contract tables — a Fabric-style pipeline, the same kind I built for
the Publication Intelligence project — refreshed on a schedule as new
installs, orders, and service events land, with two outputs: the aggregated
consumables/service baseline feeding the IBP number, and an account-level
at-risk list surfaced via Power BI to commercial/service teams.

## 4. Honest caveat

I don't have access to Agilent's actual internal data schema — this is a
proposed approach built from what's publicly confirmed about their business
model (SAP ECC/S4HANA + SAP IBP, direct + distributor channel mix, >50%
revenue from consumables/service tied to installed-base utilization) plus
standard practice for this class of problem. The right opening move in the
role would be confirming what install-base, order-history, and service
telemetry actually exists and at what grain, before assuming this exact
feature set is available.
