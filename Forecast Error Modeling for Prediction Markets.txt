p = 0.253
edge = p - 0.25
print(edge)

# WEEK 2 CORE: IMPLIED PROBABILITY & EDGE

def implied_probability(price):
    return price

def edge(model_prob, market_prob):
    return model_prob - market_prob


# Example market
market_price = 0.25      # 25¢ contract
market_prob = implied_probability(market_price)

# Your estimate
model_prob = 0.253

e = edge(model_prob, market_prob)

print("Market probability:", market_prob)
print("Model probability:", model_prob)
print("Edge:", round(e, 4))

import math

# --- Week 3: Error model -> probability of exceeding a threshold ---

# Placeholder forecast errors (°F). Replace later with real data.
errors = [-2, -1, 0, 1, 2, -1, 1, 0, -2, 2]

def mean(xs):
    return sum(xs) / len(xs)

def std_sample(xs):
    """Sample standard deviation (n-1). Better default for small samples."""
    m = mean(xs)
    var = sum((x - m) ** 2 for x in xs) / (len(xs) - 1)
    return math.sqrt(var)

def norm_cdf(z):
    """Standard normal CDF using error function (no extra libraries)."""
    return 0.5 * (1.0 + math.erf(z / math.sqrt(2.0)))

def prob_temp_ge_threshold(forecast_temp, threshold_temp, mu_error, sigma_error):
    """
    TrueTemp = Forecast + Error, Error ~ Normal(mu_error, sigma_error)
    P(TrueTemp >= threshold) = 1 - Phi((threshold - forecast - mu)/sigma)
    """
    if sigma_error <= 0:
        raise ValueError("sigma_error must be > 0")
    z = (threshold_temp - forecast_temp - mu_error) / sigma_error
    return 1.0 - norm_cdf(z)

# --- Fit toy error model ---
mu = mean(errors)
sigma = std_sample(errors)

print("Toy error model")
print("Mean error (mu):", round(mu, 3))
print("Std dev (sigma):", round(sigma, 3))

# --- Example: Dallas-style market ---
forecast = 97.0
threshold = 100.0

# --- Toy-only example (uses fitted toy sigma; NOT for trading) ---
p_toy = prob_temp_ge_threshold(forecast, threshold, mu, sigma)

print("\nToy-only probability (fitted toy sigma — diagnostic only)")
print(f"Forecast: {forecast}F, Threshold: {threshold}F")
print("P(Temp >= threshold) [toy sigma]:", round(p_toy, 4))

# Market price (used later for sigma inference)
market_price = 0.25
print("\nMarket implied prob:", market_price)

# --- Reality check: try more realistic sigmas (°F) ---
for test_sigma in [2.0, 3.0, 4.0, 5.0]:
    p_test = prob_temp_ge_threshold(forecast, threshold, mu, test_sigma)
    print(f"Sigma={test_sigma:.1f} -> P(>= {threshold}) = {p_test:.4f}, edge vs 0.25 = {p_test-0.25:.4f}")

def inv_norm_cdf(p):
    # Approximation good enough for this project phase
    # Peter John Acklam's approximation (compressed)
    if p <= 0 or p >= 1:
        raise ValueError("p must be between 0 and 1")

    a = [-3.969683028665376e+01,  2.209460984245205e+02,
         -2.759285104469687e+02,  1.383577518672690e+02,
         -3.066479806614716e+01,  2.506628277459239e+00]

    b = [-5.447609879822406e+01,  1.615858368580409e+02,
         -1.556989798598866e+02,  6.680131188771972e+01,
         -1.328068155288572e+01]

    c = [-7.784894002430293e-03, -3.223964580411365e-01,
         -2.400758277161838e+00, -2.549732539343734e+00,
          4.374664141464968e+00,  2.938163982698783e+00]

    d = [ 7.784695709041462e-03,  3.224671290700398e-01,
          2.445134137142996e+00,  3.754408661907416e+00]

    plow = 0.02425
    phigh = 1 - plow

    if p < plow:
        q = math.sqrt(-2*math.log(p))
        return (((((c[0]*q + c[1])*q + c[2])*q + c[3])*q + c[4])*q + c[5]) / \
               ((((d[0]*q + d[1])*q + d[2])*q + d[3])*q + 1)
    if p > phigh:
        q = math.sqrt(-2*math.log(1-p))
        return -(((((c[0]*q + c[1])*q + c[2])*q + c[3])*q + c[4])*q + c[5]) / \
                 ((((d[0]*q + d[1])*q + d[2])*q + d[3])*q + 1)

    q = p - 0.5
    r = q*q
    return (((((a[0]*r + a[1])*r + a[2])*r + a[3])*r + a[4])*r + a[5])*q / \
           (((((b[0]*r + b[1])*r + b[2])*r + b[3])*r + b[4])*r + 1)

