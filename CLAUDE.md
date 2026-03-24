# ezpodman-sandbox — Claude Code Project Brief

## Goal
Ansible automation to provision, configure, and tear down a sandbox
environment for testing `ezpodman` — a lazydocker wrapper for Podman.

## Operator Context
- Learning Ansible; prefer clear, well-commented playbooks over clever abstractions
- Control node: MacBook (Ansible runs locally)
- Incus remote management already configured on Mac via TLS
- Git credential helper: macOS Keychain (`credential.helper osxkeychain`)

---

## Homelab Infrastructure Overview

### Bare Metal Hosts
| Host      | Role                          | Notes |
|-----------|-------------------------------|-------|
| Badger    | Primary Incus host            | Runs ezpodman-sandbox VMs |
| Endurance | Secondary Incus host          | Runs Step-CA (keymaster), k8s-lab-cka project |
| Falcon    | Tertiary Incus host           | General purpose |

### Incus Remotes (configured on Mac)
| Remote    | Address                          | Default Project |
|-----------|----------------------------------|-----------------|
| badger    | https://badger.home.lan:8443     | default         |
| endurance | https://endurance.home.lan:8443  | k8s-lab-cka     |
| falcon    | https://falcon.home.lan:8443     | default         |

### Networking
- **Router:** Mikrotik Hex-S (RouterOS 7), hostname `HelmsDeep`
- Manages all LAN DNS, DHCP, firewall rules for the homelab
- Home WiFi has its own DHCP but forwards through HelmsDeep
- **Incus bridge:** `incusbr0` — internal NAT network (10.99.167.1/24), used by sandbox VMs
- **Incus macvlan:** `incuslan0` — direct LAN bridge on `enp1s0`, used by VMs needing LAN IPs
  (Forgejo, pihole02). Macvlan note: VMs on this network cannot reach Badger directly (kernel limitation).

### PKI / TLS
- **Step-CA:** `keymaster.home.lan` — deployed on Endurance, LAN macvlan interface
- Behind Caddy reverse proxy, ACME provisioner enabled
- `ACME_DIR="https://keymaster.home.lan:6443/acme/acme/directory"`
- `step` CLI configured on Mac
- All homelab VMs trust the Step-CA root via `/usr/local/share/ca-certificates/keymaster-root.crt`

### Git / Source Control
- **Forgejo:** `forgejo.home.lan` — self-hosted Git, deployed on Badger (macvlan, direct LAN IP)
  - Incus VM: `forgejo`, Debian Trixie, profile `git-host` (2 vCPU, 4GB RAM, 20GB disk)
  - Backend: PostgreSQL (local to VM), user `git`, data at `/var/lib/forgejo`
  - TLS via Caddy + Step-CA ACME (Forgejo itself runs plain HTTP on localhost:3000)
  - Config: `/etc/forgejo/app.ini` (read-only after install, edit as root)
  - Forgejo is the source of truth; GitHub is a push mirror (per-repo, via Settings → Repository → Mirror Settings)
  - GitHub PAT type: fine-grained, scoped per repo, `Contents` read/write + `Workflows` if needed
  - This repo (`ezpodman-sandbox`) lives on Forgejo

---

## ezpodman-sandbox Project

### Incus Project
- **Name:** `ezpodman-sandbox`
- **Target remote:** `badger` (default), selectable at runtime via `-e "incus_remote=badger"`
- Ansible owns the full lifecycle: create project → provision VMs → setup → teardown

### VMs (created by Ansible, inside project `ezpodman-sandbox`)
| Name | Distro | Role         | Software |
|------|--------|--------------|----------|
| vm1  | Fedora | local/main   | curl, podman, docker, ezpodman (git repo), lazydocker, git, go, ssh, jq |
| vm2  | Debian | remote/target| podman only |

- VMs live on the internal Incus bridge (`incusbr0`) — no direct LAN IPs
- Ansible connects via `community.general.incus` connection plugin (no SSH needed)
- Podman is **rootless** on both VMs
- VM-2 must have the **systemd user socket** for podman enabled (`podman.socket`)
- **SSH key VM-1 → VM-2** is a manual prerequisite (required for ezpodman remote podman over SSH)

### Test Containers (managed by Ansible)
| VM  | Containers        |
|-----|-------------------|
| vm1 | nginx, caddy      |
| vm2 | nginx, postgres   |

Run via `podman` (rootless). Use `containers.podman` Ansible module or `podman run`.
No Docker Compose.

### Ansible Connection
```ini
ansible_connection=community.general.incus
ansible_incus_remote={{ incus_remote }}
ansible_incus_project=ezpodman-sandbox
```
`incus_remote` defaults to `badger` in `group_vars/all.yml`.

`provision.yml` and `nuke.yml` run against `localhost` using the
`community.general.incus` *modules* to drive the Incus API directly.

### Playbooks
| Playbook             | Runs on          | Purpose |
|----------------------|------------------|---------|
| `provision.yml`      | localhost        | Create `ezpodman-sandbox` project, launch VM-1 and VM-2, wait for ready |
| `setup.yml`          | vm1, vm2         | Install and configure all software |
| `containers_up.yml`  | vm1, vm2         | Start test containers |
| `containers_down.yml`| vm1, vm2         | Stop and remove test containers |
| `nuke.yml`           | localhost + vms  | Full teardown: containers → VMs → project deleted |

`nuke.yml` must delete the `ezpodman-sandbox` Incus project as its final step.
Add a `--tags force` path that skips graceful container shutdown.

### Repository Structure
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
These are done once by the operator before running any playbook:

1. **SSH key VM-1 → VM-2** — required for ezpodman remote podman over SSH
2. **Forgejo** — already deployed manually at `forgejo.home.lan`
3. **Step-CA root trust** — must be added to VM-1 and VM-2 after provisioning:
   ```bash
   curl -k https://keymaster.home.lan:6443/roots.pem \
     -o /usr/local/share/ca-certificates/keymaster-root.crt
   update-ca-certificates
   ```

---

## Constraints and Preferences
- Ansible collections required: `community.general`
- No Docker Compose; use `containers.podman` module or `podman run`
- Playbooks should be idempotent where possible
- Add comments explaining non-obvious Ansible patterns (operator is learning)

## Known Risk
The `community.general.incus` connection plugin support for named remotes
(`ansible_incus_remote`) targeting non-local Incus servers needs validation.
If unsupported, fallback is Badger as SSH jump host for `provision.yml`/`nuke.yml`.
Flag this early and suggest a test command.
