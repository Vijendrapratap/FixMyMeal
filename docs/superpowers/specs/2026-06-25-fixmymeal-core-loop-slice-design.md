# FixMyMeal — Core-Loop Slice (Design Spec)

**Date:** 2026-06-25
**Status:** Approved for planning
**Scope:** First of several phased slices of the FixMyMeal PRD. This spec covers
**only** the thin vertical "core loop." Each later subsystem gets its own
spec → plan → build cycle.

---

## 1. Context & Decomposition

The FixMyMeal PRD describes a full product (15 user stories) spanning ~6
independent subsystems: identity/profile, lab ingestion, knowledge base,
generation engine, lifestyle rules, and the safety layer — plus the PWA shell.
That is far too large for a single design/build pass, so we decompose and build
one honest slice first.

**Product intent:** ship to real users. This rules out faking the parts that
matter — auth/RLS must be real, the medical boundary must be enforced in
architecture, and the evidence must be genuinely non-fabricated.

**This slice — the core loop:**

```
Sign up → Health profile + BMI → Pick persona → Enter markers (manual) →
Generate 7-day plan → View day×meal grid with sourced food cards
```

It exists to validate the riskiest assumption before investing in breadth:
*can we reliably produce honest, sourced, India-specific plans without
hallucinating citations?*

### In scope
- Real auth (Supabase Auth: email/password **+ Google**); all user data
  owner-only via RLS.
- Health profile (age, sex, height, weight, activity, conditions) → BMI with
  category band.
- Persona selector (Therapeutic / Fitness / General) — biases generation.
- **Manual** marker entry with reference-range high/low flagging and severity
  (Hb, ferritin, B12, vitamin D, calcium).
- Curated KB: ~15–18 Indian-staple foods, fully sourced, two evidence streams
  (Research + Ayurveda).
- Generation engine: deterministic eligibility → "assemble-not-author"
  constrained JSON → validator → persist.
- Visual 7-day grid + sourced food cards (benefit, mechanism, amount/frequency,
  Research + Ayurveda badges).
- Safety layer: persistent disclaimer + abnormal-marker "consult a physician"
  interstitial + constrained system prompt.

### Explicitly deferred to later slices
Vision/PDF OCR marker extraction, dosha quiz, lifestyle timing (sun/sleep),
supplements, conversational refinement, open Q&A, month plans, PWA/offline,
n8n orchestration.

### Deliberate deviations from the PRD (this slice only)
1. **Synchronous generation** in a Next.js server action rather than n8n — a
   weekly plan is fast enough; n8n earns its place only for month-plans / bulk
   regeneration later.
2. **Embeddings/pgvector scaffolded but retrieval starts rule-based**
   (marker → nutrient → food mapping). With ~18 foods, deterministic
   eligibility is more reliable than vector similarity. The `vector` column is
   added now so there is no migration later; vector retrieval turns on when the
   KB grows.

### Health focus
The engine is **focus-agnostic** — it reasons over whatever markers and
conditions a user has. Only the *seeded data* is bounded for launch. The
starter food set deliberately spans both deficiencies **and** a common
condition.

Starter set (≈15–18 staples, final list locked during data curation), covering
iron / B12 / calcium / vitamin D / vitamin A **and** acidity:

> moong dal · spinach (palak) · curd/dahi · ragi · sesame (til) · amla ·
> paneer · chana · bajra · peanuts · milk · banana · ghee · jaggery ·
> buttermilk · jeera · saunf · coconut water

---

## 2. Stack

| Layer | Choice | Note |
|---|---|---|
| Frontend | Next.js (App Router), Tailwind | mobile-first; PWA wrapper deferred |
| Data/Auth/Storage | Supabase (Postgres + pgvector + Auth + RLS) | one system; RLS on every user table |
| AI | Claude (Anthropic API), `claude-sonnet-4-6` for generation | strict JSON output, constrained to retrieved rows |
| Hosting | Vercel | synchronous generation (no n8n yet) |

**Prerequisites for the build/plan phase:** a Supabase project (DB + Auth +
pgvector extension) and an Anthropic API key. Google OAuth credentials for the
Google sign-in option.

---

## 3. Data Model

Two groups. **User-owned** tables enforce `user_id = auth.uid()` RLS
(owner-only). **Curated KB** tables are authenticated-read, service-role-write
(users cannot mutate evidence).

### User-owned
- **`profiles`** — `id` (= auth.users.id), age, sex, height_cm, weight_kg,
  activity_level, conditions[] , bmi, disclaimer_ack_at, timestamps.
- **`goal_profiles`** — user_id, persona (`therapeutic|fitness|general`).
  Kept as its own 1:1 table (per PRD) so personas can carry goals/targets later.
