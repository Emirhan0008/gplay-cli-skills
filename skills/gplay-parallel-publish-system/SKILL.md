---
name: gplay-parallel-publish-system
description: Dual-track Google Play publishing SYSTEM that advances the gplay CLI and Playwright web-console automation in parallel as two independent tracks with explicit sync points. Orchestrates preflight, validate, metadata, listings, testers and releases over the CLI while simultaneously driving app creation, the ten app-content declarations and store graphics through browser automation. Includes the orchestrator/worker delegation model for AI agents, a merged gotcha catalog from real publishing sessions, and safety gates. Use when publishing an Android app end-to-end and you want maximum throughput instead of sequential hand-offs.
---

# gplay-parallel-publish-system — dual-track Play publishing

Publishing a Google Play app is ONE goal served by TWO disjoint tool surfaces.
Blocking them in sequence wastes half the wall-clock time. This skill defines a
**system**: an orchestrator that runs both tracks simultaneously, syncs them at
the few points where they truly meet, and never lets one track idle.

## 1. Capability matrix — who can do what

| Task | gplay CLI (Track A) | Playwright (Track B) |
|---|---|---|
| AAB/ APK offline compliance (`preflight`) | ✅ | ❌ |
| Release readiness (`validate`) | ✅ | ❌ |
| Tracks, releases, staged rollout | ✅ | slow (UI) |
| Store listing text + images (bulk, char limits) | ✅ `sync`/`listings` | manual paste |
| Testers / closed-testing groups | ✅ | slow (UI) |
| Reviews, vitals, reports | ✅ | ❌ |
| **Create the app listing** | ❌ API-less | ✅ |
| **App-content declarations (10 forms)** | ❌ API-less | ✅ |
| **Exact-alarm policy declaration** | ❌ (policy gate) | ✅ (ask user first) |
| Screenshots capture (device) | via `gplay-screenshot-automation` | via adb pipeline |

**Rule:** anything on the right of the CLI column that is marked ❌ is mandatory
browser work; anything marked ✅ must NOT be done through the browser (it is
slower and more fragile). Overlap (listings) prefers CLI.

## 2. System architecture

```
ORCHESTRATOR (the main agent)
 ├─ TRACK A (CLI worker)      : subagent or same agent between browser rounds
 │    gplay preflight → validate → metadata sync → testers → release
 └─ TRACK B (Playwright driver): main agent, because the browser is stateful
      app creation → declarations → store graphics → final submission
        ▲ sync points (S1..S4) ▼
```

- **Track B must be single-threaded.** One browser session = one driver. Two
  parallel Playwright flows on the same console fight over the same DOM.
- **Track A is embarrassingly parallel.** Hand it to a subagent (a fast model is
  enough — the commands are mechanical) and keep driving the browser meanwhile.
- Interleave: every time Track B needs a page to settle, spend that beat issuing
  a Track A command instead of sleeping.

### Sync points

| # | When | What must be true |
|---|---|---|
| S1 | before any upload | `gplay preflight` has **0 errors** (or the user accepted them) |
| S2 | before store text is final | `gplay validate` char-limit findings resolved |
| S3 | before sending declarations for review | all 10 declarations show `0 beyan` on the overview counter |
| S4 | before release | `validate` ready + closed-testing rule satisfied (12 testers × 14 days on personal accounts) |

## 3. Phase map (parallel assignments)

| Phase | Track A (CLI) | Track B (Browser) |
|---|---|---|
| P0 Setup | `gplay auth login --service-account`, `apps list`, API enablement | log in, select developer account |
| P1 Artifact | build AAB, `preflight` (S1), fix findings | `Uygulama oluştur` (app creation) |
| P2 Content | `validate`, metadata draft (char limits) | 10 declarations (counter → 0) (S3) |
| P3 Store | `sync` listing text/images | graphics upload fallback, feature graphic |
| P4 Testing | `testers-orchestration`, internal/closed tracks | store listing review |
| P5 Release | `release --dry-run` → release, `rollout` | publishing summary submit (approval gate) |

## 4. Track A — CLI recipes

```powershell
# Setup
gplay auth login --service-account ./google-play-service-account.json
gplay apps list --output table                 # proves account-level access
gplay validate --package com.example.app       # ready: true? warnings?

# Artifact gate (S1) — human-readable output, JSON omits finding texts!
gplay preflight --file app-release.aab
gplay preflight --file app-release.aab --listings-dir ./metadata

# Store text with limit validation (S2)
gplay sync ... / gplay listings ... / gplay validate

# Testing and release (S4)
gplay release --package com.example.app --track internal --bundle app.aab --dry-run
gplay rollout ...
```

### CLI gotchas (real sessions)
- **PATH not refreshed after install** in the current shell → call
  `%LOCALAPPDATA%\gplay\bin\gplay.exe` by full path or refresh `Path` from the
  registry.