def implied_sigma_from_market(market_prob, forecast_temp, threshold_temp, mu_error):
    # market_prob = P(TrueTemp >= threshold)
    # Phi(z) = 1 - market_prob  => z = inv_norm_cdf(1 - market_prob)
    z = inv_norm_cdf(1 - market_prob)
    delta = threshold_temp - forecast_temp - mu_error
    return delta / z

sigma_implied = implied_sigma_from_market(market_price, forecast, threshold, mu)
print("\nMarket-implied sigma")
print("Implied sigma:", round(sigma_implied, 3))
# --- Lead-time buckets (toy) ---
print("\nLead-time bucket pricing (toy sigmas)")
lead_time_sigmas = {
    "D-7": 5.0,
    "D-3": 3.5,
    "D-1": 2.5,
}

for label, s in lead_time_sigmas.items():
    p_lt = prob_temp_ge_threshold(forecast, threshold, mu, s)
    print(f"{label}: sigma={s:.2f} -> P(>= {threshold})={p_lt:.4f} -> fair price={p_lt:.4f}")
print("\nSigma edge (core signal)")
assumed_sigma = 3.5  # change this when you have real error data
p_assumed = prob_temp_ge_threshold(forecast, threshold, mu, assumed_sigma)

print("Assumed sigma:", assumed_sigma)
print("Model prob (assumed sigma):", round(p_assumed, 4))
print("Market prob:", market_price)
print("Edge:", round(p_assumed - market_price, 4))

if p_assumed > market_price:
    print("Signal: BUY YES (model > market)")
else:
    print("Signal: BUY NO / SELL YES (model < market)")

from __future__ import annotations

import csv
import json
import os
from datetime import date
from pathlib import Path
from urllib.parse import urlencode
from urllib.request import Request, urlopen


# ---- CONFIG ----
STATION_ID = "GHCND:USW00013960"  # Dallas Love Field truth station (KDAL)
DATASET_ID = "GHCND"
DATATYPE_ID = "TMAX"
UNITS = "standard"
LOCATION = "Dallas (Love Field)"
# Date range to pull (YYYY-MM-DD)
START_DATE = "2025-01-01"
END_DATE = date.today().isoformat()


# Output CSV
PROJECT_ROOT = Path(__file__).resolve().parent
OBS_CSV = PROJECT_ROOT / "data" / "raw" / "observations_daily_kdal.csv"
BASE_URL = "https://www.ncei.noaa.gov/cdo-web/api/v2/data"


def get_token() -> str:
    token = os.getenv("NOAA_TOKEN", "").strip()
    if not token:
        raise RuntimeError(
            "NOAA_TOKEN env var not set. Run:\n"
            '  setx NOAA_TOKEN "YOUR_TOKEN"\n'
            "then reopen Command Prompt."
        )
    return token


def fetch_all_tmax(token: str, start_date: str, end_date: str) -> list[dict]:
    """
    Fetches all results using pagination (limit/offset) from NOAA CDO v2.
    Returns raw 'results' list items.
    Docs: token header + /data endpoint usage :contentReference[oaicite:3]{index=3}
    """
    results: list[dict] = []
    limit = 1000
    offset = 1  # NOAA uses 1-indexed offsets in examples/metadata

    while True:
        params = {
            "datasetid": DATASET_ID,
            "datatypeid": DATATYPE_ID,
            "stationid": STATION_ID,
            "startdate": start_date,
            "enddate": end_date,
            "units": UNITS,
            "limit": limit,
            "offset": offset,
        }
        url = f"{BASE_URL}?{urlencode(params)}"
        req = Request(url, headers={"token": token})
        with urlopen(req, timeout=30) as resp:
            payload = json.loads(resp.read().decode("utf-8"))

        batch = payload.get("results", [])
        results.extend(batch)

        meta = payload.get("metadata", {}).get("resultset", {})
        count = int(meta.get("count", len(results)))
        returned = int(meta.get("limit", limit))
        current_offset = int(meta.get("offset", offset))

        # Stop if we've pulled everything
        if current_offset + returned > count:
            break

        offset = current_offset + returned

    return results


def ensure_obs_header_exists() -> None:
    OBS_CSV.parent.mkdir(parents=True, exist_ok=True)
    if OBS_CSV.exists() and OBS_CSV.stat().st_size > 0:
        return
    with OBS_CSV.open("w", newline="", encoding="utf-8") as f:
        csv.writer(f).writerow(["date", "tmax_f", "station_id", "location"])


