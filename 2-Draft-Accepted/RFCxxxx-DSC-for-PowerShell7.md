---
RFC: RFCxxxx
Author: Steve Lee
Status: Draft
SupercededBy: N/A
Version: 1.0
Area: Desired State Configuration
Comments Due: Nov 22, 2020
Plan to implement: Yes
---

# DSC for PowerShell 7

[Desired State Configuration](https://docs.microsoft.com/powershell/scripting/dsc/overview/overview) (DSC) is a platform in PowerShell
enabling configuration as code.

DSC contains many components including the `configuration` keyword that is part of the PowerShell engine and compilation to
a configuratoin MOF file, DSC resources in Windows and on PowerShellGallery.com, the Local Configuration Manager in Windows,
Pull Server support, and PSDesiredStateConfiguration module.

Currently, DSC (for Windows PowerShell) is tightly coupled with WMI on Windows and OMI on Linux.
WMI is used to host the Local Configuration Manager (LCM) and WMI APIs are used to parse MOF which is used for both
configuration files as well as schema files.

In addition, it is not possible to write DSC resources that are cross platform compatible nor easily author DSC resources on Linux as
they need to be in C++ or Python.
Finally, third party solutions trying to leverage DSC resources need to still go through the LCM making it complicated to integrate with
their agents or ecosystems.

Unlike DSC for Windows PowerShell, DSC for PowerShell 7 is only intended to provide a platform for building or integrating with configuration solutions.
This means for PowerShell 7, the scope of DSC support is limited to `configuration` keyword and compliation, authoring and using
DSC resources via the PSDesiredStateConfiguration module.

## Motivation

    As a developer of configuration solutions,
    I can easily leverage DSC resources,
    so that my solution can leverage the PowerShell ecosystem and its users and contributors.

    As an ITPro,
    I can directly invoke DSC resources,
    so that I can build ad hoc cross platform configuration.

## Invoke-DscResource

About a year ago, we worked with [Gael Colas](https://twitter.com/gaelcolas) to author a RFC describing fundamental changes to 
[`Invoke-DscResource`](https://docs.microsoft.com/powershell/module/psdesiredstateconfiguration/invoke-dscresource?view=powershell-7) in PowerShell 7.0.
This enabled executing the Get, Set, and Test methods on DSC resources bypassing the LCM (Local Configuration Manager) on Windows and removing the
need for OMI ([Open Management Infrastructure](https://github.com/Microsoft/omi)) on Linux.
This was the start to enable cross platform DSC resources to be authored in PowerShell script (with some limitations).

This feature was shipped as an [experimental feature](https://docs.microsoft.com/powershell/scripting/learn/experimental-features?view=powershell-7) in PowerShell 7.0.

### Bring your own agent

A significant change from existing DSC for Windows PowerShell is there will be no default agent (LCM) shipped with PowerShell 7.

Instead, we want to make it easier to integrate with existing configuration agents like [Ansible](https://docs.ansible.com/ansible/latest/user_guide/windows_dsc.html),
[Chef](https://docs.chef.io/resources/dsc_resource/), [Puppet](https://forge.puppet.com/puppetlabs/dsc),
[Azure Policy Guest Configuration](https://docs.microsoft.com/azure/governance/policy/concepts/guest-configuration), and other popular configuration solutions.

Custom agents can simply use the existing PowerShell API (or call out to `pwsh`) to directly invoke DSC resources.

You can even simply write a PowerShell script as your agent that runs off TaskScheduler or cron, for example.

Along with the lack of a default agent, there is no pre-defined push or pull model for deploying DSC.

### Cross platform first

DSC has supported both Windows and Linux, however, the [Linux support](https://docs.microsoft.com/powershell/scripting/dsc/getting-started/lnxGettingStarted?view=powershell-7)
was dependent on [OMI](https://github.com/Microsoft/omi) and did not support authoring Linux DSC resources in PowerShell script making development
inconsistent and much more difficult.

DSC for PowerShell 7 will be cross platform having consistent capabilities and user experience on all platforms.

### Moving away from WMI/OMI/MOF

DSC on Windows was originally designed relying on [WMI](https://docs.microsoft.com/windows/win32/wmisdk/wmi-start-page) as a host on Windows,
OMI as a host on Linux, and [MOF](https://docs.microsoft.com/windows/win32/wmisdk/managed-object-format--mof-) for configuration and schema.

MOF as a format is not something most developers are familiar with and not much tooling support.
However, developers of DSC resources written in PowerShell script still needed to learn MOF and also keep their MOF schema
in sync with their script implementation.

Today, when you execute a [DSC configuration script](https://docs.microsoft.com/powershell/scripting/dsc/configurations/configurations?view=powershell-7),
the output is a MOF configuration file.
In general, users do not need to read this MOF file as it's intended to be used by machines.
However, MOF parsing creates a dependency on WMI/OMI.
Instead, we will move towards a JSON configuration file format.

Since there is no default agent to understand the JSON configuration file, it will be up to users or 3rd party agents to understand the configuration file.

Moving from WMI/OMI also means no longer supporting C++ resources nor Python based resources.

<TODO> example JSON configuration </TODO>

### No more MOF schema and script resources

Although DSC resources can be written as WMI/OMI providers, this requires writing C/C++ code and compiling for specific Linux distros and processor architectures.
Writing PowerShell script DSC resources currently still requires a matching schema file making it more complex.
Instead, the current plan is to [only support PowerShell class based DSC resources](https://github.com/PowerShell/PowerShell/issues/13731) in DSC for PowerShell 7.
Whereas previously embedded objects were CIMInstance types, they would now be .NET types based on PowerShell classes.
Additonal reasoning and discussion is in the issue linked above.

### Open Source

The current [PSDesiredStateConfiguration](https://docs.microsoft.com/powershell/module/psdesiredstateconfiguration/?view=powershell-7) module is not Open Source.
The intent is to clean up the code and prepare the GitHub repository to be made public as an Open Source project.

### PowerShell engine changes

To enable DSC for PowerShell 7 to evolve independently to PowerShell 7 itself, the code in the engine that handles the DSC configuration keyword will
be seperated as a plugin and made to be part of the PSDesiredStateConfiguration module.

## Alternate Proposals and Considerations

### Supporting existing script based resources

Existing script based resources (not class based) require a MOF schema file for PowerShell to understand the types and nested types.
We had prototyped using a JSON based schema instead of the existing MOF schema file, but it still required the DSC resource author to
create a schema file and ensure it aligns with the script implementation.

It is simpler to create a PowerShell class based resource which itself is the schema removing the need to understand and maintain a
separate schema file.
In addition, having a single model (class based) for authoring DSC resources will help the community in the long term by having a single
consistent way to write DSC resources making it easier for new authors to find [documentation and examples](https://dsccommunity.org/blog/class-based-dsc-resources/).
