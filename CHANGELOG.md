# Changelog

## 0.1.0

- Initial release: roles `base`, `harden`, `podman`, `agent_sandbox` and playbooks `basic_hardening`, `agent_sandbox`,
  `podman_lab`, `verify`.
- Molecule scenarios (vagrant-libvirt) for every playbook: converge, idempotence, verify.
- Makefile with development shortcuts.
- `harden` applies `devsec.hardening.os_hardening` (`harden_enable_os`); requires devsec.hardening 11.
- `podman` verification can run a throwaway container (`podman_verify_image`).
- `agent_sandbox`: the agent user no longer gets sudo; reruns remove the old sudoers rule and group membership.
- `harden`: `harden_os_allow_userns` and `harden_os_sysctl_overwrite`; rootless podman needs the former.
- Ubuntu 24.04 and 26.04 support, tested with Molecule alongside Debian 12 and 13.
- `harden`: works around devsec ssh_hardening leaving sshd stopped on Ubuntu (dev-sec/ansible-collection-hardening#941),
  enables `ufw.service` at boot, removes apport on Ubuntu, and keeps `/var/log` and `/run/faillock` stable across reboots.
