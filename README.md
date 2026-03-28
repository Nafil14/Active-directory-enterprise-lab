# 🏢 Active Directory, Group Policy & Enterprise Monitoring
## Hybrid Enterprise IT Lab — Phase 2

> **Domain:** `corp.corpnet.com` &nbsp;|&nbsp; **DC:** `CORP-DC-01` @ `10.0.0.10` &nbsp;|&nbsp; **Client:** `DESKTOP-F8CRF32` @ `10.0.0.20` &nbsp;|&nbsp; **NOC:** Zabbix @ `10.0.0.24`
>
> 📎 *Phase 1 — Zabbix base setup on Ubuntu 24.04: [View Zabbix Repo](#)*

---

## 🧠 The Story

After getting Zabbix up and monitoring my Ubuntu server, I asked myself the question every enterprise sysadmin deals with daily: *how do you actually manage Windows machines at scale?*

The answer is **Active Directory + Group Policy** — the backbone of Windows enterprise management in organisations worldwide. Instead of just reading about it, I built the whole thing from scratch: a domain controller, a joined Windows 10 client, multiple GPOs doing real work, and a security delegation model for IT staff.

This isn't a tutorial follow-along. Every problem here was real. Every fix was earned.

---

## 🗺️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   VirtualBox — 10.0.0.0/24                      │
│                                                                 │
│  ┌─────────────────────┐          ┌──────────────────────────┐  │
│  │  CORP-DC-01         │          │  DESKTOP-F8CRF32         │  │
│  │  Windows Server 2022│◄─────────│  Windows 10 (22H2)       │  │
│  │  10.0.0.10          │  Domain  │  10.0.0.20               │  │
│  │                     │  Member  │                          │  │
│  │  ● Active Directory │          │  ● corp.corpnet.com ✅    │  │
│  │  ● DNS Server       │  GPOs ──►│  ● GPOs applying         │  │
│  │  ● GPMC             │          │  ● H:\ drive mapped      │  │
│  │  ● SMB Shares       │  SMB  ──►│  ● CMD blocked           │  │
│  │  ● Zabbix Agent 2   │          │  ● Control Panel blocked │  │
│  └─────────────────────┘          └──────────────────────────┘  │
│            │                                                     │
│            │ TCP 10050                                           │
│            ▼                                                     │
│  ┌─────────────────────┐                                        │
│  │  Ubuntu 24.04 NOC   │                                        │
│  │  Zabbix 7.0 LTS     │                                        │
│  │  10.0.0.24          │                                        │
│  └─────────────────────┘                                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 What Was Built

| Feature | Details |
|---------|---------|
| Domain | `corp.corpnet.com` |
| Domain Controller | `CORP-DC-01` (Windows Server 2022) |
| Client Machine | `DESKTOP-F8CRF32` (Windows 10 Build 19045.2965) |
| AD Security Group | `GS-IT-Staff` — delegated user management rights |
| GPO 1 | `company Wallpaper` — desktop wallpaper via UNC path |
| GPO 2 | `security_lockdown_policy` — CMD + Control Panel blocked |
| GPO 3 | `Mapped_drive_policy` — `H:` drive → `\\10.0.0.10\CompanyData` |
| SMB Share | `CompanyData` shared from DC, accessed by domain users |
| Monitoring | Zabbix Agent 2 on DC, reporting to Ubuntu NOC |

---

## 📖 Phase-by-Phase Walkthrough

### 🔷 Phase 1 — Active Directory & Delegation of Control

The first real enterprise task beyond just "install AD" is deciding *who gets to manage what*. In any real org, you don't give every IT person Domain Admin rights — that's a security nightmare. Instead, you delegate specific permissions to specific groups.

I created the `GS-IT-Staff` security group and used the **Delegation of Control Wizard** to grant them exactly what they need — no more, no less.

**Delegated permissions for `GS-IT-Staff`:**
- ✅ Create, delete, and manage user accounts
- ✅ Reset user passwords and force password change at next logon
- ❌ Everything else (manage groups, GPO links, etc.) — not their job

This mirrors how real enterprise IT departments operate: helpdesk can reset passwords, but they can't touch Group Policy.

---

**Screenshot: Selecting the `GS-IT-Staff` group in the Delegation Wizard**

![AD Delegation Wizard](screenshots/ad-delegation-wizard.png)

---

**Screenshot: Delegating only the necessary tasks — principle of least privilege in action**

![AD Delegation Tasks](screenshots/ad-delegation-tasks.png)

---

### 🔷 Phase 2 — Joining the Windows 10 Client to the Domain

With AD running on `CORP-DC-01`, it was time to bring in the client machine. The process seems simple but has a gotcha that trips up a lot of people: **the client's DNS must point to the DC before the domain join will work.** Windows needs to resolve `corp.corpnet.com` via the DC's DNS — not a public DNS server.

Once DNS was sorted, the join went through the classic four-step flow:

**Step 1 — Enter the domain name**

![Domain Join Dialog](screenshots/domain-join-dialog.png)

> `DESKTOP-F8CRF32` joining `corp.corpnet.com` via System Properties

---

**Step 2 — Authenticate with domain admin credentials**

![Domain Join Credentials](screenshots/domain-join-credentials.png)

> `CORPNET\ADMINISTRATOR` credentials used to authorize the join

---

**Step 3 — Success confirmation**

![Domain Join Success](screenshots/domain-join-success.png)

> *"Welcome to the corp.corpnet.com domain"* — the most satisfying dialog in Windows administration

---

**Step 4 — Reboot to apply**

![Domain Join Restart](screenshots/domain-join-restart.png)

> Domain membership requires a reboot to take effect

---

**Post-reboot verification — `ipconfig`**

![Client ipconfig](screenshots/client-ipconfig.png)

> Logged in as `admin1.CORP` — a domain account. IP: `10.0.0.20`, Gateway: `10.0.0.1`

---

**Post-reboot verification — `systeminfo`**

![Client systeminfo](screenshots/client-systeminfo.png)

> `Domain: corp.corpnet.com` ✅ | `Logon Server: \\CORP-DC-01` ✅ — the client is fully domain-joined and authenticating against the DC

---

### 🔷 Phase 3 — GPO 1: Corporate Desktop Wallpaper

The classic enterprise GPO task. The goal: push a corporate wallpaper to all domain users automatically, without touching each machine individually.

**Configuration on the DC:**

The wallpaper GPO points to a UNC path on the SMB share hosted on the DC:

```
\\10.0.0.10\CompanyData\Wallpaper
```

Style set to **Fill** so it scales properly on any screen resolution.

---

**Screenshot: Wallpaper GPO — first configured (empty path)**

![GPO Wallpaper Empty](screenshots/gpo-wallpaper-empty.png)

---

**Screenshot: Wallpaper GPO — UNC path and Fill style configured**

![GPO Wallpaper Path](screenshots/gpo-wallpaper-path.png)

---

**Setting up the SMB Share — the permission trap**

To serve the wallpaper file over the network, the `CompanyData` share needed correct permissions at *two* layers. This is where most people get stuck.

![SMB Share Permissions](screenshots/smb-share-permissions.png)

> Granting `GS-IT-Staff` access to the share — both **Share Permissions** and **NTFS Permissions** must be set. One without the other = Access Denied.

---

**The result on the client — a teachable moment**

![GPO Wallpaper White Rectangle](screenshots/gpo-wallpaper-white-rect.png)

> The GPO *applied* — the white rectangle proves Windows received the policy and attempted to render the wallpaper. However, the image file wasn't found at the UNC path at render time (file missing from share folder). This is actually a useful troubleshooting exhibit: **the policy reached the client, but the file delivery failed separately.** Two different problems, two different fixes.

---

### 🔷 Phase 4 — GPO 2: Security Lockdown Policy

This is where things get interesting from a real enterprise perspective. The `security_lockdown_policy` GPO demonstrates **user-targeted security restrictions** — policies that apply to a specific user (`hruser1`) rather than all domain users.

**Policies configured inside `security_lockdown_policy`:**

| Policy | Setting | Effect |
|--------|---------|--------|
| Prohibit access to Control Panel and PC Settings | Enabled | Removes Control Panel from Start menu and File Explorer |
| Prevent access to the command prompt | Enabled | Blocks `cmd.exe`; batch files still allowed |

---

**Screenshot: Control Panel restriction — before enabling**

![GPO Control Panel Before](screenshots/gpo-controlpanel-before.png)

---

**Screenshot: Control Panel restriction — enabled**

![GPO Control Panel Enabled](screenshots/gpo-controlpanel-enabled.png)

---

**Screenshot: CMD restriction — enabled, scoped to `hruser1`**

![GPO CMD Block Enabled](screenshots/gpo-cmd-block-enabled.png)

---

**Screenshot: GPMC — `security_lockdown_policy` scoped to `hruser1@corp.corpnet.com`**

![GPO Lockdown Scope](screenshots/gpo-lockdown-scope.png)

> Security Filtering set to `hruser1` only — this is **targeted GPO application**, not a blanket domain policy. Other users are unaffected.

---

**Screenshot: Full policy summary — Settings tab**

![GPO Lockdown Summary](screenshots/gpo-lockdown-summary.png)

> Complete view of `security_lockdown_policy`: Control Panel blocked + CMD blocked, both under User Configuration → Administrative Templates

---

**Proof the policies applied on the client:**

CMD blocked by policy:

![GPO CMD Disabled on Client](screenshots/gpo-cmd-disabled.png)

> *"The command prompt has been disabled by your administrator"*

Control Panel blocked:

![GPO Restrictions Error](screenshots/gpo-restrictions-error.png)

> *"This operation has been cancelled due to restrictions in effect on this computer. Please contact your system administrator."*

---

### 🔷 Phase 5 — GPO 3: Mapped Network Drive

The third GPO automates something IT departments do constantly: mapping a shared network drive for users on login.

Using **Group Policy Preferences → Drive Maps**, I configured:

| Setting | Value |
|---------|-------|
| Action | Create |
| Drive Letter | `H:` |
| Path | `\\10.0.0.10\CompanyData` |

---

**Screenshot: Drive Maps policy in Group Policy Management Editor**

![GPO Drive Map Policy](screenshots/gpo-drive-map-policy.png)

> `Mapped_drive_policy` — maps `H:` to `\\10.0.0.10\CompanyData` automatically for domain users

---

**Proof on the client — drive appearing in File Explorer:**

![SMB Share Mapped S Drive](screenshots/smb-share-mapped.png)

> `CompanyData (\\10.0.0.10) (S:)` visible in Network Locations — 3.87 GB free of 32.3 GB

![SMB Drive Mapped H](screenshots/smb-drive-mapped-h.png)

> `CompanyData (\\10.0.0.10) (H:)` — the GPO-mapped drive letter, automatically connected on login

---

## 🔧 Troubleshooting Log

### 1. 🔴 Domain Join Fails — DNS Not Resolving
**Symptom:** "The domain could not be found" when trying to join  
**Root Cause:** Client DNS was pointing to a public server (e.g. 8.8.8.8), not the DC  
**Fix:** Set client DNS to `10.0.0.10` (DC IP) before attempting domain join  
**Lesson:** Active Directory domain resolution depends entirely on DC-hosted DNS

---

### 2. 🔴 Wallpaper GPO Applies But Image Not Showing
**Symptom:** White rectangle renders on desktop — GPO reached client but wallpaper is blank  
**Root Cause:** Wallpaper image file was missing from `\\10.0.0.10\CompanyData\Wallpaper`  
**Diagnosis:** The white rectangle proves the policy applied — the *file delivery* is the separate failure  
**Fix:** Place a valid `.jpg` or `.bmp` file at the exact UNC path specified in the GPO  
**Lesson:** Always verify the share path contains the actual file, not just that the path is correctly typed

---

### 3. 🔴 SMB Share Access Denied for Domain Users
**Symptom:** Users can't access `\\10.0.0.10\CompanyData` — permission errors  
**Root Cause:** Only **Share Permissions** were set; **NTFS Permissions** were missing  
**Fix:** Set Read access for `Authenticated Users` (or the specific group) at *both* layers:  
- Share Permissions tab → Read  
- Security tab (NTFS) → Read & Execute  
**Lesson:** Windows network shares have two independent permission layers. Both must allow access.

---

### 4. 🔴 GPO Not Applying to Client
**Symptom:** Policy configured on DC but not taking effect on client  
**Fix:** Run `gpupdate /force` on client, then verify with `gpresult /r`  
**Lesson:** Group Policy has a refresh interval — force it during testing to avoid confusion

---

## 💼 Skills Demonstrated

- Active Directory Domain Services (AD DS) setup and management
- DNS configuration integrated with AD
- Delegation of Control — principle of least privilege in practice
- Group Policy Object (GPO) creation, linking, and scoping
- Security Filtering — targeting GPOs to specific users
- SMB file share setup with dual-layer permission model (Share + NTFS)
- Group Policy Preferences — Drive Maps automation
- Windows client domain-join process end-to-end
- Systematic troubleshooting of GPO and permission issues
- Cross-platform monitoring via Zabbix Agent 2

---

## 🔗 Related

- 📡 [Phase 1 — Zabbix Monitoring Lab](#) — Ubuntu 24.04 + LAMP + Zabbix 7.0 LTS
- 🔒 Coming next: Security hardening GPOs + Zabbix alerting

---

*Built hands-on in VirtualBox. Every error was real. Every fix was earned.*