- **`apps list` 403 SERVICE_DISABLED** — "Google Play Developer Reporting API has
  not been used in project <n>": the SA's GCP project lacks the API. Fix:
  `gcloud services enable playdeveloperreporting.googleapis.com --project=<n>`.
  Note `validate` may still work (different API surface) — do not read one as
  proof of the other.
- **`preflight --output json` prints counts, not finding texts.** For messages
  and hints run it in human mode (or both).
- **`credential_file: expo-root.pem` is a false positive** — Expo's PUBLIC
  update-verification certificate. Verify: extract the PEM; `BEGIN CERTIFICATE`
  = fine, `BEGIN * PRIVATE KEY` = critical.
- **SDK detection ≠ activation.** Firebase classes ship via FCM dependencies
  (`google-services.json` absent = no data flow) — still disable analytics
  explicitly (`firebase_analytics_collection_enabled=false`) so the Data safety
  declaration stays truthful, and do NOT add `AD_ID` if there are no ads.
- **`eas submit --track` flag does not exist** on older eas-cli — put the track
  in `eas.json`. Submission failure "Something went wrong" almost always means
  **the app does not exist in Play Console yet**.
- **`gcloud ... keys create` leaves a 0-byte file on failure** — check size, not
  `Test-Path`.

## 5. Track B — browser recipes

App creation → declarations → store graphics. Progress is measured by the
overview counter: *"N beyanla ilgili işlem yapmanız gerekiyor"* must reach **0**.

Declaration answer matrix for a no-data utility: Ads **No**, Login **No**,
Ad ID **No**, Government **No**, Financial → **"no financial features" (last
checkbox)**, Data safety (wizard, collects? **No**), Content rating (IARC: all
**No**, category "All other app types", accept IARC terms), Target audience
**18+**, Health apps → pick the truthful feature (medication reminder =
"medication & treatment management"), Exact alarms → **policy gate: only
"Alarm clock"/"Calendar" offered — ask the user.**

### Browser gotchas (real sessions — each cost a debugging round)
1. **Refs go stale** after every DOM change → re-snapshot before each click;
   `browser_click` requires `target=<ref>`.
2. **No `setTimeout`** in the code runtime → wait with `browser_find` + snapshot
   loop; the first snapshot after `navigate` is a loading shell.
3. **Turkish characters in code strings get mangled** → match ASCII fragments
   only (`"Hay"`, `"Kaydet"`, `"Oturum"`); keep Turkish in `element` labels.
4. **Material checkboxes**: clicking the input only focuses it (`[active]`); click
   the adjacent label node and require `[checked]` in a snapshot.
5. **Suffix softening**: "Taslağı kaydet" does NOT contain "Taslak" (k→ğ);
   `"Kaydet"` matching is case-sensitive.
6. **Name collisions are destructive**: `"leri"` also matches
   "Değişiklikleri sil" → pick buttons by label length/endsWith; if a discard
   dialog opens, press **Cancel**.
7. **Single-page wizards**: "İleri" loops the same screen; the finish button is
   "Taslağı kaydet"/"Kaydet".
8. **Saving ≠ submitting**: toasts say *"…Yayınlama özeti sayfasından incelemeye
   gönderin."* — drafts are sent for review from the publishing summary.
9. **Questionnaire completion ≠ declaration completion**: after an IARC
   questionnaire save, the counter may not move until the terms checkbox is
   checked and the flow re-walked to a final save.
10. **Screenshots**: PowerShell `>` corrupts PNGs (UTF-16) → `screencap` on-device
    then `pull`; `adb input text` cuts at spaces → `%s`; Play wants 16:9 or 9:16 →
    crop 1080×2400 to 1080×1920 and drop the status bar (personal icons).

## 6. Delegation templates

**Track A worker (fast model is enough):**
> Windows PowerShell task. Install gplay, `auth login --service-account <key>`,
> run `preflight --file <aab>` (human output), `validate --package <pkg>`,
> report: install, auth, findings (errors with message+hint, first 10 warnings),
> validate readiness. Max 8 shell calls. Execute mechanically.

**Track B stays with the orchestrator** (stateful browser, policy judgment).

## 7. Safety gates

1. Production releases, pricing and policy declarations with ambiguity require
   **user approval**. The exact-alarm declaration is the classic trap.
2. Every write is verified (toast + counter decrement + re-read).
3. Never print, log or commit service-account keys; never delete a Google API
   key a scanner flags — advise restricting it.
4. Sample data in screenshots: no prescription drug names, no real health data,
   no medical claims.

## 8. Definition of done

- [ ] S1: preflight 0 errors (or accepted with reasons)
- [ ] S2: listing texts within limits (validate clean)
- [ ] S3: declaration counter = 0 (exact-alarm decision recorded)
- [ ] S4: closed testing started (12 testers × 14 days on personal accounts)
- [ ] Store graphics: 2-8 phone screenshots (9:16) + 1024×500 feature graphic
- [ ] Privacy policy URL live and truthful
- [ ] Final report: done / uncertain / user-only steps / links