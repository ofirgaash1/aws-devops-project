# AWS DevOps Project – Ansible Automation

## Project Structure

This repo contains roles and playbooks to:

- Provision EC2 instances (pre-existing)
- Install common packages (e.g., vim, net-tools, curl)
- Configure AWS CLI credentials for automatic Route 53 DNS update
- Manage users and dynamic welcome messages

## Roles

- `common`: installs base packages
- `users`: creates users and welcome messages
- `aws_config`: sets up AWS credentials (Vault-encrypted)

## Setup

```bash
ansible-playbook install-common.yml
ansible-playbook setup-users.yml
ansible-playbook setup-aws.yml --ask-vault-pass
```
##  Security
AWS credentials are stored in vault.yml, encrypted with Ansible Vault.

Did not upload vault.yml or .pem keys to GitHub.

## DNS_Auto_Update.sh
This script is placed on each EC2 instance that runs on boot via rc.local and cloud init, fetches the instance's public IP, and updates the Route 53 DNS record accordingly using the AWS CLI.
