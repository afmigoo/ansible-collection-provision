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
| `afmigoo.provision.files` | Copies files from the control machine to the host (`files_list`); sources may be plain, ansible-vault or sops-encrypted |

Every role is opt-in: it does nothing unless `<role>_enable: true`, and has
`<role>_enable_<subsystem>` toggles for its parts. Options are documented in
each role's `meta/argument_specs.yml` (`ansible-doc -t role afmigoo.provision.<role>`).
Each role also has a `verification` entry point (`tasks_from: verification`).

### Playbooks

| Playbook | Roles |
|----------|-------|
| `afmigoo.provision.basic_hardening` | base, harden, files |
| `afmigoo.provision.agent_sandbox` | base (with upgrade), harden, agent_sandbox, files |
| `afmigoo.provision.podman_lab` | base (with upgrade), podman for lab users `bob`, `alice`, `jack` (`podman_lab_users`), files |
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

The agent's git SSH key is generated on the host (`~agent/.ssh/id_ed25519`) when
missing and never leaves it. The run ends by printing its public key; add that
to the repos the agent should reach (e.g. as a deploy key). Re-provisioning keeps
the key; a rebuilt host gets a new one.

```sh
ansible-playbook afmigoo.provision.agent_sandbox -e agent_sandbox_host_kind=server
```

`agent_sandbox_host_kind` (`vm` or `server`) picks the wording of the agents'
global rules (`~/.claude/CLAUDE.md`, `~/.config/opencode/AGENTS.md`).

### Files

Every playbook ends with `afmigoo.provision.files`, which copies `files_list`
from the control machine to the host (nothing when the list is empty). Sources
may be plain, ansible-vault encrypted or sops-encrypted; the role tells them
apart by content.

```yaml
files_list:
  - src: "{{ inventory_dir }}/../files/app.env"   # absolute path on the control machine
    dest: /etc/app/app.env                         # absolute file path (no directory or symlink); its directory must exist
    owner: root                                    # default root
    group: root                                    # default root
    mode: "0600"                                   # default 0600
```

`src` has to be absolute because a collection playbook resolves relative paths
inside the installed collection. Vault-encrypted sources need a vault password
(`--ask-vault-pass`); sops-encrypted ones need the `sops` binary and the key to
decrypt them on the control machine (e.g. `SOPS_AGE_KEY_FILE`).

- A sops file reaches the host as `sops decrypt` prints it. Its format comes
  from the file extension, and yaml, json, dotenv and ini files are parsed and
  written out again: same data, not always the same bytes (quotes, indentation,
  YAML 1.1 values such as `0012`). For byte-exact content encrypt with
  `--input-type binary` and keep an extension sops doesn't parse (e.g. `.conf`).
- Every sops-encrypted source is decrypted. To put the encrypted file itself on
  the host, wrap it in ansible-vault.
- The role checks the sources, owners, groups and destination directories
  before copying, creates new files with their owner and mode before writing
  content, and never shows diffs (`--diff`), so decrypted content stays out of
  the output. If a copy fails (e.g. no sops key or vault password), the files
  that run created are removed again; files that existed before may already
  have their new content.

### Vagrant

```ruby
deb13.vm.provision "ansible" do |ansible|
  ansible.compatibility_mode = "2.0"
  ansible.playbook = "playbook.yml"  # - ansible.builtin.import_playbook: afmigoo.provision.agent_sandbox
  ansible.become = true
  ansible.extra_vars = {
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
VMs (4 vCPU, 6 GiB) at a time. Don't pass `--workers` above 1 on a small host.

`basic_hardening` also copies plain, vault and sops-encrypted files with
`afmigoo.provision.files`, so it needs the `sops` binary on the control machine.
Its fixtures in `extensions/molecule/fixtures/` (an age key and a vault password)
are throwaway test secrets, allowlisted for gitleaks in `.gitleaks.toml`.

Role conventions (variable prefixes, layout, check mode, tags, handlers) follow
`roles/agent_sandbox/files/agent/memories/ansible/role-conventions.md`.
