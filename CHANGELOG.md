# Changelog

## 0.5.0 (unreleased)

Breaking: `agent_sandbox` no longer carries the owner's personal setup. It is split into a minimal base and one role
per agent, and everything personal (rules, memories, skills, configs, extra tools) is now set through variables.

- `agent_sandbox` (breaking) sets up only what all agents share: the agent user without sudo, a private home, rootless
  podman (`agent_sandbox_enable_podman`, new), the git SSH key, and the shared context in `~/.agents/`:
  `AGENTS.md` (`agent_sandbox_global_rules` or `agent_sandbox_global_rules_template`), `skills/`
  (`agent_sandbox_skills`, from a directory on the control machine or a git repository, mirrored) and `commands/`
  (`agent_sandbox_commands`). `agent_sandbox_files` puts any other file in the agent home through the `files` role
  (e.g. memories seeded with `force: false`).
  - The login shell is `agent_sandbox_shell`, default `/bin/bash` (was fish). `~/.local/bin` is on the PATH of bash
    (the stock `~/.profile`) and fish (a conf.d snippet).
  - It installs only git, curl, ca-certificates, openssh-client, acl and rsync. The other tools (fish, tmux, vim, jq,
    ripgrep, gh, glab, age, pipx, ...) go in `base_extra_packages`.
  - Removed: the built-in global rules, seed memories, the `/remember` command, the `ansible-roles` skill (now
    `docs/ansible-roles/SKILL.md`), the Claude Code and opencode configs, `agent_sandbox_host_kind`,
    `agent_sandbox_server_description`, `agent_sandbox_rebuild_description`, and sops (dropped; install it yourself if
    you need it).
  - It no longer resets the agent's groups. It fails when the agent may run sudo.
- New role `claude_sandbox`: installs Claude Code (`claude_sandbox_version`, also `latest`), writes
  `~/.claude/settings.json` from `claude_sandbox_settings` (no default content), links `~/.claude/CLAUDE.md`,
  `~/.claude/skills` and `~/.claude/commands` to `~/.agents`, and installs `claude_sandbox_marketplaces` and
  `claude_sandbox_plugins` with `claude plugin`.
- New role `opencode_sandbox`: installs opencode (`opencode_sandbox_version`), writes
  `~/.config/opencode/opencode.json` from `opencode_sandbox_config` (no default content) and links
  `~/.config/opencode/AGENTS.md` and `~/.config/opencode/commands` to `~/.agents` (opencode reads `~/.agents/skills`
  itself). Its verification asks opencode which skills and commands it loads.
- New role `vagrant_session`: the former `agent_sandbox_enable_vagrant`, for any `vagrant_session_users`.
- Playbooks (breaking): `agent_sandbox` is gone. `claude_sandbox` and `opencode_sandbox` run agent_sandbox and one
  agent on a prepared host; `claude_sandbox_from_scratch` and `opencode_sandbox_from_scratch` add base, harden and
  files for a fresh host. `verify` covers the new roles.
- `files`: an entry can be a `template` (rendered on the control machine; vault-encrypted templates work, sops ones are
  rejected) or inline `content` instead of `src`; `force: false` only writes a missing file; `create_dirs` (with
  `dir_mode`) creates missing directories.
- `base`: `base_extra_packages` are installed together with `base_packages`.
- Molecule: scenarios for `claude_sandbox`, `opencode_sandbox` and `vagrant_session` (Debian only); the agent scenarios
  run `agent_sandbox` in prepare.

### Migrating from 0.4

| 0.4 | 0.5 |
|-----|-----|
| playbook `afmigoo.provision.agent_sandbox` | `claude_sandbox_from_scratch` (or your own playbook with both agents) |
| `agent_sandbox_enable_claude`, `agent_sandbox_claude_version` | `claude_sandbox_enable`, `claude_sandbox_version` |
| `agent_sandbox_enable_opencode`, `agent_sandbox_opencode_version` | `opencode_sandbox_enable`, `opencode_sandbox_version` |
| `agent_sandbox_enable_vagrant` | `vagrant_session_enable` with `vagrant_session_users: [agent]` |
| `agent_sandbox_enable_configs` | `agent_sandbox_enable_context`, plus `claude_sandbox_settings` and `opencode_sandbox_config` |
| `agent_sandbox_packages` | `base_extra_packages` (the role installs its own few) |
| `agent_sandbox_host_kind`, `agent_sandbox_server_description`, `agent_sandbox_rebuild_description` | your own `agent_sandbox_global_rules_template` |
| built-in global rules, memories, `/remember`, `ansible-roles` skill | `agent_sandbox_global_rules(_template)`, `agent_sandbox_files`, `agent_sandbox_commands`, `agent_sandbox_skills` |
| `agent_sandbox_enable_sops`, `agent_sandbox_sops_version` | removed |
| fish login shell | `agent_sandbox_shell: /usr/bin/fish` and `fish` in `base_extra_packages` |

