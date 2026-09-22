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

Modern Android forensic workflows utilize a two-stage approach to maintain evidence integrity without rooting the target device:

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
