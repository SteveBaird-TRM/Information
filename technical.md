# TRM Web Apps — Technical Brief

Scope: `admin`, `comparison`, `devops-visualiser`, `intake`, `project-list`, `roadmap`, `schedule-a`, `shared`.

## 1\. Architecture

* **No monorepo, no framework, no build step.** There is no root `package.json`, no workspaces config, no bundler. Each app folder is a self-contained set of plain `.html`/`.js`/`.css` files. The Supabase JS client is loaded via CDN `<script>` tag as a UMD global (`window.supabase.createClient(...)`).
* **Pure BaaS pattern.** Apps talk directly to Supabase Postgres from the browser (`sbClient.from(table)...`). There is no custom backend/API server for CRUD. Authorization is enforced entirely by Postgres Row Level Security (RLS), not by any server-side app code.
* **One exception**: a Supabase **Edge Function** (`admin-users`, under the excluded `supabase/functions/` folder) handles privileged `auth.users` administration (create/delete/reset-password) using the service-role key. This is the only place a service-role key is used, and it runs server-side (Deno), never shipped to a browser.
* **Config is hardcoded, not environment-driven.** There are no `.env`/`.env.example` files. `SUPABASE\_URL` and the publishable/anon key are literal strings inside every copy of `auth-gate.js`:

```
  SUPABASE\_URL = 'https://aczahhxneshsrnqsezzg.supabase.co'
  SUPABASE\_KEY = 'sb\_publishable\_...'   (public anon-equivalent key — safe to expose client-side by design)
  ```

* **Code sharing is duplication, not packaging.** `auth-gate.js` is an identical file, copy-pasted into every gated app rather than centralized — a deliberate choice noted in the file's own header comment, since there's no shared build step to import from. `shared/project-picker.js` is the one genuine shared module, consumed via `<script src="/shared/project-picker.js">` by `intake` and `roadmap` only.
* **Hosting.** Hosting is on vercel.com via GitHub.

## 2\. Database schema

Supabase project ref: `aczahhxneshsrnqsezzg`.

|Table|Key columns|Purpose|Used by|
|-|-|-|-|
|**projects**|`id uuid PK`, `canonical\_name text`, `origin\_app text`, `created\_by uuid → auth.users(id) on delete set null`, `status text` (`open`/`delivered`/`rejected`), `pm text`, `ba text`, `sme text`, `description text`, `created\_at`, `updated\_at`|Canonical cross-app project registry. `pm`/`ba`/`sme` are **intentionally free text**, not `auth.users` references — a project can have more than one person in a role, so this was a deliberate design choice, not an oversight.|intake, roadmap, schedule-a, project-list, comparison, admin|
|**project\_access**|`user\_id`, `project\_key` (`intake` / `roadmap-db` / `schedule-a-db-v2` / `implementation-forum`), `role` (`viewer`/`editor`)|Central per-user, per-app access control table. Every `auth-gate.js` and almost every RLS policy consults this.|all gated apps, admin|
|**roadmap\_tasks**|`id`, `name`, `start\_date`, `duration\_weeks`, `sort\_order`, `color`, `team`, `phase`, `health`, `project\_id uuid → projects(id) on delete set null`|Roadmap's Gantt task list.|roadmap, project-list, comparison, admin|
|**intake\_cards**|`id`, `data jsonb`, `project\_id uuid → projects(id) on delete set null`|Intake pipeline cards (free-form fields live in the `data` JSON blob).|intake, roadmap/releases.html, admin|
|**intake\_columns**|`id`, `data jsonb`|Intake board's column/lane config.|intake|
|**resource\_allocations**|`id integer PK` (legacy numeric ids), `resource\_name`, `project\_label text`, `project\_id uuid → projects(id)`, `start\_week`, `duration\_weeks`, `allocation\_pct`, `sort\_order`, `created\_at`, `updated\_at`|Schedule-A's per-resource allocation rows. Replaced a single JSON blob on 2026‑08‑26.|schedule-a, comparison, admin|
|**schedule\_settings**|`id integer PK default 1` (singleton, `check (id = 1)`), `timeline\_start date`, `next\_id integer`, `roles\_catalog jsonb`, `resource\_roles jsonb`, `updated\_at`|Schedule-A's global settings.|schedule-a|
|**gantt\_state**|`id`, `data jsonb`|**Legacy**, superseded by `resource\_allocations` + `schedule\_settings`. No longer queried directly by `schedule-a`; may still exist in the live DB.|(deprecated)|

`devops-visualiser` has no database connection at all — it's a fully offline, client-side CSV visualizer.

**Cross-app data flow to note:** `roadmap/releases.html` reads `intake\_cards` directly to pull the original requestor's name for each milestone — this is the one place a "downstream" app reaches back into an "upstream" app's table rather than going through `projects`.

## 3\. Row Level Security (RLS)

RLS is enabled on every table listed above. Findings from the tracked migrations, corroborated by an in-repo security review (`security check.txt` at the repo root — not excluded, worth reading in full):

