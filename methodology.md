# Methodology: NYC Cruising Observatory

**Version:** 1.0
**Date:** 2026-04-27
**Study site:** New York City
**Instrument:** Ekstra Observation API (rev 2, 2026-04-24)
**Companion documents:** `EKSTRA_OBSERVATION_API.md` (instrument specification); `scratch/wmin_reconciliation_memo.md` (signal-window decision record); `scratch/save_v1_validation_review.py` (manual labelling protocol).

---

## 1. Research question and motivation

### 1.1 Research question

What fraction of curb-front vehicle traffic in New York City is composed of vehicles cruising for an on-street parking space, and how does that fraction vary by neighbourhood, time of day, day of week, and adjacent off-street garage availability?

The study estimates a **cruising rate** (the fraction of camera passes that recur within a defined time window with a matching coarse appearance signature) at three nested levels of aggregation: camera-hour, corridor-day, and neighbourhood-week. From the cruising rate, in conjunction with NYC DOT vehicle counts and local meter occupancy data, we derive corollary quantities (driver-hours of excess travel, vehicle-miles of excess travel, fuel-equivalent excess emissions) at neighbourhood-week granularity.

### 1.2 Motivation

Three published cruising studies anchor the field:

1. Shoup (1995, 2006) used direct block-walk observation in 15 international study sites. The widely cited figure (that cruising accounts for roughly 30% of urban downtown traffic) derives from this small-sample, hand-counted method.
2. Hampshire & Shoup (2018) re-estimated cruising in central Stuttgart using TomTom connected-vehicle GPS traces, classifying trips as cruising when the matched path materially exceeded the shortest path within a destination buffer.
3. Weinberger, Millard-Ball & Hampshire (2020) extended the trajectory approach with a GPS-trace methodology for distinguishing parking-search excess travel from baseline trip patterns. The methodology is implemented in the open-source *Cruise Detector* codebase (FHWA), whose published thresholds (`(matchdist − netwkdist) > 5 m`, `max_dist ≤ 1400 m`, and `frc_inbuffer > 0.5`) operationalise the classification. The peer-reviewed paper carries three authors; the codebase carries additional contributors. This work remains the methodological reference point for trajectory-based cruising estimation.

Each prior method has a known coverage limitation: block-walk observation does not scale beyond a handful of study blocks; connected-vehicle data covers only the subset of the fleet that subscribes to the underlying telematics product; and both methods are blind to vehicles whose trajectories never touch a study block or instrumented vehicle. As a result, published city-wide cruising figures rest on extrapolations from samples that are demonstrably non-representative of the full vehicle population.

The NYC DOT operates a network of 943 public traffic cameras with continuous public HLS streams. This network produces a dense, near-continuous record of vehicle passage at intersections and along arterials, citywide, without imposing a fleet-subscription bias. The Ekstra Observation API processes a subset of these feeds in real time and emits per-vehicle track records with a coarse, per-camera, non-reversible appearance signature suitable for reappearance detection within a single camera. The instrument is therefore well-suited to a complementary, observation-based estimate of cruising at city scale, and to a direct comparison against the prior block-walk and trajectory-based estimates.

### 1.3 Scope and contribution

The study is descriptive and observational. It does not attempt to evaluate a parking-policy intervention. Its contribution is methodological: it establishes a third, sensor-network-based estimator of cruising, with characterised confounds and validation, that can be cross-referenced against the existing block-walk and trajectory literatures.

---

## 2. Hypotheses

The four hypotheses below are pre-registered before observation of any analytic dataset for the formal study window. Coverage data used for instrument shakedown (the 24-hour soak of 2026-04-26 through 2026-04-27, totalling 9,644 deduplicated reappearance events) is treated as engineering data and is not part of the analytic sample.

**H1.** *Within New York City, the cruising rate at a camera-hour level is positively associated with paid-meter occupancy on the same blockface, controlling for time-of-day fixed effects.* Pre-specified test: linear mixed model, cruising rate as outcome, log-odds-transformed meter-occupancy fraction (NYC DOT ParkNYC live feed) as primary predictor, camera and hour-of-week as random effects. Decision rule: H1 is supported if the estimated coefficient on meter occupancy is positive and the 95% CI excludes zero.

**H2.** *The cruising rate at a corridor-day level is negatively associated with the per-capita supply of public off-street garage spaces within a 400 m walking radius of the corridor centroid, controlling for residential density and the H1 covariates.* Pre-specified test: linear mixed model, corridor-day cruising rate as outcome, garage-supply per resident as primary predictor. Decision rule: H2 is supported if the estimated coefficient is negative and the 95% CI excludes zero.

**H3.** *The diurnal cruising-rate profile in commercial-arterial scene_class corridors is bimodal, with peaks aligned to the AM and PM commute windows defined by NYC DOT traffic-volume data.* Pre-specified test: a Hartigan dip-test on the per-corridor diurnal profile, after detrending for total volume; declared bimodal if the dip statistic is significant at α = 0.05 after Bonferroni correction across corridors.

