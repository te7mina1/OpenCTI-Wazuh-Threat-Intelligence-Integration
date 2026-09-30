# OpenCTI + Wazuh Threat Intelligence Integration

<!---

![OpenCTI](https://img.shields.io/badge/CTI-OpenCTI-blue?style=for-the-badge)
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh-4B275F?style=for-the-badge)
![Linux](https://img.shields.io/badge/Platform-Linux-black?style=for-the-badge&logo=linux&logoColor=white)
![Python](https://img.shields.io/badge/Language-Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![TAXII](https://img.shields.io/badge/Protocol-TAXII%202.1-orange?style=for-the-badge)
![STIX](https://img.shields.io/badge/CTI-STIX%202.1-lightgrey?style=for-the-badge)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-T1110.001-red?style=for-the-badge)

-->

## Project Overview

This project integrates **OpenCTI** with **Wazuh** to add threat intelligence context to SIEM detections.

The objective was not simply to collect threat intelligence, but to make that intelligence operational by connecting:

**OpenCTI → TAXII 2.1 → Python Enrichment → Wazuh → CTI-Enriched Alert**

The final detection use case focused on **SSH Password Guessing** against an Ubuntu endpoint.

## Objectives

- Deploy and configure OpenCTI in a virtual lab.
- Integrate multiple external threat intelligence feeds.
- Create a filtered OpenCTI indicator collection.
- Distribute indicators through TAXII 2.1.
- Build a Python-based CTI enrichment pipeline.
- Integrate the enrichment workflow with Wazuh.
- Improve an existing SSH password-guessing detection with CTI context.
- Validate the improved detection using controlled test activity.
- Document the architecture, implementation, challenges, and results.

## Architecture

The lab uses OpenCTI as the threat intelligence platform and Wazuh as the SIEM and detection platform.

```text
External CTI Feeds
        |
        v
     OpenCTI
        |
        | TAXII 2.1
        v
Python CTI Collector
        |
        v
Local Indicator Cache
        |
        v
Wazuh Integration
        |
        v
SSH Detection Rule
        |
        v
CTI-Enriched Wazuh Alert
```

## Threat Intelligence Feeds

Two feeds were used in the final implementation:

| Feed | Purpose | Result |
| --- | --- | --- |
| AlienVault OTX | Broad community threat intelligence | Successfully ingested |
| ThreatFox / Abuse.ch | Malware and IOC intelligence | Successfully ingested |
| VirusTotal Livehunt | Additional feed explored during development | Excluded due to API access restriction |

Both selected feeds successfully completed their connector operations in OpenCTI.

## OpenCTI Collection

A dedicated collection was created for downstream Wazuh enrichment:

```text
Wazuh-CTI-Indicators
```

The collection was configured for indicator objects involving:

- IPv4 addresses
- Domains
- File indicators

OpenCTI scores were copied to the indicator confidence field.

The collection was exposed through a TAXII 2.1 endpoint for consumption by the enrichment pipeline.

## CTI Enrichment Pipeline

The CTI consumer was developed under:

```text
/opt/opencti-wazuh
```

Main collector:

```text
/opt/opencti-wazuh/cti_enrichment.py
```

The collector performs the following:

1. Connects to the OpenCTI TAXII collection.
2. Retrieves STIX indicator objects.
3. Handles TAXII pagination.
4. Extracts IPv4, domain, and SHA-256 indicators.
5. Stores the indicators locally.
6. Makes the intelligence available to the Wazuh integration.

The final local cache contained approximately 14,212 indicators:

- 5,173 SHA-256 indicators
- 2,658 IPv4 indicators
- 6,381 domain indicators

## Wazuh Integration

Custom Wazuh integration scripts were used to inspect alerts and perform CTI lookups.

The SSH enrichment integration was:

```text
/var/ossec/integrations/custom-opencti-ssh
```

The workflow was:

```text
SSH Authentication Failures
            |
            v
      Wazuh Rule 100104
            |
            v
   custom-opencti-ssh
            |
            v
   OpenCTI Local Cache
            |
       +----+----+
       |         |
    No Match   Match
       |         |
       v         v
 Baseline     Enriched
   Alert       Event
                  |
                  v
           Wazuh Rule 100105
                  |
                  v
        CTI-Enriched Alert
```

## Detection Use Case

The selected detection came from an earlier detection engineering project:

**SSH Password Guessing**

### Before: Baseline Detection

The original Wazuh rule was:

```text
Rule: 100104
Level: 10
MITRE ATT&CK: T1110.001 – Password Guessing
```

The rule detects:

- 5 failed authentication attempts
- From the same source IP
- Within 120 seconds

The baseline detection identified the suspicious behavior but did not contain external threat intelligence context.

### After: CTI-Enriched Detection

A second Wazuh rule was introduced:

```text
Rule: 100105
Level: 12
```

When the source IP from the SSH detection matched an indicator in the OpenCTI-derived cache, the event was enriched before being ingested back into Wazuh.

The resulting alert included:

- CTI match status
- Matched indicator
- Indicator type
- OpenCTI confidence
- CTI labels
- STIX indicator ID
- Original Wazuh rule ID
- Original alert level
- Source IP
- Destination IP
- Enrichment timestamp

## Validation

A controlled SSH password-guessing test was performed against the Ubuntu endpoint.

### Baseline

```text
Rule:       100104
Level:      10
Technique:  T1110.001
```

### CTI-Enriched Result

```text
Rule:       100105
Level:      12
CTI Match:  true
Indicator:  213.111.185.108
Confidence: 60
```

The enriched alert also preserved:

```text
original_rule_id = 100104
original_level   = 10
```

This demonstrated that the original detection was preserved while CTI context was added as a second-stage enrichment layer.

> The demonstrated improvement is contextual enrichment and alert prioritization. A measured false-positive reduction was not performed, so the project does not claim that false positives were eliminated.

## Sample Intelligence

The same enrichment pipeline was tested with different IOC types:

| Type | Example | Result |
| --- | --- | --- |
| IPv4 | `104.249.10.86` | Matched |
| Domain | `web-analyzer-serv32.com` | Matched |
| SHA-256 | `138a4c9cd617912c2269fae64b6b12d57e926a36c2c62f25ee05a32cdf102212` | Matched |
| IPv4 | `213.111.185.108` | Used for SSH enrichment validation |

These tests demonstrated that the enrichment workflow was not limited to a single IOC type.

## MITRE ATT&CK Mapping

The applied detection maps to:

**T1110.001 – Password Guessing**

The CTI enrichment provides additional information about the source infrastructure.

The project does not assume that every CTI label automatically represents an ATT&CK technique directly associated with the observed event.

A future implementation could retrieve OpenCTI relationship data to provide additional actor, campaign, and ATT&CK context.

## Evidence

The project documentation contains screenshots demonstrating:

- OpenCTI feed ingestion
- AlienVault OTX connector completion
- ThreatFox connector completion
- OpenCTI indicator collection
- TAXII endpoint validation
- CTI cache contents
- Baseline Wazuh SSH detection
- OpenCTI enrichment output
- CTI-enriched Wazuh alert
- Before/after detection behavior

**Several technical issues were encountered during implementation.**

<!---
Challenges Encountered

Several technical issues were encountered during implementation.

Wazuh Integration Configuration

An initial integration configuration caused a Wazuh analysisd configuration error. The configuration was corrected and the Wazuh manager was successfully restarted.

CTI Cache Structure

The OpenCTI-derived cache was initially treated as a dictionary while the actual cache structure was a list of indicator objects.

The lookup logic was corrected to iterate through the indicator list.

Integration Output File

The enrichment integration initially failed to create its output file because the script terminated before writing the event.

After correcting the script, the output file was successfully created and consumed by Wazuh.

Manual Integration Testing

The integration script expects the Wazuh alert JSON as a command-line argument. An earlier test attempted to provide the alert through standard input, which did not produce the expected result.

The test method was corrected.

Feed Access

VirusTotal Livehunt was explored as an additional intelligence source but could not be included because of API access restrictions.

CTI Relationship Data

The current TAXII collection provides indicator objects. It does not automatically provide all OpenCTI relationship information such as actors, campaigns, or ATT&CK relationships.

-->

## Lessons Learned

The main lesson from this project was that collecting threat intelligence is only one part of an operational CTI workflow.

Threat intelligence becomes useful when it can:

1. Reach the SIEM.
2. Match observable data in an alert.
3. Add meaningful context.
4. Preserve the original detection.
5. Help an analyst investigate and prioritize the alert.

The project also demonstrated the importance of validating each stage independently when troubleshooting an integration pipeline.

> There will be 'Future Improvements' with additional development time, the project could be extended.


# Technologies Used
| Technology | Role |
|----------- | ---- |
| OpenCTI|	Threat intelligence platform|
| Wazuh|	SIEM and detection platform|
| Python|	CTI collection and enrichment|
| TAXII 2.1|	Threat intelligence distribution|
| STIX 2.1|	Threat intelligence data model|
| AlienVault OTX|	Threat intelligence feed|
| ThreatFox / Abuse.ch|	Threat intelligence feed|
| MITRE ATT&CK|	Detection technique mapping|
| Hydra|	Controlled detection testing|
| Linux / Ubuntu|	Endpoint and integration environment|

## Project Outcome

The completed project demonstrated an operational CTI-to-SIEM workflow:

```
Threat Intelligence
        ↓
     OpenCTI
        ↓
     TAXII 2.1
        ↓
 Python CTI Collector
        ↓
  Local Indicator Cache
        ↓
      Wazuh
        ↓
 Detection Rule
        ↓
 CTI Enrichment
        ↓
 Analyst Context
```

The project moved beyond simply collecting indicators and demonstrated how threat intelligence can be connected to an existing detection workflow to provide additional context during alert triage.

## Author

Randy

Cybersecurity - SOC Analyst - Threat Intelligence