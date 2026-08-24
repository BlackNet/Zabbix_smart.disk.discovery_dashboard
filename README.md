# Zabbix SMART Fleet Monitoring

Fleet-wide drive health monitoring for Zabbix 8.x — importable templates and a dashboard that discovers every drive on every host without naming any of them.

Built and field-tested against a ~100 node mixed Windows/Linux fleet with a variety of storage controllers (onboard SATA, Intel RST/CSMI passthrough, NVMe, USB bridges).

---

## Quick start

1. **Import the templates** — *Data collection → Templates → Import*
   `SMART status.yaml`, then `NVMe Extended SMART.yaml`
2. **Import the dashboard** — *Dashboards → All dashboards → Import*
   `Smart Drive Health (Auto).yaml`
3. **Drop the UserParameter config** onto your agents (see [Agent configuration](#agent-configuration))
4. **Link the templates** to your hosts alongside the stock *SMART by Zabbix agent 2*

Leave **Delete missing** unchecked on every import.

Nothing needs editing to match your hardware. Discovery is tag-driven — see below.

---

## Contents

| File | Type | Purpose |
|---|---|---|
| `SMART status.yaml` | Template | Tripwire for any item going "unsupported" on a host |
| `NVMe Extended SMART.yaml` | Template | The NVMe health-log fields the agent2 SMART plugin discards |
| `Smart Drive Health (Auto).yaml` | Dashboard | 7-page tag-driven fleet dashboard |
| `smart_ext_windows.conf` | Agent config | UserParameters, Windows |
| `smart_ext_linux.conf` | Agent config | UserParameters, Linux (+ sudoers) |

### Requirements

- Zabbix 8.0+ — the dashboard uses the Honeycomb widget schema
- Zabbix agent 2 with the built-in SMART plugin
- Hosts linked to the stock **SMART by Zabbix agent 2** template — this repo reads its items, it does not replace it
- smartmontools 7.1+ with `smartctl` reachable by the agent service

---

## Design: no device names anywhere

Earlier revisions of this dashboard listed device paths explicitly — a column per `sda`, `sdb`, `csmi0,0`, `nvme0`, and so on. That does not survive contact with a real fleet. An audit of ours found 12 distinct device names across 65 hosts, roughly 140 hand-maintained widget columns, six omissions and two duplicates, and three devices with no coverage at all. Every new drive meant editing eleven widgets.

The current dashboard uses **Honeycomb widgets matched on item-name patterns plus item tags**. Zabbix's SMART templates tag every per-drive item with `diskname`, `disktype` and `component` at discovery time, so a new drive in any workstation appears on its own. There's no host group filter either, which means the same file imports unmodified onto more than one Zabbix instance.

**Patterns are anchored as `*]: Metric`.** Every SMART item is named `[device]: Metric`, and that closing bracket is load-bearing — Zabbix matches item patterns **case-insensitively**, so a bare `*Write rate` also matches the OS disk template's `Disk write rate` (units `w/s`), pulling several unrelated cells per host into the widget. If you add panels, keep the anchor.

---

## Dashboard pages

Pages are grouped by **what kind of number** a panel holds, not by topic. A red cell tells you which class of problem you have before you've read the title.

| Page | Panels | A red cell means |
|---|---|---|
| **Fleet Health** | 2 | collection is broken, not the drive |
| **Failure Signals** | 8 | **replace this drive** |
| **Workload** | 5 | nothing — context only |
| **TBW Wear Forecast** | 7 | raise a purchase order |
| **Temperature** | 2 | fix cooling |
| **Aging & Inventory** | 5 | mechanical wear, plus model/firmware |
| **Storage Problems** | 2 | active triggers |

**Failure Signals** is the page that matters operationally: available spare, spare headroom, critical warning, media errors, error log entries, self-test, and — for SATA and CSMI drives — reallocated sectors and reported uncorrectable. All measured, no macros involved.

**TBW Wear Forecast** is the only page whose accuracy depends on configuration. Every panel divides by `{$NVME.TBW}`. Get that macro wrong and this page is wrong while the other six stay trustworthy — which is why it's isolated.

---

## SMART status.yaml

One item and one trigger:

- **Item** `zabbix[host,,items_unsupported]` — unsupported-item count per host, every 5 minutes
- **Trigger** — Warning when the count exceeds 0

Deliberately decoupled from the stock SMART template so it survives upstream re-imports. Not SMART-specific, but in practice it's the fastest way to catch a dead discovery rule before it goes quiet for months.

> **Why not an Internal Action?** Zabbix can notify on unsupported items natively via *Alerts → Actions → Internal actions*. But Internal Action events don't populate *Monitoring → Problems*, the Problems widget, or Top Hosts — this template does, and the dashboard depends on that.

**Calibrate before trusting it.** Our fleet carried 495 unsupported items across 78 hosts before tuning, nearly all ATA attribute items auto-created for devices that don't report them. At that level the alert is noise and a genuinely broken collector never stands out. Get the baseline to zero first.

---

## NVMe Extended SMART.yaml

The bundled agent 2 SMART plugin parses **five** fields out of the NVMe health log — `temperature`, `power_on_hours`, `critical_warning`, `media_errors`, `percentage_used` — and discards the rest during unmarshalling. Notably absent:

- `available_spare` and `available_spare_threshold` — the actual failure predictor
- `data_units_written` — the only field that supports a wear forecast
- `unsafe_shutdowns`, `num_err_log_entries`, thermal throttling counters

They're missing from `smart.disk.get` output entirely, so no dependent item or JSONPath can recover them. This template collects them via two UserParameters.

**One master item per drive.** `smartctl -j -a` runs once per poll and every other item is a dependent JSONPath off it — 22 items per drive, one `smartctl` invocation. Avoid the pattern of one direct-polling item per metric; at 25 drives that's 250 invocations per interval instead of 25.

**Discovery filters on `protocol == "NVMe"`** from `smartctl --scan -j`, not on the device name. NVMe presents as `nvme0` on one host and `sdb` on another; both are found with no per-host configuration.

### Schedule

| Item | Interval |
|---|---|
| `smart.ext.scan` (discovery) | `1h` |
| `smart.ext.raw` (master) | `5m` |
| dependent items | inherit master |
| endurance / projections | `10m` – `1h` |

**Check `lifetime` before deploying.** It ships at `7d`, meaning a drive that stops being discovered has its items *and history* deleted a week later. For wear tracking you usually want to keep a dead drive's record — `30d` or `0` is more appropriate.

### Setting `{$NVME.TBW}`

Rated endurance in TB, per drive, as a context macro:

```
{$NVME.TBW:"nvme0"} = 1200
{$NVME.TBW:"nvme1"} = 1200
```

Default is a deliberately conservative 600. Consumer TLC NVMe is near-universally **600 TBW per TB** (Samsung 980/990 PRO, WD SN850, Crucial P5), so capacity × 600 is a good first guess. Enterprise drives at 1–3 DWPD are far higher.

**The drive's own `percentage_used` is optimistic.** A Samsung 980 PRO 2TB reporting 30% used is at 39.7% of its 1200 TBW rating; a 990 PRO reporting 23% is at 39.5%. The dashboard shows both figures side by side for exactly this reason. Alerting on the controller's number alone fires up to 16 points late.

---

## TBW is a planning figure, not a failure prediction

Worth stating plainly, because the page name invites the opposite reading.

TBW is a warranty commitment. Drives routinely run well past it — endurance testing has documented consumer SSDs reaching 25× their rating. Google's fleet study ([Schroeder et al., FAST '16](https://www.usenix.org/conference/fast16/technical-sessions/presentation/schroeder)) found **drive age predicts failure better than write wear**, and that consumer MLC was about as reliable as enterprise SLC.

What actually kills drives, in rough order: firmware defects, sudden controller or electrical failure, and only then wearout. Wearout is the one you get warning for, and that warning is on **Failure Signals** — `available_spare` falling, `media_errors` climbing — not on the forecast page.

Use TBW to replace on a Tuesday with a spare in hand. Don't read "80% endurance" as "about to fail".

### Mirrors: check whether your wear is correlated

Two drives in a mirror take identical writes. If they're the same capacity and rating they reach end-of-endurance in the same week, which defeats the point of the mirror. Differing `percentage_used` values are not evidence of a stagger — controller generations estimate differently. Compare `bytes_written ÷ rated TBW`, not the reported percentage.

---

## Agent configuration

Two UserParameters per host: a discovery scan and a raw JSON fetch.

**Windows** — `smart_ext_windows.conf` goes in `zabbix_agent2.d\plugins.d\`, which is the directory the MSI's `Include` line actually covers. `UserParameter` is a normal config directive; the `plugins.d` name is convention, not enforcement. Placing it in the parent directory looks tidier and is never read.

Using the 8.3 short path (`C:\PROGRA~1\smartmontools\bin\smartctl.exe`) avoids argument-quoting problems, at the cost of depending on 8dot3 name creation being enabled on the volume.

**Linux** — `smart_ext_linux.conf` goes in the agent's include directory. Verify that `Include` resolves to an absolute path; some packages ship `Include=./zabbix_agent2.d/*.conf`, which resolves against the working directory rather than the config file, and others only include `plugins.d`. NVMe admin commands need root, so the sudoers drop-in at the bottom of that file is required — install it `0440`, root:root, and validate with `visudo -cf` before restarting anything.

**Verify before involving the server:**

```
zabbix_agent2 -c <conf> -t smart.ext.scan
```

That parses the whole config in-process and returns the device JSON, so a bad path or a missing sudoers rule surfaces immediately rather than as an unsupported discovery rule days later.

---

## Known caveats

- **Windows agent2 needs an explicit smartctl path.** `Plugins.Smart.Path` in `smart.conf` is commented out by default. Set it — the service does not reliably inherit interactive-session `PATH`.

- **RAID-mode NVMe (Intel RST/VMD) won't report native NVMe SMART.** Drives behind RST in RAID mode surface as generic ATA/CSMI passthrough and fail `-d nvme` outright. Driver-stack limitation; AHCI/NVMe passthrough in BIOS is the only real fix.

- **[ZBX-26359](https://support.zabbix.com/browse/ZBX-26359)** — strict exit-status handling in the agent2 SMART plugin discards valid SMART JSON on non-zero `smartctl` exit. One bad device (dead optical drive, unsupported USB bridge) silently drops *all* devices on that host.

- **Discovery times out at the default 3s agent timeout.** Zabbix 8.0 has two timeout settings that are easy to conflate: the legacy `Timeout` in `zabbix_server.conf`, and per-item-type timeouts under *Administration → General → Timeouts*. Discovery inherits the **second**. `smart.disk.discovery` takes ~4.8s on a host with 6+ paths. The rule's config page shows `Timeout: Global: 3s` when this is the cause — raise the **Zabbix agent** item-type timeout to 15–30s, once, fleet-wide.

- **`timeleft()` returns `DBL_MAX` on a flat slope**, which overflows the `unixtime` formatter and breaks the widget with *Number out of range*. Preprocessing can't undo a value already in history. This repo computes projections from the measured write rate instead — there are no regression functions anywhere in the template.

- **Zabbix matches widget item patterns case-insensitively.** Covered above; it's the single easiest way to silently pull unrelated items into a panel.

---

## License

No license file yet — treat as all rights reserved until one is added. Open an issue if you'd like a specific license applied.
