# AD Attack Lab — Setup Guide

Full build guide for the Active Directory attack and detection lab. Covers software downloads, VMware network configuration, domain setup, vulnerable account configuration, Sysmon deployment, and Wazuh agent enrollment.

---

## Lab Overview

| VM | OS | RAM | Disk | IP | Role |
|---|---|---|---|---|---|
| DC01 | Windows Server 2022 | 4 GB | 60 GB | 192.168.93.50 | Domain Controller (corp.local) |
| VICTIM | Windows 10 22H2 | 4 GB | 60 GB | 192.168.93.155 | Domain-joined workstation |
| KALI | Kali Linux | 4 GB | 40 GB | 192.168.93.156 | Attacker |
| WAZUH | Existing instance | — | — | 192.168.93.x | SIEM / log collection |

All VMs sit on the same isolated internal network. No internet access required after initial setup.

---

## Part 1 — Software Downloads

### Windows Server 2022 (ISO)

Source: Microsoft Evaluation Center
Search: "Windows Server 2022 evaluation download"
- Select **ISO** format (not VHD)
- Choose the **Desktop Experience** edition — this gives you a GUI, which you need for AD management
- The eval license is valid for 180 days — no key needed

### Windows 10 Enterprise 22H2 (ISO)

Source: Microsoft Evaluation Center
Search: "Windows 10 Enterprise evaluation download"
- Select **ISO** format
- 64-bit only

### Kali Linux (VMware image)

Source: kali.org → Downloads → Virtual Machines
- Download the **VMware** pre-built image (saves time vs installing from ISO)
- It comes as a `.7z` archive — extract it and open the `.vmx` file directly in VMware
- Default credentials: `kali` / `kali`

### Sysmon

Source: Microsoft Sysinternals (part of the Sysinternals Suite)
Search: "Sysinternals Sysmon download"
- Download `Sysmon.zip` — extract to a folder you'll copy onto the Windows VMs

### SwiftOnSecurity Sysmon Config

Source: github.com/SwiftOnSecurity/sysmon-config
- Download `sysmonconfig-export.xml` from the repository
- This is the config file you pass to Sysmon during installation

---

## Part 2 — VMware Network Setup

All three VMs need to communicate on an isolated private network. The cleanest way to do this in VMware Workstation is with a **custom Host-Only network** (VMnet).

### Create the Custom VMnet

1. Open VMware Workstation → **Edit** → **Virtual Network Editor**
2. Click **Add Network** → select an unused VMnet (e.g., `VMnet2`)
3. Set type to **Host-only**
4. Uncheck **Use local DHCP service** — you'll assign IPs statically so they match the lab
5. Set Subnet IP: `192.168.93.0`
6. Set Subnet mask: `255.255.255.0`
7. Click **Apply** → **OK**

> Why Host-only and not NAT? Host-only keeps all attack traffic isolated — it never touches your actual network. NAT would let the VMs reach the internet, which is unnecessary and adds noise to your logs.

### Assign the VMnet to Each VM

For each VM, before powering on:

1. Right-click the VM → **Settings**
2. Select the **Network Adapter**
3. Set to **Custom: Specific virtual network** → choose `VMnet2` (or whichever you created)
4. Click **OK**

Do this for all three VMs (DC01, VICTIM, KALI) and your Wazuh instance.

---

## Part 3 — Windows Server 2022 — Initial Setup

### VM Creation (if building from ISO)

In VMware → **Create a New Virtual Machine**:
- Configuration: **Typical**
- Installer disc image: browse to your Windows Server 2022 ISO
- Guest OS: **Windows Server 2022**
- VM Name: `DC01`
- Disk: **60 GB**, store as a single file
- Customize Hardware:
  - RAM: **4096 MB**
  - Processors: 2
  - Network Adapter: set to VMnet2 as above

### Windows Installation

1. Boot the VM — Windows setup will launch automatically
2. Select **Windows Server 2022 Standard Evaluation (Desktop Experience)** — this is important, the version without Desktop Experience is Server Core (no GUI)
3. Accept license → **Custom install** → select the disk → install
4. Set a local Administrator password when prompted (e.g., `Admin123!` — this is isolated lab, keep it simple)

### Static IP Configuration

After Windows boots:

1. Open **Network and Sharing Center** → **Change adapter settings**
2. Right-click the adapter → **Properties** → **Internet Protocol Version 4 (TCP/IPv4)**
3. Set:
   - IP address: `192.168.93.50`
   - Subnet mask: `255.255.255.0`
   - Default gateway: (leave blank — no gateway needed on isolated network)
   - Preferred DNS: `127.0.0.1` (the DC will be its own DNS server after promotion)

4. Rename the computer before promoting to DC:
   - Right-click **This PC** → **Properties** → **Rename this PC**
   - Name it `DC01`
   - Restart when prompted

---

