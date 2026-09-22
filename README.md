# Mobile Forensics & Threat Hunting Lab: Android Triage with MVT & AndroidQF

A digital forensics and incident response (DFIR) case study documenting evidence acquisition, artifact parsing, and IOC triage on an Android device using Amnesty International's **Mobile Verification Toolkit (MVT)** and **Android Quick Forensics (androidqf)**.

---

## 1. Executive Summary & Threat Model

* **Target Device:** Oppo A78 (ColorOS / Android)
* **Host Environment:** Debian 12 (Bookworm) via Visual Studio Code
* **Acquisition Tool:** `androidqf` (Go acquisition agent)
* **Forensic Parser:** `mvt-android` (Python forensic analysis engine)
* **Threat Feeds:** STIX2 Indicators of Compromise (Amnesty International, Citizen Lab, AssoEchap)

### Purpose
Establish a repeatable, non-invasive digital forensics pipeline capable of identifying advanced persistent threats (such as Pegasus, Predator, and commercial spyware) while performing root-cause analysis on heuristic false positives and dual-use tracker alerts.

---

## 2. Forensic Architecture

Modern Android forensic workflows utilise a two-stage approach to maintain evidence integrity without rooting the target device:

1. **Acquisition (`androidqf`):** Deployed over an authenticated Android Debug Bridge (ADB) session to pull system properties, running services, crash dumps, and telephony backups into an isolated triage directory.
2. **Analysis (`mvt-android`):** An offline forensic engine that unpacks extracted databases, reconstructs SMS communication records, and evaluates package signatures against STIX2 threat intelligence feeds.

```text
+-------------------+       ADB (USB)       +-----------------------+
|  Oppo A78 Device  | --------------------> |  androidqf (Go)       |
|  (ColorOS)        |                       |  Acquisition Agent    |
+-------------------+                       +-----------------------+
                                                        |
                                                        v
                                             [ Acquired Artifacts ]
                                             - backup.ab (SMS/MMS)
                                             - dumpsys & logcat
                                             - packages & tombstones
                                                        |
                                                        v
+-------------------+                       +-----------------------+
|  STIX2 Threat IOCs| --------------------> |  mvt-android          |
|  (Amnesty / Echap)|                       |  Forensic Parser      |
+-------------------+                       +-----------------------+
                                                        |
                                                        v
                                             [ Structured Findings ]
                                             - Detections & Alerts
                                             - Triage & Verdict
```
## 3. Implementation Workflow

### Phase 1: Environment Setup (Debian 12)

Debian 12 enforces PEP 668 to prevent external Python packages from corrupting system libraries. Tooling was isolated inside a virtual environment with required hardware abstraction libraries:

```bash
# Install core dependencies and ADB
sudo apt update && sudo apt install -y python3-venv python3-pip adb libusb-1.0-0 libsqlite3-dev

# Initialise workspace and virtual environment
mkdir -p mobile-forensics-mvt-lab && cd mobile-forensics-mvt-lab
python3 -m venv .venv
source .venv/bin/activate

# Install MVT inside virtual environment
pip install --upgrade pip
pip install mvt
```

### Phase 2: Device Authorisation & Evidence Extraction

1. Enabled Developer Options and USB Debugging within ColorOS settings.
2. Connected the device over USB (File Transfer / MTP mode) and authorised the Debian host's RSA fingerprint prompt.
3. Acquired the standalone `androidqf` Linux x86_64 binary and set execution permissions:

```bash
wget https://github.com/mvt-project/androidqf/releases/download/v1.8.3/androidqf_linux_amd64_1.8.3 -O androidqf
chmod +x androidqf
```

4. Initiated live non-invasive acquisition targeting telephony databases, running processes, and diagnostic dumps:

```bash
./androidqf -output ./output/triage
```

5. Confirmed the unencrypted system backup request on the physical device screen to extract the SMS/MMS SQLite store (`com.android.providers.telephony`).

### Phase 3: Threat Intelligence Ingestion & Forensic Parsing

Loaded STIX2 threat signatures curated by international human rights researchers and parsed the acquired triage dump:

```bash
# Download latest community and research IOCs
mvt-android download-iocs

# Run forensic analysis on the triage archive
mvt-android check-androidqf --output ./output/reports/ ./output/triage/

# Reconstruct and inspect telephony backup records
mvt-android check-backup --output ./output/reports/ ./output/triage/backup.ab
```
---

## 4. Investigative Findings & Triage Analysis

The automated scan generated **52 Medium alerts** and **764 Critical alerts**. In Digital Forensics and Incident Response (DFIR), alert volume must be correlated with technical context to separate actual compromise from operational artifacts.

```text
╭────────────────────────────────────────── ALERTS ──────────────────────────────────────────╮
│ MVT produced 0 INFORMATIONAL alerts.                                                       │
│ MVT produced 0 LOW alerts.                                                                 │
│ MVT produced 52 MEDIUM alerts.                                                             │
│ MVT produced 0 HIGH alerts.                                                                │
│ MVT produced 764 CRITICAL alerts.                                                          │
╰────────────────────────────────────────────────────────────────────────────────────────────╯
```

### Finding 1: Dual-Use Software vs. Stalkerware (Critical Alerts)

* **Artifacts Flagged:** 
  * Application ID: `com.life360.android.safetymapd` (Life360)
  * SMS Invitation URL: `https://i.lf360.co/[REDACTED_TOKEN]`
* **Trigger Indicator Feed:** AssoEchap Stalkerware Indicators (`generated_stalkerware.stix2`)
* **Triage Analysis:** 
  The 764 Critical alerts stemmed entirely from background broadcast receivers, scheduled tasks, and incoming invitation links associated with Life360. Threat intelligence repositories track commercial family-locator apps alongside stalkerware due to shared technical capabilities (persistent background geolocation, geofencing, real-time telemetry). Because the application was intentionally installed by the device owner for family location sharing, this finding was confirmed as a **known operational artifact (Benign)**.

### Finding 2: Native Process Crashes & Tombstones (Medium Alerts)

* **Artifact Flagged:** 32 native crash tombstones for process `qcc-vendor` running under UID `1000` (system user).
* **Triage Analysis:** 
  MVT inspects crash tombstones because memory corruption exploits (e.g., zero-click baseband or media engine attacks) frequently cause transient segmentation faults. Chronological review of the tombstones across 2024–2025 revealed recurring crashes in Qualcomm's wireless vendor daemon (`qcc-vendor`). These crashes correlated with known OEM driver stability issues rather than exploitation attempts.

### Finding 3: Partition Mount Heuristics (Medium Alerts)

* **Artifact Flagged:** Partitions mounted with the `noatime` option (`/product/app`, `/product/lib64`, `/system_ext/...`).
* **Triage Analysis:** 
  MVT flags non-standard filesystem mount options to identify rootkits or modified system blocks. ColorOS applies `noatime` (disabling inode access-time writes) by default on read-only system partitions to minimise flash memory wear and improve I/O speed. This is standard OEM operating behavior.

---

## 5. Security Takeaways & Post-Analysis Hygiene

1. **Context Informs Severity:** Security tools identify observables, not human intent. An analyst must cross-reference IOC alerts against user authorisation and software function before escalating to an incident.
2. **Attack Surface Reduction:** Following evidence collection, USB Debugging and Developer Options were immediately disabled on the device to eliminate unauthorised ADB access vectors.
3. **Evidence Isolation & Data Privacy:** All raw forensic dumps (telephony databases, system logs, dumpsys output) were deleted from the local analysis workstation to prevent storing sensitive personal information (PII) once reporting concluded.
