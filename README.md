# Infra-Runners

Ansible playbooks to provision GitHub Actions self-hosted runners on Proxmox LXC containers over a Tailscale network.

## Why

GitHub Actions charges per minute for hosted runners. By running self-hosted runners on existing Proxmox infrastructure over Tailscale, CI/CD costs drop to zero — with no dependency on GitHub's shared infrastructure.

This project automates runner provisioning so that adding a runner for a new repository is a single command instead of a manual SSH-and-configure process.

## Prerequisites

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) installed locally
- SSH access to the LXC via Tailscale
- Runner registration token (GitHub → Repo → Settings → Actions → Runners → New self-hosted runner)

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
  - Docker Engine + Buildx
  - GitHub Actions Runner (as a systemd service)

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