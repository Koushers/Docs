---
title: ASP.NET Core on Nano Server
author: shirhatti
description: Learn how to take an existing ASP.NET Core app and deploy it to a Nano Server instance running IIS.
manager: wpickett
ms.author: riande
ms.date: 11/04/2016
ms.prod: asp.net-core
ms.technology: aspnet
ms.topic: article
uid: tutorials/nano-server
---
# ASP.NET Core on Nano Server

By [Sourabh Shirhatti](https://twitter.com/sshirhatti)

In this tutorial, you'll take an existing ASP.NET Core app and deploy it to a Nano Server instance running IIS.

## Introduction

Nano Server is an installation option in Windows Server 2016, offering a tiny footprint, better security, and better servicing than Server Core or full Server. Please consult the official [Nano Server documentation](https://docs.microsoft.com/windows-server/get-started/getting-started-with-nano-server) for more details and download links for 180 Days evaluation versions. 

There are three easy ways for you to try out Nano Server. When you sign in with your MS account:

1. You can download the Windows Server 2016 ISO file and build a Nano Server image.

2. Download the Nano Server VHD.

3. Create a VM in Azure using the Nano Server image in the Azure Gallery. A free trial of Azure is avaiable.

In this tutorial, we will be using the 2nd option, the pre-built Nano Server VHD from Windows Server 2016.

Before proceeding with this tutorial, you will need the [published output](xref:host-and-deploy/directory-structure) of an existing ASP.NET Core application. Ensure your application is built to run in a **64-bit** process.

## Setting up the Nano Server instance

[Create a new Virtual Machine using Hyper-V](https://technet.microsoft.com/library/hh846766.aspx) on your development machine using the previously downloaded VHD. The machine will require you to set an administrator password before logging on. At the VM console, press F11 to set the password before the first log in. You also need to check your new VM's IP address either my checking your DHCP server's fixed IP supplied while provisioning your VM or in Nano Server recovery console's networking settings.

> [!NOTE]
> Let's assume your new VM runs with the local V4 IP address 192.168.1.10.

Now you're able to manage it using PowerShell remoting, which is the only way to fully administer your Nano Server.

## Connecting to your Nano Server instance using PowerShell Remoting

Open an elevated PowerShell window to add your remote Nano Server instance to your `TrustedHosts` list.

```PowerShell
$nanoServerIpAddress = "192.168.1.10"
Set-Item WSMan:\localhost\Client\TrustedHosts "$nanoServerIpAddress" -Concatenate -Force
```

> [!NOTE]
> Replace the variable `$nanoServerIpAddress` with the correct IP address.

Once you have added your Nano Server instance to your `TrustedHosts`, you can connect to it using PowerShell remoting.

```PowerShell
$nanoServerSession = New-PSSession -ComputerName $nanoServerIpAddress -Credential ~\Administrator
Enter-PSSession $nanoServerSession
```

A successful connection results in a prompt with a format looking like: `[192.168.1.10]: PS C:\Users\Administrator\Documents>`

## Creating a file share

Create a file share on the Nano server so that the published application can be copied to it. Run the following commands in the remote session:

```PowerShell
New-Item C:\PublishedApps\AspNetCoreSampleForNano -type directory
netsh advfirewall firewall set rule group="File and Printer Sharing" new enable=yes
net share AspNetCoreSampleForNano=c:\PublishedApps\AspNetCoreSampleForNano /GRANT:EVERYONE`,FULL
```

After running the above commands, you should be able to access this share by visiting `\\192.168.1.10\AspNetCoreSampleForNano` in the host machine's Windows Explorer.

## Open port in the firewall

Run the following commands in the remote session to open up a port in the firewall to let IIS listen for TCP traffic on port TCP/8000.

```PowerShell
New-NetFirewallRule -Name "AspNet5 IIS" -DisplayName "Allow HTTP on TCP/8000" -Protocol TCP -LocalPort 8000 -Action Allow -Enabled True
```

## Installing IIS

Add the `NanoServerPackage` provider from the PowerShell Gallery. Once the provider is installed and imported, you can install Windows packages.

Run the following commands in the PowerShell session that was created earlier:

```PowerShell
Install-PackageProvider NanoServerPackage
Import-PackageProvider NanoServerPackage
Install-NanoServerPackage -Name Microsoft-NanoServer-Storage-Package
Install-NanoServerPackage -Name Microsoft-NanoServer-IIS-Package
```

To quickly verify if IIS is setup correctly, you can visit the URL `http://192.168.1.10/` and should see a welcome page. When IIS is installed, a website called `Default Web Site` listening on port 80 is created by default.

## Install the ASP.NET Core Module

The ASP.NET Core Module is an IIS 7.5+ module which is responsible for process management of ASP.NET Core HTTP listeners and to proxy requests to processes that it manages. At the moment, the process to install the ASP.NET Core Module for IIS is manual. Install the [.NET Core Windows Server Hosting bundle](xref:host-and-deploy/iis/index#install-the-net-core-windows-server-hosting-bundle) on a regular (not Nano) machine. After installing the bundle on a regular machine, copy the following files to the file share that we created earlier.

On a regular (not Nano) server with IIS, run the following copy commands:

```PowerShell
Copy-Item -Path  C:\windows\system32\inetsrv\aspnetcore.dll -Destination `\\<nanoserver-ip-address>\AspNetCoreSampleForNano`
Copy-Item -Path  C:\windows\system32\inetsrv\config\schema\aspnetcore_schema.xml -Destination `\\<nanoserver-ip-address>\AspNetCoreSampleForNano`
```

Replace `C:\windows\system32\inetsrv` with `C:\Program Files\IIS Express` on a Windows 10 machine.

On the Nano side, you will need to copy the following files from the file share that we created earlier to the valid locations. So, run the following copy commands:

```PowerShell
Copy-Item -Path C:\PublishedApps\AspNetCoreSampleForNano\aspnetcore.dll -Destination C:\windows\system32\inetsrv\
Copy-Item -Path C:\PublishedApps\AspNetCoreSampleForNano\aspnetcore_schema.xml -Destination C:\windows\system32\inetsrv\config\schema\
```

Run the following script in the remote session:

[!code-powershell[](nano-server/enable-aspnetcoremodule.ps1)]

> [!NOTE]
> Delete the files *aspnetcore.dll* and *aspnetcore_schema.xml* from the share after the above step.

## Installing .NET Core Framework

If your app is published as a [framework-dependent deployment (FDD)](/dotnet/core/deploying/#framework-dependent-deployments-fdd), .NET Core must be installed on the server. Use the [dotnet-install.ps1 PowerShell script](https://dot.net/v1/dotnet-install.ps1) in a remote PowerShell session to install .NET Core on your Nano Server. Pass the CLI version with the `-Version` switch:

```console
dotnet-install.ps1 -Version 2.0.0
```

## Publishing the application

Copy the published output of your existing application to the file share's root.

You may need to make changes to your *web.config* to point to where you extracted *dotnet.exe*. Alternatively, you can add *dotnet.exe* to your PATH.

Example of how a *web.config* might look if *dotnet.exe* is **not** on the PATH:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
    </handlers>
    <aspNetCore processPath="C:\dotnet\dotnet.exe" arguments=".\AspNetCoreSampleForNano.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\aspnetcore-stdout" forwardWindowsAuthToken="true" />
  </system.webServer>
</configuration>
```

Run the following commands in the remote session to create a new site in IIS for the published app on a different port than the default website. You also need to open that port to access the web. This script uses the `DefaultAppPool` for simplicity. For more considerations on running under an application pool, see [Application Pools](xref:host-and-deploy/iis/index#application-pools).

```PowerShell
Import-module IISAdministration
New-IISSite -Name "AspNetCore" -PhysicalPath c:\PublishedApps\AspNetCoreSampleForNano -BindingInformation "*:8000:"
```

## Known issue running .NET Core CLI on Nano Server and workaround
```PowerShell
New-NetFirewallRule -Name "AspNetCore Port 81 IIS" -DisplayName "Allow HTTP on TCP/81" -Protocol TCP -LocalPort 81 -Action Allow -Enabled True
```

## Running the application

The published web app is accessible in a browser at `http://192.168.1.10:8000`. If you've set up logging as described in [Log creation and redirection](xref:host-and-deploy/aspnet-core-module#log-creation-and-redirection), you can view your logs at *C:\PublishedApps\AspNetCoreSampleForNano\logs*.
