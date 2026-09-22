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
