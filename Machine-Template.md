This folderThis # Machine Name

## Info
| Property | Value |
|----------|-------|
| Platform | HackTheBox |
| Difficulty | Easy/Medium |
| OS | Linux/Windows |
| IP | 10.10.10.X |
| Date | {{date}} |
| Status | ⬜ In Progress / ✅ Completed |

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -oA nmap/machine_name 10.10.10.X
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH X.X |
| 80 | HTTP | Apache X.X |

### Web Enumeration
```bash
ffuf -u http://10.10.10.X/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

**Findings:**
-

---

## Enumeration

### Service 1
Notes...

### Service 2
Notes...

---

## Exploitation

### Vulnerability Found
- CVE:
- Type:

### Exploit Steps
1. Step one
2. Step two

```bash
# Commands used
```

### Initial Foothold
```bash
# Shell obtained
```

**User Flag:** `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

---

## Privilege Escalation

### Enumeration
```bash
# LinPEAS / WinPEAS output
```

### PrivEsc Vector
- Type: SUID / Sudo / Service / etc.

### Steps
1. Step one
2. Step two

**Root Flag:** `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

---

## Lessons Learned
- What worked:
- What didn't work:
- New techniques learned:

---

## Commands Reference
```bash
# Useful commands from this box
```

---

**Tags:** #htb #machine #easy #linux