**H4.** *Excess driver-hours per occupied curb-space-hour, expressed as an opportunity-cost dollar value at the federal urban hourly wage rate, exceed the prevailing meter rate in at least 30% of NYC neighbourhoods (defined by Neighborhood Tabulation Areas, 2020 vintage) during paid-meter hours.* Pre-specified test: paired comparison between estimated cost-of-cruising and posted meter rate at NTA-week granularity; report fraction of NTAs where the lower 95% CI of the cost-of-cruising estimate exceeds the meter rate. Decision rule: H4 is supported if that fraction is at least 0.30.

If the Week 1 coverage audit (§5.5) reveals that camera support is insufficient to power one or more of these tests at the planned granularity, the affected hypothesis is dropped from the formal sample and reported as under-powered, rather than tested at a coarser unit of analysis.

---

## 3. Cruising signal: operational definition

### 3.1 The two-window construction

The cruising signal is defined by a same-camera, same-appearance-signature pair of vehicle observations whose time gap falls within a specified window. The study uses a two-parameter design that separates two distinct concerns:

| Parameter | Range | Role |
|---|---|---|
| `SIGNAL_WINDOW` | [120, 480] s (2–8 min) | The cruising-signal definition reported in methodology and in all consumer-facing outputs |
| `CAPTURE_WINDOW` | [60, 600] s (1–10 min) | The wider archival window used by the in-RAM rolling buffer; permits post-hoc re-cutting of the signal window without losing data |

All published rates use the signal window; the capture window exists to enable signal-window re-cuts under sensitivity analysis without re-instrumenting the pipeline. The capture window never appears in a published rate.

### 3.2 Justification for the signal-window bracket

The published cruising-for-parking literature does not use a same-camera reappearance time window as a unit of analysis. The block-walk method (Shoup) yields per-trip cruise duration, not a same-station gap. The connected-vehicle methods (Hampshire & Shoup; Weinberger, Millard-Ball et al.) classify trajectories by route excess, not by reappearance time. The 1400 m maximum-distance threshold in Cruise Detector implies an upper bound on plausible single-trip excess travel (at urban average speeds of ~25 km/h, roughly three minutes of excess driving), but this distance threshold does not directly translate into a same-camera reappearance gap, because reappearance time depends on loop geometry rather than total excess distance. The bracket is therefore justified by physical reasoning and empirical signal characterisation, not by direct citation.

**Lower bound (W_min = 120 s).** A Manhattan single-block loop has a perimeter of approximately 660 m (typical north-south block ≈ 80 m, east-west avenue block ≈ 250 m). At 15 mph (≈ 6.7 m/s) pure driving time is ≈ 98 s; adding two right-turn delays and a partial traffic-signal cycle (≈ 20 s each) brings the floor to approximately 150 s. A 120 s lower bound therefore admits the fastest plausible single-block circling. The bound is also chosen to lie a factor of four above the typical interval at which the ByteTrack tracker drops and reacquires a track-id (sub-30 s), excluding tracker-artefact reappearances by construction.

**Upper bound (W_max = 480 s).** Beyond eight minutes the conditional probability that two same-camera, same-signature observations are part of one continuous cruising session falls below the probability that one of them is a separate trip (an errand, a passenger drop-off, a return leg). Shoup's reported median cruise durations in dense urban areas sit at the upper end of single-digit minutes, so an eight-minute ceiling admits the bulk of plausible cruising while limiting errand contamination.

**Empirical context.** A 24-hour instrument-shakedown probe on 2026-04-27 (`scratch/gap_distribution_probe.py`) yielded 9,644 deduplicated same-camera, same-signature reappearance pairs in the [60, 600] s capture window. Normalised to a 30-second-equivalent rate, every band in the histogram falls in the 5–6% range. The distribution is essentially uniform. There is no empirical elbow that would identify a natural cut point. A flat distribution is consistent with a population that mixes tracker artefacts at the low end, signature-collision concurrent vehicles uniformly across time, and actual cruising whose density-versus-time is unknown. The empirical histogram alone therefore cannot identify a defensible cut; the cut is set by the physical reasoning above and the resulting choice is treated as an open question for external review (§10.6).

### 3.3 Signal-rate construction

Let `R_cam(t, Δt)` denote the count of same-camera, same-signature reappearance pairs observed at camera `cam` whose start times fall in the interval `(t, t + Δt]`. Let `T_cam(t, Δt)` denote the total count of distinct camera passes (track starts) at the same camera over the same interval. The **raw reappearance rate** is

  raw_rate(cam, t, Δt) = R_cam(t, Δt) / T_cam(t, Δt).

The raw rate over-counts cruising because two unrelated vehicles that happen to share a 13-bit signature within the signal window also produce a pair. Let `c` denote the per-camera expected collision rate (§4.4). The **adjusted reappearance rate** is

  adjusted_rate(cam, t, Δt) = max(raw_rate(cam, t, Δt) − c, 0).

