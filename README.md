# Clinical Risk Cohort Pipeline

A specialty-aware stack that turns linked clinical and claims data into prioritised, auditable outreach lists for **endocrinology** and **cardiology**, built around one idea: **risk is inferred from what is absent in the record, not only from abnormal values.**

A patient with uncontrolled glucose and *no* medication-management claim — or atrial fibrillation with *no* anticoagulant on record — is invisible to a tool that reads clinical values alone. That gap lives in the claims stream, and this pipeline treats its absence as the actionable signal.

> [!IMPORTANT]
> Research and portfolio artifact demonstrating architecture and data-integrity patterns on **synthetic data**. Not a medical device, not validated, and not for clinical decision-making or use against real patient data without independent review, validation, and appropriate regulatory and IRB processes. All thresholds and code sets shown here are illustrative.

---

## Design principles

- **Absence as a first-class signal.** Every clinical measure resolves to one of four explicit states — present, absent, or present-but-unparseable — so no value silently escapes evaluation.
- **Compute before display.** Clinical computation runs on exact values; privacy transformation is a separate downstream pass. The boundary is explicit and testable.
- **Boundaries in code, not convention.** The model integration enforces an aggregate-only data boundary and a server-side credential boundary structurally, not by trust.
- **Evidence before reason.** A pure, testable assessment decides whether a cohort is precise and stable enough *before* any model call; the statistics are separated from the IO and never fabricated.
- **Open/closed by specialty.** Each specialty is a registered strategy; adding one is a registration, not an edit to the engine.

---

## Architecture

Data flows top-to-bottom through one path. Two of the arrows are structural gates, not conveniences: the **evidence gate** can halt a cohort before any model call, and the **PHI boundary** blocks record-level identifiers from crossing into model review. Anonymisation is deliberately *not* on this path — it runs as a separate downstream pass, after computation on exact values.

```text
       Clinical Records          Claims Data
        (exact values)        (absence = signal)
              └────────────┬────────────┘
                           ▼
             ┌───────────────────────┐
             │    Four-State Engine   │ → getClinicalFlags(); specialty registry
             └───────────┬───────────┘   present · absent · unparseable · in-range
                         ▼
             ┌───────────────────────┐
             │   Risk Stratification  │ → stratifyPatients(); computes on raw values
             └───────────┬───────────┘   tiers: low · high · critical · unknown
                         ▼
       ── compute-before-display boundary ──  anonymisation is a downstream pass
                         ▼
             ┌───────────────────────┐
             │   Cohort Evidence Gate │ → assessCohortEvidence(); pure, no IO
             └───────────┬───────────┘   precision + drift + optional Beta posterior
                         ▼  (proceeds only if the cohort is fit to analyse)
             ┌───────────────────────┐
             │    Aggregate Summary   │
             └───────────┬───────────┘
                         ▼
       ══ assertAggregateOnly() · PHI boundary ══  no record-level keys or values
                         ▼
             ┌───────────────────────┐
             │       LLM Review       │ → getLLMFeedback(); aggregate-only,
             └───────────────────────┘   proxy holds the credential
```

Each box maps to a module in the table below; the two labelled boundaries are the enforcement points the design principles refer to.

**Why this architecture.** The pipeline is designed so that clinical reasoning, statistical evidence, privacy enforcement, and model interaction are independent layers that can each be audited and tested.

---

## The four-state data model

The core of the flag engine. Each measure lands in exactly one branch, and missing versus malformed route to different remediation:

| State | Example values | Endocrinology (HbA1c) | Routes to |
|---|---|---|---|
| Absent | `null`, `undefined`, `NaN`, `""` | `HBA1C-MISSING` | Clinical outreach |
| Present & unparseable | `"8"`, `"abc"`, `true`, `Infinity` | `HBA1C-INVALID` | Upstream data fix |
| Present & in-range | `6.4` | no flag (controlled) | — |
| Present & out-of-range | `9.1` | gap flags (`MTM-GAP`, `DSMT-GAP`) | Clinical action |

The same model governs cardiology's stroke score (`CHADSVASC-MISSING` / `CHADSVASC-INVALID`), gated on an AFib context.

---

## Modules

| Module | Responsibility | Key safeguard |
|---|---|---|
| `getClinicalFlags()` | Care-gap detection via the specialty strategy registry | Four-state absence inference; immutable thresholds |
| `stratifyPatients()` | Graded risk tier (`low`/`high`/`critical`/`unknown`) + flags | Computes on raw values; `unknown` when unmeasured |
| `makeTokenizer()` | Keyed (HMAC-SHA256) patient pseudonymisation | Key server-side only; output remains PHI |
| `validateDataset()` | Field validation with value redaction | Reports the rule, suppresses the patient value |
| `assessCohortEvidence()` | Whether a cohort can be analysed (precision + drift + optional Beta posterior) | Honest, **uncalibrated** prior; never fabricated from a variance |
| `getLLMFeedback()` | Streaming model review of cohort summaries | Gates on evidence; aggregate-only; proxy holds the credential |
| `assertAggregateOnly()` | PHI boundary guard before the model call | Blocks record-level **keys and** structured identifier **values** |

