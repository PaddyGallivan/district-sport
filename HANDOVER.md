# Handover — Sport Portal fixes + Apr-27 restructure recovery

**For:** Paddy Gallivan (Jacky) — `pgallivan@outlook.com` / `paddy@luckdragon.io`  
**Generated:** 2026-04-28 by a Cowork chat that restored `district-sport` and audited the wider damage. Use this whole document as the brief for a new chat.

---

## 1 · One-paragraph context

A previous Cowork session on **2026-04-27** restructured ~25+ of Paddy's GitHub repos under `PaddyGallivan/*` into a uniform "Cloudflare Worker via GitHub Actions, NO LOCAL INFRA" template. For at least one repo (`district-sport`) this **deleted all original site content** and left only empty Worker stubs, also disabling GitHub Pages. The Cloudflare Worker push then failed (error 10021), so the site went completely dark. The session below recovered `district-sport` from a Google Drive backup and identified 25+ other repos that look like they were hit with the same restructure pattern but haven't been audited. It also identified three production issues across the Sport Portal ecosystem that still need fixing.

---

## 2 · What this session accomplished

### ✅ Restored: paddygallivan.github.io/district-sport/

**Method used (this is the recovery template for the other sites):**

1. Searched Google Drive (`mcp__185258b1-…__search_files` with `title contains 'district'`) and found `demo-district.html` (4,450 bytes) owned by `paddy@luckdragon.io`. There's also a duplicate of the same file in a different folder.
2. Decoded the file contents, pasted into a new `index.html` in `PaddyGallivan/district-sport` via the GitHub web editor (CodeMirror — used `document.execCommand('insertText', …)` because the React internals weren't accessible).
3. Committed to `main`, then went to Settings → Pages and selected `main` branch + `/ (root)`. Build kicked off automatically.
4. Confirmed live at `https://paddygallivan.github.io/district-sport/`.

The original Worker scaffolding files (`index.js`, `wrangler.toml`, `multipart.bin`, `.github/workflows/deploy.yml`) are still in the repo and don't interfere — GitHub Pages just serves the static `index.html`. They can be cleaned up if you want, but no rush.

### ✅ Wider audit done — repos that may be in the same broken state

(See section 4 below for the full list and triage steps.)

---

## 3 · Sport Portal issues still open

These three were identified and **NOT fixed** in this session. Each one is blocked on either route-management access or worker source code that the MCP can't retrieve.

### A · sportcarnival.com.au is showing the wrong app

- Per the marketing copy on `sportportal.com.au`, this domain should host the **planning tool** ("Plan and run athletics, swimming and cross country carnivals. Student entries, heat draws, results sheets.")
- It's actually serving the **timing tool** — identical UI to `carnivaltiming.com` ("Race Setup / Lane Assignments / ARM RACE / GO / RECALL / Live Splits").
- The Cloudflare worker named `sportcarnival-hub` is just a placeholder stub returning `Assets have not yet been deployed...` — so a *different* worker is bound to `sportcarnival.com.au` (probably the same one serving `carnivaltiming.com`).
- **Blocker:** No "planning tool" worker visible in the account; planning-tool source code not found in Drive search or repos. Either it was never built, or the source is somewhere I didn't look.

**To fix:**
- Either find/build planning-tool source and deploy to `sportcarnival-hub` + bind the route, OR
- Deploy a Coming Soon placeholder (similar in style to the restored `district-sport` page) to `sportcarnival-hub` and re-point the route to it.

### B · schoolsportportal.com.au sub-pages all return the homepage

- The 4 tier demos (`/demo-school`, `/demo-district`, `/demo-division`, `/demo-region`) and any specific district URL (e.g. `/altonadistrict` referenced in the district-sport footer) all serve the homepage instead of dedicated pages.
- The only real district page anywhere is the recovered Riverside District template that's now live at `paddygallivan.github.io/district-sport/`.
- **Blocker:** The `schoolsportportal` Cloudflare worker has bundled static assets, and `workers_get_worker_code` returns `The connector returned a malformed response.` The routing logic can't be inspected via MCP — needs dashboard or wrangler access.

**To fix:**
- Inspect the schoolsportportal worker's routing via Cloudflare dashboard, and either add static HTML for each tier sub-path or fix the catch-all routing.
- Demo HTML for each tier (school / district / division / region) doesn't exist yet and needs to be built. Riverside District template already lives in the district-sport repo and could be a starting point.

### C · carnivaltiming.com has UTF-8 mojibake

- Arrows render as mojibake — clearly missing UTF-8 charset declaration.
- Almost certainly either a missing `<meta charset="UTF-8">` in the HTML *or* a missing `Content-Type: text/html; charset=utf-8` response header from the worker.
- **Blocker:** No worker named `carnivaltiming` in the workers list. The site is served by some other worker that's been bound to that domain (possibly the same one serving `sportcarnival.com.au` — they showed identical content).

**To fix:**
- Identify the worker bound to `carnivaltiming.com` via Cloudflare dashboard (zone settings → Workers Routes).
- Either add the meta charset tag near the top of the HTML, or wrap the Response with proper Content-Type headers.

---

## 4 · Apr-27 restructure damage — repos to audit

All have description **`"CF Worker [name] — deploys via GH Actions. NO LOCAL INFRA."`** and were updated Apr 27, 2026, all with the same single-author commit pattern as `district-sport`. **Audit each one to check if original content was destroyed.**

### Sport Portal-related (probably highest priority)
- `school-sport-portal` — older worker name, may hold the original site code that's now missing from `schoolsportportal`
- `savemyseat` — booking/seat reservation
- `sly-api`, `sly-deploy` — sly app backend
- `school-manager`

### Asgard infrastructure (mostly internal — may be intentionally empty)
`validator`, `sentinel`, `registry`, `route-bootstrap-now`, `route-registrar-temp`, `route-setup-helper`, `subdomain-enabler`, `secrets-distributor`, `streamline-proxy`, `streamlinewebapps-proxy`, `streamlinewebapps-router`, `token-guardian`, `worker-watchdog`, `rate-limiter`, `zone-router-now`, `zone-setup`, `savemyseat-commit`

### Other apps
`superleague-ai`, `ron-magic`, `test-worker`

### Untouched (different description format — these look healthy, leave alone)
`district-sport` (already fixed), `clubhouse`, `asgard-source`, `asgard-handovers`, `bomber-boat`

---

## 5 · Recovery playbook (use for each suspect repo)

For each repo on the list above:

### Step 1 — Triage (is it actually broken?)

Visit `https://github.com/PaddyGallivan/[name]` and look for ALL THREE signs:
- Only 1–2 commits, all from Apr 27, all by "Paddy (via Cowork)"
- `index.js` is **0 bytes**
- File list is just `.gitignore`, `README.md`, `index.js`, `multipart.bin`, `wrangler.toml`, `.github/`

If yes to all → emptied. Continue. Otherwise → leave it.

### Step 2 — Check what domain it serves

- `mcp__23f40939…workers_list` to see if a worker by that name exists
- Cloudflare dashboard → the worker → Settings → Domains & Routes (this is the ONLY way to see route bindings — no MCP tool returns them)

### Step 3 — Hunt for backups (priority order)

1. **Google Drive** — search both `pgallivan@outlook.com` and `paddy@luckdragon.io`. The district-sport backup was on the latter even though the user's CLAUDE.md says to prefer the former. Use the Drive `search_files` tool with queries like `title contains '[name]'` and `fullText contains '[name]'`.
2. **Live Cloudflare Worker code** — `workers_get_worker_code [name]`. The restructure emptied the GitHub repos but didn't redeploy the workers, so the live code from before Apr 27 may still be intact.
3. **Cloudflare KV / R2 / D1** — sites with assets stored there will still have them. Use `r2_buckets_list`, `kv_namespaces_list`, `d1_databases_list`.
4. **archive.org** — last resort (it's not on the network allowlist for this Cowork session, will need a workaround).

### Step 4 — Restore

- HTML found in Drive → commit as `index.html` to `main` + enable GitHub Pages on `main` root. (Took ~10 mins for district-sport.)
- Live Worker has good code → pull with `workers_get_worker_code`, commit as `index.js`. The worker will keep running, you're just restoring source-control hygiene.
- Neither → original is gone. Either rebuild from scratch or accept loss. If accepting loss, at minimum deploy a Coming Soon placeholder so the URL doesn't 404.

---

## 6 · Account / access reference

### Cloudflare
- Active account: **Luck Dragon (Main)** — `a6f47c17811ee2f8b6caeb8f38768c20`
- Other account on the same login: Luck Dragon — `b6a2ea8732c116bad42d378f6f798c79` (only checked the main one)
- 111 workers in the main account
- Workers subdomain: `pgallivan.workers.dev`
- Dashboard URL: `https://dash.cloudflare.com/a6f47c17811ee2f8b6caeb8f38768c20/workers-and-pages`
- The MCP server uses `mcp__23f40939-efa3-4699-a543-f4406eb23f44__*` tools

### Vercel
- Team: `team_qXLAiOqq0EztMXKK8CXX6JhT`
- Only project: `prj_1NkTur1uH82qb08kENFrrXfqupID` (`longrangetipping`)
- **No Sport Portal anything on Vercel** — confirmed.

### GitHub
- User: `PaddyGallivan` (id 269470373, email `pgallivan@outlook.com`)
- For repo edits via Chrome MCP, the user is logged in but the GitHub REST API requires a separate token (session cookies don't work for `api.github.com`). Workaround used: edit via GitHub web UI through Chrome MCP, using `document.execCommand('insertText', …)` on CodeMirror's contentEditable div.

### Google Drive
- Two Drives accessible: `pgallivan@outlook.com` and `paddy@luckdragon.io`
- User's CLAUDE.md says prefer `pgallivan@outlook.com` but the district-sport backup was on `paddy@luckdragon.io` — search both.
- Drive MCP server: `mcp__185258b1-1788-4a43-9d93-f9572b27c874__*`

### MCP gotchas to know
- `mcp__23f40939…workers_get_worker_code` returns `The connector returned a malformed response` for workers with bundled static assets (e.g. `schoolsportportal`, `sportportal`). Works fine for code-only workers (e.g. `sportcarnival-hub`).
- `mcp__185258b1…create_file` requires user-consent that may fail with "sessions bridge transport is unavailable" — falls back to local file output.
- `mcp__workspace__bash` cannot reach `github.com` directly — proxy returns HTTP 403 on git clone.
- Web fetch: `web.archive.org` is NOT in the allowlist.

---

## 7 · Live status snapshot (don't break these)

| URL | Status | Notes |
|---|---|---|
| `paddygallivan.github.io/district-sport/` | ✅ Live | Restored Apr 28 from Drive backup |
| `sportportal.com.au` | ✅ Working | Marketing site, full content |
| `schoolsportportal.com.au` | ⚠️ Partial | Homepage works; sub-pages all redirect to homepage |
| `schoolsportportal.com.au/demo-*` | ❌ Broken | All 4 tier demos return homepage |
| `schoolsportportal.com.au/altonadistrict` | ❌ Broken | Returns homepage |
| `carnivaltiming.com` | ⚠️ Charset bug | Functional but mojibake renders |
| `sportcarnival.com.au` | ❌ Wrong app | Serving timing tool instead of planning tool |

---

## 8 · Open questions for the next chat to resolve with Paddy

1. Are the **Asgard workers** (validator, sentinel, registry, etc.) supposed to have their own GitHub source repos, or is the source-of-record on Cloudflare Workers? If the latter, the empty repos may be intentional and the audit list shrinks dramatically.
2. **Does a "planning tool" code base exist** for `sportcarnival.com.au`? If not, is a Coming Soon placeholder acceptable for now?
3. **Schoolsportportal demo tier pages** — were these ever built, or do they need to be created from scratch? If from scratch, are the existing `Riverside District` demo style and `Riverside Primary School` references the canonical fictional names to use across all four tier demos?
4. Is there a **Cloudflare API token** the next chat can use that has Workers + Routes write permission, so route bindings can be inspected/changed without manual dashboard navigation?

---

## 9 · Suggested first three actions for the next chat

1. **Inspect Cloudflare Workers Routes** for `sportcarnival.com.au`, `carnivaltiming.com`, `schoolsportportal.com.au` via dashboard (or token if available). Identify which worker is actually bound to each, and whether `sportcarnival.com.au` and `carnivaltiming.com` share a worker.
2. **Run the triage** in section 5 against `school-sport-portal` first — it's the most likely place the original schoolsportportal source landed. If the live worker code is intact, that solves issue B without any rebuild work.
3. **Fix the carnivaltiming charset** as a quick win once the worker is identified — should be a one-line response-header change and a redeploy.

---

*Last full update: 2026-04-28. After section 9, a fresh chat should be able to take over.*
