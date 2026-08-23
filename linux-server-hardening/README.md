# Linux Server Hardening Lab

## Objective
Harden a Linux server (Ubuntu Server, VirtualBox) by reducing its attack
surface: patching, restricting SSH access, enabling a firewall, disabling
unnecessary services, and adding brute-force protection. Builds on the
baseline established in the Nmap Network Discovery project.

## Environment
- Hypervisor: VirtualBox
- Target: Ubuntu Server (192.168.56.101)
- Admin machine: Kali Linux
- Network: Host-only adapter (isolated lab network)

## Baseline Snapshot
Saved a before-hardening snapshot prior to any changes, to allow rollback
if a change locked out access.

![Snapshot before hardening](screenshots/01-snapshot-before.png)

## 1. System Updates
Ran apt update && apt upgrade to patch known vulnerabilities. Unpatched
software is one of the most common initial access vectors for attackers.

![apt update](screenshots/03-apt-update.png)
![apt upgrade](screenshots/04-apt-upgrade.png)

## 2. User Account Hardening
- Reviewed all accounts with shell access.
- Locked the root account (passwd -l root) to prevent direct root login.
- Verified only one account holds UID 0 — confirms no unauthorized
  privilege escalation via duplicate UID 0 accounts.

![Shell accounts](screenshots/05-shell-accounts.png)
![Root locked](screenshots/06-root-locked.png)
![UID 0 check](screenshots/07-uid0-check.png)

## 3. SSH Hardening
- Generated an SSH keypair on the admin machine and configured key-based
  authentication on the server.
- Disabled password authentication to eliminate brute-force login as a
  viable attack path.
- Disabled root login over SSH (PermitRootLogin no).
- Reduced MaxAuthTries to 3 to slow automated login attempts.
- Disabled X11 forwarding (not needed on a headless server).

![Keygen](screenshots/08-keygen.png)
![ssh-copy-id](screenshots/09-ssh-copy-id.png)
![Key login test](screenshots/10-key-login-test.png)
![sshd_config edited](screenshots/11-sshd-config-edited.png)
![SSH hardened test](screenshots/12-ssh-hardened-test.png)
![Root login blocked](screenshots/13-root-login-blocked.png)

## 4. Firewall Configuration (UFW)
Set default policy to deny all incoming traffic, allow all outgoing.
Explicitly allowed only required ports (22, 80) — least-privilege network
access, only reachable services are reachable.

![UFW before](screenshots/14-ufw-before.png)
![UFW after](screenshots/15-ufw-after.png)

## 5. Service Minimization
Reviewed all enabled services (`systemctl list-unit-files --type=service
--state=enabled`) and disabled anything unnecessary for a headless lab
server:

- avahi-daemon — network auto-discovery (mDNS), unneeded attack surface
- bluetooth — irrelevant on a headless server
- cups — printing support, server doesn't print
- ModemManager — for cellular/mobile modems, not applicable to a VM
- openvpn — not used in this lab
- sssd — domain/LDAP authentication, not relevant here

Kept cups-browsed, apache2, and vsftpd running since the Nmap Network
Discovery project depends on apache2/vsftpd, along with ssh, ufw,
cron, rsyslog, and core networking services.

![Services enabled](screenshots/16-services-enabled.png)

## 6. Brute-Force Protection (fail2ban)
Installed and enabled fail2ban with the default sshd jail — automatically
bans IPs after repeated failed SSH login attempts, mitigating brute-force
attacks even on the allowed port.

![fail2ban status](screenshots/17-fail2ban-status.png)

## Verification
Re-ran Nmap post-hardening and compared to the Project 1 baseline.

*Before:* 21/tcp open, 22/tcp open (password auth allowed, root login allowed), 80/tcp open
*After:* 21/tcp open, 22/tcp open (key-only, no root), 80/tcp open, fail2ban active on port 22

![Nmap after hardening](screenshots/18-nmap-after-hardening.png)

Confirmed:
- Root SSH login refused
- Password-based SSH login refused
- Key-based SSH login succeeds
- UFW blocking all ports except 22/80
- fail2ban actively monitoring SSH

## Key Takeaways
Hardening doesn't always reduce the number of open ports — it reduces
what's exploitable on those ports. Access control (key-only auth, no root
login) and monitoring (fail2ban) matter as much as closing ports outright,
which maps directly to defense-in-depth concepts used in SOC environments.