# Wazuh SIEM Setup — AD Lab

Sets up a Wazuh manager (SIEM) on Ubuntu and connects a Windows Server DC as a monitored agent. Used to capture Security/Kerberos/AD CS events generated during attacks (e.g. the ESC1 chain) for detection practice.

---

## Architecture

```
Windows Server DC (agent) ---> Ubuntu (Wazuh manager + indexer + dashboard)
        events forwarded over TCP 1514 (data) / 1515 (enrollment)
```

---

## 1. Install Wazuh manager (Ubuntu)

All-in-one install — manager, indexer, and dashboard in a single script:

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

This takes several minutes. At the end it prints an **admin password for the dashboard** — save it, it's not shown again.

Confirm the manager is running:
```bash
sudo systemctl status wazuh-manager
```

Confirm it's listening on the right ports:
```bash
sudo ss -tlnp | grep -E '1514|1515'
```

If `ufw` is active, allow the ports:
```bash
sudo ufw allow 1514/tcp
sudo ufw allow 1515/tcp
sudo ufw reload
```

Find Ubuntu's IP (this is the `WAZUH_MANAGER` value used later):
```bash
ip addr show
```

---

## 1.5. Join Ubuntu to the AD domain

Do this early if you want the Wazuh box itself domain-joined (e.g. to test Linux-AD auth, or so domain machines can resolve it by hostname). Can be done before or after the Wazuh manager install — order doesn't matter functionally.

### Install required packages
```bash
sudo apt update
sudo apt install realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin samba winbind oddjob oddjob-mkhomedir packagekit krb5-user
```

### Point DNS at the domain controller
```bash
sudo nano /etc/resolv.conf
```
```
nameserver <dc-ip>
search xyz.com
```

### Check the domain is reachable / discoverable
```bash
sudo realm discover xyz.com
```
A working response returns the realm name, domain controller, and `configured: kerberos-member` (once joined) — if this fails, it's almost always DNS pointing at the wrong nameserver.

### Join the domain
```bash
sudo realm join xyz.com -U Administrator --verbose
```

**If this fails with `Message stream modified`, skip to Section 7 (Troubleshooting) below** — it's a Kerberos encryption-negotiation issue with a known fix (`--membership-software=samba`), not a credentials/DNS problem.

### Verify the join and test AD user resolution
```bash
realm list
```
Should show `configured: kerberos-member`, `server-software: active-directory`, `client-software: sssd`.

```bash
id Orange@xyz.com
```
Should resolve Orange's UID/GID and AD group memberships.

```bash
su - Orange@xyz.com
```
Logs in as the AD user directly, confirming full auth (not just NSS lookup) works.

### (Optional) Kerberos ticket test
```bash
kinit Administrator@xyz.com
klist
```

### (Optional) Auto-create home directories for AD logins
```bash
sudo pam-auth-update
# enable "Create home directory on login"
```

---

## 2. Access the dashboard

```
https://<ubuntu-ip>
```
Login as `admin` with the password from step 1. Accept the self-signed cert warning.

### Finding "Deploy new agent" in the dashboard

Navigation varies slightly by version, but the path is:

1. Click the **☰ menu icon** (top-left).
2. Expand **Server management** (or **Agents management** on older versions).
3. Click **Endpoints summary** (or **Summary** / **Agents**).
4. The **Deploy new agent** button is at the top-right of that page.

<img width="1275" height="686" alt="Screenshot 2026-07-29 182154" src="https://github.com/user-attachments/assets/3792a29d-1356-4018-803c-b610d146683f" />


Or go directly to:
```
https://<ubuntu-ip>/app/wazuh#/agents-preview
```

Add the required Details (IP,Hostname etc) copy the generated cmd,

---

## 3. Install the agent on the Windows DC

Run in an elevated PowerShell session on the DC:

```
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER='<ubuntu-ip>' WAZUH_AGENT_GROUP='default' WAZUH_AGENT_NAME='<unique-agent-name>'
NET START WazuhSvc
```


<img width="1268" height="98" alt="Screenshot 2026-07-29 182247" src="https://github.com/user-attachments/assets/7c291387-676a-4594-bbb7-4e09d11bc5d1" />


**Important:** `WAZUH_AGENT_NAME` must NOT match the Ubuntu manager's own hostname (e.g. if the manager's `hostname` is `wazha`, don't name the agent `wazha` — Wazuh rejects it with `Invalid agent name (same as manager)`). Use something like `DC01` instead.

---

## 4. Verify the agent connects

On Ubuntu, watch the manager log while the agent starts:
```bash
sudo tail -f /var/ossec/logs/ossec.log
```
Expect:
```
INFO: Received request for a new agent (DC01) from: <dc-ip>
INFO: Agent key generated for 'DC01'
```

Check agent status:
```bash
sudo /var/ossec/bin/agent_control -l
```

Or check the dashboard → **Agents** — status should read `Active`.


<img width="1277" height="520" alt="Screenshot 2026-07-29 183502" src="https://github.com/user-attachments/assets/5b3084ec-ff7a-4a00-8bc1-93baf44608cc" />



**If connection fails, check:**
```powershell
# from the DC
Test-NetConnection -ComputerName <ubuntu-ip> -Port 1514
```
Common causes: wrong `WAZUH_MANAGER` IP in `C:\Program Files (x86)\ossec-agent\ossec.conf`, Ubuntu firewall blocking 1514/1515, or agent name collision (see step 3).