All published cruising rates are adjusted rates. Both the raw and adjusted values, the value of `c` used, and the version of the signature used are emitted with every API response (`EKSTRA_OBSERVATION_API.md` §4); the publication products in `data/` carry the same provenance.

---

## 4. Appearance signature

### 4.1 Construction

The Ekstra appearance signature (signature_version `s_v1`, locked Option A in `EKSTRA_OBSERVATION_API.md` §3.5 and §7.1) is a 13-bit, per-camera, non-reversible descriptor of a vehicle's visual presentation as observed by a single camera. It is constructed from three components:

| Component | Bits | Construction |
|---|---|---|
| Dominant body colour | 8 | HSV-quantised median pixel of the vehicle bounding-box interior. Hue is quantised to 16 buckets (4 bits); saturation and value are each quantised to 4 buckets (2 bits each). The 8-bit colour byte is rendered as 2 lowercase hex characters. |
| Bounding-box size class | 3 | Median bbox-diagonal pixel length over the track, bucketed into eight size classes (`xs, s, m, l, xl, xxl, huge, other`) |
| Stationary-fraction class | 2 | Fraction of frames in which the bbox centroid moves less than a per-camera scale-normalised threshold, bucketed into four levels (mostly-passing, partly-stationary, mostly-stationary, fully-stationary) |

The serialised form is an 8-character hex string: `ah_{color:02x}{size:01x}{dwell:01x}` (the `ah_` prefix denotes "appearance hash"). Example: `ah_a4s5d2`.

The total information content is 8 + 3 + 2 = 13 bits, partitioning the per-camera vehicle population into approximately 8,000 distinct signature buckets, sufficient to support same-camera reappearance detection while remaining well below thresholds that could be construed as biometric or vehicle-identifying.

### 4.2 Per-camera scope

The signature is **per-camera, not cross-camera**. The colour bucket depends on local exposure, white balance, and shading; the size bucket depends on the camera's field of view and mounting geometry; the stationary-fraction depends on the camera's frame rate and detection cadence. Two observations of the same vehicle by two different cameras are not expected to yield the same signature, and the study makes no cross-camera matching claim.

### 4.3 Privacy posture

The signature is, by construction, not a plate read, not a face vector, not an OSNet ReID embedding, and not invertible to a vehicle identifier. It is a deliberately coarse summary chosen to support same-camera reappearance detection while remaining well below thresholds that could be construed as biometric identification of vehicles or people. The privacy posture is detailed in §12 and is aligned with NYC Open Data Law §23-502(d), the POST Act, NYC DOT terms of service for camera feeds, and the project's internal non-negotiables (`VISION.md`).

### 4.4 Expected collision rate

Two unrelated vehicles passing the same camera within the signal window will share a signature with probability `c`. For `s_v1`, the per-camera expected `c` lies in the range 3–8% in the empirical instrument data, varying with camera-specific colour/size distributions of local fleets. Each API response carries a per-camera estimated `c` derived from a rolling background distribution (`EKSTRA_OBSERVATION_API.md` §3.5). The validation methodology in §9 uses a manually labelled stratified sample to confirm or correct the estimated `c`.

---

## 5. Camera selection and stratification

### 5.1 Source

The candidate camera frame is the NYC DOT public traffic-camera network (`https://webcams.nyctmc.org/`), comprising 943 public HLS-streamed cameras at intersections and along arterials citywide. Each camera carries a stable Motion Address (Ekstra OS), public lat/lon (DOT metadata), and a scene_class assigned by the Ekstra Space Engine (currently versioned `space_engine_worker_v1`).

### 5.2 Calibration-tier reality

The Ekstra perception pipeline supports three calibration tiers (`calibrated`, `approximate`, and `uncalibrated`) corresponding to whether a camera has a homography that places detections in real-world coordinates with verified accuracy, a coarse homography from a single annotated reference frame, or no homography at all. As of 2026-04-27, **all 72 cameras currently active in the Ekstra ingest run at tier `approximate`.** No NYC DOT camera in the active set has a verified-accuracy calibration. The full study sample therefore uses `approximate`-tier data uniformly, and the statistical methodology (§7) and limitations (§10) are framed accordingly.

The original methodology placeholder anticipated a tier-mixed analysis with a Week 1 coverage gate that required at least one `calibrated` camera. That gate cannot be satisfied with the current camera fleet, and the study has been re-scoped to operate within `approximate`-tier data only. Calibration-tier-aware confidence remains a future-work direction; it is not a precondition for the present study.

### 5.3 Scene-class stratification

Cameras are assigned a `scene_class` by the Space Engine. The classes relevant to cruising analysis are `commercial_arterial`, `residential_street`, `mixed_arterial`, and `civic_corridor`. Cameras in classes inappropriate to on-street parking (`expressway`, `bridge_or_tunnel`, `non_road`) are excluded from the analytic sample. The stratification used for the H1–H4 analyses is:

