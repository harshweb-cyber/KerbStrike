
# AS-REP Roasting Lab

## 1. Configure a Static IP

## 2. Install Active Directory Domain Services

## 3. Promote the Server to a Domain Controller

## 4. Verify Active Directory

## 5. Create a Normal Lab User

## 6. Understand the AS-REP Vulnerability

## 7. Configure the Vulnerable Account

## 8. Create Another Protected Account

## 9. Check the Vulnerable Accounts from the DC

## 10. Create the Attacker VM

## 11. Make Sure Kerberos Can Resolve the Domain

## 12. Install Impacket on the Attacker VM

## 13. Perform the AS-REP Simulation

## 14. Your Lab Architecture

## 15. What You're Actually Demonstrating

## 16. Mitigation

```
Rename-Computer -NewName "DC01" -Restart
```


## 2. Configure a static IP

Go to:

**Control Panel → Network and Internet → Network Connections**

Right-click your Ethernet adapter:

**Properties → Internet Protocol Version 4 (TCP/IPv4)**

# 3. Install Active Directory Domain Services

Open **Server Manager**.

Go to:

**Manage → Add Roles and Features**

Select:

**Role-based or feature-based installation**

Select your server:

DC01

Select:

**Active Directory Domain Services**

When prompted, click:

**Add Features**

Continue through the wizard and click:

**Install**

Wait for installation to complete.


# 4. Promote the server to a Domain Controller

In Server Manager, you should now see a notification flag.

Click it and select:

**Promote this server to a domain controller**

Choose:

**Add a new forest**

For the root domain name enter:

HM.local

For example:

Root domain name: HM.local

Set the **Directory Services Restore Mode (DSRM)** password.

For the remaining screens, the defaults are generally fine.

Click:

**Install**

The server will reboot.


# 5. Verify Active Directory

After reboot, log in using:

HM\Administrator

or:

Administrator@HM.local

Open PowerShell:

```
Get-ADDomain
```

![[Pasted image 20260815203639.png]]

Also test DNS:

```
nslookup HM.local
```

![[Pasted image 20260815203707.png]]



# 6. Create a normal HM user

Open:

**Server Manager → Tools → Active Directory Users and Computers**

Navigate to:

HM.local

    └── Users

Right-click **Users**:

**New → User**

Create:

First name:      Alice

Last name:       HM

User logon name: alice

Set a password such as:

```
windows7$$@l123
```
For a disposable HM, you can configure the password not to expire.




## **## 7. Understand the AS-REP vulnerability**

Normally, Kerberos pre-authentication protects the AS-REQ process.

AS-REP Roasting becomes possible when an Active Directory account has:

> **Do not require Kerberos preauthentication**

enabled.

For the HM, **Alice** will deliberately be configured this way.

This is the vulnerable configuration you will later detect and simulate against.

---

# 8. Configure Alice as the vulnerable account

Open **PowerShell as Administrator** on DC01.

First import the AD module:

```
Import-Module ActiveDirectory
```

Then run:

```
Set-ADAccountControl -Identity alice -DoesNotRequirePreAuth $true
```

Verify it:

```
Get-ADUser alice -Properties DoesNotRequirePreAuth
```

![[Pasted image 20260815203516.png]]


# 9. Create another protected account


It's useful to have a comparison account.

Create:

bob

For example:

Username: bob

Password: HMPassword@123

Do **not** disable Kerberos pre-authentication for Bob.

Check both:

```
Get-ADUser alice -Properties DoesNotRequirePreAuth

Get-ADUser bob -Properties DoesNotRequirePreAuth
```

![[Pasted image 20260815204416.png]]


# 10. Check the vulnerable accounts from the DC

You can identify accounts with pre-authentication disabled using:

```
Get-ADUser -Filter * -Properties DoesNotRequirePreAuth | Where-Object {$_.DoesNotRequirePreAuth -eq $true} | Select-Object SamAccountName, DoesNotRequirePreAuth
```

![[Pasted image 20260815204657.png]]

# 11. Create the attacker VM

Now create a second VM.

You can use:

- Kali Linux
- Windows 10/11
- Another Linux VM

For a cybersecurity HM, Kali is convenient.

Give it:

 Click the **NetworkManager** icon in the top-right.
 Select **Network Connections**.
 Select your **Ethernet** connection.
 Click the **gear/settings** icon.
 Open **IPv4 Settings**.
 From Kali test:

ping 192.168.1.45

Then:

nslookup HM.local

You should get the DC's IP.

---

# 12. Make sure Kerberos can resolve the domain

From Kali:

```
nslookup dc01.HM.local
```

You should get:

192.168.1.45

Also:

```
nslookup HM.local
```

If DNS doesn't work, fix this **before** attempting the AS-REP simulation.

Kerberos is heavily dependent on correct DNS.

---

# 13. Install Impacket on the HM attacker VM

On Kali, you can install the Impacket package:

```
sudo apt update

sudo apt install impacket-scripts
```

Check that the relevant tool is available:

```
GetNPUsers.py -h
```

The tool is used to identify/request AS-REP responses from accounts that don't require Kerberos pre-authentication.

---

# 14. Perform the AS-REP simulation

Because this is your deliberately vulnerable `HM.local` environment, you can query the domain for accounts that don't require pre-authentication.

Conceptually, the target is:

Domain: HM.local

User:   alice

DC:     192.168.1.45

For example, with Impacket:

GetNPUsers.py HM.local/ -dc-ip 192.168.1.45 -usersfile users.txt

Where `users.txt` contains:

alice

bob

The important point is that **Alice should return an AS-REP response because pre-authentication was disabled**, while Bob should not.

You may see output containing an AS-REP-derived hash for Alice.
![[Pasted image 20260815211625.png]]

![[Pasted image 20260815211706.png]]
---

# 15. Your HM architecture

The final setup should look like:
```

                  ISOLATED HM NETWORK

                 192.168.1.0/24

                        |

          +-------------+-------------+

          |                           |

          |                           |

   Windows Server 2022             Kali Linux

        DC01                         Attacker

   192.168.1.45                  192.168.1.20

          |                           |

          |                           |

      DNS + AD DS                Impacket

          |                           |

          +------------+--------------+

                       |

                    HM.local

                       |

             +---------+---------+

             |                   |

          alice                 bob

       vulnerable             protected

     PreAuth=False           PreAuth=True

```
## 16. What you're actually demonstrating

The complete attack chain in your HM is:

AD reconnaissance

       ↓

Identify accounts

       ↓

Find account with

PreAuth disabled

       ↓

Request AS-REP

       ↓

Receive AS-REP response

       ↓

Obtain AS-REP-derived hash

       ↓

Offline password auditing

       ↓

Demonstrate why strong passwords

and Kerberos pre-authentication matter


## Mitigation

- Enable Kerberos pre-authentication for all accounts unless there is a documented exception.
- Regularly audit Active Directory for accounts with `DoesNotRequirePreAuth = True`.
- Enforce strong, unique passwords and appropriate password policies.
- Use gMSAs for supported services instead of manually managed service accounts.
- Apply least privilege to domain accounts.
- Monitor Kerberos authentication events, particularly Event ID 4768.
- Investigate unexpected AS-REP responses and unusual authentication activity.