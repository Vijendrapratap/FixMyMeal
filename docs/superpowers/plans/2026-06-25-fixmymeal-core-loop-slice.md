# FixMyMeal Core-Loop Slice — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the honest core loop — sign up → health profile + BMI → persona → manual markers → an AI-assembled, KB-sourced 7-day Indian meal plan rendered as a grid with sourced cards, with the medical boundary enforced architecturally.

**Architecture:** Next.js (App Router) + Supabase (Postgres + pgvector + Auth + RLS) + Claude (`claude-sonnet-4-6`). Generation is "assemble, never author": deterministic eligibility selects candidate foods from a curated KB; Claude is given ONLY those rows and returns strict JSON referencing `food_id`/`benefit_id`s that exist in the candidate set; a pure validator rejects any id outside the set; cards render from DB rows, never from LLM text.

**Tech Stack:** TypeScript, Next.js (App Router, server actions), Tailwind, `@supabase/ssr`, `@supabase/supabase-js`, `@anthropic-ai/sdk`, `zod`, Vitest. Full design spec: [docs/superpowers/specs/2026-06-25-fixmymeal-core-loop-slice-design.md](../specs/2026-06-25-fixmymeal-core-loop-slice-design.md).

## Global Constraints

- **Model:** generation uses `claude-sonnet-4-6` (the PRD specifies Sonnet). No `budget_tokens`, no `temperature`/`top_p`, no assistant prefills — all 400 on current models. Use `client.messages.parse({ output_config: { format: zodOutputFormat(schema) } })` for structured output; read `response.parsed_output`.
- **No fabrication:** the LLM may only reference `food_id`/`benefit_id`/`evidence_id` values present in the candidate set passed to it. Cards render from KB rows, never from LLM free text. The validator enforces this and is the keystone safety component.
- **Medical boundary:** the generation system prompt forbids diagnosis, prescription, drug names, and treatment dosages. Severe markers route to a "consult a physician" interstitial before any plan. A persistent disclaimer is visible on markers and plan surfaces.
- **RLS everywhere:** every user-owned table (`profiles`, `goal_profiles`, `blood_markers`, `plans`) enforces `user_id = auth.uid()` owner-only. KB tables are authenticated-read, service-role-write.
- **Evidence is compiled-then-human-verified:** seed data is drafted from USDA / NIH ODS / ICMR-NIN with real citation URLs and a `license_note`; a human verifies before launch. Never invent a citation to fill a slot — omit the Ayurveda badge if no defensible source exists.
- **API verification:** before writing Next.js/Supabase glue, confirm current signatures against official docs — Next.js App Router `cookies()` is async in Next 15+, and `@supabase/ssr` uses `getAll`/`setAll` cookie methods. Confirm `@anthropic-ai/sdk` exposes `messages.parse` + `zodOutputFormat` from `@anthropic-ai/sdk/helpers/zod` at the installed version.
- **Secrets:** `ANTHROPIC_API_KEY`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` live in `.env.local` (gitignored). The service-role key is used only in server-side seed/admin scripts, never shipped to the client.
- **Every task ends green:** `npm run typecheck` and `npm run test` pass before commit. UI tasks additionally verify in-browser with the dev-browser skill.

---

### Task 1: Project scaffold & tooling

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.ts`, `tailwind.config.ts`, `postcss.config.mjs`, `vitest.config.ts`, `app/globals.css`, `app/layout.tsx`, `app/page.tsx`, `.env.local.example`, `.gitignore`
- Create: `src/lib/.gitkeep`

**Interfaces:**
- Produces: `npm run dev`, `npm run build`, `npm run typecheck` (`tsc --noEmit`), `npm run test` (`vitest run`), `npm run lint`. Path alias `@/*` → `src/*`.

- [ ] **Step 1: Scaffold Next.js + TS + Tailwind**

Run: `npx create-next-app@latest . --typescript --tailwind --app --src-dir --import-alias "@/*" --no-eslint --use-npm`
Expected: project files created in the current directory (it is empty except README/docs).

- [ ] **Step 2: Add dev/test dependencies**

Run: `npm install @supabase/ssr @supabase/supabase-js @anthropic-ai/sdk zod && npm install -D vitest @vitejs/plugin-react vite-tsconfig-paths typescript`
Expected: dependencies installed, exit 0.

- [ ] **Step 3: Add scripts to `package.json`**

```jsonc
// package.json — "scripts" block
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "typecheck": "tsc --noEmit",
  "test": "vitest run",
  "test:watch": "vitest"
}
```

- [ ] **Step 4: Create `vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths()],
  test: { environment: "node", include: ["src/**/*.test.ts"] },
});
```

- [ ] **Step 5: Create `.env.local.example`**

```bash
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
ANTHROPIC_API_KEY=
```

Append `.env.local` to `.gitignore` if not already present.

- [ ] **Step 6: Smoke-test the toolchain**

Add a trivial passing test `src/lib/smoke.test.ts`:

```ts
import { expect, test } from "vitest";
test("toolchain runs", () => { expect(1 + 1).toBe(2); });
```

Run: `npm run typecheck && npm run test`
Expected: typecheck clean; 1 test passes. Then delete `src/lib/smoke.test.ts`.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "chore: scaffold Next.js + Supabase + Vitest toolchain"
```

---

### Task 2: Database schema & RLS

**Files:**
- Create: `supabase/migrations/0001_core_loop_schema.sql`
- Create: `src/lib/db/types.ts` (hand-written row types used across the app)

**Interfaces:**
- Produces: tables `profiles`, `goal_profiles`, `blood_markers`, `plans`, `foods`, `nutrients`, `food_nutrients`, `food_benefits`, `evidence`; the `vector` extension; RLS policies. TS row types: `Profile`, `GoalProfile`, `BloodMarker`, `Plan`, `Food`, `Nutrient`, `FoodNutrient`, `FoodBenefit`, `Evidence`.

- [ ] **Step 1: Write the migration SQL**

```sql
-- supabase/migrations/0001_core_loop_schema.sql
create extension if not exists vector;

-- ---------- User-owned ----------
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  age int,
  sex text check (sex in ('male','female','other')),
  height_cm numeric,
  weight_kg numeric,
  activity_level text check (activity_level in ('sedentary','light','moderate','active','very_active')),
  conditions text[] not null default '{}',
  bmi numeric,
  disclaimer_ack_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table goal_profiles (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  persona text not null check (persona in ('therapeutic','fitness','general')),
  created_at timestamptz not null default now(),
  unique (user_id)
);

create table blood_markers (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  marker_name text not null,
  value numeric not null,
  unit text not null,
  ref_low numeric,
  ref_high numeric,
  flag text not null check (flag in ('normal','high','low')),
  severity text not null check (severity in ('normal','borderline','severe')),
  source text not null default 'manual',
  created_at timestamptz not null default now()
);

create table plans (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  persona text not null,
  week_start date,
  plan_json jsonb not null,
  markers_snapshot jsonb not null default '[]',
  status text not null default 'active',
  created_at timestamptz not null default now()
);

-- ---------- Curated KB ----------
create table foods (
  id uuid primary key default gen_random_uuid(),
  name text not null unique,
  name_local text,
  category text,
  embedding vector(1536),
  created_at timestamptz not null default now()
);

create table nutrients (
  id uuid primary key default gen_random_uuid(),
  key text not null unique,          -- e.g. 'iron','b12','vitamin_d','calcium','vitamin_a','protein'
  display_name text not null,
  unit text not null
);

create table food_nutrients (
  id uuid primary key default gen_random_uuid(),
  food_id uuid not null references foods(id) on delete cascade,
  nutrient_id uuid not null references nutrients(id) on delete cascade,
  amount_per_serving numeric not null,
  serving_desc text not null,
  source_id uuid,                    -- evidence row backing this value (nullable; set after evidence seeded)
  unique (food_id, nutrient_id)
);

create table food_benefits (
  id uuid primary key default gen_random_uuid(),
  food_id uuid not null references foods(id) on delete cascade,
  benefit text not null,
  mechanism text not null,
  serving_guidance text not null,
  frequency text not null,
  addresses text[] not null default '{}',  -- nutrient keys / condition keys this benefit targets
  embedding vector(1536)
);

create table evidence (
  id uuid primary key default gen_random_uuid(),
  benefit_id uuid not null references food_benefits(id) on delete cascade,
  stream text not null check (stream in ('research','ayurveda')),
  strength text not null check (strength in ('strong','moderate','traditional')),
  citation_text text not null,
  source_url text,
  reference text,
  license_note text
);