### Specialty strategy engine

Each specialty registers a `{ flags, tier }` handler in `SPECIALTY_STRATEGIES`. `getClinicalFlags()` and `stratifyPatients()` resolve the handler and delegate, so neither needs editing to support a new specialty, and each strategy is independently testable. An unsupported specialty fails loudly with the supported list rather than silently producing an empty cohort.

### Stratification — compute before display

Tiers run `low → high → critical`, using the per-specialty `*_HIGH` and `*_CRITICAL` thresholds. When the deciding measure is absent or unparseable the tier is `unknown` — an unmeasured patient is indeterminate, never defaulted to low. Computation reads exact values; anonymisation is a separate downstream layer.

### Cohort gating — evidence before reason

`assessCohortEvidence()` is a pure function (no IO) that decides whether a cohort summary is fit to analyse, and answers two real questions honestly:

- **Sufficiency / precision** — the standard error and 95% margin of error of the cohort mean (`√(variance/n)`), gated against a minimum *n* and a margin ceiling. The check is **monotonic**: more data never perversely blocks a cohort.
- **Drift** — a variance ratio against a `baseline_variance` that must be **supplied** (fit from history upstream). With no baseline the check is *not evaluated* — it is never invented from a placeholder constant.

An optional **proper Beta–Binomial posterior** is computed *only* when an integer `event_count` is present (coherent update `α += events`, `β += n − events`), with a credible interval from an incomplete-beta quantile validated against SciPy. The prior is **weakly-informative and explicitly uncalibrated**, and the audit metadata reports only what was computed — it never claims a calibration that did not happen.

### Privacy & model boundaries

Keyed HMAC tokenisation is **pseudonymisation, not de-identification** — tokens are re-identifiable with the server-held key, so output remains PHI. `validateDataset()` retains rule metadata (field, valid range) while suppressing the offending value, so error logs don't become a secondary PHI surface. `assertAggregateOnly()` is a **structural backstop**: it blocks record-level identifier *keys* and additionally scans *string values* for high-confidence structured identifiers (email, SSN, phone, ISO/US date-of-birth), reporting the matched pattern and JSON path but never the value. It still cannot reliably detect names in free text, and the bare long-digit-run check is opt-in (`SCAN_DIGIT_RUNS`) because clinical codes (NDC/CPT/ICD) are digit runs. It remains a backstop, not a substitute for upstream de-identification.

---

## Configuration

All thresholds and code sets live in a single deep-frozen `PIPELINE_CONFIG`. The freeze is genuine: nested threshold objects are frozen and the code `Set` mutators are disabled, so clinical constants cannot be mutated at runtime.

```js
endocrinology: { THRESHOLDS: { HBA1C_HIGH: 8.0, HBA1C_CRITICAL: 10.0 }, ... }
cardiology:    { THRESHOLDS: { CHA2DS2VASC_HIGH: 2, CHA2DS2VASC_CRITICAL: 4 }, ... }
```

---

## Usage

```js
import { getClinicalFlags, stratifyPatients } from "./Diabetes_Pipeline.js";

const cohort = stratifyPatients(clinicalRecords, claims, "endocrinology");
// → [{ ...patient, tier: "high", clinicalFlags: [{ code: "MTM-GAP", ... }] }, ...]
```

`stratifyPatients` retains raw values for accuracy; run your display-anonymisation pass downstream before anything reaches a render path.

---

## Testing

A regression harness (`test_pipeline.mjs`, run with `node test_pipeline.mjs`) exercises 22 checks and fails loudly on any drift:

- the four-state flag model and the `low / high / critical / unknown` tier ladder;
- the specialty guard (unsupported specialty throws);
- the PHI key **and** value scan (blocks email/SSN/phone/DOB in values; passes legitimate aggregates and clinical codes);
- cohort-gating monotonicity (margin of error strictly decreases with *n*);
- the **no-unearned-provenance** invariant — the audit prior is `null` with no event count and labelled `uncalibrated` otherwise;
- the Beta credible interval against reference values.

---

## Status & license

- **Data:** synthetic / demo only.
- **Stage:** architecture and data-integrity demonstration; not validated, not production.
- **Stack:** JavaScript (ES modules), WebCrypto for keyed tokenisation, a dependency-free incomplete-beta/quantile implementation (validated against SciPy) for the credible interval, and a backend proxy for model access. Confirm current model strings and API headers in the provider's documentation before any deployment.
- **License:** MIT. Copyright © 2026 Katherine J. Ombrellaro. The software grant carries no warranty of clinical validity and does not authorise use with real patient data.

---

*Author: Katherine J. Ombrellaro — Clinical Performance Analytics. This README documents a study system on synthetic data and makes no claim of clinical validity.*
