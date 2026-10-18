# Python App Server Provisioner — Automated EC2 Provisioning with Terraform and Ansible

Oluwaseyi Michael Falode | Cloud & DevOps Engineering | Toronto, ON

Provision a fully configured Python application server on AWS using Terraform to build the network and compute infrastructure, then automatically configure the instance with Git, Python 3, pip, and a dedicated application user via an Ansible playbook.

---

## Project at a Glance

| Attribute      | Detail                                                                        |
|----------------|-------------------------------------------------------------------------------|
| Tools Used     | Terraform, Ansible, AWS EC2, AWS VPC, AWS SSM Parameter Store                 |
| Platform       | Amazon Web Services (AWS) — us-east-1                                         |
| Languages      | HCL (Terraform), YAML (Ansible)                                               |
| What It Does   | Provisions a VPC with networking, launches an Amazon Linux 2023 EC2 instance, and uses Ansible to install Python 3, pip, git, and a locked-down application user with sudo access |

---

## The Problem This Project Solves

Standing up a Python application server by hand is a repetitive process that does not scale. A typical manual workflow involves logging into the AWS console, configuring networking, launching an instance, SSHing in, installing packages, creating users, and setting permissions — each step an opportunity for inconsistency or error. When a second environment is needed (staging, disaster recovery, new region), the entire process must be repeated from memory.

This project replaces that manual workflow with fully declarative, version-controlled infrastructure. Terraform provisions the VPC, subnets, internet gateway, security groups, and EC2 instance in a single apply. Crucially, Terraform's `file` and `remote-exec` provisioners automatically transfer the Ansible playbook to the instance and execute it immediately after launch, meaning the server is fully configured without a human ever needing to SSH in.

The Ansible playbook applies the principle of least privilege by creating a dedicated `python-app-server` user with wheel group membership, rather than running application workloads as `ec2-user` or root. It then installs only the packages the application requires: Python 3, pip, and git. This separation of infrastructure provisioning (Terraform) from software configuration (Ansible) is a production-grade pattern that makes both layers independently testable and maintainable.

---

## Architecture

```
Developer Workstation
        |
        | terraform apply
        v
+-----------------------------------------------+
|              AWS (us-east-1)                   |
|                                                |
|  +-------------------------------------------+ |
|  |         Terraform-VPC (10.0.0.0/16)       | |
|  |                                           | |
|  |  +---------------------------------------+| |
|  |  |  Ansible-Sandbox-Subnet (10.0.1.0/24) || |
|  |  |                                       || |
|  |  |  +--------------------------------+   || |
|  |  |  |  EC2: app_server (t3.micro)    |   || |
|  |  |  |  Amazon Linux 2023             |   || |
|  |  |  |                                |   || |
|  |  |  |  Terraform provisioners:       |   || |
|  |  |  |  1. file → install_app_server.yaml || |
|  |  |  |  2. remote-exec:               |   || |
|  |  |  |     yum update                 |   || |
|  |  |  |     install ansible            |   || |
|  |  |  |     install java-11-corretto   |   || |
|  |  |  |     ansible-playbook run       |   || |
|  |  |  |                                |   || |
|  |  |  |  Post-Ansible state:           |   || |
|  |  |  |  - git installed               |   || |
|  |  |  |  - Python 3 + pip installed    |   || |
|  |  |  |  - user: python-app-server     |   || |
|  |  |  +--------------------------------+   || |
|  |  +---------------------------------------+| |
|  +-------------------------------------------+ |
|                                                |
|  Security Group: TCP/22, TCP/80, TCP/5000,     |
|                  TCP/1233                      |
+-----------------------------------------------+
        |
  Internet Gateway
        |
  0.0.0.0/0 (outbound)
```

**Provisioning Flow**

1. Terraform creates the VPC, subnet, internet gateway, and route table
2. Terraform creates the security group with inbound rules for SSH, HTTP, and application ports (5000, 1233)
3. Terraform creates the EC2 key pair from the local public key
4. Terraform launches the EC2 instance (Amazon Linux 2023, resolved via SSM)
5. The `file` provisioner copies `install_app_server.yaml` to the instance
6. The `remote-exec` provisioner installs Ansible and Java 11, waits 60 seconds for package management to stabilize, then runs the playbook locally on the instance
7. Ansible installs git, tree, Python 3, and pip, and creates the `python-app-server` user

---

## Repository Structure

```
python-appserver-provisioner/
├── modules/
│   ├── compute/
│   │   ├── main.tf          # EC2 instance with file and remote-exec provisioners
│   │   ├── outputs.tf       # Instance public IP output
│   │   └── variables.tf     # AMI, key paths, security group, subnet variables
│   └── vpc/
│       ├── main.tf          # VPC, IGW, route table, subnet, security group
│       ├── outputs.tf       # Public subnet ID, security group ID, subnet IPs
│       └── variables.tf     # VPC module variables
├── install_app_server.yaml  # Ansible playbook — user creation, Python 3, pip, git
├── main.tf                  # Root module — calls vpc and compute modules
├── variables.tf             # Root variables (region)
└── outputs.tf               # Root outputs
```

---

## Walkthrough

**Step 1 — Initialize and validate Terraform**

```bash
terraform init
terraform validate
terraform plan
```

**Step 2 — Provision the infrastructure and configure the server**

```bash
terraform apply -auto-approve
```

Terraform will:
- Create all network resources
- Launch the EC2 instance
- Copy the Ansible playbook to the instance
- Run `ansible-playbook install_app_server.yaml` on the remote host

**Step 3 — Verify provisioning by SSHing in**

```bash
ssh -i ~/.ssh/id_ed25519 ec2-user@<INSTANCE_PUBLIC_IP>

# Confirm Python 3 is installed
python3 --version

# Confirm pip is installed
pip3 --version

# Confirm the application user exists
id python-app-server
```

**Step 4 — Tear down**

```bash
terraform destroy -auto-approve
```

---

## Ansible Playbook — What It Configures

The `install_app_server.yaml` playbook targets `localhost` and runs with `become: true` (root elevation via sudo). It performs three configuration tasks:

```
1. Install system packages: git, tree (via yum)
2. Create user: python-app-server
   - Groups: wheel (sudo access)
   - Shell: /bin/bash
   - Home: /home/python-app-server
3. Install Python 3 and pip for Python 3 (via yum)
```

---

## How to Reproduce

```bash
# Clone the repository
git clone https://github.com/seyifalode-cmd/python-appserver-provisioner.git
cd python-appserver-provisioner

# Export AWS credentials
export AWS_ACCESS_KEY_ID=<your-key-id>
export AWS_SECRET_ACCESS_KEY=<your-secret-key>

# Ensure your SSH keys are at the expected paths
# Default: ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)
# Override via variables.tf if needed

# Initialize and apply
terraform init
terraform apply -auto-approve

# SSH in to confirm configuration
ssh -i ~/.ssh/id_ed25519 ec2-user@$(terraform output -raw instance_public_ip)
python3 --version

# Destroy when done
terraform destroy -auto-approve
```

**Prerequisites**

- Terraform >= 1.5.0
- AWS account with credentials configured
- SSH key pair at `~/.ssh/id_ed25519` and `~/.ssh/id_ed25519.pub`
- AWS provider version ~> 3.44.0

---

*Oluwaseyi Michael Falode · Cybersecurity & Cloud Security Engineer · Toronto, ON*
