---

title: Intune Reporting 101
date: 2023-04-07 16:58
categories: [device management, reporting]
tags: [endpoint management, intune, microsoft, powershell]
published: true
---


## How to get PowerShell transcript logs from client machines remotely in Azure File Blobs

Any Intune administrator or sysadmin knows how important the Powershell scripting functionality is for achieving a vast array of powerful tasks across your IT infrastructure.

![Intune Powershell Scripting Interface](/assets/images/reporting/1.png)

<sup>Intune Powershell Scripting Interface</sup>

From setting up company VPNs to more complicated bespoke jobs, having an insight of these scripts when they execute on the target machines is quite limited natively in Intune and has been for some time.

![Device success/fail breakdown chart](/assets/images/reporting/2.png)

<sup>Device success/fail breakdown chart</sup>

Once a script is assigned via device or user assignment, the script will eventually be pulled down via the Intune Management Extension and executed on the client machine.

As depicted in the image above, The Intune portal gives the user a pie chart style summary of the script results showing a success and a fail rate.

If you drill into the device or user status you can click on machine entry to see a little bit more information as below:

![test](/assets/images/reporting/3.png)

![User result view](/assets/images/reporting/4.png)


Not very helpful… your script failed on the client but wouldn’t it be useful to know ***‘why?’***

You may have tested this script successfully on your personal computer or a virtual machine in the cloud however this doesn’t help when you are deploying to a number of different machines that could be across various departments or locations.

**These machines could:**

- Have different OS and applications versions
- Out of date Powershell
- Permissions or Script Execution issues

Wouldn’t it be useful to just see the logs from that machine straight away from the comfort of your own workstation without interacting with that user or their machine potentially disrupting them during their day-to-day?

Enter the power of Powershell transcript logging and Azure File Blob Storage! **💪**

With the help of some commands built into Microsoft’s **`Az.Storage`** Powershell module, it’s quite easy to upload the transcript of the script during the end of script execution to a place you can access instantly such as Azure Blob storage, or other kinds of remote file storage.

![Example logs](/assets/images/reporting/5.png)

**The basics steps to achieve this are as follows:**

