---
title: Update Powershell Modules Smarter
date: 2023-04-06 13:00
categories: [device management, scripting]
tags: [powershell, intune, microsoft, powershell modules]
---

## Updating Select Powershell Modules on Endpoints

Want a clean and thorough way to install or update Powershell modules on a machine? 

**Look no further...**

Create a new `.ps1` file (I use VS Code) with the following. 

**Make sure to choose your own Powershell modules to install:**

````powershell
# PS Modules to install/update - List your modules here:
$modules = @(
    'PowershellGet' 
    'BlobTranscript'
    'AzTable'
    'Az.Storage'
    'Az.Relay'
    'Az.Websites'
    'dbatools'
        
)

function Update-Modules([string[]]$modules) {
    #Loop through each module in the list
    foreach ($module in $modules) {
        # Check if the module exists
        if (Get-Module -Name $module -ListAvailable) {
            # Get the version of the currently installed module
            $installedVersionV = (Get-Module -ListAvailable $module) | Sort-Object Version -Descending  | Select-Object Version -First 1 

            # Convert version to string
            $stringver = $installedVersionV | Select-Object @{n = 'Version'; e = { $_.Version -as [string] } }
            $installedVersionS = $stringver | Select-Object Version -ExpandProperty Version
            Write-Host "Current version $module[$installedVersionS]"
            $installedVersionString = $installedVersionS.ToString()

            # Get the version of the latest available module from gallery
            $latestVersion = (Find-Module -Name $module).Version

            # Compare the version numbers
            if ($installedVersionString -lt $latestVersion) {
                # Update the module
                Write-Host "Found latest version $module [$latestVersion], updating.."
                # Attempt to update module via Update-Module
                try {
                    Update-Module -Name $module -Force -ErrorAction Stop -Scope AllUsers
                    Write-Host "Updated $module to [$latestVersion]"
                }
                # Catch error and force install newer module
                catch {
                    Write-Host $_
                    Write-Host "Force installing newer module"
                    Install-Module -Name $module -Force -Scope AllUsers
                    Write-Host "Updated $module to [$latestVersion]"
                }
                
            }
            else {
                # Already latest installed
                Write-Host "Latest version already installed"
            
            }
        }
        else {
            # Install the module if it doesn't exist
            Write-Host "Module not found, installing $module[$latestVersion].."
            Install-Module -Name $module -Repository PSGallery -Force -AllowClobber -Scope AllUsers
            }
        }
    }
        
Update-Modules($modules)


````


This script will run checks against the install version and the latest version from the Powershell Gallery.

It will also handle errors if Update-Module fails which can happen if some older modules were not installed originally via `Install-Module`.

## Intune Configuration


This is quite simple to deploy to endpoints via Intune.

Create a new device script with the following settings:

![Intune Settings](/assets/images/intune1.png)

Assign to a group and wait ~1 hour for the device to check in or simply restart the `Microsoft IntuneManagement Extension` service on the local machine to force check in with Intune.

You can see below that the modules were successfully deployed to the client machine.

![Powershell Modules](/assets/images/intune2.png)

![Intune Result](/assets/images/intune3.png)

Thanks for reading!