#def append_observations(rows: list[dict]) -> int:
    """
    Append observations, but dedupe by date (keep last seen).
    """
    ensure_obs_header_exists()

    # Load existing dates so we don't duplicate
    existing = {}
    if OBS_CSV.exists() and OBS_CSV.stat().st_size > 0:
        with OBS_CSV.open("r", encoding="utf-8", newline="") as f:
            r = csv.DictReader(f)
            for row in r:
                d = row.get("date", "").strip()
                if not d:
                    continue
                try:
                    t = float(row.get("tmax_f", ""))
                except ValueError:
                    continue
                existing[d] = t

    # Merge new rows (overwrite same date)
    added_or_updated = 0
    for r in rows:
        dt = r.get("date", "")[:10]
        val = r.get("value", None)
        if dt and val is not None:
            t = float(val)
            if dt not in existing or existing[dt] != t:
                existing[dt] = t
                added_or_updated += 1

    # Rewrite entire file (clean + deduped)
    with OBS_CSV.open("w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["date", "tmax_f", "station_id", "location"])
        for d in sorted(existing.keys()):
            w.writerow([d, existing[d], STATION_ID, LOCATION])

    return added_or_updated
def append_observations(rows: list[dict]) -> int:
    """
    Append observations, but dedupe by date (keep last seen).
    """
    ensure_obs_header_exists()

    # Load existing dates so we don't duplicate
    existing = {}
    if OBS_CSV.exists() and OBS_CSV.stat().st_size > 0:
        with OBS_CSV.open("r", encoding="utf-8", newline="") as f:
            r = csv.DictReader(f)
            for row in r:
                d = row.get("date", "").strip()
                if not d:
                    continue
                try:
                    t = float(row.get("tmax_f", ""))
                except ValueError:
                    continue
                existing[d] = t

    # Merge new rows (overwrite same date)
    added_or_updated = 0
    for r in rows:
        dt = r.get("date", "")[:10]
        val = r.get("value", None)
        if dt and val is not None:
            t = float(val)
            if dt not in existing or existing[dt] != t:
                existing[dt] = t
                added_or_updated += 1

    # Rewrite entire file (clean + deduped)
    with OBS_CSV.open("w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["date", "tmax_f", "station_id", "location"])
        for d in sorted(existing.keys()):
            w.writerow([d, existing[d], STATION_ID, LOCATION])

    return added_or_updated



def main() -> None:
    token = get_token()
    raw = fetch_all_tmax(token, START_DATE, END_DATE)

    if not raw:
        print("No results returned. Check dates, token, and station ID.")
        return

    n = append_observations(raw)
    print("OK — pulled observations")
    print("Station:", STATION_ID)
    print("Range :", START_DATE, "to", END_DATE)
    print("Rows appended:", n)
    print("Output file:", OBS_CSV)

    # Quick sanity hint
    print("\nSanity check:")
    print("- Open the CSV and confirm TMAX looks like Fahrenheit daily highs (e.g., ~30–110 in summer).")


if __name__ == "__main__":
    main()

from __future__ import annotations

import csv
import json
from datetime import datetime, timezone, date
from pathlib import Path
from urllib.request import Request, urlopen

# --- CONFIG ---
LOCATION = "Dallas Love Field (KDAL)"
LAT = 32.8471
LON = -96.8517
SOURCE = "NWS"

PROJECT_ROOT = Path(__file__).resolve().parent
FORECAST_CSV = PROJECT_ROOT / "data" / "raw" / "forecast_snapshots.csv"

USER_AGENT = "polymarket-weather-quant (youce; contact: you@example.com)"  # change email if you want

def ensure_header() -> None:
    FORECAST_CSV.parent.mkdir(parents=True, exist_ok=True)
    if FORECAST_CSV.exists() and FORECAST_CSV.stat().st_size > 0:
        return
    with FORECAST_CSV.open("w", newline="", encoding="utf-8") as f:
        csv.writer(f).writerow([
            "snapshot_utc",
            "target_date",
            "forecast_high_f",
            "lead_days",
            "source",
            "location",
            "lat",
            "lon",
        ])

def nws_points_url(lat: float, lon: float) -> str:
    return f"https://api.weather.gov/points/{lat:.4f},{lon:.4f}"

def fetch_json(url: str) -> dict:
    req = Request(url, headers={"User-Agent": USER_AGENT, "Accept": "application/geo+json"})
    with urlopen(req, timeout=30) as resp:
        return json.loads(resp.read().decode("utf-8"))

def snapshot_utc_now() -> str:
    return datetime.now(timezone.utc).isoformat(timespec="seconds")

def parse_periods_for_daily_high(periods: list[dict]) -> dict[date, int]:
    """
    NWS returns periods that are typically "Today", "Tonight", "Monday", "Monday Night", etc.
    We take all daytime periods (isDaytime=True) and use their temperature as the daily high.
    Returns {target_date: forecast_high_f}
    """
    daily = {}
    for p in periods:
        if p.get("isDaytime") is True and isinstance(p.get("temperature"), (int, float)):
            start = p.get("startTime", "")
            if len(start) >= 10:
                d = date.fromisoformat(start[:10])
                daily[d] = int(p["temperature"])
    return daily

def main() -> None:
    ensure_header()

    pts = fetch_json(nws_points_url(LAT, LON))
    forecast_url = pts["properties"]["forecast"]  # 7-day forecast
    fc = fetch_json(forecast_url)

    periods = fc["properties"]["periods"]
    daily_highs = parse_periods_for_daily_high(periods)

    snap = snapshot_utc_now()
    snap_date = date.fromisoformat(snap[:10])

    rows = []
    for target_d, high_f in sorted(daily_highs.items()):
        lead_days = (target_d - snap_date).days
        if lead_days < 0:
            continue
        rows.append([
            snap,
            target_d.isoformat(),
            float(high_f),
            int(lead_days),
            SOURCE,
            LOCATION,
            LAT,
            LON,
        ])

    with FORECAST_CSV.open("a", newline="", encoding="utf-8") as f:
        csv.writer(f).writerows(rows)

    print("OK — logged NWS forecast snapshot")
    print("Snapshot UTC:", snap)
    print("Rows appended:", len(rows))
    print("Output file:", FORECAST_CSV)
    print("First few targets:", [r[1] for r in rows[:3]])

if __name__ == "__main__":
    main()

from __future__ import annotations

import csv
import math
from collections import defaultdict
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent

FORECAST_CSV = PROJECT_ROOT / "data" / "raw" / "forecast_snapshots.csv"
OBS_CSV = PROJECT_ROOT / "data" / "raw" / "observations_daily_kdal.csv"
OUT_CSV = PROJECT_ROOT / "data" / "processed" / "errors_by_lead.csv"


def read_observations(path: Path) -> dict[str, float]:
    """Return {date: tmax_f}"""
    obs = {}
    with path.open("r", encoding="utf-8", newline="") as f:
        r = csv.DictReader(f)
        for row in r:
            d = row["date"].strip()
            try:
                t = float(row["tmax_f"])
            except ValueError:
                continue
            obs[d] = t
    return obs


def mean(xs: list[float]) -> float:
    return sum(xs) / len(xs)


def std_sample(xs: list[float]) -> float:
    if len(xs) < 2:
        return float("nan")
    m = mean(xs)
    var = sum((x - m) ** 2 for x in xs) / (len(xs) - 1)
    return math.sqrt(var)


def main() -> None:
    if not FORECAST_CSV.exists():
        raise FileNotFoundError(f"Missing forecasts file: {FORECAST_CSV}")
    if not OBS_CSV.exists():
        raise FileNotFoundError(f"Missing obs file: {OBS_CSV}")

    obs = read_observations(OBS_CSV)

    OUT_CSV.parent.mkdir(parents=True, exist_ok=True)

    errors_by_lead: dict[int, list[float]] = defaultdict(list)

    written = 0
    missing_obs = 0

    with FORECAST_CSV.open("r", encoding="utf-8", newline="") as f_in, \
         OUT_CSV.open("w", encoding="utf-8", newline="") as f_out:

        r = csv.DictReader(f_in)
        w = csv.writer(f_out)
        w.writerow([
            "snapshot_utc",
            "target_date",
            "lead_days",
            "forecast_high_f",
            "observed_tmax_f",
            "error_obs_minus_fcst",
            "source",
            "location",
            "lat",
            "lon",
        ])

        for row in r:
            target_date = row["target_date"].strip()
            lead_days = int(float(row["lead_days"]))
            fcst = float(row["forecast_high_f"])

            if target_date not in obs:
                missing_obs += 1
                continue

            observed = obs[target_date]
            err = observed - fcst

            w.writerow([
                row["snapshot_utc"],
                target_date,
                lead_days,
                fcst,
                observed,
                err,
                row.get("source", ""),
                row.get("location", ""),
                row.get("lat", ""),
                row.get("lon", ""),
            ])

            errors_by_lead[lead_days].append(err)
            written += 1

    print("OK — computed errors")
    print("Forecast rows matched to observations:", written)
    print("Forecast rows missing observations (future dates):", missing_obs)
    print("Output:", OUT_CSV)

    # Summary by lead time
    print("\nError summary by lead_days (obs - forecast)")
    for lead in sorted(errors_by_lead.keys()):
        xs = errors_by_lead[lead]
        mu = mean(xs)
        sd = std_sample(xs)
        print(f"Lead {lead:2d}d: n={len(xs):3d}  mean={mu:+.3f}F  sd={sd:.3f}F")


if __name__ == "__main__":
    main()
