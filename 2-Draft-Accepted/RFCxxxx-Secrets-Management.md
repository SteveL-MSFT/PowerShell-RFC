---
RFC: RFCxxxx
Author: Steve Lee
Status: Draft
SupercededBy: N/A
Version: 0.1
Area: Security
Comments Due: July 31st, 2019
Plan to implement: Yes, PS7
---

# Simplify and secure use of secrets in scripts

Advanced scripts that touch many systems require multiple secrets and
types of secrets particularly when orchestrating across different clouds.
Best practice is to not hard code any secrets in scripts, but currently
this requires custom code on different platforms to handle this securely.

This RFC proposes new cmdlets to make this simpler and secure.

## Motivation

> As a PowerShell script author,
> I can securely use multiple secrets,
> so that I can automate complex orchestration across multiple remote resources.

## User Experience

Registering secrets for later use:

```powershell
# Install the Az.KeyVault module to be used in this example.
# This module doesn't conform to the extensions model so would need to be
# updated to actually work.
Install-Module Az.KeyVault

# In this example, we explicitly register this extension
Register-SecretsVaultExtension -Name Azure -Cmdlet Get-AzKeyVaultSecret -Module Az.KeyVault -Version 1.0.0

# When using remote vaults, expectation is that secrets are
# stored using their tooling, so this module is only for retrieving secrets
# from remote vaults
$azDevOpsToken = Get-Secret -Vault Azure -Name AzDevOps
Invoke-RestMethod https://dev.azure.com/... -Credential $azDevOpsToken -Authentication Basic

# For local vault, we can register custom secrets
# In this example, we store a PSCredential object
Register-Secret -Secret $cred -Name MyCreds
New-PSSession -ComputerName myServer -Credential (Get-Secret -Name MyCreds)
```

## Specification

### Secrets Vault

There are two types of vaults: local and remote.
A local vault, by default, is already created and named `Default`.
`Default` on Windows is [CredMan](https://docs.microsoft.com/en-us/windows/desktop/SecAuthN/credentials-management).
`Default` on macOS is [KeyChain](https://developer.apple.com/documentation/security/keychain_services).

>[!NOTE]
>KeyChain support is unlikely to be available in the first release of this feature.

`Default` on Linux is [Hashicorp Vault](https://www.vaultproject.io/).

>[!NOTE]
>Linux has many options for local credential management, additional support may be
>added in the future or via community provided extensions.

### Credential Vault Extensions

Vault extensions whether they are local or remote are provided as modules.

Extensions are modules that provide a `Get-Secret` cmdlet that accepts a
`-Credential` or `-AccessToken` parameter for authentication.
Expectation is that each module will have their own cmdlet and not literally
named `Get-Secret`.

Extensions register themselves by calling `Register-SecretsVaultExtension`
passing the name of the relevant cmdlet to `-Cmdlet`, the module name to `-Module`,
and the module version to `-ModuleVersion`.
This can be done as part of importing the module or explicitly by the user.

`Register-SecretsVaultExtension` has a `-Default` switch to register the
default local vault (if applicable, see above) if it has been unregistered.

`Unregister-SecretsVaultExtension` should be used to remove any existing
registration or when updating a registration to a new version.

`Get-SecretsVaultExtension` will enumerate registered extensions returning
the module name, module version, and cmdlet used.

### Authorization

Access to the credential vault is always using the current process security context.
In the case of remote vaults, the remote credential is stored within the local
default vault and retrieved as needed when accessing secrets from that remote
vault.

### Secrets Supported

Secrets supported will be:

- PSCredential
- AccessToken as string

## Alternate Proposals and Considerations

In this release, the following are non-goals:

- Provision to rotate certs/access tokens
- Sharing local vaults across different computer systems
- Sharing local vault across different user contexts
- Support for other types of secrets
