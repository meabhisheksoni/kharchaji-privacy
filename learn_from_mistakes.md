# Learn From Mistakes: Zero-Server Google Drive Sync & OAuth Journey
## Forensic Case Studies, Real-World Roadblocks & Strategic Pivots

> **Project:** Kharchaji (Android Voice & Offline-First Personal Finance App)  
> **Authors:** Antigravity AI × Abhishek Soni  
> **Repository:** `kharcha-overlay-full`  
> **Date:** October 2026  

---

## 📌 Executive Summary

Building an offline-first, zero-server cloud sync engine for an Android app involves far more than just writing code. In theory, integrating Google Drive looks like a simple 3-step tutorial: configure OAuth, upload a file, and restore it.

In production reality, we encountered a series of real-world bottlenecks across:
- SQLite & Room schema migration validations,
- Commercial repository hosting boundaries (GitHub Free vs Pro),
- Google Cloud security protocols & Public Suffix List rules,
- The Google Cloud "App Logo" manual review trap,
- The 100-user testing cap misconception,
- Search Console domain verification caching quirks,
- Silent HTTP 403 Forbidden API blockades.

This document serves as an exhaustive, battle-tested record of **every assumption that broke**, **why it broke**, the **forensic diagnostic methods used to discover the truth**, and the **exact solutions implemented**.

---

## 🧭 The 8 Key Roadblocks & Strategic Pivots

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              ROADMAP OF PIVOTS & LEARNINGS                             │
│                                                                                        │
│ 1. Room v10 Crash     : Simple Column Add    ──► Missing Composite Indices Found       │
│ 2. Legal Hosting      : Private Repo Pages   ──► Dedicated Public Sub-Repo Decoupling  │
│ 3. Domain Auth        : github.io Top-Level  ──► PSL Rejection ──► User Subdomain      │
│ 4. Logo Submission    : Upload App Logo      ──► Manual Review Trap ──► Empty Logo     │
│ 5. 100 User Cap       : Paid / Blocked       ──► In Production + drive.appdata (Free)  │
│ 6. Search Console     : Single HTML File     ──► Dual HTML File + Meta Tag Bulletproof │
│ 7. Branding UI Stuck  : Live API Failure     ──► Historical Cache UI + Radio Button    │
│ 8. App 403 Sync Error : Cloud Logging Query  ──► Realtime ADB Logcat ──► API Disabled  │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

### 1. The Room v10 Migration Boot Crash

* **Initial Assumption:**  
  Upgrading the SQLite database from version 9 to 10 only required executing simple `ALTER TABLE` statements to add the new sync columns (`syncId`, `updatedAt`, `isDeleted`).
* **The Reality / Roadblock:**  
  The app crashed instantly on startup with `IllegalStateException: Migration didn't properly handle: calculation_records / todo_table`. Room reported:
  `Expected: TableInfo{indices=[Index{name='index_todo_table_timestamp_isDone', ...}, ...]} Found: TableInfo{indices=[]}`.
* **Why It Happened:**  
  Room strictly validates the runtime SQLite schema against compile-time generated `AppDatabase_Impl.java`. The original v9 schema had composite indices (`timestamp + isDone`, `recordDate + isMasterSave`) that were accidentally dropped in the new `@Entity` class definitions.
* **How We Solved It:**  
  Instead of taking the lazy/destructive way out (`fallbackToDestructiveMigration` which would wipe user data), we forensically restored the exact composite index definitions in `TodoItem.kt` and `CalculationRecord.kt`. Room validated the hash, unit tests passed, and the app launched without crashing.

---

### 2. Legal Document Hosting & The Private Repo Dilemma

* **Initial Assumption:**  
  We could simply enable GitHub Pages on the existing project repository to host `privacy.html`, `terms.html`, and Google Search Console verification tokens.
* **The Reality / Roadblock:**  
  GitHub restricts GitHub Pages on private repositories to GitHub Pro / Team subscribers ($4-$21/month). The user's main project `kharcha-overlay-full` had to remain **100% private and proprietary**.
