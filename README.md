# Zabbix SMART Disk Discovery — Template & Dashboard

Fleet-wide SMART disk health monitoring for Zabbix 8.x. Two importable configs: a lightweight tripwire template for catching broken item discovery, and a multi-page dashboard for visualizing drive health, wear, and failures across every host in the environment.

Built and field-tested against a ~100+ node mixed Windows/Linux fleet with a variety of storage controllers (onboard SATA, Intel RST/CSMI passthrough, NVMe, USB bridges).

## Contents

| File | Type | Purpose |
|---|---|---|
| `SMART status.yaml` | Template | Fleet-wide tripwire for any item going "unsupported" on a host |
| `Smart Drive Health (Facility).yaml` | Dashboard | 5-page fleet SMART health dashboard |

## Requirements

- Zabbix 8.0+ (dashboard widget schema targets 8.0; template is compatible with 7.x agent2 SMART plugin)
- Zabbix agent 2 with the built-in SMART plugin enabled on monitored hosts
- Hosts linked to the stock **SMART by Zabbix agent 2** template (this repo's dashboard reads items from that template — it does not replace it)
- smartmontools 7.1+ on each monitored host, with `smartctl` reachable by the agent2 service (see notes below on known SMART plugin quirks)

## SMART status.yaml

A minimal custom template containing one item and one trigger:

- **Item:** `Unsupported items on {HOST.NAME}` — internal check (`zabbix[host,,items_unsupported]`), polled every 5 minutes
- **Trigger:** fires (Warning) when the unsupported-item count on a host is greater than 0

This is intentionally decoupled from the stock SMART template so it survives future SMART template updates/re-imports without conflict. It's a general "something on this host stopped collecting" tripwire — not SMART-specific — but in practice it's the fastest way to catch a dead or broken SMART discovery rule before it goes silent for months.

**Import:** *Data collection → Templates → Import* → link to your host group (e.g. "Discovered hosts") or mass-link to all monitored hosts.

> **Why not use a built-in Internal Action instead?** Zabbix supports notifying on unsupported items natively via *Alerts → Actions → Internal actions* with zero template/item overhead. That's a good option for email/notification-only alerting. This template exists specifically because Internal Action events do **not** populate `Monitoring → Problems`, the Problems dashboard widget, or Top Hosts widgets — this template does, which is what the dashboard below depends on.

## Smart Drive Health (Facility).yaml

A 5-page dashboard, filtered to hosts tagged `class:storage` (and a subset tagged `subclass:monitoring` under the "Discovered hosts" group). Adjust these tags/group to match your own host tagging scheme before import.

### Page 1 — Fleet Health
- **Host / SMART status issues** — Top Hosts view of unsupported-item count (from `SMART status.yaml` above), agent availability, agent version, and system description per host
- **Drive Temperature (all devices)** — current temperature across every discovered device path in the fleet

### Page 2 — NVMe Wear
- **NVMe Endurance Used %** — SSD wear-leveling percentage (higher = closer to end of life)
- **NVMe Media Errors**
- **NVMe Critical Warning** — raw NVMe critical warning bitfield

### Page 3 — SATA / HDD Health
- **Self-test Passed (0 = FAILED)**
- **Reported Uncorrectable Errors**

### Page 4 — Inventory & Collection
- **Device Model** — quick fleet-wide hardware inventory by drive
- **smartctl Exit Status (0 = OK)** — surfaces per-device smartctl exit codes; non-zero values indicate a device that may need investigation (see notes below)
- **Power Cycle Count**

### Page 5 — Storage Problems
- **Active Storage Problems** — live Problems widget filtered to `class:storage` tag
- **SMART - Actionable Failures** — Problems widget scoped to the "Discovered hosts" group, filtered to `subclass:monitoring` tag

**Import:** *Dashboards → All dashboards → Import*.

## Device path naming

Widgets reference specific discovered device paths (`sda`, `sdb`, `sdc`, `nvme0`, `sda sat`, and several `csmi0,N` paths from Intel RST/RAID controller passthrough). These reflect the actual device paths discovered across our fleet's hardware mix — yours will likely differ. After import, check *Monitoring → Latest data* on a representative host to confirm your discovered device names, then edit each Top Hosts widget's column `item` values to match.

## Known SMART plugin caveats

A few fleet-wide issues worth knowing before deploying broadly:

- **Windows agent2 needs an explicit smartctl path.** The `Plugins.Smart.Path` setting in `smart.conf` is often left commented out by default — set it explicitly to your `smartctl.exe` path rather than relying on `%PATH%`, since the Agent 2 service does not reliably inherit interactive-session PATH changes.
- **RAID-mode NVMe (Intel RST/VMD) will not report native NVMe SMART data.** Drives behind RST in RAID mode surface as a generic ATA/CSMI passthrough device and will fail `-d nvme` queries outright. This is a driver-stack limitation, not a plugin bug — switching the controller to AHCI/NVMe passthrough mode in BIOS is the only real fix.
- **A confirmed upstream agent2 bug** (strict exit-status handling in the SMART plugin discarding valid SMART JSON on non-zero smartctl exit codes — [ZBX-26359](https://support.zabbix.com/browse/ZBX-26359)) can cause `smart.disk.discovery` to silently drop *all* devices on a host when just one device errors (e.g., a dead optical drive or unsupported USB bridge). A fix is in progress upstream as of this writing; until it ships, affected hosts may show incomplete or empty SMART discovery despite having healthy drives.
- **Discovery can silently time out on hosts with several devices, even when your server config looks fine.** Zabbix 8.0 has two separate timeout settings that are easy to conflate:
  1. The legacy `Timeout` value in `zabbix_server.conf`
  2. A newer, separate set of per-item-type timeouts under **Administration → General → Timeouts** (introduced in 7.0+)

  The discovery rule inherits from #2, not #1 — raising `zabbix_server.conf`'s `Timeout` alone does nothing for it. The frontend's default **Zabbix agent** item-type timeout is only **3s**, which is too short once `smart.disk.discovery` has to enumerate several real devices plus RAID/CSMI passthrough paths (observed taking ~4.8s on a host with 6+ discovered paths). The discovery rule's config page will show `Timeout: Global: 3s` when this is the cause.

  **Fix:** *Administration → General → Timeouts* → raise the **Zabbix agent** item-type timeout to 15–30s → Update. This is a global setting, so it fixes the issue fleet-wide in one change rather than per-host.

## License

No license file included yet — treat as all-rights-reserved until one is added. Open an issue if you'd like a specific license applied.
