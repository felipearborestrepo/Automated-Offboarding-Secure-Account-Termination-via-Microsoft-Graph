# Module Description
Built a PowerShell script that reads terminated employee data from a CSV file and automatically executes a complete offboarding sequence — disabling the account, revoking all active sessions, removing all group memberships, removing licenses, and exporting a timestamped audit log. Designed to eliminate the security risk of manual offboarding where steps get missed and ex-employees retain access.
**Step 1 — Connected to Microsoft Graph**
- Connected using three scopes
- User.ReadWrite.All — permission to disable accounts and remove licenses
- Group.ReadWrite.All — permission to remove users from groups
- Directory.ReadWrite.All — permission to modify directory objects
- Revoke-MgUserSignInSession requires User.ReadWrite.All
```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All","Group.ReadWrite.All","Directory.ReadWrite.All" -NoWelcome
```
**Step 2 — Created the terminated employees CSV**
- CSV acts as input from HR — simulates how HR would submit termination requests
- Same pattern as onboarding CSV — HR owns the data, IT owns the automation
- Each row represents one terminated employee with all properties needed
```powershell
UserPrincipalName,TerminationDate,ManagerUPN,ConvertMailbox
john.smith@domain.com,2026-08-06,manager@domain.com,Yes
maria.garcia@domain.com,2026-08-06,manager@domain.com,Yes
```
**Step 3 — Read the CSV and loop through terminated employees**
- Used Import-Csv to read terminated employee data
- Each row becomes a PowerShell object with named properties
- Looped through every terminated employee one at a time
- Wrapped entire process in try/catch — one failure doesn't stop the rest
```powershell
$results = [System.Collections.Generic.List[PSCustomObject]]::new()
Write-Host "`nStarting offboarding process..." -ForegroundColor Cyan
foreach ($term in $terminated) {
```
**Step 4 — Retrieved the user from Graph**
- Pulled the full user object from Graph using their UPN
- Retrieved Id — needed for all subsequent operations
- Retrieved AssignedLicenses — needed for license removal
- If user not found the catch block logs the failure and moves to next employee
```powershell
$user = Get-MgUser -UserId $term.UserPrincipalName -Property "Id,DisplayName,UserPrincipalName,AccountEnabled,AssignedLicenses"
```
**Step 5 — Disabled the account immediately**
- First action always — blocks all new sign-ins immediately
- User cannot authenticate to any Microsoft service after this runs
- AccountEnabled:$false sets the account to disabled state in Entra ID
```powershell
Update-MgUser -UserId $user.Id -AccountEnabled:$false
```
**Step 6 — Revoked all active sessions**
- Invalidates all existing refresh tokens and session cookies
- Anyone currently logged in as this user gets signed out within minutes
- Critical security step — disabling the account stops new logins but existing sessions can persist without this
- Covers all apps using Microsoft authentication — M365, Teams, SharePoint, any SAML/OIDC app
```powershell
Revoke-MgUserSignInSession -UserId $user.Id
```
**Step 7 — Removed all group memberships**
- Retrieved all group memberships using Get-MgUserMemberOf
- Filtered to only #microsoft.graph.group type — skips dynamic groups and role assignments
- Dynamic groups excluded because membership is controlled by rules not manual assignment — account being disabled removes them automatically
- Used try/catch per group — if one group removal fails the script continues to the next
- Tracked count of groups removed for audit log
```powershell
$groups = Get-MgUserMemberOf -UserId $user.Id
$groupsRemoved = 0

foreach ($group in $groups) {

$groupId = $group.Id
$groupName = $group.AdditionalProperties["displayName"]
$groupType = $group.AdditionalProperties["@data.type"]

if ($groupType -eq "#microsoft.graph.graph") {

try {
Remove-MgGroupMemberByRef -GroupId $groupId -DirectoryObjectId $user.Id
Write-Host "Removed from: $groupName" -ForegroundColor Green
$groupsRemoved++
} catch {
Write-Host "Could not remove from: $groupName — $($_.Exception.Message)" -ForegroundColor Yellow
}
}
}
```
**Step 8 — Removed all licenses**
- Got list of all assigned license SKU IDs from the user object
- Used Set-MgUserLicense with empty AddLicenses array and full RemoveLicenses list
- Frees up licenses immediately for reassignment to new hires
- If no licenses assigned logs a warning and continues
```powershell
$licensesToRemove = $user.AssignedLicenses | ForEach-Object { $_SkuId }

if ($licensestoRemove.Count -gt 0) {
Set-MgUserLicense -UserId $user.Id -AddLicenses @() -RemoveLicenses $licensesToRemove
Write-Host "Licenses removed: $($licensesToRemove.Count)" -ForegroundColor Green
} else {
Write-Host "No licenses assigned" -ForegroundColor Yellow
}
```
**Step 9 — Built timestamped audit log entry**
- Added a custom object to results for every processed employee
- Included exact timestamp of when offboarding ran
- Logged every action taken — account disabled, sessions revoked, groups removed, licenses removed
- Failed employees logged with error message for manual follow-up
```powershell
$results.Add([PSCustomObject]@{
    DisplayName       = $user.DisplayName
    UserPrincipalName = $term.UserPrincipalName
    TerminationDate   = $term.TerminationDate
    AccountDisabled   = $true
    SessionsRevoked   = $true
    GroupsRemoved     = $groupsRemoved
    LicensesRemoved   = $licensesToRemove.Count
    Status            = "Success"
    Timestamp         = (Get-Date -Format "yyyy-MM-dd HH:mm:ss")
    Error             = ""
})
```
**Step 10 — Exported offboarding audit report**
- Exported complete audit log to CSV on Desktop
- Report includes every terminated employee — successful and failed
- Timestamp column shows exactly when each offboarding ran
- Serves as compliance evidence — who was offboarded, when, what was removed
```powershell
$results | Export-Csv -Path "$HOME/Desktop/Offboarding_Report.csv" -NoTypeInformation
Write-Host "`nCSV exported to Desktop" -ForegroundColor Green
```
**Step 11 — Printed summary**
- Printed clean summary showing total processed, successful, and failed
- Failed employees highlighted in red for immediate attention
```powershell
$success = ($results | Where-Object Status -eq "Success").Count
$failed = ($results | Where-Object Status -eq "Failed").Count

Write-Host "`n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor DarkGray
Write-Host "  OFFBOARDING SUMMARY" -ForegroundColor White
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor DarkGray
Write-Host "  Total processed : $($results.Count)" -ForegroundColor White
Write-Host "  Successful      : $success" -ForegroundColor Green
Write-Host "  Failed          : $failed" -ForegroundColor Red
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`n" -ForegroundColor DarkGray
```
# PowerShell results printed

<img width="619" height="439" alt="Screenshot 2026-08-08 at 11 44 00" src="https://github.com/user-attachments/assets/932ad1c7-251d-4473-a4e6-19225f847e47" />

# Offboarding_Report.csv Results

<img width="1203" height="79" alt="Screenshot 2026-08-08 at 11 51 10" src="https://github.com/user-attachments/assets/76fda86e-497a-4ddd-81d0-f9835e3156ce" />

Retrieved AssignedLicenses — needed for license removal
If user not found the catch block logs the failure and moves to next employee
