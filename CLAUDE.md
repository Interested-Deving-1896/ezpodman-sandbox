# ezpodman-sandbox — Claude Code Project Brief

## Goal

Ansible automation to provision, configure, and tear down a sandbox
environment for testing `ezpodman` — a lazydocker wrapper for Podman.

## Operator Context

- Learning Ansible; prefer clear, well-commented playbooks over clever abstractions
- Control node: MacBook (Ansible runs locally)
- Incus remote management already configured on Mac via TLS

---

## Repositories

| Repo | URL |
|------|-----|
| This project | `https://forgejo.home.lan/alfon/ezpodman-sandbox.git` |

Installation instructions for ezpodman are in the ezpodman repo itself.

### Instructions to install ezpodman from GitHub

```bash
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/alfonsosanchez12/ezpodman/main/ezpodman -o ~/.local/bin/ezpodman
chmod +x ~/.local/bin/ezpodman
```

---

## Incus Remotes (configured on Mac)

| Remote   | Address                         |
|----------|---------------------------------|
| badger   | <https://badger.home.lan:8443>    |
| endurance| <https://endurance.home.lan:8443> |
| falcon   | <https://falcon.home.lan:8443>    |

Default target remote: `badger`. Selectable at runtime via `-e "incus_remote=badger"`.

---

## Incus Project

- **Name:** `ezpodman-sandbox`
- Ansible owns the full lifecycle: create project → provision VMs → setup → teardown

### VMs (created by Ansible, inside project `ezpodman-sandbox`)

| Name | Distro | Role          | Software |
|------|--------|---------------|----------|
| vm1  | Fedora | local / main  | curl, podman, docker, ezpodman, lazydocker, git, go, ssh, jq, fzf |
| vm2  | Debian | remote/target | podman only |

- VMs live on the internal Incus bridge — no direct LAN IPs
- Ansible connects via `community.general.incus` connection plugin (no SSH needed)
- Podman is **rootless** on both VMs
- VM-2 must have the **systemd user socket** for podman enabled (`podman.socket`)
- Remote podman connectivity VM-1 → VM-2 over SSH is a **core feature** being tested

### Test Containers

| VM  | Containers     |
|-----|----------------|
| vm1 | nginx, caddy   |
| vm2 | nginx, postgres|

Rootless podman only. Use `containers.podman` Ansible module or `podman run`. No Docker Compose.

---

## Ansible Connection

```ini
ansible_connection=community.general.incus
ansible_incus_remote={{ incus_remote }}
ansible_incus_project=ezpodman-sandbox
```

`incus_remote` defaults to `badger` in `group_vars/all.yml`.

`provision.yml` and `nuke.yml` run against `localhost` using the
`community.general.incus` *modules* to drive the Incus API directly.
All other playbooks use the connection plugin to run tasks inside VMs.

---

## Playbooks

| Playbook              | Runs on         | Purpose |
|-----------------------|-----------------|---------|
| `provision.yml`       | localhost       | Create `ezpodman-sandbox` project, launch VM-1 and VM-2, wait for ready |
| `setup.yml`           | vm1, vm2        | Install and configure all software |
| `containers_up.yml`   | vm1, vm2        | Start test containers |
| `containers_down.yml` | vm1, vm2        | Stop and remove test containers |
| `nuke.yml`            | localhost + vms | Full teardown: containers → VMs → project deleted |

`nuke.yml` must delete the `ezpodman-sandbox` Incus project as its final step.
Add a `--tags force` path that skips graceful container shutdown.

---

## Repository Structure

```
ezpodman-sandbox/
├── CLAUDE.md
├── README.md
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       ├── all.yml          # incus_remote: badger
│       ├── local.yml        # vm1-specific vars
│       └── remote.yml       # vm2-specific vars
├── playbooks/
│   ├── provision.yml
│   ├── setup.yml
│   ├── containers_up.yml
│   ├── containers_down.yml
│   └── nuke.yml
└── roles/
    ├── incus_project/       # create/delete project
    ├── podman_setup/        # rootless podman, systemd user socket (vm2)
    └── test_containers/     # container definitions
```

---

## Manual Prerequisites (out of scope for Ansible)

Done once before running any playbook:

1. **SSH key VM-1 → VM-2** — required for ezpodman remote podman over SSH
2. **Step-CA root trust** — run on each VM after provisioning so HTTPS to homelab services works:

   ```bash
   curl -k https://keymaster.home.lan:6443/roots.pem \
     -o /usr/local/share/ca-certificates/keymaster-root.crt
   update-ca-certificates
   ```

---

## Constraints and Preferences

- Ansible collections required: `community.general`, `containers.podman`
- No Docker Compose; use `containers.podman` module or `podman run`
- Playbooks should be idempotent where possible
- Add comments explaining non-obvious Ansible patterns (operator is learning)

## Known Risk

The `community.general.incus` connection plugin fully supports `ansible_incus_remote`
for named remotes. The plugin constructs `incus exec <remote>:<instance> --project <project>`.
No SSH fallback needed. Validate connectivity before first run with:
  `incus project list badger:`

---

## Current Status (session 2026-03-24)

### What is working
- `provision.yml` + `roles/incus_project` — tested and working
- `ezpodman-sandbox` project exists on `badger`
- `ezpodman-local` (Fedora 43) and `podman-remote` (Debian Trixie) are running

### What is scaffolded but not yet tested
- `playbooks/setup.yml` + `roles/podman_setup` — written, not run yet

### What is not yet written
- `playbooks/containers_up.yml`
- `playbooks/containers_down.yml`
- `playbooks/nuke.yml`
- `roles/test_containers/`

### Next step
Run and validate `setup.yml`, then scaffold `containers_up.yml` and `nuke.yml`.

### Decisions and gotchas to remember

**New Incus projects have an empty default profile** (no root disk, no network).
`incus launch` must always pass `--storage {{ storage_pool }}` and `--network {{ network_bridge }}`.
Defaults: `storage_pool: default`, `network_bridge: incusbr0` (in `roles/incus_project/defaults/main.yml`).

**ezpodman is fetched from GitHub**, not Forgejo, to avoid the Step-CA TLS prerequisite
on fresh VMs. URL: `https://raw.githubusercontent.com/alfonsosanchez12/ezpodman/main/ezpodman`

**sandbox_user** (rootless Podman user) defaults to `podman`, defined in
`roles/podman_setup/defaults/main.yml`. All containers, sockets, and binaries belong to this user.

**lazydocker arch mapping** — `ansible_architecture` returns `aarch64` on ARM VMs but
lazydocker release filenames use `arm64`. A mapping var will be needed in `setup.yml`
Play 2 before the download task if running on M2-hosted VMs.

**nuke.yml teardown order** — when deleting the `ezpodman-sandbox` project, any
cached images inside the project must be deleted first, otherwise the project
deletion will fail. Order: containers → VMs → project images → project.

**Read-only `command` tasks** must have `check_mode: false` so `--check` runs can
still query remote state. Write tasks (create, launch) stay check-mode-skippable.

**`ansible.cfg`** at project root sets `inventory = inventory/hosts.ini` and
`roles_path = roles` — always run `ansible-playbook` from the project root.
