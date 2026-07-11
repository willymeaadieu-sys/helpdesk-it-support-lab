# Active Directory Lab — Scenarios

Part of the [Help Desk & IT Support Lab](../README.md) portfolio project.

This section documents simulated Tier 1 support tickets run against the [AD Lab environment](README.md) built in the setup phase. Each scenario follows a realistic ticket-to-resolution flow, including the troubleshooting detours that came up along the way — since diagnosing unexpected behavior is as much a part of the job as the fix itself.

## Scenario 1: Password Reset

**Ticket:** "Sarah Chen forgot her password and can't log in."

**Resolution:**
- Reset Sarah's password via Active Directory Users and Computers
- Enabled "User must change password at next logon" so she sets her own password on first login rather than continuing to use one IT knows
- Left "Unlock the user's account" checked as standard practice, since forgotten passwords and lockouts often go hand in hand

<img width="1130" height="782" alt="Password reset confirmation" src="https://github.com/user-attachments/assets/b336165d-803f-4df6-a547-d3949e298d61" />


## Scenario 2: Account Lockout

**Ticket:** "Mike Johnson is locked out after too many failed login attempts."

**Setup — configuring the lockout policy:**
A fresh AD forest has no account lockout policy by default, so accounts never actually lock out no matter how many bad passwords are entered. Configured a domain-wide policy via Group Policy Management before this scenario could be simulated:
- Account lockout threshold: 5 invalid attempts
- Account lockout duration: 30 minutes
- Reset account lockout counter after: 30 minutes

<img width="1444" height="756" alt="Account lockout" src="https://github.com/user-attachments/assets/5be95a16-817f-41cd-9f31-affb07ebd952" />

**Reproducing the lockout:**
Used `runas /user:helpdesk\mjohnson cmd` with an incorrect password repeatedly to trigger the lockout. Early attempts were mistyped (`helpdask` instead of `helpdesk`), which meant they weren't hitting the real domain account and the lockout counter wasn't incrementing — a good reminder that a "fix isn't working" ticket is sometimes actually a data-entry issue, not a system issue. Once corrected, the failed attempts progressed cleanly through AD's own error codes:

- `1326` — wrong password (expected failed attempts)
- `1909` — account locked out after the 5th failed attempt
- `1907` — password must be changed before signing in (seen later, after reset — expected behavior, not an error, since `runas` can't complete an interactive forced password change)

<img width="1442" height="728" alt="Lockout error progression" src="https://github.com/user-attachments/assets/7e9e99b4-c0bf-4424-8a7d-d2a062530d38" />

**Resolution:**
- Unlocked the account via Active Directory Users and Computers → Properties → Account tab
- Reset the password with "must change at next logon" enabled

## Scenario 3: Onboarding

**Ticket:** "New hire starting Monday: David Kim, joining the IT department. Please provision his account."

**Resolution:**
- Created David Kim's account in the IT organizational unit with a temporary password and "user must change password at next logon" enabled
- Added him to the IT-Staff security group
- Set department/title details on the Organization tab and added a description for quick identification (`IT Support - New Hire`)
- Verified group membership by confirming IT-Staff now showed 3 members (John, Sarah, David)

<img width="1128" height="782" alt="David Kim account details 3" src="https://github.com/user-attachments/assets/ff84175b-ccff-43d5-bec4-78d721670b3a" />
<img width="1124" height="782" alt="David Kim account details 2" src="https://github.com/user-attachments/assets/2f6bb99c-6a5e-484f-b957-a6d9244f0bd0" />
<img width="1132" height="796" alt="David Kim account details 1" src="https://github.com/user-attachments/assets/956cb630-e29f-4eac-94b8-47a525594994" />


## Scenario 4: Offboarding

**Ticket:** "Sarah Chen's last day was Friday. Please offboard her account per company policy."

**Resolution:**
Followed the standard security practice of **disabling rather than deleting** the account — this preserves the account's SID, group history, and audit trail in case HR or IT needs to review her prior access later, while immediately blocking any further logon.

- Disabled Sarah's account
- Removed her from the IT-Staff security group
- Updated her description field with the disable date and last working day for audit purposes
- Moved her account into a dedicated "Former Employees" OU to keep offboarded accounts separated from active staff for easier review and cleanup

<img width="1128" height="784" alt="SC Offboard 1" src="https://github.com/user-attachments/assets/a73ce909-fc91-42d7-b19e-be8fa1aea1f7" />
<img width="1132" height="796" alt="SC Offboard 2" src="https://github.com/user-attachments/assets/23ed0f9e-68f7-4549-a0a6-32114e970faf" />
<img width="1120" height="788" alt="SC Offboard 3" src="https://github.com/user-attachments/assets/bd37a7eb-ad77-4155-9a59-2332eea9607a" />

<img width="1122" height="790" alt="Former Employees OU" src="https://github.com/user-attachments/assets/f2293eb3-658f-4539-adcd-6bd0a9ee0a07" />


## Scenario 5: Permissions Troubleshooting

**Ticket:** "IT needs a shared drive for internal documentation that only IT staff can access — Sales and HR should not be able to see it."

**Setup:**
- Created `C:\Shares\IT-Docs` and shared it, replacing the default "Everyone" share permission with **IT-Staff: Full Control**
- Set NTFS permissions on the Security tab to grant **IT-Staff: Modify**
- Found that the built-in **Users** group (which includes all domain accounts, including Sales and HR) still had inherited access — left as-is, this would have completely undermined the restriction
- Windows initially blocked removing that inherited permission directly; resolved by disabling inheritance ("Convert inherited permissions to explicit"), which preserved the existing SYSTEM/Administrators access while making the Users entry removable
- Removed the Users group, leaving only IT-Staff, Administrators, and SYSTEM with access

<img width="1170" height="894" alt="Permissions before fix - Users group still has access 1" src="https://github.com/user-attachments/assets/2cd22d9c-9c8c-4ecd-8906-c37b41ad3441" />
<img width="1186" height="928" alt="Permissions before fix - Users group still has access 2" src="https://github.com/user-attachments/assets/f7729a83-b907-4fc3-a025-cc90f52df35d" />

<img width="1186" height="958" alt="Permissions after fix - Users group removed" src="https://github.com/user-attachments/assets/606f6a58-1a28-47d5-bc5b-b300f0418f92" />


**Testing the restriction:**
Attempted to verify the block using `runas` as Mike Johnson (Sales), but the first attempt returned error `1385` — domain member accounts aren't granted interactive logon rights on a domain controller by default, so `runas` couldn't launch a session for him directly. Added the `/netonly` flag to bypass the local logon check:
```
runas /netonly /user:helpdesk\mjohnson cmd
```
The first access test against the local path (`C:\Shares\IT-Docs`) succeeded — but that was a false positive. `/netonly` only swaps credentials for *network* resource requests, so a local path check was still running under the original Administrator session, not Mike's. Re-tested against the folder's UNC network path instead:
```
dir \\DC01\IT-Docs
```
This correctly authenticated as `helpdesk\mjohnson` and returned **"Access is denied."** — confirming the share and NTFS permissions were both working as intended.

<img width="1466" height="754" alt="Access denied confirmation" src="https://github.com/user-attachments/assets/5be2a5c8-149c-4ec4-82d7-f0479e3a9452" />


## Status

 Password reset
 Account lockout (policy configuration + trigger + resolution)
 Onboarding
 Offboarding
Permissions troubleshooting (share + NTFS permissions, verified via network path testing)