| Stratum | Definition | Role |
|---|---|---|
| Commercial-arterial cameras | scene_class = `commercial_arterial` | Primary stratum for H1 (meter occupancy effect) and H3 (diurnal bimodality) |
| Mixed-arterial cameras | scene_class = `mixed_arterial` | Secondary stratum; included in H2 corridor analyses |
| Residential-street cameras | scene_class = `residential_street` | Background stratum; cruising rate expected to be low; serves as a negative control for H1 |
| Civic-corridor cameras | scene_class = `civic_corridor` | Excluded from H1–H3; reserved for H4 cost aggregation if NTA support requires |

Within each stratum, sampling is exhaustive: every camera in the active Ekstra ingest with the appropriate scene_class is included. The study does not down-sample within stratum.

### 5.4 Geographic representativeness

The active Ekstra ingest does not yet cover the full NYC DOT network. The current 72-camera active set is a non-random subset of the 943 DOT cameras, drawn from those with stable HLS endpoints, sufficient nighttime visibility, and scene_class assignments in the relevant categories. The representativeness of this subset relative to the full NYC vehicle environment is a known limitation (§10.1) and is reported alongside every published rate.

### 5.5 Week 1 coverage gate

Before any H1–H4 test is run on Week 1 analytic data, the coverage report (`GET /api/v1/observations/coverage?hours=168`) must satisfy:

- At least 60 cameras with `tracks_written > 0` over the 168-hour window;
- At least 40 cameras with `commercial_arterial` or `mixed_arterial` scene_class meeting the `tracks_written > 0` floor;
- The `track_floor_ratio` for the median camera in the 30–60% band, indicating that the `MIN_FRAMES_TO_PUBLISH` floor is neither too lenient (>60%, suggesting tracker noise dominates) nor too strict (<30%, suggesting genuine short tracks are being suppressed).

If the gate is not satisfied, the formal sample window is extended by one week and the gate is re-evaluated. If the gate fails after a 28-day extension, the affected hypotheses are reserved for follow-up with expanded camera coverage and are not tested at the current granularity.

### 5.6 Distribution-shift trigger

The signal-window selection (§3) and the label-sample stratification (§9) rest on the assumption that the 24-hour instrument-shakedown distribution (9,644 events on 2026-04-27) is representative of the formal-study distribution. If the Week 1 capture-window distribution differs from the shakedown distribution by more than 20% in any 60-second bucket, the population calculation underlying the label sample is re-derived and the label budget is re-allocated before proceeding to formal analysis.

---

## 6. Data pipeline

### 6.1 End-to-end flow

The full pipeline from sensor to published rate proceeds in seven stages:

1. **Camera ingest.** The Ekstra `camera_rtsp` provider opens an HLS stream from each NYC DOT camera in the active set, decodes frames at a configurable rate (typically 5–10 fps), and emits Motion Packets to the runtime daemon.
2. **Detection and tracking.** The Ekstra Space Engine runs a stock YOLOv8s vehicle detector exported to ONNX, with a nighttime-enhancement preprocessing stage, on each decoded frame. Detection classes follow the COCO 80-class taxonomy and are restricted at use to the vehicle subset (`car`, `truck`, `bus`, `motorcycle`). ByteTrack associates frame-level detections into per-camera vehicle tracks. Each track carries a `track_id`, `first_seen_ts`, `last_seen_ts`, frame-by-frame bounding boxes, and a per-track classification. The full perception-stack specification is in `EKSTRA_SPACE_ENGINE.md`.
3. **Track persistence.** When a track terminates and meets the `MIN_FRAMES_TO_PUBLISH` floor, the Space Engine worker (build-id `space_engine_worker_v1`) constructs a track payload (bbox-centroid summary, classification, first/last seen timestamps) and posts it to the platform API. Each track row carries a `source` field stamped by the worker; this provenance column is required by `DATA_INTEGRITY_RULES` Rule 1 and is materialised in production by migration V25 (deployed 2026-04-27).
4. **Signature computation.** At track-publish time, the worker computes the appearance signature (§4) over the buffered frames and writes it to the `appearance_signature` column on the track row.
5. **Reappearance detection.** The platform API exposes a same-camera, same-signature pair query (`GET /api/v1/observations/{cam}/reappearances`, `EKSTRA_OBSERVATION_API.md` §4). At consumer query time, all pairs within the configured window are returned along with a per-camera estimated collision rate.
6. **Validation-frame archival.** A bounded subset of cameras (the `archival_targets.json` allowlist, currently nine cameras drawn from the active 72) writes a pinned frame to `validation_frames` at every track-start and track-end. These frames are retained for 14 days, are not redistributed, are not used for model training, and exist solely to support the manual labelling protocol of §9. Validation-frame archival is not a default capability and is automatically removed at the end of the formal study window (`EKSTRA_OBSERVATION_API.md` §12).
7. **Aggregation and publication.** The analysis code in `code/` issues observation queries with explicit camera sets, time windows, and signature versions; computes rates at camera-hour, corridor-day, and NTA-week granularity; runs the H1–H4 tests; and emits figures and tables to `data/` with full provenance (§11).

