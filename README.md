# CAT — AWS Enterprise Cloud Migration

Cloud Solutions Consulting Pte Ltd was engaged by Real World Pte Ltd to migrate an on-premises .NET Core 6.0 web application to AWS. The build covers a multi-AZ VPC with an internet-facing Application Load Balancer, an Auto Scaling group of Windows/IIS web servers and a private Amazon RDS SQL Server database, plus an event-driven serverless tier (API Gateway, Lambda, DynamoDB, S3, SNS) for a product catalogue, upload notifications and automatic image watermarking. Every component was deployed in `us-east-1` and verified through load, scaling, self-healing and alarm tests.

## Contents

| File | Description |
| --- | --- |
| `CAT-Assignment-2-Report.docx` | Full assignment report, including the Lambda source in section 4 |
| `index.html` | Front-end for the serverless product application |

## Architecture

A single VPC (`10.1.0.0/16`) spans two Availability Zones, with one public and one private subnet in each. The public route table sends `0.0.0.0/0` to the internet gateway; the private route table carries only the local route, so the database has no internet path.

- **Ingress:** an internet-facing Application Load Balancer in both public subnets listens on HTTP/80 and forwards to a target group that health-checks `/` on port 80.
- **Compute:** the Auto Scaling group runs a custom Windows Server AMI pre-loaded with IIS, .NET 6 and the application, so scale-out needs no post-boot installation. Desired and minimum capacity are 2, maximum 4, with balanced best-effort placement across the two AZs. The launch template places instances in the public subnets, allowing the ALB to reach them directly.
- **Scaling policy:** target tracking on average EC2 CPU utilisation at 70%.
- **Database:** Amazon RDS for SQL Server Express (`db.t3.small`, 20 GiB gp2, port 1433) in the private subnets, with public access disabled. The application connects through a connection string in `appsettings.json`.
- **Security groups:** ALB accepts HTTP from anywhere; EC2 accepts HTTP only from the ALB security group and RDP only from the administrator's IP; RDS accepts 1433 only from the EC2 security group.
- **Serverless tier:** a static page on S3 calls an API Gateway HTTP API, which proxies to a Lambda function reading and writing a DynamoDB table. S3 event notifications and DynamoDB Streams drive separate SNS topics, keeping application alerts independent of infrastructure alarms.

## AWS services used

| Service | Role in the solution |
| --- | --- |
| Amazon VPC | Network isolation, four subnets across two AZs, route tables, internet gateway |
| Amazon EC2 | Windows Server web tier running IIS and the .NET Core application |
| Elastic Load Balancing | Internet-facing Application Load Balancer and target group |
| EC2 Auto Scaling | Launch template, 2-4 capacity, CPU target tracking, self-healing |
| Amazon RDS | SQL Server Express database in private subnets |
| Amazon S3 | Static site hosting, image input/output buckets, lifecycle archival |
| AWS Lambda | Product CRUD API, stream notifications, image watermarking |
| Amazon API Gateway | HTTP API exposing `/products` with Lambda proxy integration |
| Amazon DynamoDB | Schemaless `Products` table with streams enabled |
| Amazon SNS | Infrastructure alarm topic and two application event topics |
| Amazon CloudWatch | Metrics, six alarms, dashboard and Lambda logs |
| AWS IAM | Lab execution role shared by the Lambda functions |

## Serverless workflows

The three Lambda functions (`ProductsCRUD`, `NotifyOnProduct` and `ImageWatermarkFunction`) are reproduced in full in the report, in section 4 "Lambda Function Code". The workflows they implement are:

- **Image watermarking** — an S3 upload event invokes `ImageWatermarkFunction`, which applies EXIF orientation, draws a semi-transparent watermark with Pillow (supplied through a custom layer), writes the result to a separate output bucket under `watermarked/` with SSE-S3 encryption, and publishes a notification to SNS. Input and output buckets are kept separate so the write cannot re-trigger the function.
- **Upload notifications** — an S3 object-created event produces a formatted email containing the object metadata and a presigned download link.
- **Product notifications** — DynamoDB Streams on the `Products` table filter for `INSERT` events and publish to a product topic, so API writes never wait on SNS and updates or deletes raise no false alerts.
- **Lifecycle** — output objects move to Glacier Deep Archive after one day, incomplete multipart uploads abort after seven days, and originals expire from the input bucket after seven days.

## Monitoring and alarms

- `EC2-HighCPU-Alarm` — average CPU above 70% for 2 of 3 datapoints, so a single instance spike does not fire.
- `EC2-StatusCheckFailed-Alarm` — maximum of 1 or more on a single datapoint; averaging would hide one failed instance in a fleet of two.
- `ALB-HighTraffic-Alarm` — sum of `RequestCount` above 20 over 5 minutes, 2 of 3 datapoints.
- `ALB-HighProcessedBytes-Alarm` — sum of `ProcessedBytes` above 45,000 bytes, 2 of 3 datapoints.
- `ALB-UnhealthyHost-Alarm` — fires when a registered target stops passing its health check.
- `RDS-LowFreeStorage-Alarm` — free storage below 4 GB on a single datapoint, since storage drains gradually and early warning is the point.
- Dashboard `CAT_Assignment2` — eight widgets grouped by compute health, database and network/traffic rather than creation order.
- All infrastructure alarms publish to one SNS topic with a confirmed email subscription; application events use their own topics so the streams can be filtered or muted independently.

## Testing and results

- CRUD operations were validated directly against the base instance's public IP, then repeated through the ALB DNS name to prove both scaled instances reach the shared database.
- The target group reported 2 of 2 healthy targets once Windows and IIS finished starting.
- Scale-out: with the CPU target temporarily lowered to 20%, a PowerShell workload drove utilisation up and the group grew from 2 to 4 instances.
- Scale-in: after the jobs were stopped, CloudWatch and the stabilisation period delayed termination, then the group returned to its minimum of 2.
- Self-healing: a managed instance was terminated manually; the ALB kept serving from the survivor and the group replaced the instance through the launch template back to 2 of 2 healthy.
- All six alarms were driven into `ALARM` and confirmed by email, with thresholds restored afterwards.
- Serverless testing ran inside out: Lambda test event, then the API endpoint, then the hosted page, with persistence checked in the DynamoDB console.

## Key takeaways

- Separating performance alarms from failure alarms matters: a crashed instance reports near-zero CPU, so CPU alarms alone would miss a total outage.
- Splitting input and output S3 buckets is what prevents a watermarking loop, since writing back to the trigger bucket would re-invoke the function indefinitely.
- Notification logic belongs off the write path — DynamoDB Streams keep API latency low and stop alerting failures from failing product writes.
- Basing the launch template on a pre-configured AMI turns scale-out into a fast, repeatable operation instead of a scripted install.
- Matching API Gateway routes and CORS exactly (`/products` with no trailing slash, explicit allowed methods and headers) removed the 404, 403 and preflight failures seen during integration.

## Notes

- Account IDs, ARNs, bucket names, API endpoints and credentials are omitted from this repository.
- The report cover page reads "April 2025 Semester"; the assignment brief and submission dates place this work in the April 2026 semester.
- The Lambda source code is included inside the report rather than as separate files, so it is not duplicated here.
