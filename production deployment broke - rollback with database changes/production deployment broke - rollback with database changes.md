# 🚨 Production Rollback with Database Changes (Q86)

This documentation outlines the engineering strategy for handling production deployment failures that involve breaking database schema updates.

---

## 🔍 The Scenario & Crisis
Your team deploys a production update containing **both application code changes and a database migration**. 

* **The Migration:** Renames a critical database column from `old_name` to `new_name`.
* **The Code:** Updated to point exclusively to the `new_name` reference.

### 🚨 The Problem
Immediately after launch, the app starts throwing errors. You must roll back immediately. However, if you trigger a standard application code rollback, the old code will look for `old_name`—which **no longer exists** because the migration ran first. The application will remain completely broken.

---

## 🛠️ The DevOps Playbook: Expand-and-Contract

To prevent deployment lock-in and database corruption, you must enforce the **Expand-and-Contract pattern**, decoupling database changes into backward-compatible steps.

# 🚨 Production Rollback with Database Changes (Q86)

This documentation outlines the engineering strategy for handling production deployment failures that involve breaking database schema updates.

---

## 🔍 The Scenario & Crisis
Your team deploys a production update containing **both application code changes and a database migration**. 

* **The Migration:** Renames a critical database column from `old_name` to `new_name`.
* **The Code:** Updated to point exclusively to the `new_name` reference.

### 🚨 The Problem
Immediately after launch, the app starts throwing errors. You must roll back immediately. However, if you trigger a standard application code rollback, the old code will look for `old_name`—which **no longer exists** because the migration ran first. The application will remain completely broken.

---

## 🛠️ The DevOps Playbook: Expand-and-Contract

To prevent deployment lock-in and database corruption, you must enforce the **Expand-and-Contract pattern**, decoupling database changes into backward-compatible steps.

### 1. Phase 1: Expand (Deploy #1)
* **Action:** Run a migration that adds `new_name` but **keeps** `old_name`. Deploy application code that **writes to both columns simultaneously** but reads only from `old_name`.
* **Rollback Safety:** If the deployment fails, you can safely roll back the code. The database still has `old_name`, untouched.

### 2. Phase 2: Migrate (Background Process)
* **Action:** Run a background utility script to copy historical data from `old_name` over to `new_name` in small, controlled batches. Once done, deploy a code update that tells the app to **read and write exclusively from `new_name`**.
* **Rollback Safety:** If this step introduces bugs, you can still roll back instantly because `old_name` is still present and actively populated with live data.

### 3. Phase 3: Contract (Deploy #2)
* **Action:** After running stably in production for a few days, execute a clean-up database migration to **drop the legacy `old_name` column**.

---

## 🚒 Active Firefighter Mode: Emergency Recovery
If this disaster occurs in production without the pattern above in place, follow this immediate recovery sequence:

1. **Database Mitigation:** Run an emergency reverse migration script to add the old column back (`ALTER TABLE users ADD COLUMN old_name TEXT;`).
2. **Data Syncing:** Sync the data values backwards from the newly populated rows (`UPDATE users SET old_name = new_name;`).
3. **Application Rollback:** Once the database contains both columns, trigger the code rollback to the previous stable release.

---

## 💬 How to Explain This to an Interviewer
Use the **STAR methodology** when asked about complex deployment failures:

* **Situation:** A production rollout fails, requiring an immediate rollback, but a database column migration has already executed and deleted old fields.
* **Task:** Restore production stability immediately and implement an architecture that prevents database-coupled deployment bottlenecks.
* **Action:** Detail the emergency database remediation steps (re-adding the column and reverse-syncing data) to allow a safe code rollback. For long-term prevention, introduce the **Expand-and-Contract** deployment pattern.
* **Result:** Eliminated production deployment lock-in, achieved zero-downtime database rollouts, and created a fully reversible continuous deployment system.