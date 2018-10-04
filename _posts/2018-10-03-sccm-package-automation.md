---
layout: article
title:  "SCCM Package Automation part 1"
key: post20181003
tags: [powershell, scripts]
---

![](/assets/images/sccm-package-automation/PowerShell_5.0_icon.png){:.rounded}

Recently I published a project called [SCCM Package Automation](https://github.com/NathanTheGr8/SCCM-PackageAutomation) to my Github. I also posted it to [reddit](https://www.reddit.com/r/PowerShell/comments/9ilpdd/sccm_package_automation/) r/PowerShell and got a decent amount of attention. I thought I would write a more detailed blog post about it now.
<!--more-->

The purpose of the project is to automate the effort of maintaining SCCM packages for common apps. Here are some of the things it automates or eliminates
- Checking for the latest version
- The downloading of the latest version
- Updating PowerShell files with the new version or installer names
- Creating the new package folders and adding the install files
- Creating the package in SCCM
- Creating the install program in SCCM
- Distributing the package to SCCM Distribution Points
- Updating SCCM collections that track out of date apps

Things it does not do, but I want to do
- It doesn't deploy the newly created package to a test collection.
- Better error handling
- Include more common apps


The code isn't perfect, but it is presentable and relatively clean. I welcome feedback, corrections, and improvements.

## The Functions

### Get-LatestAppVersion

This function gets the latest version of a given app. The version is returned usually as a [version] object, but sometimes as a string like for Adobe Reader. Reader's version is returned as a string because its version number contains leading 0s that would have been stripped out by the version type.

For most of the apps, the function works by using `Invoke-Webrequest` to scrape a webpage for links. I then apply a regex on those links to look for a link with a naming format. If multiple links are returned I sort them by descending order and grab the latest.

```powershell
'vlc' {
            $url = "http://download.videolan.org/pub/videolan/vlc/"
            $html = Invoke-WebRequest -Uri "$url"

            $versionlinks = $html.Links | where href -match "^(\d+\.)?(\d+\.)?(\*|\d+)\/$" | Sort-Object -Property href -Descending
            $LatestAppVersion = $versionlinks[0].href -replace "/",""

        }
```
Each app is a little different. Some apps, like Firefox and Chrome, provide a web api you can query to get the latest version. This means it is super easy to implement and unlikely to break. All the apps I scrape the webpage to get the version info are vulnerable to the web pages changing significantly and breaking my script. Knowing this I still decided to use webpage scraping because I didn't want to really on third-party sources like Source Forge or Chocolatey for version info for downloads. If web pages change often enough, I may look at using a third party service for version info.

### Download-LatestAppVersion

This method is pretty similar to `Get-LatestAppVersion`, but goes one step farther and downloads the app to `$home\downloads\AppUpdates`. The Downloaded file(s) are returned by the method and are used later in Copy-PSADTFolders. Here is an example for flash.

```powershell
'flash' {
    $MajorVersion = (Get-LatestAppVersion -App $App).Major
    $FlashActiveX = "https://www.adobe.com/etc/adc/token/generation.installerlink.json?href=https%3A%2F%2Ffpdownload.macromedia.com%2Fget%2Fflashplayer%2Fdistyfp%2Fcurrent%2Fwin%2Finstall_flash_player_$($MajorVersion)_active_x.msi"
    $FlashPlugin = "https://www.adobe.com/etc/adc/token/generation.installerlink.json?href=https%3A%2F%2Ffpdownload.macromedia.com%2Fget%2Fflashplayer%2Fdistyfp%2Fcurrent%2Fwin%2Finstall_flash_player_$($MajorVersion)_plugin.msi"
    $FlashPpapi = "https://www.adobe.com/etc/adc/token/generation.installerlink.json?href=https%3A%2F%2Ffpdownload.macromedia.com%2Fget%2Fflashplayer%2Fdistyfp%2Fcurrent%2Fwin%2Finstall_flash_player_$($MajorVersion)_ppapi.msi"
    $FlashUrls = @($FlashActiveX,$FlashPlugin,$FlashPpapi)

    $FlashUrls | ForEach-Object {
        Write-Verbose -Message "Getting download token from Adobe"
        $JsonResponse = (New-Object System.Net.WebClient).DownloadString($_)

        Write-Verbose -Message "Extract the URL string from the JSON response"
        $Url = (New-Object System.Web.Script.Serialization.JavaScriptSerializer).DeserializeObject($JsonResponse).Values

        Write-Verbose -Message "Determining filename"
        $FileName = $($Url.Split('?')[0].Split('/')[-1])
        $FilePath = Join-Path -Path $DownloadDir -ChildPath $FileName

        Write-Verbose -Message "Downloading $FileName"
        (New-Object System.Net.WebClient).DownloadFile("$Url","$FilePath")
    }

    $MajorVersion = (Get-LatestAppVersion -App).Major
    $InstallFileName = "install_flash_player_$MajorVersion"
}
```

This app was pretty complicated, and I got some help from [jasonadsit](https://github.com/jasonadsit). I based my script of his [Gist](https://gist.github.com/jasonadsit/c77340fe385fe953f9c54436b926cf83) with some minor changes. I modified the URLs for each flash type to include the Major version. This way the script would not have to be updated. Jason has some nice code that generates a token to allow the msi's to download from the page.  [jasonadsit on reddit](https://www.reddit.com/r/PowerShell/comments/9htpfn/download_flash_msi_updates_from_distribution_page/e6gslmp/). I think I may find that this method breaks in the near future, but I have some backup ideas too.

### Update-PSADTAppVersion

This Functions

```powershell
function Update-PSADTAppVersion {
    param
    (
        [Parameter(Mandatory = $true)]
        [string]
        [ValidateScript({
            if(-Not (Test-Path -Path "$_") ){
                throw "Folder does not exist"
            }
            if(-Not (Test-Path -Path "$_" -PathType Container) ){
                throw "The PackageRootFolder argument must be a folder. Files are not allowed."
            }
            return $true
        })]
        $PackageRootFolder,
        [Parameter(Mandatory = $true)]
        [string]
        [ValidatePattern("^(\d+\.)?(\d+\.)?(\d+\.)?(\d+\.)?(\*|\d+)?$")]
        #basically can be up #.#.#.#.# or just one #
        $CurrentVersion,
        [Parameter(Mandatory = $true)]
        [string]
        [ValidatePattern("^(\d+\.)?(\d+\.)?(\d+\.)?(\d+\.)?(\*|\d+)?$")]
        #basically can be up #.#.#.#.# or just one #
        $NewVersion,
        [string]
        [ValidateScript({
            if(-Not (Test-Path -Path "$PackageRootFolder\$InstallScript") ){
                throw "The PackageRootFolder argument path does not exist"
            }
            if(-Not (Test-Path -Path "$PackageRootFolder\$InstallScript" -PathType Leaf) ){
                throw "The InstallScript argument must be a file."
            }
            return $true
        })]
        $InstallScript = "Deploy-Application.ps1" #defaults to this

    )

    (Get-Content "$PackageRootFolder\$InstallScript").Replace("`$appVersion = '$CurrentVersion'","`$appVersion = '$NewVersion'") | Set-Content  -Path "$PackageRootFolder\$InstallScript"
}
```

### Copy-PSADTFolders

This function is pretty short and is mostly app agnostic, meaning it behaves the same way for every app. This function works by getting the latest folder for a given app and copying it to a new folder in the naming convention `$app version# (R#)`. It then deletes the old install files in the files directory and copies the new install files to the files directory. For certain apps like Reader, it will not remove everying in the files directory since we only need to update the msp for the latest version. The base msi's and configuration files need to stay.

```powershell
function Copy-PSADTFolders {
    param
    (
        [Parameter(Mandatory = $true)]
        [string]
        [ValidateScript({
            if(-Not (Test-Path -Path "$_") ){
                throw "The PackageRootFolder argument path does not exist"
            }
            if(-Not (Test-Path -Path "$_" -PathType Container) ){
                throw "The PackageRootFolder argument must be a folder. Files are not allowed."
            }
            return $true
        })]
        $OldPackageRootFolder,
        [Parameter(Mandatory = $true)]
        [string]
        [ValidateScript({
            if((Test-Path -Path "$_") ){
                throw "The path for NewPackageRootFolder already exists"
            }
            return $true
        })]
        $NewPackageRootFolder,
        [Parameter(Mandatory = $true)]
        $NewPSADTFiles

    )
    Write-Output "Copying old package files to $NewPackageRootFolder"
    Copy-Item -Path "$OldPackageRootFolder" -Destination "$NewPackageRootFolder" -Recurse
    if ($OldPackageRootFolder -match "Reader"){
        Write-Output "Removing old msp install files"
        Remove-Item -Path "$NewPackageRootFolder\Files\*.msp"
    }
    elseif ($OldPackageRootFolder -match "BigFix"){
        Write-Output "Removing old msp install files"
        Remove-Item -Path "$NewPackageRootFolder\Files\*.exe"
    }
    else {
        Write-Output "Removing old install files"
        Remove-Item -Path "$NewPackageRootFolder\Files\*"
    }
    Write-Output "Copying new install files"
    Copy-Item -Path $NewPSADTFiles -Destination "$NewPackageRootFolder\Files" -Verbose

}
```

### Update-AppPackage

This is the main method that wraps everything together. It accepts an app from a predefined list and
1. Checks if the apps package is up to date.
2. If the package is out of date, downloads the latest app install file.
3. Calls `Copy-PSADTFolders` to make the new package folders
4. Calls `Update-PSADTAppVersion` to update the `$appVersion` variable in the Deploy-Application.ps1 file.

In step 3 I do a lot of checks when naming the new app folder. If the "new app folder" equals the "old app folder" I increment the "new app folder" until there is no conflict.

It outputs and prints a lot along the way. It can take 10-40 seconds to run depending on the app. Most of the time is spent downloading the latest app install files.

```powershell
if (($CurrentAppVersion -lt [version]$LatestAppVersion) -or ($ForceUpdate)) {
    if ($ForceUpdate) {
        Write-Host "Forcing update of $App from $CurrentAppVersion to $LatestAppVersion"
    }
    else {
        Write-Host "Upgrading $App package from $CurrentAppVersion to $LatestAppVersion"
    }

    $InstallFiles = Download-LatestAppVersion -App $App
    $rootApplicationPath = Get-RootApplicationPath -App $App
    $count = (Measure-Object -InputObject $SCCM_Share -Character).Characters + 1
    # Gets the most recent folder for a given app
    $CurrentAppPath =  "$($SCCM_Share_Letter):\" + $rootApplicationPath.Substring($count) | Get-ChildItem | sort CreationTime -desc | select -f 1

    $RevNumber = 1
    $newAppPath = "$rootApplicationPath\$App $LatestAppVersion (R$RevNumber)"

    $alreadyExists = Test-Path -Path "$newAppPath"
    while ($alreadyExists){
        #if the newAppPath already exists increments R#
        Write-Output "'$App $LatestAppVersion (R$RevNumber)' already exists, auto incrementing the R`#"
        $RevNumber++
        $newAppPath = "$rootApplicationPath\$App $LatestAppVersion (R$RevNumber)"
        $alreadyExists = Test-Path -Path "$newAppPath"
    }

    #Copies the Current Package to the new. Replaces install files and increments version.
    Write-Output "Creating folder '$App $LatestAppVersion (R$RevNumber)'"
    Copy-PSADTFolders -OldPackageRootFolder "$($CurrentAppPath.FullName)" -NewPackageRootFolder "$newAppPath" -NewPSADTFiles $InstallFiles
    Write-Output "Updating version numbers from $CurrentAppVersion to $LatestAppVersion"
    Update-PSADTAppVersion -PackageRootFolder "$newAppPath" -CurrentVersion "$CurrentAppVersion" -NewVersion "$LatestAppVersion"

}
else {
    Write-Host "$App $CurrentAppVersion package is already up to date"
}
```

## Conclusion
All in all, I had a lot of fun writing this project and hope other people find it useful. Soon I will post part 2 of this detailing the functions in `New-StandardAppPackage`
