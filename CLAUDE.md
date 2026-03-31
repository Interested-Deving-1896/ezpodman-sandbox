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

### Installing ezpodman (used by setup.yml)

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

| Name            | Distro      | Role          | Software |
|-----------------|-------------|---------------|----------|
| ezpodman-local  | Fedora 43   | local / main  | curl, podman, podman-remote, docker, ezpodman, lazydocker, git, go, jq, fzf |
| podman-remote   | Debian Trixie | remote/target | podman, openssh-server |

- VMs live on the internal Incus bridge — no direct LAN IPs
- Ansible connects via `community.general.incus` connection plugin (no SSH needed)
- Podman is **rootless** on both VMs, running as `sandbox_user` (default: `podman`)
- Both VMs have `podman.socket` enabled — `ezpodman-local` needs it for local container management; `podman-remote` needs it for remote access over SSH
- Remote podman connectivity `ezpodman-local` → `podman-remote` over SSH is the core feature being tested

### Test Containers

| VM              | Containers      | Ports         |
|-----------------|-----------------|---------------|
| ezpodman-local  | nginx, caddy    | 8080, 8081    |
| podman-remote   | nginx, postgres | 8080, 5432    |

Rootless podman only. Managed via `containers.podman` Ansible collection. No Docker Compose.

---

## Ansible Connection

```ini
ansible_connection=community.general.incus
ansible_incus_remote={{ incus_remote }}
ansible_incus_project=ezpodman-sandbox
```

`incus_remote` defaults to `badger` in `group_vars/all.yml`.

`provision.yml` and `nuke.yml` run against `localhost` and use the `incus` CLI
directly (named remotes are already configured on the Mac). All other playbooks
use the connection plugin to exec into VMs.

---

## Playbooks

| Playbook              | Runs on              | Purpose |
|-----------------------|----------------------|---------|
| `provision.yml`       | localhost            | Create project, launch VMs, wait for ready |
| `setup.yml`           | ezpodman-local, podman-remote | Install and configure all software |
| `containers_up.yml`   | ezpodman-local, podman-remote | Start test containers |
| `containers_down.yml` | ezpodman-local, podman-remote | Stop and remove test containers |
| `nuke.yml`            | localhost + VMs      | Full teardown: containers → VMs → images → project |

`nuke.yml` accepts `--tags force` to skip graceful container shutdown.

---

## Repository Structure

```
ezpodman-sandbox/
├── ansible.cfg              # sets inventory and roles_path
├── CLAUDE.md
├── README.md
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       ├── all.yml          # incus_remote, incus_project, connection vars
│       ├── local.yml        # ezpodman-local vars + container definitions
│       └── remote.yml       # podman-remote vars + container definitions
├── playbooks/
│   ├── provision.yml
│   ├── setup.yml
│   ├── containers_up.yml
│   ├── containers_down.yml
│   └── nuke.yml
└── roles/
    ├── incus_project/       # create/delete project, launch VMs
    ├── podman_setup/        # rootless podman, systemd user socket
    └── test_containers/     # start/stop containers via containers.podman
```

---

## Manual Prerequisites (out of scope for Ansible)

Done once after provisioning:

1. **SSH key `ezpodman-local` → `podman-remote`** — required for ezpodman remote podman over SSH.
   `setup.yml` installs sshd and sets a password for `sandbox_user` on `podman-remote`
   (default: `podman`/`podman`) so ezpodman's `ssh-copy-id` step can authenticate.
2. **Step-CA root trust** — run on each VM so HTTPS to homelab services works:

   ```bash
   curl -k https://keymaster.home.lan:6443/roots.pem \
     -o /usr/local/share/ca-certificates/keymaster-root.crt
   update-ca-certificates
   ```

---

## Constraints and Preferences

- Ansible collections required: `community.general`, `containers.podman`
- No Docker Compose; use `containers.podman` module
- Playbooks should be idempotent where possible
- Add comments explaining non-obvious Ansible patterns (operator is learning)
- Always run `ansible-playbook` from the project root (`ansible.cfg` is there)

---

## Gotchas

**New Incus projects have an empty default profile** (no root disk, no network).
`incus launch` must always pass `--storage {{ storage_pool }}` and `--network {{ network_bridge }}`.
Defaults: `storage_pool: default`, `network_bridge: incusbr0` (in `roles/incus_project/defaults/main.yml`).

**Fedora 43 minimal image ships without Python.** `setup.yml` and `nuke.yml` bootstrap
it via `raw` module before `gather_facts`, using a distro-agnostic shell conditional.

**lazydocker release filenames embed the version** (`lazydocker_0.25.0_Linux_x86_64.tar.gz`).
`setup.yml` queries the GitHub API for the latest tag before constructing the download URL.
`ansible_facts['architecture']` returns `aarch64` on ARM — not yet tested on M2-hosted VMs
(lazydocker filenames use `arm64`, which would need a mapping if the arch doesn't match).

**nuke.yml teardown order** — cached images inside the project must be deleted before
the project itself, otherwise project deletion fails. Order: containers → VMs → images → project.

**Read-only `command` tasks** must have `check_mode: false` so `--check` runs can
still query remote state. Write tasks stay check-mode-skippable.

**`community.general.incus` connection plugin** fully supports `ansible_incus_remote`
for named remotes. Constructs: `incus exec <remote>:<instance> --project <project>`.
Validate before first run: `incus project list badger:`

**sandbox_user** (rootless Podman user) defaults to `podman`, defined in
`roles/podman_setup/defaults/main.yml`. All containers, sockets, and binaries belong to this user.
Check containers as this user: `incus exec badger:ezpodman-local --user <uid> -- podman ps`

**ezpodman is fetched from GitHub**, not Forgejo, to avoid the Step-CA TLS prerequisite
on fresh VMs.

**`su - podman` inside `incus exec` does not trigger PAM**, so `XDG_RUNTIME_DIR`,
`DBUS_SESSION_BUS_ADDRESS`, and `DOCKER_HOST` are never set by the system. `setup.yml`
writes them explicitly to `sandbox_user`'s `~/.bashrc`. Always use `su -` (not `su`)
so the login shell sources `.bashrc`.

**Ghostty terminal users** — `TERM=xterm-ghostty` is propagated from the Mac into the VM
via `incus exec`, but Fedora doesn't have Ghostty's terminfo. TUI apps (lazydocker, ezpodman)
fail with a cryptic `exec.ExitError exit status 1`. Fix: add `export TERM="xterm-256color"`
to the `podman` user's `~/.bashrc` on `ezpodman-local` (manual step — not automated by Ansible).
