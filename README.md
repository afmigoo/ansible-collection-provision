# afmigoo.provision

Ansible collection to provision, set up and harden Debian (12, 13) and Ubuntu
(24.04, 26.04) boxes: cloud servers and local Vagrant VMs.

## Contents

### Roles

| Role | What it does |
|------|--------------|
| `afmigoo.provision.base` | Waits for the host, sets DNS servers for systemd-resolved, installs base packages, optionally upgrades, removes unattended-upgrades |
| `afmigoo.provision.harden` | IPv6 off (GRUB drop-in, reboots), `devsec.hardening.os_hardening` and `ssh_hardening`, ufw deny-by-default with ssh and extra ports allowed. Tune the devsec roles with their own `os_*` / `ssh_*` variables |
| `afmigoo.provision.podman` | Installs podman, gives rootless users non-overlapping subuid/subgid ranges, enables lingering |
| `afmigoo.provision.agent_sandbox` | `agent` user without sudo, with rootless podman, fish, Claude Code, opencode, their configs, global rules, seed memories and git SSH key |

Every role is opt-in: it does nothing unless `<role>_enable: true`, and has
`<role>_enable_<subsystem>` toggles for its parts. Options are documented in
each role's `meta/argument_specs.yml` (`ansible-doc -t role afmigoo.provision.<role>`).
Each role also has a `verification` entry point (`tasks_from: verification`).

### Playbooks

| Playbook | Roles |
|----------|-------|
| `afmigoo.provision.basic_hardening` | base, harden |
| `afmigoo.provision.agent_sandbox` | base (with upgrade), harden, agent_sandbox |
| `afmigoo.provision.podman_lab` | base (with upgrade), podman for lab users `bob`, `alice`, `jack` (`podman_lab_users`) |
| `afmigoo.provision.verify` | verification of every role enabled for the host |

All playbooks target `all` unless `provision_hosts` is set. They enable their
roles with play vars, so override with `-e` (e.g. `-e harden_enable=false`).

## Install

```sh
ansible-galaxy collection install git+https://github.com/afmigoo/ansible-collection-provision.git
```

or in `requirements.yml`:

```yaml
collections:
  - name: https://github.com/afmigoo/ansible-collection-provision.git
    type: git
    version: main
```

Dependencies (`devsec.hardening`, `community.general`) are installed with it.

## Usage

### Basic hardening

```sh
ansible-playbook afmigoo.provision.basic_hardening -e '{"harden_ufw_allowed_ports": [80, 443]}'
```

### Agent sandbox

The agent's git SSH key lives on the control machine in
`agent_sandbox_ssh_key_dir` (`id_ed25519`, `id_ed25519.pub`). On the first run it
is generated there, unencrypted. Encrypt the private key with either tool; the
role tells them apart by content:

```sh
ansible-vault encrypt keys/agent-sandbox/id_ed25519      # then run with --ask-vault-pass
sops encrypt -i keys/agent-sandbox/id_ed25519            # needs sops and a .sops.yaml or --age
ansible-playbook afmigoo.provision.agent_sandbox --ask-vault-pass \
  -e agent_sandbox_ssh_key_dir=$PWD/keys/agent-sandbox -e agent_sandbox_host_kind=server
```

A sops-encrypted key is decrypted with the `community.sops` lookup, so the
control machine needs the `sops` binary and the key to decrypt it (e.g.
`SOPS_AGE_KEY_FILE`).

`agent_sandbox_host_kind` (`vm` or `server`) picks the wording of the agents'
global rules (`~/.claude/CLAUDE.md`, `~/.config/opencode/AGENTS.md`).

### Vagrant

```ruby
deb13.vm.provision "ansible" do |ansible|
  ansible.compatibility_mode = "2.0"
  ansible.playbook = "playbook.yml"  # - ansible.builtin.import_playbook: afmigoo.provision.agent_sandbox
  ansible.ask_vault_pass = true  # only for a vault-encrypted key
  ansible.become = true
  ansible.extra_vars = {
    agent_sandbox_ssh_key_dir: "#{__dir__}/keys/agent-sandbox",
    harden_enable: false,
  }
end
```

Vagrant's ansible provisioner needs a playbook path, so use a one-line local
playbook that imports the collection playbook.

## Development

```sh
pyenv virtualenv 3.14.6 afmigoo.provision     # once; see .python-version
make install    # pip packages and collections
make lint       # ansible-lint (production profile)
make build      # collection tarball
```

`make` lists every target.

CI (GitHub Actions) runs `make lint`, builds and installs the tarball to syntax-check the playbooks from it, and scans
for secrets (gitleaks) and workflow problems (actionlint, zizmor). The Molecule tests need VMs and stay local.

### Molecule tests

`extensions/molecule/` has one scenario per playbook (`basic_hardening`,
`podman_lab`, `agent_sandbox`). Each boots four VMs with vagrant-libvirt,
Debian 12 and 13 (1 vCPU, 1 GiB) and Ubuntu 24.04 and 26.04 (1 vCPU, 2 GiB), runs
converge, idempotence, then `afmigoo.provision.verify` on all of them at once,
and destroys them.

```sh
make test-basic_hardening   # one scenario (molecule test -s basic_hardening)
make test                   # all of them, one VM at a time (molecule test --all)
make destroy                # clean up after an interrupted run
```

Scenarios run sequentially, so a test never uses more than one scenario's four
VMs (4 vCPU, 6 GiB) at a time. Don't pass `--workers` above 1 on a small host. `agent_sandbox`
skips the git SSH key.

Role conventions (variable prefixes, layout, check mode, tags, handlers) follow
`roles/agent_sandbox/files/agent/memories/ansible/role-conventions.md`.
