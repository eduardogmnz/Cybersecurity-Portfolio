# SOC Incident Investigation Report

## Incident Overview

This report documents the investigation of a simulated credential-access incident within an isolated cybersecurity home lab. The activity originated from a controlled Kali Linux system at `192.168.56.30` and targeted the Windows 11 endpoint `WIN11-LAB` at `192.168.56.20`.

The investigation focused on repeated failed SMB authentication attempts against the `labadmin` account. Windows Security logs, Sysmon telemetry, and Wazuh SIEM alerts were analyzed and correlated to determine the source of the activity, authentication method, targeted account, detection behavior, and whether the activity resulted in successful access or subsequent compromise.

Wazuh initially generated individual failed-authentication alerts under Rule `60122`. After eight failed authentication events from the same source IP occurred within the configured correlation window, Wazuh generated Rule `60204` at Level 10 for **Multiple Windows Logon Failures**, mapped by Wazuh to MITRE ATT&CK technique `T1110 - Brute Force`.

Investigation of Windows Security Event ID `4625` and Sysmon Event ID `3` confirmed repeated NTLM network authentication attempts over SMB/TCP 445 from `192.168.56.30`. The eight authentication events were correlated with their corresponding network connections using matching source IP addresses and TCP source ports.

No successful remote authentication from the controlled source was observed within the investigated time window. Review of subsequent Sysmon process-creation and file-creation telemetry did not identify evidence that the failed authentication activity progressed to remote execution or payload deployment.

## Environment and Data Sources

The investigation was performed within an isolated VirtualBox cybersecurity lab designed to simulate a small SOC monitoring environment.

### Systems

| System | Role | SOC-LAB IP Address |

|---|---|---|

| `Kali-Lab` | Controlled attack simulation system | `192.168.56.30` |

| `WIN11-LAB` | Monitored Windows 11 endpoint | `192.168.56.20` |

| `Wazuh-Lab` | Wazuh SIEM manager, indexer, and dashboard | `192.168.56.10` |

| `CYBERHOST1` | SOC analyst workstation / VirtualBox host | `192.168.56.1` |

The systems communicated through the isolated `192.168.56.0/24` SOC-LAB host-only network. Internet connectivity, when required for software installation and updates, was provided separately through NAT interfaces.

### Security Monitoring

The Windows 11 endpoint was configured with the Wazuh Agent and Sysmon. Windows process-creation auditing and command-line logging were also enabled to provide additional endpoint visibility.

The investigation used the following telemetry sources:

- **Windows Security Event Log** — authentication events, including Event ID `4625` for failed logons and Event ID `4624` for successful logons.

- **Sysmon Event ID 3** — network connection telemetry used to correlate inbound SMB connections with authentication attempts.

- **Sysmon Event ID 1** — process-creation telemetry reviewed for evidence of subsequent execution.

- **Sysmon Event ID 11** — file-creation telemetry reviewed for evidence of payload or file deployment.

- **Wazuh alerts** — individual and correlated security alerts generated from endpoint telemetry.

- **PowerShell Operational Log** — Event ID `4104` Script Block Logging, later enabled and validated as part of telemetry improvement testing.

Wazuh and Windows used different time-display contexts during portions of the investigation. Windows endpoint evidence was primarily reviewed in Central Daylight Time (CDT), while raw Wazuh and Windows event data also contained UTC timestamps. Time-zone context was therefore considered during event correlation.

## Detection and Initial Triage

The investigation began by evaluating whether activity generated from `Kali-Lab` could be detected and investigated through the telemetry available on `WIN11-LAB` and Wazuh.

An initial Nmap SYN scan was performed from `192.168.56.30` against the Windows endpoint. The scan did not produce a corresponding Wazuh alert. This was documented as a visibility limitation rather than evidence that the scan did not occur. The monitoring architecture was primarily endpoint-based and did not include a dedicated network intrusion detection sensor capable of independently observing all network reconnaissance traffic.

