# VMCTI01 — MISP Initial Configuration (Org, Feeds, Warninglists, Schedules)

* * *

## Overview

This runbook covers **post-install** configuration of MISP on **VMCTI01**:

- First-time web UI setup (organisation, users, roles)
- Enabling OSINT **feeds** and pulling data
- Enabling & updating **warninglists, taxonomies, galaxies, object templates**
- Configuring **cron** so MISP metadata stays fresh
- Capturing **real-world quirks** we hit during setup

> **Scope:**  
> MISP is already installed, workers are wired up via Supervisor, HTTPS is working, and the web UI is reachable at:
> - `https://misp.vmcti01.lan/`  
> - or `https://<VMCTI01 IP>/`

For base OS and application install steps, see:  
**“VMCTI01 — MISP Install & Base OS”** runbook.

* * *

## Prerequisites

- Proxmox VM **VMCTI01** deployed (Ubuntu Server 24.04 minimal)
- MISP installed and reachable over HTTPS
- SSH access to VMCTI01 as a sudo-capable user
- Default MISP login available (e.g. `admin@admin.test` + initial password from install runbook)
- Supervisor already managing MISP workers (from install runbook)

Assumed MISP path:

- App root: `/var/www/MISP`
- Cake CLI: `/var/www/MISP/app/Console/cake`

Adjust paths as needed if your install differs.

* * *

## Initial UI Setup

### 1. First Login

1. Open a browser and go to:

   - `https://misp.vmcti01.lan/`  
   - or `https://<VMCTI01 IP>/`

2. Log in with the default admin credentials (from the install runbook).  
   Example (not exact):

   - Username: `admin@admin.test`
   - Password: `<password from installer>`

3. **Immediately change the password** for the default admin user:

   - Top-right → click on your username → **My Profile**
   - Click **Edit** and set a strong password
   - Save

> We’ll replace this user with a proper named admin account shortly. For now, keep it working as a “bootstrap” account.

---

### 2. Rename the Default Organisation

MISP ships with a generic, stock organisation. Update it to something homelab-specific (e.g. `Homelab CTI`).

#### 2.1. Update the Organisation Object

1. In the top menu, go to:  
   **Administration → Organisations**
2. You should see the default org (often named something like **ORGNAME** or similar).
3. Click the **edit** icon for the default organisation.
4. Update fields:

   - **Name:** `Homelab CTI` (or your own name)
   - **Description:** `Homelab Cyber Threat Intelligence`
   - Adjust other fields as desired (nationality, sector, etc.)

5. Save.

#### 2.2. Ensure Server Uses the New Org

1. Go to:  
   **Administration → Server Settings & Maintenance**
