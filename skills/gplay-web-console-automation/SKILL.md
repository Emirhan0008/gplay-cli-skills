---
name: gplay-web-console-automation
description: Automate the Google Play Console WEB UI with Playwright for the tasks gplay CLI cannot do - creating an app listing, app category + contact email (Store settings), the ten app-content declarations (ads, login, content rating, target audience, data safety, ad ID, government, financial, health apps, exact alarms), the exact-alarm policy gate, and the "submit for review" lock (closed-testing channel setup). Includes battle-tested gotchas: stale refs, Turkish-character matching failures, checkbox-label clicks, and destructive-button name collisions. Use when a release is blocked on a Play Console form or the app is not created yet.
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

## Category + contact email (web-only, API cannot)
- Route: "Kullanici sayisini artirin" -> "Play Store'daki varligi" -> **"Magaza
  ayarlari"** (Store settings). NOT "Magaza girisleri" - the category lives in
  Store settings, and "Magaza ayarlarlari" name-shadows the two adjacent menus.
- "Uygulama kategorisi -> Duzenle" opens a modal; the category dropdown lists
  top-level categories ("Saglik ve Fitness" AND "Tip" are separate entries).
  Health/medication apps: pick "Saglik ve Fitness". "Kaydet" is disabled until a
  change; saving keeps it a DRAFT.
- Draft toast opens a dialog: "Yayinlama ozeti sayfasina gidilsin mi?" -> button
  "Genel bakisa git" navigates to the publishing summary.
- Contact email: "Magaza girisi iletisim bilgileri -> Duzenle" -> E-posta is
  mandatory -> button "Kaydet ve yayinla" -> a SECOND confirm dialog repeats the
  same label "Kaydet ve yayinla" -> click it again -> toast "Degisiklik
  yayinlandi" (contact info publishes immediately, no review).

## The "Uygulamayi incelemeye gonder" lock (misleading on purpose)
- Disabled button + lock text: "Degisiklikleri incelemeye gondermek için
  uygulamanin kontrol panelindeki gerekli adimlari tamamlayin".
- RED HERRINGS that do NOT unlock it: the setup card reaching "11/11 tamamlandi"
  and vanishing; "Uygulama icerigi" saying "Tamamlanmayi bekleyen beyan yok";
  a complete health declaration ("Kaydet" disabled = already saved).
- REAL gate: dashboard "Kapali test" card -> "Gorevleri goruntule" -> the
  closed-testing channel setup (page "Surum kanali - Kapali test - Alpha"):
  1. "Ulke ve bolge secin" -> "Ulke/bolge ekle" -> checkbox list (Turkiye) ->
     "Kaydet" (this save also triggers the "go to publishing summary?" dialog).
  2. "Test kullanicilarini secin" - needs the tester emails (human input).
  3. "Yeni surum olusturun" -> "Surumu onizleyin ve onaylayin" -> "Surumu
     incelenmek uzere Google'a gonderin" (both show lock icons until 1-2 done).
- Publishing summary "Degisiklikler" grid: each row menu "Diger islemler" has a
  single item "View change" that opens the exact form. Row text like "Saglik
  uygulamalari beyanini tamamlayin" is a change SUMMARY, not proof of an
  unfinished form - check the form's Kaydet state instead.

## Safety
- Never answer a policy declaration you are unsure about - ask the user.
  (Exact-alarm declaration is the classic trap.)
- Verify each save with the toast + the counter decrement.
