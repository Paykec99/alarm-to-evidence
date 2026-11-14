# Investigation Summary

## Symptoms

### devops-agent-demo-errors went to ALARM twice
**Description:** The alarm `devops-agent-demo-errors` (DevOpsAgentDemo/TestErrors, Sum >= 1 per 60s, eu-central-1, account [redacted account ID]) went to ALARM at 01:26:56 and again at 01:34:53 UTC on 2026-11-14. Each time it returned to OK at the next check (01:27:48 and 01:36:11) and has stayed OK since. The reported start time of 01:44:41 does not match any state change.
**Time:** 2026-11-14T01:26:56Z - 2026-11-14T01:36:11Z

## Findings

### Root Cause: Root user set the ALARM state by hand to test emails
**Description:** The root user of account [redacted account ID] called SetAlarmState from the AWS CLI on Windows (source IP [redacted IP]) at 01:26:56 ('Synthetic test of the alarm state transition') and at 01:34:53 ('Confirmed subscription: end-to-end test email'). The metric DevOpsAgentDemo/TestErrors has never received data, so no threshold was ever crossed. This was not a real incident and nothing needs fixing.
**Cascades to:** symptom-alarm-state-alarm

#### Supporting Observations
- **Alarm transitions were manual/synthetic, not metric-driven:** Alarm created 01:26:53 UTC (actions disabled). Both OK→ALARM transitions carry manual state reasons: 01:26:56 'Synthetic test of the alarm state transition' and 01:34:53 'Confirmed subscription: end-to-end test email'. Each reverted to OK on next evaluation (01:27:48, 01:36:11) because no TestErrors datapoints existed (missing data treated as notBreaching). ARN: arn:aws:cloudwatch:eu-central-1:[redacted account ID]:alarm:devops-agent-demo-errors
- **TestErrors metric has never been published:** get_metric_data for DevOpsAgentDemo/TestErrors (Sum and SampleCount, 60s, 00:00–02:30 UTC) returned zero datapoints, and list_metrics finds no metrics in the DevOpsAgentDemo namespace. That means no real error data drove either ALARM transition. Both were manual set-alarm-state test transitions.
- **CloudTrail: the account root user managed the alarm from the console:** CloudWatch management events between 01:00 and 01:46 UTC show the account root user (arn:aws:iam::[redacted account ID]:root, console session with MFA) working on `devops-agent-demo-errors`. One example is ListAlarmMuteRules at 01:27:30. The DevOps Agent role (incident-email-AgentRole-J9wxSUwuMSUh) ran ListTagsForResource at 01:39:43. Alarm history shows no further state changes after the 01:36:11 return to OK.
- **Alarm created by root user via AWS CLI:** CloudTrail shows the root user (arn:aws:iam::[redacted account ID]:root, source IP [redacted IP]) calling PutMetricAlarm through aws-cli/2.37.12 to create devops-agent-demo-errors with actions disabled. It also shows console activity (GetOTelEnrichment, ListAlarmMuteRules) around 01:27.

### Hypothesis: First test sent no email because the email rule wasn't wired up yet
**Description:** The incident-email-notify rule matched nothing at 01:26:56. Its targets were only added at 01:31:44-45 (PutTargets by root), so the rule probably did not exist yet. This is inferred from timing only.
**Cascades to:** symptom-alarm-state-alarm

#### Supporting Observations
- **EventBridge rule targets were set up after the first ALARM:** CloudTrail shows the root user calling PutTargets (01:31:44, 01:31:45) and TagResource (01:31:46–01:32:15) on EventBridge. That was after the first ALARM transition at 01:26:56, which explains why only the 01:34:53 transition matched incident-email-notify. The 01:30 SNS delivery was probably the subscription-confirmation flow.
- **Notification pipeline delivered test emails successfully:** EventBridge rule incident-email-notify matched and invoked 1 event at 01:34 with 0 failed invocations. SNS topic incident-email-Topic-YPPTjgOSuFTt published 1 message at 01:34 and delivered notifications at 01:30 and 01:34, with 0 failures. The end-to-end email workflow worked as intended.

#### Gaps
- **Email rule creation time not checked:** The investigation did not look up when the EventBridge rule incident-email-notify was created. The explanation for the missing email at 01:26:56 is therefore inferred from the PutTargets timing, not confirmed.


