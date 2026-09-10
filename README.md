# nxc-sweep
Bash wrapper for NetExec to quickly validate compromised credentials across SMB, WinRM, RDP, MSSQL, and FTP


## Why?
Running nxc per-protocol manually can be tedious sometimes, especially when you're constantly pivoting from different users. This wrapper pre-validates ports with netcat to see if they're present, then sweeps all relevant services in one go. Built for HTB labs and my OSCP prep to speed up early Windows/AD credentialed enumeration.

It's easily customizable too, you can add WMI, VNC, or even change up the options to the pre-existing protcols.


## Features
- **Quick Port Validation:** Uses `nc` to verify port status before checking the protocol with `nxc`
- **Protocol Suite:** Automatically sweeps **SMB**, **LDAP**, **WinRM**, **RDP**, **MSSQL**, and **FTP**
- **Password or Hash Auth:** Use `-p <password>` or `-H <hash>` (Pass-the-Hash) — FTP is auto-skipped in hash mode
- **Single Host or Subnet:** Accepts a single IP or a CIDR (e.g. `10.10.10.0/24`) — subnet mode hands the range to `nxc` natively, so flags like `--continue-on-success` work as expected
- **Versatile Targeting:** Seamless use in both **Active Directory** and standalone **Windows** environments
- **Dynamic Flag Passing:** Pass any native, global NetExec flags (e.g., `--local-auth`, `--continue-on-success`) directly through the wrapper
- **Clean Output:** Preserves native NetExec color coding for easy readability of `(Pwn3d!)` and share permissions


## Installation
For system-wide accessibility, use this one-liner to download the script, apply execution permissions, and move it into your $PATH:
```
curl -sSL 'https://raw.githubusercontent.com/corey-farley/nxc-sweep/main/nxc-sweep' -o nxc-sweep && chmod +x nxc-sweep && sudo mv nxc-sweep /usr/local/bin/
```


## Usage
```
nxc-sweep <IP|CIDR> -u <username> (-p <password> | -H <hash>) [--local-auth] [--continue-on-success]
```
`-p` and `-H` are mutually exclusive — supply exactly one. The hash accepted by `-H` is the same NT hash format NetExec takes (e.g. `aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0` or just the NT portion).

For a single IP, the wrapper port-checks with `nc` before invoking `nxc`. For a CIDR (e.g. `172.16.155.0/24`), the wrapper skips the port check and passes the range straight to `nxc`, which handles host iteration itself. Combine with `--continue-on-success` to validate creds across every host in the subnet:
```
nxc-sweep 172.16.155.0/24 -u joe -p 'Flowers1' --continue-on-success
```

LDAP is swept on port 389 and defaults to `--users` (domain user list with `badpwdcount` and descriptions). Swap that for whatever fits your engagement — e.g. `--asreproast`, `--kerberoasting`, or `--trusted-for-delegation` — by editing the `check_and_run 389 "ldap"` line in the script. Hash auth works with LDAP, so `-H` sweeps it normally, but LDAP is auto-skipped when `--local-auth` is passed — it's a domain service, so local auth can never succeed against it.

