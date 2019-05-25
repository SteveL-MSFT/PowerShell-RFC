---
RFC: RFCxxxx
Author: Steve Lee
Status: Draft
SupercededBy: N/A
Version: 0.1
Area: PowerShellGet
Comments Due: 7/30
Plan to implement: Yes
---

# PowerShellGet 3.0 Module

PowerShellGet has enabled an active community to publish and install PowerShell
Modules.
However, it has some architectural and design limitations that prevent it from
moving forward more quickly.
This RFC proposes some significant changes to address this.

## Motivation

    As a PowerShell user,
    I can discover how to install missing cmdlets,
    so that I can be more productive.

    As a PowerShellGet contributor,
    I can more easily contribute code,
    so that PowerShellGet evolves more quickly.

## User Experience

```powershell
PS> Update-PSResourceCache
PS> Get-Satisfaction
Get-Satisfaction : The term 'Get-Satisfaction' is not recognized as the name of a cmdlet, function, script file, or operable program.
Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
At line:1 char:1
+ Get-Satisfaction
+ ~~~
+ CategoryInfo          : ObjectNotFound: (Get-Satisfaction:String) [], CommandNotFoundException
+ FullyQualifiedErrorId : CommandNotFoundException

Suggestion: You can obtain `Get-Satisfaction` by running `Install-PSResource HappyDays`.

PS> Install-PSResource HappyDays
```

## Specification

### Rewrite of PowerShellGet

PowerShellGet is currently written as PowerShell script with a dependency on PackageManagement.
Proposal is to write PSResource in C# to reduce complexity and make easier to maintain.
In addition, remove dependency on PackageManagement completely as well as dependency on
nuget.exe.
This module will only use REST APIs to get nupkgs and publishing nupkgs.

### Side-by-side with PowerShellGet

To ease transition due to the enormity of the breaking change, this module will
have a different names for the cmdlets to allow for side by side use
with the existing PowerShellGet module.
Since the current PowerShellGet 2.x version is a script module and this new one
is a C# based module, they can coexist side-by-side.

### Cmdlets

### Local cache

Rather than connecting to PSGallery and other repositories for finding modules,
the user must explicitly request the current metadata of
modules from the repositories and that is cached locally.
`Find-PSResource` (see below) works against this local cache.
This will also enable changes in PowerShell to use `Find-PSResource -Type Command` to look
in this cache when it can't find a command and suggest to the user the module to install to
get that command.
This will be a local json file containing only the latest version of each module.
The cache will be stored in the user path.
There is no system cache that is shared.

### Updating the cache

A `Update-PSResourceCache` cmdlet will download the json file from the registered
repositories.
A hash check is performed to determine if the current cache needs to be updated.
After the update is complete, it will compare installed versions with the cache
and list out modules that are updatable.

### Repository management

`Register-PSResourceRepository` will allow for registering additional repositories.
A `-Default` switch enables registering PSGallery should it be accidentally removed.
The `-URL` will accept the HTTP address without the need to specify `/api/v2` as
that will be assumed and discovered at runtime.
A `-InstallationPolicy` switch accepts `Trusted` or `Untrusted` (default) indicating
whether to prompt the user when installing resources from that repository.

`Get-PSResourceRepository` will list out the registered repositories.

`Set-PSResourceRepository` can be used to update a repository URL or installation policy.

`Unregister-PSResourceRepository` can be used to un-register a repository.

### Finding resources

Current design of PowerShellGet is to have different cmdlets for each of the
different resources types that it supports:

`Find-Command`
`Find-DscResource`
`Find-Module`
`Find-RoleCapability`
`Find-Script`

Instead, these will be abstracted by a `Find-PSResource` cmdlet.
A `-Type` parameter will accept an array of types to allow filtering.

With support of the generic `PSResource`, this means we can also find and
install arbitrary nupkgs that may only contain an assembly the user wants to
use in their script.

If the local cache does not exist, the cmdlet will call `Update-PSResourceCache`
first and then attempt a search.
However, this cmdlet does not update the cache if one exists.

### Installing resources

`Install-PSResource` will only work for modules.
A `-AllowPrerelease` switch allows installing prerelease versions.
Other types will use `Save-PSResource` (see below).
A `-Repository` parameter accepts a specific repository name.
If `-Repository` is not specified and the same resource is found in multiple
repositories, then the resource is automatically installed quietly from the
trusted repository with the highest version matching the `-Version` parameter
(if specified, otherwise newest non-prerelease version unless `-AllowPrerelease`
is used).
If there are no trusted repositories matching the query, then the newest version
fulfilling the query will be prompted to be installed.
`-AllowUntrusted` can be used to suppress being prompted for untrusted sources.
`-AllowDifferentPublisher` can be used to suppress being prompted if the publisher
of the module is different from the currently installed version.
`-AllowClobber` can be used to re-install a module.

### Dependencies and version management

`Install-PSResource` can accept a path to a psd1 file (using `-RequiredModulesFile`)
or a hashtable (using `-RequiredModules`) where the key is the module name and the
value is either the required version specified using Nuget version range syntax or
a hash table where `repository` is set to the URL of the repository and
`version` contains the Nuget version range syntax.

```powershell
Install-PSResource -RequiredModules @{
  "Configuration" = "[1.3.1,2.0)"
  "Pester"        = @{
    version = "[4.4.2,4.7.0]"
    repository = "https://www.powershellgallery.com"
  }
}
```

### Saving resources

With the removal of PackageManagement, there is still a need to support saving
arbitrary nupkgs used for scripts.

`Save-PSResource -Type Library` will download nupkgs that have a `lib` folder.
A `-AllowPrerelease` switch allows saving prerelease versions.
The dependent native library in `runtimes` matching the current system runtime
will be copied to the same destination.

This cmdlet has the same parameters as `Install-PSResource` with the addition
of mandatory `-Path` or `-LiteralPath` parameters.

### Updating resources

`Update-PSResource` will update all resources to most recent minor version by
default.
A `-AllowMajorVersionChange` switch will allow updating to newer major versions.
If the installed resource is a pre-release, it will automatically update to
latest prerelease or stable version (if available).

### Publishing resources

`Publish-PSResource` will supporting publishing modules and scripts.
Modules will be given as a path to the module folder.
Publishing by name and version is no longer supported.

### Listing installed resources

`Get-PSResource` will list all installed resources with new `Type` column.

```output
Version  Name      Type    Repository  Description
-------  ----      ----    ----------  -----------
5.0.0    VSTeam    Module  PSGallery   Adds functionality for working with Visual …
0.14.94  PSGitHub  Module  PSGallery   This PowerShell module enables integration …
0.7.3    posh-git  Module  PSGallery   Provides prompt with Git status summary …
```

## Alternate Proposals and Considerations

None
