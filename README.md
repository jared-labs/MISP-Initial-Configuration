# VMCTI01 — MISP Initial Configuration (Org, Feeds, Warninglists, Schedules)

> Post-install configuration of MISP: organisation setup, user/role creation, OSINT feeds, warninglists, and scheduled metadata updates.

---

## At a Glance

| Field | Value |
|-------|-------|
| Service URL | `https://misp.vmcti01.lan/` or `https://<VMCTI01 IP>/` |
| Default Credentials | `admin@admin.test` + password from install runbook |
| App Root | `/var/www/MISP` |
| Cake CLI | `/var/www/MISP/app/Console/cake` |
| Covers | Organisation, Users/Roles, Feeds, Warninglists, Cron Schedules |
| Depends On | VMCTI01 base install runbook (OS + MISP + Supervisor workers) |

---

## Prerequisites

- [ ] Proxmox VM **VMCTI01** deployed (Ubuntu Server 24.04 minimal)
- [ ] MISP installed and reachable over HTTPS
- [ ] SSH access to VMCTI01 as a sudo-capable user
- [ ] Default MISP login available (`admin@admin.test` + initial password from install runbook)
- [ ] Supervisor already managing MISP workers (from install runbook)

---

## 1 — First Login

1. Open a browser and go to `https://misp.vmcti01.lan/` or `https://<VMCTI01 IP>/`.
2. Log in with the default admin credentials from the install runbook:
   - Username: `admin@admin.test`
   - Password: `<password from installer>`
3. **Immediately change the password** for the default admin user:
   - Top-right → click on your username → **My Profile**
   - Click **Edit** and set a strong password
   - Save

> We'll replace this user with a proper named admin account shortly. For now, keep it working as a "bootstrap" account.

---

## 2 — Rename the Default Organisation

MISP ships with a generic, stock organisation. Update it to something homelab-specific (e.g. `Homelab CTI`).

### 2.1 Update the Organisation Object

1. Go to **Administration → Organisations**.
2. Click the **edit** icon for the default organisation.
3. Update fields:
   - **Name:** `Homelab CTI`
   - **Description:** `Homelab Cyber Threat Intelligence`
   - Adjust other fields as desired (nationality, sector, etc.)
4. Save.

### 2.2 Ensure Server Uses the New Org

1. Go to **Administration → Server Settings & Maintenance**.
2. Click the **MISP** tab.
3. Locate settings referencing the **host organisation** (look for anything referencing Host organisation, Default organisation, or Local instance organisation).
4. Make sure these point at the org you just renamed/created (e.g. `Homelab CTI`).
5. Save changes.

> Once this is done, any local events you create should show `Homelab CTI` as the owning organisation.

---

## 3 — Create Proper Users & Roles

Goal:

- Replace `admin@admin.test` with a **named, day-to-day admin** account.
- Create a dedicated **API/service user** (for Cribl, scripts, etc.).
- Make sure roles are configured so we can see Feeds, Warninglists, etc.

### 3.1 Create a Named Admin Account

1. Go to **Administration → List Users**.
2. Click **Add User**:
   - **Email:** `cti_admin@homelab.lan`
   - **Organisation:** `Homelab CTI`
   - **Role:** admin-level role (often **Org Admin** or **Admin**)
3. Enable privilege flags:
   - **Site admin** (global administrator)
   - **Sync user** / **Synchronization** capabilities
4. Save the user.
5. Set a password (directly or via email reset if configured).
6. Log out of `admin@admin.test` and log in as `cti_admin@homelab.lan`.
7. Confirm the full set of menus appears (Administration, Feeds, Warninglists, etc.).
8. **Disable** the `admin@admin.test` account (recommended: disable rather than delete for audit history).

> **Quirk:** The MISP UI doesn't always show a clear "Site Admin" role in the role dropdown. Instead, there may be a **role** for org admin plus a separate **"site admin" checkbox** or similar flag on the user. You must enable both the correct role and this site/sync flag to see global features like Feeds and Warninglists.