### 6.2 Provenance and data integrity

Every track row in `camera_tracks` carries a `source` value identifying the build that wrote it. Following deployment, post-deploy track writes are stamped with the canonical worker source value; pre-deploy rows retain a backfill marker and are excluded from the analytic sample. The study sample is restricted to rows produced under the V25 contract; legacy rows are retained for archival and historical-comparison purposes but are not used in the formal analysis. The `source` column thereby permits a clean separation between data produced under the locked V25 contract and data produced under earlier, unaudited code paths.

### 6.3 Aggregation hierarchy

Rates are reported at three nested granularities, each with its own confidence-interval treatment (§7):

| Level | Unit | Sample size (typical) | Use |
|---|---|---|---|
| Camera-hour | (camera × hour-of-week) | 1 to ~10² pairs | H1, H3 inputs; raw research-tier output |
| Corridor-day | (corridor × calendar-day) | 10² to 10³ pairs | H2 inputs; primary research-tier publication |
| NTA-week | (NTA × calendar-week) | 10³ to 10⁴ pairs | H4 inputs; primary public-facing publication |

Corridors are defined as contiguous segments of cameras along a named NYC street; NTAs follow the NYC Department of City Planning 2020 vintage. The mapping from camera to corridor and NTA is fixed at the start of the study window and is published with the data products.

### 6.4 Adjusted-versus-raw publication policy

Every published rate is reported as both raw and adjusted (§3.3). The adjusted value is the headline; the raw value, the per-camera collision-rate estimate `c`, and the signature version are reported alongside, in the same row of the same table. No published figure or table reports an adjusted rate without the corresponding raw rate and `c`.

---

## 7. Statistical methodology

### 7.1 Sample size

The 24-hour instrument shakedown of 2026-04-27 produced 9,644 deduplicated reappearance pairs from the validation-frame archive. The validation-frame archival worker rotates among the nine archival-target cameras with one camera active at a time, so the shakedown throughput is approximately one-ninth of what continuous nine-camera capture would yield, and a small fraction of the throughput from the full active set. The formal study runs against the full active set (currently 72 cameras at `approximate` tier) without rotation, so the per-window pair count in the formal sample is expected to be substantially higher than the shakedown total. A direct linear extrapolation from the shakedown to a four-week formal-window pair count is therefore not warranted; the formal-window population is measured at the Week 1 gate (§5.5) rather than projected from the shakedown.

Power analysis is preliminary. The realised pair count, distribution across cameras, and inter-camera variance are observed at Week 1, and the final analysis lock (with finalised hypothesis-test specifications, sample-size confirmation, and pre-registered decision rules) is registered before any H1–H4 test is executed on Week 2 or later data. If realised power is insufficient for any specific hypothesis at the planned granularity, that hypothesis is reported as under-powered rather than tested. H2 power is camera-coverage-bound rather than pair-count-bound; H4 power depends on inter-NTA variance and is reported as a sensitivity in the published results.

### 7.2 Confidence intervals

Reappearance-rate confidence intervals are constructed using a clustered bootstrap (Cameron, Gelbach & Miller 2008) with cameras as the cluster unit. Each bootstrap replicate resamples cameras with replacement, then resamples camera-hours within each drawn camera with replacement, then recomputes the adjusted rate. The reported 95% interval is the [2.5, 97.5] percentile of 10,000 replicates. This double-clustered construction respects both within-camera serial correlation and between-camera heterogeneity.

For rates expressed at the NTA-week level, the bootstrap clustering is at the (NTA × week) level rather than the camera level, with within-cluster resampling at the camera-hour grain.

### 7.3 Sensitivity analyses

Three pre-specified sensitivity analyses are reported alongside the primary results:

1. **Capture-window re-cut.** Each rate is re-computed at the wider [60, 600] s capture window, and at a tighter [180, 360] s sub-window, to assess robustness of the headline rate to the [120, 480] s cut.
2. **Signature-version cross-check.** A small subsample (~1% of pairs) is re-scored with a second signature definition that adds a 1-bit "estimated motion direction" field, to assess whether the headline rate changes materially under a richer signature.
3. **Collision-rate substitution.** Each rate is re-computed using the validation-derived `c` (§9) in place of the model-estimated `c`, to assess whether the choice of `c` source materially shifts the adjusted rate.

### 7.4 Aggregation weighting

Within a corridor-day or NTA-week, camera-hours are weighted by the count of distinct camera passes `T_cam(t, Δt)` rather than by camera count. This choice gives heavier-traffic camera-hours proportionally more weight, which matches the analytic interpretation of the cruising rate as a fraction of curb-front traffic. An equal-weight aggregation is reported as a sensitivity.

---

## 8. Confound handling

