# nxc-sweep

Bash wrapper around [NetExec](https://github.com/Pennyw0rth/NetExec) that takes one set of credentials and sweeps every service worth checking on a Windows/AD host — **SMB, LDAP, WinRM, RDP, MSSQL, FTP** — in a single command.

```
nxc-sweep 10.129.34.53 -u 'rose' -p 'KxEPkKe6R8su'
```


## Why?
Running `nxc` per-protocol by hand gets tedious fast, especially when you keep pivoting between users. This wrapper pre-validates ports with netcat, then sweeps every relevant service in one go, so a fresh set of creds turns into a full picture of what they unlock without six separate commands.

Built for HTB labs and OSCP prep, where the first question after every credential dump is always "what does this actually get me?"


## Features
- **One credential, every protocol** — SMB, LDAP, WinRM, RDP, MSSQL and FTP swept in a single pass
- **Quick port validation** — `nc` checks each port before `nxc` is invoked, so closed services cost nothing
- **Password or hash auth** — `-p <password>` or `-H <hash>` for Pass-the-Hash
- **Single host or subnet** — takes an IP or a CIDR; subnet mode hands the range to `nxc` natively
- **Smart skips** — protocols that cannot work with the current auth mode are skipped instead of throwing errors
- **Dynamic flag passing** — any native NetExec flag (`--local-auth`, `--continue-on-success`, …) is forwarded straight through
- **Clean output** — native NetExec colouring is preserved, so `(Pwn3d!)` and share permissions stay readable


## Requirements
| Tool | Why |
| --- | --- |
| [NetExec](https://github.com/Pennyw0rth/NetExec) (`nxc`) | does the actual work |
| `nc` (netcat) | port preflight in single-host mode |
| `bash` 4.0+ | arrays and `[[ ]]` |

Everything else is stock — the wrapper is one self-contained script with no libraries to install.


## Installation
One-liner: download, make executable, drop it in your `$PATH`.
```
curl -sSL 'https://raw.githubusercontent.com/baktistr/nxc-sweep/main/nxc-sweep' -o nxc-sweep && chmod +x nxc-sweep && sudo mv nxc-sweep /usr/local/bin/
```

Or from a clone, if you plan to tweak the protocol list:
```
git clone https://github.com/baktistr/nxc-sweep.git
cd nxc-sweep && chmod +x nxc-sweep
sudo ln -s "$PWD/nxc-sweep" /usr/local/bin/nxc-sweep
```

To update an existing install, re-run the one-liner — it overwrites in place. Check which build you are on: the banner of a current build reads `... as rose (password auth) ...`. If the auth mode is missing from that line, you are running an older copy.


## Usage
```
nxc-sweep <IP|CIDR> -u <username> (-p <password> | -H <hash>) [global nxc flags]
```
`-p` and `-H` are mutually exclusive — supply exactly one. `-H` takes the same NT hash format NetExec does (`aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0`, or just the NT half). Every other flag is forwarded to `nxc` untouched.

### Common invocations
```
# domain creds, single host — sweeps every protocol
nxc-sweep 10.129.34.53 -u 'rose' -p 'KxEPkKe6R8su'

# local (SAM) account — LDAP is skipped, --local-auth is stripped for FTP
nxc-sweep 10.129.34.51 -u 'administrator' -p 'password123' --local-auth

# pass-the-hash — FTP is skipped, LDAP still sweeps
nxc-sweep 10.129.34.53 -u 'administrator' -H 'aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0'

# whole subnet, keep going past the first valid login
nxc-sweep 172.16.155.0/24 -u 'joe' -p 'Flowers1' --continue-on-success
```

### What gets swept
| Port | Protocol | Default options | Auto-skipped when |
| --- | --- | --- | --- |
| 445 | `smb` | `--shares` | — |
| 389 | `ldap` | `--users` | `--local-auth` is passed |
| 5985 | `winrm` | — | — |
| 3389 | `rdp` | — | — |
| 1433 | `mssql` | `-q "SELECT name FROM master.sys.databases;"` | — |
| 21 | `ftp` | `--ls` | `-H` (hash) auth is used |

The two skips exist because the combination cannot succeed: LDAP is a domain service, so `--local-auth` can never authenticate against it, and FTP has no notion of Pass-the-Hash. `--local-auth` is also stripped from the FTP call specifically, since passing it there crashes the FTP module.

### Targeting
For a single IP the wrapper port-checks with `nc` first, so only live services are queried. For a CIDR such as `172.16.155.0/24` it skips the port check entirely and passes the range straight to `nxc`, which handles host iteration itself — pair that with `--continue-on-success` to validate creds across every host in the subnet.


## Customizing
The whole protocol list lives in one block at the bottom of the script:
```
check_and_run 445 "smb" "--shares"
check_and_run 389 "ldap" "--users"
```
`check_and_run <port> <protocol> [extra nxc flags...]` — add a line to sweep another protocol (`wmi`, `vnc`, `ssh`), or change the trailing flags to retune an existing check. LDAP defaults to `--users`; swap it for `--asreproast`, `--kerberoasting`, or `--trusted-for-delegation` depending on what the engagement calls for.


## Sample output
Domain credentials against a DC:
```
┌──[19:34:35]─[kali@kali]─[Documents]
└──╼ $ nxc-sweep 10.1.186.176 -u 'mprice' -p '*martini*'
[*] Starting NXC sweep for 10.1.186.176 as mprice (password auth) ...

[+] Port 445 open. Checking smb ...
SMB         10.1.186.176    445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:DRY.MARTINI.BARS) (signing:False) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.1.186.176    445    DC01             [+] DRY.MARTINI.BARS\mprice:*martini*
SMB         10.1.186.176    445    DC01             [*] Enumerated shares
SMB         10.1.186.176    445    DC01             Share           Permissions            Remark
SMB         10.1.186.176    445    DC01             -----           -----------            ------
SMB         10.1.186.176    445    DC01             ADMIN$                                 Remote Admin
SMB         10.1.186.176    445    DC01             C$                                     Default share
SMB         10.1.186.176    445    DC01             IPC$            READ                   Remote IPC
SMB         10.1.186.176    445    DC01             NETLOGON        READ                   Logon server share
SMB         10.1.186.176    445    DC01             notes           READ,WRITE
SMB         10.1.186.176    445    DC01             SYSVOL          READ                   Logon server share

[+] Port 389 open. Checking ldap ...
LDAP        10.1.186.176    389    DC01             [*] Windows 11 / Server 2025 Build 26100 (name:DC01) (domain:DRY.MARTINI.BARS) (signing:Enforced) (channel binding:No TLS cert)
LDAP        10.1.186.176    389    DC01             [+] DRY.MARTINI.BARS\mprice:*martini*
LDAP        10.1.186.176    389    DC01             [*] Enumerated 6 domain users: DRY.MARTINI.BARS
LDAP        10.1.186.176    389    DC01             -Username-                    -Last PW Set-       -BadPW-  -Description-
LDAP        10.1.186.176    389    DC01             Administrator                 2026-01-12 11:00:19 2        Built-in account for administering the computer/domain
LDAP        10.1.186.176    389    DC01             Guest                         <never>             0        Built-in account for guest access to the computer/domain
LDAP        10.1.186.176    389    DC01             krbtgt                        2026-01-16 20:19:20 0        Key Distribution Center Service Account
LDAP        10.1.186.176    389    DC01             mprice                        2026-01-17 11:40:55 0
LDAP        10.1.186.176    389    DC01             athena.t0                     2026-01-20 13:20:44 0
LDAP        10.1.186.176    389    DC01             ATHENA_SVC                    2026-01-20 13:20:32 0

[+] Port 5985 open. Checking winrm ...
WINRM       10.1.186.176    5985   DC01             [*] Windows 11 / Server 2025 Build 26100 (name:DC01) (domain:DRY.MARTINI.BARS)
WINRM       10.1.186.176    5985   DC01             [-] DRY.MARTINI.BARS\mprice:*martini*

[+] Port 3389 open. Checking rdp ...
RDP         10.1.186.176    3389   DC01             [*] Windows 10 or Windows Server 2016 Build 26100 (name:DC01) (domain:DRY.MARTINI.BARS) (nla:True)
RDP         10.1.186.176    3389   DC01             [+] DRY.MARTINI.BARS\mprice:*martini*

[-] Port 1433 closed/filtered. Skipping mssql

[-] Port 21 closed/filtered. Skipping ftp

[*] All active services checked.
```

> The two runs below were captured on an earlier build — they predate the LDAP check and the `(password auth)` suffix in the banner, so no LDAP block appears.

Local account with `--local-auth`:
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

Pass-the-Hash:
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


## Credits
Fork of [corey-farley/nxc-sweep](https://github.com/corey-farley/nxc-sweep). This fork adds hash authentication, CIDR/subnet sweeping, and the LDAP check.
