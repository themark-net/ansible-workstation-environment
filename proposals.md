# High-Level Proposals for Workstation Environment Ansible

**All specific configuration details below are PROPOSALS ONLY. No detailed implementation or config files have been added to roles/playbooks yet. Review, approve, modify, or reject before I proceed with coding the details in the repo.**

## 1. Package Installation Strategy (High Level)
Use Ansible's `package` module or distro-specific (dnf/apt) with lists in vars/ for common packages. Conditionals for Rocky vs Ubuntu family. Groups: workstation_core, dev_tools, container_tools, sysadmin_tools, ceph_tools.

**Proposed packages to include (uniform where possible):**
- Editors: vim (full, not vi-only; ensure alternatives set), nano optional.
- Terminal multiplexers: screen (required), tmux as complement.
- Monitoring/Top: btop or htop, glances.
- File/Process: fd-find, ripgrep, fzf, bat, eza or exa, procs, dust, ncdu, duf.
- Network/Sys: tcpdump, wireshark (CLI tshark), nethogs, iotop, strace, lsof, net-tools or iproute.
- Dev: git, build-essential / @development-tools group, python3-pip, python3-venv, nodejs (or n), go, make, cmake, gcc, clang.
- Containers: podman, podman-plugins, fuse-overlayfs for rootless, slirp4netns. Optional: podman-compose.
- K8s: kubectl, helm, k9s, stern, kubectx/kubens or fzf integration.
- Ceph: ceph-common, cephadm (if managing), rbd, ceph-fuse.
- Other: ansible (latest or pinned), terraform or opentofu, packer, jq, yq, curl, wget, unzip, zip, rsync, openssh-server/client, ufw or firewalld, chrony, etc.
- From tool research (Tom Doerr X and general): Since @tomdoerr posts are historical (pre-2016) and not focused on modern CLI/sysadmin stacks, prioritizing current standards: atuin (superior shell history), starship (prompt), zellij (tmux alt), lazygit. Will propose specific inclusion or not.

**High-level only**: Exact package names per distro (e.g. vim vs vim-enhanced on RHEL, fd vs fd-find on Debian), versions, and any PPAs/repos added will be detailed post-approval.

## 2. Unlimited Bash History (High Level)
Create /etc/profile.d/99-unlimited-history.sh or edit /etc/bash.bashrc (Debian) /etc/profile (RHEL). Set:
- HISTSIZE=-1
- HISTFILESIZE=-1
- HISTTIMEFORMAT="%F %T "
- shopt -s histappend
- PROMPT_COMMAND for sync.
For all users or /etc/skel + existing. Consider atuin as modern alternative for sync/search across machines (with approval).

Specific file content, permissions, and handling for non-interactive shells proposed later.

## 3. Vim Configuration (High Level)
Install full vim. Basic /etc/vim/vimrc or per-user in /etc/skel/.vimrc with common settings: syntax on, line numbers, mouse, tabs=2/4, search highlight, etc. No heavy plugins initially (keep lightweight for uniformity). Set EDITOR=vim, VISUAL=vim. Alternatives system updated.

Detailed vimrc content proposed for approval.

## 4. Podman & Kubernetes Setup (High Level)
Install podman, enable lingering for rootless if desktop, sysctl for user namespaces if needed (but modern kernels ok). Configure storage (overlay). For K8s: install kubectl binary from official, helm via script or package, add k9s binary. Optional kind for local clusters. No full k8s cluster on workstation by default.

Any podman registries, policy, or quadlet/systemd integration proposed separately.

## 5. Ceph & Other Sysadmin (High Level)
Install client packages. If Ceph cluster access, keyring handling or ceph.conf template proposed.
Other tools as listed. Firewall uniform: enable with SSH allowed, or profile-based.

## 6. Desktop Uniformity (High Level)
Focus on CLI and common paths (/usr/local/bin, /opt). For GUI: common fonts (nerd-fonts?), but optional since "mostly uniform" and many headless or varied DEs (GNOME, KDE, Cinnamon on Mint). Avoid DE-specific unless approved. XDG dirs, mime types basic.

## SPIFFE/SPIRE Architecture Outline & Proposal (Detailed High Level)

**Overall Goal**: Unified authentication and identity across ALL your systems (workstations + servers + clusters) using SPIFFE/SPIRE for workload identity, linked/integrated with Yubikeys for human/user auth and hardware root of trust where possible.

**Architecture**:

