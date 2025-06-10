# 🔄 Why Mobile Numbers Aren't Syncing from AD to Entra ID

> **TL;DR**: Your AD mobile numbers aren't appearing in Entra ID? It's likely that `BypassDirSyncOverridesEnabled` is not what you expect it to be.

---

## 🚨 The Problem

In hybrid Microsoft Entra ID environments, many admins are **surprised** to find that mobile numbers in Active Directory don't sync to Entra for some users — even though all other attributes do perfectly. You also don't see any signs of failure on Entra Connect log.

**What you see:**
```
Active Directory          Entra ID
┌─────────────────┐      ┌─────────────────┐
│ 📱 +1234567890  │ ───► │ 📱 +1219999999  │
│ 👤 John Doe     │ ───► │ 👤 John Doe     │ ✅
│ 📧 john@co.com  │ ───► │ 📧 john@co.com  │ ✅
└─────────────────┘      └─────────────────┘
```

**The Impact:**
- 🔐 **MFA setup fails** (no mobile number)
- 🔒 **SSPR doesn't work** (missing recovery method)  
- 📞 **Teams contact data incomplete**

---

## 🧠 Root Cause: BypassDirSyncOverridesEnabled (or is it)

The culprit is a tenant setting called **`BypassDirSyncOverridesEnabled`**:

| Setting Value | Behavior |
|---------------|----------|
| `false` (default) | 🚫 **Blocks** AD mobile sync - Entra keeps cloud values |
| `True` | ✅ **Allows** AD mobile sync - Traditional behavior |

This setting specifically impacts:
- 📱 `mobile` → `MobilePhone`
- 📱 `otherMobile` → `AlternateMobilePhones`


---

## 📅 When Did This Change?

Prior to mid-2023, Entra ID allowed cloud updates to certain contact fields (like Mobile) for hybrid users. Users were were bale to update on their my profile page on Entra and Admins were also update. These changes were only updated on their Entra ID User object.

Later, Microsoft introduced `BypassDirSyncOverridesEnabled` setting with-in Entra Connect, that allows organization to control this behavior. 

**Why admins are caught off-guard:**
- 🤐 **Silent rollout** - No widespread communication
- 🔍 **PowerShell-only** - Not visible in admin portals  
- 🎯 **Selective blocking** - Only mobile attributes affected
- ❌ **No error messages** - Sync appears to work fine

---

## The REAL issue IMO
*No issues logged on Entra Connect.*

The biggest issue with this setting is that you may see that mobile number fields are updating from Local AD to Entra, for most users. And you may see it not updating for some users. But you will NOT see any issues or any trace of it on your Entra Connect SYNC log.

It is also confusing that you will see your accounts sync the mobile field, even when the `BypassDirSyncOverridesEnabled` setting is set to FALSE.

If you are a traditional organization on hybrid, where you still consider Local AD as your "sourece of truth", you want to ensure that Local AD values are synced to Entra ID. If that is the case, you probably want this setting to be Enabled, so you could continue to sync mobile number values from Local AD to Entra.

## Solution
1. Status Check > Identify the current status of `BypassDirSyncOverridesEnabled`
2. Evaluate the difference > Identify the user objects that has a different value (there is a command for that). This is in the event if you need to capture that data to be updated to Local AD.
3. The Fix > Enable the `BypassDirSyncOverridesEnabled` set to TRUE
4. Sync > Perform a Full Entra Connect Sync

  

## 🔍 1. Status Check

```powershell
# Connect to Microsoft Graph
Connect-MgGraph -Scopes "Directory.Read.All"

# Ensure to install ADSyncTools Module
Install-Module ADSyncTools

# Check current status
(Get-MgDirectoryOnPremiseSynchronization).Features.BypassDirSyncOverridesEnabled
```

**Result:**
- `True` = 🚫 Mobile sync blocked
- `False` = ✅ Mobile sync enabled

---

## 📊 2. Evaluate the difference

Find users that have different mobile numbers between Loca AD vs EntraID, using the ADSyncTools Module

```powershell
# Get sync gaps between AD and Entra ID
Compare-ADSyncToolsDirSyncOverrides
```

---

## ⚡ 3. The Fix

To restore traditional mobile number synchronization:

```powershell
# Connect with write permissions
Connect-MgGraph -Scopes "Directory.ReadWrite.All"

# Disable the bypass (enable AD sync)
$directorySynchronization = Get-MgDirectoryOnPremiseSynchronization
$directorySynchronization.Features.BypassDirSyncOverridesEnabled = $true
Update-MgDirectoryOnPremiseSynchronization -OnPremisesDirectorySynchronizationId $directorySynchronization.Id -Features $directorySynchronization.Features

```

## 🔄 4. Sync
Start a Full Sync cycle in MS Entra Connect (on your AD Connect server)

```powershell
Start-ADSyncSyncCycle -PolicyType Initial

```

---

## ⚠️ Important Considerations

| ✅ Pros | ⚠️ Considerations |
|---------|-------------------|
| Mobile numbers sync from AD | Cloud-edited mobile numbers will be overwritten |
| Consistent with other attributes | Must manage mobile numbers in Local AD |
| Resolves MFA/SSPR issues | Changes are logged in audit logs |

---

## 📚 Official Microsoft Resources

- **Primary Documentation**: [How to use the BypassDirSyncOverridesEnabled feature](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-bypassdirsyncoverrides)
- **Attribute Mapping**: [Attributes synchronized by Microsoft Entra Connect](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-attributes-synchronized)  
- **Graph API Reference**: [onPremisesDirectorySynchronizationFeature resource](https://learn.microsoft.com/en-us/graph/api/resources/onpremisesdirectorysynchronizationfeature)

---

**🎯 Bottom Line**: If your organization expects AD to be the source of truth for mobile numbers, set `BypassDirSyncOverridesEnabled` to TRUE, to restore traditional sync behavior.