## Examples
Example 1:
```
└─$ nxc-sweep 10.129.34.53 -u 'rose' -p 'KxEPkKe6R8su'
[*] Starting NXC sweep for 10.129.34.53 as rose ...
                                                                                                                                                                   
[+] Port 445 open. Checking smb ...                                                                                                                                
SMB         10.129.34.53    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:None) (Null Auth:True)                                                                                                                                                         
SMB         10.129.34.53    445    DC01             [+] sequel.htb\rose:KxEPkKe6R8su 
SMB         10.129.34.53    445    DC01             [*] Enumerated shares
SMB         10.129.34.53    445    DC01             Share           Permissions     Remark
SMB         10.129.34.53    445    DC01             -----           -----------     ------
SMB         10.129.34.53    445    DC01             Accounting Department READ            
SMB         10.129.34.53    445    DC01             ADMIN$                          Remote Admin
SMB         10.129.34.53    445    DC01             C$                              Default share
SMB         10.129.34.53    445    DC01             IPC$            READ            Remote IPC
SMB         10.129.34.53    445    DC01             NETLOGON        READ            Logon server share 
SMB         10.129.34.53    445    DC01             SYSVOL          READ            Logon server share 
SMB         10.129.34.53    445    DC01             Users           READ            

[+] Port 5985 open. Checking winrm ...
WINRM       10.129.34.53    5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb)                                       
WINRM       10.129.34.53    5985   DC01             [-] sequel.htb\rose:KxEPkKe6R8su

[-] Port 3389 closed/filtered. Skipping rdp
                                                                                                                                                                   
[+] Port 1433 open. Checking mssql ...                                                                                                                             
MSSQL       10.129.34.53    1433   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb) (EncryptionReq:False)                 
MSSQL       10.129.34.53    1433   DC01             [+] sequel.htb\rose:KxEPkKe6R8su 
MSSQL       10.129.34.53    1433   DC01             name:master
MSSQL       10.129.34.53    1433   DC01             name:tempdb
MSSQL       10.129.34.53    1433   DC01             name:model
MSSQL       10.129.34.53    1433   DC01             name:msdb

[-] Port 21 closed/filtered. Skipping ftp
                                                                                                                                                                   
[*] All active services checked.  
```
Exmaple 2:
```
└─$ nxc-sweep 10.129.34.51 -u 'craig' -p 'password123' --local-auth
[*] Starting NXC sweep for 10.129.34.51 as craig ...
                                                                                                                                                                   
[+] Port 445 open. Checking smb ...                                                                                                                                
SMB         10.129.34.51    445    SUPPORTDESK      [*] Windows 10 / Server 2019 Build 17763 x64 (name:SUPPORTDESK) (domain:SUPPORTDESK) (signing:False) (SMBv1:None)                                                                                                                                                                 
SMB         10.129.34.51    445    SUPPORTDESK      [+] SUPPORTDESK\craig:password123
SMB         10.129.34.51    445    SUPPORTDESK      [*] Enumerated shares
SMB         10.129.34.51    445    SUPPORTDESK      Share           Permissions     Remark
SMB         10.129.34.51    445    SUPPORTDESK      -----           -----------     ------
SMB         10.129.34.51    445    SUPPORTDESK      ADMIN$                          Remote Admin
SMB         10.129.34.51    445    SUPPORTDESK      C$                              Default share
SMB         10.129.34.51    445    SUPPORTDESK      IPC$            READ            Remote IPC

[-] Port 5985 closed/filtered. Skipping winrm

[+] Port 3389 open. Checking rdp ...                                                                                                                                 
RDP       10.129.34.51    3389   SUPPORTDESK      [*] Windows 10 / Server 2019 Build 17763 (name:SUPPORTDESK) (domain:SUPPORTDESK)                               
RDP       10.129.34.51    3389   SUPPORTDESK      [+] SUPPORTDESK\craig:password123 (Pwn3d!)
                                                                                                                                                                   
[-] Port 1433 closed/filtered. Skipping mssql                                                                                                                      
                                                                                                                                                                   
[+] Port 21 open. Checking ftp ...                                                                                                                                 
FTP         10.129.34.51    21     10.129.34.51     [+] craig:password123
FTP         10.129.34.51    21     10.129.34.51     [*] Directory Listing
FTP         10.129.34.51    21     10.129.34.51     10-05-24  09:13AM                  952 Backup.psafe3                                                                                                                       
                                                                                                                                                                   
[*] All active services checked.                                                                                                                                                                    
```
Example 3 (Pass-the-Hash):
```
└─$ nxc-sweep 10.129.34.53 -u 'administrator' -H 'aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0' --local-auth
[*] Starting NXC sweep for 10.129.34.53 as administrator (hash auth) ...

[+] Port 445 open. Checking smb ...
SMB         10.129.34.53    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:DC01) (signing:True) (SMBv1:None)
SMB         10.129.34.53    445    DC01             [+] DC01\administrator:31d6cfe0d16ae931b73c59d7e0c089c0 (Pwn3d!)
SMB         10.129.34.53    445    DC01             [*] Enumerated shares
SMB         10.129.34.53    445    DC01             Share           Permissions     Remark
SMB         10.129.34.53    445    DC01             -----           -----------     ------
SMB         10.129.34.53    445    DC01             ADMIN$          READ,WRITE      Remote Admin
SMB         10.129.34.53    445    DC01             C$              READ,WRITE      Default share
SMB         10.129.34.53    445    DC01             IPC$            READ            Remote IPC

[+] Port 5985 open. Checking winrm ...
WINRM       10.129.34.53    5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:DC01)
WINRM       10.129.34.53    5985   DC01             [+] DC01\administrator (Pwn3d!)

[-] Port 3389 closed/filtered. Skipping rdp

[-] Port 1433 closed/filtered. Skipping mssql

[-] Skipping ftp (pass-the-hash not supported)

[*] All active services checked.
```