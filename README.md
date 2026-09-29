# Route 53 backup

This CloudFormation template creates a scheduled backup solution for all Route
53 hosted zones in the AWS account where the stack is deployed.

For every hosted zone, the Lambda function:

1. Lists all resource record sets, including paginated results.
2. Converts the records to readable BIND-style zone text.
3. Stores the result in an existing S3 bucket.

Backup objects use the following key format:

```text
YYYY-MM-DDTHH:MMZ/zone-name.zone
```

The timestamp is UTC. A single Lambda invocation processes all hosted zones in
parallel, with a maximum of eight worker threads.

## Resources created

The stack creates:

- An AWS Lambda function running Python 3.13.
- An IAM execution role for the Lambda function.
- An EventBridge scheduled rule that invokes the Lambda.
- Permission for EventBridge to invoke the Lambda.
- A CloudWatch log group with configurable retention.
- A CloudWatch Logs metric filter named `FailedBackups`.
- A CloudWatch alarm that publishes when one or more zones fail.
- An SNS topic and email subscription for alarm notifications.

The Lambda has permissions to:

- List Route 53 hosted zones and resource record sets.
- Write backup objects to the configured S3 bucket.
- Publish test notifications to the stack-created SNS topic.
- Write logs to its own CloudWatch log group.

## Parameters

| Parameter | Required | Default | Description |
| --- | --- | --- | --- |
| `S3BucketName` | Yes | None | Existing S3 bucket where backup objects are stored. The name must follow the S3 bucket naming rules. |
| `ScheduleExpression` | No | `cron(0 2 * * ? *)` | EventBridge schedule expression. The default runs daily at 02:00 UTC. |
| `ExpireLog` | No | `14` | CloudWatch Logs retention in days. Supported values are `1`, `3`, `5`, `7`, `14`, `30`, `60`, `90`, `120`, `150`, `180`, `365`, `400`, `545`, `731`, `1827`, and `3653`. |
| `TestNotification` | No | Empty | Any non-empty value causes a test email to be sent on every Lambda invocation. Leave empty during normal operation. |
| `NotificationEmail` | No | `notifications@example.com` | Email address subscribed to the failure SNS topic. |

Example parameter values:

```text
S3BucketName       = route53-backups-example
ScheduleExpression = cron(15 3 * * ? *)
ExpireLog          = 30
TestNotification   =
NotificationEmail  = dns-alerts@example.com
```

## S3 bucket and encryption

The S3 bucket is **not** created by this stack. Create it before deploying the
stack and configure its default encryption as:

```text
Encryption type: SSE-KMS
Key: AWS managed key aws/s3
```

The Lambda does not send a KMS key ID or encryption header in `PutObject`.
S3 therefore applies the bucket's default encryption automatically. No KMS
parameter or KMS permission is required in the Lambda execution role.

Because the bucket can be in another account, add the required cross-account
`s3:PutObject` permission manually to the bucket policy. The principal is the
Lambda execution role created by the stack. Its ARN is available from the IAM
role in the stack resources; the stack does not expose that role as an output.

The bucket policy should restrict uploads to the required source account or
Lambda role and should normally deny insecure transport. Do not require the
`s3:x-amz-server-side-encryption` request header in that policy: this Lambda
relies on the bucket's default encryption rather than sending that header.

Configure an S3 lifecycle rule separately to expire old backups, for example
after 30 or 90 days. The lifecycle policy is intentionally outside this stack
because the bucket is externally managed.

## Backup format and limitations

Normal records with `ResourceRecords` are written as BIND-style lines. Route 53
alias records do not have a direct BIND equivalent; they are written as a
comment and a readable pseudo-record with a default TTL of 60 seconds when no
TTL is present.

If formatting an individual record fails, that record is logged and skipped.
If reading or writing an entire hosted zone fails, the zone is counted as a
failure. The Lambda logs a warning and returns a partial-success response when
one or more zones fail. The CloudWatch metric filter detects the failure log
entries and drives the alarm.

## Notifications and monitoring

The stack creates an SNS email subscription. The recipient must confirm the
subscription before SNS can deliver alarm notifications.

The `FailedBackups` custom metric is published in the `Route53Backup`
namespace. The alarm evaluates the sum over one five-minute period and alarms
when the value is at least `1`. Missing data is treated as not breaching.

EventBridge retries failed Lambda invocations up to two times, with a maximum
event age of one hour.

If `TestNotification` is non-empty, the Lambda publishes a test message on
every invocation, including scheduled invocations. Set it to an empty value
after testing.

## Deployment

1. Create the destination S3 bucket.
2. Configure S3 default encryption with the AWS-managed `aws/s3` KMS key.
3. Add the cross-account bucket policy allowing the Lambda execution role to
   upload objects.
4. Deploy `backupdns.yml` with the required parameters.
5. Confirm the SNS email subscription.
6. Optionally invoke the Lambda manually or set `TestNotification` temporarily
   to verify the notification path.

The Lambda function name is generated as:

```text
<stack-name>-backupdns
```

## Outputs

| Output | Description |
| --- | --- |
| `BackupFunctionArn` | ARN of the backup Lambda function. |
| `FailedBackupAlarmTopicArn` | ARN of the SNS topic used by the failure alarm. |
