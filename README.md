# SCCM / SMS Monitoring for Dynatrace

Observability for Microsoft SCCM (Configuration Manager / SMS) using a Dynatrace Extension 2.0 and two Notebooks/Dashboards. The extension pulls WMI performance counters from your site servers and management points every minute; the dashboards surface those metrics alongside IIS log analysis, host health, and Davis AI problems — giving you a single pane of glass for your SCCM estate.

---

## Repository contents

| File | Purpose |
|---|---|
| `extension.yaml` | Dynatrace Extension 2.0 — WMI metric collection from SCCM servers |
| `SCCM - Reworked.json` | Main operational dashboard (IIS logs, client health, host health, problems) |
| `SMS Dashboard.json` | Focused WMI metrics dashboard (inbox/outbox depth, MP throughput, SQL Server) |

---

## What the extension collects

The extension (`custom:sms-data-extension`, min Dynatrace version 1.303.0) runs WMI queries on your SCCM servers and reports the following custom metrics at a 1-minute interval:

### Site Server metrics
| Metric key | Description |
|---|---|
| `custom.sccm.inbox.file_current_count` | File depth per SMS inbox queue (per `Name`) |
| `custom.sccm.outbox.file_current_count` | File depth per SMS outbox queue (per `Name`) |
| `custom.sccm.inventory_loader.mifs_processed_per_minute` | MIFs processed/min by the Inventory Data Loader |
| `custom.sccm.software_inventory.sinvs_processed_per_minute` | SINVs processed/min by the Software Inventory Processor |
| `custom.sccm.status_manager.processed_per_sec` | Status messages processed/sec (per component `Name`) |
| `custom.sccm.discovery.non_user_ddrs_per_minute` | Non-user DDRs processed/min by Discovery Data Manager |

### Management Point metrics
| Metric key | Description |
|---|---|
| `custom.sccm.mp.hinv.total_reports_per_sec` | Hardware inventory reports/sec at the MP |
| `custom.sccm.mp.sinv.total_reports_per_sec` | Software inventory reports/sec at the MP |
| `custom.sccm.mp.ddr.total_reports_per_sec` | DDR reports/sec at the MP |

### Windows OS
| Metric key | Description |
|---|---|
| `custom.sccm.os.memory.available_mbytes` | Available memory in MBytes (all servers) |

All metrics carry a `host.name` dimension so you can filter per server.

---

## What the dashboards show

### SMS Dashboard (`SMS Dashboard.json`)
A tight operational view focused purely on WMI metrics and the backend SQL Server:

- **Inbox / Outbox depth** — current file counts as single-value tiles and bar charts per host, flagging queue build-up at a glance
- **Status Messages processed/sec** — gauge and line chart per host
- **Management Point throughput** — HInv and DDR reports/sec over time
- **SQL Server health** — compilations, transactions, total memory, and active user connections (from the Dynatrace SQL Server extension)

### SCCM - Reworked Dashboard (`SCCM - Reworked.json`)
The full operational dashboard. Filtered by a `$HostGroups` variable so you can scope to a subset of your environment.

**Host Health**
- Monitoring mode breakdown (Full Stack / Infrastructure / Discovery) — donut chart
- Host availability state (Running / Offline)
- Memory usage per host — honeycomb heat map with warning (≥90%) and critical (≥95%) thresholds; table for the worst offenders
- Disk usage per host — honeycomb heat map (warning ≥80%, critical ≥90%)
- Network traffic (NIC bytes sent/received) across host groups
- Total traffic volume (single value)

**Davis AI Problems**
- Problem count (red if >0, green if 0)
- Full problem table with severity colour-coding, affected entities, duration, and root cause — filtered to selected host groups

**SCCM Client Highlights** *(IIS log analysis)*
- Active clients per day over 7 days (unique IPs hitting `/CCM_System`, `/SMS_MP`, etc.)
- Policy request failure rate per hour — success vs. 4xx vs. 5xx stacked bar chart and table
- Client version distribution (donut chart of SCCM client versions seen in the last 24 h)
- Client staleness tables — last-seen bucketed at 7–14 days, 14–21 days, and 21+ days

**Management Point (MP) APIs** *(IIS log analysis)*
- Total request count and error count (single value tiles)
- Request volume over time by HTTP method and status code
- Average request duration trend (successful requests only)
- Per-endpoint summary table (call count + avg duration + status code)
- Authentication endpoint stats (`/SimpleAuthWebService`, `/DssAuthWebService`)

**Content Distribution & Distribution Point (DP) Access**
- Client update bootstrap count and timeline (`/selfupdate/wuident.cab` hits)
- IIS files sent/received totals and bar chart
- IIS web service cache hit count
- Server Sync Web Service (`/ServerSyncWebService`) average and max duration

**SCCM Endpoint Details**
- Full endpoint breakdown table: every URL path seen, call count, avg duration, and HTTP status

**Endpoint Errors**
- HTTP error codes > 299 across all SCCM IIS processes — categorical bar chart

---

## Deployment

### Option A — Dynatrace Extensions app (recommended, no SDK required)

The easiest way to deploy on Dynatrace SaaS is directly through the **Extensions** app in your tenant — no local toolchain needed.