* **Strategic Pivot (Decoupling):**  
  We created a dedicated, standalone public repository:  
  👉 `https://github.com/meabhisheksoni/kharchaji-privacy`
  - Main application code remains **100% Private** in `kharcha-overlay-full`.
  - Public legal and verification assets are hosted **100% Free** on GitHub Pages (`https://meabhisheksoni.github.io/kharchaji-privacy/`).
  - Zero recurring hosting cost, zero source code exposure.

---

### 3. Google Cloud OAuth "Authorized Domains" & Public Suffix List (PSL)

* **Initial Assumption:**  
  In Google Cloud Console -> **Branding** -> **Authorised Domains**, we could simply enter `github.io`.
* **The Reality / Roadblock:**  
  Google Cloud Console rejected the input with an error:  
  `"Invalid domain. Top-level domains and public suffixes are not allowed."`
* **Why It Happened:**  
  `github.io` is listed on the global **Public Suffix List (PSL)** (like `.co.uk` or `.com`). Google prevents users from claiming root public suffixes because anyone could claim another developer's GitHub Pages.
* **Strategic Pivot:**  
  We scoped the domain to the user's specific account host:  
  👉 `meabhisheksoni.github.io`  
  Google Cloud accepted this immediately.

---

### 4. The Google Cloud "App Logo" Review Trap

* **Initial Assumption:**  
  A complete, professional OAuth consent screen should have an App Logo uploaded.
* **The Reality / Roadblock:**  
  Google has a strict rule:  
  `"If your app has a logo, you will need to submit your app for verification."`  
  Uploading an App Logo immediately triggers mandatory manual review by Google's Trust & Safety human team, freezing the app for days or weeks with requests for business registration, video walkthroughs, and domain ownership documents.
* **Strategic Pivot:**  
  **We deliberately left the App Logo input empty.**  
  Because Kharchaji uses only non-sensitive scopes (`drive.appdata`), leaving the logo blank allows the app to bypass human review entirely and publish immediately to **"In production"**!

---

### 5. The 100-User Testing Cap Myth

* **Initial Assumption / Fear:**  
  The user saw Google Cloud's warning: *"Allowed user cap prior to app verification is 100"*, and feared having to pay Google or being locked out after 100 users.