### 8.1 For-hire-vehicle and livery cruising

A FHV (Uber, Lyft, livery) waiting between rides may pass the same camera repeatedly within the signal window, generating reappearance pairs that are not parking-cruising. The Ekstra signature does not distinguish FHVs from private vehicles. The study addresses this confound through three mechanisms:

1. **Dwell-class filter.** The signature's stationary-fraction component (§4.1) flags vehicles whose track shows materially more stationary frames than the camera-pair average. Pairs in which both endpoints carry a stationary-fraction class of `mostly-stationary` or above are flagged in the published data products. The headline rate is reported both with and without these flagged pairs.
2. **Planned NYC TLC trip-record cross-reference.** The NYC Taxi & Limousine Commission publishes anonymised FHV trip records at NTA × hour granularity. A planned secondary analysis will compute the ratio of cruising rate to FHV trip density at NTA × hour and test whether this ratio is constant across NTAs (which would suggest the cruising rate is dominated by FHV behaviour) or varies materially (which would suggest the cruising rate is responsive to neighbourhood-level parking conditions, the H1 prediction). The TLC ingest is not committed at the time of this methodology version; it is registered here as a planned analysis, contingent on Week 1 gate satisfaction and TLC data availability.
3. **External review (§10.6).** The choice of how to attribute observed reappearance to parking-cruising versus FHV idling is explicitly framed as a methodological question for external review prior to formal publication. The study reports both attributed and un-attributed rates rather than collapsing the choice into a single number.

### 8.2 Delivery-vehicle dwell

Delivery vehicles (USPS, UPS, FedEx, Amazon, food-delivery) park briefly mid-block, depart, and return repeatedly through the day. They generate same-camera, same-signature pairs that are not parking-cruising. The study uses two filters:

1. **Bounding-box size restriction.** The published cruising rate is computed over signature size-classes `s, m, l` (passenger sedan through SUV/small truck). Sizes `xl, xxl, huge` are reported separately as a "delivery-class" rate but are not aggregated into the headline cruising rate.
2. **Time-of-day restriction.** The headline H1 rate is restricted to paid-meter-hours windows (typically 09:00–19:00 weekdays, varying by neighbourhood per NYC DOT meter-zone schedules). Outside these windows, cruising-for-paid-parking is undefined and the rate is reported only for context.

### 8.3 Queue and signal-phase artifacts

A queue at a red light or a long signal cycle can cause two distinct vehicles to occupy the same camera-frame region within the signal window with similar appearance signatures. This is a tracker-association artifact rather than a vehicle re-identification. Two mechanisms address it:

1. **`MIN_FRAMES_TO_PUBLISH` floor.** Tracks whose duration falls below this floor are not published; this excludes the shortest fragments that contribute most heavily to queue-artifact pair generation.
2. **`ABS(time_offset_ms)` deduplication.** The reappearance query uses `DISTINCT ON (camera, track_id, first_seen_ts) ORDER BY ABS(time_offset_ms)` to select the single closest-time frame for each track, avoiding cluster-inflation when a track has multiple archived frames.

### 8.4 Signature collisions

The expected collision rate `c` is the principal quantitative confound (§4.4). It is handled via the adjusted-rate construction (§3.3), the per-camera estimate carried in every API response, and the validation-derived empirical estimate (§9).

### 8.5 Calibration tier

Because the full study sample runs at `approximate` tier (§5.2), tier-related confounds are uniform across the sample and do not contaminate within-sample comparisons. They do affect the absolute scale of any cost-of-cruising estimate (H4), and are reported in the H4 confidence interval as a calibration-uncertainty term.

---

## 9. Validation methodology

### 9.1 Manual labelling protocol

A stratified random sample of reappearance events is manually labelled by two human raters using the validation-frame archive (§6.1.6). The labelling protocol is implemented in `scratch/save_v1_validation_review.py` and produces a labelled set in `validation_labels`. For each labelled event, the rater is shown the two pinned frames (the start frame of the first track and the start frame of the second track) and answers a fixed three-question schedule:

1. *Same vehicle?* (yes / no / unsure)
2. *If yes, is this consistent with cruising for parking?* (yes / no / unsure)
3. *Notable confound?* (FHV-idling / delivery-dwell / signature-collision / queue-artifact / none / other)

Inter-rater agreement is computed as Cohen's κ on Q1 and Q2 separately. Where raters disagree, a third senior rater adjudicates. The published `c` (§4.4) and the FHV-attribution share (§8.1) draw on the resolved labels.

### 9.2 Sample design

The label sample is drawn after the W_min decision (`scratch/wmin_reconciliation_memo.md`, locked 2026-04-27). The sample is stratified across three signal-window-interior buckets:

| Bucket | Range | Target labels | Population basis |
|---|---|---|---|
| B1 | [120, 180) s | ~33 | ~17% of in-window pairs |
| B2 | [180, 300) s | ~33 | ~38% of in-window pairs |
| B3 | [300, 480) s | ~33 | ~45% of in-window pairs |

