# AWS DevOps Project – Full Infrastructure Automation

This repository contains Ansible playbooks and roles for deploying and configuring a full DevOps infrastructure on AWS, including:

- VPC + EC2 + networking setup
- Jenkins CI/CD server (with Docker & Docker Compose)
- HashiCorp Vault and Consul
- NGINX web server with Consul-configured port
- OpenVPN server for secure access
- Automatic Route53 DNS update on boot
- Secrets management using Ansible Vault

---

## 🔧 Prerequisites

- AWS CLI access (with a configured `vault.yml` containing credentials)
- Valid EC2 key pair
- DNS domain managed in Route53 (e.g., `ofirgaash.click`)
- Jenkins running in Docker on EC2 (automatically configured via `docker-compose`)

---

## 🚀 Step-by-Step Usage

### 1. Clone the Repository

```bash
git clone https://github.com/your-username/aws-devops-project.git
cd aws-devops-project
```

### 2. Encrypt Secrets with Vault

Encrypt credentials with:

```bash
ansible-vault encrypt vault.yml
```

Vault should include at least:

```yaml
aws_key_path: /home/ubuntu/aws2/keys/key-pair.pem
aws_access_key_id: YOUR_KEY
aws_secret_access_key: YOUR_SECRET
```

### 3. Bring Up Infrastructure (via Jenkins or CLI)

To run manually:

```bash
export VAULT_PASS="your-vault-password"
ansible-playbook setup-infra.yml --vault-password-file=<(echo "$VAULT_PASS")
```

This provisions Jenkins, Vault, and Consul via Docker.

---

### 4. Configure EC2 and VPC

Configure SSH access and basic tools on EC2 machines:

```bash
ansible-playbook install-common.yml --vault-password-file=<(echo "$VAULT_PASS")
```

Create users:

```bash
ansible-playbook setup-users.yml --vault-password-file=<(echo "$VAULT_PASS")
```

---

### 5. Configure Jenkins

Jenkins is auto-configured using Configuration as Code (JCasC).
Make sure Jenkins starts with the volumes:

```yaml
volumes:
  - ./jenkins/jcasc:/var/jenkins_home/casc_configs
```

---

### 6. Set NGINX Port in Consul + Deploy NGINX

Set port in Consul and run NGINX setup:

```bash
export NGINX_PORT=6789
curl --request PUT --data "$NGINX_PORT" http://localhost:8500/v1/kv/nginx_port

ansible-playbook setup-nginx.yml --vault-password-file=<(echo "$VAULT_PASS")
```

---

### 7. Setup OpenVPN

```bash
ansible-playbook setup-vpn.yml --vault-password-file=<(echo "$VAULT_PASS")
```

You can then connect to VPN using the `.ovpn` client file generated.

---

### 8. Enable Auto-DNS Update on Reboot

This is handled by:

- `roles/aws_config/files/update-dns.sh` – invoked via `/etc/rc.local`
- Credentials are written to `/root/.aws/credentials` from Vault secrets

---

## 📁 Directory Reference

| Path                                | Purpose |
|-------------------------------------|---------|
| `setup-infra.yml`                   | Launches Docker containers (Jenkins, Vault, Consul) |
| `setup-users.yml`                   | Adds users and SSH keys |
| `setup-nginx.yml`                   | Installs and configures NGINX |
| `setup-vpn.yml`                     | Deploys OpenVPN |
| `vault.yml`                         | Vault-encrypted credentials |
| `roles/aws_config/files/update-dns.sh` | Auto DNS update on EC2 reboot |
| `roles/infra_services/files/docker-compose.yml` | Defines Jenkins + Vault + Consul stack |
| `roles/nginx/tasks/main.yml`        | NGINX installation + Consul integration |
| `roles/openvpn/`                    | Forked and modified VPN deployment role |

---

## 🔐 Security

- Secrets are encrypted using Ansible Vault
- SSH keys are injected securely and cleaned up post-job
- Jenkins credentials are defined via Configuration as Code (JCasC)

---

## ✅ Notes

- Ensure EC2 security groups allow required ports: `22`, `80`, `443`, `8080`, `8200`, `8500`, `1194/udp` (for VPN)
- Jenkins jobs automatically decrypt Vault secrets and run playbooks

---

## 🧠 Summary

This project automates the full setup of a production-ready DevOps environment on AWS using **Jenkins**, **Ansible**, **Vault**, **Consul**, and **OpenVPN**, all wired together with CI/CD and secret management.
