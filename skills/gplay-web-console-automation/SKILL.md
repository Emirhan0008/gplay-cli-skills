---
name: gplay-web-console-automation
description: Automate the Google Play Console WEB UI with Playwright for the tasks gplay CLI cannot do - creating an app listing, the ten app-content declarations (ads, login, content rating, target audience, data safety, ad ID, government, financial, health apps, exact alarms), and the exact-alarm policy gate. Includes battle-tested gotchas: stale refs, Turkish-character matching failures, checkbox-label clicks, and destructive-button name collisions. Use when a release is blocked on a Play Console form or the app is not created yet.
---

# Play Console web automation (Playwright)

The Play Developer API does NOT expose app creation or the app-content
declaration forms. Those are web-only and must be driven through Playwright.

## Workflow
1. Play Console -> app list -> "Uygulama oluştur" (create app: name <=30, package,
   default language, App/Game, Free/Paid, 2 policy checkboxes).
2. "Uygulama içeriği" overview shows "N beyanla ilgili işlem yapmanız gerekiyor" -
   use N as your progress counter; it must reach 0.
3. Ten declarations: Ads (No), Login (No), Ad ID (No), Government (No), Financial
   ("Uygulamamda finans ile ilgili ozellik saglanmiyor" - the LAST checkbox),
   Data safety (wizard: collects data? -> Hayir), Content rating (IARC: all "none"),
   Target audience (18+), Health apps ("Ilac ve tedavi yonetimi" for a medication
   reminder), Exact alarms (policy gate: only "Alarm clock" or "Calendar" offered -
   ask the user before declaring).
4. Saving a form shows: "Degisiklik kaydedildi. Yayinlama ozeti sayfasindan
   incelemeye gonderin." - drafts are submitted for review from the publishing page.

## Gotchas (each cost a debugging round)
- Refs go stale after every DOM change: re-snapshot before each click. Clicking
  requires target=<ref>; element alone is not accepted.
- setTimeout is unavailable in the code runtime: wait with browser_find + a
  snapshot loop (the first snapshot after navigate is a loading shell).
- Turkish text in code strings gets mangled: match ASCII fragments only
  ("Hay", "Kaydet", "Oturum", "Finans"); put Turkish only in the element label.
- Material checkboxes: clicking the input only focuses it ([active]); click the
  adjacent generic LABEL node and verify the [checked] attribute in a snapshot.
- Suffix softening: the button "Taslaki kaydet" (Save draft) does NOT contain
  "Taslak" (k->g softening) and "Kaydet" matching is case-sensitive.
- Name collisions: "Ileri" also matches "Degisiklikleri sil" (destructive!). Pick
  buttons by label length/endsWith, and if a "Changes discarded?" dialog opens,
  press Cancel - never confirm.
- Some forms are single-page wizards: pressing Next loops the same screen; the
  finish button is actually "save draft".

## Safety
- Never answer a policy declaration you are unsure about - ask the user.
  (Exact-alarm declaration is the classic trap.)
- Verify each save with the toast + the counter decrement.