The total label budget is n = 100 events drawn from the in-window population (estimated 6,438 events on the 2026-04-27 shakedown day, after excluding the 60–119 s and 480–600 s out-of-signal bands). The marginal-error margin at this n is approximately ±9.0 percentage points at 95% CI for an underlying same-vehicle rate around 0.7. Within each bucket, sampling is round-robin across cameras with proportional quotas, with disjoint track-id-set preference (events from previously sampled tracks are deferred), and with time-spread preference (events within a single bucket are spread across the full 24-hour shakedown window).

### 9.3 Expansion trigger

If after the initial n = 100 label run, any per-bucket same-vehicle rate has a 95% CI that overlaps with the 95% CI of an adjacent bucket, the label budget is expanded to n = 200 with the additional 100 events stratified identically. This trigger is operational and is not contingent on the result of the labelling. It is decided in advance by the CI-overlap criterion alone.

### 9.4 Validation-frame retention

`validation_frames` are retained for 14 days and are removed at the end of the formal study window (`EKSTRA_OBSERVATION_API.md` §12). They are operator-only-authenticated, are not redistributed, are not used for model training, and are not exposed via any public API surface. The labelled outputs (`validation_labels`) are retained beyond the frame TTL but contain no image data: only the rater's verdict, confidence, and notable-confound flag.

### 9.5 Feed into expected_collision_rate

The labelled set produces an empirical per-camera estimate of the same-vehicle rate `s_emp`, which by construction satisfies `s_emp = 1 − c_emp` for the in-sample event population (subject to the FHV/delivery confounds of §8). The `c_emp` value is compared against the model-estimated `c_model` carried in the API response. Where they disagree materially, the API response is updated to use `c_emp` for the camera in question, and the underlying model-estimation logic is flagged for a code-side correction. The threshold for "material" disagreement is set at the analysis lock (§7.1) once the empirical between-camera variance of `c_emp` has been observed. The validation-derived `c_emp` is also used in the §7.3 sensitivity analysis.

---

## 10. Limitations and known biases

**10.1 Camera-set non-randomness.** The active 72-camera set is a non-random subset of the 943 NYC DOT cameras. Cameras with stable HLS endpoints, sufficient nighttime visibility, and on-road scene_class are over-represented; cameras pointed away from the curb, at expressway segments, or with intermittent feeds are under-represented. Published rates apply to the support of the active set, not to the city-wide average, and the active-set composition is published alongside every result.

**10.2 Uniform calibration tier.** All 72 active cameras run at `approximate` tier. Within-sample comparisons are unaffected; absolute cost-of-cruising estimates (H4) carry an additional calibration-uncertainty term.

**10.3 Per-camera signature.** The signature is not designed for cross-camera matching. The study makes no claim that observed cruising at camera A and at adjacent camera B is the same vehicle. Corridor-level rates are aggregations of per-camera rates, not corridor-trajectory reconstructions.

**10.4 Coarse signature collision.** The 13-bit signature has a non-trivial expected collision rate (§4.4). The adjusted-rate construction handles the average case; outlier-camera collision (e.g., a camera dominated by yellow taxis, where colour-bucket variance is degenerate) may produce locally elevated `c` that the model under-estimates. The validation-derived `c_emp` is the corrective.

**10.5 W_min/W_max are physically motivated, not literature-cited.** The signal-window bracket [120, 480] s is justified by physical reasoning and empirical-distribution context (§3.2), not by citation to a prior cruising study. The choice is a methodological decision specific to this study, and is the subject of the principal external-review question (§10.6 below).

**10.6 Open methodological questions for external review.** Three questions are pre-registered as open and are reported alongside the headline results:

- *Q1. Is [120, 480] s a defensible signal-window bracket for an observational camera study?* Tighter or wider brackets are technically feasible and we will report the [60, 600] s capture-window result as a sensitivity.
- *Q2. How should the FHV-idling versus parking-cruising attribution be made?* The study reports both attributed and un-attributed rates and treats this as a question for the field rather than a settled answer.
- *Q3. Is the cruising rate, as defined here, a valid lower-bound estimator of true cruising volume?* The same-camera reappearance criterion misses cruising vehicles that do not pass the same camera twice within the window; the study reports its rate as a defensible lower bound rather than a point estimate of true cruising.

**10.7 Temporal coverage.** The formal study window is four weeks. Seasonal variation in cruising (particularly between summer street-fair conditions, autumn weekday baseline, and winter snow-affected conditions) is not addressed within a single study window. Multi-season replication is reserved for a follow-up.

**10.8 Single-city scope.** The methodology is developed for New York City. Translating it to other cities requires re-fitting the signature collision-rate model, re-stratifying the camera scene-class assignments, and re-evaluating the W_min/W_max physical reasoning against local block geometry.

---

## 11. Reproducibility

### 11.1 Code