On a host provisioned with 0.4, `~/.claude/CLAUDE.md`, `~/.claude/skills`, `~/.claude/commands`,
`~/.config/opencode/AGENTS.md` and `~/.config/opencode/commands` are real files and directories, and the agent roles
stop rather than replace them with links. Move anything the agents added there into `~/.agents` (or your variables),
delete them and run again. The agent's login shell changes to `agent_sandbox_shell` on the next run.

## 0.4.1

- `agent_sandbox`: Claude Code no longer adds the claude.ai session link (the `Claude-Session` trailer and the PR-body
  link) to commits and pull requests (`attribution.sessionUrl: false`), nor the `Co-Authored-By: Claude` trailer to
  commits (`attribution.commit: ""`).

## 0.4.0

- `agent_sandbox`: the Ansible role conventions are installed as the `ansible-roles` skill in
  `~/.claude/skills/` (Claude Code and opencode both read it) and overwritten on every run, instead of being seeded
  once as the `ansible/role-conventions.md` memory. Existing VMs keep the old memory file until it is deleted.

## 0.3.1

- `files`: with sops-encrypted sources it checks that the `sops` binary is on the control machine first and fails
  with "install the sops binary" instead of the lookup's "No such file or directory: 'sops'".

## 0.3.0

- `base` no longer removes unattended-upgrades; `base_remove_unattended_upgrades` is gone. Hosts where it was already
  removed don't get it back.
- `harden` (breaking): `harden_ufw_allowed_ports` entries are mappings with `port`, `proto` (`tcp`, `udp` or `any`;
  default `tcp`) and `comment`, so a rule can be UDP or both protocols. Plain port numbers are no longer accepted:
  write `[80, 443]` as `[{port: 80}, {port: 443}]`. Rules without a comment get `harden_ufw_comment`
  ("Managed by Ansible (afmigoo.provision.harden)"), so existing rules are updated once. The verification checks that
  every listed port and protocol has a rule.
- Molecule: one scenario per role (`base`, `harden`, `podman`, `agent_sandbox`, `files`) instead of per playbook.
  Each runs its role with `import_role` and the role's test vars in `roles/<role>/vars/test.yml` (`vars_from: test`),
  then its `verification` entry point; `prepare.yml` sets up what a role expects (users for `podman`, the apt cache
  for `harden`).

## 0.2.0

- `agent_sandbox`: the agent's git SSH key is generated on the host when missing instead of being copied from the
  control machine, and its public key is printed at the end. `agent_sandbox_ssh_key_dir` is gone and runs no longer
  need a vault password. The verification checks the key.
- New role `files`: copies `files_list` from the control machine to the host. Sources may be plain, ansible-vault or
  sops-encrypted (detected by content, decrypted with `community.sops`, now a dependency). It checks sources, owners,
  groups and destination directories first, creates new files with their owner and mode before writing content, and
  never shows diffs. Every playbook runs it last and `verify` checks it; a new `files` Molecule scenario covers
  all three formats and checks the fixture contents.
- `agent_sandbox`: installs `make`, `gh`, `glab`, `age`, `python3-venv` and `pipx`, and sops from the upstream
  release binary (`agent_sandbox_enable_sops`, `agent_sandbox_sops_version`, checked against the release
  checksums). The global rules tell the agents to use pipx and project venvs for Python tools.
- `agent_sandbox_enable_vagrant` (off by default, Debian only): Vagrant with vagrant-libvirt in the agent's own libvirt
  session. The agent joins `kvm`, `qemu-bridge-helper` is setuid root (a dpkg stat override) and limited to `virbr0`,
  and `~/.vagrant.d/Vagrantfile` sets up session mode without synced folders. The verification checks the setup.

## 0.1.0

- Initial release: roles `base`, `harden`, `podman`, `agent_sandbox` and playbooks `basic_hardening`, `agent_sandbox`,
  `podman_lab`, `verify`.
- Molecule scenarios (vagrant-libvirt) for every playbook: converge, idempotence, verify.
- Makefile with development shortcuts.
- `harden` applies `devsec.hardening.os_hardening` (`harden_enable_os`); requires devsec.hardening 11.
- `podman` verification can run a throwaway container (`podman_verify_image`).
- `agent_sandbox`: the agent user has no sudo.
- `harden`: `harden_os_allow_userns` and `harden_os_sysctl_overwrite`; rootless podman needs the former.
- Ubuntu 24.04 and 26.04 support, tested with Molecule alongside Debian 12 and 13.
- `harden`: works around devsec ssh_hardening leaving sshd stopped on Ubuntu (dev-sec/ansible-collection-hardening#941),
  enables `ufw.service` at boot, removes apport on Ubuntu, and keeps `/var/log` and `/run/faillock` stable across reboots.
- GitHub Actions CI: ansible-lint, a build and install check, gitleaks, actionlint and zizmor. Collections are pinned in
  `requirements.yml`; Dependabot updates the pip packages and actions.
