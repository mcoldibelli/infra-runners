# Infra-Runners

Ansible playbooks to provision GitHub Actions self-hosted runners on Proxmox LXC containers over a Tailscale network.

## Why

GitHub Actions charges per minute for hosted runners. By running self-hosted runners on existing Proxmox infrastructure over Tailscale, CI/CD costs drop to zero — with no dependency on GitHub's shared infrastructure.

This project automates runner provisioning so that adding a runner for a new repository is a single command instead of a manual SSH-and-configure process.

## Prerequisites

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) installed locally
- Runner registration token (GitHub → Repo → Settings → Actions → Runners → New self-hosted runner)

### LXC setup (one-time)

The target LXC container must be prepared before running the playbooks:
1. **Enable TUN device** on the Proxmox host (Required for Tailscale):
```
  echo "lxc.cgroup2.devices.allow: c 10:200 rwm" >> /etc/pve/lxc/<ID>.conf
  echo "lxc.mount.entry: /dev/net dev/net none bind,create=dir" >> /etc/pve/lxc/<ID>.conf
  pct restart <ID>
```
2. Install and start Tailscale inside the LXC:
```
  curl -fsSL https://tailscale.com/install.sh | sh
  tailscaled --state=/var/lib/tailscale/tailscaled.state &
  tailscale up
▎ Note: LXC containers don't run systemd by default, so tailscaled must be started manually or via a startup script.
```
3. Configure the runner user with passwordless sudo:
```
  echo "runner ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/runner
  chmod 440 /etc/sudoers.d/runner
```
4. Add your SSH public key to the runner user:
```
  mkdir -p /home/runner/.ssh
  echo "<your-public-key>" > /home/runner/.ssh/authorized_keys
  chmod 700 /home/runner/.ssh
  chmod 600 /home/runner/.ssh/authorized_keys
  chown -R runner:runner /home/runner/.ssh
```

## Usage

### Add a runner

```bash
    export RUNNER_HOST=<tailscale-ip>

    ansible-playbook playbooks/setup-runner.yml \
    -e "repo_name=<org>/<repo>" \
    -e "runner_name=<runner-name>" \
    -e "github_token=<TOKEN>"
```
### Remove a runner
```bash
  export RUNNER_HOST=<tailscale-ip>

  ansible-playbook playbooks/remove-runner.yml \
    -e "runner_name=<runner-name>" \
    -e "github_token=<TOKEN>"
```
### What gets installed on the LXC

  - Java 21 (Temurin/Adoptium)
  - Docker Engine + Buildx (skipped if already present)
  - GitHub Actions Runner (as a systemd service)

The playbook is idempotent — safe to run multiple times. Existing software is detected and skipped.

### Project structure

```bash
  infra-runners/
  ├── ansible.cfg
  ├── inventory/
  │   └── hosts.yml              # LXC host (IP via RUNNER_HOST env var)
  ├── playbooks/
  │   ├── setup-runner.yml       # Provision a runner
  │   └── remove-runner.yml      # Remove a runner
  └── roles/
      └── github-runner/
          ├── defaults/main.yml  # Default variables (versions, labels)
          └── tasks/main.yml     # Installation and configuration
```