# Highly Available WordPress on AWS using CloudFormation

A production-style, highly available WordPress deployment on Amazon Web Services, fully automated using AWS CloudFormation (Infrastructure as Code).

## Architecture Overview

![Architecture Diagram](docs/architecture-diagram.png)

The infrastructure spans two Availability Zones and is split into two independently deployable CloudFormation stacks.

```
Internet
    │
    ▼
Application Load Balancer (Public Subnets)
    │
    ▼
Auto Scaling Group — EC2 t3.micro (Private Subnets)
    │                        │
    ▼                        ▼
Amazon RDS MySQL         Amazon S3
(Multi-AZ + Read Replica)  (Media Storage)
```

## AWS Services Used

| Service | Role |
|---|---|
| VPC | Custom network with public/private subnet isolation |
| EC2 (t3.micro) | WordPress web servers in private subnets |
| Application Load Balancer | Distributes HTTP traffic across EC2 instances |
| Auto Scaling Group | Automatically replaces failed instances; scales on CPU |
| Amazon RDS MySQL 8.0 | Managed database with Multi-AZ and read replica |
| Amazon S3 | Media file offloading via Offload Media plugin |
| CloudWatch Alarms | Triggers scale-out (>70% CPU) and scale-in (<25% CPU) |
| CloudFormation | Full infrastructure provisioning via IaC |
| IAM Role | Grants EC2 instances secure access to S3 (no static keys) |
| NAT Gateway | Outbound internet for private subnet resources |

## Repository Structure

```
├── cloudformation/
│   ├── wordpress-network-stack.yaml        # VPC, subnets, IGW, NAT Gateway, route tables
│   └── wordpress-application-stack.yaml    # EC2, ALB, ASG, RDS, S3, CloudWatch alarms
├── docs/
│   ├── architecture-diagram.png            # System architecture diagram
│   └── screenshots/                        # AWS Console screenshots
├── report/
│   └── AWS_WordPress_Infrastructure_Report.pdf
└── README.md
```

## Networking Stack

**File:** `cloudformation/wordpress-network-stack.yaml`

Provisions all network infrastructure. Outputs are exported for use by the application stack via `Fn::ImportValue`.

| Resource | Details |
|---|---|
| VPC CIDR | 192.168.0.0/16 |
| Public Subnet 1 | 192.168.0.0/24 — us-east-1a |
| Public Subnet 2 | 192.168.1.0/24 — us-east-1b |
| Private Subnet 1 | 192.168.2.0/24 — us-east-1a |
| Private Subnet 2 | 192.168.3.0/24 — us-east-1b |
| Internet Gateway | Attached to VPC for public subnet access |
| NAT Gateway | In Public Subnet 1, used by private subnets |

## Application Stack

**File:** `cloudformation/wordpress-application-stack.yaml`

Provisions all application-layer resources. Imports VPC/subnet IDs from the networking stack.

**Security Groups (least privilege):**
- ALB SG → allows HTTP (port 80) from `0.0.0.0/0`
- Web Server SG → allows HTTP only from ALB SG
- RDS SG → allows MySQL (port 3306) only from Web Server SG

**Auto Scaling:**
- Min: 1 / Desired: 1 / Max: 3
- Scale-out: CPU > 70% for 5 minutes
- Scale-in: CPU < 25% for 5 minutes
- Cooldown: 300 seconds

**RDS:**
- Engine: MySQL 8.0
- Multi-AZ enabled
- Read replica in separate Availability Zone
- Private subnets only, no public access

## Deployment Instructions

### Prerequisites
- AWS CLI configured (`aws configure`)
- An existing EC2 key pair in your target region
- IAM permissions to create VPC, EC2, RDS, S3, ALB, and CloudFormation resources

### Step 1 — Deploy the Networking Stack

```bash
aws cloudformation create-stack \
  --stack-name wordpress-network-stack \
  --template-body file://cloudformation/wordpress-network-stack.yaml \
  --region us-east-1
```

Wait for `CREATE_COMPLETE`:
```bash
aws cloudformation wait stack-create-complete \
  --stack-name wordpress-network-stack \
  --region us-east-1
```

### Step 2 — Deploy the Application Stack

```bash
aws cloudformation create-stack \
  --stack-name wordpress-application-stack \
  --template-body file://cloudformation/wordpress-application-stack.yaml \
  --parameters \
    ParameterKey=NetworkStackName,ParameterValue=wordpress-network-stack \
    ParameterKey=KeyName,ParameterValue=YOUR_KEY_PAIR_NAME \
    ParameterKey=DBPassword,ParameterValue=YOUR_SECURE_PASSWORD \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

### Step 3 — Access WordPress

Once the stack is complete, retrieve the ALB DNS name:
```bash
aws cloudformation describe-stacks \
  --stack-name wordpress-application-stack \
  --query "Stacks[0].Outputs[?OutputKey=='LoadBalancerDNS'].OutputValue" \
  --output text
