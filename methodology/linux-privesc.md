# Linux Privilege Escalation — Checklist

A repeatable enumeration order I follow after landing a low-privileged shell on Linux.
Work top-down; the cheap wins are near the top.

## 0. Stabilise the shell
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z, then: stty raw -echo; fg
```

## 1. Context — who and where am I
- [ ] `id`, `whoami`, `sudo -l`  ← **check `sudo -l` first, always**
- [ ] `hostname`, `uname -a`, `cat /etc/os-release`  (kernel + distro → known exploits)
- [ ] `cat /etc/passwd`, other users, home dirs you can read

## 2. Low-hanging fruit
- [ ] `sudo -l` — any `NOPASSWD` entry? Check it on [GTFOBins](https://gtfobins.github.io/)
- [ ] **SUID/SGID binaries**: `find / -perm -4000 -type f 2>/dev/null` → GTFOBins
- [ ] **Capabilities**: `getcap -r / 2>/dev/null` (e.g. `cap_setuid` on python/perl)
- [ ] World-writable files/dirs owned by root: `find / -writable -type f 2>/dev/null`

## 3. Credentials & secrets
- [ ] History files: `~/.bash_history`, `~/.zsh_history`, `.mysql_history`
- [ ] Config files with creds: `.env`, `wp-config.php`, `settings.py`, `*.conf`
- [ ] SSH keys: `~/.ssh/id_*`, `authorized_keys`
- [ ] `grep -RiE 'password|secret|api[_-]?key' /var/www /opt /home 2>/dev/null`

## 4. Scheduled tasks & services
- [ ] `cat /etc/crontab`, `ls -la /etc/cron.*`, `/var/spool/cron`
- [ ] Any cron job running a **writable script** or using a **relative PATH**?
- [ ] Root processes / listening internal services: `ps aux`, `ss -tlnp`

## 5. Mounts, containers, kernel
- [ ] `mount`, `cat /etc/fstab`, `lsblk` — NFS `no_root_squash`?
- [ ] Inside a container? `/.dockerenv`, cgroup checks
- [ ] Kernel exploit **only as a last resort** (noisy, can crash the box)

## 6. Automate the enumeration
- [ ] [`linpeas.sh`](https://github.com/peass-ng/PEASS-ng) — run and read the **red/yellow** lines
- [ ] [`pspy`](https://github.com/DominicBreuker/pspy) — watch cron/processes without root
- [ ] `linenum.sh`, `linux-exploit-suggester.sh`

## Defensive takeaways
When reporting privesc, always pair the finding with the fix:
- Remove unneeded SUID bits and dangerous `sudo` NOPASSWD rules.
- Never store plaintext secrets in world-readable config.
- Cron jobs must use absolute paths and non-writable scripts.
- Keep the kernel and packages patched.
