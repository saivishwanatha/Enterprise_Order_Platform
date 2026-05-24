# AWS Standards

## Services Used

| AWS Service | Purpose |
|---|---|
| **EC2** | Compute for service containers (ECS on EC2 or direct EC2) |
| **RDS (PostgreSQL)** | Managed database per service |
| **SNS** | Event fan-out (one topic per aggregate) |
| **SQS** | Event consumption queues (one queue per consumer per aggregate) |
| **S3** | Invoice PDF storage; application artifact storage |
| **CloudWatch** | Logs, metrics, alarms, dashboards |
| **Secrets Manager** | Database credentials, JWT signing keys, third-party API keys |
| **IAM** | Service roles and policies (least-privilege) |

## SDK Configuration

- Use **AWS SDK for Java v2** (`software.amazon.awssdk`). Never use v1.
- Configure all SDK clients as Spring beans in a `@Configuration` class.
- In production, use the default credential provider chain (IAM role attached to EC2/ECS task).
- In local/test, override the endpoint URL to `http://localhost:4566` (LocalStack) via `application-local.yml`.
- Always set an explicit AWS region. Read it from `${aws.region}` environment variable.
- Set SDK retry policy: 3 retries with exponential backoff. Do not use the default unlimited retry.

```yaml
# application-local.yml
aws:
  region: us-east-1
  endpoint-override: http://localhost:4566
```

## SNS Standards

- One SNS topic per aggregate domain (e.g., `prod-order-events`, `prod-payment-events`).
- Topic naming: `{env}-{aggregate}-events`.
- All topics use **Standard** type (not FIFO) unless strict ordering is required and documented.
- Enable **server-side encryption** (SSE) on all topics using AWS-managed keys.
- Message size limit: 256 KB. For larger payloads, store the payload in S3 and include the S3 key in the event.
- Use SNS **message attributes** for `eventType` to enable SQS subscription filter policies.
- All topics are provisioned via Terraform. No manual creation.

## SQS Standards

- One SQS queue per consumer service per aggregate (e.g., `prod-inventory-order-queue`).
- Every queue must have a paired DLQ: `{queue-name}-dlq`.
- Queue configuration:
  - `VisibilityTimeout`: 30 s
  - `MessageRetentionPeriod`: 345600 s (4 days)
  - `ReceiveMessageWaitTimeSeconds`: 20 (long polling)
  - DLQ `maxReceiveCount`: 5
- Enable **server-side encryption** on all queues.
- Subscribe queues to SNS topics with a filter policy on `eventType` attribute.
- CloudWatch alarm on DLQ `ApproximateNumberOfMessagesVisible > 0`.
- All queues are provisioned via Terraform.

## S3 Standards

- Bucket naming: `{env}-enterprise-order-{purpose}` (e.g., `prod-enterprise-order-invoices`).
- Enable **versioning** on all buckets.
- Enable **server-side encryption** (SSE-S3 minimum; SSE-KMS for sensitive data like invoices).
- Block all public access on all buckets. No public bucket policies.
- Invoice PDFs: stored at `invoices/{year}/{month}/{invoiceId}.pdf`.
- Access via **pre-signed URLs** (15-minute expiry) — never expose bucket URLs directly.
- Enable S3 access logging to a dedicated `{env}-enterprise-order-access-logs` bucket.
- Lifecycle policy: transition to S3 Glacier after 90 days for invoice archives.

## RDS Standards

- One RDS instance per service (or one instance with separate schemas if cost-constrained, but separate users per schema).
- Engine: **PostgreSQL 15+**.
- Instance class: `db.t3.medium` minimum for production.
- Enable **Multi-AZ** in production.
- Enable **automated backups** with 7-day retention.
- Enable **encryption at rest** using AWS-managed keys.
- Database credentials stored in **Secrets Manager**, not in environment variables or config files.
- Enable **Performance Insights** in production.
- VPC: RDS instances are in private subnets. No public accessibility.

## CloudWatch Standards

- **Log Groups**: one per service, named `/enterprise-order/{env}/{service-name}`.
- Log retention: 30 days in production, 7 days in non-production.
- All application logs are structured JSON (see `tech.md` observability section).
- **Metric Filters**: create metric filters for `ERROR` log level and publish to a custom namespace `EnterpriseOrder/{ServiceName}`.
- **Alarms** (mandatory for every service):
  - `5xxErrorRate > 1%` over 5 minutes → SNS alert
  - `P99Latency > 2000ms` over 5 minutes → SNS alert
  - `DLQMessageCount > 0` → SNS alert
  - `CPUUtilization > 80%` over 10 minutes → SNS alert
- **Dashboards**: one dashboard per service showing error rate, latency, throughput, and DLQ depth.

## IAM Standards

- Every service runs under a dedicated IAM role. No shared roles between services.
- Roles follow least-privilege: grant only the specific SNS/SQS/S3/Secrets Manager actions the service needs.
- No `*` actions or `*` resources in any policy.
- Use IAM **resource-based policies** for S3 bucket access.
- Rotate all IAM access keys (if used) every 90 days. Prefer IAM roles over access keys.
- All IAM roles and policies are defined in Terraform.

## Secrets Manager Standards

- Secret naming: `{env}/{service-name}/{secret-name}` (e.g., `prod/order-service/db-credentials`).
- Rotate database credentials automatically every 30 days using the RDS rotation Lambda.
- Application reads secrets at startup via Spring Cloud AWS or the AWS SDK. Never cache secrets beyond the application lifecycle.
- Never log secret values. Reference secrets by their Secrets Manager ARN in logs.

## Terraform Standards

- All AWS resources are provisioned via Terraform in `infra/terraform/`.
- State is stored in an S3 backend with DynamoDB state locking.
- Modules: one module per service's infrastructure (RDS, SQS queues, IAM role).
- Shared resources (SNS topics, VPC, S3 buckets) are in a `shared` module.
- Tag all resources with: `Environment`, `Service`, `Project = enterprise-order-platform`, `ManagedBy = terraform`.
- Run `terraform plan` in CI on every PR that touches `infra/terraform/`. Apply only from `main` branch.

## LocalStack (Local Development)

- LocalStack emulates SNS, SQS, S3, and Secrets Manager locally.
- Init scripts in `infra/localstack/` run on container startup to create all required resources.
- Use the `local` Spring profile to point all AWS SDK clients to `http://localhost:4566`.
- LocalStack version must be pinned in `docker-compose.yml`. Do not use `latest`.
- Testcontainers uses the `localstack/localstack` image for integration tests.
