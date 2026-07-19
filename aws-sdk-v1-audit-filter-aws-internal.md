# AWS SDK v1 "usage" is mostly AWS's own managed services

CloudTrail calls with an `aws-internal/` userAgent prefix come from AWS-managed services, not your code, so exclude them before concluding an SDK-v1 migration is needed.

## Why it matters

A "you still use AWS SDK for Java 1.x, migrate now" alert can be entirely AWS-managed calls, costing a migration sprint on code you do not own. Filtering the noise first turns an 89-hit alarm into zero customer-owned calls.

## How it works

Several tells mark a CloudTrail record as AWS-origin, none of which customer code can produce:

- `userAgent` starts with `aws-internal/3` — a token only AWS managed services attach.
- Source IPs are all in AWS-owned ranges, never your VPC or office.
- Lambda VPC networking shows as EC2 `CreateNetworkInterface`/`DeleteNetworkInterface`, often with `Client.DryRunOperation` errors and a fixed `AWS Lambda VPC ENI-<fn>` description.
- Session names follow `awslambda_<n>_<timestamp>`, and one numeric session name can appear under two function roles at once.

## Example

Keep only customer-owned SDK-v1 calls by excluding the internal prefix.

```sql
SELECT eventTime, eventName, userIdentity.arn, useragent, sourceIPAddress
FROM cloudtrail_logs
WHERE useragent LIKE '%aws-sdk-java/1.%'
  AND useragent NOT LIKE 'aws-internal/%';
```

## Gotchas

- AWS SSO console sign-in emits `AssumeRoleWithSAML` on SDK v1 from AWS's side; it is not your app.
- The `aws-internal/` prefix cannot be forged by customer code, so it is a safe exclusion filter.
