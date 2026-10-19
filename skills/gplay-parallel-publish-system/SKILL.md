---
name: gplay-parallel-publish-system
description: Complete dual-track Google Play publishing system that advances the gplay CLI and Playwright web-console automation in parallel - orchestrator/worker model, phase-by-phase playbook with exact commands, a 40+ entry error catalog (symptom cause fix) built from real publishing sessions, policy gates, and verification doctrine. Use when publishing an Android app end-to-end, when a Play Console form or upload fails, or when you need the fastest safe path from AAB to a store release.
---

# gplay-parallel-publish-system — dual-track Play publishing (v2)

> **App-agnostic / global skill.** Works for any Android package and locale:
> substitute com.example.app, 	r-TR, paths and texts with your own. Not tied
> to any repository. Written from a real end-to-end publishing session.

Publishing touches TWO disjoint tool surfaces. The Play Developer API (what
`gplay` drives) cannot create an app or fill the ten app-content declarations;
the browser cannot efficiently bulk-upload metadata and images. Blocking them in
sequence wastes half the wall-clock time. This skill runs both **in parallel**
and pre-answers the errors each track throws.

```
ORCHESTRATOR ── TRACK A (CLI worker, delegable to a fast model)
            └─ TRACK B (Playwright driver, keep on the main agent)
                 ▲ sync points S1..S4 ▼
```

- Track B is **single-threaded** (one browser session = one driver).
- Track A is embarrassingly parallel — hand it to a subagent and drive the
  browser while it works. Never sleep between browser rounds; spend that beat on
  a CLI command.

## 1. Capability matrix

| Task | gplay CLI | Playwright |
|---|---|---|
| `preflight` offline compliance, `validate` readiness | ✅ | ❌ |
| Tracks, releases, staged rollout, reviews, vitals, reports | ✅ | slow |
| Store listing text + images (bulk, char limits) | ✅ preferred | fallback |
| Testers / closed-testing groups | ✅ | slow |
| **Create the app listing** | ❌ API-less | ✅ |
| **10 app-content declarations** | ❌ API-less | ✅ |
| **Exact-alarm policy declaration** | ❌ | ✅ (ask user first) |

## 2. Phase map

| Phase | Track A (CLI) | Track B (Browser) |
|---|---|---|
| P0 Setup | `gplay auth login --service-account`, `apps list` | login, pick developer account |
| P1 Artifact | build AAB, `preflight` (S1) | create app listing |
| P2 Content | `validate`, metadata draft | 10 declarations → counter = 0 (S3) |
| P3 Store | `sync import-listings`, `images upload` (S2) | verify in UI |
| P4 Testing | `testers-orchestration`, closed track (S4) | listing review |
| P5 Release | `release --dry-run` → `release`, `rollout` | publishing summary (approval gate) |

Sync points: **S1** 0 preflight errors before upload · **S2** metadata within
limits · **S3** declaration counter 0 · **S4** closed-testing rule satisfied.

## 3. Command spine (Track A)

```powershell
$g = "$env:LOCALAPPDATA\gplay\bin\gplay.exe"     # PATH may be stale - use full path
& $g auth login --service-account ./google-play-service-account.json
& $g apps list --output table                    # proves account access
& $g validate --package com.example.app          # ready: true?
& $g preflight --file app-release.aab            # human output has the texts!
& $g preflight --file app-release.aab --listings-dir ./store-metadata

# edit-session pattern (listings AND images both need it)
$edit = ([regex]::Match((& $g edits create --package com.example.app | Out-String), '\d{15,}')).Value
& $g sync import-listings --package com.example.app --edit $edit --dir ./store-metadata
& $g images upload --package com.example.app --edit $edit --locale tr-TR --type featureGraphic --file fg.png
& $g images upload --package com.example.app --edit $edit --locale tr-TR --type icon --file icon-512.png
& $g images upload --package com.example.app --edit $edit --locale tr-TR --type phoneScreenshots --file s1.png
& $g edits validate --package com.example.app --edit $edit
& $g edits commit  --package com.example.app --edit $edit
```

Verification doctrine: every write is proven by **re-reading it** (`images list`
needs `--edit --type`; `listings get --locale`). An upload response is evidence;
a commit response alone is not proof the payload persisted.

## 4. Error catalog — symptom → cause → fix

