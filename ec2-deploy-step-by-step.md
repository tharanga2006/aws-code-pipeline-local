# Deploy to Amazon EC2 with CodePipeline — Step by Step v 1.0

Follow these steps in order. This is based on the [AWS EC2 deploy tutorial](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-ec2-deploy.html) plus the IAM policies you actually need.

**Important**

- Create all resources in the **same AWS Region**.
- The EC2 deploy action works only on **V2** pipelines.
- Linux instances only (Amazon Linux recommended).
- You need **two IAM roles**. They are not interchangeable.

| Role | Who uses it | Job |
|------|-------------|-----|
| EC2 instance role | The EC2 instance (SSM Agent) | Be managed by SSM and download artifacts from S3 |
| CodePipeline service role | CodePipeline | Find instances, send SSM commands, write logs |

---

## How this deploy works

1. You push code to GitHub.
2. CodePipeline stores the files in the **pipeline artifact bucket** (S3).
3. CodePipeline finds EC2 instances by **tag**.
4. CodePipeline sends an SSM command to those instances.
5. **SSM Agent on the instance** downloads the artifact from S3, copies files to a folder, and runs `script.sh`.

CodePipeline does **not** SSH into the instance. If SSM Agent is not running, deploy cannot happen.

### What is SSM Agent?

**SSM Agent** is a small process that runs **on the EC2 instance**. It is how AWS Systems Manager (and this pipeline) talks to the machine without SSH.

It:

- Registers the instance with Systems Manager (“this box is online”).
- Opens a secure outbound channel to AWS.
- Waits for commands (Run Command).
- Runs those commands locally (copy files, run your script).
- Sends status and logs back to AWS.

Amazon Linux AMIs usually already have SSM Agent installed. You still must attach an IAM instance role so the agent can register.

Check on the instance later:

```bash
sudo systemctl status amazon-ssm-agent
```

In the console: **Systems Manager → Fleet Manager**. The instance must be **Online**.

---

## What you will create

- [ ] IAM role for EC2 (`EC2InstanceRole`)
- [ ] 2 Amazon Linux EC2 instances tagged `Name=MyInstances`
- [ ] Extra permissions on the CodePipeline service role
- [ ] `script.sh` in your GitHub repo
- [ ] A V2 CodePipeline (Source → Deploy to EC2)
- [ ] S3 read permission on the EC2 instance role (after the pipeline exists)

Replace these placeholders everywhere you see them:

| Placeholder | Example | Where to get it |
|-------------|---------|-----------------|
| `YOUR-REGION` | `ap-south-1` | AWS console top-right |
| `YOUR-ACCOUNT-ID` | `111122223333` | IAM / account menu |
| `YOUR-PIPELINE-NAME` | `MyPipeline` | You choose this |
| `YOUR-ARTIFACT-BUCKET` | `codepipeline-ap-south-1-123456789` | Pipeline → Settings (after create) |
| Instance tag | `Name` = `MyInstances` | You set this on launch |

---

## Step 1 — Create the EC2 instance role

This role is attached to the **instances**. SSM Agent uses it.

### 1.1 Create the role