```

Browse to the URL and complete the WordPress setup wizard.

### Teardown

```bash
# Delete application stack first (depends on networking stack)
aws cloudformation delete-stack --stack-name wordpress-application-stack

# Then delete networking stack
aws cloudformation delete-stack --stack-name wordpress-network-stack
```

## Key Design Decisions

**Why two separate stacks?**
Networking and application resources have different lifecycles. Separating them allows the application stack to be torn down and redeployed without destroying the underlying network — a common real-world pattern used in production environments.

**Why IAM roles instead of access keys for S3?**
Static AWS access keys embedded in application config are a security risk. Attaching an IAM role directly to EC2 instances allows the WordPress Offload Media plugin to authenticate to S3 without storing credentials anywhere in the application.

**Why t3.micro instead of t2.micro?**
Repeated instability during WordPress installation (crashes, resource exhaustion) on t2.micro led to migrating to t3.micro, which uses a newer CPU architecture and provides more consistent burstable performance.

## High Availability Testing

An EC2 instance was manually terminated to simulate an infrastructure failure. The Auto Scaling Group detected the capacity drop and automatically launched a replacement instance, registering it with the ALB target group — all without manual intervention.

![HA Testing](docs/screenshots/HighAvailability_EC2_Instance_Replacement.png)
*Auto Scaling Group automatically launching a replacement EC2 instance after manual termination*

## Screenshots

### Networking

![VPC Architecture](docs/screenshots/VPC_Architecture_Overview.png)
*Custom VPC with public and private subnets across two Availability Zones, Internet Gateway, NAT Gateway, and route table configuration*

### Database

![RDS](docs/screenshots/RDS_Primary_And_Read_Replica.png)
*Amazon RDS MySQL 8.0 primary instance with Multi-AZ enabled and a read replica deployed in a separate Availability Zone*

### Load Balancer

![ALB](docs/screenshots/Application_Load_Balancer_Configuration.png)
*Internet-facing Application Load Balancer distributing HTTP traffic across EC2 instances in private subnets*

![ALB Targets](docs/screenshots/Application_Load_Balancer_Targets.png)
*ALB target group showing registered EC2 instances passing health checks*

### Auto Scaling

![ASG](docs/screenshots/AutoScaling_Group_Configuration.png)
*Auto Scaling Group configured with min/desired/max capacity of 1/1/3 and CPU-based scaling policies*

![Launch Template](docs/screenshots/Launch_Template_Configuration.png)
*Launch template referencing the WordPress AMI, instance type, IAM profile, and security group*

### S3 Media Offloading

![S3](docs/screenshots/S3_Offloaded_Media_Files.png)
*WordPress media files stored in S3 via the Offload Media plugin, removing dependency on local EC2 storage*

![WP Offload](docs/screenshots/WP_Offload_Media_Configuration.png)
*Offload Media plugin configured to use IAM role authentication — no static credentials stored*

### WordPress

![Homepage](docs/screenshots/WordPress_Site_Homepage.png)
*WordPress site accessible via the ALB public DNS name*

![Admin](docs/screenshots/WordPress_Admin_Login.png)
*WordPress admin dashboard confirming successful deployment*

![DB Config](docs/screenshots/WordPress_RDS_Database_Configuration.png)
*WordPress connected to the Amazon RDS MySQL endpoint*

![Media Upload](docs/screenshots/WordPress_Media_Upload_Test.png)
*Media upload test confirming files are routed directly to S3*

### CloudFormation

![Network Stack](docs/screenshots/CloudFormation_Network_Deployment_Timeline.png)
*Networking stack deployment timeline showing all resources created successfully*

![Network Resources](docs/screenshots/CloudFormation_Network_Resources.png)
*Full list of resources provisioned by the networking stack*

![Network Outputs](docs/screenshots/CloudFormation_Network_Outputs.png)
*Networking stack outputs exported for consumption by the application stack*

![App Stack](docs/screenshots/CloudFormation_Application_Deployment_Timeline.png)
*Application stack deployment timeline showing compute, database, and scaling resources*

![App Outputs](docs/screenshots/CloudFormation_Application_Stack_Outputs.png)
*Application stack outputs including ALB DNS, RDS endpoint, read replica endpoint, and S3 bucket name*