Subsequent testing focused on SMB authentication over TCP port 445. A controlled failed authentication attempt against the `labadmin` account generated Windows Security Event ID `4625` and was detected by Wazuh Rule `60122`, **Logon Failure - Unknown user or bad password**, at Level 5.

The failed authentication event contained several fields useful for triage:

- Source IP: `192.168.56.30`

- Source system: `KALI`

- Target system: `WIN11-LAB`

- Target account: `labadmin`

- Logon Type: `3` (Network)

- Authentication package: `NTLM`

- Status: `0xc000006d`

- Substatus: `0xc000006a`

- Windows Event ID: `4625`

- Wazuh Rule: `60122`

The `0xc000006a` substatus indicated that the account existed but the supplied password was incorrect. The network logon type and source information were consistent with the controlled SMB authentication activity generated from Kali.

A small number of repeated failures initially continued to generate individual Rule `60122` alerts without producing a higher-severity correlated alert. Rather than treating this immediately as a detection failure, the Wazuh ruleset was examined to determine whether correlation logic already existed.

## Detection Logic Investigation

Review of the active Wazuh configuration confirmed that no custom Project 008 correlation rule had been created. The local rules file contained only the existing default/example configuration.

The Wazuh built-in Windows security ruleset was then examined. Rule `60122` was confirmed as the rule responsible for detecting individual Windows failed-logon events.

Further investigation identified built-in Wazuh Rule `60204`, which correlates multiple events belonging to the `authentication_failed` group when they originate from the same source IP address.

The relevant detection behavior was determined to be:

- Rule ID: `60204`

- Description: **Multiple Windows Logon Failures**

- Alert level: `10`

- Required frequency: `8`

- Correlation timeframe: `240` seconds

- Correlation field: source IP address

- MITRE ATT&CK mapping: `T1110 - Brute Force`

This explained why the earlier sequence of three failed authentication attempts produced individual alerts but did not generate the higher-level correlation alert. The activity had not reached the built-in threshold of eight matching failures within the configured time window.

This finding corrected the preliminary assumption that repeated authentication failures represented a correlation gap. The detection capability already existed; the original test simply did not satisfy its threshold.

No redundant custom Wazuh rule was created. Instead, the existing detection logic was validated through controlled testing.

## Correlated Authentication Investigation

To validate Wazuh Rule `60204`, eight controlled failed SMB authentication attempts were generated from `Kali-Lab` against the `labadmin` account on `WIN11-LAB`.

Windows Security telemetry recorded eight Event ID `4625` authentication failures between approximately `14:56:44` and `14:57:25` CDT on September 22, 2026.

All eight events shared the primary characteristics expected from the controlled activity:

- Source IP `192.168.56.30`

- Target account `labadmin`

- Authentication package `NTLM`

- Logon Type `3`

- Status `0xc000006d`

- Substatus `0xc000006a`

The eight failures occurred over approximately 41 seconds, satisfying the Wazuh correlation threshold.

Wazuh subsequently generated Rule `60204`, **Multiple Windows Logon Failures**, at Level 10. The alert recorded a frequency of eight and mapped the activity to MITRE ATT&CK technique `T1110 - Brute Force`.

The final Windows failed-logon event in the sequence was Event Record ID `111962`, with a Windows system timestamp of `2026-09-22T19:57:25.3216454Z`. The correlated Wazuh alert carried a timestamp of `2026-09-22T19:58:12.281+0000`.

The difference between the endpoint event time and Wazuh alert timestamp was preserved during analysis rather than treating the timestamps as identical.

The resulting detection chain was:

`Repeated SMB authentication failures → Windows Security 4625 → Wazuh Rule 60122 → correlation threshold reached → Wazuh Rule 60204`

This demonstrated that Wazuh could escalate multiple individually lower-severity authentication failures into a higher-severity correlated alert when the configured threshold was satisfied.

## Endpoint and Network Correlation