- Ensure end devices have the latest version of the **`Az`** Powershell modules are installed on them as interacting with Azure Blob Storage requires the Az.Storage in particular
- To make things a tad easier you can use my own Powershell module - [`BlobTranscript`](https://www.powershellgallery.com/packages/BlobTranscript/2.22) which is just a basic function wrapper for PS to communicate with Azure storage blobs
- Create a suitable blob file storage in your Azure tenant and create a SAS key for use in the Intune scripts
- Amend your Powershell scripts to include the following:
    - Start and stop transcript - **to create a local log**
    - Function to upload the transcript file to the blob storage
    

# Deploy the required Powershell modules to your endpoints

Before you can take advantage of enhanced logging, the client machines will require a few Powershell modules installing.

### Create a new PS script and assign to your desired groups

This is the script from my previous article: [Powershell Module Updating](https://mrdobing.github.io/device%20management/scripting/2023/powershell-updatemodule/)

This function will not just try to install the modules but check for older versions and appropriately update them including error handling for modules that were not originally installed via **`Install-Module`** which I found was quite common during testing.

```powershell
# PS Modules to install/update - List your modules here:
$modules = @(
    'PowershellGet' 
    'BlobTranscript'
    'Az.Storage'        
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
```

**`Az.Storage`** is just a sub-module of the primary **`Az`** Azure collection but it’s the only one required to manipulate blob storage which is all we are doing.

BlobTranscript allows you to simply use the included **`Send-Transcript`** command anywhere in your scripts to upload the log file to your designated storage.

To test this prior I personally used a Windows VM in Azure but feel free to just use your local machine.

### **Upload a new PowerShell script within Intune**

![intune1.png](/assets/images/reporting/6.png)

I’ve assigned to a testing group ‘AP Testing’ I created in Azure AD that contains the lab VM username.

Once assigned, to quickly force the client to pull down the new script, restart the **“Microsoft Intune Management Extension”** service. 

From checking, you can see the modules were installed/updated on the client machine:

![Result of `Get-InstalledModule`](/assets/images/reporting/7.png)

Result of `Get-InstalledModule`

Intune reports that the script was successfully deployed.

![Intune status report](/assets/images/reporting/8.png)
<sub>Intune status report</sub>

# Set up blob storage in Azure

The chosen storage medium for this was Blob storage as I’m going to assume you have access to this since you are utilising Microsoft Intune on your estate.

### Create a new Resource Group and Storage Account

Create a new resource group in Azure with default settings and choose a name. I’m using my Azure monthly credits in this example.

![Untitled](/assets/images/reporting/9.png)

Create a new storage account within the new resource group. Keep all settings default aside Replication, set this to `Locally-redundant storage (LRS)`. 

We don’t require any kind of data redundancy on these files as they just for diagnostics and testing.

![Untitled](/assets/images/reporting/10.png)

- Once the storage has deployed, in the left column scroll down to 📁**Containers**
- Click ➕ Container
- Name: intunelogcollection
- Public access level: Blob

![Untitled](/assets/images/reporting/11.png)

- Once the blob has been created, on the left blade head into ♾️**Shared access tokens**
- Set the follow values:
- Permissions: **Read, Write**

![Untitled](/assets/images/reporting/12.png)

<aside>
💡 Set your start and expiry dates to whatever you require

</aside>

- Click **Generate SAS token and URL**
- Go back to the Storage account window previous and click 🔑**Access keys** on the left hand blade
- Copy the value from key1 below:

![Untitled](/assets/images/reporting/13.png)

# Implement transcript upload in your scripts

Create or amend one of your powershell scripts that you want to deploy via Intune and add these variables to the top. 

These will be passed into the functions:

- **`Start-Transcript`** - Starts logging Powershell output to file. Ensure to use verbose in your scripts to enhance the level of logging
- **`Send-Transcript`** - Uploads the log file to your blob storage location

```powershell
	$computerName = hostname 
    $userName = (Get-ChildItem Env:\USERNAME).Value # Grab current username
    $date = (get-date).ToString("yyyy-MM-dd-HH-mm-ss")
    $logPath = "C:\Intune\logs" # Where to store logs locally
    $transcriptFile = "$logPath\$computerName-$userName-$date.transcript"
    $scriptName = $MyInvocation.MyCommand.Name # Get current script name

    # BlobTranscript Variables
    $storageAccountName = "intunelogcollection" # Storage account name
    $storageAccountKey = "**INSERT YOUR STORAGE KEY**" # Access key
    $containerName = "intunelogcollection" # Blob name
    $blobName = "$computerName-$userName/$scriptName-$date" # Format the log file name

Start-Transcript -Path $transcriptFile # Start logging
Import-Module BlobTranscript # Imports the blob storage module
```


>💡 Make sure you use the correct values for the storage account variables.

Paste the **key1** value from the previous Access Key dialog box for the variable `$storageAccountKey`.

At the end of your script place the following code:

```powershell
Stop-Transcript # Stops transcript logging 
Send-Transcript -transcript $transcriptFile -storageAccountName $storageAccountName -storageAccountKey $storageAccountKey -containerName $containerName -blobName $blobName # Upload the log file to blob storage
```

This will stop the logging inside Powershell and call the module BlobTranscript and the function `Send-Transcript` for uploading the log file to the blob storage. 

>A new folder and naming convention will be created for that particular device using the following format:

**`$computerName-$userName/$scriptName-$date.log`**

**`DEV-2512-JoeBloggs/Script1-2023-04-07.log`**

### Testing locally

In the below example I am using the module updater script from earlier but implementing transcript logging and uploading:

```powershell
#region Initial Variables
$computerName = hostname 
    $userName = (Get-ChildItem Env:\USERNAME).Value # Grab current username
    $date = (get-date).ToString("yyyy-MM-dd-HH-mm-ss")
    $logPath = "C:\Intune\logs" # Where to store logs locally
    $transcriptFile = "$logPath\$computerName-$userName-$date.transcript"
    $scriptName = $MyInvocation.MyCommand.Name # Get script name

    # BlobTranscript Variables
    $storageAccountName = "intunelogcollection" # Storage account name
    $storageAccountKey = "**INSERT YOUR STORAGE KEY**" # Access key
    $containerName = "intunelogcollection" # Blob name
    $blobName = "$computerName-$userName/$scriptName-$date" # Format the log file name
#endregion

Start-Transcript -Path $transcriptFile # Start logging
Import-Module BlobTranscript # Imports the blob storage module

#region Main Script Body - Update Modules

# PS Modules to install/update - List your modules here:
$modules = @(
    'PowershellGet' 
    'BlobTranscript'
    'Az.Storage'        
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

#endregion

Stop-Transcript # Stops transcript logging 
Send-Transcript -transcript $transcriptFile -storageAccountName $storageAccountName -storageAccountKey $storageAccountKey -containerName $containerName -blobName $blobName # Upload the log file to blob storage
```

Running this locally on my own machine I get the following:

![Untitled](/assets/images/reporting/14.png)

As you can see, the script calls the Blob storage with the provided parameters and uploads the completed transcript.

### Testing via Intune

Assign this to a testing group in Intune with the same settings as before:

![Untitled](/assets/images/reporting/15.png)

Trigger a check in with Intune on your testing device and double check that you see the folder and log file uploading to your storage.

The transcript log is saved locally on the target machine:

![Untitled](/assets/images/reporting/16.png)

Voila! The folder is created and log file is present in our blob storage container:

![Untitled](/assets/images/reporting/17.png)

Unfortunately when scripts are deployed via Intune they lose their intial script name as seen below..

![Untitled](/assets/images/reporting/18.png)

Download and save your log file and open with notepad or any text editor, I can see anything that I wrote to host during script execution or any verbose commands. 

**This could be adapted to suite your individual needs and include error handling and much more in-depth information!**

![Untitled](/assets/images/reporting/19.png)

I hope this was a useful post and I’m sure people can adapt this to their needs. I found this especially useful when testing script/apps deployments across remote devices where it might be difficult to fault find errors when all you see in the portal is a pass or fail based on return codes from your scripts.

In the future I am going to try evolve this into a more useful and easily deployable solution similar to Windows Update for Business Reports in Azure.. keep tuned.

[https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-overview](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-overview)

**Other suggestions for improvements could include:**

- Inclusion of the Microsoft Intune logs or package installer logs such as winget and chocolatey for app deployment
- Other storage locations such as PowerBI, Teams, Slack Bots etc…
- Utilise Azure Key Vault

I’m honestly surprised Microsoft haven’t given us more tools and reporting potential for Endpoint Management. It’s a good job the community is doing what they can to fill these gaps!

Please feel free to critique, suggest improvements or post any errors or issues you are encountering and I will do my best to help resolve.

**Happy Intuning!**  🤓

<div class="commentbox"></div>
<script src="https://unpkg.com/commentbox.io/dist/commentBox.min.js"></script>
<script>commentBox('5680867183689728-proj')</script>