-- ---------- RLS ----------
alter table profiles enable row level security;
alter table goal_profiles enable row level security;
alter table blood_markers enable row level security;
alter table plans enable row level security;

create policy "own profile" on profiles
  for all using (auth.uid() = id) with check (auth.uid() = id);
create policy "own goal_profiles" on goal_profiles
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "own blood_markers" on blood_markers
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "own plans" on plans
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- KB: authenticated read, no client writes (service role bypasses RLS for seeding)
alter table foods enable row level security;
alter table nutrients enable row level security;
alter table food_nutrients enable row level security;
alter table food_benefits enable row level security;
alter table evidence enable row level security;

create policy "kb read foods" on foods for select to authenticated using (true);
create policy "kb read nutrients" on nutrients for select to authenticated using (true);
create policy "kb read food_nutrients" on food_nutrients for select to authenticated using (true);
create policy "kb read food_benefits" on food_benefits for select to authenticated using (true);
create policy "kb read evidence" on evidence for select to authenticated using (true);
```

- [ ] **Step 2: Apply the migration**

Apply via the Supabase MCP `apply_migration` tool (name `0001_core_loop_schema`) or the Supabase CLI (`supabase db push`). Then confirm with `list_tables` that all nine tables exist with RLS enabled.
Expected: nine tables present; `rowsecurity = true` on the four user-owned tables and five KB tables.

- [ ] **Step 3: Write `src/lib/db/types.ts`**

```ts
export type Persona = "therapeutic" | "fitness" | "general";
export type Flag = "normal" | "high" | "low";
export type Severity = "normal" | "borderline" | "severe";
export type Stream = "research" | "ayurveda";
export type Strength = "strong" | "moderate" | "traditional";

export interface Profile {
  id: string; age: number | null; sex: "male" | "female" | "other" | null;
  height_cm: number | null; weight_kg: number | null;
  activity_level: string | null; conditions: string[]; bmi: number | null;
  disclaimer_ack_at: string | null;
}
export interface GoalProfile { id: string; user_id: string; persona: Persona; }
export interface BloodMarker {
  id: string; user_id: string; marker_name: string; value: number; unit: string;
  ref_low: number | null; ref_high: number | null; flag: Flag; severity: Severity; source: string;
}
export interface Plan {
  id: string; user_id: string; persona: Persona; week_start: string | null;
  plan_json: unknown; markers_snapshot: unknown; status: string;
}
export interface Food { id: string; name: string; name_local: string | null; category: string | null; }
export interface Nutrient { id: string; key: string; display_name: string; unit: string; }
export interface FoodNutrient { id: string; food_id: string; nutrient_id: string; amount_per_serving: number; serving_desc: string; }
export interface FoodBenefit {
  id: string; food_id: string; benefit: string; mechanism: string;
  serving_guidance: string; frequency: string; addresses: string[];
}
export interface Evidence {
  id: string; benefit_id: string; stream: Stream; strength: Strength;
  citation_text: string; source_url: string | null; reference: string | null; license_note: string | null;
}
```

- [ ] **Step 4: Typecheck & commit**

Run: `npm run typecheck`
Expected: clean.

```bash
git add -A
git commit -m "feat(db): core-loop schema, RLS policies, and row types"
```

---

### Task 3: Supabase clients, auth, and session middleware

**Files:**
- Create: `src/lib/supabase/server.ts`, `src/lib/supabase/client.ts`, `src/lib/supabase/admin.ts`, `src/middleware.ts`
- Create: `app/(auth)/login/page.tsx`, `app/(auth)/login/actions.ts`, `app/auth/callback/route.ts`

**Interfaces:**
- Consumes: env vars from Task 1.
- Produces: `createServerSupabase()` (RLS-scoped, server components/actions), `createBrowserSupabase()`, `createAdminSupabase()` (service-role, seed/admin only); middleware that refreshes the session cookie; an email/password + Google login flow that lands authenticated users in onboarding.

- [ ] **Step 1: Server & browser clients (`@supabase/ssr`)**

```ts
// src/lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createServerSupabase() {
  const cookieStore = await cookies(); // async in Next 15+
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (toSet) => {
          try { toSet.forEach(({ name, value, options }) => cookieStore.set(name, value, options)); }
          catch { /* called from a Server Component; middleware refreshes instead */ }
        },
      },
    },
  );
}
```

```ts
// src/lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";
export function createBrowserSupabase() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  );
}
```

```ts
// src/lib/supabase/admin.ts  — service role; server-only scripts/actions
import { createClient } from "@supabase/supabase-js";
export function createAdminSupabase() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    { auth: { persistSession: false } },
  );
}
```

- [ ] **Step 2: Session-refresh middleware**

```ts
// src/middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => request.cookies.getAll(),
        setAll: (toSet) => {
          toSet.forEach(({ name, value }) => request.cookies.set(name, value));
          response = NextResponse.next({ request });
          toSet.forEach(({ name, value, options }) => response.cookies.set(name, value, options));
        },
      },
    },
  );
  await supabase.auth.getUser();
  return response;
}

export const config = { matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"] };
```

- [ ] **Step 3: Login page + server actions (email/password + Google)**

```tsx
// app/(auth)/login/page.tsx
import { signInWithPassword, signUpWithPassword, signInWithGoogle } from "./actions";

export default function LoginPage() {
  return (
    <main className="mx-auto max-w-sm p-6 space-y-4">
      <h1 className="text-xl font-semibold">FixMyMeal</h1>
      <form className="space-y-3">
        <input name="email" type="email" required placeholder="Email" className="w-full rounded border p-2" />
        <input name="password" type="password" required placeholder="Password" className="w-full rounded border p-2" />
        <button formAction={signInWithPassword} className="w-full rounded bg-emerald-700 p-2 text-white">Log in</button>
        <button formAction={signUpWithPassword} className="w-full rounded border p-2">Sign up</button>
      </form>
      <form action={signInWithGoogle}>
        <button className="w-full rounded border p-2">Continue with Google</button>
      </form>
    </main>
  );
}
```

```ts
// app/(auth)/login/actions.ts
"use server";
import { redirect } from "next/navigation";
import { headers } from "next/headers";
import { createServerSupabase } from "@/lib/supabase/server";

export async function signInWithPassword(formData: FormData) {
  const supabase = await createServerSupabase();
  const { error } = await supabase.auth.signInWithPassword({
    email: String(formData.get("email")), password: String(formData.get("password")),
  });
  if (error) redirect(`/login?error=${encodeURIComponent(error.message)}`);
  redirect("/onboarding/profile");
}

export async function signUpWithPassword(formData: FormData) {
  const supabase = await createServerSupabase();
  const { error } = await supabase.auth.signUp({
    email: String(formData.get("email")), password: String(formData.get("password")),
  });
  if (error) redirect(`/login?error=${encodeURIComponent(error.message)}`);
  redirect("/onboarding/profile");
}

export async function signInWithGoogle() {
  const supabase = await createServerSupabase();
  const origin = (await headers()).get("origin")!;
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: "google", options: { redirectTo: `${origin}/auth/callback` },
  });
  if (error) redirect(`/login?error=${encodeURIComponent(error.message)}`);
  redirect(data.url);
}
```

```ts
// app/auth/callback/route.ts
import { NextResponse } from "next/server";
import { createServerSupabase } from "@/lib/supabase/server";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");
  if (code) {
    const supabase = await createServerSupabase();
    await supabase.auth.exchangeCodeForSession(code);
  }
  return NextResponse.redirect(`${origin}/onboarding/profile`);
}
```

- [ ] **Step 4: Typecheck**

Run: `npm run typecheck`
Expected: clean.

- [ ] **Step 5: Verify in browser (dev-browser skill)**

Start `npm run dev`. Confirm: `/login` renders; email/password sign-up creates a user (check `auth.users` via Supabase MCP) and redirects to `/onboarding/profile`; the Google button initiates OAuth (configure Google provider in Supabase Auth settings first, or note it as a launch prerequisite if creds are pending).

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(auth): Supabase SSR clients, session middleware, email+Google login"
```

---

### Task 4: BMI calculation (pure, TDD)

**Files:**
- Create: `src/lib/bmi.ts`, `src/lib/bmi.test.ts`

**Interfaces:**
- Produces: `computeBmi(heightCm: number, weightKg: number): { bmi: number; category: BmiCategory }` where `BmiCategory = "underweight" | "normal" | "overweight" | "obese"`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/bmi.test.ts
import { expect, test } from "vitest";
import { computeBmi } from "./bmi";