The authentication events were correlated with Sysmon network telemetry to determine whether the failed logons could be tied to the expected SMB network connections.

Sysmon Event ID `3` recorded inbound TCP connections from `192.168.56.30` to `192.168.56.20` on destination port `445`. The events showed the Windows `System` process receiving the connections with `Initiated: false`, consistent with the monitored Windows endpoint acting as the receiving system.

The TCP source ports recorded by Sysmon matched the source ports contained in the corresponding Windows Security Event ID `4625` records:

| Attempt | Security 4625 Time | Source Port | Sysmon Event 3 Time | Destination |

|---|---|---:|---|---|

| 1 | 14:56:44 | `50220` | 14:56:46 | `192.168.56.20:445` |

| 2 | 14:56:50 | `50228` | 14:56:52 | `192.168.56.20:445` |

| 3 | 14:56:56 | `53304` | 14:56:58 | `192.168.56.20:445` |

| 4 | 14:57:03 | `53308` | 14:57:05 | `192.168.56.20:445` |

| 5 | 14:57:09 | `33548` | 14:57:10 | `192.168.56.20:445` |

| 6 | 14:57:14 | `33550` | 14:57:17 | `192.168.56.20:445` |

| 7 | 14:57:20 | `33554` | 14:57:22 | `192.168.56.20:445` |

| 8 | 14:57:25 | `33564` | 14:57:27 | `192.168.56.20:445` |

The matching source IP addresses and TCP source ports provide strong correlation between the network connections and the failed authentication events.

Together, the evidence establishes the following sequence:

`Kali-Lab (192.168.56.30)` → `TCP/445 SMB` → `WIN11-LAB (192.168.56.20)` → `NTLM network authentication` → `labadmin` → `incorrect password` → `Windows Event 4625` → `Wazuh detection`

Nearby Sysmon network events involving UDP port 137 were also observed. These events were not automatically attributed to the SMB authentication sequence because temporal proximity alone was insufficient to establish that relationship.

The correlated evidence demonstrates repeated failed authentication activity but does not demonstrate successful SMB access, successful lateral movement, or execution on the target system.

## Scope and Impact Analysis

After validating the correlated authentication alert, the investigation shifted from detection to determining whether the activity resulted in successful access or additional activity on `WIN11-LAB`.

### Successful Authentication Review

Windows Security Event ID `4624` was reviewed following the failed authentication sequence to identify potentially successful logons.

Two successful logon events were identified shortly after the failed attempts:

- `14:59:52` CDT

- `15:00:35` CDT

Both events involved the `SYSTEM` account, contained no remote source IP address, and used Logon Type `5`, which represents a service logon. These events did not match the characteristics of the SMB authentication attempts from `192.168.56.30`.

No successful remote authentication from the controlled Kali source was observed within the investigated time window.

### Process Execution Review

Sysmon Event ID `1` process-creation telemetry was reviewed following the authentication attempts to identify evidence of subsequent execution.

Processes observed during the reviewed period included normal Windows components and maintenance activity such as:

- `svchost.exe`

- `MoUsoCoreWorker.exe`

- `MpCmdRun.exe`

- `backgroundTaskHost.exe`

- `ProvTool.exe`

- `cmd.exe`

- `reg.exe`

- `conhost.exe`

The `cmd.exe` and `reg.exe` activity was examined more closely because command-shell and registry utilities can also appear during malicious activity. The observed command line executed `hpatchmonTask.cmd` and queried the Windows HotPatch registry configuration under:

`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\HotPatch`

The processes ran under the `SYSTEM` account in the context of surrounding Windows system and maintenance activity. No evidence was identified connecting these processes to the preceding SMB authentication attempts.

Review of the available process-creation telemetry therefore did not identify evidence that the failed authentication activity progressed to remote command execution.

### File Creation Review

Sysmon Event ID `11` file-creation telemetry was also reviewed for the period surrounding and following the authentication attempts.