## Part 4 — Promote to Domain Controller

### Install the AD DS Role

Open **Server Manager** → **Add Roles and Features**:
1. Role-based installation → select local server
2. Check **Active Directory Domain Services**
3. Accept all additional features → install
4. Wait for installation to complete

### Promote to Domain Controller

In Server Manager, click the flag notification → **Promote this server to a domain controller**:

1. Select **Add a new forest**
2. Root domain name: `corp.local`
3. Click **Next**

On **Domain Controller Options**:
- Forest functional level: **Windows Server 2016** (fine for this lab)
- Domain functional level: **Windows Server 2016**
- Leave **DNS server** checked
- Set a DSRM password (Directory Services Restore Mode — e.g., `Admin123!`)

Accept all defaults through the remaining screens → **Install**

The server will restart automatically. After reboot, log in as `CORP\Administrator`.

---

## Part 5 — Domain Configuration

### Create the OU Structure

Open **Active Directory Users and Computers** (ADUC):
- Start → **Windows Administrative Tools** → **Active Directory Users and Computers**

Right-click `corp.local` → **New** → **Organizational Unit**:
- Create: `IT`
- Create: `Finance`
- Create: `HR`
- Create: `Service Accounts` (for the SPN accounts)

### Create User Accounts

Create 10–15 user accounts distributed across the OUs. Right-click an OU → **New** → **User**.

Suggested accounts (use realistic names — it makes the lab feel real):

| Name | Username | OU | Password | Notes |
|---|---|---|---|---|
| Ava Simmons | asimmons | HR | Summer2024! | AS-REP Roasting target |
| Eric Brooks | ebrooks | Finance | Winter2026! | Kerberoasting / PtH target |
| David Price | dprice | IT | Company1! | Normal user |
| Thomas Turner | tturner | IT | Spring2023! | Normal user |
| Nancy Bennett | nbennett | HR | Password1! | Normal user |
| Michael Coleman | mcoleman | Finance | Login123! | Normal user |
| SQL Service | svc_sql | Service Accounts | Winter2026! | Kerberoasting target (SPN) |
| Backup Service | svc_backup | Service Accounts | Winter2026! | Kerberoasting target (SPN) |
| (add ~5 more) | | | | Filler accounts for the spray |

**Uncheck "User must change password at next logon"** and check **"Password never expires"** on all accounts — this is a lab, you don't want lockouts.

### Set Weak Passwords

At least 3 accounts should use passwords that would appear in `rockyou.txt` or common corporate wordlists: `Summer2024!`, `Winter2026!`, `Company1!`. These are what the password spray and hash cracking will hit.

### Configure Kerberoastable Service Accounts (SPNs)

Open an elevated **Command Prompt** on the DC and register SPNs:

```cmd
setspn -A MSSQLSvc/sql.corp.local:1433 corp\svc_sql
setspn -A HTTP/backup.corp.local corp\svc_backup
```

Verify they registered:
```cmd
setspn -L corp\svc_sql
setspn -L corp\svc_backup
```

Both accounts now appear as Kerberoasting targets when enumerated from Kali.

### Configure the AS-REP Roastable Account

In ADUC, open the properties for `asimmons`:
1. Go to the **Account** tab
2. Scroll down in **Account options**
3. Check **Do not require Kerberos preauthentication**
4. Click **OK**

This disables the pre-auth check for asimmons. Any unauthenticated attacker can now request her AS-REP hash.

---

## Part 6 — Windows 10 Workstation Setup

### VM Creation

Same process as the DC:
- ISO: Windows 10 Enterprise 22H2
- VM Name: `VICTIM`
- RAM: 4096 MB
- Disk: 60 GB
- Network Adapter: VMnet2

During Windows installation, skip the Microsoft account login — choose **Domain join instead** (offline account) and complete setup with a local account first.

### Static IP

Same as the DC, but:
- IP address: `192.168.93.155`
- Subnet mask: `255.255.255.0`
- Preferred DNS: `192.168.93.50` ← point to the DC for domain resolution

### Join the Domain

1. Right-click **This PC** → **Properties** → **Advanced system settings** → **Computer Name** tab → **Change**
2. Select **Domain**, type `corp.local`
3. When prompted, enter DC Administrator credentials: `CORP\Administrator` and the admin password
4. Restart when prompted

After restart, log in as a domain user (e.g., `CORP\ebrooks`) to simulate a normal workstation session.

---

## Part 7 — Sysmon Deployment

Deploy Sysmon on **both the DC and the Windows 10 workstation**.

### Copy Files to Each VM

Copy both files to the VM (easiest via VMware drag-and-drop or a shared folder):
- `Sysmon64.exe`
- `sysmonconfig-export.xml` (from SwiftOnSecurity)

### Install Sysmon

Open an elevated PowerShell or Command Prompt and run:

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

