---
layout: article
title:  "Deploying Volume License Apps Along Side Office 365"
key: post20180325
tags: [server]
---

![Office365 Logo](/assets/images/office-365-xml/office-365.jpg){:.rounded}

At work I was tasked with deploying volume licensed Project and Visio. My company is currently deploying Office 365 Pro Plus, which uses click-to-run, which [cannot be installed](https://docs.microsoft.com/en-us/office365/troubleshoot/installation/office-click-to-run-perpetual) side-by-side with the 2016 version of Project and Visio that are MSI based. Instead you must use the [Office Deployment Tool](https://www.microsoft.com/en-us/download/details.aspx?id=49117) (ODT) and xml files to add Visio/Project to an existing install.

<!--more-->

I followed this [guide](https://docs.microsoft.com/en-us/deployoffice/use-the-office-deployment-tool-to-install-volume-licensed-editions-of-visio-2016) from Microsoft and used their wonderful [xml generation tool](https://config.office.com/) to make the following config.

```xml
<Configuration ID="41ca57ae-ab80-4ba0-b270-23e9a5068a5c" Host="cm">
  <Info Description="Project 2016 Pro 32bit" />
  <Add OfficeClientEdition="32" Channel="Broad" OfficeMgmtCOM="TRUE" Version="16.0.11328.20468" ForceUpgrade="TRUE">
    <Product ID="ProjectProXVolume" PIDKEY="WGT24-HCNMF-FQ7XH-6M8K7-DRTW9">
      <Language ID="MatchOS" />
      <ExcludeApp ID="Groove" />
      <ExcludeApp ID="Teams" />
    </Product>
  </Add>
  <Property Name="SharedComputerLicensing" Value="0" />
  <Property Name="PinIconsToTaskbar" Value="TRUE" />
  <Property Name="SCLCacheOverride" Value="0" />
  <Property Name="AUTOACTIVATE" Value="1" />
  <Property Name="FORCEAPPSHUTDOWN" Value="TRUE" />
  <Property Name="DeviceBasedLicensing" Value="0" />
  <AppSettings>
    <Setup Name="Company" Value="Name" />
  </AppSettings>
  <Display Level="Full" AcceptEULA="TRUE" />
  <Logging Level="Standard" Path="C:\Windows\Logs\Software" />
</Configuration>
```

Note the PIDKEY field, this is where you can put the Generic KMS key listed or your C2R-P key. I missed the detail that I needed to change the key and instead installed the generic KMS key. This lead to an issue with my users getting prompted to activate Visio/Project after a few days. At first I was stumped by the problem until I reread the MS guide and the comments at the bottom of the guide.  

![Office365 Comment](/assets/images/office-365-xml/office365-comment.png){:.rounded}

[williestylez](https://github.com/williestylez) pointed out what I had missed. Rereading the article I do see they stressed this point, but I missed it. So now I had my install xml, but I still needed the uninstall xml. Whenever I make an application in sccm I like to create the install and uninstall regardless if I plan to use the uninstall. I find people will ask for programs to be mass removed in a hurry so it always helps to have the work done ahead of time. I struggled with the remove xml but eventually came up with this.

```xml
<Configuration>
  <Remove OfficeClientEdition="32">
    <Product ID="ProjectProXVolume">
      <Language ID="en-us" />
    </Product>
  </Remove>
</Configuration>
```

Note the line `<Language ID="en-us" />`, for some reason I had to specify the language I was uninstalling. Without that the command `setup.exe /configuration remove.xml` would not remove any office programs. 
