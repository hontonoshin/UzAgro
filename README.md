# Canopy monitoring service

Weekly Sentinel-2 canopy monitoring for agricultural clusters. Every parcel a
client declares is measured — there is no sampling. Each parcel is compared to
the client's other parcels of the same crop this week, and to its own history.

## Scope — read this first

**Measured:** canopy vigour (NDVI), red-edge vigour (NDRE), canopy water content (NDMI).

**Not measured, and never to be claimed:** soil organic carbon, NPK, pH, texture,
salinity, soil moisture, irrigation requirement, yield, disease, pest, legal status.

Sentinel-2 cannot retrieve soil properties under a canopy. NDMI describes water in
the *leaves*, not in the ground. Every output says so, and `config.SCOPE_STATEMENT`
is printed in every report. If a client asks for soil quality, the answer is that
it requires soil sampling — which is a different product with a different cost.

Reports flag **where to look**, never **what is wrong**.

## Install

```bash
python -m venv .venv && source .venv/bin/activate
pip install openeo pandas streamlit altair pyproj shapely
```

Headless auth (required for cron). Create an OAuth client in the Copernicus Data
Space dashboard, then:

```bash
export CDSE_CLIENT_ID=...
export CDSE_CLIENT_SECRET=...
```

Without these, openEO falls back to an interactive browser login.

## Onboard a client

```
clients/<client>/
  client.json      label, contact, language (uz|ru|en), monitoring_start
  fields.geojson   the client's parcels
```

Every feature needs `field_id` (unique int), `name`, and `seasons`:

```json
{
  "type": "Feature",
  "properties": {
    "field_id": 12,
    "name": "Paxta-12",
    "seasons": {"2025": "cotton", "2026": "cotton"}
  },
  "geometry": {"type": "Polygon", "coordinates": [[[60.6, 41.5], ...]]}
}
```

Crops: `cotton`, `wheat`, `rice`, `orchard`, `other` (see `config.CROP_CALENDARS`).

Validate before spending any quota:

```bash
python scripts/01_setup_check.py --client <client>
```

This catches duplicate ids, unknown crops, sub-hectare parcels, missing seasons.

## Run order

```bash
# 1. validate backend + client definition
python scripts/01_setup_check.py --client acme

# 2. backfill history once (slow — this is the expensive run)
python scripts/02_fetch_indices.py --client acme --start 2024-01-01

# 3-4. detect and report
python scripts/03_anomalies.py --client acme
python scripts/04_report.py --client acme --language uz

# 5. dashboard
streamlit run app.py
```

Then all of it weekly, in one command:

```bash
python scripts/05_run_weekly.py --client acme --headless
python scripts/05_run_weekly.py --all --headless        # every client
```

Cron:

```cron
0 4 * * 1 cd /srv/agri && .venv/bin/python scripts/05_run_weekly.py --all --headless >> logs/cron.log 2>&1
```

`05_run_weekly.py` skips clients with no new imagery since their last successful
run (openEO quota is the binding constraint), and exits non-zero if any client
fails so the scheduler notices.

## Test without spending quota

```bash
python scripts/99_make_demo_client.py --client demo   # synthetic, 4 planted anomalies
python scripts/03_anomalies.py --client demo
python scripts/04_report.py --client demo --language en
streamlit run app.py
```

The detector should find exactly fields 1, 2, 3, 4. If it finds more, a rule has
started producing false positives.

## Why weekly, not daily

Sentinel-2 revisit is 5 days; clouds remove some of those. In practice a parcel is
observed 2–4 times a month. A daily report would re-send the same observation six
times and imply a precision the data does not have. `03_anomalies.py` therefore
works on a **window** (default 10 days), and reports how many parcels had no clear
observation rather than silently carrying forward stale values.

## Design notes

**No sampling.** `aggregate_spatial` computes zonal statistics on the backend, so
measuring 2,000 parcels costs a small table, not 2,000 rasters.

**Incremental.** Each run fetches only dates after the last stored observation.
Steady-state cost is one small job per chunk per week.

**Two reference frames.** Cohort rules survive region-wide weather (the reference
moves with it) but need ≥ `MIN_COHORT` parcels of a crop. Own-history rules work
for a single parcel but need the crop calendar to distinguish collapse from normal
senescence. Both are reported; neither is trusted alone.

**Robust outliers, not percentiles.** A percentile gate flags a fixed share of the
cohort by construction — with 20 parcels the bottom decile is two, so a third
genuinely failing parcel is invisible. Rules use a median/MAD z-score, which fires
on however many parcels actually deviate and on none when the cohort is healthy.

**Nothing runs outside the growing window.** Before emergence a field is legitimately
bare; after senescence it is harvested. Both look like collapse to every rule, so
`evaluate()` returns early outside `emergence[0] .. senescence[1]`.

## Known limits

- Parcels below ~0.5 ha are unreliable at 20 m; `01_setup_check.py` warns.
- Crop calendars are regional defaults, not client-specific. Override per client.
- Mixed-crop parcels break the cohort comparison; split them.
- Persistent cloud can leave a parcel unobserved for weeks. This is reported, not hidden.
- No within-field variability yet — that needs rasters, not zonal means, and is the
  natural next module.