### 3.2 Create a Dedicated API/Service User

1. Go to **Administration → List Users**.
2. Click **Add User**:
   - **Email:** `cti_api@homelab.lan`
   - **Organisation:** `Homelab CTI`
   - **Role:** a restricted role appropriate for automation (read-most, write where needed for pulling/pushing IOCs)
   - Typical permissions for a pull-only integration: can read events/attributes, can use the API, does not need to be a site admin.
3. Save the user.

### 3.3 Generate an Authkey (API Key) for the Service User

1. Log in as `cti_api@homelab.lan` (or impersonate as admin).
2. Top-right → click on the username → **My Profile**.
3. Find the **Auth keys** or **API keys** section.
4. Click **Add authkey**:
   - **Comment:** `Cribl VMCTI01 IOC Pull`
   - Save to generate the key.
5. Copy the Authkey value.
6. Store the key securely (e.g. in your Vaultwarden instance on **VMVLT01**).

> You'll use this Authkey later in your Cribl pipeline / API clients to authenticate against MISP.

---

## 4 — Warninglists (False-Positive Lists)

MISP Warninglists are curated sets of **known-good** or **highly common** values used to spot **false positives** (RFC1918 ranges, cloud provider IPs, popular top domains, etc.). When you export IOCs with `enforceWarninglist: 1`, any attributes matching an enabled warninglist are filtered out.

### 4.1 Navigate to Warninglists

1. Go to **Input Filters → Warninglists**.
2. You should see a list of warninglists with enable/disable status, description, and **Last updated** date.

### 4.2 Update Warninglists Initially

1. On the Warninglists page, look for an **Update** or **Fetch all** button.
2. Trigger an update to download the latest warninglists.
3. If background workers are running, **Last updated** times should change after a short delay.

### 4.3 Enable Useful Warninglists for Homelab

Enable lists that help avoid noisy false positives:

- **RFC1918 private address spaces**
- **Reserved networks** (RFC5735, etc.)
- Microsoft / Azure / Office 365
- Google / G Suite
- AWS / Cloudflare / other top CDNs
- Dynamic DNS providers

Steps:

1. Use the **search/filter** bar to find relevant lists (`RFC1918`, `Microsoft`, `Azure`, `Google`, `O365`, `Dynamic DNS`, `Top domains`, etc.).
2. Toggle **Enabled** to **On** for each relevant list.
3. Confirm status updates on the table.

> When we later export IOCs (e.g. to Cribl), setting `enforceWarninglist: 1` ensures any IOC matching an enabled warninglist won't be exported, reducing false positives in downstream tools.

---

## 5 — Feeds Setup

MISP Feeds pull in OSINT and other external data sources as MISP events/attributes.

### 5.1 Navigate to Feeds

1. Go to **Sync Actions → List Feeds** (or **Sync Actions → Feeds**).
2. You should see a list of feed definitions with columns: Enabled, Name, URL/provider, Source format, Actions (Edit, Fetch, Cache, Import).

### 5.2 Enable Standard OSINT Feeds

Start with a small number of reputable feeds:

- **CIRCL OSINT MISP feed**
- Any other well-known, low-noise OSINT feeds included by default

Steps:

1. Locate a feed (e.g. **"CIRCL OSINT Feed"**).
2. Click the **edit** icon.
3. In the edit form:
   - Toggle **Enabled**: **On**
   - Ensure **Input source** and **Format** are correct (defaults usually fine)
   - Consider enabling: Delta merge (if present), Exclude local organisations, Override date, Publish events (optional)
4. Save the feed.

> Start small. It's better to have a few high-quality feeds you understand than a flood of noisy data.

### 5.3 Pull Data from a Feed

On **Sync Actions → List Feeds**, for each enabled feed:

1. Click **Cache all** (download feed content to local cache).
2. Once caching completes, click **Import all cached** (turn data into MISP events/attributes).

