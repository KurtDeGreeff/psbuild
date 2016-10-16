psbuild
=======

[![Build status](https://ci.appveyor.com/api/projects/status/k7p2m9b6h5m9w2q3/branch/master)](https://ci.appveyor.com/project/sayedihashimi/psbuild/branch/master)

The main purpose of this project to provide a better experience calling ```msbuild.exe``` from PowerShell. When using psbuild by default you'll get:

 - the latest version of msbuild.exe on your machine
 - multi-core build
 - log files, three formats: detailed, diagnostic and markdown
 - 32 bit version of ```msbuild.exe```. Using the 64 bit version accidently can cause issues

It also simplifies passing in properties, targets by handling the work to translate PowerShell syntax to the
call to ```msbuild.exe```.

psbuild also has some functionality that you can use to create and edit MSBuild files from PowerShell.

To see the full set of commands that psbuild makes available just execute.

<code>Get-Command -Module psbuild</code>

## Getting Started

##### download and install psbulid
<code style="background-color:grey">(new-object Net.WebClient).DownloadString("https://raw.githubusercontent.com/ligershark/psbuild/master/src/GetPSBuild.ps1") | iex</code>

##### build an msbuild file
<code>Invoke-MSBuild C:\temp\msbuild\msbuild.proj</code>

##### build a file and specify Configuration, Platform and VisualStudioVersion

psbuild has first class support for some well known properties. Configuration, Platform and VisualStudioVersion are just a few.

<code>Invoke-MSBuild C:\temp\msbuild\msbuild.proj -configuration Release -platform 'Mixed Platforms' -visualStudioVersion 14.0</code>

#### build a file passing arbitrary properties

To pass in properties that psbuild doesn't have first class support just use the ```-properties``` parameter.

<code>Invoke-MSBuild C:\temp\msbuild\msbuild.proj -properties @{'MyProperty01'='myp1';'MyProperty02'='myp2'}</code>

##### build an msbuild file and execute a specific target

<code>Invoke-MSBuild C:\temp\msbuild\msbuild.proj -targets Demo</code>

##### build an msbuild file and execute multiple targets

<code>Invoke-MSBuild C:\temp\msbuild\msbuild.proj -targets Build,Demo</code>

##### how to get the log file for the last build

When calling ```Invoke-MSBuild``` log files will be written by default in a temp folder. You can access those
log files using the ```Open-PSBuildLog``` after the build completes. There are three log files by
default: detailed, diagnostic and markdown.

```powershell
PS> Invoke-MSBuild C:\temp\msbuild\proj1.proj
# returns the detailed log in the default editor
PS> Open-PSBuildLog

# returns the log in markdown format
PS> Open-PSBuildLog markdown

# returns the diagnostic
PS> Open-PSBuildLog diagnostic
```

#### how to open the log directory

To see logs from previous builds use the ```Open-PSBuildLogDirectory``` command.
In this directory you'll find a folder for each project, each with log files.

#### How to pass extra arguments to msbuild.exe

When you call ```Invoke-MSBuild``` the call to ```msbuild.exe``` will be constructed for you. If you need to add
additional arguments to ```msbuild.exe``` you can use the ```-extraArgs``` parameter. For example if you wanted
to attach a custom logger or write a log file to a specific location.

<code>Invoke-MSBuild C:\temp\msbuild\msbuild.proj -extraArgs '/flp3:v=d;logfile="C:\temp\msbuild\msbuild.detailed.log"'</code>

#### show msbuild reserved properties

When authoring MSBuild files you'll often need to use some of MSBuild's
[reserved properties](https://msdn.microsoft.com/en-us/library/ms164309.aspx). You can either look this
up on the web, or use psbuild to give you the info.

<code>Get-MSBuildReservedProperties</code>

This will display the list of known reserved properties and their values.

#### show common msbuild escape characters

When authoring MSBuild files there are a few [special characters](https://msdn.microsoft.com/en-us/library/bb546106.aspx)
that you'll need to escape. Instead of searching the web for the result you can simply call this.

<code>Get-MSBuildEscapeCharacters</code>

##### You can also create a new MSBuild file with the following

When creating a new MSBuild file from scratch most people copy an existing one and remove the contents. psbuild
offers a command to enable you to easily create a new empty MSBuild project file.

<code>New-MSBuildProject -filePath C:\temp\msbuild\fromps.proj</code>

##### to see what commands are available
<code>Get-Command -Module psbuild</code>

Most functions have help defined so you can use ```get-help``` on most commands for more details.

#### How to add psbuild to your build script.

If you're automating your build process then you should make sure they are portable.
You can add the function below to your build script, and call it before using psbuild.
If psbuild is not already installed it will be downloaded.

```powershell
<#
.SYNOPSIS
    You can add this to you build script to ensure that psbuild is available before calling
    Invoke-MSBuild. If psbuild is not available locally it will be downloaded automatically.
#>
function EnsurePsbuildInstlled{
    [cmdletbinding()]
    param(
        [string]$psbuildInstallUri = 'https://raw.githubusercontent.com/ligershark/psbuild/master/src/GetPSBuild.ps1'
    )
    process{
        if(-not (Get-Command "Invoke-MsBuild" -errorAction SilentlyContinue)){
            'Installing psbuild from [{0}]' -f $psbuildInstallUri | Write-Verbose
            (new-object Net.WebClient).DownloadString($psbuildInstallUri) | iex
        }
        else{
            'psbuild already loaded, skipping download' | Write-Verbose
        }

        # make sure it's loaded and throw if not
        if(-not (Get-Command "Invoke-MsBuild" -errorAction SilentlyContinue)){
            throw ('Unable to install/load psbuild from [{0}]' -f $psbuildInstallUri)
        }
    }
}
```

## Debug mode
In many cases after a build it would be helpful to be able to answer questions like the following.
 
 - What is the value of `x` property?
 - What is the value of `y` property?
 - What would the expression ```'@(Compile->'%(FullPath)')``` be?

But when you call msbuild.exe the project that is built is created in memory and trashed at the end of the
process. ```Invoke-MSBuild``` now has a way that you can invoke your build and then have a _"handle"_ to your
project that was built. Using this object you can evaluate the properties and items. To enable this you just need to
pass in the ```-debugMode``` switch to ```Invoke-MSBuild``` (_Note: this is actively under development so if you
run into an problems please open an issue_). Here are some examples of what you can do.

```powershell
PS> $bResult = Invoke-MSBuild .\temp.proj -debugMode

PS> $bResult.EvalProperty('someprop')
default

PS> $bResult.EvalItem('someitem')
temp.proj

PS> $bResult.ExpandString('$(someprop)')
default

PS> $bResult.ExpandString('@(someitem->''$(someprop)\%(Filename)%(Extension)'')')
default\temp.proj
```

You can get full access to the [ProjectInstance](http://msdn.microsoft.com/en-us/library/microsoft.build.execution.projectinstance(v=vs.121).aspx)
object with the ProjectInstance property.

More functionality is available via the ProjectInstance object.

```powershell
PS> $bResult.ProjectInstance.GetItems('someitem').EvaluatedInclude
temp.proj
```

You can get the [BuildResuilt](http://msdn.microsoft.com/en-us/library/microsoft.build.execution.buildresult(v=vs.121).aspx)
via the BuildResult parameter.

```powershell
PS> $bResult.BuildResult.OverallResult
Failure
```

# Reporting Issues

To report any issues please [create an new item](https://github.com/ligershark/psbuild/issues/new) on the [issues page](https://github.com/ligershark/psbuild/issues/).

# Release Notes

- Made psbuild [self contained](https://github.com/ligershark/psbuild/issues/76) so that nothing needs to be downloaded at build time
- Added Get-PSBuildVersion method
- Update on how [secrets are masked](https://github.com/ligershark/psbuild/commit/ce2386b46aa6884e3f023ca7d4b1e32ec32712e3). The previous impl caused the users prompt to be modified.
- Update to [only add quotes when needed](https://github.com/ligershark/psbuild/issues/76)
- Fixed [an issue](https://github.com/ligershark/psbuild/issues/76) where commands were being quoted unnecessarily
- Update to allow overriding settings with env variables https://github.com/ligershark/psbuild/commit/b00cc3c63a81c20f9101cbd111d1b1e1bffe874a
- Update to ensure log directory is unique for every project built https://github.com/ligershark/psbuild/issues/71.
- Update to filter secrets from log files. https://github.com/ligershark/psbuild/issues/59.
- Added ```Import-FileReplacer``` to load [```file-replacer```](https://github.com/ligershark/template-builder/blob/master/file-replacer.psm1). This is a PowerShell module that can be used to replace text in files.
- Added ```Import-NuGetPowershell``` to load [```nuget-powershell```](https://github.com/ligershark/nuget-powershell)
- Added a module manifest to support versioning https://github.com/ligershark/psbuild/issues/67
- Added ```Open-PSBuildLogDirectory``` to open the log directory https://github.com/ligershark/psbuild/issues/66
- Updated ```New-MSBuildProject``` to create files with the highest ```ToolsVersion``` on the box. Also added a ```-toolsVersion``` parameter.
- Updated ```Invoke-MSBuild``` to not require targets when passing in ```-debugMode```.
- Added a function, Import-Pester, to get and load [pester](https://github.com/pester/Pester). If pester is not installed it will be downloaded. See https://github.com/ligershark/psbuild/issues/56.
- Update to filter secrets in PowerShell output. When passing ```-password``` the value will automatically be masked. You can also add additional values to be masked. For more info see ```Get-Help Get-FilteredString``` or ```Get-Help Invoke-MSBuild -Examples```. See https://github.com/ligershark/psbuild/issues/57.
- Update to add entries to AppVeyor messages for projects that are built. See https://github.com/ligershark/psbuild/issues/56.
- Updates to properly handle properties which have spaces. See https://github.com/ligershark/psbuild/issues/49.
- Update to add ```-Platform``` as a paramter to Invoke-MSBuild
- Added ```Invoke-CommandString``` which is used to call external .exe files. It is exported as well so can be used to simplify calling .exe files by users.
- Added a parameter ```noLogFiles``` to ```Invoke-MSBuild``` which will disable writing log files. See https://github.com/ligershark/psbuild/issues/52.
- Added ```psbuild``` alias for ```Invoke-MSBuild```.
- Added ```-bitness``` parameter to ```Invoke-MSBuild``` so that you can pick either 32 or 64 bit msbuild
- Update to select 32 bit msbuild.exe when running ```Invoke-MSBuild```
- Update to throw an exception when the exit code from msbuild.exe is non-zero.
- Update to add parameters to ```Open-PSBuildLog``` to specify the type of log to be opened.
- Added a Markdown logger
- Added ```-whatif``` support to ```Invoke-MSBuild```

# Contributing
Contributing is pretty simple. The project mostly consists of one .psm1 file located at ```/src/psbuild.psm1```.
You should send PRs to the ```dev``` branch. If it's a simple bug fix feel free to go ahead and submit the fix
as a PR. If you have a feature please propose it in the [issues](https://github.com/ligershark/psbuild/issues)
section so that we can dicsuss your idea.

# Credits

This project uses the following open source components.

- [pester](https://github.com/pester/Pester) - Apache v2 ([link](https://github.com/pester/Pester/blob/master/LICENSE)) 
- [MarkdownLog](https://github.com/Wheelies/MarkdownLog) - MIT License ([link](https://github.com/Wheelies/MarkdownLog/blob/master/LICENSE))
