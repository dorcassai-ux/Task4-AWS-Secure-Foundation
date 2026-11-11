# Secure Cloud Foundation on AWS

![AWS](https://img.shields.io/badge/AWS-eu--north--1-FF9900?logo=amazonaws&logoColor=white)
![VPC](https://img.shields.io/badge/VPC-Multi--AZ-blue)
![IAM](https://img.shields.io/badge/IAM-Least%20Privilege-green)
![S3](https://img.shields.io/badge/S3-KMS%20encrypted-orange)

A two-Availability-Zone AWS network with public and private subnets, a bastion host, and a private EC2 instance that reaches the internet only outbound through a NAT gateway. The instance uses an IAM role, with no access keys, to read one S3 bucket and one secret. Built as **Task 4** of the Elevvo Pathways Cloud & DevOps Engineering Internship.

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [AWS Resources](#aws-resources)
- [IAM Policy](#iam-policy)
- [Implementation](#implementation)
- [Verification](#verification)
- [Screenshots](#screenshots)
- [Security Design](#security-design)
- [Cost Control and Cleanup](#cost-control-and-cleanup)
- [Troubleshooting](#troubleshooting)
- [Limitations](#limitations)
- [Repository Contents](#repository-contents)
- [Author](#author)

## Features

- VPC spanning **two Availability Zones** with public and private subnets
- Private EC2 instance with **no public IP address**, reaching the internet outbound through a **NAT gateway**
- **Bastion host** as the only way in, with SSH on the private instance allowed only from the bastion's security group
- **IAM role with least privilege**: specific actions on specific ARNs, no wildcards, no stored access keys
- S3 bucket with **Block Public Access** and **default KMS encryption**
- Secret stored in **AWS Secrets Manager** and retrieved at runtime through the role

## Architecture

```
                       Internet
                          |
                 Internet gateway (task4-igw)
                          |
  VPC task4-vpc  10.0.0.0/16
  |
  |-- eu-north-1a
  |     |-- public subnet   10.0.0.0/20    bastion host, NAT gateway
  |     '-- private subnet  10.0.128.0/20  private app server (no public IP, IAM role)
  |
  '-- eu-north-1b
        |-- public subnet   10.0.16.0/20
        '-- private subnet  10.0.144.0/20

  Private route tables:  0.0.0.0/0 -> NAT gateway
  Public route table:    0.0.0.0/0 -> internet gateway
```

Access path: `laptop -> bastion (public IP) -> private server (10.0.x.x)`

## AWS Resources

Region: **Europe (Stockholm), eu-north-1**

| Resource | Name | Key settings |
|---|---|---|
| VPC | `task4-vpc` | `10.0.0.0/16`, DNS hostnames and resolution enabled |
| Public subnets | `task4-subnet-public1-eu-north-1a`, `task4-subnet-public2-eu-north-1b` | `10.0.0.0/20`, `10.0.16.0/20` |
| Private subnets | `task4-subnet-private1-eu-north-1a`, `task4-subnet-private2-eu-north-1b` | `10.0.128.0/20`, `10.0.144.0/20` |
| Internet gateway | `task4-igw` | Attached to the VPC |
| NAT gateway | `task4-nat-public1-eu-north-1a` | Public NAT gateway with one Elastic IP |
| Route tables | `task4-rtb-public`, `task4-rtb-private1-eu-north-1a`, `task4-rtb-private2-eu-north-1b` | Public to IGW, private to NAT |
| Security group | `task4-bastion-sg` | SSH (22) from my IP only |
| Security group | `task4-private-sg` | SSH (22) from `task4-bastion-sg` only |
| EC2 instance | `task4-bastion` | t3.micro, Amazon Linux 2023, public subnet, public IP, no IAM role |
| EC2 instance | `task4-private-app` | t3.micro, Amazon Linux 2023, private subnet, **no public IP**, IAM role attached |
| IAM policy | `task4-ec2-read-policy` | Least-privilege policy (below) |
| IAM role | `task4-ec2-role` | Trusted entity: EC2, policy attached |
| S3 bucket | `task4-secure-bucket-10-11-26` | Block all public access on, default SSE-KMS encryption |
| Secret | `task4/app-secret` | Fake value for the demonstration |

## IAM Policy

The role can do exactly three things. The full file is [`iam-policy.json`](iam-policy.json), with the account ID and secret suffix replaced by placeholders:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadObjectsFromOneBucket",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::task4-secure-bucket-10-11-26/*"
    },
    {
      "Sid": "ListOneBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::task4-secure-bucket-10-11-26"
    },
    {
      "Sid": "ReadOneSecret",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:eu-north-1:<ACCOUNT_ID>:secret:task4/app-secret-<SUFFIX>"
    }
  ]
}
```

| Statement | Why |
|---|---|
| `s3:GetObject` on `bucket/*` | Read files inside the bucket |
| `s3:ListBucket` on `bucket` | List the bucket itself (a separate resource from its objects) |
| `secretsmanager:GetSecretValue` on one ARN | Read only this secret, not every secret |

## Implementation

1. **Safety setup:** IAM user (not root) in Stockholm, with a zero-spend budget.
2. **S3 bucket:** created with Block all public access on and default SSE-KMS encryption; a test file uploaded.
3. **Secret:** `task4/app-secret` created in Secrets Manager.
4. **IAM:** policy `task4-ec2-read-policy` created, then role `task4-ec2-role` (trusted entity EC2) with the policy attached.
5. **VPC:** created with "VPC and more": 2 AZs, 2 public and 2 private subnets, one NAT gateway, no VPC endpoints.
6. **Security groups:** bastion SG (SSH from my IP) and private SG (SSH from the bastion SG).
7. **Instances:** private server launched in a private subnet with no public IP and the IAM role; bastion launched in a public subnet with a public IP.
8. **Access:** SSH from a Windows laptop to the bastion, then to the private server using an SSH config with `ProxyCommand`.
9. **Verification:** the commands below, run on the private server.
10. **Cleanup:** every resource deleted the same day.

SSH config used on the laptop:

```
Host task4-bastion
    HostName <BASTION_PUBLIC_IP>
    User ec2-user
    IdentityFile ~/Downloads/task4-key.pem

Host task4-private
    HostName <PRIVATE_IP>
    User ec2-user
    IdentityFile ~/Downloads/task4-key.pem
    ProxyCommand C:/Windows/System32/OpenSSH/ssh.exe -W %h:%p task4-bastion
```

## Verification

Run on the private server after `ssh task4-private`:

| Check | Command | Result |
|---|---|---|
| Role in use, no keys | `aws sts get-caller-identity` | ARN shows `assumed-role/task4-ec2-role` |
| Read the bucket | `aws s3 ls s3://task4-secure-bucket-10-11-26` | Lists the test file |
| Read the secret | `aws secretsmanager get-secret-value --secret-id task4/app-secret --region eu-north-1` | Returns the secret |
| Least privilege | `aws s3 ls` | **AccessDenied** (`s3:ListAllMyBuckets` is not allowed) |
| No stored credentials | `ls ~/.aws/credentials` | No such file or directory |
| Outbound through NAT | `curl -sI https://aws.amazon.com \| head -1` | `HTTP/2 200` |

The brief mentions running `aws s3 ls`. Listing a specific bucket is the least-privilege version, and the plain command failing with AccessDenied is the intended proof that the role cannot list other buckets.

## Screenshots

### Cost guard

![Zero-spend budget](screenshots/01-budget-guard.png)

### Storage: Block Public Access and KMS encryption

![Block public access](screenshots/02-s3-block-public-access.png)

![Default encryption](screenshots/03-s3-default-encryption.png)

### Secret

![Secret details](screenshots/04-secret-details.png)

### IAM

![IAM policy summary](screenshots/05-iam-policy-summary.png)

![IAM policy JSON](screenshots/06-iam-policy-json.png)

![IAM role with the policy attached](screenshots/07-iam-role.png)

### Network

![VPC resource map](screenshots/08-vpc-resource-map.png)

### Compute and security groups

![EC2 instances](screenshots/09-ec2-instances.png)

![Private security group](screenshots/10-private-sg-details.png)

![Private security group inbound rule](screenshots/11-private-sg-inbound-rule.png)

### Access and verification

![Bastion login](screenshots/12-bastion-login.png)

![Private server login](screenshots/13-private-server-login.png)

![Identity, S3 and secret](screenshots/14-proof-identity-s3-secret.png)

![AccessDenied, no credentials file, outbound through NAT](screenshots/15-proof-least-privilege-nat-no-keys.png)

## Security Design

- **Defence in depth:** the private instance has no public IP, its security group accepts SSH only from the bastion's security group, and the bastion accepts SSH only from one IP address.
- **No static credentials:** the instance uses a role with temporary credentials, so there are no access keys to leak.
- **Least privilege:** three actions on three specific resources, with no `*`.
- **Data at rest:** S3 default encryption with KMS and Block Public Access on all four settings.
- **Centralised secrets:** the application retrieves the secret at runtime instead of storing it on the server.
- **Key handling:** the SSH key pair is never committed (`.gitignore` blocks `*.pem`) and was deleted after use.

## Cost Control and Cleanup

The NAT gateway is billed hourly, so the lab was built, verified and deleted in one day. A zero-spend budget was in place throughout.

Deleted, in order: EC2 instances, NAT gateway, Elastic IP, VPC (subnets, route tables, internet gateway and security groups), S3 bucket (emptied first), secret (scheduled for deletion), IAM role, IAM policy, key pair.

## Troubleshooting

| Problem | Cause | Resolution |
|---|---|---|
| `Invalid parameter "/grant:r"` in PowerShell | The command was split over two lines | Run it as one line: `icacls .\task4-key.pem /grant:r <user>:R` |
| `Load key ... Permission denied` | Permissions were removed from the key file, including mine | Grant read access to my own user, then retry |
| `Connection to UNKNOWN port 65535` with `ProxyJump` | Windows OpenSSH quirk with `ProxyJump` | Use `ProxyCommand ... -W %h:%p task4-bastion` |
| `aws s3 ls` returns AccessDenied | Listing all buckets is not in the policy | Expected. Use `aws s3 ls s3://<bucket>` |
| `ls ~/.aws` shows a `cli` folder | The AWS CLI keeps cache files there | Not credentials. Check `~/.aws/credentials` does not exist |

## Limitations

- **Single NAT gateway** in one AZ, to keep cost low. In production, use one per AZ so that an AZ failure does not cut outbound access for the other.
- SSH keys were used for the demonstration. A production setup would prefer AWS Systems Manager Session Manager and no open SSH port.
- The secret holds a fake value, and automatic rotation was not configured.

## Repository Contents

```
.
├── README.md
├── iam-policy.json     # Least-privilege policy (placeholders for account ID and secret suffix)
├── .gitignore          # Blocks keys and secrets
└── screenshots/        # Evidence for each stage
```

## Author

**Dorcas Torshie Sai**
GitHub: [@dorcassai-ux](https://github.com/dorcassai-ux)
