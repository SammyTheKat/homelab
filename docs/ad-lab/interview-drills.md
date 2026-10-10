# AD interview drills

> The post-build range. Each drill is a scenario recruiters actually ask
> about — run it, break it, fix it, note what you learned. Do them in
> order: 3 creates the test users that 4 retires, and 2 and 5 share the
> same GPO tooling, so the second one in each pair goes faster.

Assumptions: the DC is promoted (`ad.lab` used below — this matches
Friday's build: forest `ad.lab`, DC `dc01`, isolated 10.20.30.0/24),
at least one Windows 11 client is domain-joined, and DNS on the
clients points at the DC.
Nothing here touches the production LAN.

---

## 1. Locked-out user

**The ticket:** "Sarah in shipping can't log in — says the password
stopped working. Third call this week."

**Setup:** create a test user, then simulate the fat-finger lockout by
logging in wrong a few times from the client (or skip the theater and
lock the account directly with the policy below set low).

**Walkthrough:**
1. Confirm it's a lockout, not a bad password: in ADUC, find the user →
   Account tab shows "Unlock account". Or in PowerShell:
   `Get-ADUser sarah -Properties LockedOut, lockoutTime | Select SamAccountName, LockedOut, lockoutTime`
2. Find every locked account in the domain at once (the fastest way to
   survey the whole domain): `Search-ADAccount -LockedOut | Select Name, SamAccountName`
3. Unlock it: `Unlock-ADAccount -Identity sarah` — then verify
   `LockedOut` is `$false`.
4. Check the policy that fired: Default Domain Policy → Computer
   Configuration → Policies → Windows Settings → Security Settings →
   Account Policies → Account Lockout Policy. Note the three values that
   matter: threshold (bad attempts), duration (how long it stays locked),
   and reset counter (the window).

**Break it on purpose:** set the threshold to 3, lock the test user,
watch the Security event log on the DC (event 4740) record the lockout
with the caller computer name. That event ID is the difference between
guessing and knowing.

**Takeaway:** "I don't just unlock — I check 4740 to see
where the bad attempts came from, because a lockout that keeps
recurring is usually a stale credential on a phone or a mapped drive,
not a user problem."

*Ran 2026-10-09: set the threshold to 3, locked `sarah` with three bad
logins from CLIENT01, confirmed `LockedOut=True` via `Get-ADUser`,
swept the domain with `Search-ADAccount -LockedOut` (only sarah),
event 4740 named CLIENT01 as the caller, unlocked with
`Unlock-ADAccount` and verified `LockedOut=False`.*

---

## 2. GPO not applying

**The ticket:** "We pushed a wallpaper GPO to the Sales OU and half the
team doesn't have it."

**Setup:** link a test GPO (wallpaper, or a mapped drive from drill 5)
to one OU containing one test client. Log in as a domain user on the
client.

**Walkthrough — the ladder, in order:**
1. On the client: `gpresult /r` — is the GPO listed under Applied, or
   under Denied/Filtered Out? The answer tells you which rung to climb.
2. Force it and re-test: `gpupdate /force`, log off and back on
   (user-side settings need a logon, computer-side need a reboot).
3. Check linkage: GPMC → is the GPO actually linked to the right OU? Is
   the link enabled? Is Block Inheritance set on the OU?
4. Check security filtering — the classic: if someone removed
   "Authenticated Users" and added a group, the computer also needs read
   access to the GPO or nothing applies. The fix most people miss.
5. Check WMI filters: a filter targeting the wrong OS build silently
   excludes machines.
6. Replication: with one DC this is a non-issue in the lab, but say it
   anyway — in production, `repadmin /syncall` or waiting out SYSVOL
   replication explains the "works on some machines" pattern.

**Break it on purpose:** remove Authenticated Users from security
filtering, add only the test user group, watch `gpresult /r` move the
GPO to Denied. Put it back. You now have a story for "the most common
GPO mistake I've actually reproduced."

**Takeaway:** "gpresult first, always — it tells you whether
the GPO was denied, filtered, or never in scope, and each answer is a
different fix."

*Ran 2026-10-09: created a `Sales Wallpaper` GPO (Desktop Wallpaper →
`img0.jpg`), linked to the Sales OU — applied cleanly, wallpaper
picker greyed out for sarah. Broke it by removing Authenticated Users
from security filtering and adding only sarah: `gpresult /r` showed
"Not Applied (Unknown Reason)" under filtered-out GPOs. Fixed by
granting Domain Computers Read (not Apply) on the Delegation tab —
back under Applied after `gpupdate /force` and a fresh logon.*

---

## 3. New-hire provisioning

**The ticket:** "New hire starts Monday — account, groups, home folder,
temp password, must change it at first logon."

**Walkthrough — manual first, then scripted:**
1. Manual in ADUC: create the user in the right OU, set a temp password,
   check "User must change password at next logon", add to the
   department group, set the home folder path
   (`\\dc01\home$\%username%`).
2. Then do the whole thing in one PowerShell pass:
   ```powershell
   $pw = ConvertTo-SecureString "TempPass123!" -AsPlainText -Force
   New-ADUser -Name "Test Hire" -SamAccountName "thire" `
     -UserPrincipalName "thire@ad.lab" `
     -Path "OU=Sales,DC=ad,DC=lab" `
     -AccountPassword $pw -Enabled $true `
     -ChangePasswordAtLogon $true `
     -HomeDirectory "\\dc01\home$\thire" -HomeDrive "H:"
   Add-ADGroupMember -Identity "Sales" -Members "thire"
   ```
3. Scale it: put five hires in a CSV and loop `Import-Csv |
   ForEach-Object { New-ADUser ... }`. Compare the time against the
   manual run — that's the story.

**Takeaway:** "I provision manually to understand every
field, then script it, because the fifth new hire of the week deserves
the same setup as the first."

---

## 4. Deprovisioning

**The ticket:** "Mike left Friday. Kill his access, keep the data."

**Walkthrough:**
1. Disable, don't delete — you need the SID history and the audit
   trail: `Disable-ADAccount -Identity mike`
2. Strip group memberships so nothing lingers:
   `Get-ADPrincipalGroupMembership mike | Where-Object { $_.Name -ne "Domain Users" } | ForEach-Object { Remove-ADGroupMember -Identity $_ -Members mike -Confirm:$false }`
3. Move to a Disabled Users OU: `Move-ADObject` (or drag in ADUC).
4. Stamp the account so the next admin knows the story:
   `Set-ADUser mike -Description "Disabled 2026-10-11 — departed, data retained per policy"`
5. Note the handoffs a real shop does: manager gets the files, IT
   reclaims licenses, the account sits disabled 30–90 days before
   deletion.

**Break it on purpose:** try logging in as the disabled test user from
the client. Watch it fail. Try accessing a share with the stripped
groups. The failure modes are the lesson.

**Takeaway:** "Disable first, strip groups, park in a
disabled OU, stamp the description with the date. Deletion is a
retention-policy decision, not a Friday-afternoon click."

---

## 5. Drive mappings

**The ticket:** "Sales needs the S: drive. Only Sales. It should follow
the user to any machine."

**Walkthrough:**
1. Create a "Sales" security group and put the test user in it.
2. New GPO linked to the OU holding the *users* (this is a user-side
   preference): User Configuration → Preferences → Windows Settings →
   Drive Maps → New Mapped Drive.
3. Action: **Update** (not Replace — Replace disconnects and remaps every
   refresh, which users notice). Location: `\\dc01\sales`. Label it.
4. Item-level targeting: Targeting → New Item → Security Group → the
   Sales group. This is the whole trick — the mapping applies by group
   membership, not by machine.
5. `gpupdate /force`, log off/on, verify the drive appears for the
   Sales user and *not* for a user outside the group.
6. Confirm with `gpresult /r` that the GPO applied to the user scope.

**Break it on purpose:** link it to a computer OU instead and watch
nothing happen for the user — user preferences need user scope. Then
move it and watch it work. Preferences-vs-policies is a distinction
most candidates fumble; you'll have the demo.

**Takeaway:** "User Configuration, drive-map preference,
item-level targeting on the security group. Follows the user, not the
machine — and Update, not Replace, so it doesn't flap on every
refresh."

*Ran 2026-10-10: created a `Sales` security group (sarah, mike);
shared `C:\Shares\Sales` as `\\dc01\sales` with Sales-group share and
NTFS read; new `Sales Drive Mapping` GPO linked to the Sales OU with a
Drive Maps preference (Update, S:, item-level targeting on the Sales
group). sarah got `Sales (S:)` after gpupdate + fresh logon; alex (IT,
not in the group) got nothing — targeting proven both directions.*

---

## 6. PowerShell AD queries

**The ticket:** "Audit wants a list of accounts that haven't logged in
for 90 days, and all disabled accounts still sitting in enabled OUs."

**Walkthrough:**
1. Stale accounts: `Get-ADUser -Filter * -Properties LastLogonDate |
   Where-Object { $_.LastLogonDate -lt (Get-Date).AddDays(-90) } |
   Select Name, SamAccountName, LastLogonDate | Export-Csv stale.csv`
   (Caveat worth knowing: on a multi-DC domain LastLogonDate can lag
   replication by up to 14 days — single DC in the lab, so it's exact
   here.)
2. Disabled accounts outside the disabled OU:
   `Get-ADUser -Filter { Enabled -eq $false } | Select Name, DistinguishedName`
3. Bulk group work: `Get-ADUser -Filter { Department -eq "Sales" } |
   ForEach-Object { Add-ADGroupMember -Identity "Sales" -Members $_ }`
4. Group membership audit: `Get-ADGroupMember "Domain Admins" |
   Select Name, objectClass` — know who's in the privileged groups.

**Takeaway:** "Anything I do twice in ADUC, I script the
third time. The audit queries are reports first — Export-Csv — because
nobody wants a screenshot of a console."

---

## Weekend order of operations

- **Friday:** build the lab, join both clients, create the OUs and a
  handful of test users — plus drills 1 (locked-out user) and 2 (GPO
  not applying), since the afternoon had bandwidth.
- **Saturday:** drill 5 (drive mappings) — ran 2026-10-10, reuses
  drill 2's GPO tooling.
- **Sunday:** drills 3 (new-hire provisioning, manual then scripted),
  4 (deprovisioning), and 6 (PowerShell audit queries). End the weekend
  with the CSV-provisioned users and the stale-account report saved as
  artifacts for the portfolio write-up.
