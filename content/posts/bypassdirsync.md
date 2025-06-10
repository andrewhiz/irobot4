---
date: '2025-06-10T02:32:16-07:00'
draft: false
title: 'The Sync Gap'
---

# BypassDirSyncOverridesEnabled: Why Mobile Numbers Aren't Syncing from Active Directory to Entra ID

## The Unexpected (or expected) Sync Gap

In hybrid Microsoft Entra ID environments, administrators often encounter a puzzling scenario: mobile phone numbers populated in on-premises Active Directory are not synchronizing to Entra ID, despite other user attributes syncing correctly. Users appear in Entra ID with blank mobile phone fields, even though the `mobile` and `otherMobile` attributes are properly populated in their on-premises AD accounts.

The root cause of this synchronization gap is a tenant setting called `BypassDirSyncOverridesEnabled`. When this setting is enabled (which it often is by default in newer tenants), it prevents the mobile phone attributes from syncing from on-premises Active Directory to Entra ID, breaking the expected synchronization flow.

## Understanding the Logic Behind the Setting

Microsoft introduced `BypassDirSyncOverridesEnabled` as part of their strategy to allow cloud-based updates to certain user attributes in hybrid environments. However, this creates an unintended consequence: when enabled, the system assumes mobile phone numbers should be managed from the cloud rather than synchronized from on-premises AD.

The setting affects these specific attributes:
- **Mobile phone** (mobile attribute in AD)
- **Alternate mobile phone** (otherMobile attribute in AD)

## The Evolution of Mobile Number Management

Throughout 2023 and early 2024, Microsoft gradually introduced the capability for users and administrators to update mobile phone numbers directly in Entra ID for synchronized users. This was designed to provide flexibility without requiring changes to on-premises Active Directory. However, this feature came with the `BypassDirSyncOverridesEnabled` setting enabled by default in many tenants, effectively blocking the traditional AD-to-Entra sync for mobile attributes.

## Why Administrators Are Caught Off Guard

Many administrators remain unaware of this behavior change for several reasons:

- **Silent implementation**: Microsoft didn't widely communicate this change in synchronization behavior
- **Inconsistent defaults**: Some tenants have the setting enabled by default, others don't
- **Hidden configuration**: The setting is only accessible via PowerShell, not through admin portals
- **Selective blocking**: Only mobile phone attributes are affected, while other attributes sync normally
- **No error messages**: Directory synchronization appears to work correctly overall

## Checking the Current Status

To verify if your tenant has this setting enabled (and thus blocking mobile sync from AD), use the Microsoft Graph PowerShell module:

```powershell
# Connect to Microsoft Graph
Connect-MgGraph -Scopes "Directory.Read.All"

# Get the Directory Feature Settings template
$template = Get-MgDirectorySettingTemplate | Where-Object {$_.DisplayName -eq "Directory Feature Settings"}

# Check if settings exist for this template
$setting = Get-MgDirectorySetting | Where-Object {$_.TemplateId -eq $template.Id}

if ($setting) {
    $bypassSetting = $setting.Values | Where-Object {$_.Name -eq "BypassDirSyncOverridesEnabled"}
    Write-Host "BypassDirSyncOverridesEnabled: $($bypassSetting.Value)"
} else {
    Write-Host "Directory Feature Settings not configured - using defaults"
}
```

If the value shows `true`, mobile numbers won't sync from AD to Entra ID.

## Generating a Gap Analysis Report

To identify users where mobile numbers exist in AD but are missing in Entra ID:

```powershell
# Get all synchronized users from Entra ID
$entraUsers = Get-MgUser -All -Property UserPrincipalName,MobilePhone,OnPremisesSyncEnabled | 
    Where-Object {$_.OnPremisesSyncEnabled -eq $true}

# Create gap analysis report
$gapReport = @()

foreach ($user in $entraUsers) {
    # Get corresponding on-premises user (requires AD PowerShell module)
    $onPremUser = Get-ADUser -Filter "UserPrincipalName -eq '$($user.UserPrincipalName)'" -Properties mobile,otherMobile -ErrorAction SilentlyContinue
    
    if ($onPremUser) {
        # Check for mobile number gaps
        if (($onPremUser.mobile -and !$user.MobilePhone) -or 
            ($onPremUser.mobile -ne $user.MobilePhone)) {
            
            $gapReport += [PSCustomObject]@{
                UserPrincipalName = $user.UserPrincipalName
                ADMobile = $onPremUser.mobile
                ADOtherMobile = $onPremUser.otherMobile
                EntraIDMobile = $user.MobilePhone
                SyncGap = if ($onPremUser.mobile -and !$user.MobilePhone) { "Missing in Entra" } else { "Value Mismatch" }
            }
        }
    }
}

# Export gap analysis
$gapReport | Export-Csv -Path "MobileSyncGapAnalysis.csv" -NoTypeInformation
Write-Host "Found $($gapReport.Count) users with mobile number sync gaps"
```

## Disabling BypassDirSyncOverridesEnabled to Restore AD Sync

To restore normal synchronization of mobile numbers from Active Directory to Entra ID:

```powershell
# Connect with write permissions
Connect-MgGraph -Scopes "Directory.ReadWrite.All"

# Get the directory settings template
$template = Get-MgDirectorySettingTemplate | Where-Object {$_.DisplayName -eq "Directory Feature Settings"}

# Check if setting already exists
$setting = Get-MgDirectorySetting | Where-Object {$_.TemplateId -eq $template.Id}

if ($setting) {
    # Update existing setting to disable bypass
    $bypassSetting = $setting.Values | Where-Object {$_.Name -eq "BypassDirSyncOverridesEnabled"}
    if ($bypassSetting) {
        $bypassSetting.Value = "false"
        Update-MgDirectorySetting -DirectorySettingId $setting.Id -BodyParameter $setting
        Write-Host "BypassDirSyncOverridesEnabled disabled - mobile numbers will now sync from AD"
    }
} else {
    # Create new setting with bypass disabled
    $values = @()
    $template.SettingTemplateValues | ForEach-Object {
        if ($_.Name -eq "BypassDirSyncOverridesEnabled") {
            $values += @{Name = $_.Name; Value = "false"}
        } else {
            $values += @{Name = $_.Name; Value = $_.DefaultValue}
        }
    }
    
    $newSetting = @{
        TemplateId = $template.Id
        Values = $values
    }
    
    New-MgDirectorySetting -BodyParameter $newSetting
    Write-Host "Directory Feature Settings created with BypassDirSyncOverridesEnabled disabled"
}
```

## Important Considerations After Disabling

- **Sync timing**: Changes take effect immediately, but the next directory synchronization cycle (typically 30 minutes) is required to populate mobile numbers from AD
- **Manual sync**: Use `Start-ADSyncSyncCycle -PolicyType Delta` to trigger immediate synchronization
- **Cloud changes lost**: Any mobile numbers manually entered in Entra ID will be overwritten by AD values
- **Ongoing management**: Mobile numbers must now be managed in on-premises Active Directory
- **Audit logs**: The configuration change and subsequent sync updates are logged in Entra ID audit logs

## Recommended Approach

For organizations expecting traditional directory synchronization behavior, disabling `BypassDirSyncOverridesEnabled` is typically the correct solution. This restores the expected flow where mobile phone attributes are managed in Active Directory and automatically synchronized to Entra ID, maintaining consistency with other synchronized user attributes.