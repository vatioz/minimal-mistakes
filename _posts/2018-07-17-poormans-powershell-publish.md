---
title: Poor man's Powershell publish of .NET Core
excerpt: How to automatize publishing of .NET Core project and releasing to GitHub
tags:
  - powershell
  - github
  - release
  - publish
category: programming
toc: true
draft: false
---

My latest project is supposed to be public and that was a new requirement for me. 

## Where to store and publish whatever comes out of my project?

I quickly learned that GitHub can host your releases and that there is no storage limit (apart from one file 2GB max). That seems reasonable - have releases right next to your code. I tried it and it is quite easy to do.


### GitHubReleaseManager

So I opened powershellgallery.org and entered GitHub...and there it is - module that takes care of releases on GitHub - https://www.powershellgallery.com/packages/GitHubReleaseManager

This will come handy later!

## .NET Core

My project tries to be multi-platform and uses .NET Core 2.1 to achieve that. What is different in .NET Core in comparison to full .NET Framework is that it is command line based. The center of all is `dotnet.exe` utility (located in `C:\Program Files\dotnet\`). 

### dotnet publish

Other thing that changed with new framework is that it doesn't produce executable when it's built. It kind of make sense, you don't know what platform it will be executed on - exe file is no-go for linux or mac.

Instead, the main output of your application is DLL and you use `dotnet run myapp.dll` to execute it. 

But this has its own problems. It assumes end user has .NET Core CLI installed and can run such command (end users on Windows don't use CLI, ever!).

And that's where `dotnet publish` comes into rescue. That command will solve both those issues:
* it generates self contained *package*, meaning it will contain whole .NET Core necessary to run the app
* it also generates platform specific *run script* that will take care of the `dotnet run some.dll` - for Windows it will generate exe file, that only runs mentioned command

Here is what's needed to publish .NET Core project:

```powershell
$project = "C:\path\to\apps\project.csproj"
$output = "C:\path\to\releases"
$dotnet = "C:\Program Files\dotnet\dotnet.exe"

# clear previous releases
Remove-Item "$output\*" -Recurse -Confirm:$true

& $dotnet publish $project -c release -o $output
```

This will generate publish folder for default set of platforms. If you want to control for which platforms you will release you need to change two things.  
(To find what platforms we want to publish for, go to https://docs.microsoft.com/en-us/dotnet/core/rid-catalog and choose from the supported platforms. I chose `win10-x64`, `win10-x86`, `win-x64`, `win-x86`, `osx-x64` and `linux-x64`.)

1. Edit your `csproj` and include chosen RIDs into `<RuntimeIdentifiers>` element under `<PropertyGroup>` element. My looks like this:

    ```xml
    <Project Sdk="Microsoft.NET.Sdk">
      <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>netcoreapp2.0</TargetFramework>
        <Platforms>AnyCPU;x86;x64</Platforms>
        <RuntimeIdentifiers>win-x64;win-x86;osx-x64;linux-x64;win10-x64;win10-x86;</RuntimeIdentifiers>    
      </PropertyGroup>
      <ItemGroup>
        <Folder ... />
        ...
      </ItemGroup>
    </Project>
    ```

2. Include these identifiers in our publishing Powershell script:

    ```powershell
    $project = "C:\path\to\apps\project.csproj"
    $output = "C:\path\to\releases"
    $dotnet = "C:\Program Files\dotnet\dotnet.exe"
    $runtimes = @(
    "win10-x64"
    "win10-x86"
    "win-x64"
    "win-x86"
    "osx-x64"
    "linux-x64"
    )
    
    # clear previous releases
    Remove-Item "$output\*" -Recurse -Confirm:$true
    
    $runtimes | %{
        & $dotnet publish $project -c release -r $_  -o ("{0}\{1}" -f $output,$_)
    }
    ```

This will generate publish folder only for specified platforms/runtimes - in my case 6 folders.

## Packing

Now, we need to create single file from each of those publish folders. Let's zip it with Powershell's integrated cmdlet `Compress-Archive`.

Simply do some magic with composing file names and output paths and ZIP it:

```powershell
$version = "0.5-beta"
$product = "MyApp"
$runtimes | %{
    $runtime = $_
    $toPack = ("{0}\{1}" -f $output,$runtime)
    $fileName = "{0}.{1}.{2}" -f $product,$version,$runtime #MyApp.0.5-beta.win-x86
    $zipFile = ("{0}\{1}.zip" -f $output,$fileName)
    Compress-Archive -Path $toPack -DestinationPath $zipFile
}
```

Ok, we have our release files ready, what now? Remember GitHubReleaseManager module? Using that we can create new release and upload our release files to it.

## GitHub Release

Let's begin with installing the module. This is done only once and is not part of the script. 

```powershell
Install-Module GitHubReleaseManager
```

Then, continue with importing and initializing module.

```powershell
# newer powershell versions do this automatically behind the scenes, so this is not required
Import-Module GitHubReleaseManager 

# this will prompt for API Token which should not be saved with the script and should be entered interactively
Set-GitHubSessionInformation -Username githubUser 
```

Now we can create new GitHub release. Note that variables are used conveniently, you may need different ones for repository and tag.

```powershell
New-GitHubRelease -Repository $product -Name "$product $version" -Tag $version
```

This is enough to create new release on GitHub. You can find it on url like this:
`https://github.com/$githubUser/$product/releases/tag/$version`

But how about the actual release, those zips we created? Cmdlet `New-GitHubRelease` also has parameter `-Asset`! An asset is a Powershell hashtable with two keys - `Path` and `Content-Type` (mind the hyphen!). The parameter accepts array of those.

So for example, this is how it could look like:

```powershell
$assets = @()
$asset1 = @{
        "Path"=$zipFile1;
        "Content-Type"="application/zip";
}
$asset2 = @{
        "Path"=$zipFile2;
        "Content-Type"="application/zip";
}
$assets += $asset1
$assets += $asset2

New-GitHubRelease -Repository $product -Name "$product $version" -Tag $version -Asset $assets
```

But using our releases array from before, we got this:

```powershell
$version = "0.5-beta"
$product = "MyApp"

$assets = @()
$runtimes | %{
    $runtime = $_
    $toPack = ("{0}\{1}" -f $output,$runtime)
    $fileName = "{0}.{1}.{2}" -f $product,$version,$runtime #MyApp.0.5-beta.win-x86
    $zipFile = ("{0}\{1}.zip" -f $output,$fileName)
    Compress-Archive -Path $toPack -DestinationPath $zipFile
    $asset = @{
        "Path"=$zipFile;
        "Content-Type"="application/zip";
        Name = $fileName # save this for later
    }
    $assets += $asset
}

Set-GitHubSessionInformation -Username githubUser 
New-GitHubRelease -Repository $product -Name "$product $version" -Tag $version -Asset $assets
```


That wraps it up. The whole script looks like this:




```powershell
$version = "0.5-beta"
$product = "MyApp"
$project = "C:\path\to\apps\project.csproj"
$output = "C:\path\to\releases"
$dotnet = "C:\Program Files\dotnet\dotnet.exe"
$runtimes = @(
"win10-x64"
"win10-x86"
"win-x64"
"win-x86"
"osx-x64"
"linux-x64"
)

# clear previous releases
Remove-Item "$output\*" -Recurse -Confirm:$true

$runtimes | %{
    & $dotnet publish $project -c release -r $_  -o ("{0}\{1}" -f $output,$_)
}

$assets = @()

$runtimes | %{
    $runtime = $_
    $toPack = ("{0}\{1}" -f $output,$runtime)
    $fileName = "{0}.{1}.{2}" -f $product,$version,$runtime
    $zipFile = ("{0}\{1}.zip" -f $output,$fileName)
    Compress-Archive -Path $toPack -DestinationPath $zipFile
    $asset = @{
        "Path"=$zipFile;
        "Content-Type"="application/zip";
        Name = $fileName
    }
    $assets += $asset
}

$tmp = Set-GitHubSessionInformation -Username githubUser
$release = New-GitHubRelease -Repository $product -Name "$product $version" -Tag $version -Asset $assets -Verbose
Write-Information "Path to release: $($release.html_url)"

# show me
Start-Process $release.html_url
```



I am using this now and it does its work sufficiently. However, next I will take a look at **PSake**, I think it may help make this script more structured and it is based on standard approach to this problem, which  would be better than homegrown approach. 