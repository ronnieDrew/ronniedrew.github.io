---
layout: post
title: "Creating a Chocolatey package"
date: 2013-08-04 14:01
comments: true
categories: [Chocolatey]
published: true
---
These days [Chocolatey](http://chocolatey.org/) is pretty much the first thing I install when working on a new PC. The utility of having that tool you need just a few keystrokes away is something we've been missing on Windows for years.

It's getting to the point where if a package isn't on Chocolatey then it's dead to me. 

That said with the [package list](http://chocolatey.org/packages) hovering around the 1,200 mark there are gaps that you're going to run into sooner or later. It's up to the community to fill these gaps.

<!--more-->

Today I'm planning on testing some VM configurations on Azure with a view to optimizing IO performance. I'm a big [RavenDb](http://ravendb.net) fan but running it on an Azure VM is problematic due to IO issues. Both the main Raven hosted solution providers ([CloudBird](http://www.cloudbird.net) & [RavenHQ](http://ravenhq.com)) run on AWS so I thought I'd test the IO performance of some comparable VM instances on Azure & AWS.

In order to benchmark IO I'm using [SQLIO](http://www.microsoft.com/en-us/download/details.aspx?id=20163) but as it's not on the Chocolatey package gallery I thought I'd take 5 minutes out of my day to add it.

The Chocolatey package really just consists of some meta data in a [nuspec file](https://github.com/chocolatey/chocolatey/wiki/CreatePackages#nuspec), an image to display on the gallery and 2 powershell scripts to silently install/uninstall the SQLIO MSI.

The first thing we need to do is setup a [GitHub repository](https://github.com/ChocolateyPackages) to host my Chocolatey packages. Then we create a directory to hold the sqlio package with the following 4 files.

```
ChocolateyPackages\sqlio\sqlio.nuspec
ChocolateyPackages\sqlio\sqlio.png
ChocolateyPackages\sqlio\tools\chocolateyInstall.ps1
ChocolateyPackages\sqlio\tools\chocolateyUninstall.ps1
```

The meta data in the nuspec file is all fairly self-explanatory, note that the iconUrl setting is the image that will be shown within the Chocolatey package gallery. This should be a png with a minimum size of 128x128.

``` xml sqlio.nuspec
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2011/08/nuspec.xsd">
    <metadata>
        <id>sqlio</id>
        <version>1.0</version>
        <title>sqlio</title>
        <authors>Ronan Gibney</authors>
        <owners>Ronan Gibney</owners>
        <licenseUrl>http://www.microsoft.com/en-us/download/details.aspx?id=20163</licenseUrl>
        <projectUrl>http://www.microsoft.com/en-us/download/details.aspx?id=20163</projectUrl>
        <iconUrl>https://github.com/ronnieDrew/ChocolateyPackages/raw/master/sqlio/sqlio.png</iconUrl>
        <requireLicenseAcceptance>true</requireLicenseAcceptance>
        <description>SQLIO is a tool provided by Microsoft which can also be used to determine the I/O capacity of a given configuration. SQLIO is provided ‘as is’ and there is no support offered for any problems encountered when using the tool. Please refer to the EULA.doc for the license agreement prior to using this tool.</description>
        <summary>SQLIO is a tool provided by Microsoft which can also be used to determine the I/O capacity of a given configuration.</summary>
        <releaseNotes>http://www.microsoft.com/en-us/download/details.aspx?id=20163</releaseNotes>
        <tags>sqlio</tags>
    </metadata>
</package>
```

The PowerShell files are just small wrappers around the MSIs own install/uninstall functionality. On both install/uninstall we need to check the install state in the registry and in order to do this we need to know the GUID assigned to the MSI.

A quick way to get the GUID is to run the following PowerShell command after you've installed the MSI. 
``` powershell
Get-WmiObject -Class Win32_Product | Where-Object { $_.Name -match "sqlio" }
```

The install script just checks the install state in the registry and then calls the Chocolatey helper function <code>Install-ChocolateyPackage</code> which will download and install the MSI.

``` powershell chocolateyInstall.ps1
$package = 'sqlio'
$build = '1'
$guid='{DD971DE9-FAF0-4A15-9BE4-5D766B05D11E}'

try {

  if (Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\$guid") {
    Write-ChocolateyFailure $package "$package is already installed!"
  }
  else {
    $params = @{
      PackageName = $package;
      FileType = 'msi';
      SilentArgs = '/QUIET';
      Url = "http://download.microsoft.com/download/f/3/f/f3f92f8b-b24e-4c2e-9e86-d66df1f6f83b/SQLIO.msi";
    }

    Install-ChocolateyPackage @params
    Write-ChocolateySuccess $package
  }

} catch {

  Write-ChocolateyFailure $package "$($_.Exception.Message)"
  throw 

}

```

And then we handle the uninstall. Again we just check the install state using the same GUID as before and hand over to the appropriate Chocolatey helper function to uninstall the MSI. Chocolatey in turn will elevate admin access and use msiexec to perform the uninstall. 

``` powershell chocolateyUninstall.ps1
$package = 'sqlio'
$build = '1'
$clsid='{DD971DE9-FAF0-4A15-9BE4-5D766B05D11E}'

try {
  
  if (Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\$clsid") {
    
    Uninstall-ChocolateyPackage $package 'msi' "$clsid /quiet" 
    Write-ChocolateySuccess $package

  } else {
    Write-ChocolateyFailure $package "$package is not installed!"
  }

} catch {

  Write-ChocolateyFailure $package "$($_.Exception.Message)"
  throw 

}
```

With that we just need to create & test the package itself. To create the package we just run <code>cpack .\sqlio.nuspec</code> in package directory, this will bundle our nuspec and PowerShell files into a package called <code>sqlio.1.0.nupkg</code>. To test the package we can use <code>cinst sqlio -source %cd%</code> which will tell Chocolatey to install the package using the current directory. 

``` powershell
cpack .\sqlio.nuspec
cinst sqlio -source %cd%
```

The final step is to push the package to the Chocolatey gallery. To do this we need to setup an account on [http://chocolatey.org](https://chocolatey.org/account/Register) to obtain an api key. We then need to store the apikey for chocolatey.org and then push the package to the gallery.

```
nuget setApiKey <Your-Api-Key> -Source http://chocolatey.org/
cpush sqlio.1.0.nupkg 
```

With that we're done, now back to Azure & RavenDb...