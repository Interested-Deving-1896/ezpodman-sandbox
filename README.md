# ezpodman-sandbox

Ansible automation to provision, configure, and tear down a sandbox environment
for testing [ezpodman](https://github.com/alfonsosanchez12/ezpodman) — a lazydocker wrapper for Podman.

---

## Deploy everything

```bash
ansible-playbook playbooks/provision.yml && \
ansible-playbook playbooks/setup.yml && \
ansible-playbook playbooks/containers_up.yml
```

This runs against the `badger` Incus remote by default. When complete you have:

- **ezpodman-local** (Fedora 43) — podman, lazydocker, ezpodman, full toolchain
- **podman-remote** (Debian Trixie) — podman + `podman.socket` enabled
- Containers running: nginx + caddy on ezpodman-local, nginx + postgres on podman-remote

---

## Prerequisites

Install the required Ansible collections once:

```bash
ansible-galaxy collection install community.general containers.podman
```

Verify the target remote is reachable:

```bash
incus project list badger:
```

---

## Options

### Target a different remote

Any playbook accepts `-e "incus_remote=<name>"` at runtime:

```bash
ansible-playbook playbooks/provision.yml -e "incus_remote=falcon"
```

Available remotes: `badger` (default), `endurance`, `falcon`.

### Change storage pool or network bridge

```bash
ansible-playbook playbooks/provision.yml \
  -e "storage_pool=local" \
  -e "network_bridge=br0"
```

### Dry run (check mode)

```bash
ansible-playbook playbooks/provision.yml --check
```

Read-only tasks (list, query) still execute so the output is meaningful.
Write tasks (create, launch) are simulated.

---

## Playbooks

| Playbook | Purpose |
|----------|---------|
| `provision.yml` | Create the Incus project, launch VMs, wait for boot |
| `setup.yml` | Install and configure all software on the VMs |
| `containers_up.yml` | Start test containers |
| `containers_down.yml` | Stop and remove test containers |
| `nuke.yml` | Full teardown — everything deleted |

Each playbook is idempotent. Re-running it skips what already exists.

---

## Teardown

### Graceful (stops containers first)

```bash
ansible-playbook playbooks/nuke.yml
```

### Force (skip container shutdown)

```bash
ansible-playbook playbooks/nuke.yml --tags force
```

### Containers only (keep VMs)

```bash
ansible-playbook playbooks/containers_down.yml
```

---

## After provisioning (manual steps)

Two steps are out of scope for Ansible and must be done once after `provision.yml`:

1. **SSH key from ezpodman-local to podman-remote** — needed for ezpodman remote Podman over SSH:

   ```bash
   # From inside ezpodman-local as the podman user:
   ssh-keygen -t ed25519
   ssh-copy-id podman@podman-remote
   ```

2. **Step-CA root trust** — needed for HTTPS to homelab services (forgejo, etc.):

   ```bash
   # Run on each VM:
   curl -k https://keymaster.home.lan:6443/roots.pem \
     -o /usr/local/share/ca-certificates/keymaster-root.crt
   update-ca-certificates
   ```

---

## Connecting to VMs

```bash
# Root shell
incus shell --project ezpodman-sandbox badger:ezpodman-local

# Shell as the podman user (rootless containers live here)
incus shell --project ezpodman-sandbox badger:ezpodman-local
su - podman

# Check running containers
incus exec --project ezpodman-sandbox badger:ezpodman-local \
  --user $(incus exec --project ezpodman-sandbox badger:ezpodman-local -- id -u podman) \
  -- podman ps
```
