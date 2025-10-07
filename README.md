<p align="center">
  <img src="https://i.imgur.com/pU5A58S.png" alt="Microsoft Active Directory Logo"/>
</p>

<h1 align="center">Active Directory in Azure: My On-Prem Style Lab</h1>

I stood up a small AD environment in Azure using two VMs: one as a **Domain Controller** and one as a **client** joined to the domain.  
This doc is my personal runbook — the exact steps I took, notes I wished I had, and screenshots for quick reference.

---

##  What I Used

- **Microsoft Azure** (VMs / vNet / NICs)
- **RDP** for management
- **Active Directory Domain Services (AD DS)**
- **PowerShell** (ISE for scripting)

### OS
- **Windows Server 2022** (DC)
- **Windows 10 (21H2)** (Client)

---

##  High-Level Flow

1) Build two Azure VMs on the same vNet  
2) Make the DC’s **private IP static**  
3) Fix ICMP on the DC so the client can ping  
4) Install **AD DS** and **promote** to a new forest  
5) Create OUs/users and grant admin membership  
6) Point the client’s **DNS** at the DC, join the domain  
7) Enable RDP for domain users and bulk-create accounts with PowerShell  

---

## 1) Azure Resources

Create two VMs in the same resource group/vNet.

- **DC-1** (Windows Server 2022) → will become the Domain Controller  
  Take note of the **vNet** Azure creates.

<p align="center">
  <img src="https://i.imgur.com/mrpBWtM.png" width="70%" alt="Azure DC VM"/>
</p>

> Make **DC-1’s NIC IP** static so it never changes:
> Azure VM → **Networking** → NIC link → **IP configurations → ipconfig1** → Assignment: **Static**.

<p align="center">
  <img src="https://i.imgur.com/xcyLUOG.png" width="70%"/>
  <img src="https://i.imgur.com/ZaWdzTl.png" width="70%"/>
  <img src="https://i.imgur.com/Vn0UhWm.png" width="70%"/>
</p>

- **Client-1** (Windows 10 Pro)  
  Same **resource group** and **vNet** as DC-1.

<p align="center">
  <img src="https://i.imgur.com/Vf7yeY1.png" width="70%"/>
  <img src="https://i.imgur.com/3DK41Cr.png" width="70%"/>
</p>

---

## 2) Verify Connectivity (Ping)

From **Client-1** (RDP in), open **Command Prompt** and run a continuous ping to the DC’s private IP (e.g., `ping -t 10.1.0.4`).

- It will **time out** initially (Windows Firewall blocks ICMP by default on Server).

<p align="center">
  <img src="https://i.imgur.com/U6UOqj5.png" width="70%"/>
</p>

On **DC-1**, enable ICMP:

`Start → Windows Administrative Tools → Windows Defender Firewall with Advanced Security → Inbound Rules`

Enable the rules for **Core Networking Diagnostics** / **ICMPv4**.

<p align="center">
  <img src="https://i.imgur.com/bw6eoLh.png" width="50%"/>
  <img src="https://i.imgur.com/BY1Ohgb.png" width="80%"/>
</p>

Back on **Client-1**, pings should now succeed.

<p align="center">
  <img src="https://i.imgur.com/890WIJB.png" width="70%"/>
</p>

---

## 3) Install & Promote AD DS

On **DC-1**:

- Open **Server Manager → Add Roles and Features**
- Check **Active Directory Domain Services**
- Add features and complete the wizard

<p align="center">
  <img src="https://i.imgur.com/DQRVNnm.png" width="80%"/>
  <img src="https://i.imgur.com/RpzngRi.png" width="50%"/>
</p>

Promote to a Domain Controller:

- Click the **flag** in Server Manager → **Promote this server to a domain controller**
- Choose **Add a new forest**
- **Root domain**: `mydomain.com`
- Set a **DSRM password** and finish the wizard

<p align="center">
  <img src="https://i.imgur.com/GOYiTFe.png" width="70%"/>
  <img src="https://i.imgur.com/IjfUZ0a.png" width="70%"/>
</p>

