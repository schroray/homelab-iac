# homelab-iac

Infrastructure as Code and automation for my Proxmox Home Lab using Ansible.

The repository is used to develop infrastructure automation step by step while keeping provisioning, configuration management, validation and secrets handling logically separated.

## Current Scope

```text
GitHub
   │
   ▼
Windows 11
   │
   ▼
Visual Studio Code
   │
   ▼
WSL2 / Ubuntu
   │
   ├── Python vEnv
   ├── Ansible
   ├── ansible-lint
   ├── proxmoxer
   └── community.proxmox
          │
          │ HTTPS / REST API
          ▼
     Proxmox VE Cluster
     ├── proxmox-01  192.168.0.101
     └── proxmox-02  192.168.0.110
```

Provisioning and configuration management are intentionally separated:

```text
Git
 │
 ▼
Ansible
 ├── Proxmox REST API
 │      └── Infrastructure provisioning
 │
 └── SSH
        └── OS and application configuration
```

---

# 1. Workstation

Development workstation:

```text
Windows 11
└── WSL2
    └── Ubuntu
```

Check WSL:

```powershell
wsl --status
wsl --list --verbose
```

Ubuntu should run using WSL version 2.

The repository is stored inside the native Linux filesystem:

```text
~/git/homelab-iac
```

Avoid using `/mnt/c/...` for the Ansible project unless required.

---

# 2. Visual Studio Code

Visual Studio Code runs on Windows and connects to Ubuntu using the WSL extension.

Required/recommended extensions:

```text
ms-python.python
redhat.ansible
redhat.vscode-yaml
```

Install from the WSL terminal:

```bash
code --install-extension ms-python.python
code --install-extension redhat.ansible
```

Verify:

```bash
code --list-extensions | grep -E 'ms-python|redhat'
```

The project Python interpreter is:

```text
/home/raysc/.venvs/ansible/bin/python
```

Select it using:

```text
Ctrl+Shift+P
Python: Select Interpreter
```

---

# 3. Clone Repository

Verify GitHub SSH authentication:

```bash
ssh -T git@github.com
```

Clone:

```bash
mkdir -p ~/git
cd ~/git

git clone git@github.com:schroray/homelab-iac.git

cd homelab-iac
```

Verify:

```bash
git status
```

---

# 4. Python / Ansible Environment

Create a dedicated virtual environment:

```bash
python3 -m venv ~/.venvs/ansible
```

Activate:

```bash
source ~/.venvs/ansible/bin/activate
```

Upgrade pip:

```bash
python -m pip install --upgrade pip
```

Install Python dependencies:

```bash
python -m pip install -r requirements.txt
```

Install Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

Verify:

```bash
which python
which ansible
ansible --version
```

Expected:

```text
/home/raysc/.venvs/ansible/bin/python
/home/raysc/.venvs/ansible/bin/ansible
```

Deactivate if required:

```bash
deactivate
```

---

# 5. Repository Structure

```text
homelab-iac/
├── .env.example
├── .gitignore
├── README.md
├── ansible.cfg
├── requirements.txt
├── requirements.yml
├── inventory/
│   └── hosts.yml
├── vars/
│   └── proxmox.yml
└── playbooks/
    ├── 01-proxmox-info.yml
    └── 02-proxmox-health.yml
```

Current playbooks:

```text
01-proxmox-info.yml
    Read-only discovery and API validation

02-proxmox-health.yml
    Validate expected cluster nodes and node state
```

Future stages:

```text
03-proxmox-vm-plan.yml
04-proxmox-vm-create.yml
05-linux-bootstrap.yml
06-linux-configure.yml
07-validate.yml
```

---

# 6. Ansible Configuration

The repository contains:

```text
ansible.cfg
```

Verify it is detected:

```bash
ansible --version
```

Expected:

```text
config file = .../homelab-iac/ansible.cfg
```

---

# 7. Proxmox Environment

Cluster:

```text
homelab-pve
```

Nodes:

```text
proxmox-01    192.168.0.101
proxmox-02    192.168.0.110
```

Native Proxmox API:

```text
https://<node>:8006/api2/json/
```

The initial automation endpoint is:

```text
https://192.168.0.101:8006
```

The Proxmox web interface is additionally available through a reverse proxy/load balancer on TCP/443.

For automation development, direct API access on TCP/8006 is preferred to reduce dependencies and simplify troubleshooting.

---

# 8. Proxmox Automation Account

Dedicated service account:

```text
ansible@pve
```

API token:

```text
homelab-iac
```

Initial role:

```text
PVEAuditor
```

Create the account:

```bash
pveum user add ansible@pve \
  --comment "Ansible automation service account"
```

Assign read-only access:

```bash
pveum acl modify / \
  --user ansible@pve \
  --role PVEAuditor
```

Create a privilege-separated API token:

```bash
pveum user token add ansible@pve homelab-iac \
  --privsep 1
```

