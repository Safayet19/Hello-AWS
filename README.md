
<h1 align="center">AWS - Practical Cloud Guide</h1>

<p align="center">Essential AWS services, CLI commands, IAM, VPC networking, EC2, S3, monitoring, and hands-on labs.</p>

---

A practical AWS reference with **complete commands**, short explanations, and guided exercises. The five labs cover IAM, a VPC with a public subnet, an EC2 web server, private S3 storage, and CloudWatch monitoring.

> **Read before running:** The CLI labs use **Bash** (Linux, WSL, or Git Bash), **AWS CLI v2**, and an authorized AWS account. Some commands **create billable resources**. Your account, region, free-plan eligibility, and permissions determine whether a lab is available and what it costs. Do not run the full README as one script. Perform each lab in order, review every resource, and execute cleanup.

## Table of Contents

1. [AWS Fundamentals](#1-aws-fundamentals)
2. [IAM and Security](#2-iam-and-security)
3. [AWS CLI](#3-aws-cli)
4. [VPC and Networking](#4-vpc-and-networking)
5. [Security Groups and Network ACLs](#5-security-groups-and-network-acls)
6. [EC2](#6-ec2)
7. [S3](#7-s3)
8. [RDS](#8-rds)
9. [Load Balancing and Auto Scaling](#9-load-balancing-and-auto-scaling)
10. [Monitoring and Auditing](#10-monitoring-and-auditing)
11. [Costs and Cleanup](#11-costs-and-cleanup)
12. [Five Practical AWS Labs](#12-five-practical-aws-labs)

---

## 1. AWS Fundamentals

AWS provides cloud computing services: virtual machines, object storage, networking, databases, monitoring, and security. You can manage them using the **AWS Console**, **AWS CLI**, or Infrastructure as Code tools.

| Concept / service | What it does | Real use |
|---|---|---|
| Region | Geographic area containing AWS infrastructure | Choose `us-east-1` for a lab |
| Availability Zone (AZ) | Separate infrastructure location inside a Region | Spread production workloads across AZs |
| IAM | Identities and access permissions | Decide who can launch EC2 |
| VPC | Isolated virtual network | Control address ranges and routing |
| EC2 | Virtual servers | Run a Linux web application |
| EBS | Block storage attached to EC2 | Store OS and application data |
| S3 | Object storage | Keep backups, static files, and uploads |
| RDS | Managed relational databases | Host MySQL or PostgreSQL |
| ALB / ELB | Route incoming requests to targets | Send HTTP requests to healthy servers |
| Auto Scaling | Adjust EC2 capacity | Add instances as demand grows |
| CloudWatch | Metrics, logs, alarms | Monitor EC2 CPU usage |
| CloudTrail | Records AWS API activity | Investigate who changed resources |
| AWS Budgets | Spending/usage alerts | Warn about approaching a spending limit |

**Console:** After signing in, choose your intended **Region in the top-right selector** before opening EC2, VPC, RDS, or CloudWatch. **IAM is a global service**; most infrastructure examples in this guide use `us-east-1` consistently.

## 2. IAM and Security

IAM policies define **what actions an identity is allowed to perform**. Prefer temporary credentials through IAM Identity Center or roles, rather than permanent access keys. Enable MFA for human access, especially privileged access.

| Item | Meaning | Practical example |
|---|---|---|
| User | IAM identity inside an account | Temporary lab user without a console password |
| Group | Collection of IAM users | `aws-guide-readers` |
| Role | Identity assumed for temporary permissions | An EC2 application accessing S3 |
| Policy | JSON permissions document | Permit `s3:ListAllMyBuckets` |
| `Effect` | `Allow` or `Deny` | `"Effect": "Allow"` |
| `Action` | AWS API operation | `"s3:ListAllMyBuckets"` |
| `Resource` | Resources the policy applies to | `"*"` for actions that require it |
| MFA | Second factor for sign-in | Authenticator application or security key |

**Example IAM policy** (used in Lab 1, `s3-list-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    }
  ]
}
```

This permits listing bucket names, **not** reading objects or deleting buckets. Effective access can still be limited by permissions boundaries, service control policies, or other account controls.

| Task | Complete AWS CLI command | Explanation |
|---|---|---|
| Identify active AWS identity | `aws sts get-caller-identity` | Show account ID and current caller ARN. |
| List IAM groups | `aws iam list-groups --query 'Groups[].GroupName' --output table` | Display available IAM groups. |
| List IAM users | `aws iam list-users --query 'Users[].UserName' --output table` | Display IAM users, if authorized. |
| List account roles | `aws iam list-roles --query 'Roles[].RoleName' --output table` | Display IAM roles, if authorized. |

**Console path:** **IAM > User groups / Users / Roles / Policies**. Lab 1 shows group creation, policy assignment, permission simulation, and cleanup **without creating passwords or access keys**.

## 3. AWS CLI

Install **AWS CLI v2** using the official instructions: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html . A lab or company account may use a preconfigured environment (such as AWS CloudShell); do not overwrite credentials supplied by that environment.

### Authenticate safely

If your organization uses IAM Identity Center, set up a named profile:

```bash
aws configure sso --profile aws-lab
aws sso login --profile aws-lab
aws sts get-caller-identity --profile aws-lab
```

For subsequent **Bash** commands, select that profile:

```bash
export AWS_PROFILE=aws-lab
export AWS_DEFAULT_REGION=us-east-1
export AWS_PAGER=""
aws sts get-caller-identity
```

If CloudShell or a training lab supplies temporary credentials automatically, **skip profile selection** and instead run `export AWS_DEFAULT_REGION=us-east-1` followed by `aws sts get-caller-identity`. Do not paste credential values into files or a README. The labs also use `AWS_REGION=us-east-1`; set it in Section 12.

### Essential CLI options with real examples

| Task | Complete command | Explanation |
|---|---|---|
| Check CLI version | `aws --version` | Confirm CLI is installed. |
| Show current caller | `aws sts get-caller-identity` | Verify the account and principal before making changes. |
| Specify Region | `aws ec2 describe-vpcs --region us-east-1` | Run this request in `us-east-1`. |
| Specify profile | `aws sts get-caller-identity --profile aws-lab` | Use a configured named profile. |
| Print a table | `aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,State.Name]' --output table` | Select useful response fields with `--query` and format them. |
| Filter results | `aws ec2 describe-instances --filters Name=instance-state-name,Values=running` | Restrict results to running instances. |
| List buckets | `aws s3 ls` | Show accessible bucket names. |
| Read command help | `aws ec2 run-instances help` | Open detailed usage documentation. |

**CLI quoting:** Examples in this guide use Bash syntax (`$VARIABLE`, single quotes, `\` line continuations). PowerShell uses different variable and quoting rules.

## 4. VPC and Networking

A VPC controls private IP ranges, subnets, gateways, and routing. A **public subnet** has a route to an Internet Gateway; an EC2 instance also needs a public address and a suitable security-group rule to be reached from the internet.

| Component | Purpose | Example from Lab 2 |
|---|---|---|
| VPC | Private virtual network | `10.20.0.0/16` |
| Public subnet | Smaller VPC IP range with internet route | `10.20.1.0/24` |
| Availability Zone | Location for the subnet | First available AZ in `us-east-1` |
| Internet Gateway (IGW) | Route between the VPC and internet | Gateway attached to the lab VPC |
| Route table | Decides where network packets go | `0.0.0.0/0` to the IGW |
| NAT Gateway | Allows outbound internet from private subnets | **Not used** in labs; can be costly |
| VPC endpoint | Private connectivity to supported AWS services | Optional advanced networking, not required |

**CIDR meaning:** `10.20.0.0/16` describes a range of 65,536 IPv4 addresses, *not* 65,536 usable EC2 addresses. `10.20.1.0/24` contains 256 addresses; AWS reserves five IPv4 addresses per subnet, leaving 251 available for assignment.

| Task | Complete command | Explanation |
|---|---|---|
| List VPCs | `aws ec2 describe-vpcs --query 'Vpcs[].[VpcId,CidrBlock,IsDefault]' --output table` | View VPC IDs and CIDR ranges. |
| List subnets | `aws ec2 describe-subnets --query 'Subnets[].[SubnetId,VpcId,CidrBlock,AvailabilityZone]' --output table` | Find subnets and their AZs. |
| View route tables | `aws ec2 describe-route-tables --query 'RouteTables[].[RouteTableId,VpcId]' --output table` | List configured route tables. |
| List Internet Gateways | `aws ec2 describe-internet-gateways --query 'InternetGateways[].InternetGatewayId' --output table` | Find IGWs. |

**Console path:** **VPC > Your VPCs / Subnets / Route tables / Internet gateways**. Lab 2 creates and connects these resources with full commands.

## 5. Security Groups and Network ACLs

| Feature | Security Group (SG) | Network ACL (NACL) |
|---|---|---|
| Applied to | EC2 network interface / resource | Entire subnet |
| State | Stateful; response traffic automatically allowed | Stateless; reply traffic needs suitable rules |
| Rules | Allow rules | Allow and deny rules |
| Rule evaluation | Combined matching rules | Ordered by rule number |
| Main use | Primary instance-level traffic control | Optional extra subnet-level control |

**Our web lab permits TCP port 80 only from your public IPv4 address (`x.x.x.x/32`)**. It does **not** open SSH (port 22) or allow the whole internet to reach the web server. The default NACL of a newly created VPC normally allows inbound/outbound traffic unless changed. Lab 2 shows the security-group command.

| Task | Complete command | Explanation |
|---|---|---|
| List security groups | `aws ec2 describe-security-groups --query 'SecurityGroups[].[GroupId,GroupName,VpcId]' --output table` | Find SG IDs and their VPCs. |
| Inspect a group's rules | `aws ec2 describe-security-groups --group-ids "$LAB_SG_ID"` | Display the lab group after Lab 2; requires `LAB_SG_ID`. |
| Inspect subnet NACLs | `aws ec2 describe-network-acls --filters Name=vpc-id,Values="$LAB_VPC_ID"` | Show NACLs associated with the lab VPC. |

**Security caution:** Avoid opening administrative ports to `0.0.0.0/0`. In production, also restrict application and database ports to the smallest required sources.

## 6. EC2

An **EC2 instance** is a virtual server. An AMI defines its operating-system image; an instance type defines compute capacity; EBS holds block storage. A key pair is only needed for a compatible remote-login method, not for launching the example HTTP server in Lab 3.

| Item | Purpose | Lab setting |
|---|---|---|
| AMI | OS image | Current Amazon Linux 2023 x86_64 via SSM |
| Instance type | CPU and RAM allocation | `t3.micro` (**not guaranteed free**) |
| Subnet | Network placement | Public subnet from Lab 2 |
| Security group | Inbound/outbound rules | HTTP from your IP only |
| User data | Script executed on first launch | Install and start Nginx |
| Public IPv4 | Reach the web app from your laptop | Allocated in Lab 3; **may be billed** |

### User data Bash example (`user-data.sh`)

```bash
#!/bin/bash
set -e
dnf install -y nginx
printf '<h1>Hello from AWS EC2</h1>\n' > /usr/share/nginx/html/index.html
systemctl enable --now nginx
```

This is intended for **Amazon Linux 2023**, which uses `dnf` and `systemd`. It is delivered to EC2 at launch via `--user-data file://user-data.sh`. The instance must have outbound internet access to download Nginx. User-data execution may finish *after* the instance enters the `running` state.

| Task | Complete command | Explanation |
|---|---|---|
| List instance IDs/states | `aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name]' --output table` | Inspect EC2 inventory. |
| Start a stopped lab instance | `aws ec2 start-instances --instance-ids "$INSTANCE_ID"` | Starts an existing stopped instance; may incur charges. |
| Stop a running lab instance | `aws ec2 stop-instances --instance-ids "$INSTANCE_ID"` | Stops compute; storage can still incur charges. |
| Find public IPv4 | `aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query 'Reservations[0].Instances[0].PublicIpAddress' --output text` | Read the lab instance's current public IP. |
| Inspect instance status | `aws ec2 describe-instance-status --instance-ids "$INSTANCE_ID" --include-all-instances` | Check EC2 instance/system health. |

**Console path:** **EC2 > Instances > Launch instance**; select AMI, instance type, VPC/subnet, security group, and **Advanced details > User data**. Lab 3 shows the equivalent CLI creation and termination.

## 7. S3

S3 stores **objects** in **buckets**. The lab bucket will remain **private**. Bucket names must be globally unique within the applicable namespace, so the lab generates a name using your account ID and a timestamp.

| Task | Complete command | Explanation |
|---|---|---|
| List buckets | `aws s3 ls` | List accessible buckets. |
| List one bucket | `aws s3 ls "s3://$S3_BUCKET/"` | List objects in the bucket from Lab 4. |
| Upload a file | `aws s3 cp sample.txt "s3://$S3_BUCKET/sample.txt"` | Copy a local file to S3. |
| Download a file | `aws s3 cp "s3://$S3_BUCKET/sample.txt" downloaded-sample.txt` | Copy a private object back locally. |
| Copy local folder (preview only) | `aws s3 sync ./website/ "s3://$S3_BUCKET/website/" --dryrun` | Preview uploads without changing the bucket; requires local `website/`. |
| Check versioning status | `aws s3api get-bucket-versioning --bucket "$S3_BUCKET"` | Shows whether versioning is enabled for the lab bucket. |
| Check Block Public Access | `aws s3api get-public-access-block --bucket "$S3_BUCKET"` | Read the public access protection settings. |

**Versioning:** Enabling S3 versioning preserves historical versions when objects are overwritten/deleted. It also affects storage costs and cleanup. For this beginner lab, **leave versioning disabled** so `aws s3 rb ... --force` can clean up. In a versioned bucket, `--force` does **not** delete historical object versions; every version and delete marker must be removed separately.

**Console path:** **S3 > General purpose buckets**. Lab 4 creates the bucket, blocks public access, uploads/downloads objects, and deletes the bucket safely.

## 8. RDS

Amazon RDS manages relational database infrastructure (for example, MySQL and PostgreSQL). You still configure engine/version, credentials or IAM authentication where supported, networking, security groups, backups, and database permissions.

| Term | Meaning | Practical guidance |
|---|---|---|
| DB instance | Managed database server | Choose engine and capacity based on need |
| DB subnet group | VPC subnets available to the database | Normally use private subnets across AZs |
| Security group | Controls DB port access | Allow `3306` for MySQL only from the application SG |
| Automated backup | Point-in-time recovery within configured retention | Review retention and storage costs |
| Snapshot | On-demand backup | Verify retained snapshots after deleting DB instances |
| Multi-AZ | High-availability deployment | Additional costs; not necessary for basic labs |

| Task | Complete command | Explanation |
|---|---|---|
| List RDS databases | `aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,Engine,DBInstanceStatus,PubliclyAccessible]' --output table` | Read DB status and public-access setting. |
| List manual snapshots | `aws rds describe-db-snapshots --snapshot-type manual --query 'DBSnapshots[].[DBSnapshotIdentifier,Status]' --output table` | Inspect stored manual snapshots. |
| List DB subnet groups | `aws rds describe-db-subnet-groups --query 'DBSubnetGroups[].DBSubnetGroupName' --output table` | Read configured RDS subnet groups. |

**Console path:** **RDS > Databases > Create database**. This guide **does not create a billable database**. For a real project, use private networking, a restricted SG, strong authentication, and planned backup/deletion policies.

## 9. Load Balancing and Auto Scaling

An **Application Load Balancer (ALB)** handles HTTP/HTTPS and forwards traffic to healthy targets in target groups. An **Auto Scaling Group (ASG)** maintains a desired number of EC2 instances and can scale based on metrics.

| Item | Role | Example |
|---|---|---|
| ALB | HTTP/HTTPS entry point | Route `/api` requests to app instances |
| Listener | Port/protocol configuration | HTTPS on port 443 |
| Target group | Backend targets and health checks | Check `/health` on port 8080 |
| Auto Scaling Group | Controls instance count | Desired 2, min 2, max 4 |
| Launch template | EC2 settings reused by an ASG | AMI, instance type, security groups |

| Task | Complete command | Explanation |
|---|---|---|
| List load balancers | `aws elbv2 describe-load-balancers --query 'LoadBalancers[].[LoadBalancerName,Type,State.Code]' --output table` | Show ALBs/NLBs and states. |
| List target groups | `aws elbv2 describe-target-groups --query 'TargetGroups[].[TargetGroupName,Port,Protocol]' --output table` | Inspect backend group definitions. |
| List Auto Scaling groups | `aws autoscaling describe-auto-scaling-groups --query 'AutoScalingGroups[].[AutoScalingGroupName,MinSize,DesiredCapacity,MaxSize]' --output table` | Show scaling capacity settings. |

**Console paths:** **EC2 > Load balancers / Target groups** and **EC2 > Auto Scaling Groups**. No ALB or ASG is created here: these are separate, potentially billable infrastructure projects.

## 10. Monitoring and Auditing

| AWS service | What it gives you | Example |
|---|---|---|
| CloudWatch Metrics | Numerical resource measurements | EC2 CPU utilization |
| CloudWatch Alarms | Evaluate metric thresholds | CPU average exceeds 70% |
| CloudWatch Logs | Central application/service log storage | EC2 application log group |
| CloudTrail Event history | Recent management-event activity | Who invoked `RunInstances` |

| Task | Complete command | Explanation |
|---|---|---|
| List EC2 CPU metrics | `aws cloudwatch list-metrics --namespace AWS/EC2 --metric-name CPUUtilization` | List matching published metrics. |
| List alarms | `aws cloudwatch describe-alarms --query 'MetricAlarms[].[AlarmName,StateValue]' --output table` | Show alarm names and states. |
| View recent API events | `aws cloudtrail lookup-events --max-results 5 --query 'Events[].[EventTime,EventName,Username]' --output table` | View recent CloudTrail Event history management events, when allowed. |
| List log groups | `aws logs describe-log-groups --query 'logGroups[].logGroupName' --output table` | Find CloudWatch log groups. |

**CloudTrail distinction:** Event history is not a replacement for configuring a full log-retention/auditing strategy. **CloudWatch alarms without alarm actions do not send emails.** Lab 5 creates an observation-only CPU alarm and deletes it afterward.

## 11. Costs and Cleanup

**Do not assume AWS is free.** As of 2026, AWS has account-plan-specific credits, eligibility, and restrictions. Exact eligibility varies by account and when it was opened. A service marked "free-tier eligible" is not necessarily free for *your* account.

| Resource | Cost consideration | Cleanup action |
|---|---|---|
| EC2 | Compute time, EBS, data transfer | Terminate the test instance; verify state |
| Public IPv4 | AWS charges for public IPv4 usage unless covered by applicable benefits | Terminate the instance and release its assigned public IP |
| S3 | Stored objects and requests | Empty and delete the non-versioned lab bucket |
| RDS | DB compute, storage, backups/snapshots | Do not create in this guide |
| ALB / NAT Gateway | May have hourly/data-processing charges | Not created in these labs |
| CloudWatch alarm | May contribute to monitoring charges | Delete the test alarm |
| IAM group/user | IAM users/groups themselves are not charged | Remove temporary identities and policies |

**Console actions:** Open **Billing and Cost Management > Budgets** to configure a small cost budget with an email alert, and **Cost Explorer** to inspect usage when supported. Check the current price/eligibility before launching EC2. Billing data and budget alerts may be delayed; an alert **does not automatically stop resources**.

**Clean up in order:** CloudWatch alarm → EC2 instance → security group → subnet and custom route table → Internet Gateway → VPC. S3 and IAM have separate cleanup. The complete cleanup commands are provided in the labs.

---

## 12. Five Practical AWS Labs

### Before the labs: choose one AWS account and Region

All labs use **Bash syntax**, including environment variables and command substitution. Use one terminal session for Labs 2, 3, and 5, because later steps reuse resource IDs created earlier. Run the profile/authentication instructions in Section 3 first, then:

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION="$AWS_REGION"
export AWS_PAGER=""
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
printf 'AWS account: %s | Region: %s\n' "$ACCOUNT_ID" "$AWS_REGION"
```

**Stop if the account or region is not the one you intend to use.** A restricted AWS Academy/Educate lab may prevent IAM, VPC, or EC2 creation. That is a permission limitation, not a reason to use the root account.

### Lab 1 - IAM user, group, policy, and permission test

**Goal:** Create a temporary group and user, permit listing bucket names only, then inspect the policy using the IAM policy simulator. Requires IAM administration and simulation permissions. **No user password or access key is created.**

**Step 1 — Create the group, user, and JSON policy.**

```bash
aws iam create-group --group-name aws-guide-readers
aws iam create-user --user-name aws-guide-test-user

cat > s3-list-policy.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": "s3:ListAllMyBuckets", "Resource": "*"}
  ]
}
JSON

aws iam put-group-policy \
  --group-name aws-guide-readers \
  --policy-name ListBucketsOnly \
  --policy-document file://s3-list-policy.json

aws iam add-user-to-group \
  --user-name aws-guide-test-user \
  --group-name aws-guide-readers
```

**Step 2 — Verify and simulate.**

```bash
aws iam list-groups-for-user --user-name aws-guide-test-user
aws iam simulate-principal-policy \
  --policy-source-arn "arn:aws:iam::$ACCOUNT_ID:user/aws-guide-test-user" \
  --action-names s3:ListAllMyBuckets s3:DeleteBucket \
  --query 'EvaluationResults[].[EvalActionName,EvalDecision]' \
  --output table
```

**Expected:** Listing buckets is reported `allowed`; deleting a bucket is normally `implicitDeny` under this policy. This is a **simulation**, not a live test login; other account controls can affect real authorization.

**Step 3 — Clean up these exact test resources.**

```bash
aws iam remove-user-from-group --user-name aws-guide-test-user --group-name aws-guide-readers
aws iam delete-group-policy --group-name aws-guide-readers --policy-name ListBucketsOnly
aws iam delete-group --group-name aws-guide-readers
aws iam delete-user --user-name aws-guide-test-user
rm s3-list-policy.json
```

### Lab 2 - VPC, subnet, routes, Internet Gateway, security group

**Goal:** Create an isolated lab VPC `10.20.0.0/16`, one public subnet `10.20.1.0/24`, a route to the internet, and a security group allowing HTTP **only from your IP**. **Do not clean up yet** if you plan to complete Lab 3.

**Step 1 — Create network resources.**

```bash
export LAB_VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.20.0.0/16 \
  --query 'Vpc.VpcId' --output text)

export LAB_AZ=$(aws ec2 describe-availability-zones \
  --filters Name=state,Values=available \
  --query 'AvailabilityZones[0].ZoneName' --output text)

export LAB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$LAB_VPC_ID" \
  --cidr-block 10.20.1.0/24 \
  --availability-zone "$LAB_AZ" \
  --query 'Subnet.SubnetId' --output text)

export LAB_IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway \
  --internet-gateway-id "$LAB_IGW_ID" \
  --vpc-id "$LAB_VPC_ID"

export LAB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$LAB_VPC_ID" \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route \
  --route-table-id "$LAB_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$LAB_IGW_ID"

export LAB_RTA_ID=$(aws ec2 associate-route-table \
  --route-table-id "$LAB_RT_ID" \
  --subnet-id "$LAB_SUBNET_ID" \
  --query 'AssociationId' --output text)
```

**Step 2 — Create the SG and restrict inbound HTTP to your current public IPv4.**

```bash
export LAB_SG_ID=$(aws ec2 create-security-group \
  --group-name aws-guide-http \
  --description 'HTTP from learner IP only' \
  --vpc-id "$LAB_VPC_ID" \
  --query GroupId --output text)

export MY_IP=$(curl -4fsS https://checkip.amazonaws.com | tr -d '\r\n')
echo "Your detected public IPv4: $MY_IP"
aws ec2 authorize-security-group-ingress \
  --group-id "$LAB_SG_ID" \
  --protocol tcp --port 80 --cidr "$MY_IP/32"
```

**Verify the IP:** If the detected address is wrong because of VPN/proxy routing, do not continue; update the rule to your real source IPv4 CIDR. This lab does not open port 22. By default, this new group's outbound traffic is allowed; the VPC's default NACL also allows traffic unless changed.

**Step 3 — Verify the network.**

```bash
printf 'VPC=%s\nSUBNET=%s\nIGW=%s\nROUTE_TABLE=%s\nROUTE_ASSOC=%s\nSG=%s\n' \
  "$LAB_VPC_ID" "$LAB_SUBNET_ID" "$LAB_IGW_ID" "$LAB_RT_ID" "$LAB_RTA_ID" "$LAB_SG_ID"
aws ec2 describe-route-tables --route-table-ids "$LAB_RT_ID"
aws ec2 describe-security-groups --group-ids "$LAB_SG_ID"
```

**Expected:** The subnet is associated with a route table containing `0.0.0.0/0` toward the IGW. The SG has an inbound TCP 80 rule from **your** `/32` IP. **Keep this terminal open for Lab 3.**

### Lab 3 - Launch EC2 and automatically install Nginx

**Requires Lab 2** (`LAB_SUBNET_ID`, `LAB_SG_ID`, and its VPC route). Creating an EC2 instance, EBS volume, and public IPv4 can incur charges. **Check account eligibility first.** Choose a different suitable supported instance type if `t3.micro` is unavailable or not appropriate for your account.

**Step 1 — Create the EC2 startup script.**

```bash
cat > user-data.sh <<'BASH'
#!/bin/bash
set -e
dnf install -y nginx
printf '<h1>Hello from AWS EC2</h1>\n' > /usr/share/nginx/html/index.html
systemctl enable --now nginx
BASH
```

**Step 2 — Resolve the current Amazon Linux 2023 AMI and launch ONE instance.**

```bash
export AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text)

echo "AMI: $AMI_ID"

export INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --credit-specification CpuCredits=standard \
  --count 1 \
  --subnet-id "$LAB_SUBNET_ID" \
  --security-group-ids "$LAB_SG_ID" \
  --associate-public-ip-address \
  --metadata-options HttpTokens=required \
  --user-data file://user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=aws-guide-nginx}]' \
  --query 'Instances[0].InstanceId' --output text)

printf 'Instance ID: %s\n' "$INSTANCE_ID"
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
```

`CpuCredits=standard` avoids the default unlimited-credit mode for the burstable T3 lab example. This does **not** remove ordinary instance charges.

**Step 3 — Find the public IP and test the website.**

```bash
export PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Website: http://$PUBLIC_IP"
curl --max-time 10 "http://$PUBLIC_IP"
```

**Expected:** HTML containing `Hello from AWS EC2`. If the first request fails, allow extra time for user data to finish, then retry. Verify the IP, the route table, and the security-group source IP. `instance-running` does **not** mean Nginx installation has completed.

**Step 4 — Terminate EC2 after Lab 5** (or now if you do not plan to monitor it):

```bash
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID"
```

**Important:** Do not delete the VPC before the instance is terminated. Termination cannot be reversed; ensure this is the instance **you created for this lab**.

### Lab 4 - Create, use, and delete a private S3 bucket

This lab is independent of VPC/EC2. It does **not** enable bucket versioning, so cleanup is simple. Requires permissions to create buckets, set public-access blocking, upload/download, and delete.

```bash
export S3_BUCKET="aws-guide-$ACCOUNT_ID-$(date +%s)"
echo "Lab bucket: $S3_BUCKET"

# This syntax is for us-east-1, the Region selected at the start.
aws s3api create-bucket --bucket "$S3_BUCKET" --region "$AWS_REGION"

aws s3api put-public-access-block \
  --bucket "$S3_BUCKET" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

printf 'Hello from my private AWS S3 bucket\n' > sample.txt
aws s3 cp sample.txt "s3://$S3_BUCKET/sample.txt"
aws s3 ls "s3://$S3_BUCKET/"
aws s3 cp "s3://$S3_BUCKET/sample.txt" downloaded-sample.txt
cat downloaded-sample.txt
aws s3api get-public-access-block --bucket "$S3_BUCKET"
```

**Expected:** One object named `sample.txt`, correct downloaded text, and all four Block Public Access settings enabled. A bucket-name collision is possible because names are globally unique; if one occurs, choose a different valid unique name. For other AWS Regions, S3 bucket creation requires a matching location constraint.

**Cleanup (permanently deletes this lab bucket and its objects):**

```bash
aws s3 rb "s3://$S3_BUCKET" --force
rm sample.txt downloaded-sample.txt
```

`--force` works here **because versioning was not enabled**. Never run this against a bucket containing data you want to keep.

### Lab 5 - Monitor EC2 using CloudWatch

**Requires Lab 3 and its `INSTANCE_ID`**; perform this before terminating EC2. Creates one CPU alarm with **no notification action**. CloudWatch alarm usage may incur a charge.

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name aws-guide-cpu-high \
  --alarm-description 'EC2 CPU over 70 percent' \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value="$INSTANCE_ID" \
  --treat-missing-data missing

aws cloudwatch describe-alarms \
  --alarm-names aws-guide-cpu-high \
  --query 'MetricAlarms[].[AlarmName,StateValue]' \
  --output table
```

**Expected:** A CloudWatch alarm exists. It may initially show `INSUFFICIENT_DATA` until metrics become available. It **does not send an email** because no SNS action was configured. Optionally open **CloudWatch > Alarms / Metrics** in the same Region to inspect the graph.

**Delete the alarm:**

```bash
aws cloudwatch delete-alarms --alarm-names aws-guide-cpu-high
```

### Final cleanup - remove the VPC lab resources

First complete Lab 5 and **terminate EC2** with the Lab 3 cleanup commands. After `aws ec2 wait instance-terminated` succeeds, remove only the resources whose IDs were recorded in Lab 2:

```bash
aws ec2 delete-security-group --group-id "$LAB_SG_ID"
aws ec2 disassociate-route-table --association-id "$LAB_RTA_ID"
aws ec2 delete-subnet --subnet-id "$LAB_SUBNET_ID"
aws ec2 delete-route-table --route-table-id "$LAB_RT_ID"
aws ec2 detach-internet-gateway --internet-gateway-id "$LAB_IGW_ID" --vpc-id "$LAB_VPC_ID"
aws ec2 delete-internet-gateway --internet-gateway-id "$LAB_IGW_ID"
aws ec2 delete-vpc --vpc-id "$LAB_VPC_ID"
```

If AWS reports a dependency error, inspect remaining EC2 instances, network interfaces, security groups, subnets, routes, or gateway attachments **in that VPC** rather than forcing deletion. Run `aws ec2 describe-vpcs --vpc-ids "$LAB_VPC_ID"` after cleanup; a `VpcNotFound` error means the VPC was deleted. Avoid deleting similarly named resources from another project.

### Practice checklist

- [ ] Identify the AWS account and Region using the CLI.
- [ ] Explain IAM users, groups, roles, and policies; simulate a least-privilege policy.
- [ ] Create a VPC and subnet and explain the Internet Gateway route.
- [ ] Compare security groups (stateful) with NACLs (stateless).
- [ ] Launch one EC2 instance with Nginx user data and verify its HTTP response.
- [ ] Create a **private** S3 bucket; upload, download, and delete one object.
- [ ] Create and inspect a CloudWatch CPU alarm.
- [ ] Clean up the alarm, EC2, S3 bucket, IAM test identities, and VPC resources.

---

## Official References

- [AWS documentation](https://docs.aws.amazon.com/)
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
- [IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)
- [VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [EC2 AMI public parameters](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami-parameter-store.html)
- [EC2 CLI reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/)
- [S3 CLI reference](https://docs.aws.amazon.com/cli/latest/reference/s3/)
- [CloudWatch CLI reference](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/)
- [AWS Free Tier eligibility](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier.html)
- [AWS pricing](https://aws.amazon.com/pricing/)

*This guide is for learning in an authorized AWS account. Permission limits, regional availability, prices, and AWS UI labels may change. Review the current AWS documentation and pricing before performing billable actions.*
