# Methodology

> **Status: placeholder.** This file will hold the full preregistered study design once the Week 1 coverage audit produces real numbers. Do not cite from this version.

## Research question

What is the cost of cruising for parking in New York City — measured in driver-hours, vehicle-miles, and excess emissions per occupied curb-space-hour, aggregated by neighborhood and time-of-day?

## Why this question, why now

Existing cruising estimates (Shoup 2006; Hampshire & Shoup 2018) rely on small block-level samples extrapolated city-wide. Ekstra's network of public DOT cameras lets us measure the same phenomenon **continuously, citywide, with calibration-tier-aware confidence**.

## Hypotheses (to be preregistered after Week 1 audit)

- H1: Cruising volume scales with paid-meter occupancy on adjacent blocks.
- H2: Cruising volume scales inversely with adjacent off-street garage availability.
- H3: Cruising volume is bimodal across the day (commute peaks).
- H4: Cruising cost per occupied curb-space-hour exceeds the meter rate in at least N neighborhoods.

(Placeholders — will be sharpened or dropped after the coverage audit.)

## Data pipeline

1. **Tracks** from Ekstra Observation API (`GET /api/v1/observations/{camera_motion_address}/tracks`, Phase 2).
2. **Camera metadata** — Motion Address, lat/lon, calibration tier (uncalibrated/approximate/calibrated), facing.
3. **Cruising signal** = a vehicle that returns past the same camera ≥2× within 8 minutes without parking, OR a vehicle whose dwell signature is "passing" in a corridor where ≥X% of nearby curb is occupied.
4. **Aggregation** — by camera, by hour, by neighborhood, by tier. Calibrated cameras get full weight; approximate get reduced weight; uncalibrated are excluded from cost estimates but kept for cruising-rate-only summaries.

## Coverage gate (must pass before any analysis is published)

Before any analysis is run on real data, the Ekstra coverage report must satisfy:

- **≥10 cameras** with `tracks_written > 0` over a 24h window;
- **sane tier breakdown** — at least one camera in each of `approximate` and `calibrated` (uncalibrated alone is not sufficient for cost estimates);
- **track_floor_ratio** in the **30–60% band** for the median camera (catches systematic bias from `MIN_FRAMES_TO_PUBLISH` floor).

Source of truth: `GET /api/v1/observations/coverage?hours=24`.

## Limitations and known biases

- **DOT camera coverage is not random.** Cameras concentrate at intersections and on arterials; quiet residential streets are under-observed. We will not extrapolate beyond the observed support.
- **Calibration tier varies.** Uncalibrated cameras can count vehicles but cannot place them in world space; their contribution is restricted to rate-only summaries.
- **Cruising vs. circulation.** A vehicle passing a camera twice in 8 minutes may be cruising or may be running an errand. We will publish both a strict and a permissive definition.
- **No plate-level identification.** The Ekstra appearance signature is intentionally coarse (13 bits total, non-reversible). Re-identification across non-adjacent cameras is unreliable by design — and we treat that as a feature, not a bug.

## Privacy and ethics

This study uses **only** the Ekstra Observation API. No raw video. No vehicle plates. No facial data. No PII.

The appearance signature (Option A, locked at signature_version `s_v1`) emits 8 bits of color + 3 bits of size + 2 bits of dwell — collectively 13 bits of soft identity, which collapses to ~8,000 distinct buckets. This is enough to estimate cruising at the camera-pair level under heavy aggregation; it is far too coarse to identify a vehicle.

The aggregated data products in `data/` will only be published at temporal/spatial granularity sufficient to prevent re-identification.

## Reproducibility

Every figure and table will ship with:

1. The exact API query (camera set, time window, signature version, calibration-tier filter).
2. The SQL or Python that produced the result.
3. A SHA of the `code/` tree at analysis time.
4. A list of cameras included and their calibration tier at the time of measurement.

## Open questions (resolve after Week 1 audit)

- Power analysis: how many camera-hours do we need for the cost estimate to converge to ±10%?
- Should "cruising" be defined per-vehicle (track-level) or per-corridor (camera-level)?
- Do we include rideshare cruising (waiting for a ping) in the same measurement, or separate it?
- What is the publication cadence — quarterly snapshots, or live dashboard?