Assign read-only access to the token:

```bash
pveum acl modify / \
  --token 'ansible@pve!homelab-iac' \
  --role PVEAuditor
```

Verify:

```bash
pveum user permissions ansible@pve

pveum user token permissions \
  ansible@pve \
  homelab-iac
```

The API token secret must never be committed to Git.

---

# 9. Credentials

Copy the example:

```bash
cp .env.example .env
```

Edit:

```bash
nano .env
```

Load credentials:

```bash
source .env
```

Verify without displaying the secret:

```bash
echo "$PROXMOX_USER"
echo "$PROXMOX_TOKEN_ID"

if [ -n "$PROXMOX_TOKEN_SECRET" ]; then
  echo "PROXMOX_TOKEN_SECRET is set"
fi
```

`.env` is ignored by Git.

Never commit:

```text
API token secrets
passwords
SSH private keys
vault passwords
TLS private keys
```

---

# 10. Testing

## Syntax

```bash
ansible-playbook \
  --syntax-check \
  playbooks/01-proxmox-info.yml
```

## Lint

```bash
ansible-lint playbooks/01-proxmox-info.yml
```

Expected:

```text
Passed: 0 failure(s), 0 warning(s)
```

## Proxmox Discovery

```bash
ansible-playbook playbooks/01-proxmox-info.yml
```

Expected:

```text
changed=0
failed=0
```

## Cluster Health

```bash
ansible-playbook playbooks/02-proxmox-health.yml
```

The playbook fails when an expected node is missing or not online.

---

# 11. Network / API Troubleshooting

Check reachability:

```bash
ping 192.168.0.101
```

Check API TCP port:

```bash
nc -vz 192.168.0.101 8006
```

Check API endpoint:

```bash
curl -vk \
  https://192.168.0.101:8006/api2/json/version
```

Without authentication the following response is expected:

```text
HTTP/1.1 401 No ticket
```

This confirms:

```text
Network       OK
TCP/8006      OK
TLS           OK
pveproxy      OK
Proxmox API   OK
```

---

# 12. Troubleshooting Ansible

Check virtual environment:

```bash
which python
which ansible
```

If Ansible is not found:

```bash
source ~/.venvs/ansible/bin/activate
```

Check lint:

```bash
which ansible-lint
ansible-lint --version
```

Check installed collections:

```bash
ansible-galaxy collection list
```

Check specifically:

```bash
ansible-galaxy collection list community.proxmox
```

Avoid installing the same collection simultaneously into multiple Ansible collection paths.

---

# 13. Troubleshooting Proxmox

## Connection refused

Check:

```bash
nc -vz 192.168.0.101 8006
```

On Proxmox:

```bash
ss -lntp | grep 8006
systemctl status pveproxy
```

## Authentication failure

Check:

```bash
echo "$PROXMOX_USER"
echo "$PROXMOX_TOKEN_ID"
```

Verify that the token secret is loaded:

```bash
test -n "$PROXMOX_TOKEN_SECRET" \
  && echo "Token loaded" \
  || echo "Token missing"
```

## No quorum

Check:

```bash
pvecm status
pvecm nodes
```

The lab currently consists of two voting Proxmox nodes.

A two-node cluster loses quorum when one voting node is offline.

For controlled maintenance only, and only when split brain can be excluded:

```bash
pvecm expected 1
```

This is not a permanent HA solution.

A future improvement is a third quorum vote using a QDevice.

---

# 14. Security Principles

```text
Dedicated automation account
        │
        ▼
Privilege-separated API token
        │
        ▼
Least-privilege ACL
        │
        ▼
No secrets in Git
        │
        ▼
TLS verification as target state
        │
        ▼
Provisioning separated from configuration management
        │
        ▼
Validation before changes
```

Current TLS configuration uses:

```yaml
proxmox_validate_certs: false
```

because the Proxmox cluster CA is not yet trusted by the WSL environment.

Target state:

```yaml
proxmox_validate_certs: true
```

with the Proxmox CA installed as a trusted CA.

---

# 15. Current Status

```text
WSL2 / Ubuntu                 PASS
VS Code WSL integration       PASS
Python vEnv                   PASS
Ansible                       PASS
ansible-lint                  PASS
community.proxmox             PASS
proxmoxer                     PASS
GitHub SSH                    PASS
Proxmox API connectivity      PASS
API token authentication      PASS
Read-only discovery           PASS
Cluster node discovery        PASS
```

Current playbook result:

```text
ok=2
changed=0
unreachable=0
failed=0
```

---

# 16. Next Stage

The next implementation stage is VM resource modelling.

The following parameters will be defined before provisioning code is created:

```text
VM name
VMID
target Proxmox node
source template
CPU
memory
disk size
storage
network bridge
VLAN
IP configuration
DNS
gateway
Cloud-Init user
SSH public key
```

Only after these parameters are defined will VM-create permissions be added to the automation account.