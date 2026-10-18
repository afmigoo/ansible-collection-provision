# Changelog

## Unreleased

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