* **The Reality:**  
  The 100-user cap applies **only** when the app is in "Testing" status, OR when requesting unapproved sensitive/restricted scopes (like full access to Gmail or the user's entire Google Drive).
* **Strategic Pivot:**  
  By configuring the authorized domain, completing Search Console verification, and flipping Publishing Status to **"In production"**, the 100-user cap was permanently lifted. Any Google user worldwide can now sync without restrictions, and it is **₹0 (100% Free)**.

---

### 6. Bulletproof Dual Domain Verification (Search Console)

* **Initial Setup:**  
  Uploaded `googlebc125d1784830bdf.html` to root and verified ownership in Google Search Console.
* **User's Insight:**  
  The user requested: *"Main multiple verification method chahta hun jisse main bulletproof ho sakun."*
* **Strategic Pivot:**  
  We implemented **Dual-Method Verification**:
  1. **Method 1:** Standalone verification file `googlebc125d1784830bdf.html`.
  2. **Method 2:** Verification meta tag `<meta name="google-site-verification" content="wovPcnDXO_fBrE1H_BIe0BnV9x4CAn_P2T7hORr3s1c" />` injected into `<head>` of `index.html`.
  Even if one file is ever accidentally removed during a future website redesign, the other method guarantees uninterrupted ownership.

---

### 7. The "Branding Verification Issues" Drawer Confusion

* **The Reality / Roadblock:**  
  After verifying in Search Console, returning to Google Cloud Console still showed a drawer saying:  
  *"The website of your homepage URL is not registered to you... Proceed button is greyed out."*
* **Forensic Diagnosis from Screenshot:**  
  1. The browser URL had `;showIssues=true` hardcoded in the query string.
  2. The drawer was displaying: *"Issues found from the **previous** verification attempt"* (cached history).
  3. The radio button `○ I have fixed the issues` was unselected, which kept `Proceed` disabled.
* **How We Solved It:**  
  Selecting the radio button unlocked `Proceed`, and closing the drawer cleared the state. The underlying production status was already green.

---

### 8. The 403 Forbidden Mystery (Audit Logs vs. Live ADB Logcat)

* **The Confusion:**  
  When testing sync in the app, a red badge appeared: `Google Drive Cloud Sync [Error]`.  
  The user opened Google Cloud Console -> **Logs Explorer**, saw 2 error entries from earlier in the afternoon, and asked if a SQL/LQL query was needed.
* **Forensic Breakthrough via ADB Logcat:**  
  Instead of guessing, we ran `adb logcat` directly on the connected Android phone:
  ```json
  {
    "code": 403,
    "status": "PERMISSION_DENIED",
    "reason": "accessNotConfigured",
    "message": "Google Drive API has not been used in project 458535284989 before or it is disabled. Enable it by visiting https://console.developers.google.com/apis/api/drive.googleapis.com/overview?project=458535284989 then retry."
  }
  ```
* **The Revelation:**  
  The Google Sign-in had completely succeeded! The OAuth client, SHA-1 key, and tokens were 100% correct.  
  The **only** issue was that in a brand-new Google Cloud project, all APIs are turned OFF by default.
* **The Instant Fix:**  
  We provided the direct link to the Google Drive API library. The user clicked the blue **"ENABLE"** button.  
  Within 10 seconds, tapping "Sync Now" in the app uploaded the backup:
  ```text
  10-08 16:39:58 D SyncRepository: Created fresh backup file ID: 1bWUEk-zJAkf3fvGF33KrptjpMYky5W53POGTFe6gqeSWf2jbpQ
  10-08 16:41:55 D SyncRepository: Updated existing backup file ID: 1bWUEk-zJAkf3fvGF33KrptjpMYky5W53POGTFe6gqeSWf2jbpQ
  ```

---

## 🔒 The Zero-Server Architecture: What Actually Runs

Kharchaji's cloud backup operates under strict zero-server privacy principles:

1. **Private Scope Only (`drive.appdata`):**  
   The app requests `https://www.googleapis.com/auth/drive.appdata`. It cannot read, see, or touch any other files in the user's Google Drive.
2. **Hidden AppData Vault:**  
   The backup file (`kharchaji_cloud_backup.zip`) is stored inside Google Drive's hidden application folder. Users cannot accidentally delete it while organizing photos or documents.
3. **Direct Streaming ZIP:**  
   JSON metadata and JPG/PNG receipt images are written directly to a compressed ZIP stream via `OutputStreamWriter`, avoiding double-buffering and OutOfMemoryError crashes on large ledgers.
4. **Deterministic LWW Conflict Resolution:**  
   If multiple devices edit an expense simultaneously, latest `updatedAt` wins. On millisecond collisions, a deterministic tie-breaker (`remote.syncId > local.syncId`) ensures exact consistency across all devices.

---

## 💡 The Human UX Principle: "Trust & Transparency"

A critical user insight gained from this project:
> *"Screen par sirf 'Synced' likh dene se user ko vishwas nahi hota. Kitne transactions gaye? Kitni photos gayi? Kitne KB data cloud par hai?"*

**Engineering Correctness ≠ User Trust.**
- Never display vague status labels like *"Synced"*.
- Show transparent, physical numbers:  
  `✅ Synced: 45 Expenses • 12 Records • 3 Receipts (520 KB)`
- Give users verifiable ways to check their data (such as Google Drive -> Settings -> *Manage Apps* -> *Hidden app data*).

---
*Documented and committed to GitHub for permanent architectural reference.*