Depending on MISP version, you may also see direct **Pull** or **Fetch and import** options.

> **Quirk: No "Delta Merge"**
> In our install, we did **not** see a "Delta Merge" action for feeds. Instead, we only had full pull options (`Cache all`, `Import all cached`, etc.). This is version/feed dependent — don't panic if it's missing.

### 5.4 "Pull Queued for Background Execution" Message

When triggering a feed pull, you may see:

> **"Pull queued for background execution."**

This means the request has been scheduled for the **scheduler worker**. If the scheduler worker isn't running, nothing happens.

### 5.5 Verify Scheduler Worker via Supervisor

```bash
sudo supervisorctl status
```

Expected output:

```text
misp-workers:default                  RUNNING
misp-workers:scheduler                RUNNING
misp-workers:email                    RUNNING
```

If the scheduler is **STOPPED** or **FATAL**:

```bash
sudo supervisorctl start misp-workers:scheduler
```

After confirming workers are running, go back to the Feeds page and check for **Last pulled** timestamps or new events in the Events list.

---

## 6 — Scheduled Updates (Warninglists, Taxonomies, Galaxies, Objects)

MISP metadata (warninglists, taxonomies, galaxies, object templates) needs periodic updates to stay useful.

### 6.1 Install & Enable Cron

```bash
sudo apt update
sudo apt install -y cron
sudo systemctl enable --now cron
```

Verify:

```bash
systemctl status cron
```

You should see **active (running)**.

### 6.2 Run MISP Update Commands Manually (One-Time)

Run as `www-data` so permissions and ownership are correct:

```bash
sudo -u www-data /var/www/MISP/app/Console/cake Admin updateWarningLists
sudo -u www-data /var/www/MISP/app/Console/cake Admin updateTaxonomies
sudo -u www-data /var/www/MISP/app/Console/cake Admin updateGalaxies
sudo -u www-data /var/www/MISP/app/Console/cake Admin updateObjectTemplates
```

> These commands may take a while depending on network. Watch for errors (timeouts, permission issues, missing directories).

### 6.3 Configure Cron Jobs for Regular Updates

Edit root's crontab:

```bash
sudo crontab -e
```

Add these entries (staggered to avoid simultaneous runs):

```cron
# MISP metadata updates (run as www-data)
# Warninglists: daily at 03:05
5 3 * * *  /usr/bin/sudo -u www-data /var/www/MISP/app/Console/cake Admin updateWarningLists > /var/log/misp-update-warninglists.log 2>&1

# Taxonomies: daily at 03:15
15 3 * * * /usr/bin/sudo -u www-data /var/www/MISP/app/Console/cake Admin updateTaxonomies > /var/log/misp-update-taxonomies.log 2>&1

# Galaxies: daily at 03:25
25 3 * * * /usr/bin/sudo -u www-data /var/www/MISP/app/Console/cake Admin updateGalaxies > /var/log/misp-update-galaxies.log 2>&1

# Object templates: daily at 03:35
35 3 * * * /usr/bin/sudo -u www-data /var/www/MISP/app/Console/cake Admin updateObjectTemplates > /var/log/misp-update-objecttemplates.log 2>&1
```

Key points:

- All jobs run during a **low-traffic window** (around 03:00).
- Each writes to its own log file under `/var/log/` for easy troubleshooting.
- If `sudo` in crontab is problematic, use a `www-data`-specific crontab instead:

```bash
sudo crontab -u www-data -e
```

> Up-to-date warninglists/taxonomies/galaxies/object templates ensure false-positive handling is current, tagging/classification is modern, and galaxy mapping (threat actors, malware families, intrusion sets) stays fresh.

---

## 7 — Validation

### 7.1 Warninglists Status

1. Go to **Input Filters → Warninglists**.
2. Confirm enable/disable flags match your selections and **Last updated** timestamps are recent.

### 7.2 Taxonomies, Galaxies, and Objects

