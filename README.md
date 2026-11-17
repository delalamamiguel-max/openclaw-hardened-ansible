# OpenClaw Hardened Deployment (Ansible)

**Automated Tier 3+ Security Hardening for OpenClaw AI Agents**

This Ansible playbook implements and extends the security hardening measures described in the [OpenClaw Security Guide](https://nextkicklabs.substack.com/p/openclaw-hardened-deployment-security-with-ansible), providing a fully automated deployment with additional defense-in-depth layers.

## 🎯 What This Playbook Does

Deploys a **hardened OpenClaw installation** with:
- **Rootless Podman containers:** Strict isolation running as a non-privileged user.
- **Network egress filtering:** Squid proxy sidecar with a domain allowlist.
- **HTTPS termination:** Caddy reverse proxy with auto-generated self-signed certificates.
- **LiteLLM credential brokering:** OpenClaw never sees real API keys; LiteLLM spoofs models (e.g., Deepseek acting as Claude).
- **Consolidated Configuration:** Single `openclaw.json` master config for Gateway, Tools, and Agents.
- **Automated Identity:** EFF wordlist hostname generation and persistent SSH key management.
- **Multi-OS support:** Native tasks for **Arch Linux** and **Debian/Ubuntu** (AWS ready).
- **Security Monitoring:** Systemd-based weekly audits for prompt injections and blocked domains.

## 📊 Comparison: Article vs. This Implementation

| Feature | Original Article (Tier 3) | This Ansible Implementation |
|---------|---------------------------|------------------------------|
| **Container Runtime** | Docker | **Podman (rootless)** ⭐ |
| **Network Filtering** | Firewall only | **Firewall + Squid egress allowlist** ⭐ |
| **HTTPS** | Optional/Manual | **Caddy reverse proxy (Terminated HTTPS)** ⭐ |
| **Identity Management** | Manual setup | **Automated EFF wordlist generation** |
| **OS Support** | Ubuntu focus | **Arch + Debian/Ubuntu auto-detection** |
| **Deployment Method** | Manual | **Fully automated interactive script** |
| **Monitoring** | Manual cron | **Systemd timers + audit script** |
| **LLM Providers** | Anthropic focus | **Ollama (Deepseek) / Anthropic / OpenAI** |
| **Secrets Management** | Manual generation | **Auto-gen with PERSISTENCE across runs** ⭐ |
| **Access Control** | Token only | **Token + Manual Device Pairing** |

## 📋 Prerequisites

**Local Machine (Controller):**
- Ansible 2.10+
- OpenSSL (for cert generation)
- SSH Client (`ssh-keygen`)
- Python 3.8+

**Target Machine:**
- Arch Linux OR Debian/Ubuntu
- Initial root/sudo access (Password or AWS .pem key)
- 2GB+ RAM

## 🚀 Quick Start

### 1. Prepare
```bash
cd openclaw-hardened-ansible
chmod +x deploy.sh update-allowlist.sh
```

### 2. Deploy
Run the interactive script. It will prompt for your IP, provider, and keys.
```bash
./deploy.sh
```

**AWS/Cloud Example:**
```bash
./deploy.sh \
  --target 54.x.x.x \
  --ssh-user ubuntu \
  --ssh-key ~/my-aws-key.pem \
  --mgmt-cidr 192.168.20.0/24 \
  --provider ollama \
  --model "deepseek-r1:8b" \
  --url "http://10.100.1.25:11434"
```

### 3. Authenticate
Once finished, get your persistent token:
```bash
ssh -i ssh-keys/your-name.pem openclaw@IP "cat ~/openclaw-docker/.env | grep TOKEN"
```
Open **`https://IP:18789`**, click through the SSL warning, and paste the token in Settings.

### 4. Approve Device (Hardening Step 7)
Since device auth is enabled, you must approve your browser from the host CLI:
```bash
# Inside the OpenClaw host
podman exec openclaw-agent openclaw devices pending
podman exec openclaw-agent openclaw devices approve <YOUR_ID>
```

## 🔧 Maintenance

### Update Egress Allowlist
1. Edit `roles/tier3-setup/templates/allowlist.txt.j2`.
2. Run `./update-allowlist.sh -t IP --ssh-user USER --ask-pass`.

### Security Audits
A systemd timer runs `monitor-openclaw.sh` weekly. To run manually:
```bash
sudo /home/openclaw/openclaw-docker/monitor-openclaw.sh
```
Check the reports at `~/openclaw-docker/security-audit-YYYYMMDD.log`.

### Configuration Validation
If you see errors, run the OpenClaw "Doctor" to check the schema:
```bash
podman exec openclaw-agent openclaw doctor
```

## 📁 File Structure
- `deploy.sh`: Main entry point (interactive/CLI).
- `update-allowlist.sh`: Lightweight allowlist updater.
- `ssh-keys/`: Stores generated `.pem` and `.crt` files.
- `roles/tier3-setup/`: The core hardening logic.
- `requirements.yml`: Ansible dependencies (auto-installed).

## 📄 License
Provided as-is for harm-reduction. OpenClaw is architecturally "spicy"—this deployment reduces the blast radius but prompt injection remains an inherent risk of LLMs. Use burner accounts only.


---

# Owned checkout guide

## Purpose and status

This owned reference fork preserves Next-Kick’s deployment guide above. It contains executable Ansible infrastructure automation, not evidence of an owner deployment or a verified security tier. Security and compatibility claims in the upstream text have not been independently tested here.

## Repository map

[`deploy.sh`](deploy.sh) collects inputs and invokes Ansible; [`playbook.yml`](playbook.yml) handles remote identity checks, local key/certificate generation and remote provisioning. [`roles/tier3-setup/`](roles/tier3-setup/) contains OS tasks, security tasks, container configuration and monitoring templates. [`update-allowlist.sh`](update-allowlist.sh) and [`update-allowlist.yml`](update-allowlist.yml) update proxy policy. [`requirements.yml`](requirements.yml) declares the community.general collection; [`ansible.cfg`](ansible.cfg) and [`inventory.ini`](inventory.ini) configure targeting.

## Architecture and integrations

The role selects Arch or Debian-family provisioning, then applies security and container tasks. Templates configure a rootless Podman stack with OpenClaw, Squid and LiteLLM, plus host HTTPS and systemd monitoring. Deployment can stop an existing stack, rebuild images, start services and run OpenClaw Doctor with fixes. These are host mutations, not passive configuration checks.

## Setup and environments

Upstream prerequisites list Ansible 2.10+, Python 3.8+, OpenSSL and SSH on the controller, plus privileged access to a supported target. Actual compatibility is unverified. The wrapper installs Ansible collections and generates a temporary inventory; the playbook creates local SSH keys and certificates. Review target, management access and provider settings privately before any separately authorized run. Do not copy real inventory, keys or provider values into this public README.

## Safe commands

For this documentation review, use only local inspection such as `git status --short`, `git ls-files`, `git diff -- README.md` and `git diff --check`. Read referenced scripts as text; do not install dependencies, execute examples, authenticate, run tests or publish. Neither deploy nor allowlist update is a documentation check. Even an Ansible check-mode or syntax invocation was not used here; no target connection occurred.

## Testing and validation

No deployment, security audit, OpenClaw Doctor, container build or Ansible validation was executed. Static checks cover only the new README guide, added relative paths, whitespace and original-content preservation. Idempotency, OS support, isolation and recovery behavior remain unverified.

## Deployment and rollback

The deployment wrapper and playbook are the implementation entry points, but no owner environment is established by this fork. Before future provisioning, retain verified host recovery access, backups, prior configuration and image versions. Reverting Git does not undo users, firewall/SSH changes, credentials, packages or container state. No automated complete host rollback is documented here.

## Risks and guardrails

The checked-in Ansible configuration disables SSH host-key checking, and container templates include mutable image tags. Review these choices before deployment; do not equate the upstream hardening label with a security guarantee. Preserve the upstream harm-reduction/license statement and EFF wordlist; no standalone license file is tracked and this guide adds no license grant. All infrastructure and provider work remains outside this documentation review.

## Related documentation

Consult the preserved upstream article link and maintenance guidance, [playbook](playbook.yml), [role](roles/tier3-setup/), [Ansible configuration](ansible.cfg) and [collection requirements](requirements.yml).