The server reboots. Log back in as: `mydomain.com\labuser`.

<p align="center">
  <img src="https://i.imgur.com/oNp39DK.png" width="70%"/>
</p>

---

## 4) Create OUs and an Admin User

On **DC-1**:

- **Server Manager → Tools → Active Directory Users and Computers**
- Right-click the domain → **New → Organizational Unit**
  - `_EMPLOYEES`
  - `_ADMINS`

<p align="center">
  <img src="https://i.imgur.com/udGHbGs.png" width="70%"/>
  <img src="https://i.imgur.com/5wSZuA4.png" width="70%"/>
</p>

Create an admin user in `_ADMINS`:

- **New → User**
  - Name: **Jane Doe**
  - User logon: **jane_admin**
  - Set password (uncheck forced reset if desired)

<p align="center">
  <img src="https://i.imgur.com/nv6jc9p.png" width="70%"/>
  <img src="https://i.imgur.com/uLopQTZ.png" width="70%"/>
</p>

Grant domain admin rights:

- Right-click **Jane Doe → Properties → Member Of → Add**
- Add **Domain Admins** (and other admin groups as needed)
- Sign out and back in as `mydomain.com\jane_admin`

<p align="center">
  <img src="https://i.imgur.com/EapMhBs.png" width="70%"/>
  <img src="https://i.imgur.com/vGb8Kx8.png" width="70%"/>
</p>

---

## 5) Join the Client to the Domain

Point **Client-1** DNS to the DC:

Azure Portal → **Client-1 → Networking → NIC** → **DNS servers → Custom** → **DC-1 private IP** → **Save** → **Restart**.

<p align="center">
  <img src="https://i.imgur.com/z6UesO7.png" width="70%"/>
  <img src="https://i.imgur.com/bt0yK17.png" width="70%"/>
  <img src="https://i.imgur.com/sB5edH5.png" width="70%"/>
</p>

Join the domain (on Client-1, RDP as local admin):

- **Start → System → Rename this PC (Advanced) → Change**
- Member of: **Domain** → `mydomain.com`
- Credentials: `mydomain.com\jane_admin`
- Reboot when prompted

<p align="center">
  <img src="https://i.imgur.com/3HxJLpe.png" width="80%"/>
  <img src="https://i.imgur.com/J8M4zBU.png" width="50%"/>
</p>

---

## 6) Allow RDP for Non-Admins (Client-1)

Sign in to **Client-1** as `mydomain.com\jane_admin`:

- **Start → System → Remote Desktop**
- Under **User Accounts**, choose **Select users that can remotely access this PC → Add**
- Add the domain users/groups you want to allow

<p align="center">
  <img src="https://i.imgur.com/HgAXVMX.png" width="70%"/>
  <img src="https://i.imgur.com/0QDUk5l.png" width="60%"/>
</p>

---

## 7) Bulk-Create Users with PowerShell

Back on **DC-1** as **jane_admin**:

- Open **PowerShell ISE (Admin)**
- Click the **green arrow** to run it

<p align="center">
  <img src="https://i.imgur.com/MpvLIbB.png" width="70%"/>
  <img src="https://i.imgur.com/V4vIvre.png" width="70%"/>
</p>

Check results in **ADUC**: `_EMPLOYEES` should be filled with generated accounts.  
Test logging into **Client-1** with one of them (e.g., `base.milu` / `Password1`).

<p align="center">
  <img src="https://i.imgur.com/3HN1Nf4.png" width="80%"/>
  <img src="https://i.imgur.com/CeE8LGh.png" width="50%"/>
  <img src="https://i.imgur.com/7ZVBp8a.png" width="70%"/>
</p>

<p align="center">
  <img src="https://i.imgur.com/EzgHWRs.png" width="70%"/>
  <img src="https://i.imgur.com/hYFodxu.png" width="70%"/>
</p>

---

##  Result

- New **forest** (`mydomain.com`) running in Azure  
- **Client VM** joined to the domain and RDP-enabled for domain users  
- **OUs** created and **users** provisioned (scripted)  
- Clean connectivity and DNS flow (client → DC)



---