The telemetry contained substantial normal background activity, including file creation associated with the Wazuh Agent, Windows Content Delivery Manager, Windows Update components, provisioning activity, and Microsoft Defender.

No file creation was identified that could be attributed to the SMB authentication attempts, and no evidence of payload deployment was identified within the investigated window.

### Scope Determination

Based on the available evidence, the observed activity was limited to repeated failed SMB/NTLM authentication attempts against the `labadmin` account on `WIN11-LAB`.

The investigation did not identify evidence of:

- Successful remote authentication from `192.168.56.30`

- Successful SMB access

- Remote command execution attributable to the authentication attempts

- Payload or malicious file deployment attributable to the authentication attempts

- Confirmed compromise of `WIN11-LAB`

These findings are limited to the telemetry sources and time window investigated. Absence of identified evidence is not treated as absolute proof that an event could not have occurred outside the available visibility or investigation scope.

## Incident Timeline

The following timeline reconstructs the primary correlated authentication sequence. Windows endpoint times are presented in Central Daylight Time (CDT).

| Time (CDT) | Data Source | Event | Analysis |

|---|---|---|---|

| 14:56:44 | Windows Security | Event ID `4625`, source `192.168.56.30:50220`, target `labadmin` | First failed authentication in the eight-event sequence |

| 14:56:46 | Sysmon | Event ID `3`, `192.168.56.30:50220` → `192.168.56.20:445` | Network connection correlated to first authentication failure |

| 14:56:50 | Windows Security | Event ID `4625`, source port `50228` | Second failed authentication |

| 14:56:52 | Sysmon | Event ID `3`, source port `50228` → TCP/445 | Matching SMB network connection |

| 14:56:56 | Windows Security | Event ID `4625`, source port `53304` | Third failed authentication |

| 14:56:58 | Sysmon | Event ID `3`, source port `53304` → TCP/445 | Matching SMB network connection |

| 14:57:03 | Windows Security | Event ID `4625`, source port `53308` | Fourth failed authentication |

| 14:57:05 | Sysmon | Event ID `3`, source port `53308` → TCP/445 | Matching SMB network connection |

| 14:57:09 | Windows Security | Event ID `4625`, source port `33548` | Fifth failed authentication |

| 14:57:10 | Sysmon | Event ID `3`, source port `33548` → TCP/445 | Matching SMB network connection |

| 14:57:14 | Windows Security | Event ID `4625`, source port `33550` | Sixth failed authentication |

| 14:57:17 | Sysmon | Event ID `3`, source port `33550` → TCP/445 | Matching SMB network connection |

| 14:57:20 | Windows Security | Event ID `4625`, source port `33554` | Seventh failed authentication |

| 14:57:22 | Sysmon | Event ID `3`, source port `33554` → TCP/445 | Matching SMB network connection |

| 14:57:25 | Windows Security | Event ID `4625`, Record ID `111962`, source port `33564` | Eighth failed authentication; correlation threshold satisfied |

| 14:57:27 | Sysmon | Event ID `3`, source port `33564` → TCP/445 | Network telemetry correlates with eighth failure |

| ~14:58:12 | Wazuh | Rule `60204`, Level 10 | Eight failed logons correlated as **Multiple Windows Logon Failures** |

| 14:59:52 | Windows Security | Event ID `4624`, `SYSTEM`, Logon Type `5` | Service logon; no remote source IP and not evidence of successful Kali authentication |

| 15:00:35 | Windows Security | Event ID `4624`, `SYSTEM`, Logon Type `5` | Additional service logon; not associated with the controlled source |

| Through 15:02 | Sysmon | Event IDs `1` and `11` reviewed | No subsequent execution or file creation identified as attributable to the authentication attempts |

The eight failed authentication events occurred within approximately 41 seconds. The source ports in the Windows Security events matched the source ports of the corresponding Sysmon TCP/445 connections, providing endpoint-level correlation between the network and authentication telemetry.

The correlated Wazuh alert occurred after the endpoint events were generated and processed. The timing difference was retained rather than modifying timestamps to make the data sources appear simultaneous.

