
# Investigating a Suspicious Azure Sign-In and Monitoring Gaps

## Scenario

A security operations investigation was initiated under ticket SOC-1058 to identify anomalous activity within an Azure training environment. The objective was to investigate sign-in telemetry, identify suspicious authentication activity, and assess whether the available monitoring data provided sufficient visibility into the activity.

## Environment

- **Platform:** Microsoft Azure
- **Environment:** Live multi-user Azure training tenant
- **Logging:** Azure Monitor and Log Analytics
- **Query language:** Kusto Query Language (KQL)
- **Data sources:** Custom sign-in logs (`LabSigninLogs_CL`) and Azure Activity Log
- **Services examined:** Azure Portal, Log Analytics workspace
- **Access level:** Reader access to the resource group and Log Analytics Reader access to the workspace
- **Investigation reference:** SOC-1058

## Investigation

### 1. Orienting the investigation

I started by searching the Log Analytics workspace for the investigation ticket identifier. Because I was working in an unfamiliar workspace, I used KQL's global `search` operator to locate records associated with the case.

```kql
search "SOC-1058"
| where TimeGenerated > ago(730d)
```

The search returned records from Azure Activity and a custom handoff table. The handoff record provided context for the investigation and confirmed that the ticket was associated with a previous analyst handoff.

This established the investigation's starting point and helped identify relevant telemetry before narrowing the search.

**Evidence — Initial ticket search**

<img width="626" height="356" alt="image" src="https://github.com/user-attachments/assets/08938b4c-f50c-4263-9ae9-32ff3452454b" />

<!-- Suggested filename: 01-investigation-ticket-search.png -->

### 2. Investigating suspicious authentication activity

I examined the custom sign-in telemetry in `LabSigninLogs_CL` to identify abnormal authentication patterns.

The investigation focused on failed sign-in activity followed by a successful authentication. I used the account identified during the initial hunt to isolate its successful sign-in record.

```kql
LabSigninLogs_CL
| where TimeGenerated > ago(730d)
| where ResultType_d == 0
| where UserPrincipalName_s == "<REDACTED_ACCOUNT>"
| project TimeGenerated, ResultType_d, IPAddress_s, AppDisplayName_s
| sort by TimeGenerated asc
```

The results identified a successful authentication to the Azure Portal. The record included a source IP address and a location outside the expected corporate environment.

The successful sign-in was important because it showed that the investigation was not limited to failed authentication attempts. A successful authentication following suspicious activity warranted further investigation.

The record also contained a `UserAgent_s` field, which can help an analyst understand the browser or client associated with an authentication event. I did not retain the actual value in this public write-up.

**Evidence — Successful Azure Portal sign-in**

<img width="630" height="486" alt="image" src="https://github.com/user-attachments/assets/33f1aa1d-4f3e-44e3-a836-46057a408337" />

<!-- Suggested filename: 02-suspicious-signin-result.png -->

### 3. Monitoring visibility and detection considerations

The broader lab scenario also involved evaluating whether Azure resource activity was being logged and whether an automated alert could identify the suspicious authentication pattern.

Two separate monitoring questions are important:

- **Logging coverage:** Was the relevant resource configured to send the necessary diagnostic data to Log Analytics?
- **Detection coverage:** Did the alert rule's query and threshold identify the suspicious authentication pattern?

These are separate controls. A resource can generate activity that is not being collected, and collected events can still go undetected if an alert rule is not configured appropriately.

I did not capture screenshots of the diagnostic settings, alert configuration, or Workbook results during this session, so I am not presenting specific configuration values or claiming independent verification of those findings.

## What broke / what surprised me

- The initial investigation required searching across workspace tables rather than assuming the relevant data source was already known.
- A successful authentication record provided more useful context than looking at failed attempts alone.
- The investigation highlighted the difference between collecting security telemetry and actually detecting suspicious behavior.
- I would capture the diagnostic settings, Azure Activity Log change records, and alert configuration in a future investigation to make the monitoring-gap conclusions independently verifiable.

## Findings and recommendations

### Findings

1. The investigation ticket was identifiable through a workspace-wide KQL search.
2. The custom sign-in table contained a successful Azure Portal authentication record for the account under investigation.
3. The authentication record included source IP, timestamp, application, and location fields that could support further investigation.
4. The available screenshots do not independently establish the diagnostic-setting configuration or alert-rule threshold.

### Recommendations

1. **Review authentication activity:** Correlate failed and successful sign-ins by account, source IP, timestamp, application, and user-agent information. Investigate unusual geographic or network patterns.
2. **Validate logging coverage:** Review diagnostic settings on critical Azure resources and confirm that required logs are being sent to the designated Log Analytics workspace.
3. **Validate alert effectiveness:** Compare actual authentication activity against alert-rule conditions and thresholds. Test detection logic against known suspicious patterns to identify missed detections without creating excessive false positives.

## What I learned

- **Technical:** KQL's `search` operator is useful for orienting yourself in an unfamiliar Log Analytics workspace before narrowing the investigation to a specific table.
- **Authentication investigation:** A successful sign-in following suspicious failed attempts deserves investigation, particularly when the source location or network is unexpected.
- **Monitoring architecture:** Logging coverage and alert coverage are different. Collecting an event does not guarantee that an analyst or automated detection rule will identify it.
- **Investigation discipline:** Findings should be tied to actual telemetry. If a configuration or event was not examined, it should not be presented as independently verified.
- **What I would do differently:** Capture evidence at each stage of the investigation, especially diagnostic-setting changes, alert configurations, and the final monitoring assessment.
