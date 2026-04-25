# NYC Cruising Observatory

An open observatory for measuring the **cost of cruising for parking** in New York City.

## What this is

A reproducible, auditable study built on top of the [Ekstra Observation API](https://ekstra-platform-api-production.up.railway.app/api/v1/observations/coverage). Public DOT cameras → vehicle tracks → cruising signal → space-supply estimates → cost per cruising hour (driver time, emissions, congestion externalities).

This repo holds the **methodology, code, and aggregated data** — not raw video, not personally identifiable information, not vehicle plates. The Ekstra runtime never persists raw frames; only signed Motion Packets and coarse, non-reversible appearance signatures (8-bit color + 3-bit size + 2-bit dwell) leave the device.

## Status

**Day 0 — skeleton committed.** Methodology placeholder, repo structure, licenses. No analysis yet.

The 24h coverage gate (`≥10 cameras with sane tier breakdown`, see `methodology.md`) has not yet passed. No API calls happen from this repo until Phase 2 of the Observation API ships the public read endpoint. Preregistration is intentionally deferred until the Week 1 coverage audit produces real numbers.

## Repo layout

```
nyc-cruising-observatory/
├── README.md           — this file
├── methodology.md      — study design, hypotheses, gates, limitations (placeholder)
├── LICENSE             — MIT (code)
├── LICENSE-DATA        — CC-BY-4.0 (aggregated data products)
├── CITATION.cff        — citation metadata (stub)
├── data/               — aggregated, anonymized data products only
│   └── .gitkeep
└── code/               — analysis scripts (Python, R, SQL)
    └── .gitkeep
```

## Data sources (planned)

- **Ekstra Observation API** — vehicle tracks per camera, 72h rolling window. License: see Ekstra terms.
- **NYC DOT public cameras** — ~9,700 traffic cameras, public feeds.
- **NYC DOT muni-meter occupancy** (where published) — ground truth for paid blockfaces.
- **NYC DCP PLUTO** — parcel-level land use.
- **OpenStreetMap** — street network + curb geometry where available.

## Reproducibility

Every result will ship with:
- the exact Observation API query that produced it (camera set, time window, signature version);
- the SQL or Python that aggregated it;
- the calibration tier of every contributing camera at the time of measurement;
- a SHA of the `code/` directory at analysis time.

## Boundary with Ekstra

This study is built **on top of** the Ekstra platform; it is not part of it. Ekstra owns the runtime and the API; this repo owns the analysis. If a question can be answered by reading the Ekstra docs, it does not belong here.

## License

- Code: MIT (see `LICENSE`)
- Aggregated data products in `data/`: CC-BY-4.0 (see `LICENSE-DATA`)
