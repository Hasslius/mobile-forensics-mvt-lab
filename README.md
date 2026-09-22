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

# Initialize workspace and virtual environment
mkdir -p mobile-forensics-mvt-lab && cd mobile-forensics-mvt-lab
python3 -m venv .venv
source .venv/bin/activate

# Install MVT inside virtual environment
pip install --upgrade pip
pip install mvt
```

### Phase 2: Device Authorisation & Evidence Extraction

1. Enabled Developer Options and USB Debugging within ColorOS settings.
2. Connected the device over USB (File Transfer / MTP mode) and authorized the Debian host's RSA fingerprint prompt.
3. Acquired the standalone `androidqf` Linux x86_64 binary and set execution permissions:

```bash
wget [https://github.com/mvt-project/androidqf/releases/download/v1.8.3/androidqf_linux_amd64_1.8.3](https://github.com/mvt-project/androidqf/releases/download/v1.8.3/androidqf_linux_amd64_1.8.3) -O androidqf
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