### 4.1 Setup & auth
| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | `gcloud projects create` → RESOURCES_NOT_FOUND / quota | personal account out of project slots | use an existing project, or delete an unused one (30-day recovery) |
| 2 | `keys create` failed but `Test-Path` says the file exists | failed command leaves a **0-byte** file | check **size**, delete stubs |
| 3 | `gcloud --impersonate-service-account` fails | missing `roles/iam.serviceAccountTokenCreator` | grant the role or use the key file directly |
| 4 | "Where is API & Services in Play Console?" | it is in **Google Cloud Console** | create SA there; invite it under Play Console → Users and permissions (Publisher) |
| 5 | Playwright lands on `google.play/business/` marketing page | no console session | user logs in once; then **select the developer account** in the account picker |
| 6 | `gplay` not found right after install | PATH not refreshed in this shell | call `%LOCALAPPDATA%\gplay\bin\gplay.exe` or reload Path from the registry |
| 7 | CLI prompts die with "stdin is not readable" | interactive prompt in a non-TTY | pass flags explicitly; use `--confirm` |
| 8 | `gplay apps list` → 403 SERVICE_DISABLED | "Play Developer Reporting API" disabled in the SA's GCP project | `gcloud services enable playdeveloperreporting.googleapis.com --project=<n>` |
| 9 | SA cannot see a newly created app | grant is app-scoped, not account-scoped | re-invite at account level, or grant per app |
| 10 | `validate` works but `apps list` 403s (or vice versa) | different API surfaces | fix each API separately; never read one as proof of the other |

### 4.2 Artifact & preflight
| # | Symptom | Cause | Fix |
|---|---|---|---|
| 11 | `eas submit` → "Something went wrong when submitting your app" | **the app does not exist in Play Console yet** (message is always vague) | create the app listing first, resubmit |
| 12 | `eas submit --track` → "Nonexistent flag" | older eas-cli | put the track in `eas.json > submit.*.android.track`, use `--latest` |
| 13 | preflight ERROR `credential_file: ... expo-root.pem` | Expo's **public** update-verification certificate | extract the PEM: `BEGIN CERTIFICATE` = false positive (document it); `BEGIN * PRIVATE KEY` = critical |
| 14 | preflight `--output json` shows counts only | JSON omits finding texts | run human output for messages + hints |
| 15 | `tracking_sdk: Firebase Analytics` + `advertising_id` warning | FCM dependency ships SDK classes | no `google-services.json` = no data flow; still set `firebase_analytics_collection_enabled=false`; do NOT add `AD_ID` without ads |
| 16 | `exported_component` warnings on widget providers | the launcher must read the provider | document as accepted risk; blind `exported=false` breaks widgets |
| 17 | `policy: targetSdkVersion below floor` | Play raises the floor each August (36 from 2026-08-31) | bump targetSdk; `--min-target-sdk` override if the tool lags |
| 18 | `native_libs: 16 KB page alignment` / missing arm64 | old NDK/build tooling | rebuild with current RN/NDK templates |

### 4.3 Browser automation (Playwright)
| # | Symptom | Cause | Fix |
|---|---|---|---|
| 19 | click lands wrong or times out | **refs go stale** after every DOM change | re-snapshot before every click |
| 20 | `browser_click` → "target: Missing key" | `element` alone is not accepted | pass `target: "<ref>"` |
| 21 | snapshot right after navigate shows a shell | page not rendered; `setTimeout` unavailable | wait loop: `browser_find` + snapshot until the marker appears |
| 22 | `includes("Oturum açma")` silently false | **Turkish characters get mangled** in code strings | match ASCII fragments only (`"Hay"`, `"Kaydet"`, `"Oturum"`); Turkish only in `element` labels |
| 23 | checkbox click only focuses (`[active]`) | Material inputs swallow clicks | click the adjacent label `generic` node; require `[checked]` in a snapshot |
| 24 | button not found although it exists | **suffix softening**: "Taslağı kaydet" contains neither "Taslak" nor "Kaydet" (k→ğ, case) | match `m[1].toLowerCase().endsWith("kaydet")` on the parsed label |
| 25 | "Changes discarded?" dialog appears unexpectedly | `"leri"` matched **"Değişiklikleri sil"** (destructive!) | pick buttons by label length/`endsWith`; in such a dialog always press **Cancel** |
| 26 | Next button loops the same screen | single-page wizard; "İleri" only scrolls | the finish button is "Taslağı kaydet"/"Kaydet" |
| 27 | Save stays `[disabled]` after typing | React validation fires on blur | press Tab then Enter; verify the value in the snapshot |
| 28 | click blocked: "intercepts pointer events" | modal/overlay pane | Escape, or navigate directly — the error log leaks the real URL |
| 29 | `/app/<id>/store-listing` lands on the app list | wrong path in the modern console | use `/app/<id>/store-presence/store-listing` |
| 30 | questionnaire "saved" but declaration still counted | terms checkbox unchecked / flow not re-walked | check the terms box, re-walk to a final save; watch the counter |
| 31 | help tabs multiply and confuse snapshots | "learn more" links open tabs | `browser_tabs close` the extras; verify `Page URL` before parsing |
| 32 | regex in code fails "Invalid regular expression flag" | regex literals with Turkish/space/quote content | use `includes()` filters, never regex with localized text |