1. Go to **Administration → Taxonomies**, **Administration → Galaxies**, and **Administration → Object Templates**.
2. Confirm each page shows a populated list with recent "Last updated" indicators.

### 7.3 Feeds & Events

1. Go to **Sync Actions → List Feeds**.
2. Confirm each enabled feed shows **Enabled: Yes** and has a recent **Last pulled / Last cache** timestamp.
3. Go to **Event Actions → List Events**.
4. Verify events from enabled feeds are present with reasonable attributes (IPs, domains, etc.).

### 7.4 Quick Search Sanity Test

Use the search bar to confirm lookups work:

- Search for a known-OSINT IP or hostname from one of your enabled feeds.
- Search for `Homelab CTI` to see local organisation events.

### 7.5 Scheduler & Workers Re-Check

```bash
sudo supervisorctl status
```

Confirm all required workers (especially **scheduler**) are still **RUNNING**. If you see repeated failures, check:

- Worker logs (depending on logging config)
- `/var/log/misp-update-*.log` for cron job output

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Admin user can't see Feeds/Warninglists menus | Missing site admin flag on user | Enable both admin role **and** site admin/sync checkbox on the user object |
| Feed pull never completes | Scheduler worker not running | `sudo supervisorctl start misp-workers:scheduler` |
| "Pull queued for background execution" but no events appear | Scheduler worker stopped/fatal | Check `sudo supervisorctl status`, restart scheduler |
| No "Delta Merge" option on feeds | Version/feed-type dependent | Use `Cache all` → `Import all cached` workflow instead |
| Cron update commands produce permission errors | Running as wrong user | Use `sudo -u www-data` prefix or `www-data` crontab |
| File ownership mismatches after CLI commands | Ran cake CLI as root | Re-run as `www-data`; fix ownership with `chown -R www-data:www-data /var/www/MISP/app/tmp` |
| UI shows healthy but feeds/metadata never update | Workers down but UI doesn't alert | Make `supervisorctl status` part of periodic health checks |

---

## Quick Reference

```bash
# Check MISP worker status
sudo supervisorctl status

# Restart scheduler worker
sudo supervisorctl start misp-workers:scheduler

# Manual warninglist update
sudo -u www-data /var/www/MISP/app/Console/cake Admin updateWarningLists

# Manual taxonomy update
sudo -u www-data /var/www/MISP/app/Console/cake Admin updateTaxonomies

# Manual galaxy update
sudo -u www-data /var/www/MISP/app/Console/cake Admin updateGalaxies

# Manual object template update
sudo -u www-data /var/www/MISP/app/Console/cake Admin updateObjectTemplates

# Check cron job logs
cat /var/log/misp-update-warninglists.log
cat /var/log/misp-update-taxonomies.log
cat /var/log/misp-update-galaxies.log
cat /var/log/misp-update-objecttemplates.log

# Verify cron is running
systemctl status cron
```

---

## Quirks & Gotchas

- **"Site Admin" vs Role confusion:** There isn't always an obvious "Site Admin" role in the dropdown. Look for a role (e.g. Org Admin) plus a separate site admin checkbox/flag on the user. Both must be enabled to see global features.
- **No "Delta Merge" for feeds:** Documentation mentions it, but our environment only had `Cache all` / `Import all cached`. This is version/feed dependent — document which actions you actually see.
- **"Pull queued for background execution":** Not an error. The job is queued for the scheduler worker. If the scheduler isn't running, nothing happens silently.
- **Default admin account:** Don't keep using `admin@admin.test`. Create named accounts immediately, then disable the default for audit hygiene.
- **Cron ownership:** MISP CLI commands must run as `www-data`. Running as root causes file ownership mismatches. Always use `sudo -u www-data` or a `www-data` crontab.
- **UI "Health" vs real worker status:** The MISP UI doesn't loudly complain when background workers are down. Feeds won't pull and metadata won't update, but you won't see errors in the UI. Make `supervisorctl status` part of periodic health checks.

---

*Last updated: 2025-XX-XX*
