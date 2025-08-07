# Ansible-VM-Watch

![Ansible](https://img.shields.io/badge/Ansible-Automation-red?logo=ansible&style=flat-square) 
![Shell Script](https://img.shields.io/badge/Script-Bash-blue?style=flat-square)
![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazon-aws&style=flat-square)

## Overview

**Ansible-VM-Watch** is a powerful automation tool designed to monitor the health of AWS-hosted virtual machines. This project combines **Ansible**, **AWS CLI**, and **Shell scripting** to dynamically manage EC2 inventory, inject SSH keys, tag instances, and extract system health metrics like CPU, RAM, and disk usage.

It is ideal for DevOps teams managing cloud infrastructure and provides a centralized way to audit and monitor server performance.

---

## Features

- Automatically fetches and inventories EC2 instances with specific tags
- Automatically tags EC2 instances (e.g., `web-01`, `web-02`)
- Uses Ansible's AWS EC2 dynamic inventory plugin
- Automatically injects public SSH key into instances
- Collects system information (CPU, RAM, Disk usage) from all instances
- Uses Python virtual environment for isolated package installation

---

## Prerequisites

Ensure the following are installed on your control node:

- Ansible
- AWS CLI
- Python 3 & venv
- jq (for JSON parsing)
- Python packages: `boto3`, `botocore`, `docker`
- Ansible Galaxy collection: `amazon.aws`
---

## Installation & Setup

### Step 1: Install Ansible
```bash
sudo apt update && sudo apt upgrade -y
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```
### Step 2: Install AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
```
### Step 3: Set Up Python Virtual Environment
```bash
sudo apt install python3-venv -y
python3 -m venv ansible-env
source ansible-env/bin/activate
pip install boto3 botocore docker
ansible-galaxy collection install amazon.aws
```
---
## Project Structure
```bash
Ansible-VM-Watch/
├── ansible.cfg
├── inventory/
│   └── aws_ec2.yaml
├── tag_instances.sh
├── inject_ssh_key.sh
├── playbook.yaml
└── README.md
```
---
## Configuration Files

### ansible.cfg
```bash
[defaults]
inventory = ./inventory/aws_ec2.yaml
host_key_checking = False

[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
```

### inventory/aws_ec2.yaml
```bash
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
filters:
  tag:Environment: dev
  instance-state-name: running
compose:
  ansible_host: public_ip_address
keyed_groups:
  - key: tags.Name
    prefix: name
  - key: tags.Environment
    prefix: env
```

### Tagging Script (tag_instances.sh)
```bash
#!/bin/bash
# Fetch instance IDs that match Environment=dev and Role=web
instance_ids=$(aws ec2 describe-instances \
--filters "Name=tag:Environment,Values=dev" "Name=instance-state-name,Values=running" \
--query 'Reservations[*].Instances[*].InstanceId' \
--output text)

# Sort instance IDs deterministically
sorted_ids=($(echo "$instance_ids" | tr '\t' '\n' | sort))

# Rename instances sequentially
counter=1
for id in "${sorted_ids[@]}"; do
  name="web-$(printf "%02d" $counter)"
  echo "Tagging $id as $name"
  aws ec2 create-tags --resources "$id" \
  --tags Key=Name,Value="$name"
  ((counter++))
done
```
##### Usage
```bash
./tag_instances.sh
```

### SSH Public Key Injection Script (inject_ssh_key.sh)
```bash
#!/bin/bash
PEM_FILE="piyush.pem"
PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
USER="ubuntu"
INVENTORY_FILE="inventory/aws_ec2.yaml"

HOSTS=$(ansible-inventory -i $INVENTORY_FILE --list | jq -r '._meta.hostvars | keys[]')
for HOST in $HOSTS; do
  echo "Injecting key into $HOST"
  ssh -o StrictHostKeyChecking=no -i $PEM_FILE $USER@$HOST "
    mkdir -p ~/.ssh && \
    echo \"$PUB_KEY\" >> ~/.ssh/authorized_keys && \
    chmod 700 ~/.ssh && \
    chmod 600 ~/.ssh/authorized_keys
  "
done
```
##### Usage
```bash
./inject_ssh_key.sh
```
## Run Playbook to Monitor VMs
```bash
ansible-playbook playbook.yaml
```
## Contributing

Contributions, issues, and feature requests are welcome!
Please fork the repository, make your changes in a new branch, and submit a pull request.

This project is licensed under the MIT License. See the LICENSE file for details.

