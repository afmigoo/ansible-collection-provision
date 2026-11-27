# afmigoo.provision

Ansible collection to provision, set up and harden Debian (12, 13) and Ubuntu
(24.04, 26.04) boxes: cloud servers and local Vagrant VMs.

## Contents

### Roles

| Role | What it does |
|------|--------------|
| `afmigoo.provision.base` | Waits for the host, sets DNS servers for systemd-resolved, installs base packages (plus `base_extra_packages`), optionally upgrades |
| `afmigoo.provision.harden` | IPv6 off (GRUB drop-in, reboots), `devsec.hardening.os_hardening` and `ssh_hardening`, ufw deny-by-default with ssh and extra ports allowed. Tune the devsec roles with their own `os_*` / `ssh_*` variables |
| `afmigoo.provision.podman` | Installs podman, gives rootless users non-overlapping subuid/subgid ranges, enables lingering |
| `afmigoo.provision.agent_sandbox` | `agent` user without sudo, with a private home, rootless podman, a git SSH key and the context shared by coding agents in `~/.agents` (global rules, skills, commands); installs no agent |
| `afmigoo.provision.claude_sandbox` | Claude Code for the agent user: pinned install, `settings.json`, plugins, links to `~/.agents` |
| `afmigoo.provision.opencode_sandbox` | opencode for the agent user: pinned binary, `opencode.json`, links to `~/.agents` |
| `afmigoo.provision.vagrant_session` | Vagrant without root (vagrant-libvirt in each user's libvirt session) for `vagrant_session_users`; Debian only |
| `afmigoo.provision.files` | Puts files on the host (`files_list`): copied (plain, ansible-vault or sops-encrypted), rendered from templates or inline content |

Every role is opt-in: it does nothing unless `<role>_enable: true`, and has
`<role>_enable_<subsystem>` toggles for its parts. Options are documented in
each role's `meta/argument_specs.yml` (`ansible-doc -t role afmigoo.provision.<role>`).
Each role also has a `verification` entry point (`tasks_from: verification`).

### Playbooks

| Playbook | Roles |
|----------|-------|
| `afmigoo.provision.basic_hardening` | base, harden, files |
| `afmigoo.provision.claude_sandbox` | agent_sandbox, claude_sandbox |
| `afmigoo.provision.claude_sandbox_from_scratch` | base (with upgrade), harden, agent_sandbox, claude_sandbox, files |
| `afmigoo.provision.opencode_sandbox` | agent_sandbox, opencode_sandbox |
| `afmigoo.provision.opencode_sandbox_from_scratch` | base (with upgrade), harden, agent_sandbox, opencode_sandbox, files |
| `afmigoo.provision.podman_lab` | base (with upgrade), podman for lab users `bob`, `alice`, `jack` (`podman_lab_users`), files |
| `afmigoo.provision.verify` | verification of every role enabled for the host |

Playbook names follow one convention (shared with sibling collections):

- `<thing>` runs only the roles of that thing and expects a prepared host
  (packages, hardening and the like already done).
- `<thing>_from_scratch` is for a fresh host: base, harden and any prerequisites,
  then the same roles as `<thing>`, then files.

All playbooks target `all` unless `provision_hosts` is set. They enable their
roles with play vars, so override with `-e` (e.g. `-e harden_enable=false`).
To combine roles differently (e.g. both agents, plus `vagrant_session`), write a
short playbook that lists the roles you want.

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
ansible-playbook afmigoo.provision.basic_hardening -e '{"harden_ufw_allowed_ports": [{"port": 80}, {"port": 443}]}'
```

`harden_ufw_allowed_ports` opens extra ports besides ssh:

```yaml
harden_ufw_allowed_ports:
  - port: 443                # proto tcp, comment harden_ufw_comment
  - port: 51820
    proto: udp               # tcp (default), udp or any
    comment: WireGuard       # "" for no comment
```

A rule without its own comment gets `harden_ufw_comment` ("Managed by Ansible (afmigoo.provision.harden)").

Provisioning only adds ufw rules: a port removed from the list stays open until you delete its rule
(`ufw delete allow 51820/udp`).

### Agent sandbox

`claude_sandbox` and `opencode_sandbox` don't depend on `agent_sandbox`, but
expect its user (`agent_sandbox_user`, default `agent`) and `~/.agents`, so run
`agent_sandbox` first (the playbooks do). The agent's login shell is
`agent_sandbox_shell` (default `/bin/bash`), with `~/.local/bin` on the PATH for
bash and fish.

`agent_sandbox` installs only the packages it needs itself (git, ssh-keygen,
acl, rsync). Put more tools for the agent in `base_extra_packages`, which the
`*_from_scratch` playbooks install with base:

```yaml
base_extra_packages: [fish, jq, ripgrep, make, gh, python3-venv, pipx]
```

The shared context:

```yaml
# ~/.agents/AGENTS.md, inline or a template on the control machine (set one of them)
agent_sandbox_global_rules: |
  # Global rules
  Run `git status` before assuming the working tree is yours.
# agent_sandbox_global_rules_template: "{{ inventory_dir }}/../files/AGENTS.md.j2"

# ~/.agents/skills/<name>/, mirrored: files the source doesn't have are removed
agent_sandbox_skills:
  - name: my-skill
    src: "{{ inventory_dir }}/../skills/my-skill"   # directory on the control machine
  - name: brand-guidelines
    git: https://github.com/anthropics/skills.git    # cloned on the host as the agent
    version: e4bb5d960d87126ff3b3a8af1a104b7db67560ba
    path: skills/brand-guidelines                    # subdirectory holding the skill

# ~/.agents/commands/<name>.md
agent_sandbox_commands:
  - name: remember
    content: |
      ---
      description: Save a note to memory
      ---
      Save this to ~/.agent/memories: $ARGUMENTS

# Any other files for the agent, entries as in files_list. A relative dest is under
# the agent home, owner and group default to the agent, missing directories are created
agent_sandbox_files:
  - src: "{{ inventory_dir }}/../files/memories/README.md"
    dest: .agent/memories/README.md
    mode: "0644"
```

Claude Code reads the shared context through links: `~/.claude/CLAUDE.md`,
`~/.claude/skills` and `~/.claude/commands` point into `~/.agents`. opencode
reads `~/.agents/skills` itself and gets links for `~/.config/opencode/AGENTS.md`
and `~/.config/opencode/commands`. A link is only made when its target exists, and
replaces whatever is at its path.

```yaml
claude_sandbox_version: "2.1.280"   # or latest: installed once, never upgraded by the role
claude_sandbox_settings:            # ~/.claude/settings.json; {} leaves it alone
  env:
    DISABLE_AUTOUPDATER: "1"        # keep the pinned version
claude_sandbox_marketplaces:
  - name: claude-plugins-official   # the name the marketplace declares
    source: https://github.com/anthropics/claude-plugins-official.git
claude_sandbox_plugins:
  - commit-commands@claude-plugins-official

opencode_sandbox_version: "1.18.11"
opencode_sandbox_config:            # ~/.config/opencode/opencode.json; {} leaves it alone
  $schema: https://opencode.ai/config.json
  autoupdate: false
```

The role keeps the keys `claude plugin` writes to `settings.json`.

The agent's git SSH key is generated on the host (`~agent/.ssh/id_ed25519`) when
missing and never leaves it. The run ends by printing its public key; add that
to the repos the agent should reach (e.g. as a deploy key). Re-provisioning keeps
the key; a rebuilt host gets a new one.

```sh
ansible-playbook afmigoo.provision.claude_sandbox_from_scratch -e @agent.yml
```

#### Vagrant in the sandbox

`vagrant_session` (Debian only, Ubuntu doesn't package Vagrant) lets
`vagrant_session_users` (e.g. `[agent]`) run Vagrant VMs, e.g. for Molecule,
without root: vagrant-libvirt uses each user's own libvirt session, and the VMs
join libvirt's default network (`virbr0`) through `qemu-bridge-helper`, which
becomes setuid root and may attach to `virbr0` only. The users join the `kvm`
group; synced folders are off since they need root. On a VM host this needs
nested virtualization, and the default network's 192.168.122.0/24 must not be the
host's own subnet. No playbook runs it; add it to your own after `agent_sandbox`.

### Files

The `*_from_scratch` playbooks and `basic_hardening` and `podman_lab` end with
`afmigoo.provision.files`, which puts `files_list` on the host (nothing when the
list is empty). Each entry has exactly one of `src` (a file copied from the
control machine: plain, ansible-vault encrypted or sops-encrypted, told apart by
content), `template` (a Jinja template on the control machine, plain or
ansible-vault encrypted) or `content` (an inline string).

```yaml
files_list:
  - src: "{{ inventory_dir }}/../files/app.env"   # absolute path on the control machine
    dest: /etc/app/app.env                         # absolute file path (no directory or symlink); its directory must exist
    owner: root                                    # default root
    group: root                                    # default root
    mode: "0600"                                   # default 0600
  - template: "{{ inventory_dir }}/../files/app.conf.j2"
    dest: /etc/app/conf.d/app.conf
    create_dirs: true                              # create missing directories with owner, group and dir_mode
    dir_mode: "0755"                               # default 0755
  - content: "inline\n"
    dest: /etc/app/notes.txt
```

`src` and `template` have to be absolute because a collection playbook resolves
relative paths inside the installed collection. Vault-encrypted sources and
templates need a vault password (`--ask-vault-pass`); sops-encrypted sources need
the `sops` binary and the key to decrypt them on the control machine (e.g.
`SOPS_AGE_KEY_FILE`). Templates are rendered with `lstrip_blocks` and can't be
sops-encrypted. `create_dirs` only creates directories that are missing and never
changes existing ones.

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
  ansible.playbook = "playbook.yml"  # - ansible.builtin.import_playbook: afmigoo.provision.claude_sandbox_from_scratch
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

`extensions/molecule/` has one scenario per role (`base`, `harden`, `podman`,
`agent_sandbox`, `claude_sandbox`, `opencode_sandbox`, `vagrant_session`, `files`);
the playbooks only combine roles and are syntax-checked in CI. Each scenario runs its role alone with the role's test vars,
`roles/<role>/vars/test.yml` (`import_role` with `vars_from: test`; the role never
loads that file itself). A scenario's `prepare.yml` sets up what the role expects
from the host, e.g. the users for `podman` or `agent_sandbox` for the agent roles.
Each boots four VMs (`vagrant_session` only the two Debian ones) with vagrant-libvirt,
Debian 12 and 13 (1 vCPU, 1 GiB) and Ubuntu 24.04 and 26.04 (1 vCPU, 2 GiB), runs
prepare, converge, idempotence, then the role's `verification` entry point on all
of them at once, and destroys them.

```sh
make test-harden            # one scenario (molecule test -s harden)
make test                   # all of them, one VM at a time (molecule test --all)
make destroy                # clean up after an interrupted run
```

Scenarios run sequentially, so a test never uses more than one scenario's four
VMs (4 vCPU, 6 GiB) at a time. Don't pass `--workers` above 1 on a small host.

`files` copies plain, vault and sops-encrypted fixtures and checks their
contents, so it needs the `sops` binary on the control machine. Its fixtures in `extensions/molecule/fixtures/` (an age key and a vault password)
are throwaway test secrets, allowlisted for gitleaks in `.gitleaks.toml`.

The agent scenarios clone a skill and a plugin marketplace from GitHub, so they
need network access.

Role conventions (variable prefixes, layout, check mode, tags, handlers) follow
`docs/ansible-roles/SKILL.md`. It is written as a skill, so it can be installed
for the agents with `agent_sandbox_skills` (`git:` this repository,
`path: docs/ansible-roles`).
