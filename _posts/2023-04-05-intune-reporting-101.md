---

title: Intune Reporting 101
date: 2023-04-05 12:00
categories: [device management, reporting]
tags: [endpoint management, intune, microsoft]
---

## How to get PowerShell transcript logs from client machines remotely in Azure File Blobs

Any Intune administrator or sysadmin knows how important the Powershell scripting functionality is for achieving a vast array of powerful tasks across your IT infrastructure.

![Intune Powershell Scripting Interface](https://lh5.googleusercontent.com/yUKMXpmn1gmc0z0oAQgwakZU7SIKWUYV8J7orV0Ny0c4EZSY8GFav87LBzf108ijgqS-FIBWkGPKp6oNK19eUJKexEE1wV1mipP4CdhSSAKpBurodR7Q-B-1gLwDEx6ZRhO25-F9b69LVAaeVNNBcZQ)

Intune Powershell Scripting Interface

From setting up company VPNs to more complicated bespoke jobs, having an insight of these scripts when they execute on the target machines is quite limited natively in Intune and has been for some time.

![Device success/fail breakdown chart](https://lh5.googleusercontent.com/4gsc5KgnAslTmYiFl5rXwTqET9m5YzpOTnAifSSHDn4aMI3pRC2G5vQ6WpfTkhsYb_y37b334nwsvxxDtLWgha96DsA23KUxKXrZm5IFoDhujAOPSR7a4RWl6QKEfw5D4e__o3ANiuWMz8qBSXhbOTc)

Device success/fail breakdown chart

Once a script is assigned via device or user assignment, the script will eventually be pulled down via the Intune Management Extension and executed on the client machine.

As depicted in the image above, The Intune portal gives the user a pie chart style summary of the script results showing a success and a fail rate.

If you drill into the device or user status you can click on machine entry to see a little bit more information as below:

![https://lh4.googleusercontent.com/z-BSmtHhQmfLhUBbAnW6De04RiMU34-YDeIHpkMUW7PwgvXVyOQrW3NbpOLF3qFhMoyJxDXqBT3Fq9zuqqMjnwgF2P9t4TSkt9B_2s_cGGwAf5y5NObA-Pp7BeEebTfDrXxjVWLIIj2Onqh24hNI-po](https://lh4.googleusercontent.com/z-BSmtHhQmfLhUBbAnW6De04RiMU34-YDeIHpkMUW7PwgvXVyOQrW3NbpOLF3qFhMoyJxDXqBT3Fq9zuqqMjnwgF2P9t4TSkt9B_2s_cGGwAf5y5NObA-Pp7BeEebTfDrXxjVWLIIj2Onqh24hNI-po)

![User result view](https://lh3.googleusercontent.com/2H2O2U1tZMPCGOd01OQMgiFVYPro2A5StjTDuTvCQr1cO6WLV9deUGec1TVnpflk_hyHrOHffoyFfbHwOrj4rl45qe9xBuSZym1i4TfNFaKmx-UoGoXimO_J0s-iSgrvhyDg72Y4wGIgMegSOXTrJZU)

User result view

Not very helpful… your script failed on the client but wouldn’t it be useful to know ***‘why?’***

You may have tested this script successfully on your personal computer or a virtual machine in the cloud however this doesn’t help when you are deploying to a number of different machines that could be across various departments or locations.

**These machines could:**

- Have different OS and applications versions
- Out of date Powershell
- Permissions or Script Execution issues

Wouldn’t it be useful to just see the logs from that machine straight away from the comfort of your own workstation without interacting with that user or their machine potentially disrupting them during their day-to-day?

**Enter the power of Powershell transcript logging and Azure File Blob Storage!**

With the help of some commands built into Microsoft’s [Az.Storage](http://Az.Storage) Powershell module it’s quite easy to just upload the transcript of the script during the end of script execution to a place you can access instantly such as Blob storage, or some kind of remote file storage.

![https://lh3.googleusercontent.com/jHcupNIkwlLoBmAzaRjDuToyNBL9VB-ec762uYbcp2hEOXIqLFOBWu3Lly5YhjPbt9eel2QSnX7dNuXfxyVIxjgm_UZ_UIx4I1odHuuB--GTPWtOXOnovJep6fWGtba0R67vSX1CovaNnMPGLHRSsJc](https://lh3.googleusercontent.com/jHcupNIkwlLoBmAzaRjDuToyNBL9VB-ec762uYbcp2hEOXIqLFOBWu3Lly5YhjPbt9eel2QSnX7dNuXfxyVIxjgm_UZ_UIx4I1odHuuB--GTPWtOXOnovJep6fWGtba0R67vSX1CovaNnMPGLHRSsJc)

**The basics steps to achieve this are as follows:**

- Ensure end devices have the latest version of the **‘Az’** Powershell modules are installed on them as interacting with Azure Blob Storage requires the Az.Storage in particular
- To make things a tad easier you can use my own Powershell module - [BlobTranscript](https://www.powershellgallery.com/packages/BlobTranscript/2.22) which is just a basic function wrapper for PS to communicate with Azure storage blobs
- Create a suitable blob file storage in your Azure tenant and create a SAS key for use in the Intune scripts
- Amend your Powershell scripts to include the following:
    - Start and stop transcript - **to create a local log**
    - Function to upload the transcript file to the blob storage
    

### **Installing latest Az.Storage Powershell Module and BlobTranscript Module from Gallery**

Create a new PS script and assign to your selected clients:

```powershell
Install-Module -Name Az.Storage -AllowClobber -Force
Install-Module -Name BlobTranscript
```

**Az.Storage** is just a sub-module of the primary **Az** Azure collection but it’s the only one required to manipulate blob storage which is all we are doing.

**-AllowClobber** just in case the client has an older version already installed, it will update to the latest.

BlobTranscript allows you to simply use the included **Send-Transcript** command anywhere in your scripts to upload the log file to your designated storage.

To test this prior I personally used a lab VM in Azure but feel free to just use your local machine.

Upload a new PowerShell script and use the below settings:


**Make sure to run this script using logged on credentials.**

I’ve assigned to a testing group ‘AP Testing’ I created in Azure AD that contains the lab VM username.

Once assigned, to quickly force the client to pull down the new script, restart the **“Microsoft Intune Management Extension”** service.