## MITRE ATT&CK Analysis

Wazuh Rule `60204` mapped the correlated authentication activity to MITRE ATT&CK technique:

**T1110 - Brute Force**

This mapping is consistent with the repeated controlled password attempts performed during the simulation. However, MITRE ATT&CK mappings generated by security tools were treated as analytical context rather than proof of attacker intent or successful technique execution.

An earlier Wazuh Rule `60122` alert for an individual Windows Event ID `4625` was automatically mapped by Wazuh to `T1531 - Account Access Removal`. That mapping did not accurately describe the activity observed during this simulation because the event represented a failed authentication attempt rather than removal or denial of account access.

This difference demonstrated the importance of analyst validation of automatically assigned MITRE ATT&CK mappings.

The investigation therefore uses `T1110 - Brute Force` as the relevant technique for the correlated repeated-authentication simulation while documenting the inaccurate `T1531` mapping as a detection-analysis finding.

## Telemetry Improvements and Detection Validation

The investigation also identified opportunities to improve endpoint visibility.

### Sysmon Telemetry Tuning

The initial Sysmon configuration collected broad registry telemetry. During testing, this produced a large volume of registry events over a short period, including approximately:

- `65,436` Sysmon Event ID `12` events

- `15,677` Sysmon Event ID `13` events

The resulting event volume caused the Wazuh Agent event buffer to reach high utilization and eventually report that events could be lost.

Rather than disabling registry monitoring, the Sysmon configuration was tuned to focus registry telemetry on higher-value locations, including:

- `CurrentVersion\Run`

- `CurrentVersion\RunOnce`