### 4.3 Metadata & images
| # | Symptom | Cause | Fix |
|---|---|---|---|
| 33 | `images upload` → "Error: --locale is required" | store images are **per-locale** | add `--locale tr-TR` to every upload |
| 34 | `images list` → "--type is required" / "--edit is required" | listing is per-edit+type | pass `--edit <id> --type phoneScreenshots` |
| 35 | feature graphic rejected | it must be **exactly 1024×500** (1020 was tried) | regenerate precisely; icon exactly 512×512 |
| 36 | uploaded images vanish | `edits commit` forgotten | always `edits validate` → `edits commit` |
| 37 | `edits delete` refuses | requires `--confirm` | pass it; otherwise stale edits expire (harmless) |
| 38 | PNG corrupted (`FF FE` signature) after capture | PowerShell `>` writes UTF-16 | capture on-device (`screencap -p /sdcard/x.png`) then `adb pull` |
| 39 | typed names lose spaces ("D Vitamini" → "D") | `adb input text` splits at spaces | use `%s` escapes (`C%sVitamini%s1000`) |
| 40 | Turkish text looks like `?la?` in the console | console code page, not file corruption | verify bytes (`C4 B0` = İ) or re-read via API |
| 41 | metadata scanner errors | limits are counted in **runes**: title ≤30, short ≤80, full ≤4000, notes ≤500 | validate with `preflight --listings-dir` before upload |

### 4.4 Submission & release
| # | Symptom | Cause | Fix |
|---|---|---|---|
| 42 | production release blocked | personal developer accounts require **12 testers × 14 days** closed testing | run closed track first (`gplay testers-orchestration`); that wait cannot be skipped |
| 43 | exact-alarm declaration offers only "Alarm clock"/"Calendar" | Play restricts `USE_EXACT_ALARM` | **policy gate — ask the user**. Options: declare the closest truthful one, or drop to `SCHEDULE_EXACT_ALARM` + settings prompt (rebuild) |
| 44 | listing review asks for more | missing privacy policy URL / data safety / graphics | complete the declaration counter (must show 0) |
| 45 | first release needs review | Play reviews new apps fully | internal track is instant; closed/production get reviewed — set expectations |

## 5. Policy gates (never decide alone)

1. Any declaration you are not **sure** is truthful (exact alarms is the classic).
2. Production releases, pricing changes, app deletion.
3. Data safety wording when an SDK ships but does not flow data — say what is
   true after your fix (disable the SDK, then declare "no collection").
4. Screenshot/sample data: **no prescription drug names**, no real health data,
   no medical claims ("tedavi eder" etc.). Vitamins/OTC only.

## 6. Browser gotcha quick-card (paste into any session)

- Re-snapshot before every click; `target=<ref>` is mandatory.
- No `setTimeout`; wait with `browser_find` + snapshot loop.
- Match ASCII fragments only; Turkish goes in `element` labels only.
- Checkboxes: click the label, then require `[checked]`.
- Button labels: parse `button "(.+?)" \[` and compare `endsWith` + length.
- Destructive dialogs: always Cancel, then re-verify state.
- Verify every write twice: toast + re-read (counter, `listings get`, `images list`).

## 7. Delegation templates

**Track A worker** (fast model is enough — mechanical):
> Windows PowerShell batch. Full-path gplay. Tasks: (1) push branch, (2) `apps
> list`, (3) write UTF-8 metadata files with exact texts, (4) `preflight
> --listings-dir`, (5) edit-session upload + commit. Report each command's
> evidence. Max 12 shell calls.

**Track B stays with the orchestrator** — stateful browser + policy judgment.

## 8. Definition of done

- [ ] S1 preflight clean (or accepted with reasons recorded)
- [ ] S2 metadata validated and uploaded (re-read proof)
- [ ] S3 declaration counter = 0 ("Tamamlanmayı bekleyen beyan yok")
- [ ] S4 closed testing started (12 × 14) or production access applied
- [ ] Graphics: icon 512×512, feature 1024×500, 2-8 phone screenshots (9:16 crop,
      status bar removed)
- [ ] Privacy policy URL live and truthful
- [ ] Final report: done / uncertain / user-only steps / links