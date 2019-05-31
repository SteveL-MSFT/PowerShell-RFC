---
RFC: RFCxxxx
Author: Steve Lee
Status: Draft
SupercededBy: N/A
Version: 0.1
Area: Security
Comments Due: June 30th, 2019
Plan to implement: Yes, PS7
---

# Simplify and secure use of credentials in scripts

Advanced scripts that touch many systems require multiple credentials and
types of credentials particularly when orchestrating across different clouds.
Best practice is to not hard code any credentials in scripts, but currently
this requires custom code on different platforms to handle this securely.

This RFC proposes new cmdlets to make this simpler and secure.

## Motivation

> As a PowerShell script author,
> I can securely use multiple credentials,
> so that I can automate complex orchestration across multiple remote resources.

## User Experience

Registering credentials for later use:

```powershell
Install-Module AzVault
New-CredentialVault -Name Azure -Extension AzVault -Credential $azCred
$cred = Get-Credential -AccessToken $token
Register-Credential -Credential $cred -Name AzDevOps -Vault Azure
$azDevOpsCred = Get-Credential -Name AzDevOps
Invoke-RestMethod https://dev.azure.com/... -Credential $azDevOpsCred -Authentication Basic
```

## Specification

### Credential Vaults

There are two types of vaults: local and remote.
A local vault, by default, is already created and named `Default`.
`Default` on Windows is [CredMan](https://docs.microsoft.com/en-us/windows/desktop/SecAuthN/credentials-management).
`Default` on macOS is [KeyChain](https://developer.apple.com/documentation/security/keychain_services).

>[!NOTE]
>KeyChain support is unlikely to be available in the first release of this feature.

`Default` on Linux is [GNOME Keyring](https://wiki.gnome.org/Projects/GnomeKeyring/).

>[!NOTE]
>Linux has many options for local credential management, additional support may be
>added in the future or via community provided extensions.

### Credential Vault Extensions

Vault extensions whether they are local or remote are provided as modules.

Extensions need to expose a `Get-Credential` cmdlet that accepts an optional `-Credential`
for authentication.
Expectation is that each module will have their own cmdlet and not literally
named `Get-Credential`.

Extensions register themselves by calling `Register-CredentialVaultExtension`
passing the name of the relevant cmdlet to `-Cmdlet`, and the fully qualified
module name and version to `-Module` as `@{modulename="name";moduleversion="x.y.z"}`.
This can be done as part of importing the module.

`Register-CredentialVaultExtension` has a `-Default` switch to register the
default local vault (if applicable, see above).

`Unregister-CredentialVaultExtension` should be used to remove any existing
registration or when updating a registration to a new version.

`Get-CredentialVaultExtension` will enumerate registered extensions returning
the module name, module version, and cmdlet used.

### Authorization

Access to the credential vault is always using the current process security context.
In the case of remote vaults, the remote credential is stored within the local
default vault and retrieved as needed when accessing secrets from that remote
vault.

### PSCredential Changes

Secrets supported will be:

- Username/password
- Personal Access Token

## Alternate Proposals and Considerations

In this release, the following are non-goals:

- Provision to rotate certs/access tokens
- Sharing local vaults across different computer systems
- Sharing local vault across different user contexts
- Support for other types of secrets