1. **SPIRE Server** (Central, one per trust domain or federated):
   - Deploy on hardened Linux (Rocky or Ubuntu) or in k8s.
   - Trust domain: `spiffe://workstations.yourdomain.com` (or spiffe.io style).
   - Database: SQLite for small, or Postgres/MySQL for prod.
   - Node attestor: Initially `join_token` or `k8s_psat` if in cluster; later TPM 2.0 plugin for hardware workstations/servers (requires TPM present and enabled in BIOS/firmware).
   - Server config: server.conf with TTLs for SVIDs (e.g. 1h for X.509), CA key rotation policy.
   - Expose via SPIRE API or bundled with Envoy/ingress for federation.

2. **SPIRE Agents on Workstations**:
   - Ansible role installs spire-agent binary (from official releases or package if available), systemd service.
   - agent.conf: trust_domain, server_address (mTLS or via bootstrap), data_dir, log.
   - Node attestation: join token initially (generated per host or via your inventory). For production: TPM plugin + EK/AIK verification if hardware supports. Yubikey: Not direct node attestor in upstream; instead, use Yubikey for:
     - Initial bootstrap/attestation token signing if custom.
     - Primary: User authentication layer (see below).
   - Workload attestation: Unix domain socket or systemd for process identity. SPIRE issues SVIDs to processes matching selectors (unix uid, k8s pod if containerized, etc.).

3. **Yubikey Linkage**:
   - **User Auth**: Deploy yubico-pam or pamu2f (FIDO2) + yubikey-manager. Configure PAM stacks for login (tty, gdm, sudo) to require or allow Yubikey touch/OTP/PIV cert. This unifies *human* login across systems.
   - **Workload/ Node**: Yubikey PIV slot for storing intermediate certs or as additional factor. Or use Yubikey to protect SPIRE agent keys if possible (advanced).
   - **Unified Scheme**: Login with Yubikey -> get local session -> workloads/containers get SPIFFE SVIDs automatically via spiffe-helper or SDK. All authN/Z decisions (to other systems, APIs, k8s, Ceph?) can verify SPIFFE ID + Yubikey-backed user context if needed. No passwords shared. Short-lived certs. Audit via SPIRE logs.
   - Example flow: Workstation boots, agent attests (TPM or token), user logs in with Yubikey, starts podman container -> container gets SPIFFE ID `spiffe://.../workstation/hostname/container/nginx` attested to user session.

4. **Integration with Existing**:
   - Podman/K8s: SPIRE can attest k8s workloads natively; for podman, use spiffe-helper sidecar or volume for socket.
   - Ceph: Possible to use SPIFFE for Ceph auth if extended, or separate.
   - All systems: Same trust domain, agents everywhere. Federation for external.

5. **Security Considerations** (no moralizing, just facts):
   - Single point of compromise (SPIRE server) is high value; harden it, use mTLS everywhere, short SVID TTLs, node attestation strong.
   - Yubikey loss: revocation via SPIRE or PAM.
   - Initial bootstrap security critical.

**Implementation Proposal**:
- Phase 1 (core): Ansible role `spire-agent` : install binary, basic config with join_token, systemd, test `spire-agent api fetch x509`.
- Phase 2: TPM attestor if hardware confirms.
- Phase 3: Separate role for Yubikey PAM integration (test on target DEs).
- Phase 4: Workload examples, spiffe-helper, docs for using SVIDs in scripts/apps.
- Server deployment playbook separate (or in same repo under server/).

This is fully implementable in Ansible. I am capable. But per your instruction, **no code for SPIRE configs, Yubikey PAM, or detailed agent.conf written yet**. High-level architecture documented here for review. Approve structure, trust domain choice, attestation method priority (token vs TPM vs other), and Yubikey scope (user only or more) before proceeding.

## Approval Process
Reply with approvals like:
- "Approve packages list, proceed with role/common/packages.yml"
- "Modify bash history to use atuin instead, approve unlimited + atuin"
- "Approve SPIFFE architecture, implement agent role with join token first"
- Or specific changes.

Once approved, I will use tools to add detailed Ansible tasks, vars, templates, and update playbooks in the repo. Inventory examples, usage docs, and testing notes (molecule or manual on Rocky/Ubuntu) will follow.

**Current Repo State**: Repo created. Initial README and this proposals.md added with high-level only. No roles or detailed configs committed. Skeleton playbooks/roles can be added post some approvals if you want fast start on core packages.

Double-checked: Distro support via Ansible facts (ansible_os_family == 'RedHat' or 'Debian'). Package lists verified common names. SPIRE/Yubikey architecture based on official SPIRE docs (node attestor plugins include join_token, tpm, k8s; no native yubikey but PAM separate is standard). Tom Doerr search returned only old non-relevant posts; modern tool selection from established CLI/tooling community consensus (ripgrep etc. ubiquitous).