* **`is\_app\_admin()`** — a `security definer` SQL function that returns true only if the caller's JWT email matches one hardcoded admin address. This is the sole "super admin" check used throughout the schema (governs `projects` delete, and full read/write/delete on `project\_access`).
* **`has\_project\_access(project\_key, min\_role)`** — the workhorse function behind almost every other policy (not present in tracked migrations — predates tracking, confirm its current definition against the live database).
* **`project\_access`**: self-read (`user\_id = auth.uid()`) plus admin-only read-all/insert/update/delete, gated on `is\_app\_admin()`.
* **`projects`**: select — any user with viewer+ access to `intake`, `roadmap-db`, or `schedule-a-db-v2`; insert — editors of any of those three keys; update — editors of any of the three; delete — admin only.
* **`resource\_allocations` / `schedule\_settings`**: select requires `schedule-a-db-v2` viewer; insert/update/delete require `schedule-a-db-v2` editor.
* **`roadmap\_tasks` / `intake\_cards` / `intake\_columns`**: standard `has\_project\_access()`-based read/write policies scoped to their respective project keys (a redundant, inconsistent policy set on `intake\_columns` was cleaned up in a later migration to match this pattern).
* The prior security review states explicitly: **"no wide-open `using(true)` policies anywhere."**
* **A recurring foot-gun in this codebase**: RLS policies being correct is not sufficient — the underlying Postgres `GRANT` to the `authenticated`/`service\_role` roles must also exist, or you get "permission denied" even with a correct policy. Two migrations exist purely to fix missing base grants after RLS was already right (`project\_access`, `projects` delete). Worth remembering if you hit unexplained permission errors while extending the schema.

## 4\. Authentication

* Every gated app loads its own copy of **`auth-gate.js`**, which:

  1. Creates one Supabase client (`window.sbClient`).
  2. Uses `sbClient.auth.signInWithPassword({ email, password })` — email/password auth only, no OAuth/SSO/magic-link in evidence.
  3. Reads a page-level `window.AUTH\_REQUIREMENTS` array (e.g. `\[{ project: 'roadmap-db', role: 'viewer' }]`), fetches the signed-in user's rows from `project\_access`, and shows a login/blocked overlay if requirements aren't met.
* **This client-side gate is UX only — real enforcement is RLS.** Every query independently re-checks `project\_access` via `has\_project\_access()`/`is\_app\_admin()` at the database layer, regardless of what the client-side overlay shows.
* Per-page access requirements:

|App / page|Required access|
|-|-|
|roadmap (`index.html`, `releases.html`)|`roadmap-db` viewer (editor to persist changes)|
|schedule-a|`schedule-a-db-v2` viewer (editor to persist changes)|
|intake (`index.html`, dashboards)|`intake` viewer|
|intake `admin.html`|`intake` editor|
|project-list (`index.html`, `view.html`)|`roadmap-db` viewer (no key of its own — piggybacks on roadmap)|
|comparison|`roadmap-db` viewer **and** `schedule-a-db-v2` viewer|
|admin|any authenticated user can load the shell; every actual action is additionally gated server-side by `is\_app\_admin()` and by the hardcoded admin email|
|devops-visualiser|none — no login|

* **Privileged user management** (`supabase/functions/admin-users/index.ts`, excluded folder but worth knowing about) is an Edge Function running under the service-role key. It never trusts the client's claimed identity — it independently validates the caller's bearer token via `admin.auth.getUser(token)` and re-checks the resulting email against the hardcoded admin address on every request.

## 5\. Security notes and open items

* **No client-side service-role usage found.** A full-tree search for `service\_role` only turns up server-side GRANTs (SQL) and the Edge Function's Deno environment variable — never shipped to the browser. No issue here.
* **CORS** on the `admin-users` Edge Function is `Access-Control-Allow-Origin: \*`. Assessed as low-risk since every request still requires a valid bearer token and independent identity re-check, but could be tightened to the app's real domain for defense in depth.
* **Outstanding recommendations from the last security review, not yet applied: (only available in a paid plan)**

  * Enable Supabase Auth's "leaked password protection" (HaveIBeenPwned check) — a dashboard toggle, no code change needed.
  * Harden the `set\_updated\_at()` trigger function with `SET search\_path = public, pg\_temp` (low real-world risk today, since it uses no unqualified identifiers, but it's the standard hardening for `security definer`-adjacent functions).
* **The public anon/publishable key being hardcoded and identical across every app is expected and fine** — it's a public key by design, and the whole security model deliberately rests on RLS + `project\_access`, not key secrecy.

## 6\. Quick reference: tables ↔ project keys

|project\_key|Tables it governs|
|-|-|
|`intake`|`intake\_cards`, `intake\_columns`|
|`roadmap-db`|`roadmap\_tasks`|
|`schedule-a-db-v2`|`resource\_allocations`, `schedule\_settings`|
|*(none — admin only)*|`project\_access`, `projects` delete|
|*(shared, multi-key read)*|`projects` (readable by viewers of any of the three keys above)|

See `overview.md` for a plain-language walkthrough of what each app does and how they fit together.

