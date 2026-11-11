# Secure Cloud Foundation on AWS

A multi-AZ AWS network with a bastion host and a private EC2 instance that has no public IP. The instance reaches the internet through a NAT gateway and uses an IAM role (no access keys) to read one S3 bucket and one Secrets Manager secret.

Built for Task 4 of the Elevvo Pathways Cloud & DevOps Engineering Internship.

## Features

- VPC across **2 Availability Zones** with public and private subnets
- **Bastion host** as the only entry point; the private instance accepts SSH only from the bastion's security group
- **NAT gateway** for outbound-only internet access from private subnets
- **Least-privilege IAM role**: three actions on three specific resources, no wildcards, no stored keys
- **S3 bucket** with Block Public Access and default KMS encryption
- **Secrets Manager** secret retrieved at runtime through the role

## Architecture

```
                    Internet
                       |
              Internet gateway
                       |
 VPC 10.0.0.0/16
 |
 |-- eu-north-1a
 |     |-- public subnet   10.0.0.0/20     bastion host, NAT gateway
 |     '-- private subnet  10.0.128.0/20   private EC2 (no public IP, IAM role)
 |
 '-- eu-north-1b
       |-- public subnet   10.0.16.0/20
       '-- private subnet  10.0.144.0/20

 Public route table:   0.0.0.0/0 -> internet gateway
 Private route tables: 0.0.0.0/0 -> NAT gateway
```

Access path: `laptop -> bastion (public IP) -> private EC2 (10.0.x.x)`

## Prerequisites

- An AWS account with permissions to create VPC, EC2, IAM, S3 and Secrets Manager resources (use an IAM user, not root)
- An SSH client and an EC2 key pair
- A budget alert, because the NAT gateway is billed hourly

## Build Steps

Region used: `eu-north-1` (Stockholm).

1. **S3:** create a bucket with *Block all public access* on and *default encryption* set to SSE-KMS. Upload a test file.
2. **Secrets Manager:** create a secret named `task4/app-secret` with a dummy value.
3. **IAM:** create a policy from [`iam-policy.json`](iam-policy.json) (replace the placeholders), then a role for **EC2** with that policy attached.
4. **VPC:** use *VPC and more* with CIDR `10.0.0.0/16`, 2 AZs, 2 public and 2 private subnets, 1 NAT gateway and no VPC endpoints.
5. **Security groups:**
   - `task4-bastion-sg`: SSH (22) from your IP
   - `task4-private-sg`: SSH (22) from `task4-bastion-sg`
6. **EC2:**
   - Private instance: private subnet, **no public IP**, IAM role attached
   - Bastion: public subnet, public IP, no role
7. **Connect** through the bastion with an SSH config:

```
Host task4-bastion
    HostName <BASTION_PUBLIC_IP>
    User ec2-user
    IdentityFile ~/.ssh/task4-key.pem

Host task4-private
    HostName <PRIVATE_IP>
    User ec2-user
    IdentityFile ~/.ssh/task4-key.pem
    ProxyJump task4-bastion
```

```bash
ssh task4-private
```

> On Windows OpenSSH, `ProxyJump` can fail with "Connection to UNKNOWN port 65535". Use `ProxyCommand C:/Windows/System32/OpenSSH/ssh.exe -W %h:%p task4-bastion` instead.

## IAM Policy

The role may only:

| Action | Resource |
|---|---|
| `s3:GetObject` | objects in the one bucket |
| `s3:ListBucket` | the one bucket |
| `secretsmanager:GetSecretValue` | the one secret |

Full policy: [`iam-policy.json`](iam-policy.json)

## Verification

Run on the private instance:

| Check | Command | Expected result |
|---|---|---|
| Role in use | `aws sts get-caller-identity` | ARN contains `assumed-role/task4-ec2-role` |
| Read bucket | `aws s3 ls s3://<bucket>` | Lists the test file |
| Read secret | `aws secretsmanager get-secret-value --secret-id task4/app-secret --region eu-north-1` | Returns the secret |
| Least privilege | `aws s3 ls` | `AccessDenied` |
| No stored keys | `ls ~/.aws/credentials` | No such file |
| Outbound via NAT | `curl -sI https://aws.amazon.com \| head -1` | `HTTP/2 200` |

## Screenshots

**VPC resource map**

![VPC resource map](screenshots/08-vpc-resource-map.png)

**Least-privilege IAM policy**

![IAM policy](screenshots/06-iam-policy-json.png)

**S3: public access blocked**

![Block public access](screenshots/02-s3-block-public-access.png)

**Verification from the private instance**

![Verification](screenshots/14-proof-identity-s3-secret.png)

All evidence is in [`screenshots/`](screenshots/).

## Security Notes

- The private instance has no public IP and no stored credentials.
- SSH keys are never committed (`.gitignore` blocks `*.pem` and `*.key`).
- Account IDs are redacted from the screenshots and replaced with placeholders in the policy.
- A single NAT gateway is used to save cost. In production, use one per AZ.

## Cleanup

Everything was deleted after testing to avoid charges: EC2 instances, NAT gateway, Elastic IP, VPC, S3 bucket, secret, IAM role and policy, and the key pair.

## Repository Structure

```
.
├── README.md
├── iam-policy.json
├── .gitignore
└── screenshots/
```

## Author

**Dorcas Torshie Sai**
[@dorcassai-ux](https://github.com/dorcassai-ux)
