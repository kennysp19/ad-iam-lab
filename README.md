# Active Directory IAM Lab: Role-Based Access Control

**Status:** In progress. Role-based access and permission testing are complete. Audit logging is next.

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

## Lessons Learned

- Permissions belong on access groups, not users. Changing someone's role is one membership change.
- Modify already includes Read, so a Read group is only useful for a role that should *not* have Modify. That is why the design uses separate Analyst and Auditor roles.
- Share permissions and NTFS permissions combine, and the stricter one wins. Keeping the share open and restricting with NTFS is a common pattern.
- Turning off inheritance on a folder does not remove entries from its parent. Check both.

## Next Steps

- [ ] Enable advanced audit policy and capture events: failed logons (4625), account lockouts (4740), group membership changes (4728/4732), file access failures
- [ ] Delegate password reset on the user OUs to `ACC-HelpdeskResetPW`
- [ ] Automate joiner/mover/leaver processes in PowerShell
- [ ] Sync the domain to Microsoft Entra ID with Entra Connect and add Conditional Access and MFA
- [ ] Write KQL queries for identity-based detections against Entra sign-in logs