- **`blood_markers`** — user_id, marker_name, value, unit, ref_low, ref_high,
  flag (`normal|high|low`), severity (`normal|borderline|severe`), source
  (`manual`), timestamps.
- **`plans`** — user_id, persona, week_start, plan_json (validated structure),
  markers_snapshot (jsonb), status, created_at.

### Curated KB
- **`foods`** — name, name_local, category, embedding `vector` (scaffolded).
- **`nutrients`** — canonical list (iron, b12, vitamin_d, calcium, vitamin_a,
  protein, …), unit.
- **`food_nutrients`** — food_id, nutrient_id, amount_per_serving, serving_desc,
  source_id (citation for the *value*, e.g. USDA).
- **`food_benefits`** — food_id, benefit, mechanism, serving_guidance,
  frequency, addresses[] (markers/nutrients/conditions it targets), embedding
  (scaffolded).
- **`evidence`** — benefit_id, stream (`research|ayurveda`), strength
  (`strong|moderate|traditional`), citation_text, source_url, reference,
  license_note.

**`ayurveda_properties` is folded into `evidence.stream='ayurveda'`** for this
slice (one table, two streams). **`supplements` is deferred.**

### Evidence sourcing & licensing
- Evidence is **compiled by the assistant, then human-verified** before launch
  (the PRD's mandatory confirmation gate, applied to seed data).
- Safe public-domain sources: USDA FoodData Central (nutrient values), NIH ODS
  fact sheets (mechanisms). ICMR-NIN guidelines and classical Ayurvedic text
  references require a licensing/attribution check before use; each evidence row
  records a `license_note`.
- Every `food_benefits` row gets ≥1 **Research** evidence row (strong/moderate,
  real citation + URL). An **Ayurveda** row is added only where a clean,
  attributable classical reference exists (labeled *traditional*). If none
  exists, the card omits that badge — **we never invent a citation to fill a
  slot.**

---

## 4. Generation Pipeline — "Assemble, Never Author"

This is the architectural decision that makes "never fabricate a citation"
structurally true rather than a hope.

```
1. ELIGIBILITY (deterministic)
   flagged markers + conditions + persona
   → target nutrients/benefits (rule map)
   → SELECT foods whose food_benefits.addresses ∩ targets ≠ ∅
   → candidate set: foods + nutrients + benefits + evidence (stable IDs)

2. PROMPT ASSEMBLY
   compact JSON of candidates (ids only) + medical-guardrail system prompt
   → "use ONLY these ids; invent nothing"

3. GENERATE (Claude Sonnet)
   structured-output JSON, schema-constrained:
   week[].meals{breakfast|lunch|snack|dinner}[] = { food_id, portion, benefit_ids[] }

4. VALIDATE  ★ keystone ★
   every food_id ∈ candidates; every benefit_id/evidence_id ∈ that food's rows;
   schema shape valid; no stray free-text citations.
   fail → bounded retry (×2) feeding errors back → then honest error.
   NEVER render unvalidated output.

5. PERSIST
   validated plan_json + markers_snapshot → plans

6. RENDER
   grid + cards hydrate from KB rows (foods/benefits/evidence).
   LLM text is NEVER shown as a citation.
```

**Load-bearing pieces:**
- **The validator is the keystone** — a pure function (LLM output + candidate
  set → verdict, no I/O). Step 3 (constrained generation) + step 4 (validation)
  = belt and suspenders. The after-the-fact validator from the rejected
  "LLM authors, app validates" approach is retained here purely as defense in
  depth.
- **Two evidence streams, no forced fabrication** — as described in §3.

---

## 5. User Journey & Screens

Mobile-first, thumb-reachable, one primary action per screen.

| # | Screen | What happens | Persists to |
|---|---|---|---|
| 1 | **Auth** | Email/password + Google (Supabase Auth) | `auth.users` |
| 2 | **Disclaimer gate** *(first run)* | Non-dismissable medical disclaimer; must acknowledge | `profiles.disclaimer_ack_at` |
| 3 | **Profile + BMI** | Guided form → BMI + category band shown instantly | `profiles` |
| 4 | **Persona** | Therapeutic / Fitness / General (3 cards) | `goal_profiles` |
| 5 | **Markers** | Add marker from known list (unit + range pre-filled) → auto-flag green/amber/red by severity; editable | `blood_markers` |
| 6 | **Abnormal interstitial** *(conditional)* | If any marker is **severe** → "consult a physician" before any plan | — |
| 7 | **Plan** | "Generate" → pipeline → 7-day × 4-meal grid; tap food → sourced card; Regenerate | `plans` |

**Screen 7 — grid & card:**
- Grid: 7 rows (days) × 4 columns (breakfast/lunch/snack/dinner); each cell a
  tappable food chip; horizontal-scroll on mobile.
- Sourced card (bottom sheet on mobile): food name → benefit → mechanism →
  amount/frequency → two separated badges: 🔬 **Research** (strong/moderate +
  source link) and 🪔 **Ayurveda** (traditional + text reference). Card content
  comes 100% from KB rows.

**Honesty on missing coverage:** if a user's markers/conditions map to targets
with **no eligible curated foods**, show an honest "not enough verified guidance
yet" message — never a padded plan.

---

## 6. Safety Layer (three architectural layers)

1. **Severity gate (deterministic).** Each marker has a *severe* threshold
   compiled from defensible references (e.g. Hb < 8 g/dL = severe anemia per
   WHO; B12 < 150 pg/mL; 25-OH vitamin D < 10 ng/mL) and human-verified. Any
   `severity='severe'` → interstitial (screen 6) **before** generation. The user
   may acknowledge and still get *general, clearly-hedged* guidance — never a
   confident "this fixes it" plan.
2. **Constrained system prompt.** Forbids diagnosis, prescription, drug names,
   and any dosage framed as treatment; instructs assemble-only-from-candidates.
   Lives in one auditable place.
3. **Persistent disclaimer.** Non-dismissable on first run (screen 2), then a
   slim always-visible strip on the markers and plan surfaces.

---

## 7. Component Boundaries

Small units, one purpose each; pure logic separated from I/O for isolated
testing.

```
lib/
  bmi.ts              pure: (height, weight) → { bmi, category }
  markers.ts          pure: (value, range, thresholds) → { flag, severity }
  eligibility.ts      pure: (markers, conditions, persona) → target set
                      (+ thin query to fetch candidates by targets)
  generation/
    prompt.ts         pure: (candidates, profile, persona) → messages + JSON schema
    client.ts         ◆ ONLY place that calls Claude (mocked elsewhere)
    validator.ts      ★ pure: (llmOutput, candidateSet) → {ok,plan} | {errors}
    index.ts          orchestrates: eligibility → prompt → client → validate → retry → persist
  supabase/           typed server/browser clients (RLS-aware)
supabase/
  migrations/         schema + RLS policies
  seed/               curated foods/nutrients/benefits/evidence (compile → verify)
components/           PlanGrid · FoodCard · MarkerList · BmiBadge ·
                      DisclaimerBanner · AbnormalInterstitial
```

`validator.ts` touches no I/O → deterministic and adversarially testable.
`client.ts` is the single Claude boundary → every other test mocks it.

---

## 8. Error Handling

| Failure | Behavior |
|---|---|
| LLM output fails validation | bounded retry (×2) feeding errors back → then honest "couldn't build a reliable plan"; never render unvalidated output |
| Claude API error/timeout | retryable error surfaced; plan not saved |
| No eligible foods for targets | honest empty-state (no padded plan) |
| Cross-user access | denied by RLS *and* re-checked in server action |
| Session expiry / bad form input | redirect to sign-in / inline validation |

---

## 9. Testing Strategy

Ship-to-real-users ⇒ real tests; **TDD the load-bearing core.**

- **Unit (TDD first):** `bmi`, `markers` (flag/severity incl. boundary values),
  `eligibility`, and especially **`validator`** with adversarial fixtures —
  unknown food_id, fabricated evidence_id, stray prose, malformed JSON, and the
  valid case.
- **Integration:** generation orchestrator with a **mocked Claude client**
  returning valid / hallucinated / malformed → assert the validator gates and
  that persistence happens *only* on success. Plus RLS tests (owner CRUD ok,
  others denied).
- **Safety audit (automated):** assert no plan contains a food_id outside
  candidates; severe markers always route to the interstitial; scripted check
  that output carries **zero medicine names / treatment dosages** (PRD success
  metric).
- **Real-Claude smoke (sparingly):** one live generation against the seeded KB
  that must pass validation — catches prompt/schema drift.
- **Browser verification:** each flow via the dev-browser skill; profile →
  first-plan under the 3-minute target.

---

## 10. Success Criteria (for this slice)

- A real user can go sign-up → first plan in under 3 minutes.
- 100% of rendered food cards cite KB-backed sources; 0 fabricated citations
  (enforced by the validator, verified by the safety audit).
- 0 outputs containing medicine names or treatment dosages (audited).
- Severe markers always route to the physician interstitial before a plan.
- All user tables enforce owner-only RLS (verified by tests).
- Typecheck, lint, and the test suite pass.

---

## 11. Open Items (carried to planning, not blocking)

- Final locked food list and per-food evidence rows (produced in the data
  curation step, human-verified).
- Exact severe-threshold values per marker (compiled, human-verified).
- Whether `goal_profiles` stays 1:1 or gains fields this slice (default: minimal
  — persona only).
```