2. Click the **MISP** tab.
3. Locate the settings that reference the **host organisation** (the exact labels vary by version, but look for anything referencing:

   - Host organisation
   - Default organisation
   - Local instance organisation

4. Make sure these point at the org you just renamed/created (e.g. `Homelab CTI`).
5. Save changes.

> Once this is done, any local events you create should show `Homelab CTI` as the owning organisation.

---

### 3. Create Proper Users & Roles

Goal:

- Replace `admin@admin.test` with a **named, day-to-day admin** account.
- Create a dedicated **API/service user** (for Cribl, scripts, etc.).
- Make sure roles are configured so we can see **Feeds**, **Warninglists**, etc.

#### 3.1. Create a Named Admin Account

1. Go to:  
   **Administration → List Users**
2. Click **Add User**.

   Use something like:

   - **Email:** `cti_admin@homelab.lan`
   - **Organisation:** `Homelab CTI`
   - **Role:** choose the admin-level role provided by your MISP install (often something like **Org Admin** or **Admin**).

3. On the same screen, look for any **privilege flags** that control scope, such as:

   - **Site admin** (global administrator)
   - **Sync user** / **Synchronization** capabilities
   - Other extended permissions (depends on version)

4. For your primary homelab admin:

   - Ensure they have **organisation admin** privileges.
   - Enable any **site-wide / sync** capabilities you’ll need to manage feeds, warninglists, server settings, etc.

5. Save the user.

6. A password reset/invitation email may be sent if email is configured. If not, you can:

   - Temporarily set a password when creating the user, or
   - Edit the user afterward and set a password directly.

7. Log out of `admin@admin.test` and log in as `cti_admin@homelab.lan`.

8. Once you confirm the new admin can access **Administration**, **Feeds**, **Warninglists**, etc., you can **disable** or **delete** the `admin@admin.test` account (recommended: disable rather than delete for audit history).

> **Quirk:**  
> The MISP UI doesn’t always show a clear “Site Admin” role in the role dropdown. Instead, there may be a **role** for org admin plus a separate **“site admin” checkbox** or similar flag on the user. You must enable both the correct role and this site/sync flag to see global features like Feeds and Warninglists.

---

#### 3.2. Create a Dedicated API/Service User

This account is for integrations (e.g. Cribl), not for humans.

1. Still as your new admin, go to:  
   **Administration → List Users**
2. Click **Add User**.

   Example:

   - **Email:** `cti_api@homelab.lan`
   - **Organisation:** `Homelab CTI`
   - **Role:** a restricted role appropriate for automation (read-most, write where needed for pulling/pushing IOCs).

   Typical permissions for a pull-only integration:

   - Can **read events/attributes**.
   - Can **use the API**.
   - Does **not** need to be a site admin in most cases.

3. Save the user.

---

#### 3.3. Generate an Authkey (API Key) for the Service User

1. Log out, then log in as **`cti_api@homelab.lan`** (or log in as admin and impersonate, depending on your MISP version).
2. Top-right → click on the username → **My Profile**.
3. Look for a section named **Auth keys** or **API keys**.
4. Click **Add authkey** (or similar).

   - Give it a descriptive **comment**:
     - `Cribl VMCTI01 IOC Pull`
   - Save to generate the key.

5. Copy the Authkey value.

6. Store the key securely (e.g. in your Vaultwarden instance on **VMVLT01**).

> You’ll use this Authkey later in your Cribl pipeline / API clients to authenticate against MISP.

* * *

## Warninglists (False-Positive Lists)

MISP Warninglists are **not** lists of “bad” indicators. They are curated sets of **known-good** or **highly common** values used to spot **false positives**, such as:

- RFC1918 private IP ranges
- Reserved networks (RFC5735, etc.)
- Common cloud provider IPs (Microsoft, Google, O365, AWS, etc.)
- Popular top domains or dynamic DNS services

When you export IOCs with **`enforceWarninglist: 1`**, any attributes that match an **enabled** warninglist are **hidden/filtered out**.

---

### 1. Navigate to Warninglists

1. In the top menu, go to:  
   **Input Filters → Warninglists**

2. You should see:

   - A list of warninglists with:
     - Enable/disable status
     - Description
     - **Last updated** date

---

### 2. Update Warninglists Initially

1. On the Warninglists page, look for an **Update** or **Fetch all** button (label can vary by version).
2. Trigger an update to download the latest warninglists.

   - The UI may show a confirmation message like **“Update queued”** or similar.
   - If background workers are correctly running, status and **Last updated** times should change after a short delay.

---

### 3. Enable Useful Warninglists for Homelab

From the Warninglists table, enable lists that help avoid noisy false positives in a homelab environment, such as:

- **RFC1918 private address spaces**
- **Reserved networks** (RFC5735, etc.)
- Major cloud providers and SaaS:

  - Microsoft / Azure / Office 365
  - Google / G Suite
  - AWS / Cloudflare / other top CDNs
  - Dynamic DNS providers

Checklist:

1. Use the **search/filter** bar to find relevant lists (`RFC1918`, `Microsoft`, `Azure`, `Google`, `O365`, `Dynamic DNS`, `Top domains`, etc.).
2. For each that makes sense:

   - Toggle **Enabled** to **On**.
   - Confirm status updates on the table.

3. Leave explicitly “malicious” or unrelated lists alone; warninglists are about “likely benign”/noisy stuff, not threats.

> Reminder: When we later export IOCs (e.g. to Cribl), setting `enforceWarninglist: 1` ensures any IOC matching an enabled warninglist **won’t be exported**, reducing false positives in downstream tools.

* * *

## Feeds Setup

MISP Feeds allow you to pull in OSINT and other external data sources as MISP **events/attributes**.

We’ll:

- Enable some standard OSINT feeds (e.g. **CIRCL OSINT**).
- Manually pull data once.
- Verify background workers are handling the pulls.

---

### 1. Navigate to Feeds

1. In the top menu, go to:  
   **Sync Actions → List Feeds**  
   (Names may vary slightly by version, e.g. **Sync Actions → Feeds**.)

2. You should see a list of feed definitions with columns such as:

   - **Enabled**
   - **Name**
   - URL / provider
   - **Source format** (MISP, CSV, etc.)
   - Actions (e.g. **Edit**, **Fetch**, **Cache**, **Import**)

---

### 2. Enable Standard OSINT Feeds

For a homelab, start with a small number of reputable feeds:

- **CIRCL OSINT MISP feed**
- Any other well-known, low-noise OSINT feeds included by default

Steps:

1. On the Feeds page, locate a feed you want to enable (e.g. **“CIRCL OSINT Feed”**).
2. Click the **edit** icon for that feed.
3. In the edit form:

   - Toggle **Enabled**: **On**
   - Ensure **Input source** and **Format** are correct (default values are usually fine).
   - Consider enabling options such as:
     - **Delta merge** (if present – see quirk below)
     - **Exclude local organisations** (if desired)
     - **Override date** or **publish events** (optional, based on your workflow)

4. Save the feed.

Repeat for any other feeds you want to use initially.

> Start small. It’s better to have a few high-quality feeds you understand than a flood of noisy data.

---

### 3. Pull Data from a Feed

Back on **Sync Actions → List Feeds**:

1. For each enabled feed, in the **Actions** column, you should see options like:

   - `Cache all`
   - `Fetch all`
   - `Import all cached`
   - Possibly `Pull`, depending on version and feed type

2. Basic workflow (simplest path):

   1. Click **Cache all** (if available) to download feed content to local cache.
   2. Once caching completes, click **Import all cached** to turn that data into MISP events/attributes.

3. Depending on your MISP version, you may also see direct **Pull** or **Fetch and import** options. Use the options that match your UI.

> **Quirk: No “Delta Merge”**  
> In our install, we did **not** see a “Delta Merge” action for feeds. Instead, we only had **full pull options** (`Cache all`, `Import all cached`, etc.).  
> This appears to be **version/feed dependent**, so don’t panic if “Delta Merge” is missing — just document which actions are available in your environment.

---

### 4. “Pull Queued for Background Execution” Message

When you trigger a feed pull, the UI may show:

> **“Pull queued for background execution.”**

This means:

- The request has been scheduled for processing by the **scheduler** worker.
- If the scheduler (and other MISP workers) aren’t running, the pull will never actually happen.

We’ll verify worker status next.

---

### 5. Verify Scheduler Worker via Supervisor

SSH to **VMCTI01**:

1. Check Supervisor status:

    sudo supervisorctl status

2. You should see a list of worker processes, including something like:

    misp-workers:default                  RUNNING
    misp-workers:scheduler                RUNNING
    misp-workers:email                    RUNNING
    ...

(Exact names can vary based on how you configured workers in the install runbook.)

3. Confirm:

   - The **scheduler** worker is **RUNNING**.
   - Any other critical workers you rely on (e.g. default, email) are also RUNNING.

4. If the scheduler is **STOPPED** or **FATAL**, start it:

    sudo supervisorctl start misp-workers:scheduler

   Adjust the process group/name to match your setup.

5. After confirming workers are running, go back to the Feeds page in the UI and:

   - Refresh the feed list.
   - Check for any **Last pulled** timestamp, or
   - Look at the Events list to see whether feed events have appeared.

* * *

## Scheduled Updates (Warninglists, Taxonomies, Galaxies, Objects)

MISP relies on several types of **metadata**:

- Warninglists
- Taxonomies
- Galaxies
- Object templates

These need periodic updates to stay useful. We’ll:

1. Ensure **cron** is installed and running on VMCTI01.
2. Run each update command **once manually** (as `www-data`) to verify.
3. Add **cron jobs** to automate updates.

---

### 1. Install & Enable Cron on VMCTI01

SSH to **VMCTI01** and run:

    sudo apt update
    sudo apt install -y cron
    sudo systemctl enable --now cron

Verify:

    systemctl status cron

You should see **active (running)**.

---

### 2. Run MISP Update Commands Manually (One-Time)

Run these as `www-data` so permissions and ownership are correct.

From an SSH session on VMCTI01:

    sudo -u www-data /var/www/MISP/app/Console/cake Admin updateWarningLists
    sudo -u www-data /var/www/MISP/app/Console/cake Admin updateTaxonomies
    sudo -u www-data /var/www/MISP/app/Console/cake Admin updateGalaxies
    sudo -u www-data /var/www/MISP/app/Console/cake Admin updateObjectTemplates

Notes:

- These commands may take a while depending on network and how much needs updating.
- Watch for any obvious errors (timeouts, permission issues, missing directories).

If these complete successfully, you’re ready to schedule them via cron.

---

### 3. Configure Cron Jobs for Regular Updates

We’ll run everything as **root’s cron**, but each job will explicitly use `sudo -u www-data` to execute Cake as the web user.

Edit root’s crontab:

    sudo crontab -e

Add entries similar to the following (staggered to avoid all running at the same time):

    # MISP metadata updates (run as www-data)
    # Warninglists: daily at 03:05
    5 3 * * *  /usr/bin/sudo -u www-data /var/www/MISP/app/Console/cake Admin updateWarningLists > /var/log/misp-update-warninglists.log 2>&1

    # Taxonomies: daily at 03:15
    15 3 * * * /usr/bin/sudo -u www-data /var/www/MISP/app/Console/cake Admin updateTaxonomies > /var/log/misp-update-taxonomies.log 2>&1

    # Galaxies: daily at 03:25
    25 3 * * * /usr/bin/sudo -u www-data /var/www/MISP/app/Console/cake Admin updateGalaxies > /var/log/misp-update-galaxies.log 2>&1

    # Object templates: daily at 03:35
    35 3 * * * /usr/bin/sudo -u www-data /var/www/MISP/app/Console/cake Admin updateObjectTemplates > /var/log/misp-update-objecttemplates.log 2>&1

Key points:

- All jobs run during a **low-traffic window** (around 03:00).
- Each writes to its own log file under `/var/log/` for easy troubleshooting.
- If your system doesn’t like `sudo` in crontab, you can:

  - Use a root-owned script in `/usr/local/sbin` that su’s to `www-data`, or
  - Use a dedicated crontab for `www-data` via `sudo crontab -u www-data -e`.

> **Rationale:**  
> Up-to-date warninglists/taxonomies/galaxies/object templates ensure:
> - False-positive handling is as good as possible.
> - Tagging and classification are modernised.
> - Galaxy mapping (threat actors, malware families, intrusion sets, etc.) stays current.

* * *

## Validation / Sanity Checks

After completing the above steps, verify that MISP is **healthy** and that the new configuration behaves as expected.

---

### 1. Warninglists Status

1. In the UI, go to:  
   **Input Filters → Warninglists**

2. Confirm:

   - **Enable/disable** flags match your homelab selections.
   - **Last updated** timestamps are recent (reflecting your manual CLI run or the last cron run).
   - No obvious errors or red indicators in the UI.

---

### 2. Taxonomies, Galaxies, and Objects

1. Go to:

   - **Administration → Taxonomies**
   - **Administration → Galaxies**
   - **Administration → Object Templates**

2. Confirm:

   - Each page shows a populated list of items.
   - “Last updated” or similar indicators look recent.
   - No errors about missing metadata.

---

### 3. Feeds & Events

1. Go to:  
   **Sync Actions → List Feeds**

2. Check for each enabled feed:

   - **Enabled:** Yes
   - **Last pulled / Last cache:** has a recent timestamp.
   - No obvious error messages.

3. Go to:  
   **Event Actions → List Events** (or just **Events** in some versions).

4. Verify that:

   - Events from your enabled feeds are present.
   - Attributes in those events look reasonable (IPs, domains, etc.).

---

### 4. Quick Search Sanity Test

Use the search bar to confirm lookups work and data is present.

Examples:

- Search for a known-OSINT IP or hostname from one of your enabled feeds.
- Search for `Homelab CTI` to see local organisation events (after you create some).

You should see:

- Results that match feed data (if present).
- No major error banners in the UI.

---

### 5. Scheduler & Workers Re-Check

On VMCTI01:

    sudo supervisorctl status

Confirm again that:

- All required workers (especially **scheduler**) are still **RUNNING**.
- No jobs are stuck in a crashed or fatal state.

If you see repeated failures, check:

- Worker logs (depending on how you configured logging).
- `/var/log/misp-update-*.log` for your cron jobs.

* * *

## Quirks & Gotchas

This section captures the real-world weirdness we hit while getting MISP initially configured.

---

### 1. “Site Admin” vs Role Confusion

- In the MISP UI, there isn’t always an obvious **“Site Admin”** role name in the role dropdown.
- Instead, there’s typically:

  - A **role** that defines base permissions (e.g. **Org Admin**), and
  - One or more **checkboxes/flags** on the user object (e.g. **Site admin**, **Sync user**) that extend capabilities.

**Impact:**  
If the right combination isn’t enabled, the user may **not see**:

- **Feeds** under Sync Actions
- **Warninglists** and certain **Administration** items

**Fix:**  
When creating your primary admin account:

1. Assign an **admin-capable role** (Org Admin/Admin).
2. Enable any **site-wide / sync privileges** on the user.
3. Log in as that user and confirm the full set of menus (Administration, Feeds, Warninglists, etc.) appears.

---

### 2. No “Delta Merge” Option for Feeds

- Some documentation mentions **“Delta Merge”** actions for feeds.
- In our environment, we **did not** see a “Delta Merge” option in the Feeds UI. Instead, we had:

  - `Cache all`
  - `Import all cached`
  - Other feed-specific actions

**Impact:**  
You might expect a delta-based update mode that only imports differences, but it may not be exposed for all feed types or MISP versions.

**Mitigation:**  

- Accept that your feed actions are **version-dependent**.
- Document the exact options you see (for future you).
- Use the available `Cache all` / `Import all cached` flow.

---

### 3. “Pull Queued for Background Execution”

- When triggering feed pulls, the UI message:

  > **“Pull queued for background execution.”**

  can be confusing if you’re expecting instant results.

**What it actually means:**

- MISP queued the job for the **scheduler worker**.
- If the scheduler worker isn’t running, **nothing happens** after that.

**Fix:**

- Use `sudo supervisorctl status` on VMCTI01.
- Ensure a **scheduler** worker is present and **RUNNING**.
- If it’s stopped or fatal, start it:
  
    sudo supervisorctl start misp-workers:scheduler

- After starting, re-try the feed pull.

---

### 4. Default Admin (`admin@admin.test`) vs Real Accounts

- It’s easy to keep using `admin@admin.test` out of convenience.
- This can make later auditing, API usage, and access control confusing.

**Best practice:**

- Use the default admin account **only** to bootstrap the system.
- Immediately create:

  - A named **admin** account (e.g. `cti_admin@homelab.lan`) for day-to-day work.
  - A dedicated **API/service** account (e.g. `cti_api@homelab.lan`) for integrations.

- Disable (or at least strongly randomise) the original `admin@admin.test` account once migration is complete.

---

### 5. Cron Ownership & Permissions

- MISP CLI commands must run as **`www-data`** (or the user your web server runs as).
- Running them as root directly can result in:

  - File ownership mismatches
  - Potential permission issues later

**Mitigation:**

- Use `sudo -u www-data` in cron entries (as shown in this runbook).
- Or use a `www-data`-specific crontab via:

    sudo crontab -u www-data -e

- Keep all MISP update logs (e.g. `/var/log/misp-update-*.log`) readable for troubleshooting.

---

### 6. UI “Health” vs Real Worker Status

- The MISP UI doesn’t always loudly complain when background workers (scheduler, default workers) are down.
- You can think everything is fine while:

  - Feeds never complete pulls
  - Warninglists/taxonomies never update

**Mitigation:**

- Make **`supervisorctl status` on VMCTI01** part of your periodic health checks.
- Optionally, track:

  - Worker status via external monitoring (e.g. Prometheus/Grafana on `vmmon01`).
  - Cron job logs for update commands.

---

With this runbook in place, VMCTI01 should have:

- A clean organisation identity (`Homelab CTI`)
- Properly scoped admin and API accounts
- Warninglists and metadata that stay up to date via cron
- Feeds pulled in by a functioning scheduler worker
- Known quirks documented so you don’t have to rediscover them twice
