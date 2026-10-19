# AWS Systems Manager Session Manager — Secure Access to a Private Linux EC2 Instance

![AWS Systems Manager](https://img.shields.io/badge/AWS-Systems%20Manager-FF9900?logo=amazonaws&logoColor=white)
![Amazon Linux](https://img.shields.io/badge/OS-Amazon%20Linux%202023-232F3E)
![Network](https://img.shields.io/badge/Network-Private%20Subnet-1E7C56)
![Access](https://img.shields.io/badge/Access-No%20SSH%20Required-1675C1)

## Project Overview

This lab demonstrates how to **securely manage a private Amazon Linux 2023 EC2 instance using AWS Systems Manager Session Manager**, without assigning a public IP address, opening inbound SSH (TCP 22), deploying a bastion host, or using a NAT Gateway.

The EC2 instance runs **SSM Agent** and uses an **IAM instance role** to authenticate to AWS. Its outbound HTTPS (TCP 443) connections reach Systems Manager services over **AWS PrivateLink interface VPC endpoints** within the existing VPC.

The lab builds on earlier work configuring a custom VPC, private/public subnets, security groups, IAM roles, and VPC endpoints.

> **Scope:** This README documents the intended lab configuration and validation procedures. Command output depends on the instance and whether the checks have been completed. Screenshots are maintained separately and are not embedded here.

## Learning Objectives

- Configure secure remote administration of private EC2 instances without conventional SSH access.
- Apply `AmazonSSMManagedInstanceCore` through an EC2 IAM instance profile.
- Configure security groups with **no inbound rules on the EC2 instance**.
- Create private interface VPC endpoints for `ssm` and `ssmmessages`.
- Understand private DNS resolution, AWS PrivateLink, and outbound HTTPS connectivity.
- Start and troubleshoot a Session Manager browser-based shell.
- Verify Linux services and network connectivity using administrative commands.
- Identify and remove chargeable resources when testing is complete.

## Architecture

**Diagram:** Place the separately generated architecture image at `architecture.png` in the repository root, alongside this README.

![AWS Session Manager Lab Architecture](architecture.png)

### Connection Flow

```text
Administrator / Web Browser
         |
         | AWS Console → Systems Manager → Start Session
         v
AWS Systems Manager / Session Manager (AWS-managed service)
         ^
         | AWS PrivateLink over HTTPS (TCP 443)
         |
 +---------------------------- zen-vpc (10.0.0.0/16) --------------------------+
 |                                                                             |
 |  Private subnet: 10.0.20.0/24                                               |
 |                                                                             |
 |  +-------------------------------------+                                    |
 |  | EC2: zen-private-linux-ssm-testing  |                                    |
 |  | Amazon Linux 2023                   |                                    |
 |  | SSM Agent + attached IAM role       |                                    |
 |  | No public IP / no inbound SSH       |                                    |
 |  +-------------------+-----------------+                                    |
 |                      | Outbound TCP 443                                     |
 |                      v                                                      |
 |  +---------------------------------------------------------------------+    |
 |  | Interface endpoints (private IP addresses; private DNS enabled)      |    |
 |  | - com.amazonaws.us-east-1.ssm                                       |    |
 |  | - com.amazonaws.us-east-1.ssmmessages                               |    |
 |  +---------------------------------------------------------------------+    |
 +-----------------------------------------------------------------------------+
```

**Important:** The SSM Agent **initiates** its connections outbound. Session Manager does not require an inbound network path to the EC2 instance. The VPC endpoint network interfaces have their own security group that accepts HTTPS from the instance security group.

## Environment and Resource Inventory

| Component | Name / Configuration |
| --- | --- |
| AWS Region | `us-east-1` (N. Virginia) |
| Existing VPC | `zen-vpc` |
| VPC CIDR | `10.0.0.0/16` |
| Existing private subnet | `10.0.20.0/24` |
| EC2 instance name | `zen-private-linux-ssm-testing` |
| Operating system | Amazon Linux 2023 |
| Instance type | `t3.micro` |
| Example root volume | 8 GiB `gp3` (adjust as needed) |
| EC2 security group | `zen-ssm-private-security-group` |
| VPC endpoint security group | `zen-ssm-inbound-security-group` |
| IAM instance role | `zen-ssm-instance-role` |
| IAM managed policy | `AmazonSSMManagedInstanceCore` |
| SSM interface endpoint | `zen-ssm-endpoint` → `com.amazonaws.us-east-1.ssm` |
| Session messaging endpoint | `zen-ssmmessages-endpoint` → `com.amazonaws.us-east-1.ssmmessages` |
| EC2 public IPv4 address | None |
| SSH key pair | Not required for Session Manager |
| NAT Gateway / bastion host | Not required |

> The endpoint names above are friendly **Name tags**. AWS also assigns each endpoint a unique ID such as `vpce-xxxxxxxx`.

## Prerequisites

- An AWS account with permission to create IAM roles, EC2 instances, security groups, and VPC interface endpoints.
- Permission for the AWS Console identity to start Systems Manager sessions (for example, `ssm:StartSession` and associated discovery/session permissions). This is **separate** from the EC2 instance's IAM role.
- The existing VPC `zen-vpc` and its private subnet `10.0.20.0/24`.
- VPC DNS resolution and DNS hostnames enabled.
- A private subnet route table retaining its normal local VPC route; no NAT or Internet Gateway route is necessary for this project.
- An Amazon Linux 2023 AMI that contains a functioning SSM Agent (normally preinstalled by AWS).

---

## Implementation

### Step 1 — Create an IAM Role for the EC2 Instance

1. Open **AWS Console → IAM → Roles → Create role**.
2. Select **AWS service** as the trusted entity type.
3. Select **EC2** as the use case.
4. Attach the AWS-managed permission policy `AmazonSSMManagedInstanceCore`.
5. Name the role **`zen-ssm-instance-role`** and create it.

**Why:** This grants SSM Agent the required permissions to register and communicate with Systems Manager without configuring static AWS access keys inside the EC2 instance.

**Security:** Do not attach `AdministratorAccess` for this lab. The Systems Manager managed policy is sufficient for the core agent tasks.

### Step 2 — Create the EC2 Security Group

Open **EC2 → Security Groups → Create security group** and configure:

| Setting | Configuration |
| --- | --- |
| Security group name | `zen-ssm-private-security-group` |
| VPC | `zen-vpc` |
| Inbound rules | **None** |
| Outbound rules | Allow all outbound traffic for this initial lab |

No inbound TCP 22, TCP 3389, or other inbound rule is needed for the EC2 instance.

Although the instance security group initially allows all outbound traffic, **this does not automatically provide internet access**: internet connectivity also requires suitable routes and gateways. For a tighter production design, outbound access can be restricted to TCP 443 to the endpoint security group, plus any other explicitly required destinations.

### Step 3 — Create the Interface Endpoint Security Group

Create a second security group:

| Setting | Configuration |
| --- | --- |
| Security group name | `zen-ssm-inbound-security-group` |
| VPC | `zen-vpc` |
| Inbound type | HTTPS |
| Protocol / port | TCP 443 |
| Inbound source | `zen-ssm-private-security-group` (security group reference) |
| Outbound rules | Keep the default for this initial lab |

**Why:** The endpoint's elastic network interfaces must accept the **outbound-initiated** HTTPS connections coming from the EC2 instance. This inbound rule is on the **endpoint security group**, not on the EC2 security group.

### Step 4 — Verify VPC DNS Settings

1. Open **VPC → Your VPCs → zen-vpc**.
2. Verify **DNS resolution** is enabled.
3. Verify **DNS hostnames** is enabled.
4. If needed, use **Actions → Edit VPC settings** to enable them.

Interface endpoint **Private DNS** maps the public AWS service hostname (for example, `ssm.us-east-1.amazonaws.com`) to the private endpoint IP address inside the VPC. The EC2 instance can therefore reach the service without a public IP or NAT Gateway.

### Step 5 — Create the SSM Interface Endpoint

1. Navigate to **VPC → Endpoints → Create endpoint**.
2. For the name tag, use `zen-ssm-endpoint`.
3. Choose the **AWS services** category.
4. Find and select the **Interface** service:

   ```text
   com.amazonaws.us-east-1.ssm
   ```

5. Select VPC `zen-vpc`.
6. Select the Availability Zone containing the private EC2 subnet, and choose that private subnet (`10.0.20.0/24`).
7. Select IPv4 and enable **Private DNS**.
8. Assign security group `zen-ssm-inbound-security-group` (remove the default security group if it was automatically selected).
9. Leave the endpoint policy at **Full access** for this basic lab.
10. Create the endpoint and wait until its status is **Available**.

### Step 6 — Create the Session Messaging Interface Endpoint

Repeat the previous step with:

- **Name tag:** `zen-ssmmessages-endpoint`
- **Service:** `com.amazonaws.us-east-1.ssmmessages`
- **VPC:** `zen-vpc`
- **Subnet:** `10.0.20.0/24`
- **Private DNS:** Enabled
- **Security group:** `zen-ssm-inbound-security-group`

**Why two endpoints?** The `ssm` endpoint provides access to the Systems Manager API, while `ssmmessages` is used by SSM Agent for Session Manager control/data messaging.

**About `ec2messages`:** Older Systems Manager Agent versions may also use the `ec2messages` endpoint. Modern SSM Agent versions (3.3.40.0 and newer) prefer `ssmmessages` when available. This lab uses the two endpoints above with a current agent. If troubleshooting an older image or agent, review the AWS documentation before deciding whether a third endpoint is needed.

**Routing note:** These are **interface endpoints**, not gateway endpoints. AWS provisions private endpoint network interfaces; there is no requirement to manually add a route-table entry for `ssm` or `ssmmessages`. The existing VPC local route supports communication with the endpoint IPs.

### Step 7 — Launch the Private Amazon Linux Instance

Go to **EC2 → Instances → Launch instance** and enter:

| EC2 configuration | Value |
| --- | --- |
| Name | `zen-private-linux-ssm-testing` |
| AMI | Amazon Linux 2023 |
| Instance type | `t3.micro` |
| Key pair | Proceed without a key pair (for Session Manager-only access) |
| VPC | `zen-vpc` |
| Subnet | Existing private subnet `10.0.20.0/24` |
| Auto-assign public IP | Disabled |
| Security group | **Select existing** `zen-ssm-private-security-group` |
| Storage | Example: 8 GiB `gp3` |
| Advanced details → IAM instance profile | `zen-ssm-instance-role` |

Launch the instance, wait for it to enter the **Running** state, and allow time for SSM Agent to register with Systems Manager.

**Important:** Selecting an IAM role in **Advanced details** is easy to overlook. Missing IAM permissions are a common reason for an unavailable Session Manager connection.

### Step 8 — Start a Session Through the AWS Console

1. Open **EC2 → Instances**.
2. Select `zen-private-linux-ssm-testing`.
3. Click **Connect**.
4. Select the **Session Manager** tab.
5. Click **Connect**.

If IAM permissions, agent registration, endpoint connectivity, and DNS are correct, a browser-based terminal opens **without using SSH**.

With the default Session Manager shell settings on Linux, the shell typically runs as `ssm-user`. Organizations can customize the Session Manager run-as settings, so this is not an absolute guarantee.

---

## Linux Administration and Connectivity Verification

Run the following commands **inside the Session Manager terminal on the EC2 instance**, unless otherwise specified.

### 1. Confirm the current operating-system user

```bash
whoami
id
```

**Purpose:** Identify the login account and its group memberships. The default user is generally `ssm-user`.

### 2. Inspect the instance hostname and OS

```bash
hostname
hostnamectl
cat /etc/os-release
uname -r
```

**Purpose:** Confirm the current hostname, Linux distribution, and kernel version.

### 3. Inspect network interfaces and IP addresses

```bash
ip -4 addr
ip route
```

**Expected:** The instance has a private IPv4 address from `10.0.20.0/24`. The subnet should not require a NAT/default internet route for Session Manager connectivity through the interface endpoints.

### 4. Verify the SSM Agent service

```bash
sudo systemctl status amazon-ssm-agent
```

**Expected:** `active (running)`. Press `q` to leave the interactive service status display if necessary.

For a compact check:

```bash
sudo systemctl is-active amazon-ssm-agent
```

**Expected:** `active`.

If the agent is installed but stopped, enable/start it:

```bash
sudo systemctl enable --now amazon-ssm-agent
```

### 5. Verify DNS resolution for both Systems Manager endpoints

```bash
getent ahostsv4 ssm.us-east-1.amazonaws.com
getent ahostsv4 ssmmessages.us-east-1.amazonaws.com
```

**Expected:** Addresses returned for the SSM hostnames should be **private IPv4 addresses within the VPC**, because private DNS is enabled on the interface endpoints. The exact addresses vary by AWS account and endpoint network interfaces.

### 6. Test basic HTTPS connectivity to the SSM endpoints

```bash
curl -I --connect-timeout 5 --max-time 10 https://ssm.us-east-1.amazonaws.com
curl -I --connect-timeout 5 --max-time 10 https://ssmmessages.us-east-1.amazonaws.com
```

**Interpretation:** A TLS connection and an HTTP response (including an expected `4xx` for an unauthenticated or unsupported request) indicate that the endpoint is reachable. These commands are **network checks**, not functional authentication tests. A successful Session Manager session is the stronger end-to-end validation.

### 7. Demonstrate the lack of general internet egress

```bash
curl -I --connect-timeout 5 --max-time 10 https://example.com
```

**Expected for this design:** The request should time out or fail if the private subnet has **no working internet egress route**, even while access to the AWS SSM interface endpoints succeeds. Note that the EC2 security group's broad outbound rule alone does not grant internet access.

### 8. Optional — Inspect SSM Agent logs

```bash
sudo journalctl -u amazon-ssm-agent --no-pager -n 50
```

If needed, review the agent log file:

```bash
sudo tail -n 50 /var/log/amazon/ssm/amazon-ssm-agent.log
```

**Purpose:** Diagnose agent startup, credential, DNS, connection, or registration issues. Redact account identifiers and sensitive log details before publishing screenshots or output.

### 9. Optional — Start a session using the AWS CLI on your own computer

The following command is run **locally**, not inside the EC2 instance:

```bash
aws ssm start-session --target i-xxxxxxxxxxxxxxxxx --region us-east-1
```

Replace the example instance ID with the actual EC2 instance ID. This requires authenticated local AWS CLI access, the **Session Manager plugin** for AWS CLI, and permissions to start a session.

---

## Security Design Explained

### Why Session Manager Instead of SSH?

Traditional SSH usually requires a network-reachable host, inbound TCP 22, an SSH client, and credentials or a key pair. A private EC2 instance can be reached through a bastion host, VPN, or other approved network path, but those approaches add infrastructure and access-management responsibilities.

With Session Manager:

1. SSM Agent runs inside the instance.
2. Its EC2 IAM role provides temporary AWS credentials for Systems Manager operations.
3. The agent initiates outbound HTTPS connections to the Systems Manager endpoints.
4. Private DNS resolves AWS service hostnames to the VPC endpoint network interfaces.
5. The administrator starts an IAM-authorized session from the AWS Console.
6. The session opens in the browser without exposing an inbound SSH service.

### Security Controls Used

| Control | Benefit |
| --- | --- |
| No public EC2 IP | The instance is not directly addressable from the public internet. |
| Zero EC2 inbound rules | No inbound SSH port needs to be exposed. |
| EC2 IAM instance role | No static access keys embedded in the server. |
| `AmazonSSMManagedInstanceCore` | Provides the agent's core permissions. |
| Interface VPC endpoints | Private connectivity to required AWS services. |
| Endpoint SG with TCP 443 from EC2 SG | Restricts which clients may reach the endpoints. |
| Private DNS enabled | Supports normal AWS service DNS names resolving to private IPs. |
| IAM-controlled session access | User permissions determine who may start sessions. |

**Production considerations:** Apply least-privilege IAM controls and consider Session Manager session preferences, CloudTrail auditing, session logging to S3 or CloudWatch Logs, KMS encryption, and more restrictive outbound security group rules. Using these optional logging/encryption integrations **may require additional VPC endpoints, S3 access, policies, and associated charges** for an instance without internet access.

---

## Troubleshooting

| Symptom | What to verify |
| --- | --- |
| **SSM Agent is not online** | EC2 IAM role includes `AmazonSSMManagedInstanceCore`; agent is running; outbound TCP 443 works; required endpoints are Available. |
| **Session Manager tab is unavailable** | Check that the node appears in **Systems Manager → Managed nodes**, then verify agent registration and region. |
| **AccessDenied on StartSession** | Fix permissions for the **human/console IAM identity**; the instance IAM role alone does not grant the administrator session access. |
| **DNS resolves to public addresses** | Enable VPC DNS resolution and hostnames, endpoint Private DNS, and ensure the instance uses an appropriate DNS resolver. |
| **DNS works but the SSM service is unreachable** | Check EC2 SG outbound TCP 443 and the inbound TCP 443 rule on `zen-ssm-inbound-security-group` referencing `zen-ssm-private-security-group`; inspect NACLs if customized. |
| **One endpoint missing** | Verify both `ssm` and `ssmmessages` endpoints exist in `us-east-1` and show **Available**. |
| **Session starts, but commands fail** | Check Linux user permissions, shell configuration, and command syntax separately from Session Manager connectivity. |
| **Internet tests fail** | Expected if there is no route to a NAT Gateway or other internet egress; this does not prevent private SSM connectivity. |

The original VPC also contains other lab resources (such as public subnet routing and an existing S3 gateway endpoint). They are **not required for this Session Manager access path**.

## Cost Awareness and Cleanup

**Chargeable items** can include:

- EC2 compute while the instance is running.
- EBS storage while the volume exists (including when EC2 is stopped).
- Each **interface VPC endpoint**, billed by endpoint/AZ-hours and data processing. These continue to incur charges while provisioned, even if EC2 is stopped.
- Any extra resources created for monitoring, logs, or encryption.

**After testing:**

1. In **EC2 → Instances**, terminate `zen-private-linux-ssm-testing` when you no longer need it.
2. In **EC2 → Volumes**, verify the root EBS volume was deleted; also check **Snapshots** for any optional snapshots.
3. In **VPC → Endpoints**, delete `zen-ssm-endpoint` and `zen-ssmmessages-endpoint` if not actively using them. This is especially important because interface endpoints have ongoing hourly charges.
4. Optionally delete `zen-ssm-private-security-group` and `zen-ssm-inbound-security-group` after their instances/endpoints no longer reference them.
5. Keep `zen-vpc`, reusable subnets, and `zen-ssm-instance-role` if you plan to use them for future labs. Delete unused IAM and security configuration as appropriate.
6. Review **AWS Billing and Cost Management** to confirm that no unexpected resource charges remain.

> Stopping an instance reduces compute charges but does **not** stop VPC interface endpoint or EBS storage charges.

## What I Learned

This project demonstrates a secure alternative to direct SSH administration of cloud servers. The key concepts are:

- **Network isolation:** Private EC2 instances need not have public IPs or internet access for remote management.
- **Identity-based management:** The instance role enables the agent, while the administrator's own IAM permissions control starting the session.
- **Private service connectivity:** Interface VPC endpoints and private DNS provide access to AWS Systems Manager without NAT.
- **Security-group separation:** The EC2 instance requires no inbound rules, while endpoint interfaces accept the required HTTPS connections from the instance's security group.
- **Linux operations:** Commands such as `systemctl`, `journalctl`, `ip`, `hostnamectl`, `getent`, and `curl` help verify and troubleshoot the environment.
- **Cost hygiene:** EC2, EBS, and interface endpoints must each be reviewed during cleanup.

## Suggested Repository Layout

```text
aws-ssm-session-manager/
├── README.md
├── architecture.png
└── images/                 # Screenshots uploaded separately
```

No screenshots are linked or embedded in this README. They can be uploaded to `images/` independently.

## AWS Documentation / References

- [Use VPC endpoints to improve Systems Manager security](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html)
- [Connect to EC2 using Session Manager](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-with-systems-manager-session-manager.html)
- [Amazon Linux AMIs with SSM Agent preinstalled](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html)
- [Check and start SSM Agent](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent-status-and-restart.html)
- [Session Manager with AWS PrivateLink](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-privatelink.html)

---

**Lab category:** AWS Cloud Administration · Amazon EC2 · IAM · VPC Networking · AWS Systems Manager · Linux Administration · Secure Remote Access
