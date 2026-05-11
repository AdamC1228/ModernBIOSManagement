# Modern BIOS Management

A collection of PowerShell scripts for automating BIOS/firmware detection and updates across multiple manufacturers using Microsoft Configuration Manager (ConfigMgr) task sequences.

> Full implementation guide: [msendpointmgr.com/modern-bios-management](https://www.msendpointmgr.com/modern-bios-management)

---

## Overview

Modern BIOS Management automates the process of detecting, downloading, and applying BIOS updates during ConfigMgr OSD task sequences (bare metal and in-place). It queries the ConfigMgr AdminService (or legacy WebService) to find a matching BIOS package for the current device, downloads the package content, and then hands off to a manufacturer-specific flash script.

### Supported Manufacturers

| Manufacturer | Flash Script |
|---|---|
| HP / Hewlett-Packard | `Invoke-HPBIOSUpdate.ps1` |
| Dell | `Invoke-DellBIOSUpdate.ps1` |
| Lenovo | `Invoke-LenovoBIOSUpdate.ps1` |
| Microsoft Surface | `Invoke-MicrosoftBIOSUpdate.ps1` |
| Fujitsu | `Invoke-CMDownloadBIOSPackage.ps1` (detection only) |
| Panasonic | `Invoke-CMDownloadBIOSPackage.ps1` (detection only) |
| Viglen | `Invoke-CMDownloadBIOSPackage.ps1` (detection only) |
| AZW | `Invoke-CMDownloadBIOSPackage.ps1` (detection only) |

---

## Scripts

### Invoke-CMDownloadBIOSPackage.ps1
**Current version — uses the ConfigMgr AdminService (REST API)**

Detects the device manufacturer, model, and SystemSKU, queries the AdminService for a matching BIOS package, compares the available version to the currently installed BIOS, and downloads the package if an update is available.

Supports both internal (on-prem) and external (Cloud Management Gateway) AdminService endpoints.

**Parameters**

| Parameter | Required | Description |
|---|---|---|
| `-BareMetal` | Yes* | Run in WinPE / bare metal deployment mode |
| `-BIOSUpdate` | Yes* | Run in full OS deployment mode |
| `-Endpoint` | Yes | FQDN of the ConfigMgr server hosting the AdminService |
| `-Filter` | No | Package name filter (default: `BIOS`) |
| `-OperationalMode` | No | `Production` (default) or `Pilot` |
| `-DebugMode` | No | Run outside of a task sequence for testing |
| `-UserName` | Debug | Service account username for AdminService auth |
| `-Password` | Debug | Service account password for AdminService auth |
| `-Manufacturer` | Debug | Override detected manufacturer |
| `-ComputerModel` | Debug | Override detected model |
| `-SystemSKU` | Debug | Override detected SystemSKU |

*`-BareMetal` and `-BIOSUpdate` are mutually exclusive parameter sets.

**Examples**

```powershell
# Bare metal / WinPE deployment
.\Invoke-CMDownloadBIOSPackage.ps1 -BareMetal -Endpoint "CM01.domain.com"

# Full OS / in-place BIOS update
.\Invoke-CMDownloadBIOSPackage.ps1 -BIOSUpdate -Endpoint "CM01.domain.com"

# Debug mode — detect and report without downloading
.\Invoke-CMDownloadBIOSPackage.ps1 -DebugMode -Endpoint "CM01.domain.com" -UserName "svc_account" -Password "P@ssw0rd"

# Debug mode with manual overrides
.\Invoke-CMDownloadBIOSPackage.ps1 -DebugMode -Endpoint "CM01.domain.com" -UserName "svc_account" -Password "P@ssw0rd" -Manufacturer "HP" -ComputerModel "ZBook Studio x360 G5" -SystemSKU "8427"
```

**Log file:** `ApplyBIOSPackage.log` (written to `_SMSTSLogPath`)

---

### Invoke-CMDownloadBIOSPackage_Legacy.ps1
**Legacy version — uses the ConfigMgr WebService (SOAP)**

Functionally equivalent to the current version but targets the older ConfigMgr WebService endpoint rather than the AdminService. Use this if your environment has not yet adopted the AdminService.

**Parameters**

| Parameter | Required | Description |
|---|---|---|
| `-URI` | Yes | Full URI of the ConfigMgr WebService endpoint |
| `-SecretKey` | Yes | Secret key configured on the WebService |
| `-Filter` | No | Package name filter (default: empty, returns all) |
| `-DeploymentType` | No | `BareMetal` (default) or `BIOSUpdate` |
| `-DebugMode` | No | Run outside of a task sequence for testing |

**Examples**

```powershell
# Bare metal deployment
.\Invoke-CMDownloadBIOSPackage_Legacy.ps1 -URI "http://CM01.domain.com/ConfigMgrWebService/ConfigMgr.asmx" -SecretKey "12345" -Filter "BIOS"

# Full OS BIOS update
.\Invoke-CMDownloadBIOSPackage_Legacy.ps1 -URI "http://CM01.domain.com/ConfigMgrWebService/ConfigMgr.asmx" -SecretKey "12345" -Filter "BIOS" -DeploymentType BIOSUpdate

# Pilot ring packages
.\Invoke-CMDownloadBIOSPackage_Legacy.ps1 -URI "http://CM01.domain.com/ConfigMgrWebService/ConfigMgr.asmx" -SecretKey "12345" -Filter "BIOS Update Pilot"
```

**Log file:** `ApplyBIOSPackage.log`

---

### Invoke-HPBIOSUpdate.ps1

Detects which HP flash utility is present in the downloaded BIOS package and invokes it silently. Supports the following utilities (in priority order):

1. `HPBIOSUPDREC64.exe` / `HPBIOSUPDREC.exe`
2. `HpFirmwareUpdRec64.exe` / `HpFirmwareUpdRec.exe`
3. `HPQFlash64.exe` / `HPQFlash.exe`

Automatically suspends BitLocker before flashing when running in the full OS.

**Parameters**

| Parameter | Required | Description |
|---|---|---|
| `-Path` | Yes | Path to the downloaded BIOS package (`%OSDBIOSPackage01%`) |
| `-PasswordBin` | No | BIOS password `.bin` filename (must be in the same directory as this script) |

**Example**

```powershell
.\Invoke-HPBIOSUpdate.ps1 -Path "%OSDBIOSPackage01%" -PasswordBin "Password.bin"
```

**Log file:** `Invoke-HPBIOSUpdate.log`

---

### Invoke-DellBIOSUpdate.ps1

Invokes a Dell BIOS update using `Flash64W.exe` on 64-bit systems, or the standalone BIOS executable on 32-bit systems. Automatically suspends BitLocker before flashing when running in the full OS.

**Parameters**

| Parameter | Required | Description |
|---|---|---|
| `-Path` | No | Path to the BIOS package. Defaults to the `OSDBIOSPackage01` TS variable |
| `-Password` | No | BIOS password |
| `-LogFileName` | No | Flash utility log filename (default: `DellFlashBIOSUpdate.log`) |

**Example**

```powershell
.\Invoke-DellBIOSUpdate.ps1 -Password "BIOSPassword" -LogFileName "DellBIOSUpdate.log"
```

**Log file:** `Invoke-DellBIOSUpdate.log`

---

### Invoke-LenovoBIOSUpdate.ps1

Invokes a Lenovo BIOS update using `WinUPTP64.exe` / `WinUPTP.exe` or `Flash64.cmd` / `Flash.cmd`, whichever is found in the package. Copies `OLEDLG.dll` into the package directory if missing (required by WinUPTP). Automatically suspends BitLocker before flashing when running in the full OS.

> **Note:** Requires the **WinPE-HTA** optional component in the boot image when running in WinPE.

**Parameters**

| Parameter | Required | Description |
|---|---|---|
| `-Path` | Yes | Path to the downloaded BIOS package (`%OSDBIOSPackage01%`) |
| `-Password` | No | BIOS password |
| `-LogFileName` | No | Flash utility log filename (default: `LenovoFlashBiosUpdate.log`) |

**Example**

```powershell
.\Invoke-LenovoBIOSUpdate.ps1 -Path "%OSDBIOSPackage01%" -Password "BIOSPassword"
```

**Log file:** `Invoke-LenovoBIOSUpdate.log`

---

### Invoke-MicrosoftBIOSUpdate.ps1

Applies Microsoft Surface firmware updates by invoking `pnputil` against all `.inf` files in the downloaded firmware package. Reads the package path from the `OSDBIOSPackage01` task sequence variable.

**Parameters**

| Parameter | Required | Description |
|---|---|---|
| `-LogFileName` | No | Log filename for pnputil output (default: `MicrosoftBIOSUpdate.log`) |

**Example**

```powershell
.\Invoke-MicrosoftBIOSUpdate.ps1
```

**Log file:** `Invoke-MicrosoftBIOSUpdate.log`

---

## Task Sequence Variables

| Variable | Set by | Used by | Description |
|---|---|---|---|
| `OSDBIOSPackage01` | `Invoke-CMDownloadBIOSPackage.ps1` | Flash scripts | Path to the downloaded BIOS package |
| `NewBIOSAvailable` | `Invoke-CMDownloadBIOSPackage.ps1` | Task sequence | `true` if a newer BIOS version was found |
| `SMSTSBIOSUpdateRebootRequired` | `Invoke-DellBIOSUpdate.ps1` | Task sequence | `true` if a reboot is required after flashing |
| `SMSTSBIOSInOSUpdateRequired` | `Invoke-DellBIOSUpdate.ps1` | Task sequence | `false` after a successful WinPE flash |
| `MDMUserName` | Environment | Download script | Service account username (alternative to parameter) |
| `MDMPassword` | Environment | Download script | Service account password (alternative to parameter) |
| `MDMExternalEndpoint` | Environment | Download script | CMG external endpoint URL |
| `MDMClientID` | Environment | Download script | Azure AD app client ID for CMG auth |
| `MDMTenantName` | Environment | Download script | Azure AD tenant name for CMG auth |
| `MDMApplicationIDURI` | Environment | Download script | App ID URI (default: `https://ConfigMgrService`) |

---

## Recommended Task Sequence Layout

```
[Check BIOS]
  Run PowerShell Script: Invoke-CMDownloadBIOSPackage.ps1
    Condition: NewBIOSAvailable equals true

[Download Package Content]
  (Handled automatically by the detection script)

[Apply BIOS Update]
  Run PowerShell Script: Invoke-HPBIOSUpdate.ps1       (Condition: Manufacturer = HP)
  Run PowerShell Script: Invoke-DellBIOSUpdate.ps1     (Condition: Manufacturer = Dell)
  Run PowerShell Script: Invoke-LenovoBIOSUpdate.ps1   (Condition: Manufacturer = Lenovo)
  Run PowerShell Script: Invoke-MicrosoftBIOSUpdate.ps1 (Condition: Manufacturer = Microsoft)

[Restart Computer]
```

---

## Authors

| Name | Twitter |
|---|---|
| Nickolaj Andersen | [@NickolajA](https://twitter.com/NickolajA) |
| Maurice Daly | [@MoDaly_IT](https://twitter.com/MoDaly_IT) |
| Lauri Kurvinen | [@estmi](https://twitter.com/estmi) |

---

## License

This project is licensed under the terms of the [LICENSE](LICENSE) file included in this repository.