### Verify Sysmon is Running

```powershell
Get-Service Sysmon64
```

Should show `Running`. Sysmon logs appear in:
**Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational**

Do this on both the DC and the workstation.

---

## Part 8 — Wazuh Agent Enrollment

Your Wazuh manager is already running. You need to enroll both Windows machines as agents.

### Download the Wazuh Agent

On each Windows VM, download the Wazuh Windows agent installer from your Wazuh manager's dashboard:
- Wazuh Dashboard → **Agents** → **Deploy new agent**
- Select **Windows**
- Copy the install command — it will include your manager's IP and a registration key

### Install the Agent

Run the installation command from an elevated PowerShell on each Windows VM. It will look something like:

```powershell
Invoke-WebRequest -Uri https://<WAZUH-IP>:8443/agents/wazuh-agent.msi -OutFile wazuh-agent.msi
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.93.x" WAZUH_AGENT_NAME="DC01"
NET START WazuhSvc
```

Adjust the agent name for each machine (`DC01` and `VICTIM`).

### Verify Agent Connection

In the Wazuh Dashboard → **Agents** — both agents should appear as **Active** within a minute or two.

### Enable Log Archiving on the Wazuh Manager

By default, Wazuh only indexes events that match a rule. For this lab you need all events (especially 4768/4769 for Kerberos attacks) to be searchable. SSH into your Wazuh manager and edit the config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find the `<global>` section and add:
```xml
<logall>yes</logall>
<logall_json>yes</logall_json>
```

Restart the manager:
```bash
sudo systemctl restart wazuh-manager
```

### Add the Archives Index Pattern

In the Wazuh Dashboard:
1. Go to **Dashboard Management** → **Index Patterns**
2. Click **Create index pattern**
3. Enter: `wazuh-archives-*`
4. Set **Time field** to `timestamp`
5. Save

Now you can search all logs in Discover — not just those that matched a rule.

---

## Part 9 — Baseline Verification

Before running any attacks, confirm the full log pipeline is working.

### Test 1 — Security Events Are Flowing

On the DC, open Event Viewer and generate a deliberate 4625 (failed logon):
- Lock the screen and enter a wrong password
- Check Wazuh Discover → filter on `data.win.system.eventID: 4625` — it should appear

### Test 2 — Kerberos Events Are Visible

From Kali (after setting up /etc/hosts):
```bash
echo "192.168.93.50 corp.local" >> /etc/hosts
```

Then authenticate as a domain user to generate a 4769:
```bash
impacket-GetUserSPNs corp.local/ebrooks:'Winter2026!' -dc-ip 192.168.93.50
```

Check Wazuh Discover → filter on `data.win.system.eventID: 4769` in `wazuh-archives-*`.

### Test 3 — Sysmon Events Are Flowing

On the Windows 10 machine, open a PowerShell window (Sysmon Event 1 — process create). In Wazuh Discover, filter on `data.win.system.channel: Microsoft-Windows-Sysmon/Operational` and confirm events appear.

If all three tests pass, the lab is ready for attack execution.

---

## Kali Setup

The Kali VMware image comes mostly ready. A few things to configure before starting attacks:

```bash
# Update the system
sudo apt update && sudo apt full-upgrade -y

# Install Impacket (if not already installed)
sudo apt install -y impacket-scripts

# Install kerbrute
sudo apt install -y kerbrute

# Install netexec (successor to crackmapexec)
sudo apt install -y netexec

# Set up the working directory structure
mkdir -p ~/ad_attacks/{password_spray,kerberoasting,asreproasting,pass_the_hash,dcsync}

# Add the DC to /etc/hosts
echo "192.168.93.50 corp.local dc01.corp.local" | sudo tee -a /etc/hosts
```

Assign the static IP on Kali:
```bash
sudo nano /etc/network/interfaces
```
```
auto eth0
iface eth0 inet static
  address 192.168.93.156
  netmask 255.255.255.0
```
```bash
sudo systemctl restart networking
```

Verify connectivity to the DC:
```bash
ping 192.168.93.50
nmap -p 88,389,445 192.168.93.50   # Kerberos, LDAP, SMB — all should be open
```

---

## Snapshot Strategy

Take VMware snapshots at each of these points so you can roll back without rebuilding:

| Snapshot Name | When to Take |
|---|---|
| `DC01 — Clean Domain` | After domain promotion, OU/user creation, SPN setup |
| `VICTIM — Domain Joined` | After joining the domain and installing Sysmon + Wazuh agent |
| `KALI — Ready` | After installing all tools and setting the static IP |
| `ALL — Pre-Attack Baseline` | After confirming logs are flowing in Wazuh — before Phase 2 |

Taking the `ALL — Pre-Attack Baseline` snapshot is the most important. It lets you revert all three VMs to a clean state and re-run attacks cleanly for Phase 3 rule testing.
