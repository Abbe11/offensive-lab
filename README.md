# Offensive Lab

Hands-on penetration testing practice. Each entry documents a machine I attacked, how I got in, why it worked, and how a defender would stop it.

Environment: HackTheBox Starting Point (Tier 0), authorized lab machines over VPN.

## Key takeaway so far

Across the first four machines, not one needed a software exploit. Every one was a misconfiguration or a service left open with no authentication. That mirrors the real world, where most compromise comes from bad configuration and weak credentials, not advanced exploits.

| Box | Service | Port | Way in | Type |
|-----|---------|------|--------|------|
| Meow | Telnet | 23 | root login, no password | Credentials |
| Fawn | FTP | 21 | anonymous login | Misconfiguration |
| Dancing | SMB | 445 | anonymous share access | Misconfiguration |
| Redeemer | Redis | 6379 | no authentication | Misconfiguration |

## Meow

Target OS: Linux. Difficulty: Very Easy.

Recon: ran `nmap -sV -sC <target>`. One open port, 23, running telnet.

Enumeration: telnet is an old remote login service that sends everything in plain text. Seeing it open on a modern machine is a red flag. A login service means the way in is credentials, not a software exploit.

Foothold: connected with `telnet <target>`. At the login prompt I entered root with no password, and it let me in, giving a root shell.

Why it worked: the machine allowed root to log in over telnet with no password. nmap found the door, telnet opened it, the missing password meant nothing stopped me. A misconfiguration, not a software bug.

Defense: disable telnet and use SSH, never allow direct root login, require a strong password.

## Fawn

Target OS: Unix. Difficulty: Very Easy.

Recon: `nmap -sV -sC <target>`. One open port, 21, running FTP (vsftpd 3.0.3). The nmap scripts reported anonymous FTP login was allowed and listed a file called flag.txt.

Foothold: connected with `ftp <target>`, logged in as anonymous with no password, listed files, and downloaded flag.txt with `get flag.txt`.

Why it worked: the FTP server allowed anonymous access and a sensitive file was readable by anyone. Same pattern as Meow, a login service left open.

Defense: disable anonymous FTP, never store sensitive files where anonymous users can reach them.

## Dancing

Target OS: Windows. Difficulty: Very Easy.

Recon: `nmap -sV -sC <target>`. Open ports 135 (RPC), 139 and 445 (SMB), 5985 (WinRM). A Windows machine with SMB as the main attack surface.

Enumeration: listed SMB shares anonymously with `smbclient -L //<target>/ -N`. Returned ADMIN$, C$, IPC$, and WorkShares. The three ending in $ are default hidden admin shares. WorkShares was a custom share, so that was the target.

Foothold: connected with `smbclient //<target>/WorkShares -N`, no password, navigated the folders, found flag.txt inside the James.P directory, and downloaded it with `get`.

Why it worked: SMB allowed anonymous null-session access, so shares could be listed and read with no credentials.

Defense: disable anonymous and guest SMB access, require authentication on shares, keep sensitive files off open shares.

## Redeemer

Target OS: Linux. Difficulty: Very Easy.

Recon: a default nmap scan found nothing, because nmap only scans the top 1000 ports by default and this service was on an uncommon port. A full scan found it: `nmap -p- --min-rate 1000 <target>`. Open port 6379, running Redis.

Enumeration: Redis is an in-memory key-value database that should never be exposed without a password. Connected with `redis-cli -h <target>`, ran `info` with no authentication, and the keyspace section showed database 0 held four keys.

Foothold: `select 0` to pick the database, `keys *` to list the keys, `get flag` to read the flag value.

Why it worked: Redis was exposed to the network with authentication disabled, so anyone could connect and read the data.

Defense: bind Redis to localhost, set a password with requirepass, firewall port 6379. In the real world an exposed Redis can lead to full server takeover, not just data theft.
