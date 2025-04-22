# AWS DevOps Environment Setup with Ansible

Hello Bynet! Welcome to this DevOps assignment project.

These Ansible playbooks automate setting up a DevOps environment (Docker, Jenkins, Consul, Vault, Nginx, MySQL) on an AWS EC2 instance (Ubuntu 22.04), including user management and dynamic Route53 DNS updates.

## Example Setup (Ofir Gaash)

This repository includes configuration pointing to a live example environment set up by me, Ofir Gaash on AWS:

* **Domain:** `devops1.ofirgaash.click` (A 3$ domain purchased for this project!)
* **Instance:** A running EC2 instance hosting the services (Created on my own AWS account!)
* **Access:** To test against this specific instance, you'll need the SSH `.pem` key and Jenkins credentials. You can request these directly from me via WhatsApp:

[https://api.whatsapp.com/send?phone=972532852599](https://api.whatsapp.com/send?phone=972532852599)

So you can either use this example setup (by requesting the key) or follow the steps below to configure the playbooks for your *own* AWS resources.

## Features

* Ansible-driven setup for core services, users, and AWS integration.
* Dockerized Jenkins, Consul, Vault.
* Dynamic Nginx port configuration via Consul.
* Automatic Route53 DNS updates on boot.
* Secrets managed with Ansible Vault.
* Example Jenkins pipelines included.

## Architecture Overview

Uses Ansible to configure an EC2 instance with Docker (running Jenkins, Consul, Vault), Nginx, MySQL, and scripts for Route53 updates.

## Prerequisites (For Your Own Setup)

1.  **AWS Account & Resources:** Your own EC2 instance (Ubuntu 22.04), SSH key pair (`.pem` file), Route53 Hosted Zone for your domain.
2.  **Local Machine:** `git`, `python3`, `pip3`, `ansible` (`pip3 install ansible`).

## Setup & Configuration

1.  **Clone Repository:**
    ```bash
    git clone [https://github.com/ofirgaash1/aws-devops-project.git](https://github.com/ofirgaash1/aws-devops-project.git)
    cd aws-devops-project
    ```

## 2. Configure Inventory (`inventory/hosts`)

* Edit the `inventory/hosts` file to define your target servers.

* **For direct Ansible runs (outside of Jenkins AWS setup):**
    * **To use Ofir's example:** Leave `ansible_host=devops1.ofirgaash.click` and `ansible_user=ofir1`. If you intend to run Ansible directly against this host, you will need the correct `.pem` file (ask Ofir!) and should update `ansible_ssh_private_key_file` to its local path (e.g., `~/.ssh/ofir-key.pem`).
    * **To use your own server:** Set `ansible_host` to your EC2 Public IP/DNS. Set `ansible_user` (e.g., `ubuntu`). If you intend to run Ansible directly against this host, set `ansible_ssh_private_key_file` to the local path of your `.pem` key.
    * Set key permissions (if you intend to use the key directly with Ansible):
        ```bash
        # Replace with the correct path to the .pem file you are using
        chmod 600 /path/to/your-key.pem
        ```
    * *(Optional)* Add key to agent (if you intend to use the key directly with `ssh` or Ansible):
        ```bash
        # Replace with the correct path to the .pem file you are using
        ssh-add /path/to/your-key.pem

* **Note on Jenkins AWS Setup:** The `setup-aws.yml` playbook, when run via the Jenkins pipeline (`Jenkinsfile.aws`), now uses the SSH private key path defined by the `ANSIBLE_PRIVATE_KEY_FILE` environment variable within the Jenkins pipeline configuration. The `ansible_ssh_private_key_file` setting in the `inventory/hosts` is primarily relevant for direct Ansible executions outside of this automated pipeline.

3.  **Configure Secrets (`vault.yml`):**
    * Copy example:
        ```bash
        cp vault.yml.example vault.yml
        ```
    * Edit `vault.yml` with **your** AWS Access Key ID and Secret Access Key if configuring Route53 for **your** domain. (If only testing against Ofir's instance without DNS updates, these might not be strictly needed for all playbooks, but `setup-aws.yml` requires them).
    * Encrypt the file (remember the password):
        ```bash
        ansible-vault encrypt vault.yml
        ```
    * *(Recommended)* Export vault password:
        ```bash
        export VAULT_PASS='your-vault-password'
        ```
        *(Otherwise, add `--ask-vault-pass` to `ansible-playbook` commands)*

4.  **Configure Dynamic DNS Script (`roles/aws_config/files/update-dns.sh`):**
    * **If using your own domain:** Edit `roles/aws_config/files/update-dns.sh`.
        * Set `DOMAIN_NAME` (e.g., `"your-domain.com."` **<- note the trailing dot**)
        * Set `SUB_DOMAIN_PREFIX` (e.g., `"devops-server"`)
    * **If using Ofir's example:** You can leave this as is, but the `setup-aws.yml` playbook will attempt (and likely fail without Ofir's AWS credentials in `vault.yml`) to update his DNS.
5. **Configure User Public Key (`group_vars/all.yml`)**

* **Important:** The public SSH key that will allow you to access the created users is configured in the `group_vars/all.yml` file.

* **Open `group_vars/all.yml`:**
    * Locate and open the `group_vars/all.yml` file in your project.
    * Inside this file, you will find a variable named `public_key`.

* **Set Your Public Key:**
    * The `public_key` variable determines which SSH key will be added to the `authorized_keys` file of each user defined in the `users` list within this same file. This will grant passwordless SSH access using the corresponding private key.
    * By default, it might look like this:
      ```yaml
      public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
      ```
    * **To enable your SSH access:** Ensure that the path specified in the `lookup('file', '...')` function points to the location of your local public SSH key file (e.g., `~/.ssh/id_rsa.pub`). Update the path if your public key is stored in a different location.

* **How it Works:**
    * The `roles/users/tasks/main.yml` playbook reads the list of users from the `users` variable in `group_vars/all.yml`.
    * For each user, it creates the user account, sets up sudo privileges, adds them to the docker group, creates the `.ssh` directory, and **copies the public key defined in the `public_key` variable (in this file) to the user's `authorized_keys` file.**


6.  **(Optional)** Configure Users (`group_vars/all.yml`):
    * Edit `group_vars/all.yml` to change the list of users to create (default includes `ofir1`, `ofir2`, `ofir3`).

7.  **(Security Notes)** Review:
    * `users` role grants passwordless sudo.
    * `mysql` role uses a hardcoded password (`SuperSecret123`). Secure this in production (e.g., using Vault).

## Usage: Running Playbooks

Run sequentially. Use `--vault-password-file=<(echo "$VAULT_PASS")` if `VAULT_PASS` is set, otherwise use `--ask-vault-pass`.

```bash
# 1. Install base packages
ansible-playbook install-common.yml --vault-password-file=<(echo "$VAULT_PASS")

# 2. Create users & configure SSH access (using YOUR public key from step 5)
ansible-playbook setup-users.yml --vault-password-file=<(echo "$VAULT_PASS")

# 3. Setup Docker, Jenkins, Vault & Consul (via Docker Compose)
#    Access Jenkins: http://<SERVER_IP_OR_DOMAIN>:8080
#    (e.g., http://devops1.ofirgaash.click:8080 for the example setup)
ansible-playbook setup-infra.yml --vault-password-file=<(echo "$VAULT_PASS")

# 4. Configure AWS CLI & Route53 dynamic DNS update script
#    (Requires valid AWS keys in vault.yml for the target domain)
ansible-playbook setup-aws.yml --vault-password-file=<(echo "$VAULT_PASS")

# 5. Setup MySQL Server
ansible-playbook setup-mysql.yml --vault-password-file=<(echo "$VAULT_PASS")

# 6. Setup NGINX (port from Consul - see below)
ansible-playbook setup-nginx.yml --vault-password-file=<(echo "$VAULT_PASS")

# (Optional) 7. Setup OpenVPN
# ansible-playbook setup-vpn.yml --vault-password-file=<(echo "$VAULT_PASS")

```

# Component Details

## Ansible Vault

This project utilizes Ansible Vault for managing sensitive information.

* **Encrypt:** To encrypt a file (e.g., `vault.yml`), use the command:
    ```bash
    ansible-vault encrypt vault.yml
    ```
* **Edit:** To edit an encrypted file:
    ```bash
    ansible-vault edit vault.yml
    ```
* **Decrypt:** To decrypt a file:
    ```bash
    ansible-vault decrypt vault.yml
    ```

## Consul & Dynamic Nginx Port

Nginx's listening port is dynamically configured through Consul.

* Nginx retrieves its port from the Consul key `nginx_port`. The default port is `6789`.
* **Setting the Port:** You can set the desired port value in Consul before running the `setup-nginx.yml` playbook. Execute the following command from your local machine, replacing `<SERVER_IP_OR_DOMAIN>` and `6789` with your server's address and desired port:
    ```bash
    curl --request PUT --data "6789" http://<SERVER_IP_OR_DOMAIN>:8500/v1/kv/nginx_port
    ```
* **Jenkins Integration:** Alternatively, you can specify the Nginx port using the `NGINX_PORT` parameter within the `Jenkinsfile.nginx` pipeline.

## Route53 Dynamic DNS

This setup includes a script for automatically updating a Route53 'A' record.

* **Script Location:** The script responsible for updating the DNS record is located at `roles/aws_config/files/update-dns.sh`.
* **Execution:** This script runs automatically on boot via the `/etc/rc.local` file on the server.
* **Configuration:** The script updates the Route53 'A' record for the subdomain `SUB_DOMAIN_PREFIX.DOMAIN_NAME`. Ensure these variables are correctly configured within the script itself.
* **AWS Credentials:** The script uses AWS credentials stored in `/root/.aws/credentials`. These credentials are set up by the `setup-aws.yml` playbook, which utilizes data from the `vault.yml` file.
* **Logs:** You can find the logs for the DNS update process on the server at `/var/log/cloud-init-dns.log`.

## Jenkins Integration

This project includes Jenkins pipelines for automation.

* **Access UI:** The Jenkins UI can be accessed at `http://<SERVER_IP_OR_DOMAIN>:8080`.

You can actually visit my Jenkins here:

[https://deveops1.ofirgaash.click:8080](https://deveops1.ofirgaash.click:8080) 

For credentials to Jenkins, please ask Ofir.


* **Pipelines:** The Jenkins pipeline definitions are located in the `jenkins/` directory.
    * **Credentials:** The pipelines require a Jenkins "Secret Text" credential named `VAULT_PASS` for accessing Ansible Vault secrets.
    * **`Jenkinsfile.aws`:** This pipeline executes the `setup-aws.yml` playbook. It assumes that your SSH private key is located at `keys/key-pair.pem` within the Jenkins workspace. Adjust this path in the `Jenkinsfile` if your key is stored elsewhere.
    * **`Jenkinsfile.nginx`:** This pipeline runs the `install-common.yml` and `setup-nginx.yml` playbooks. It allows you to set the Consul port dynamically using the `NGINX_PORT` build parameter.

# Project structure
```
.
├── ansible.cfg
├── group_vars
│   └── all.yml
├── install-common.yml
├── inventory
│   └── hosts
├── jenkins
│   ├── Jenkinsfile.aws
│   └── Jenkinsfile.nginx
├── roles
│   ├── aws_config
│   │   ├── files
│   │   │   ├── rc.local
│   │   │   └── update-dns.sh
│   │   └── tasks
│   │       └── main.yml
│   ├── common
│   │   └── tasks
│   │       └── main.yml
│   ├── infra_services
│   │   ├── files
│   │   │   ├── docker-compose.yml
│   │   │   └── Dockerfile.jenkins
│   │   └── tasks
│   │       └── main.yml
│   ├── mysql
│   │   └── tasks
│   │       └── main.yml
│   ├── nginx
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── vars
│   │       └── main.yml
│   ├── openvpn
│   │   ├── clientlist
│   │   ├── defaults
│   │   │   └── main.yml
│   │   ├── files
│   │   │   └── make_config.sh
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── meta
│   │   │   └── main.yaml
│   │   ├── revokelist
│   │   ├── tasks
│   │   │   ├── client_keys.yaml
│   │   │   ├── config.yaml
│   │   │   ├── easy-rsa.yaml
│   │   │   ├── firewall.yaml
│   │   │   ├── install.yaml
│   │   │   ├── main.yaml
│   │   │   ├── revoke.yaml
│   │   │   └── server_keys.yaml
│   │   └── templates
│   │       ├── before.rules.j2
│   │       ├── client.conf.j2
│   │       └── server.conf.j2
│   └── users
│       ├── tasks
│       │   └── main.yml
│       └── templates
│           └── hello_message.j2
├── setup-aws.yml
├── setup-infra.yml
├── setup-mysql.yml
├── setup-nginx.yml
├── setup-users.yml
├── setup-vpn.yml
└── vault.yml.example
```
28 directories, 42 files


