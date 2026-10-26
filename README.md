# Active Directory IAM Lab: Role-Based Access Control

**Status:** In progress. Role-based access, permission testing, and audit logging are complete. Next up: delegation and lifecycle automation.

This is Lab 2 of my identity and access management (IAM) series. [Lab 1](https://github.com/kennysp19/Active-Directory-Home-Lab) covered building the domain: Windows Server 2022 domain controller, DHCP, NAT, and a domain-joined Windows 10 client. This lab builds on that environment to practice how access is designed, assigned, and verified in a real directory.

## Goal

Design and test a role-based access control (RBAC) model in Active Directory where:

- Users are placed in **role groups** that describe their job function.
- Role groups are nested into **access groups** that carry the actual permissions.
- Folder permissions are assigned **only to access groups**, never directly to users or role groups.
- Different roles can have different permission levels (Modify vs Read) on the same resource.

The point of this design is that when someone changes jobs, an admin changes one group membership instead of editing permissions on every folder.

## Environment

| Component | Details |
|---|---|
| Virtualization | VirtualBox |
| Domain controller | Windows Server 2022, domain `mydomain.com` |
| Client | Windows 10, domain-joined |
| Network | Internal network, DHCP scope 192.168.1.100-200, NAT through the DC |
| Tools | Active Directory Users and Computers, PowerShell (ActiveDirectory module), NTFS and SMB share permissions |

## Directory Design

### OU structure

```
mydomain.com
├── Disabled Accounts
├── Finance          (Finance user accounts)
├── HR               (HR user accounts)
├── IT               (IT user accounts)
├── Workstations
└── Groups
    ├── Roles        (who someone is)
    └── Access       (what they can touch)
```

### Groups

| Type | Group | Purpose |
|---|---|---|
| Role | `Role-HR-Analyst` | HR staff who edit HR files |
| Role | `Role-HR-Auditor` | HR reviewers with read-only access |
| Role | `Role-Finance-Analyst` | Finance staff who edit Finance files |
| Role | `Role-Finance-Auditor` | Finance reviewers with read-only access |
| Role | `Role-IT-Helpdesk` | Helpdesk staff |
| Access | `ACC-HRShare-Modify` | Modify on the HR share |
| Access | `ACC-HRShare-Read` | Read & execute on the HR share |
| Access | `ACC-FinanceShare-Modify` | Modify on the Finance share |
| Access | `ACC-FinanceShare-Read` | Read & execute on the Finance share |
| Access | `ACC-HelpdeskResetPW` | Reserved for password reset delegation (not yet configured) |

All groups are Global security groups. A stricter AGDLP design would use domain local groups on the access side; Global groups are a simplification that works for a single-domain lab.

![Roles OU](screenshots/roles-ou.png)
![Access OU](screenshots/access-ou.png)
![Group creation commands](screenshots/group-commands.png)

### Nesting (user → role → access)

```
druth  →  Role-HR-Analyst       →  ACC-HRShare-Modify
jliem  →  Role-HR-Auditor       →  ACC-HRShare-Read
smuney →  Role-Finance-Analyst  →  ACC-FinanceShare-Modify
sjones →  Role-Finance-Auditor  →  ACC-FinanceShare-Read
```

![Users added to role groups](screenshots/add-users-commands.png)
![Role-HR-Analyst members](screenshots/role-hr-analyst-members.png)
![Role-HR-Auditor members](screenshots/role-hr-auditor-members.png)

![Role-HR-Auditor nested in ACC-HRShare-Read](screenshots/nesting-hr-auditor.png)

*jliem is a member of Role-HR-Auditor, which is a member of ACC-HRShare-Read.*

![Nesting verification in PowerShell](screenshots/nesting-powershell.png)

## Shares and Permissions

Two folders on the domain controller, each containing a `note.txt` used to test access:

- `C:\shares\HR` shared as `\\DC\HR`
- `C:\shares\Finance` shared as `\\DC\Finance`

**Share permissions:** Everyone, Full Control. NTFS permissions do the real restricting, and the effective permission is the more restrictive of the two.

**NTFS permissions (inheritance disabled and converted to explicit):**

| Principal | HR folder | Finance folder |
|---|---|---|
| SYSTEM | Full control | Full control |
| Administrators | Full control | Full control |
| CREATOR OWNER | Full control (subfolders and files) | Full control (subfolders and files) |
| `ACC-HRShare-Modify` | Modify | none |
| `ACC-HRShare-Read` | Read & execute | none |
| `ACC-FinanceShare-Modify` | none | Modify |
| `ACC-FinanceShare-Read` | none | Read & execute |

Users and Everyone were removed from the NTFS entries so that membership in an ACC group is the only way to gain access.

![HR folder security settings](screenshots/ntfs-hr.png)
![Finance folder security settings](screenshots/ntfs-finance.png)

## Test Results

Tests were run from the Windows 10 client, signed in as each user, opening files by UNC path (`notepad \\DC\HR\note.txt`).

| User | Role | Own share | Other department's share |
|---|---|---|---|
| `smuney` | Finance Analyst | Opened, edited, and saved | Denied |
| `sjones` | Finance Auditor | Opened. Overwrite, Save As, and delete all denied | Denied |
| `druth` | HR Analyst | Opened, edited, and saved | Denied |
| `jliem` | HR Auditor | Opened. Overwrite and delete denied | Denied |

All results matched the design.

### Analysts can modify

![smuney edits Finance note](screenshots/test-smuney-finance-edit.png)
![druth edits HR note](screenshots/test-druth-hr-edit.png)

### Auditors are read-only

![sjones Save As denied](screenshots/test-sjones-saveas-denied.png)
![sjones overwrite denied](screenshots/test-sjones-overwrite-denied.png)
![sjones delete denied](screenshots/test-sjones-delete-denied.png)
![jliem overwrite denied](screenshots/test-jliem-overwrite-denied.png)
![jliem delete denied](screenshots/test-jliem-delete-denied.png)

*Notepad reports "You do not have permission to open this file" when a save to a read-only location fails. The file had already opened; the denial is on the write.*

### Cross-department access is denied

![smuney denied on HR](screenshots/test-smuney-hr-denied.png)
![druth denied on Finance](screenshots/test-druth-finance-denied.png)
![jliem denied on Finance](screenshots/test-jliem-finance-denied.png)
![sjones denied on HR](screenshots/test-sjones-hr-denied.png)

## Monitoring and Audit Logging

Once the access model worked, I wanted to see what failure looks like in the logs. I turned on advanced auditing on the domain controller, set an account lockout policy, and used a throwaway account (`tuser`, display name "test user") to generate failed logons, a lockout, an unlock, a successful logon, and group membership changes. Times below are the domain controller's clock.

### Audit and lockout settings

Advanced audit policy (Default Domain Controllers Policy), Success and Failure: Credential Validation, Kerberos Authentication Service, User Account Management, Security Group Management, Logon, and Account Lockout.

Lockout policy (Default Domain Policy): lock the account after 5 bad attempts, for 5 minutes, with a 5-minute reset window.

A policy that is set is not always a policy that is applied, so I checked the live result with `auditpol`:

```powershell
gpupdate /force
auditpol /get /subcategory:"Credential Validation","Kerberos Authentication Service","User Account Management","Security Group Management","Logon","Account Lockout"
```

![Audit policy: Account Logon](screenshots/gpo-audit-account-logon.png)
![Audit policy: Account Management](screenshots/gpo-audit-account-mgmt.png)
![Audit policy: Logon/Logoff](screenshots/gpo-audit-logon-logoff.png)
![auditpol showing the settings in effect](screenshots/auditpol-verify.png)
![Lockout policy from net accounts](screenshots/net-accounts.png)

### Wrong passwords until lockout (10/22)

I entered five wrong passwords for `tuser` on the Windows 10 client, then tried once more after the account locked.

| Time | Event | What it shows |
|---|---|---|
| 3:30:26 to 3:30:35 | 4771 (five events) | Kerberos pre-authentication failed. The 3:30:35 event shows Failure Code `0x18`, which means a wrong password |
| 3:30:35 | 4740 | `tuser` was locked out. The caller computer is `CLIENT01` |
| 3:30:38 | 4771 | Failure Code `0x12`: the attempt was refused because the account was already locked out |

The two codes are the useful part. `0x18` says the password was wrong, and `0x12` says the account was blocked whatever the password was. The whole sequence took about 12 seconds.

![4771 with failure code 0x18](screenshots/event-4771-0x18.png)
![4740 account lockout](screenshots/event-4740.png)
![4771 with failure code 0x12](screenshots/event-4771-0x12.png)
![Events pulled with Get-WinEvent](screenshots/eventlog-4771-4740-summary.png)
![Client refusing the sign-in](screenshots/lockout-screen.png)

### Unlock and successful sign-in (10/26)

The lockout lasts 5 minutes by policy, so it can expire on its own. An admin unlock is logged separately as event 4767, which is why I locked the account a second time and unlocked it by hand with `Unlock-ADAccount`.

After the unlock, `tuser` signed in from the client. `$env:LOGONSERVER` on the client showed `\\DC`, and the domain controller logged:

- **4768**: a Kerberos ticket (TGT) was requested for `tuser`, with Result Code `0x0` (success)
- **4624**: a successful logon for `tuser`, Logon Type 3 (network). The sign-in itself happens on the client, so the DC sees it as a network logon

The `0x12` shown in the 4768 is the ticket encryption type (AES-256). It is a different field from the `0x12` failure code in the 4771.

![Account unlocked, LockoutTime back to 0](screenshots/aduser-unlocked.png)
![4767 account unlocked](screenshots/event-4767.png)
![Client authenticated by the DC](screenshots/logonserver.png)
![4768 ticket granted](screenshots/event-4768.png)
![4624 successful logon](screenshots/event-4624.png)

One attempt did not go cleanly: `tuser` signed in on the client while the DC showed `LockedOut: True`. I suspect the client used cached credentials, since the lockout lives on the DC, but I did not confirm that. After a clean sign-out and a fresh lockout, the client refused the correct password, as expected.

### Group membership changes

I added `tuser` to `Role-HR-Auditor` and removed it again. Each change was logged with the admin account that made it:

- **4728**: member added to a security-enabled global group (`tuser` added to `Role-HR-Auditor` by Administrator)
- **4729**: member removed from the same group

Each change also produced a 4737 (group changed) event alongside it.

![4728 member added](screenshots/event-4728.png)
![4729 member removed](screenshots/event-4729.png)

### What I took away

- A burst of 4771 events against one account within seconds, followed by a 4740, is the pattern of password guessing. It is the kind of sequence an alert would key on.
- Reading the failure code matters: `0x18` is a wrong password and `0x12` is a locked account.
- Group changes are logged with who made them, which is the evidence trail an access review needs.
- The same sign-in data exists in Microsoft Entra ID, so these patterns are the starting point for KQL detections later.

## Key Commands

```powershell
# Create groups in the Roles and Access OUs
New-ADGroup -Name "Role-HR-Analyst" -GroupScope Global -GroupCategory Security -Path "OU=Roles,OU=Groups,DC=mydomain,DC=com"
New-ADGroup -Name "ACC-HRShare-Modify" -GroupScope Global -GroupCategory Security -Path "OU=Access,OU=Groups,DC=mydomain,DC=com"

# Nest role groups into access groups
Add-ADGroupMember -Identity "ACC-HRShare-Modify" -Members "Role-HR-Analyst"
Add-ADGroupMember -Identity "ACC-HRShare-Read" -Members "Role-HR-Auditor"

# Add users to role groups
Add-ADGroupMember -Identity "Role-HR-Analyst" -Members "druth"

# Verify membership
Get-ADGroupMember "ACC-HRShare-Modify" | Select Name, objectClass
```

## Troubleshooting Notes

| Problem | Cause | Fix |
|---|---|---|
| `New-ADGroup : Directory object not found` | The `-Path` had the OUs in the wrong order or used the wrong OU name | Build paths from the deepest OU up to the domain, and copy the exact value from `Get-ADOrganizationalUnit -Filter * \| Select DistinguishedName` |
| `New-ADGroup : Access is denied` | PowerShell was not elevated and the account lacked rights | Ran PowerShell as Administrator |
| Groups showed up as OUs | Created OUs by mistake instead of groups | Removed the OUs (after clearing deletion protection) and created real Security Groups |
| Finance permissions showed "Inherited from C:\shares" | ACC groups had been added to the parent folder | Disabled inheritance and converted to explicit entries on the Finance folder |
| `Add-ADGroupMembers` not recognized | Typo; the cmdlet name is singular | Used `Add-ADGroupMember`. Tab completion avoids this |
| Share unreachable and "server unavailable" errors mid-test | Windows Server evaluation license expired, causing the DC to shut down periodically | Rearmed the license with `slmgr /rearm` and restarted. Lab snapshots provide a rollback point |
| Two commands ran together as one line and PowerShell reported a positional parameter error | A multi-line paste merged the lines | Ran commands one at a time |
| `auditpol` returned "The parameter is incorrect" | A space between `/subcategory:` and the first quote | Removed the space so the value is attached to the switch |
| Lockout did not seem to apply on one sign-in | Likely cached credentials on the client (not confirmed) | Signed out fully, locked the account again, and confirmed the client refused the correct password |

## Lessons Learned

- Permissions belong on access groups, not users. Changing someone's role is one membership change.
- Modify already includes Read, so a Read group is only useful for a role that should *not* have Modify. That is why the design uses separate Analyst and Auditor roles.
- Share permissions and NTFS permissions combine, and the stricter one wins. Keeping the share open and restricting with NTFS is a common pattern.
- Turning off inheritance on a folder does not remove entries from its parent. Check both.

## Next Steps

- [x] Enable advanced audit policy and capture failed logons, a lockout, an unlock, a successful logon, and group membership changes
- [ ] Audit file access failures on the HR and Finance shares
- [ ] Delegate password reset on the user OUs to `ACC-HelpdeskResetPW`
- [ ] Automate joiner/mover/leaver processes in PowerShell
- [ ] Sync the domain to Microsoft Entra ID with Entra Connect and add Conditional Access and MFA
- [ ] Write KQL queries for identity-based detections against Entra sign-in logs