- `HKLM\System\CurrentControlSet\Services\`

- `Microsoft\Windows NT\CurrentVersion\Winlogon`

- `Microsoft\Windows NT\CurrentVersion\Image File Execution Options\`

After applying the tuned configuration and observing the endpoint for approximately five minutes, registry telemetry decreased substantially to approximately 33 Event ID `12` events and 35 Event ID `13` events, with no recurrence of the previously observed buffer saturation warnings during the validation period.

This demonstrated the importance of balancing telemetry coverage with event volume and collection reliability.

### PowerShell Visibility Improvement

Review of the Wazuh Agent configuration showed that the Windows Application, Security, System, and Sysmon Operational logs were being collected, but the `Microsoft-Windows-PowerShell/Operational` channel was not configured for collection.

PowerShell Script Block Logging was also not enabled through the tested machine policy location.

The Wazuh Agent configuration was updated to collect:

`Microsoft-Windows-PowerShell/Operational`

PowerShell Script Block Logging was then enabled, allowing Windows Event ID `4104` events to be generated.

A harmless marker confirmed that Event ID `4104` was being created locally. Because the marker did not generate a visible Wazuh alert, the Wazuh ruleset was examined rather than assuming that collection had failed.

The investigation identified Wazuh's built-in PowerShell detection rules, including Rule `91808`, which detects PowerShell registry queries and maps them to MITRE ATT&CK technique `T1012 - Query Registry`.

A controlled read-only PowerShell registry query was then executed:

`Get-ItemProperty -Path HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion`

Windows generated Event ID `4104`, and Wazuh subsequently generated Rule `91808`, **Powershell script queried registry value**, at Level 3.

The test validated the complete telemetry pipeline:

`PowerShell execution → Script Block Logging → Windows Event 4104 → Wazuh Agent collection → Wazuh decoding → Rule 91808 → SIEM alert`

This PowerShell activity was performed as post-incident telemetry validation and was not part of the SMB authentication incident itself.

## Indicators and Observables

The following observables were used to correlate and investigate the simulated activity.

| Observable | Value | Context |

|---|---|---|

| Source host | `Kali-Lab` | Controlled attack simulation system |

| Source IP | `192.168.56.30` | Origin of the SMB authentication attempts |

| Destination host | `WIN11-LAB` | Monitored Windows endpoint |

| Destination IP | `192.168.56.20` | Target of the authentication attempts |

| Destination port | `445/TCP` | SMB service |

| Target account | `labadmin` | Account used during controlled authentication testing |

| Authentication | `NTLM` | Authentication package recorded in Event ID 4625 |

| Logon Type | `3` | Windows network logon |

| Status | `0xc000006d` | Logon failure |

| Substatus | `0xc000006a` | Incorrect password for an existing account |

| Windows Event ID | `4625` | Failed authentication |

| Sysmon Event ID | `3` | Network connection telemetry |

| Wazuh Rule | `60122` | Individual failed-logon detection |

| Wazuh Rule | `60204` | Correlated multiple-logon-failure detection |

| MITRE ATT&CK | `T1110 - Brute Force` | Mapping associated with the correlated authentication activity |

Because this activity occurred in a controlled lab, `192.168.56.30` and the other values above should be treated as investigation observables rather than real-world malicious indicators of compromise.

## Containment and Remediation

Because the activity was intentionally generated within an isolated lab and no successful remote authentication or subsequent compromise was identified, destructive containment actions were not required.

In a production environment, an alert with similar characteristics would require validation before containment decisions were made. Appropriate response actions could include reviewing the source system and targeted account, determining whether the source IP is expected, examining authentication history, identifying successful logons associated with the activity, and determining whether additional endpoints or accounts were targeted.

If the source were determined to be unauthorized, potential containment actions could include blocking the source at an appropriate network control, restricting affected account access, terminating confirmed unauthorized sessions, and isolating systems if evidence indicated that the activity had progressed beyond failed authentication.

Credential resets or account lockouts should be based on investigation findings and organizational response procedures rather than performed automatically solely because failed authentication events occurred.

For this lab, remediation focused primarily on monitoring quality. Sysmon registry telemetry was tuned to reduce excessive event volume, and PowerShell Operational logging and Script Block Logging were enabled to improve endpoint visibility.

## Detection Gaps and Limitations

Several monitoring limitations were identified during the project.

The initial Nmap SYN reconnaissance activity did not produce a corresponding Wazuh alert. The environment relied primarily on endpoint telemetry and did not include a dedicated network intrusion detection sensor. As a result, the absence of a Wazuh alert could not be interpreted as evidence that reconnaissance had not occurred.

Wazuh alert data was also not treated as a complete archive of every raw endpoint event. The manager configuration retained alerts but did not enable full event archival through `logall` or `logall_json`. Investigation therefore relied on both SIEM alerts and direct endpoint telemetry where appropriate.

The initial Wazuh Agent configuration did not collect the PowerShell Operational event channel, and Script Block Logging was not enabled through the tested machine policy location. This represented a telemetry visibility gap that was corrected and subsequently validated.

The project also demonstrated that automated severity and MITRE ATT&CK mappings require analyst review. A high alert level or technique mapping alone was not treated as proof that malicious activity or successful compromise had occurred.

## Key Findings and Lessons Learned

The investigation produced several technical and analytical findings.

1. **Multiple data sources provided stronger evidence than a single alert.** Windows Security Event ID `4625` established the authentication failures, while Sysmon Event ID `3` established the corresponding inbound TCP/445 network connections. Matching source ports allowed the two telemetry sources to be directly correlated.

2. **Detection thresholds must be understood before declaring a detection gap.** Three repeated failures did not trigger Wazuh Rule `60204`, but inspection of the built-in ruleset showed that the rule required eight matching authentication failures within 240 seconds from the same source IP.

3. **Existing detection logic should be understood before creating custom rules.** A custom correlation rule was unnecessary because Wazuh already contained appropriate built-in logic for the tested behavior.

4. **Alert severity does not establish compromise.** Wazuh generated a Level 10 correlated authentication alert, but subsequent investigation did not identify successful remote authentication, related remote execution, or attributable payload deployment.

5. **Automated MITRE ATT&CK mappings require validation.** Wazuh's `T1110 - Brute Force` mapping was consistent with the repeated password-attempt simulation, while the `T1531 - Account Access Removal` mapping associated with the individual failed-logon rule did not accurately characterize the observed activity.

6. **Excessive telemetry can reduce monitoring reliability.** Broad Sysmon registry collection generated enough events to saturate the Wazuh Agent buffer. Tuning the configuration significantly reduced event volume while retaining monitoring of higher-value registry locations.

7. **Collection and detection are separate concepts.** A PowerShell event can be collected without necessarily generating a visible alert. Examination of the Wazuh PowerShell ruleset helped distinguish telemetry collection from rule-triggering behavior.

8. **Scope analysis is necessary after detection.** Investigation continued beyond the initial alert by checking successful logons, process creation, file creation, and network telemetry to determine whether the activity progressed beyond failed authentication.

## Evidence and Preserved Artifacts

Screenshots and raw alert data were preserved within the Project 008 repository to support the investigation findings.

### Key Authentication Evidence

- `09_Wazuh-Phase2-SMB-Failed-Logon-Details-1.png`

- `10_Wazuh-Phase2-SMB-Failed-Logon-Rule-Details.png`

- `11_Wazuh-Phase2-Repeated-SMB-Failed-Logons.png`

- `12_Wazuh-Phase2-SMB-Event-Correlation.png`

- `13_Wazuh-Repeated-Authentication-Correlation-60204.png`

- `14_Wazuh-60204-Correlated-Alert-Details.png`

- `16_Sysmon-SMB-Network-Correlation.png`

- `17_Windows-4625-SMB-Failed-Logon-Correlation.png`

### Telemetry Engineering Evidence

- `05_Sysmon-Event-Volume-Before-Tuning.png`

- `06_Sysmon-Event-Volume-After-Tuning.png`

- `07_Wazuh-Agent-Healthy-After-Sysmon-Tuning.png`

- `08_Wazuh-Sysmon-PowerShell-Correlation.png`

- `15_Wazuh-PowerShell-4104-Detection-91808.png`

### Raw Wazuh Alert Data

- `2026-09-22_SMB-Failed-Logon-4625.json`

- `2026-09-22_Multiple-Windows-Logon-Failures-60204.json`

- `2026-09-23_PowerShell-Registry-Query-91808.json`

The evidence files preserve both the detection results and the analyst investigation used to validate those results.

## Conclusion

This simulated SOC investigation demonstrated an end-to-end workflow from security-event generation through detection, triage, correlation, scope analysis, and documentation.

Repeated failed SMB/NTLM authentication attempts from `Kali-Lab` against the `labadmin` account on `WIN11-LAB` generated Windows Security Event ID `4625` telemetry and individual Wazuh alerts. After the built-in correlation threshold was satisfied, Wazuh Rule `60204` generated a Level 10 **Multiple Windows Logon Failures** alert mapped to MITRE ATT&CK `T1110 - Brute Force`.

The alert was not treated as proof of compromise. Windows authentication telemetry was correlated with Sysmon network connections, and subsequent successful-logon, process-creation, and file-creation telemetry was reviewed to determine the scope of the activity. No successful remote authentication from the controlled source or evidence linking the authentication attempts to subsequent execution or payload deployment was identified within the investigated window.

The project also identified and addressed monitoring-quality issues. Excessive Sysmon registry telemetry was reduced through targeted configuration tuning, and a PowerShell visibility gap was corrected by collecting the PowerShell Operational channel and enabling Script Block Logging. The improved PowerShell telemetry pipeline was then validated end-to-end using Windows Event ID `4104` and Wazuh Rule `91808`.

Overall, the project demonstrates practical SOC skills including SIEM alert triage, Windows event-log analysis, Sysmon analysis, cross-source event correlation, detection-rule investigation, MITRE ATT&CK validation, telemetry tuning, scope determination, and evidence-based incident reporting.