The full analysis pipeline lives in `code/`. The Ekstra Observation API client and aggregation utilities are MIT-licensed (`LICENSE`); the analysis scripts and figure-generation code are released under the same terms. Each figure, table, and headline rate in the published paper is reproducible from a single command and a stable API client version.

### 11.2 Data

Aggregated data products in `data/` are released under CC-BY-4.0 (`LICENSE-DATA`) at the granularity defined in §6.3. No raw video, no per-frame imagery, and no per-track bounding-box series are released; the privacy posture of §12 forbids it.

### 11.3 Versioning

Every published artifact carries:

1. The exact API query string used (camera set, time window, signature version, scene_class filter, calibration-tier filter).
2. The git SHA of the `code/` tree at analysis time.
3. The signature_version (`s_v1`).
4. The Space Engine worker build-id (`space_engine_worker_v1`).
5. The platform-API revision and migration version (currently V25 for the formal study).
6. The list of cameras in the analytic sample with their scene_class and calibration tier at measurement time.

### 11.4 Methodology page commitments

This methodology page is itself versioned (CITATION.cff). Substantive methodological changes after a published rate result in a new methodology version and a re-published rate, not a silent overwrite. The version history is preserved in the repository.

### 11.5 Consumer-cache requirement

Per `EKSTRA_OBSERVATION_API.md` §7.2, the Ekstra Observation API enforces a 72-hour rolling retention on raw track data. Studies whose analytic windows exceed 72 hours (including the present study) are required to maintain a consumer-side cache of API responses, with timestamps and full request provenance, for the duration of the study window. The NYC Cruising Observatory study maintains this cache; it is the canonical record for the published results and is preserved beyond the API's own retention window.

---

## 12. Privacy and ethics

### 12.1 Data handled

The study handles only the Ekstra Observation API and the bounded validation-frame archive (§9.4). It does not access raw video streams from NYC DOT cameras beyond what the Ekstra perception pipeline ingests in real time, and it does not retain raw frame imagery beyond the 14-day validation-frame TTL. No vehicle plates, no facial features, no person-level data, and no cross-camera re-identification are involved at any stage.

### 12.2 Signature is not biometric

The 13-bit appearance signature (§4) is, by construction, far below thresholds that could be construed as biometric or vehicle-identifying. The information content is 13 bits, partitioning the per-camera vehicle population into approximately 8,000 distinct signature buckets, sufficient to estimate same-camera reappearance under heavy aggregation, insufficient to identify a vehicle. The signature is not a plate read, not a face vector, not an OSNet ReID embedding, and is non-reversible.

### 12.3 Aggregation thresholds

Published data products are released only at granularities sufficient to prevent re-identification of vehicles or individuals:

| Granularity | Minimum population | Release |
|---|---|---|
| Camera-hour | ≥ 30 distinct passes | Research-tier only, with API authentication |
| Corridor-day | ≥ 200 distinct passes | Research-tier with attribution |
| NTA-week | ≥ 1,000 distinct passes | Public release |

Aggregations below these thresholds are not published.

### 12.4 Legal and policy alignment

The study's data-handling posture is aligned with:

- NYC Open Data Law §23-502(d) restrictions on identifiable information;
- the New York City Public Oversight of Surveillance Technology (POST) Act;
- NYC DOT's terms of service for public traffic camera feeds;
- the project's internal non-negotiables documented in `VISION.md`.

### 12.5 Validation-frame archive

The bounded validation-frame archive (§6.1.6, §9.4) is operator-only-authenticated, is not redistributed, is not used for training of any model, and is removed at the end of the formal study window. Its existence is disclosed in this methodology page and in `EKSTRA_OBSERVATION_API.md` §12. It is the only place in the pipeline where bounded raw imagery is retained beyond the perception-pipeline ingest moment, and its scope is the minimum required to support the manual labelling protocol of §9.

### 12.6 No ad-targeting use

The Ekstra Observation API output and all data products produced by this study are not used for ad-targeting, vehicle-fleet identification, or any commercial vehicle-level analytics. The non-commercial-vehicle-analytics posture is a project non-negotiable and is enforced by the API's signature design.

---

## Bibliography

Cameron, A. C., Gelbach, J. B., & Miller, D. L. (2008). Bootstrap-based improvements for inference with clustered errors. *Review of Economics and Statistics*, 90(3), 414–427.

Hampshire, R. C., & Shoup, D. (2018). What share of traffic is cruising for parking? *Journal of Transport Economics and Policy*, 52(3), 184–201.

Weinberger, R. R., Millard-Ball, A., & Hampshire, R. C. (2020). Parking search caused congestion: Where's all the fuss? *Transportation Research Part C: Emerging Technologies*, 120, 102781.

Shoup, D. (1995). An opportunity to reduce minimum parking requirements. *Journal of the American Planning Association*, 61(1), 14–28.

Shoup, D. (2006). Cruising for parking. *Transport Policy*, 13(6), 479–486.

*end methodology*
