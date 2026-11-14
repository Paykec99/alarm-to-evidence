# CloudWatch → AWS DevOps Agent → email

Deploy in Frankfurt (eu-central-1), in the same AWS project and Region as the existing alarm. The stack creates an Agent Space, its monitoring association, an SNS email subscription, and EventBridge rules. The monitoring role uses AWS's standard DevOps Agent access policy. Use a test AWS project if production telemetry should remain outside this scope.

First deploy with a valid non-root CLI profile:

```powershell
aws cloudformation deploy --template-file devops-agent/stack.json --stack-name incident-email --capabilities CAPABILITY_IAM --region eu-central-1 --profile <profile> --parameter-overrides AlarmName=<existing-alarm-name> EmailAddress=<recipient>
```

Confirm the SNS subscription from the recipient's inbox. An actual transition to ALARM then sends the first email. This works while automatic investigations are disabled.

To connect investigations, create an API-key generic webhook in the new Agent Space under Capabilities → Webhook. Store its URL and token in a Secrets Manager secret as JSON with keys "url" and "token". Keep the token out of source and chat. Redeploy the same stack with the same AlarmName and EmailAddress plus EnableInvestigations=true and WebhookSecretArn=<secret-arn>. Use the standard Secrets Manager encryption key, or grant the Lambda role decrypt permission scoped to your custom key.

For an end-to-end test, use one isolated alarm with a known test error in its CloudWatch logs. Observe the initial email, then check for an investigation in DevOps Agent and a second email with its summary or failure status. The alarm evaluation and investigation duration affect when each message arrives. The stack forwards the investigation summary to the confirmed recipient, so use an inbox authorized to receive those findings.

The DevOps Agent charges per active agent-second, plus separate charges for connected services. At USD 0.0083 per agent-second, five minutes of active work costs about USD 2.49 before credits and other charges. Keep EnableInvestigations=false until the budget is confirmed. Disable the rule again after the test if the alarm should no longer start investigations. Disabling it does not cancel work that is already running.

When provisioning via the eventChannel API, store the returned URL and signing secret under "url" and "hmac_secret". The bridge signs the outgoing request using AWS's timestamp and body signing format. A stack update with WebhookSecretArn set and EnableInvestigations=false configures secret access while leaving the alarm trigger disabled.