test("computes BMI and rounds to one decimal", () => {
  const r = computeBmi(170, 65);
  expect(r.bmi).toBe(22.5);
  expect(r.category).toBe("normal");
});

test.each([
  [170, 50, "underweight"],
  [170, 65, "normal"],
  [170, 80, "overweight"],
  [170, 100, "obese"],
])("height %i weight %i -> %s", (h, w, cat) => {
  expect(computeBmi(h, w).category).toBe(cat);
});

test("throws on non-positive inputs", () => {
  expect(() => computeBmi(0, 65)).toThrow();
  expect(() => computeBmi(170, 0)).toThrow();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/bmi.test.ts`
Expected: FAIL — `computeBmi` is not defined.

- [ ] **Step 3: Implement**

```ts
// src/lib/bmi.ts
export type BmiCategory = "underweight" | "normal" | "overweight" | "obese";

export function computeBmi(heightCm: number, weightKg: number): { bmi: number; category: BmiCategory } {
  if (heightCm <= 0 || weightKg <= 0) throw new Error("height and weight must be positive");
  const m = heightCm / 100;
  const bmi = Math.round((weightKg / (m * m)) * 10) / 10;
  const category: BmiCategory =
    bmi < 18.5 ? "underweight" : bmi < 25 ? "normal" : bmi < 30 ? "overweight" : "obese";
  return { bmi, category };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/bmi.test.ts`
Expected: PASS (all cases).

- [ ] **Step 5: Commit**

```bash
git add src/lib/bmi.ts src/lib/bmi.test.ts
git commit -m "feat(bmi): pure BMI computation with category bands"
```

---

### Task 5: Marker flagging & severity (pure, TDD)

**Files:**
- Create: `src/lib/markers.ts`, `src/lib/markers.test.ts`

**Interfaces:**
- Produces:
  - `MARKER_DEFS: Record<string, MarkerDef>` where `MarkerDef = { unit: string; refLow: number; refHigh: number; severeLow?: number; severeHigh?: number; targets: string[] }` (`targets` = nutrient keys/conditions the marker's deficiency implies).
  - `evaluateMarker(name: string, value: number): { flag: Flag; severity: Severity }`.
  - Severe thresholds in `MARKER_DEFS` are illustrative defaults — replaced with human-verified values during Task 7's verification gate.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/markers.test.ts
import { expect, test } from "vitest";
import { evaluateMarker, MARKER_DEFS } from "./markers";

test("hemoglobin within range is normal/normal", () => {
  expect(evaluateMarker("hemoglobin", 14)).toEqual({ flag: "normal", severity: "normal" });
});

test("low hemoglobin flags low, borderline by default", () => {
  expect(evaluateMarker("hemoglobin", 11)).toEqual({ flag: "low", severity: "borderline" });
});

test("severely low hemoglobin flags low + severe", () => {
  expect(evaluateMarker("hemoglobin", 7)).toEqual({ flag: "low", severity: "severe" });
});

test("high marker flags high", () => {
  expect(evaluateMarker("vitamin_b12", 2000).flag).toBe("high");
});

test("boundary value equal to refLow is normal", () => {
  const def = MARKER_DEFS["hemoglobin"];
  expect(evaluateMarker("hemoglobin", def.refLow).flag).toBe("normal");
});

test("unknown marker throws", () => {
  expect(() => evaluateMarker("nonsense", 1)).toThrow();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/markers.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

```ts
// src/lib/markers.ts
import type { Flag, Severity } from "@/lib/db/types";

export interface MarkerDef {
  unit: string; refLow: number; refHigh: number;
  severeLow?: number; severeHigh?: number; targets: string[];
}

// NOTE: severe* thresholds are illustrative defaults — human-verify in Task 7.
export const MARKER_DEFS: Record<string, MarkerDef> = {
  hemoglobin:   { unit: "g/dL",  refLow: 12,  refHigh: 16,  severeLow: 8,   targets: ["iron"] },
  ferritin:     { unit: "ng/mL", refLow: 30,  refHigh: 300, severeLow: 10,  targets: ["iron"] },
  vitamin_b12:  { unit: "pg/mL", refLow: 200, refHigh: 900, severeLow: 150, targets: ["b12"] },
  vitamin_d:    { unit: "ng/mL", refLow: 30,  refHigh: 100, severeLow: 10,  targets: ["vitamin_d"] },
  calcium:      { unit: "mg/dL", refLow: 8.5, refHigh: 10.5, severeLow: 7,  severeHigh: 13, targets: ["calcium"] },
};

export function evaluateMarker(name: string, value: number): { flag: Flag; severity: Severity } {
  const def = MARKER_DEFS[name];
  if (!def) throw new Error(`unknown marker: ${name}`);
  let flag: Flag = "normal";
  if (value < def.refLow) flag = "low";
  else if (value > def.refHigh) flag = "high";

  let severity: Severity = "normal";
  if (flag === "low") severity = def.severeLow !== undefined && value < def.severeLow ? "severe" : "borderline";
  else if (flag === "high") severity = def.severeHigh !== undefined && value > def.severeHigh ? "severe" : "borderline";
  return { flag, severity };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/markers.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/markers.ts src/lib/markers.test.ts
git commit -m "feat(markers): pure flag + severity evaluation with verified-threshold stubs"
```

---

### Task 6: Eligibility mapping (pure target derivation, TDD)

**Files:**
- Create: `src/lib/eligibility.ts`, `src/lib/eligibility.test.ts`

**Interfaces:**
- Consumes: `BloodMarker[]`, `Profile.conditions`, `Persona` (from `@/lib/db/types`); `MARKER_DEFS` (Task 5).
- Produces: `deriveTargets(markers, conditions, persona): string[]` — the deduped set of nutrient/condition keys the plan should address. (The DB query that turns targets into candidate foods lives in Task 11's orchestrator; this task is the pure mapping only.)

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/eligibility.test.ts
import { expect, test } from "vitest";
import { deriveTargets } from "./eligibility";
import type { BloodMarker } from "@/lib/db/types";

const m = (marker_name: string, flag: BloodMarker["flag"]): BloodMarker => ({
  id: "x", user_id: "u", marker_name, value: 1, unit: "", ref_low: null, ref_high: null,
  flag, severity: flag === "normal" ? "normal" : "borderline", source: "manual",
});

test("low hemoglobin yields iron target", () => {
  expect(deriveTargets([m("hemoglobin", "low")], [], "therapeutic")).toContain("iron");
});

test("normal markers contribute no nutrient targets", () => {
  expect(deriveTargets([m("hemoglobin", "normal")], [], "general")).not.toContain("iron");
});

test("acidity condition is passed through as a target", () => {
  expect(deriveTargets([], ["acidity"], "therapeutic")).toContain("acidity");
});

test("targets are deduped", () => {
  const t = deriveTargets([m("hemoglobin", "low"), m("ferritin", "low")], [], "therapeutic");
  expect(t.filter((x) => x === "iron")).toHaveLength(1);
});

test("empty inputs yield empty targets", () => {
  expect(deriveTargets([], [], "general")).toEqual([]);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/eligibility.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

```ts
// src/lib/eligibility.ts
import type { BloodMarker, Persona } from "@/lib/db/types";
import { MARKER_DEFS } from "@/lib/markers";

export function deriveTargets(
  markers: BloodMarker[], conditions: string[], _persona: Persona,
): string[] {
  const targets = new Set<string>();
  for (const marker of markers) {
    if (marker.flag === "normal") continue;
    const def = MARKER_DEFS[marker.marker_name];
    if (def) def.targets.forEach((t) => targets.add(t));
  }
  for (const c of conditions) targets.add(c);
  return [...targets];
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/eligibility.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/eligibility.ts src/lib/eligibility.test.ts
git commit -m "feat(eligibility): pure marker+condition -> target derivation"
```

---

### Task 7: Curated KB seed (compiled → human-verified)

**Files:**
- Create: `supabase/seed/foods.ts` (the structured dataset), `scripts/seed-kb.ts` (idempotent loader), `supabase/seed/VERIFY.md` (the human sign-off checklist)

**Interfaces:**
- Consumes: `createAdminSupabase()` (Task 3).
- Produces: ~15–18 foods, their nutrients, ≥1 `food_benefit` each with `addresses[]`, and ≥1 `research` evidence row per benefit (real citation + URL), `ayurveda` rows only where defensible. `npm run seed:kb` loads it idempotently.

- [ ] **Step 1: Author the dataset (compile from public sources)**

Create `supabase/seed/foods.ts` exporting a typed array. One representative entry shown; populate the full starter set (moong dal, spinach, curd, ragi, sesame, amla, paneer, chana, bajra, peanuts, milk, banana, ghee, jaggery, buttermilk, jeera, saunf, coconut water). Nutrient values from USDA FoodData Central; mechanisms from NIH ODS; `addresses[]` from Task 5/6 keys (`iron`, `b12`, `calcium`, `vitamin_d`, `vitamin_a`, `acidity`).

```ts
// supabase/seed/foods.ts
export interface SeedEvidence {
  stream: "research" | "ayurveda"; strength: "strong" | "moderate" | "traditional";
  citation_text: string; source_url?: string; reference?: string; license_note?: string;
}
export interface SeedBenefit {
  benefit: string; mechanism: string; serving_guidance: string; frequency: string;
  addresses: string[]; evidence: SeedEvidence[];
}
export interface SeedNutrient { key: string; display_name: string; unit: string; amount_per_serving: number; serving_desc: string; }
export interface SeedFood { name: string; name_local?: string; category: string; nutrients: SeedNutrient[]; benefits: SeedBenefit[]; }

export const SEED_FOODS: SeedFood[] = [
  {
    name: "Spinach (palak)", name_local: "पालक", category: "leafy_vegetable",
    nutrients: [
      { key: "iron", display_name: "Iron", unit: "mg", amount_per_serving: 3.6, serving_desc: "100 g cooked" },
      { key: "vitamin_a", display_name: "Vitamin A", unit: "mcg RAE", amount_per_serving: 469, serving_desc: "100 g cooked" },
    ],
    benefits: [
      {
        benefit: "Supports iron intake for deficiency",
        mechanism: "Non-heme iron contributes to haemoglobin synthesis; pair with vitamin C (e.g. amla, lemon) to aid absorption.",
        serving_guidance: "1 cup cooked (~100 g)", frequency: "3–4 times/week", addresses: ["iron"],
        evidence: [
          { stream: "research", strength: "moderate",
            citation_text: "USDA FoodData Central — Spinach, cooked; NIH ODS Iron fact sheet.",
            source_url: "https://ods.od.nih.gov/factsheets/Iron-HealthProfessional/",
            license_note: "US government public domain" },
          { stream: "ayurveda", strength: "traditional",
            citation_text: "Classically described as cooling; traditionally taken cooked, not raw.",
            reference: "Bhavaprakasha Nighantu (verify edition/translation before launch)",
            license_note: "verify licensing of translation" },
        ],
      },
    ],
  },
  // ... remaining foods, each with >=1 research-backed benefit
];
```

- [ ] **Step 2: Write the idempotent loader**

```ts
// scripts/seed-kb.ts
import "dotenv/config";
import { createAdminSupabase } from "@/lib/supabase/admin";
import { SEED_FOODS } from "../supabase/seed/foods";

async function main() {
  const db = createAdminSupabase();
  // upsert nutrients (unique by key)
  const nutrientKeys = new Map<string, { display_name: string; unit: string }>();
  for (const f of SEED_FOODS) for (const n of f.nutrients) nutrientKeys.set(n.key, { display_name: n.display_name, unit: n.unit });
  const { data: nutrients } = await db.from("nutrients")
    .upsert([...nutrientKeys].map(([key, v]) => ({ key, ...v })), { onConflict: "key" }).select();
  const nutrientId = new Map(nutrients!.map((n) => [n.key, n.id]));

  for (const f of SEED_FOODS) {
    const { data: food } = await db.from("foods")
      .upsert({ name: f.name, name_local: f.name_local ?? null, category: f.category }, { onConflict: "name" })
      .select().single();
    for (const n of f.nutrients) {
      await db.from("food_nutrients").upsert(
        { food_id: food!.id, nutrient_id: nutrientId.get(n.key)!, amount_per_serving: n.amount_per_serving, serving_desc: n.serving_desc },
        { onConflict: "food_id,nutrient_id" });
    }
    for (const b of f.benefits) {
      const { data: benefit } = await db.from("food_benefits")
        .insert({ food_id: food!.id, benefit: b.benefit, mechanism: b.mechanism,
          serving_guidance: b.serving_guidance, frequency: b.frequency, addresses: b.addresses })
        .select().single();
      for (const e of b.evidence) {
        await db.from("evidence").insert({ benefit_id: benefit!.id, ...e });
      }
    }
  }
  console.log("KB seed complete.");
}
main().catch((e) => { console.error(e); process.exit(1); });
```

Add to `package.json` scripts: `"seed:kb": "tsx scripts/seed-kb.ts"`, and install the runner: `npm install -D tsx dotenv`.

> Note: `food_benefits` insert (not upsert) is non-idempotent. For re-runs, the loader should first `delete` benefits/evidence for seeded foods, or wrap in a "clear KB" step. Add a `--reset` flag that truncates `evidence`, `food_benefits`, `food_nutrients`, `foods` before loading when iterating.

- [ ] **Step 3: Write the human-verification checklist**

```md
<!-- supabase/seed/VERIFY.md -->
# KB Verification Gate (must pass before launch)
- [ ] Every nutrient amount matches its cited USDA FoodData Central entry.
- [ ] Every research evidence row has a working, non-fabricated source_url.
- [ ] Every Ayurveda row has an attributable text reference OR is removed (no invented citations).
- [ ] Licensing of each ICMR-NIN / classical-text reference confirmed; license_note filled.
- [ ] Severe marker thresholds in src/lib/markers.ts confirmed against WHO/clinical references.
Reviewer: __________  Date: __________
```

- [ ] **Step 4: Run the seed and verify rows**

Run: `npm run seed:kb`
Then via Supabase MCP `execute_sql`: confirm `select count(*) from foods` ≥ 15, every `food_benefits` row has ≥1 `evidence` row with `stream='research'`, and no `evidence.source_url` is null for research rows.
Expected: counts as above; zero research rows missing a URL.

- [ ] **Step 5: Commit**

```bash
git add supabase/seed scripts/seed-kb.ts package.json package-lock.json
git commit -m "feat(kb): compiled seed dataset, idempotent loader, and verification gate"
```

---

### Task 8: Plan schema & prompt assembly (pure, TDD)

**Files:**
- Create: `src/lib/generation/types.ts`, `src/lib/generation/prompt.ts`, `src/lib/generation/prompt.test.ts`

**Interfaces:**
- Produces:
  - `Candidate` type: `{ food_id: string; name: string; category: string | null; benefits: { benefit_id: string; benefit: string; addresses: string[] }[] }`.
  - `PlanSchema` (zod): `{ week: { day: string; meals: { breakfast: Cell[]; lunch: Cell[]; snack: Cell[]; dinner: Cell[] } }[] }`, `Cell = { food_id: string; portion: string; benefit_ids: string[] }`.
  - `SYSTEM_PROMPT: string` (the medical guardrail + assemble-only instruction).
  - `buildUserMessage(candidates: Candidate[], profile, persona, targets): string`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/generation/prompt.test.ts
import { expect, test } from "vitest";
import { SYSTEM_PROMPT, buildUserMessage, PlanSchema } from "./prompt";

const candidates = [{ food_id: "f1", name: "Spinach", category: "leafy",
  benefits: [{ benefit_id: "b1", benefit: "iron", addresses: ["iron"] }] }];

test("system prompt forbids diagnosis and invention", () => {
  expect(SYSTEM_PROMPT.toLowerCase()).toContain("do not diagnose");
  expect(SYSTEM_PROMPT.toLowerCase()).toContain("only");
});

test("user message embeds candidate ids and never leaks unknown ids", () => {
  const msg = buildUserMessage(candidates, { conditions: [] }, "therapeutic", ["iron"]);
  expect(msg).toContain("f1");
  expect(msg).toContain("b1");
  expect(msg).toContain("therapeutic");
});

test("PlanSchema accepts a well-formed plan", () => {
  const ok = PlanSchema.safeParse({
    week: [{ day: "Monday", meals: { breakfast: [{ food_id: "f1", portion: "1 cup", benefit_ids: ["b1"] }], lunch: [], snack: [], dinner: [] } }],
  });
  expect(ok.success).toBe(true);
});

test("PlanSchema rejects a cell missing portion", () => {
  const bad = PlanSchema.safeParse({
    week: [{ day: "Monday", meals: { breakfast: [{ food_id: "f1", benefit_ids: [] }], lunch: [], snack: [], dinner: [] } }],
  });
  expect(bad.success).toBe(false);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/generation/prompt.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement types + prompt**

```ts
// src/lib/generation/types.ts
export interface CandidateBenefit { benefit_id: string; benefit: string; addresses: string[]; }
export interface Candidate { food_id: string; name: string; category: string | null; benefits: CandidateBenefit[]; }
```

```ts
// src/lib/generation/prompt.ts
import { z } from "zod";
import type { Candidate } from "./types";
import type { Persona } from "@/lib/db/types";

export type { Candidate } from "./types";

export const CellSchema = z.object({
  food_id: z.string(),
  portion: z.string().min(1),
  benefit_ids: z.array(z.string()),
});
export const PlanSchema = z.object({
  week: z.array(z.object({
    day: z.string(),
    meals: z.object({
      breakfast: z.array(CellSchema),
      lunch: z.array(CellSchema),
      snack: z.array(CellSchema),
      dinner: z.array(CellSchema),
    }),
  })),
});
export type Plan = z.infer<typeof PlanSchema>;

export const SYSTEM_PROMPT = [
  "You assemble an Indian weekly meal plan from a fixed candidate list.",
  "You are NOT a doctor. Do not diagnose, do not prescribe medicine, do not give drug names or treatment dosages.",
  "You may ONLY use food_id and benefit_id values that appear in the CANDIDATES list.",
  "Never invent foods, nutrients, citations, or ids. If a meal has no suitable candidate, leave it empty.",
  "Return strict JSON matching the required schema: 7 days, each with breakfast/lunch/snack/dinner cells referencing candidate ids.",
].join(" ");

export function buildUserMessage(
  candidates: Candidate[], profile: { conditions: string[] }, persona: Persona, targets: string[],
): string {
  return JSON.stringify({
    persona,
    targets,
    conditions: profile.conditions,
    instruction: "Build a 7-day plan (Monday..Sunday). Use only the candidate ids below. Prefer foods whose benefits address the targets.",
    CANDIDATES: candidates,
  });
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/generation/prompt.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/generation/types.ts src/lib/generation/prompt.ts src/lib/generation/prompt.test.ts
git commit -m "feat(generation): plan zod schema, guardrail system prompt, candidate message"
```

---

### Task 9: Claude client wrapper (the single LLM boundary)

**Files:**
- Create: `src/lib/generation/client.ts`

**Interfaces:**
- Consumes: `SYSTEM_PROMPT`, `buildUserMessage`, `PlanSchema` (Task 8); `ANTHROPIC_API_KEY`.
- Produces: `generatePlanRaw(candidates, profile, persona, targets): Promise<Plan>` — calls `claude-sonnet-4-6` via `messages.parse` with structured output and returns `parsed_output`. This is the ONLY module that imports `@anthropic-ai/sdk`; everything else mocks it.

- [ ] **Step 1: Implement the wrapper**

```ts
// src/lib/generation/client.ts
import Anthropic from "@anthropic-ai/sdk";
import { zodOutputFormat } from "@anthropic-ai/sdk/helpers/zod";
import type { Persona } from "@/lib/db/types";
import { SYSTEM_PROMPT, buildUserMessage, PlanSchema, type Plan, type Candidate } from "./prompt";

export async function generatePlanRaw(
  candidates: Candidate[], profile: { conditions: string[] }, persona: Persona, targets: string[],
): Promise<Plan> {
  const client = new Anthropic(); // reads ANTHROPIC_API_KEY
  const response = await client.messages.parse({
    model: "claude-sonnet-4-6",
    max_tokens: 16000,
    system: SYSTEM_PROMPT,
    messages: [{ role: "user", content: buildUserMessage(candidates, profile, persona, targets) }],
    output_config: { format: zodOutputFormat(PlanSchema) },
  });
  if (!response.parsed_output) throw new Error("model returned no parseable plan");
  return response.parsed_output;
}
```

- [ ] **Step 2: Typecheck**

Run: `npm run typecheck`
Expected: clean. (If `zodOutputFormat`'s import path differs at the installed SDK version, confirm via `node -e "console.log(Object.keys(require('@anthropic-ai/sdk/helpers/zod')))"` and adjust.)

- [ ] **Step 3: Commit**

```bash
git add src/lib/generation/client.ts
git commit -m "feat(generation): Claude client wrapper using messages.parse structured output"
```

---

### Task 10: The validator (keystone, adversarial TDD)

**Files:**
- Create: `src/lib/generation/validator.ts`, `src/lib/generation/validator.test.ts`

**Interfaces:**
- Consumes: `Plan`, `Candidate` (Task 8).
- Produces: `validatePlan(plan: unknown, candidates: Candidate[]): { ok: true; plan: Plan } | { ok: false; errors: string[] }` — pure, no I/O. Guarantees every `food_id` ∈ candidates and every `benefit_id` ∈ that food's benefits, and that the shape matches `PlanSchema`.

- [ ] **Step 1: Write the failing adversarial tests**

```ts
// src/lib/generation/validator.test.ts
import { expect, test } from "vitest";
import { validatePlan } from "./validator";
import type { Candidate } from "./prompt";

const candidates: Candidate[] = [
  { food_id: "f1", name: "Spinach", category: "leafy", benefits: [{ benefit_id: "b1", benefit: "iron", addresses: ["iron"] }] },
];
const cell = (food_id: string, benefit_ids: string[]) => ({ food_id, portion: "1 cup", benefit_ids });
const planWith = (c: ReturnType<typeof cell>) => ({
  week: [{ day: "Monday", meals: { breakfast: [c], lunch: [], snack: [], dinner: [] } }],
});

test("accepts a plan referencing only known ids", () => {
  const r = validatePlan(planWith(cell("f1", ["b1"])), candidates);
  expect(r.ok).toBe(true);
});

test("rejects an unknown food_id", () => {
  const r = validatePlan(planWith(cell("UNKNOWN", ["b1"])), candidates);
  expect(r.ok).toBe(false);
  if (!r.ok) expect(r.errors.join(" ")).toContain("UNKNOWN");
});

test("rejects a fabricated benefit_id for a known food", () => {
  const r = validatePlan(planWith(cell("f1", ["b_fake"])), candidates);
  expect(r.ok).toBe(false);
});

test("rejects malformed JSON shape (missing meals)", () => {
  const r = validatePlan({ week: [{ day: "Monday" }] }, candidates);
  expect(r.ok).toBe(false);
});

test("rejects a non-object", () => {
  expect(validatePlan("not a plan", candidates).ok).toBe(false);
});

test("accepts empty meals (honest no-suitable-food)", () => {
  const r = validatePlan({ week: [{ day: "Monday", meals: { breakfast: [], lunch: [], snack: [], dinner: [] } }] }, candidates);
  expect(r.ok).toBe(true);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/generation/validator.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

```ts
// src/lib/generation/validator.ts
import { PlanSchema, type Plan, type Candidate } from "./prompt";

export function validatePlan(
  plan: unknown, candidates: Candidate[],
): { ok: true; plan: Plan } | { ok: false; errors: string[] } {
  const parsed = PlanSchema.safeParse(plan);
  if (!parsed.success) {
    return { ok: false, errors: parsed.error.issues.map((i) => `${i.path.join(".")}: ${i.message}`) };
  }
  const benefitsByFood = new Map(candidates.map((c) => [c.food_id, new Set(c.benefits.map((b) => b.benefit_id))]));
  const errors: string[] = [];
  for (const day of parsed.data.week) {
    for (const [meal, cells] of Object.entries(day.meals)) {
      for (const c of cells) {
        const known = benefitsByFood.get(c.food_id);
        if (!known) { errors.push(`${day.day}/${meal}: unknown food_id ${c.food_id}`); continue; }
        for (const bid of c.benefit_ids) {
          if (!known.has(bid)) errors.push(`${day.day}/${meal}: benefit_id ${bid} not valid for food ${c.food_id}`);
        }
      }
    }
  }
  return errors.length ? { ok: false, errors } : { ok: true, plan: parsed.data };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/generation/validator.test.ts`
Expected: PASS (all adversarial cases gated).

- [ ] **Step 5: Commit**

```bash
git add src/lib/generation/validator.ts src/lib/generation/validator.test.ts
git commit -m "feat(generation): keystone validator rejecting any id outside candidate set"
```

---

### Task 11: Generation orchestrator (integration, mocked LLM)

**Files:**
- Create: `src/lib/generation/index.ts`, `src/lib/generation/index.test.ts`

**Interfaces:**
- Consumes: `deriveTargets` (Task 6), `generatePlanRaw` (Task 9, injected for tests), `validatePlan` (Task 10).
- Produces: `runGeneration(deps): Promise<{ status: "ok"; plan: Plan } | { status: "no_candidates" } | { status: "failed"; errors: string[] }>` where `deps = { candidates, profile, persona, targets, generate, maxRetries? }`. Loops generate→validate up to `maxRetries` (default 2); never returns an unvalidated plan.

- [ ] **Step 1: Write the failing integration tests**

```ts
// src/lib/generation/index.test.ts
import { expect, test, vi } from "vitest";
import { runGeneration } from "./index";
import type { Candidate, Plan } from "./prompt";

const candidates: Candidate[] = [
  { food_id: "f1", name: "Spinach", category: "leafy", benefits: [{ benefit_id: "b1", benefit: "iron", addresses: ["iron"] }] },
];
const goodPlan: Plan = { week: [{ day: "Monday", meals: { breakfast: [{ food_id: "f1", portion: "1 cup", benefit_ids: ["b1"] }], lunch: [], snack: [], dinner: [] } }] };
const hallucinated: Plan = { week: [{ day: "Monday", meals: { breakfast: [{ food_id: "GHOST", portion: "1", benefit_ids: [] }], lunch: [], snack: [], dinner: [] } }] };
const base = { candidates, profile: { conditions: [] }, persona: "therapeutic" as const, targets: ["iron"] };

test("returns ok on a valid first response", async () => {
  const generate = vi.fn().mockResolvedValue(goodPlan);
  const r = await runGeneration({ ...base, generate });
  expect(r.status).toBe("ok");
  expect(generate).toHaveBeenCalledTimes(1);
});

test("retries on hallucination, then succeeds", async () => {
  const generate = vi.fn().mockResolvedValueOnce(hallucinated).mockResolvedValueOnce(goodPlan);
  const r = await runGeneration({ ...base, generate, maxRetries: 2 });
  expect(r.status).toBe("ok");
  expect(generate).toHaveBeenCalledTimes(2);
});

test("fails (never returns bad plan) when all attempts hallucinate", async () => {
  const generate = vi.fn().mockResolvedValue(hallucinated);
  const r = await runGeneration({ ...base, generate, maxRetries: 2 });
  expect(r.status).toBe("failed");
});

test("short-circuits to no_candidates when candidate set is empty", async () => {
  const generate = vi.fn();
  const r = await runGeneration({ ...base, candidates: [], generate });
  expect(r.status).toBe("no_candidates");
  expect(generate).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/generation/index.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

```ts
// src/lib/generation/index.ts
import type { Persona } from "@/lib/db/types";
import { validatePlan } from "./validator";
import type { Candidate, Plan } from "./prompt";

export interface GenerationDeps {
  candidates: Candidate[];
  profile: { conditions: string[] };
  persona: Persona;
  targets: string[];
  generate: (candidates: Candidate[], profile: { conditions: string[] }, persona: Persona, targets: string[]) => Promise<Plan>;
  maxRetries?: number;
}

export type GenerationResult =
  | { status: "ok"; plan: Plan }
  | { status: "no_candidates" }
  | { status: "failed"; errors: string[] };

export async function runGeneration(deps: GenerationDeps): Promise<GenerationResult> {
  if (deps.candidates.length === 0) return { status: "no_candidates" };
  const attempts = (deps.maxRetries ?? 2) + 1;
  let lastErrors: string[] = [];
  for (let i = 0; i < attempts; i++) {
    const raw = await deps.generate(deps.candidates, deps.profile, deps.persona, deps.targets);
    const v = validatePlan(raw, deps.candidates);
    if (v.ok) return { status: "ok", plan: v.plan };
    lastErrors = v.errors;
  }
  return { status: "failed", errors: lastErrors };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/generation/index.test.ts`
Expected: PASS.

- [ ] **Step 5: Full suite + commit**

Run: `npm run test && npm run typecheck`
Expected: all green.

```bash
git add src/lib/generation/index.ts src/lib/generation/index.test.ts
git commit -m "feat(generation): orchestrator with bounded retry, honest failure, no unvalidated output"
```

---

### Task 12: Onboarding UI — disclaimer, profile+BMI, persona, markers

**Files:**
- Create: `app/onboarding/profile/page.tsx`, `app/onboarding/profile/actions.ts`
- Create: `app/onboarding/persona/page.tsx`, `app/onboarding/persona/actions.ts`
- Create: `app/onboarding/markers/page.tsx`, `app/onboarding/markers/actions.ts`
- Create: `src/components/DisclaimerGate.tsx`, `src/components/BmiBadge.tsx`, `src/components/DisclaimerBanner.tsx`

**Interfaces:**
- Consumes: `createServerSupabase` (Task 3), `computeBmi` (Task 4), `evaluateMarker`, `MARKER_DEFS` (Task 5).
- Produces: persisted `profiles` (incl. `disclaimer_ack_at`, `bmi`), `goal_profiles`, `blood_markers`; navigation profile → persona → markers → `/plan`.

- [ ] **Step 1: Disclaimer gate + profile form with server action**

```tsx
// src/components/DisclaimerGate.tsx
export function DisclaimerGate() {
  return (
    <p className="rounded bg-amber-50 p-3 text-sm text-amber-900">
      Educational and traditional wellness guidance only. FixMyMeal does not diagnose conditions or
      prescribe medicine. For abnormal results, consult a physician.
    </p>
  );
}
```

```tsx
// src/components/BmiBadge.tsx
export function BmiBadge({ bmi, category }: { bmi: number; category: string }) {
  return <span className="rounded bg-emerald-100 px-2 py-1 text-sm">BMI {bmi} · {category}</span>;
}
```

```tsx
// app/onboarding/profile/page.tsx
import { DisclaimerGate } from "@/components/DisclaimerGate";
import { saveProfile } from "./actions";

export default function ProfilePage() {
  return (
    <main className="mx-auto max-w-md space-y-4 p-6">
      <DisclaimerGate />
      <h1 className="text-lg font-semibold">Your health profile</h1>
      <form action={saveProfile} className="space-y-3">
        <input name="age" type="number" required placeholder="Age" className="w-full rounded border p-2" />
        <select name="sex" required className="w-full rounded border p-2">
          <option value="">Sex</option><option value="male">Male</option><option value="female">Female</option><option value="other">Other</option>
        </select>
        <input name="height_cm" type="number" step="0.1" required placeholder="Height (cm)" className="w-full rounded border p-2" />
        <input name="weight_kg" type="number" step="0.1" required placeholder="Weight (kg)" className="w-full rounded border p-2" />
        <select name="activity_level" required className="w-full rounded border p-2">
          <option value="">Activity level</option>
          <option value="sedentary">Sedentary</option><option value="light">Light</option>
          <option value="moderate">Moderate</option><option value="active">Active</option><option value="very_active">Very active</option>
        </select>
        <fieldset className="rounded border p-2">
          <legend className="text-sm">Conditions</legend>
          <label className="block"><input type="checkbox" name="conditions" value="acidity" /> Acidity</label>
        </fieldset>
        <label className="flex items-center gap-2 text-sm">
          <input type="checkbox" name="ack" required /> I understand this is not medical advice.
        </label>
        <button className="w-full rounded bg-emerald-700 p-2 text-white">Save & continue</button>
      </form>
    </main>
  );
}
```

```ts
// app/onboarding/profile/actions.ts
"use server";
import { redirect } from "next/navigation";
import { createServerSupabase } from "@/lib/supabase/server";
import { computeBmi } from "@/lib/bmi";

export async function saveProfile(formData: FormData) {
  const supabase = await createServerSupabase();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");
  const height = Number(formData.get("height_cm"));
  const weight = Number(formData.get("weight_kg"));
  const { bmi } = computeBmi(height, weight);
  await supabase.from("profiles").upsert({
    id: user!.id,
    age: Number(formData.get("age")),
    sex: String(formData.get("sex")),
    height_cm: height, weight_kg: weight,
    activity_level: String(formData.get("activity_level")),
    conditions: formData.getAll("conditions").map(String),
    bmi,
    disclaimer_ack_at: new Date().toISOString(),
    updated_at: new Date().toISOString(),
  });
  redirect("/onboarding/persona");
}
```

- [ ] **Step 2: Persona selector + action**

```tsx
// app/onboarding/persona/page.tsx
import { savePersona } from "./actions";
const OPTIONS = [
  { value: "therapeutic", label: "Therapeutic", desc: "Address markers & conditions" },
  { value: "fitness", label: "Fitness", desc: "Performance & body composition" },
  { value: "general", label: "General wellness", desc: "Balanced everyday eating" },
];
export default function PersonaPage() {
  return (
    <main className="mx-auto max-w-md space-y-4 p-6">
      <h1 className="text-lg font-semibold">Pick your focus</h1>
      <form action={savePersona} className="space-y-3">
        {OPTIONS.map((o) => (
          <label key={o.value} className="block rounded border p-3">
            <input type="radio" name="persona" value={o.value} required className="mr-2" />
            <span className="font-medium">{o.label}</span>
            <span className="block text-sm text-gray-600">{o.desc}</span>
          </label>
        ))}
        <button className="w-full rounded bg-emerald-700 p-2 text-white">Continue</button>
      </form>
    </main>
  );
}
```

```ts
// app/onboarding/persona/actions.ts
"use server";
import { redirect } from "next/navigation";
import { createServerSupabase } from "@/lib/supabase/server";

export async function savePersona(formData: FormData) {
  const supabase = await createServerSupabase();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");
  await supabase.from("goal_profiles").upsert(
    { user_id: user!.id, persona: String(formData.get("persona")) }, { onConflict: "user_id" });
  redirect("/onboarding/markers");
}
```

- [ ] **Step 3: Markers entry + flagging action**

```tsx
// app/onboarding/markers/page.tsx
import { DisclaimerBanner } from "@/components/DisclaimerBanner";
import { addMarker } from "./actions";
import { MARKER_DEFS } from "@/lib/markers";
import { createServerSupabase } from "@/lib/supabase/server";

export default async function MarkersPage() {
  const supabase = await createServerSupabase();
  const { data: markers } = await supabase.from("blood_markers").select("*");
  return (
    <main className="mx-auto max-w-md space-y-4 p-6">
      <DisclaimerBanner />
      <h1 className="text-lg font-semibold">Your markers (optional)</h1>
      <form action={addMarker} className="flex gap-2">
        <select name="marker_name" required className="rounded border p-2">
          {Object.keys(MARKER_DEFS).map((k) => <option key={k} value={k}>{k}</option>)}
        </select>
        <input name="value" type="number" step="0.1" required placeholder="Value" className="w-24 rounded border p-2" />
        <button className="rounded bg-emerald-700 px-3 text-white">Add</button>
      </form>
      <ul className="space-y-1">
        {(markers ?? []).map((m) => (
          <li key={m.id} className="flex justify-between rounded border p-2 text-sm">
            <span>{m.marker_name}: {m.value} {m.unit}</span>
            <span className={m.flag === "normal" ? "text-emerald-700" : m.severity === "severe" ? "text-red-700" : "text-amber-700"}>
              {m.flag}{m.severity !== "normal" ? ` · ${m.severity}` : ""}
            </span>
          </li>
        ))}
      </ul>
      <a href="/plan" className="block rounded bg-emerald-700 p-2 text-center text-white">Generate my plan</a>
    </main>
  );
}
```

```ts
// app/onboarding/markers/actions.ts
"use server";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { createServerSupabase } from "@/lib/supabase/server";
import { evaluateMarker, MARKER_DEFS } from "@/lib/markers";

export async function addMarker(formData: FormData) {
  const supabase = await createServerSupabase();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");
  const name = String(formData.get("marker_name"));
  const value = Number(formData.get("value"));
  const def = MARKER_DEFS[name];
  const { flag, severity } = evaluateMarker(name, value);
  await supabase.from("blood_markers").insert({
    user_id: user!.id, marker_name: name, value, unit: def.unit,
    ref_low: def.refLow, ref_high: def.refHigh, flag, severity, source: "manual",
  });
  revalidatePath("/onboarding/markers");
}
```

```tsx
// src/components/DisclaimerBanner.tsx
export function DisclaimerBanner() {
  return (
    <div className="sticky top-0 z-10 bg-amber-100 px-3 py-1 text-center text-xs text-amber-900">
      Educational guidance only — not medical advice. Consult a physician for abnormal results.
    </div>
  );
}
```

- [ ] **Step 4: Typecheck & browser-verify the flow**

Run: `npm run typecheck`
Then with the dev-browser skill: sign up → profile shows BMI persisted → persona saved → add a normal and a severe marker, confirm green vs red flagging and that rows persist (Supabase MCP `execute_sql` on `blood_markers`).

- [ ] **Step 5: Commit**

```bash
git add app/onboarding src/components/DisclaimerGate.tsx src/components/BmiBadge.tsx src/components/DisclaimerBanner.tsx
git commit -m "feat(onboarding): disclaimer gate, profile+BMI, persona, marker entry with flagging"
```

---

### Task 13: Plan view — generate action, grid, sourced card, abnormal interstitial

**Files:**
- Create: `app/plan/page.tsx`, `app/plan/actions.ts`
- Create: `src/lib/generation/candidates.ts` (DB query: targets → candidates + a hydration helper)
- Create: `src/components/PlanGrid.tsx`, `src/components/FoodCard.tsx`, `src/components/AbnormalInterstitial.tsx`

**Interfaces:**
- Consumes: everything from Tasks 4–11; `createServerSupabase`.
- Produces: `fetchCandidates(supabase, targets): Promise<Candidate[]>`; a `generatePlan` server action that runs eligibility → fetch candidates → `runGeneration` (injecting `generatePlanRaw`) → persist; the rendered grid + tappable sourced cards hydrated from KB rows; the severe-marker interstitial gating generation.

- [ ] **Step 1: Candidate fetch + hydration helpers**

```ts
// src/lib/generation/candidates.ts
import type { SupabaseClient } from "@supabase/supabase-js";
import type { Candidate } from "./prompt";

export async function fetchCandidates(supabase: SupabaseClient, targets: string[]): Promise<Candidate[]> {
  if (targets.length === 0) return [];
  const { data: benefits } = await supabase
    .from("food_benefits")
    .select("id, food_id, addresses, foods(id, name, category)")
    .overlaps("addresses", targets);
  const byFood = new Map<string, Candidate>();
  for (const b of benefits ?? []) {
    const food = (b as any).foods;
    if (!byFood.has(food.id)) byFood.set(food.id, { food_id: food.id, name: food.name, category: food.category, benefits: [] });
    byFood.get(food.id)!.benefits.push({ benefit_id: (b as any).id, benefit: "", addresses: (b as any).addresses });
  }
  return [...byFood.values()];
}
```

- [ ] **Step 2: Generate server action (eligibility → candidates → orchestrate → persist)**

```ts
// app/plan/actions.ts
"use server";
import { redirect } from "next/navigation";
import { revalidatePath } from "next/cache";
import { createServerSupabase } from "@/lib/supabase/server";
import { deriveTargets } from "@/lib/eligibility";
import { fetchCandidates } from "@/lib/generation/candidates";
import { runGeneration } from "@/lib/generation";
import { generatePlanRaw } from "@/lib/generation/client";
import type { Persona } from "@/lib/db/types";

export async function generatePlan() {
  const supabase = await createServerSupabase();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");

  const [{ data: profile }, { data: goal }, { data: markers }] = await Promise.all([
    supabase.from("profiles").select("conditions").single(),
    supabase.from("goal_profiles").select("persona").single(),
    supabase.from("blood_markers").select("*"),
  ]);

  const persona = (goal?.persona ?? "general") as Persona;
  const targets = deriveTargets(markers ?? [], profile?.conditions ?? [], persona);
  const candidates = await fetchCandidates(supabase, targets);

  const result = await runGeneration({
    candidates, profile: { conditions: profile?.conditions ?? [] }, persona, targets,
    generate: generatePlanRaw,
  });

  if (result.status === "ok") {
    await supabase.from("plans").insert({
      user_id: user!.id, persona, plan_json: result.plan, markers_snapshot: markers ?? [], status: "active",
    });
  }
  revalidatePath("/plan");
}
```

- [ ] **Step 3: Grid + card + interstitial + plan page**

```tsx
// src/components/FoodCard.tsx
export interface FoodCardData {
  name: string; benefit: string; mechanism: string; serving_guidance: string; frequency: string;
  research?: { strength: string; citation_text: string; source_url: string | null };
  ayurveda?: { strength: string; citation_text: string; reference: string | null };
}
export function FoodCard({ d }: { d: FoodCardData }) {
  return (
    <div className="space-y-2 rounded border p-3 text-sm">
      <h3 className="font-semibold">{d.name}</h3>
      <p>{d.benefit}</p>
      <p className="text-gray-600">{d.mechanism}</p>
      <p className="text-gray-600">{d.serving_guidance} · {d.frequency}</p>
      {d.research && (
        <p className="rounded bg-blue-50 p-1">🔬 Research ({d.research.strength}): {d.research.citation_text}{" "}
          {d.research.source_url && <a className="underline" href={d.research.source_url}>source</a>}</p>
      )}
      {d.ayurveda && (
        <p className="rounded bg-orange-50 p-1">🪔 Ayurveda ({d.ayurveda.strength}): {d.ayurveda.citation_text}
          {d.ayurveda.reference ? ` — ${d.ayurveda.reference}` : ""}</p>
      )}
    </div>
  );
}
```

```tsx
// src/components/PlanGrid.tsx
import type { Plan } from "@/lib/generation/prompt";
export function PlanGrid({ plan, nameById }: { plan: Plan; nameById: Record<string, string> }) {
  const meals = ["breakfast", "lunch", "snack", "dinner"] as const;
  return (
    <div className="overflow-x-auto">
      <table className="min-w-full border text-sm">
        <thead><tr><th className="border p-1">Day</th>{meals.map((m) => <th key={m} className="border p-1 capitalize">{m}</th>)}</tr></thead>
        <tbody>
          {plan.week.map((day) => (
            <tr key={day.day}>
              <td className="border p-1 font-medium">{day.day}</td>
              {meals.map((m) => (
                <td key={m} className="border p-1">
                  {day.meals[m].map((c, i) => (
                    <span key={i} className="mr-1 inline-block rounded bg-emerald-50 px-1">
                      {nameById[c.food_id] ?? "?"} <span className="text-gray-500">({c.portion})</span>
                    </span>
                  ))}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

```tsx
// src/components/AbnormalInterstitial.tsx
export function AbnormalInterstitial({ markers }: { markers: { marker_name: string }[] }) {
  return (
    <div className="rounded border border-red-300 bg-red-50 p-4 text-sm text-red-900">
      <h2 className="font-semibold">Please consult a physician</h2>
      <p>Some of your markers are significantly out of range ({markers.map((m) => m.marker_name).join(", ")}).
        We can still show general, non-prescriptive guidance, but this is not a substitute for medical care.</p>
    </div>
  );
}
```

```tsx
// app/plan/page.tsx
import { DisclaimerBanner } from "@/components/DisclaimerBanner";
import { AbnormalInterstitial } from "@/components/AbnormalInterstitial";
import { PlanGrid } from "@/components/PlanGrid";
import { generatePlan } from "./actions";
import { createServerSupabase } from "@/lib/supabase/server";
import type { Plan } from "@/lib/generation/prompt";

export default async function PlanPage() {
  const supabase = await createServerSupabase();
  const [{ data: markers }, { data: latest }, { data: foods }] = await Promise.all([
    supabase.from("blood_markers").select("marker_name, severity"),
    supabase.from("plans").select("plan_json").order("created_at", { ascending: false }).limit(1).maybeSingle(),
    supabase.from("foods").select("id, name"),
  ]);
  const severe = (markers ?? []).filter((m) => m.severity === "severe");
  const nameById = Object.fromEntries((foods ?? []).map((f) => [f.id, f.name]));
  const plan = latest?.plan_json as Plan | undefined;

  return (
    <main className="mx-auto max-w-2xl space-y-4 p-4">
      <DisclaimerBanner />
      {severe.length > 0 && <AbnormalInterstitial markers={severe} />}
      <form action={generatePlan}><button className="rounded bg-emerald-700 px-4 py-2 text-white">
        {plan ? "Regenerate plan" : "Generate plan"}
      </button></form>
      {plan ? <PlanGrid plan={plan} nameById={nameById} />
            : <p className="text-sm text-gray-600">No plan yet. Tap generate.</p>}
    </main>
  );
}
```

> Card hydration: tapping a food chip should open a `FoodCard` populated from `food_benefits` + `evidence` for that `food_id` (a server component or route that joins `food_benefits`→`evidence`). Wire the chip to fetch and render `FoodCard` for the selected `food_id`; the card data comes entirely from those rows.

- [ ] **Step 4: Typecheck, full suite, browser-verify end to end**

Run: `npm run typecheck && npm run test`
Then with the dev-browser skill: complete profile → persona → add a low-iron marker → `/plan` → generate → confirm a grid renders with foods from the KB, a card shows a real Research citation link, and a severe marker shows the interstitial. Time the profile→first-plan path (<3 min target).

- [ ] **Step 5: Commit**

```bash
git add app/plan src/lib/generation/candidates.ts src/components/PlanGrid.tsx src/components/FoodCard.tsx src/components/AbnormalInterstitial.tsx
git commit -m "feat(plan): generate action, sourced grid + cards, abnormal-marker interstitial"
```

---

### Task 14: Safety audit tests & final verification

**Files:**
- Create: `src/lib/generation/safety.test.ts`
- Create: `src/lib/generation/audit.ts`

**Interfaces:**
- Produces: `containsMedicalLanguage(text: string): boolean` (scans for medicine names / dosage patterns); tests asserting (a) no validated plan references a food outside candidates, (b) severe markers always route to the interstitial path, (c) rendered plan/card text carries zero medicine names or treatment dosages.

- [ ] **Step 1: Write the audit helper + failing tests**

```ts
// src/lib/generation/audit.ts
const DOSAGE = /\b\d+\s?(mg|mcg|µg|iu|ml|tablet|capsule|injection)\b/i;
const MEDICINE_HINTS = /\b(tablet|capsule|injection|prescribe|prescription|antibiotic|metformin|insulin)\b/i;
export function containsMedicalLanguage(text: string): boolean {
  return DOSAGE.test(text) || MEDICINE_HINTS.test(text);
}
```

```ts
// src/lib/generation/safety.test.ts
import { expect, test } from "vitest";
import { containsMedicalLanguage } from "./audit";
import { validatePlan } from "./validator";
import { SYSTEM_PROMPT } from "./prompt";
import type { Candidate } from "./prompt";

test("audit flags dosage and medicine language", () => {
  expect(containsMedicalLanguage("take 500 mg twice daily")).toBe(true);
  expect(containsMedicalLanguage("prescribe metformin")).toBe(true);
  expect(containsMedicalLanguage("1 cup cooked spinach, 3 times a week")).toBe(false);
});

test("system prompt itself carries no dosage instructions", () => {
  expect(containsMedicalLanguage(SYSTEM_PROMPT)).toBe(false);
});

test("validator guarantees no out-of-candidate food survives", () => {
  const candidates: Candidate[] = [{ food_id: "f1", name: "Spinach", category: null, benefits: [] }];
  const r = validatePlan(
    { week: [{ day: "Monday", meals: { breakfast: [{ food_id: "f2", portion: "1", benefit_ids: [] }], lunch: [], snack: [], dinner: [] } }] },
    candidates);
  expect(r.ok).toBe(false);
});
```

- [ ] **Step 2: Run to verify it fails then passes**

Run: `npx vitest run src/lib/generation/safety.test.ts`
Expected: FAIL (audit missing) → after Step 1's `audit.ts` exists, PASS.

- [ ] **Step 3: Full suite + typecheck**

Run: `npm run test && npm run typecheck && npm run build`
Expected: all tests pass; production build succeeds.

- [ ] **Step 4: Browser smoke of the safety routing**

With the dev-browser skill: enter a severe marker (e.g. hemoglobin 7) and confirm the interstitial appears before any plan; enter only normal markers and confirm a plan generates; inspect a generated plan + open cards and confirm no medicine names/dosages appear.

- [ ] **Step 5: Commit**

```bash
git add src/lib/generation/safety.test.ts src/lib/generation/audit.ts
git commit -m "test(safety): no-fabrication, abnormal-routing, and no-medicine-language audits"
```

---

## Done criteria (slice complete)

- profile → first plan reachable in under 3 minutes in-browser.
- 100% of rendered cards cite KB-backed sources; validator + safety tests prove zero fabricated ids and zero medicine/dosage language.
- Severe markers always route to the physician interstitial before a plan.
- RLS verified: a second user cannot read the first user's profile/markers/plans (manual cross-account check via Supabase MCP).
- `npm run test`, `npm run typecheck`, and `npm run build` all green.
- KB `VERIFY.md` signed off by a human before any real-user launch.
