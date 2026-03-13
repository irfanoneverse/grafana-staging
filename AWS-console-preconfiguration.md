### 1: Create S3 Bucket

1. Click the **Create bucket** button.
2. Fill in the following:

| Setting         | Value                                                                |
| --------------- | -------------------------------------------------------------------- |
| **Bucket type** | General purpose                                                      |
| **Bucket name** | `oneone-grafana-prod-bucket`                                         |
| **AWS Region**  | `Asia Pacific (Singapore) ap-southeast-1` (or your preferred region) |

3. Block Public Access
4. Enable Bucket Versioning
5. Default Encryption

Under **Default encryption**:

| Setting             | Value                                                       |
| ------------------- | ----------------------------------------------------------- |
| **Encryption type** | Server-side encryption with Amazon S3 managed keys (SSE-S3) |
| **Bucket Key**      | Enable                                                      |

6. Create Bucket

### 2: Create IAM Policy

1. Navigate to **IAM** and click **Policies** in the left sidebar.
2. Click **Create policy**. Select the **JSON** tab. Paste the following JSON:

{
"Version": "2012-10-17",
"Statement": [
{
"Sid": "LGTMStackS3Access",
"Effect": "Allow",
"Action": [
"s3:PutObject",
"s3:GetObject",
"s3:DeleteObject",
"s3:ListBucket",
"s3:GetBucketLocation"
],
"Resource": [
"arn:aws:s3:::oneone-grafana-prod-bucket",
"arn:aws:s3:::oneone-grafana-prod-bucket/*"
]
}
]
}

3. Name and Create Policy

| Setting         | Value                                                                                   |
| --------------- | --------------------------------------------------------------------------------------- |
| **Policy name** | `LGTMStackS3Access`                                                                     |
| **Description** | `Allows the LGTM observability stack to read/write to the S3 observability data bucket` |

### 3: Create IAM Role

1. In the IAM left sidebar, click **Roles**. Click **Create role**.

| Setting                 | Value       |
| ----------------------- | ----------- |
| **Trusted entity type** | AWS service |
| **Use case**            | EC2         |

2. Attach the Policy. In the search box under **Permissions policies**, type `LGTMStackS3Access`.
3. Name and Create Role

| Setting         | Value                                                           |
| --------------- | --------------------------------------------------------------- |
| **Role name**   | `LGTMServerRole`                                                |
| **Description** | `IAM role for the LGTM observability EC2 instance to access S3` |

4. Click next and create the role.

### 4: Attach the Role to EC2 Instance

1. Navigate to **EC2** and click **Instances** in the left sidebar.
2. Click **Actions** -> **Security** -> **Modify IAM role**.
3. Select the **LGTMServerRole** you created in the previous step.
4. Click **Update IAM role**.

### 5: Security Group: 'ov-aws-app'

| Setting                 | Value                                                  |
| ----------------------- | ------------------------------------------------------ |
| **Security group name** | `ov-aws-app`                                           |
| **Description**         | `Security group for Laravel application EC2 instances` |
| **VPC**                 | Select the VPC where your instances reside             |

Inbound Rules:

| Port | Protocol | Source             | Purpose            |
| ---- | -------- | ------------------ | ------------------ |
| 80   | TCP      | ALB / 0.0.0.0/0    | HTTP traffic       |
| 443  | TCP      | ALB / 0.0.0.0/0    | HTTPS traffic      |
| 22   | TCP      | Your IP / VPN CIDR | SSH administration |

Outbound Rules: Keep default (Allow all outbound traffic).

Create Security Group.

### 6: Security Group: 'ov-aws-grafana'

| Setting                 | Value                                                              |
| ----------------------- | ------------------------------------------------------------------ |
| **Security group name** | `ov-aws-grafana`                                                    |
| **Description**         | `LGTM Observability Stack - allows inbound from Laravel instances` |
| **VPC**                 | Same VPC as above                                                  |

Inbound Rules:

| Port | Protocol | Source             | Service         | Purpose                    |
| ---- | -------- | ------------------ | --------------- | -------------------------- |
| 3100 | TCP      | `ov-aws-app`       | Loki HTTP API   | Alloy pushes logs          |
| 9009 | TCP      | `ov-aws-app`       | Mimir HTTP API  | Alloy pushes metrics       |
| 4317 | TCP      | `ov-aws-app`       | Tempo OTLP gRPC | Alloy pushes traces (gRPC) |
| 4318 | TCP      | `ov-aws-app`       | Tempo OTLP HTTP | Alloy pushes traces (HTTP) |
| 3000 | TCP      | Your IP / VPN CIDR | Grafana UI      | Web dashboard access       |
| 22   | TCP      | Your IP / VPN CIDR | SSH             | Administration             |

Outbound Rules: Keep default (Allow all outbound traffic).

Create Security Group.

Attach the Security Groups to Instances.

### 7. S3 Lifecycle Policy Configuration

| Age (Days) | Storage Class       | Cost Behavior                          |
| ---------- | ------------------- | -------------------------------------- |
| 0–30       | S3 Standard         | Hot — fast access, higher cost         |
| 30–100     | Intelligent-Tiering | Warm — auto-moves between access tiers |
| 100+       | **Deleted**         | Data permanently removed               |

1. Navigate to S3 and select your bucket. Click the **Management** tab.
2. Click **Create lifecycle rule**.

| Setting                 | Value                              |
| ----------------------- | ---------------------------------- |
| **Lifecycle rule name** | `ObservabilityDataLifecycle`       |
| **Choose a rule scope** | Apply to all objects in the bucket |

3. **Check** ✅ the acknowledgment box: _"I acknowledge that this rule will apply to all objects in the bucket."_
4. **Check** ✅ the following actions:

- ✅ **Move current versions of objects between storage classes**
- ✅ **Expire current versions of objects**

5. Transition Rules.

Under **Transition current versions of objects between storage classes**:

| Setting                        | Value               |
| ------------------------------ | ------------------- |
| **Storage class transition**   | Intelligent-Tiering |
| **Days after object creation** | `30`                |

Under **Expire current versions of objects**:

| Setting                        | Value |
| ------------------------------ | ----- |
| **Days after object creation** | `100` |