1. In your Dynatrace tenant, open the **Extensions** app (search for it in the app launcher, or navigate to **Hub → Extensions**).
2. Click **Upload custom extension**.
3. Paste or upload `extension.yaml` directly — the platform handles signing automatically.
4. Once uploaded, click the extension name, then **Add monitoring configuration** to assign it to your SCCM hosts.

> This approach is the recommended path for SaaS tenants. It skips certificate management entirely and lets you iterate quickly by re-uploading a revised YAML whenever you make changes.

---

### Option B — Extension SDK (local build & sign)

Use this if you're on Dynatrace Managed, need to automate deployments via CI/CD, or want to manage signing certificates yourself.

```bash
# Install the Dynatrace Extension SDK if you haven't already
pip install dt-sdk

# Sign with your developer certificate
dt-sdk build
dt-sdk sign --certificate developer.pem --private-key developer.key
```

Upload the resulting `.zip` to your Dynatrace tenant under **Settings → Extensions → Upload extension**.

---

### 2. Configure monitoring

The extension uses WMI feature sets — deploy selectively based on which roles each server hosts:

| Feature set | WMI provider | Deploy to |
|---|---|---|
| `inbox` | `SMSINBOXMONITOR_SMSInbox` | Site server |
| `outbox` | `SMSMPFILEDISPATCHMANAGER_SMSOutbox` | Site server |
| `inventoryLoader` | `SMSINVENTORYDATALOADER_SMSInventoryDataLoader` | Site server |
| `softwareInventory` | `SMSSOFTWAREINVENTORYPROCESSOR_SMSSoftwareInventoryProcessor` | Site server |
| `statusManager` | `SMSSTATUSMANAGER_SMSStatusMessages` | Site server |
| `discoveryDataManager` | `SMSDISCOVERYDATAMANAGER_SMSDiscoveryDataManager` | Site server (e.g. VIRTSCM-CSS008) |
| `mpInventory` | `SmsMPInventory_SMSMPHinvMgr/SInvMgr/DdrMgr` | Management Point |
| `osMemory` | `PerfOS_Memory` | All SCCM servers |

The OneAgent on each target server must have the extension activated. Enable the relevant feature sets via the extension configuration in the Dynatrace UI.

### 3. Import the dashboards

1. In your Dynatrace tenant, navigate to **Dashboards** (or **Notebooks**).
2. Click **Import** and upload `SMS Dashboard.json`, then repeat for `SCCM - Reworked.json`.
3. On the main dashboard, set the `$HostGroups` variable to the host groups covering your SCCM servers.

> **IIS log prerequisites:** The client highlight and MP API tiles parse IIS W3C log entries. Ensure that Dynatrace Log Management is enabled on the IIS hosts and that the log format includes `cs-uri-stem`, `c-ip`, `cs(User-Agent)`, `sc-status`, and `time-taken` fields.

---

## How it works — data flow

```
SCCM Site / MP servers
  └── OneAgent (Windows)
        └── Extension 2.0 (WMI queries, every 1 min)
              └── Custom metrics → Dynatrace
  └── OneAgent Log Ingest
        └── IIS W3C logs → Dynatrace Log Management

Dynatrace
  ├── SMS Dashboard.json     (WMI metrics + SQL Server)
  └── SCCM - Reworked.json  (logs + metrics + Davis problems)
```

---

## Potential improvements

### Extension
- **Alerting** — Add `alerts` blocks to the extension YAML for inbox/outbox depth thresholds (e.g. alert when `file_current_count` for any inbox exceeds a baseline for >5 minutes).
- **Additional WMI counters** — Extend coverage with `SMS_SiteSystemSummarizer` for component availability, or `Win32_PerfFormattedData_SMSDPMONITOR_*` for Distribution Point transfer rates.
- **Version pinning** — Increment the extension `version` field on every change so the Dynatrace upgrade workflow tracks history properly.
- **Multiple instance support** — Some WMI classes are single-instance today; verify against environments that have multiple MPs to ensure the `Name` dimension is present where needed.

### Dashboards
- **Parameterise the hardcoded entity** — Tile 40 in the main dashboard references a hardcoded `PROCESS_GROUP_INSTANCE-716757DB0B7A4549` for SMS Exec network throughput. Replace this with a variable or entity selector filter so the dashboard is portable across tenants.
- **Client staleness alerting** — The "last seen > 21 days" queries are good candidates for a Davis metric event or custom alert so ops teams are notified without having to open the dashboard.
- **Sampling** — Most tiles run with `enableSampling: false`, which is accurate but expensive at scale. For trend tiles (not count SLOs), enabling sampling with a sensible ratio will reduce DPS consumption.
- **Synthetic monitors** — Add a Dynatrace HTTP monitor hitting the MP health endpoint (`/SMS_MP/.sms_aut`) so you get proactive uptime alerting on top of the reactive log analysis.
- **Log ingest optimisation** — The IIS filter (`dt.process_group.detected_name == "IIS" or ...`) scans all IIS logs. Narrowing to a specific `dt.entity.host_group` via the `$HostGroups` variable (already available) would cut log scan costs significantly.

---

## Requirements

- Dynatrace SaaS or Managed ≥ 1.303.0
- OneAgent deployed on all SCCM site servers and management points (Windows)
- Dynatrace Extension SDK (for local builds/signing)
- Log Management enabled for IIS on the monitored hosts
- SQL Server extension configured if you want the SQL tiles in SMS Dashboard