1. Open [IAM Roles](https://console.aws.amazon.com/iam/).
2. Choose **Create role**.
3. Trusted entity: **AWS service**.
4. Use case: **EC2**.
5. Choose **Next**.

### 1.2 Attach SSM policies

Search for and select **both**:

- `AmazonSSMManagedInstanceCore` — **required**. Lets the instance register with SSM and accept Run Command.
- `AmazonSSMManagedEC2InstanceDefaultPolicy` — tutorial also attaches this for Default Host Management.

Choose **Next**.

### 1.3 Name and create

1. Role name: `EC2InstanceRole`
2. Choose **Create role**.

**Why these policies**

| Policy | Why |
|--------|-----|
| `AmazonSSMManagedInstanceCore` | Without this, the instance never becomes an SSM managed node, so CodePipeline cannot send deploy commands. |
| `AmazonSSMManagedEC2InstanceDefaultPolicy` | Extra SSM path used by Default Host Management. Safe to keep if you follow the tutorial exactly. |

You will add **S3 artifact-bucket access** in Step 6, after the pipeline exists (you need the bucket name first).

---

## Step 2 — Launch EC2 Linux instances

1. Open [EC2 Launch instances](https://console.aws.amazon.com/ec2/).
2. **Name:** `MyInstances`  
   This sets tag **Key** `Name` and **Value** `MyInstances`. CodePipeline finds instances by this tag.
3. **AMI:** Amazon Linux (Amazon Linux 2 or AL2023). SSM Agent is already on these AMIs.
4. **Instance type:** `t2.micro` or `t3.micro`.
5. **Key pair:** choose or create one (optional for this deploy; SSM does not need SSH).
6. **Network settings:** keep default. Outbound HTTPS must work (public subnet + IGW, or VPC endpoints for SSM/S3).
7. Expand **Advanced details**.
8. **IAM instance profile:** choose `EC2InstanceRole`.  
   Do **not** leave this blank.
9. **Number of instances:** `2`.
10. Choose **Launch instance**.
11. Wait until both instances are **running**.

### 2.1 Confirm SSM can see them

1. Open **Systems Manager → Fleet Manager** (or **Managed nodes**).
2. Both instances should appear as **Online**.

If they are missing:

- Instance profile is not `EC2InstanceRole`
- Instance has no outbound path to SSM
- Wait 1–2 minutes and refresh

---

## Step 3 — Add EC2-deploy permissions to the CodePipeline service role

CodePipeline uses **this** role, not the instance role. Default pipeline roles often do **not** include EC2 deploy permissions.

### 3.1 Open the service role

1. IAM → **Roles**.
2. Open the role your pipeline will use (often `AWSCodePipelineServiceRole-<region>-<name>`).
3. If you do not have one yet, create a pipeline later with “Create a new service role”, then come back and add this policy to that new role.

### 3.2 Add an inline policy

**Permissions tab → Add permissions → Create inline policy → JSON.**

Paste this. Replace the placeholders.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FindInstancesAndTrackSsm",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "elasticloadbalancing:DescribeTargetGroupAttributes",
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DescribeTargetHealth",
        "ssm:CancelCommand",
        "ssm:DescribeInstanceInformation",
        "ssm:ListCommandInvocations"
      ],
      "Resource": "*"
    },
    {
      "Sid": "WriteDeployLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:YOUR-REGION:YOUR-ACCOUNT-ID:log-group:/aws/codepipeline/YOUR-PIPELINE-NAME:*"
    },
    {
      "Sid": "SendCommandToTaggedInstances",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": "arn:aws:ec2:YOUR-REGION:YOUR-ACCOUNT-ID:instance/*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Name": "MyInstances"
        }
      }
    },
    {
      "Sid": "AllowApprovedSsmDocuments",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": [
        "arn:aws:ssm:YOUR-REGION::document/AWS-RunShellScript",
        "arn:aws:ssm:YOUR-REGION::document/AWS-RunPowerShellScript"
      ]
    }
  ]
}
```

Name the policy: `CodePipelineEC2DeployPermissions`.

Save.

**Skip the ALB statement** unless you use a load balancer target group. If you do, add:

```json
{
  "Sid": "DrainAndRejoinAlbTargets",
  "Effect": "Allow",
  "Action": [
    "elasticloadbalancing:DeregisterTargets",
    "elasticloadbalancing:RegisterTargets"
  ],
  "Resource": "arn:aws:elasticloadbalancing:YOUR-REGION:YOUR-ACCOUNT-ID:targetgroup/YOUR-TARGET-GROUP/*"
}
```

### 3.3 Why each permission

| Permission | Why |
|------------|-----|
| `ec2:DescribeInstances` | Find instances with tag `Name=MyInstances`. |
| `ssm:DescribeInstanceInformation` | Confirm those instances are SSM-online. |
| `ssm:SendCommand` on instances | Send the deploy command to tagged instances only. |
| `ssm:SendCommand` on `AWS-RunShellScript` | The document SSM Agent actually runs (Linux). |
| `ssm:ListCommandInvocations` / `ssm:CancelCommand` | Watch progress and cancel on failure. |
| `logs:CreateLogGroup` / `CreateLogStream` / `PutLogEvents` | Write deploy output to `/aws/codepipeline/YOUR-PIPELINE-NAME`. That is **View details**. |
| ELB register/deregister | Optional. Take instances out of a target group during deploy, then put them back. |

Keep any existing S3 permissions on this role. CodePipeline already needs them to store artifacts between stages.

Official reference: [EC2 action service role permissions](https://docs.aws.amazon.com/codepipeline/latest/userguide/action-reference-EC2Deploy.html).

---

## Step 4 — Add `script.sh` to your GitHub repo

This is the **PostScript** CodePipeline runs on each instance after files are copied.

Create `test/script.sh`:

```bash
#!/bin/bash
echo "Hello World!"
```

Commit and push:

```bash
git add test/script.sh
git commit -m "Adding script.sh."
git push
```

Note the path in the repo: `test/script.sh`. You will enter that path in the pipeline (not a path on the EC2 disk).

---

## Step 5 — Create the pipeline

1. Open [CodePipeline](https://console.aws.amazon.com/codepipeline/).
2. Choose **Create pipeline**.
3. **Build custom pipeline** → **Next**.

### Pipeline settings

4. **Pipeline name:** `MyPipeline` (must match the log ARN in Step 3).
5. Pipeline type: **V2** (console default).
6. **Service role:** **Use existing service role** and pick the role you updated in Step 3.  
   If you create a new role here, go back to Step 3 and attach the inline policy to that new role before the first deploy.
7. Leave **Advanced settings** default → **Next**.

### Source

8. **Source provider:** GitHub (via GitHub App).
9. Choose or create a **connection**.
10. Choose your **repository** and **branch**.
11. **Next**.

### Build

12. Choose **Skip**.

### Deploy

13. Provider: **EC2**.
14. **Instance tag key:** `Name`
15. **Instance tag value:** `MyInstances`
16. **Target directory:** `/home/ec2-user/testhelloworld`  
    CodePipeline creates this folder on the instance.
17. **PostScript:** `test/script.sh`
18. **Next** → review → **Create pipeline**.

The first run may fail until you finish Step 6 (instance cannot read S3 yet). That is expected.

---

## Step 6 — Give the EC2 instance role access to the artifact bucket

The **instance** downloads the zip from S3. The pipeline role cannot do that for it.

### 6.1 Get the bucket name

1. CodePipeline → `MyPipeline` → **Settings**.
2. Copy the **artifact store** bucket name.

### 6.2 Add an inline policy to `EC2InstanceRole`

IAM → Roles → `EC2InstanceRole` → **Add permissions** → **Create inline policy** → JSON.

Paste this. Replace `YOUR-ARTIFACT-BUCKET`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadCodePipelineArtifacts",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::YOUR-ARTIFACT-BUCKET/*"
    },
    {
      "Sid": "ListArtifactBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::YOUR-ARTIFACT-BUCKET"
    }
  ]
}
```

Name it: `ReadCodePipelineArtifacts`. Save.

**Why:** Without `s3:GetObject` on this bucket, SSM Agent gets `Access Denied` and the deploy fails.

Do **not** copy the tutorial’s sample that includes `"Principal": "*"`. That belongs on a bucket policy, not an IAM role policy.

If Fleet Manager shows IAM role `AWS-QuickSetup-SSM-DefaultEC2MgmtRole-<region>` instead of `EC2InstanceRole`, add the same S3 policy to **that** role too.

---

## Step 7 — Run and verify

1. CodePipeline → `MyPipeline` → **Release change** (or push a small commit).
2. Wait until Source and Deploy are green.
3. Open the Deploy action → **View details** for logs (`Hello World!`).

On an instance (Session Manager or SSH):

```bash
ls /home/ec2-user/testhelloworld
```

You should see the repo files, including `test/script.sh`.

---

## Step 8 — Test with a code change

1. Change something in the repo (for example the echo text in `script.sh`).
2. Commit and push.
3. Watch the pipeline run again.
4. Confirm the new output in Deploy logs.

---

## Troubleshooting

| What you see | Likely cause | Fix |
|--------------|--------------|-----|
| Instance not in Fleet Manager | No instance role, or no path to SSM | Attach `EC2InstanceRole`; check outbound HTTPS / VPC endpoints |
| `Access Denied` downloading from S3 | Instance role cannot read artifact bucket | Step 6 S3 policy; confirm bucket name |
| Pipeline cannot find instances | Wrong tag, or missing `ec2:DescribeInstances` | Tag `Name=MyInstances`; Step 3 pipeline role |
| Instances found, command never runs | Missing `ssm:SendCommand` | Step 3: instances **and** `AWS-RunShellScript` |
| `No such file` for PostScript | Wrong path | Use repo path `test/script.sh`, not an EC2 path |
| Deploy succeeds, no logs in console | Missing `logs:*` on pipeline role | Step 3 log group ARN must match pipeline name |
| Action missing in console | V1 pipeline | Create a **V2** pipeline |

SSM Agent on the instance:

```bash
sudo systemctl status amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

---

## Role checklist

### `EC2InstanceRole` (instance profile)

- [ ] Trust: `ec2.amazonaws.com`
- [ ] `AmazonSSMManagedInstanceCore`
- [ ] `AmazonSSMManagedEC2InstanceDefaultPolicy` (tutorial)
- [ ] Inline: `s3:GetObject` on `YOUR-ARTIFACT-BUCKET/*`

### CodePipeline service role

- [ ] Existing pipeline S3 / source permissions (keep them)
- [ ] Inline: `ec2:DescribeInstances`
- [ ] Inline: `ssm:SendCommand`, `DescribeInstanceInformation`, `ListCommandInvocations`, `CancelCommand`
- [ ] Inline: CloudWatch Logs for `/aws/codepipeline/YOUR-PIPELINE-NAME`
- [ ] Optional: ELB target group register/deregister

---

## Official docs

- [Tutorial: Deploy to Amazon EC2 with CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-ec2-deploy.html)
- [EC2 action reference and service role policy](https://docs.aws.amazon.com/codepipeline/latest/userguide/action-reference-EC2Deploy.html)
- [Manage the CodePipeline service role](https://docs.aws.amazon.com/codepipeline/latest/userguide/how-to-custom-role.html)
- [AmazonSSMManagedInstanceCore](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonSSMManagedInstanceCore.html)

word change-2
