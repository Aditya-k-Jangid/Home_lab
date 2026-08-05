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