To fix config after install without reinstalling:
```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
# fix <address> or <agent_name> under <client>/<enrollment>
NET STOP WazuhSvc
NET START WazuhSvc
```

---

## 5. Enable Windows auditing (required — off by default)

Wazuh only forwards what Windows actually logs. Most AD/Kerberos/AD CS events are **not audited by default**, so the pipeline can look "connected" while producing nothing useful.

**AD CS auditing** (needed for cert request/issuance events like 4886/4887/4888):
```powershell
certutil -setreg CA\AuditFilter 127
auditpol /set /subcategory:"Certification Services" /success:enable /failure:enable
Restart-Service certsvc
```

**Kerberos auditing** (needed for 4768/4769/4770 — TGT requests, service tickets):
```powershell
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
auditpol /set /subcategory:"Kerberos Service Ticket Operations" /success:enable /failure:enable
```

Check current audit categories:
```powershell
auditpol /get /category:*
```

---

## 6. (Optional) Add Sysmon for process-level visibility

Security event log alone won't show process creation/command-line detail. Sysmon fills that gap:

```powershell
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile sysmon.zip
Expand-Archive sysmon.zip
.\Sysmon\Sysmon64.exe -i -accepteula
```

Add to the agent config so Wazuh picks up Sysmon events:
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```
Edit at `C:\Program Files (x86)\ossec-agent\ossec.conf`, then restart the agent:
```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```

---

## 7. Domain-join troubleshooting: `Message stream modified`

Continuation of Section 1.5 — use this if `realm join` failed there.

### If join fails with: `Couldn't set password for computer account: WAZHA$: Message stream modified`

This is a Kerberos/LDAP encryption-negotiation issue (adcli defaulting to encryption types the DC rejects), not a DNS or credentials problem. Editing `/etc/krb5.conf` does **not** fix it — `realmd`/`adcli` generate their own temporary krb5 config snippet that ignores it.

**Fix that actually worked — force the Samba/MS-RPC join path instead of adcli/LDAP:**

```bash
# delete any partially-created computer object first (ADUC on the DC, or adcli):
sudo adcli delete-computer --domain=xyz.com WAZHA -U Administrator

# then join forcing samba as the membership backend
sudo realm join xyz.com -U Administrator --membership-software=samba --verbose
```

Samba uses RPC to set the machine account password instead of the LDAP SASL exchange adcli uses, which sidesteps the encryption negotiation adcli was failing on.

You may see two harmless warnings during this join:
- `DNS update failed: NT_STATUS_INVALID_PARAMETER` / `No DNS domain configured` — the join succeeded, but Samba couldn't auto-register the hostname in AD DNS. Add the DNS record manually (below) if other machines need to reach this box by name.
- `Failed to update Kerberos configuration, not fatal...` — a non-fatal local file-attribute write; safe to ignore.

### Verify the join
```bash
realm list
```
Should show `configured: kerberos-member`, `server-software: active-directory`, `client-software: sssd`.

```bash
id Administrator@xyz.com
kinit Administrator@xyz.com
klist
```

### (Optional) Enable home directory auto-creation for AD logins
```bash
sudo pam-auth-update
# enable "Create home directory on login"
```

### Register DNS for this host (since dynamic DNS update failed above)

**Manual (simplest, recommended) — on the DC:**
1. Open `dnsmgmt.msc`
2. Forward Lookup Zones → `xyz.com` → right-click → **New Host (A or AAAA)**
3. Name: `wazha` (or your hostname), IP: this machine's IP → Add Host

**Or enable dynamic DNS updates from the Linux side:**
```bash
sudo hostnamectl set-hostname wazha.xyz.com
hostname -f   # should print wazha.xyz.com
```
Edit `/etc/sssd/sssd.conf`, under `[domain/xyz.com]`:
```ini
dyndns_update = true
dyndns_refresh_interval = 43200
dyndns_update_ptr = true
dyndns_auth = GSSKEYTAB
```
```bash
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl restart sssd
```

**Test resolution from another domain machine:**
```
nslookup wazha.xyz.com
ping wazha.xyz.com
```

---

## 8. End-to-end test

Trigger an event from the attacker box (e.g. re-run ESC1 enumeration):
```bash
certipy-ad find -u Orange -p 'secret2!' -dc-ip <dc-ip> -vulnerable -stdout
```

In the Wazuh dashboard, go to **Threat Hunting** (or **Security Events**), filter by `agent.name: DC01`, and check the last few minutes for related events. This confirms full pipeline: attack → Windows logs it → Wazuh agent forwards it → manager indexes it → visible in dashboard.

---

## Known issues hit during setup

- **`Invalid agent name (same as manager)`** — Ubuntu manager's hostname collided with the chosen agent name. Fix: pick a distinct agent name (e.g. `DC01`), edit `ossec.conf`, restart the service.
- **`Test-NetConnection` false alarm** — double-check `SourceAddress` in the output isn't accidentally the same as the target IP (sign you're testing against the wrong host, e.g. the DC's own IP instead of Ubuntu's).
- **`realm join` fails with `Message stream modified`** — encryption-type negotiation issue between `adcli`/LDAP and the DC (not fixable via `/etc/krb5.conf`, since `realmd` generates its own temp config). Fix: delete the partial computer account, then rejoin with `--membership-software=samba` to use the RPC-based join path instead.
- **`kdestroy: command not found`** — Kerberos client tools weren't installed; `sudo apt install krb5-user` fixes it.
- **`update-crypto-policies: command not found`** — that command is RHEL/Fedora-only; not applicable on Ubuntu/Debian.
