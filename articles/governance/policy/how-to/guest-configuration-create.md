---
title: How to create Guest Configuration policies
description: Learn how to create an Azure Policy Guest Configuration policy for Windows or Linux VMs.
author: DCtheGeek
ms.author: dacoulte
ms.date: 07/26/2019
ms.topic: conceptual
ms.service: azure-policy
manager: carmonm
---
# How to create Guest Configuration policies

Guest Configuration uses a [Desired State Configuration](/powershell/dsc) (DSC) resource module to
create the configuration for auditing of the Azure virtual machines. The DSC configuration defines
the condition that the virtual machine should be in. If the evaluation of the configuration fails,
the Policy effect **audit** is triggered and the virtual machine is considered **non-compliant**.

[Azure Policy Guest Configuration](/azure/governance/policy/concepts/guest-configuration)
can only be used to audit settings inside virtual machines. Remediation
of settings inside virtual machines isn't yet available.

Use the following actions to create your own configuration for validating the state of an Azure
virtual machine.

> [!IMPORTANT]
> Custom policies with Guest Configuration is a Preview feature.

## Add the GuestConfiguration resource module

To create a Guest Configuration policy, the resource module must be added. This resource module can
be used with locally installed PowerShell, with [Azure Cloud Shell](https://shell.azure.com), or
with the [Azure PowerShell Docker image](https://hub.docker.com/rsdk-powershell/).

### Base requirements

The Guest Configuration resource module requires the following software:

- PowerShell. If it isn't yet installed, follow [these instructions](/powershell/scripting/install/installing-powershell).
- Azure PowerShell 1.5.0 or higher. If it isn't yet installed, follow [these instructions](/powershell/azure/install-az-ps).

### Install the module

Guest Configuration uses the **GuestConfiguration** resource module for creating DSC configurations
and publishing them to Azure Policy:

1. From a PowerShell prompt, run the following command:

   ```azurepowershell-interactive
   # Install the Guest Configuration DSC resource module from PowerShell Gallery
   Install-Module -Name GuestConfiguration
   ```

1. Validate that the module has been imported:

   ```azurepowershell-interactive
   # Get a list of commands for the imported GuestConfiguration module
   Get-Command -Module 'GuestConfiguration'
   ```

## Create custom Guest Configuration configuration

The first step to creating a custom policy for Guest Configuration is to create the DSC
configuration. For an overview of DSC concepts and terminology, see [PowerShell DSC Overview](/powershell/dsc/overview/overview).

### Custom Guest Configuration configuration on Linux

The DSC configuration for Guest Configuration on Linux uses the `ChefInSpecResource` resource to
provide the engine the name of the [Chef InSpec](https://www.chef.io/inspec/) definition. **Name**
is the only required resource property.

The following example creates a configuration named **baseline**, imports the **GuestConfiguration**
resource module, and uses the `ChefInSpecResource` resource set the name of the InSpec definition to
**linux-patch-baseline**:

```azurepowershell-interactive
# Define the DSC configuration and import GuestConfiguration
Configuration baseline
{
    Import-DscResource -ModuleName 'GuestConfiguration'

    ChefInSpecResource 'Audit Linux patch baseline'
    {
        Name = 'linux-patch-baseline'
    }
}

# Compile the configuration to create the MOF files
baseline
```

For more information, see [Write, Compile, and Apply a Configuration](/powershell/dsc/configurations/write-compile-apply-configuration).

### Custom Guest Configuration configuration on Windows

The DSC configuration for Azure Policy Guest Configuration is used by the Guest Configuration agent
only, it doesn't conflict with Windows PowerShell Desired State Configuration.

The following example creates a configuration named **AuditBitLocker**, imports the
**GuestConfiguration** resource module, and uses the `Service` resource to audit for a running
service:

```azurepowershell-interactive
# Define the DSC configuration and import GuestConfiguration
Configuration AuditBitLocker
{
    Import-DscResource -ModuleName 'PSDscResources'

    Service 'Ensure BitLocker service is present and running'
    {
        Name = 'BDESVC'
        Ensure = 'Present'
        State = 'Running'
    }
}

# Compile the configuration to create the MOF files
AuditBitLocker
```

For more information, see [Write, Compile, and Apply a Configuration](/powershell/dsc/configurations/write-compile-apply-configuration).

## Create Guest Configuration custom policy package

Once the MOF is compiled, the supporting files must be packaged together. The completed package is
used by Guest Configuration to create the Azure Policy definitions. The package consists of:

- The compiled DSC configuration as a MOF
- Modules folder
  - GuestConfiguration module
  - DscNativeResources module
  - (Linux) A folder with the Chef InSpec definition and additional content
  - (Windows) DSC resource modules that aren't built in

The `New-GuestConfigurationPackage` cmdlet creates the package. The following format is used for
creating a custom package:

```azurepowershell-interactive
New-GuestConfigurationPackage -Name '{PackageName}' -Configuration '{PathToMOF}' `
    -Path '{OutputFolder}' -Verbose
```

Parameters of the `New-GuestConfigurationPackage` cmdlet:

- **Name**: Guest Configuration package name.
- **Configuration**: Compiled DSC configuration document full path.
- **Path**: Output folder path. This parameter is optional. If not specified, the package is created
  in current directory.
- **ChefProfilePath**: Full path to InSpec profile. This parameter is supported only when creating
  content to audit Linux.

The completed package must be stored in a location that is accessible by the managed virtual
machines. Examples include GitHub repositories, an Azure Repo, or Azure storage. If you prefer to
not make the package public, you can include a [SAS token](../../../storage/common/storage-dotnet-shared-access-signature-part-1.md)
in the URL. You could also implement
[service endpoint](../../../storage/common/storage-network-security.md#grant-access-from-a-virtual-network)
for virtual machines in a private network, although this configuration applies only to accessing the
package and not communicating with the service.

### Working with secrets in Guest Configuration packages

In Azure Policy Guest Configuration, the optimal way to manage secrets used at run time is to store
them in Azure Key Vault. This design is implemented within custom DSC resources.

First, create a user-assigned managed identity in Azure. The identity is used by virtual machines to
access secrets stored in Key Vault. For detailed steps, see
[Create, list or delete a user-assigned managed identity using Azure PowerShell](../../../active-directory/managed-identities-azure-resources/how-to-manage-ua-identity-powershell.md).

Next, create a Key Vault instance. For detailed steps, see [Set and retrieve a secret - PowerShell](../../../key-vault/quick-create-powershell.md).
Assign permissions to the instance to give the user-assigned identity access to secrets stored in
Key Vault. For detailed steps, see
[Set and retrieve a secret - .NET](../../../key-vault/quick-create-net.md#give-the-service-principal-access-to-your-key-vault).

Then, assign the user-assigned identity to your virtual machine. For detailed steps, see
[Configure managed identities for Azure resources on an Azure VM using PowerShell](../../../active-directory/managed-identities-azure-resources/qs-configure-powershell-windows-vm.md#user-assigned-managed-identity).
At scale, assign this identity using Azure Resource Manager via Azure Policy. For detailed steps,
see
[Configure managed identities for Azure resources on an Azure VM using a template](../../../active-directory/managed-identities-azure-resources/qs-configure-template-windows-vm.md#assign-a-user-assigned-managed-identity-to-an-azure-vm).

Finally, within your custom resource use the client ID generated above to access Key Vault using the
token available from the machine. The `client_id` and url to the Key Vault instance can be passed to
the resource as [properties](/powershell/dsc/resources/authoringresourcemof#creating-the-mof-schema)
so the resource won't need to be updated for multiple environments or if the values need to be
changed.

The following code sample can be used in a custom resource to retrieve secrets from Key Vault using
a user-assigned identity. The value returned from the request to Key Vault is plain text. As a best
practice, store it within a credential object.

```azurepowershell-interactive
# the following values should be input as properties
$client_id = 'e3a78c9b-4dd2-46e1-8bfa-88c0574697ce'
$keyvault_url = 'https://keyvaultname.vault.azure.net/secrets/mysecret'

$access_token = ((Invoke-WebRequest -Uri "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&client_id=$client_id&resource=https%3A%2F%2Fvault.azure.net" -Method GET -Headers @{Metadata='true'}).Content | ConvertFrom-Json).access_token

$value = ((Invoke-WebRequest -Uri $($keyvault_url+'?api-version=2016-10-01') -Method GET -Headers @{Authorization="Bearer $access_token"}).content | convertfrom-json).value |  ConvertTo-SecureString -asplaintext -force

$credential = New-Object System.Management.Automation.PSCredential('secret',$value)
```

## Test a Guest Configuration package

After creating the Configuration package but before publishing it to Azure, you can test the
functionality of the package from your workstation or CI/CD environment. The GuestConfiguration
module includes a cmdlet `Test-GuestConfigurationPackage` that loads the same agent in your
development environment as is used inside Azure virtual machines. Using this solution, you can
perform integration testing locally before releasing to billed test/QA/production environments.

```azurepowershell-interactive
Test-GuestConfigurationPackage -Path .\package\AuditWindowsService\AuditWindowsService.zip -Verbose
```

Parameters of the `Test-GuestConfigurationPackage` cmdlet:

- **Name**: Guest Configuration Policy name.
- **Parameter**: Policy parameters provided in hashtable format.
- **Path**: Full path of the Guest Configuration package.

The cmdlet also supports input from the PowerShell pipeline. Pipe the output of
`New-GuestConfigurationPackage` cmdlet to the `Test-GuestConfigurationPackage` cmdlet.

```azurepowershell-interactive
New-GuestConfigurationPackage -Name AuditWindowsService -Configuration .\DSCConfig\localhost.mof -Path .\package -Verbose | Test-GuestConfigurationPackage -Verbose
```

For more information about how to test with parameters,
see the section below
[Using parameters in custom Guest Configuration policies](/azure/governance/policy/how-to/guest-configuration-create#using-parameters-in-custom-guest-configuration-policies).

## Create the Azure Policy definition and initiative deployment files

Once a Guest Configuration custom policy package has been created and uploaded to a location
accessible by the virtual machines, create the Guest Configuration policy definition for Azure
Policy. The `New-GuestConfigurationPolicy` cmdlet takes a publicly accessible Guest Configuration
custom policy package and creates an **auditIfNotExists** and **deployIfNotExists** policy
definition. A policy initiative definition that includes both policy definitions is also created.

The following example creates the policy and initiative definitions in a specified path from a Guest
Configuration custom policy package for Windows and provides a name, description, and version:

```azurepowershell-interactive
New-GuestConfigurationPolicy
    -ContentUri 'https://storageaccountname.blob.core.windows.net/packages/AuditBitLocker.zip?st=2019-07-01T00%3A00%3A00Z&se=2024-07-01T00%3A00%3A00Z&sp=rl&sv=2018-03-28&sr=b&sig=JdUf4nOCo8fvuflOoX%2FnGo4sXqVfP5BYXHzTl3%2BovJo%3D' `
    -DisplayName 'Audit BitLocker Service.' `
    -Description 'Audit if BitLocker is not enabled on Windows machine.' `
    -Path '.\policyDefinitions' `
    -Platform 'Windows' `
    -Version 1.2.3.4 `
    -Verbose
```

Parameters of the `New-GuestConfigurationPolicy` cmdlet:

- **ContentUri**: Public http(s) uri of Guest Configuration content package.
- **DisplayName**: Policy display name.
- **Description**: Policy description.
- **Parameter**: Policy parameters provided in hashtable format.
- **Version**: Policy version.
- **Path**: Destination path where policy definitions are created.
- **Platform**: Target platform (Windows/Linux) for Guest Configuration policy and content package.

The following files are created by `New-GuestConfigurationPolicy`:

- **auditIfNotExists.json**
- **deployIfNotExists.json**
- **Initiative.json**

The cmdlet output returns an object containing the initiative display name and path of the policy
files.

If you would like to use this command to scaffold a custom policy project, you can make changes to
these files. An example would be modifying the 'If' section to evaluate whether a specific Tag is
present for virtual machines. For details on creating policies, see [Programmatically create policies](./programmatically-create.md).

### Using parameters in custom Guest Configuration policies

Guest Configuration supports overriding properties of a Configuration at run time. This feature
means that the values in the MOF file in the package don't have to be considered static. The
override values are provided through Azure Policy and don't impact how the Configurations are
authored or compiled.

The cmdlets `New-GuestConfigurationPolicy` and `Test-GuestConfigurationPolicyPackage` include a parameter named **Parameters**.
This parameter takes a hashtable definition
including all details about each parameter and automatically creates all the required sections of
the files used to create each Azure Policy definition.

The following example would create an Azure Policy to audit a service, where the user selects from a
list of services at the time of Policy assignment.

```azurepowershell-interactive
$PolicyParameterInfo = @(
    @{
        Name = 'ServiceName'                                            # Policy parameter name (mandatory)
        DisplayName = 'windows service name.'                           # Policy parameter display name (mandatory)
        Description = "Name of the windows service to be audited."      # Policy parameter description (optional)
        ResourceType = "Service"                                        # DSC configuration resource type (mandatory)
        ResourceId = 'windowsService'                                   # DSC configuration resource property name (mandatory)
        ResourcePropertyName = "Name"                                   # DSC configuration resource property name (mandatory)
        DefaultValue = 'winrm'                                          # Policy parameter default value (optional)
        AllowedValues = @('BDESVC','TermService','wuauserv','winrm')    # Policy parameter allowed values (optional)
    }
)

New-GuestConfigurationPolicy
    -ContentUri 'https://storageaccountname.blob.core.windows.net/packages/AuditBitLocker.zip?st=2019-07-01T00%3A00%3A00Z&se=2024-07-01T00%3A00%3A00Z&sp=rl&sv=2018-03-28&sr=b&sig=JdUf4nOCo8fvuflOoX%2FnGo4sXqVfP5BYXHzTl3%2BovJo%3D' `
    -DisplayName 'Audit Windows Service.' `
    -Description 'Audit if a Windows Service is not enabled on Windows machine.' `
    -Path '.\policyDefinitions' `
    -Parameters $PolicyParameterInfo `
    -Platform 'Windows' `
    -Version 1.2.3.4 `
    -Verbose
```

For Linux policies, include the property `AttributesYmlContent` in your configuration and overwrite
the values accordingly. The Guest Configuration agent automatically creates the YaML file used by
InSpec to store attributes. See the example below.

```azurepowershell-interactive
Configuration FirewalldEnabled {

    Import-DscResource -ModuleName 'GuestConfiguration'

    Node FirewalldEnabled {

        ChefInSpecResource FirewalldEnabled {
            Name = 'FirewalldEnabled'
            AttributesYmlContent = "DefaultFirewalldProfile: [public]"
        }
    }
}
```

For each additional parameter, add a hashtable to the array. In the policy files, you'll see
properties added to the Guest Configuration configurationName that identify the resource type, name,
property, and value.

```json
{
    "apiVersion": "2018-11-20",
    "type": "Microsoft.Compute/virtualMachines/providers/guestConfigurationAssignments",
    "name": "[concat(parameters('vmName'), '/Microsoft.GuestConfiguration/', parameters('configurationName'))]",
    "location": "[parameters('location')]",
    "properties": {
        "guestConfiguration": {
            "name": "[parameters('configurationName')]",
            "version": "1.*",
            "configurationParameter": [{
                "name": "[Service]windowsService;Name",
                "value": "[parameters('ServiceName')]"
            }]
        }
    }
}
```

## Publish to Azure Policy

The **GuestConfiguration** resource module offers a way to create both policy definitions and the
initiative definition in Azure with one step through the `Publish-GuestConfigurationPolicy` cmdlet.
The cmdlet only has the **Path** parameter that points to the location of the three JSON files
created by `New-GuestConfigurationPolicy`.

```azurepowershell-interactive
Publish-GuestConfigurationPolicy -Path '.\policyDefinitions' -Verbose
```

The `Publish-GuestConfigurationPolicy` cmdlet accepts the path from the PowerShell pipeline. This
feature means you can create the policy files and publish them in a single set of piped commands.

```azurepowershell-interactive
New-GuestConfigurationPolicy -ContentUri 'https://storageaccountname.blob.core.windows.net/packages/AuditBitLocker.zip?st=2019-07-01T00%3A00%3A00Z&se=2024-07-01T00%3A00%3A00Z&sp=rl&sv=2018-03-28&sr=b&sig=JdUf4nOCo8fvuflOoX%2FnGo4sXqVfP5BYXHzTl3%2BovJo%3D' -DisplayName 'Audit BitLocker service.' -Description 'Audit if the BitLocker service is not enabled on Windows machine.' -Path '.\policyDefinitions' -Platform 'Windows' -Version 1.2.3.4 -Verbose | ForEach-Object {$_.Path} | Publish-GuestConfigurationPolicy -Verbose
```

With the policy and initiative definitions created in Azure, the last step is to assign the
initiative. See how to assign the initiative with [Portal](../assign-policy-portal.md), [Azure CLI](../assign-policy-azurecli.md),
and [Azure PowerShell](../assign-policy-powershell.md).

> [!IMPORTANT]
> Guest Configuration policies must **always** be assigned using the initiative that combines the
> _AuditIfNotExists_ and _DeployIfNotExists_ policies. If only the _AuditIfNotExists_ policy is
> assigned, the prerequisites aren't deployed and the policy always shows that '0' servers are
> compliant.

## Policy lifecycle

After you've published a custom Azure Policy using your custom content package,
there are two fields that must be updated if you would like to publish a new release.

- **Version**: When you run the `New-GuestConfigurationPolicy` cmdlet you must specify a version number greater than what is currently published.  The property updates the version of the Guest Configuration assignment in the new policy file so the extension will recognize that the package has been updated.
- **contentHash**: This property is updated automatically by the `New-GuestConfigurationPolicy` cmdlet.  It's a hash value of the package created by `New-GuestConfigurationPackage`.  The property must be correct for the `.zip` file you publish.  If only the `contentUri` property is updated, such as in the case where someone could make a manual change to the Policy definition from the portal, the Extension won't accept the content package.

The easiest way to release an updated package is to repeat the process described in this article
and provide an updated version number.
That process guarantees all properties have been correctly updated.

## Converting Windows Group Policy content to Azure Policy Guest Configuration

Guest Configuration, when auditing Windows machines,
is an implementation of the PowerShell Desired State Configuration syntax.
The DSC community has published tooling to convert exported Group Policy templates to DSC format.
By using this tool together with the Guest Configuration cmdlets described above,
you can convert Windows Group Policy content and package/publish it for Azure Policy to audit.
For details about using the tool, see the article
[Quickstart: Convert Group Policy into DSC](/powershell/dsc/quickstarts/gpo-quickstart).
Once the content has been converted, the steps above to create a package and publish it
as Azure Policy will be the same as for any DSC content.

## OPTIONAL: Signing Guest Configuration packages

Guest Configuration custom policies by default use SHA256 hash to validate that the policy package
hasn't changed from when it was published to when it's read by the server that is being audited.
Optionally, customers may also use a certificate to sign packages and force the Guest Configuration
extension to only allow signed content.

To enable this scenario, there are two steps you need to complete. Run the cmdlet to sign the
content package, and append a tag to the virtual machines that should require code to be signed.

To use the Signature Validation feature, run the `Protect-GuestConfigurationPackage` cmdlet to sign
the package before it's published. This cmdlet requires a 'Code Signing' certificate.

```azurepowershell-interactive
$Cert = Get-ChildItem -Path cert:\LocalMachine\My | Where-Object {($_.Subject-eq "CN=mycert") }
Protect-GuestConfigurationPackage -Path .\package\AuditWindowsService\AuditWindowsService.zip -Certificate $Cert -Verbose
```

Parameters of the `Protect-GuestConfigurationPackage` cmdlet:

- **Path**: Full path of the Guest Configuration package.
- **Certificate**: Code signing certificate to sign the package. This parameter is only supported
  when signing content for Windows.
- **PrivateGpgKeyPath**: Private GPG key path. This parameter is only supported when signing content
  for Linux.
- **PublicGpgKeyPath**: Public GPG key path. This parameter is only supported when signing content
  for Linux.

GuestConfiguration agent expects the certificate public key to be present in "Trusted Root
Certificate Authorities" on Windows machines and in the path `/usr/local/share/ca-certificates/extra`
on Linux machines. For the node to verify signed content, install the
certificate public key on the virtual machine before applying the custom policy. This
process can be done using any technique inside the VM, or by using Azure Policy. An example template
is
[provided here](https://github.com/Azure/azure-quickstart-templates/tree/master/201-vm-push-certificate-windows).
The Key Vault access policy must allow the Compute resource provider to access certificates during
deployments. For detailed steps, see
[Set up Key Vault for virtual machines in Azure Resource Manager](../../../virtual-machines/windows/key-vault-setup.md#use-templates-to-set-up-key-vault).

Following is an example to export the public key from a signing certificate, to import to the
virtual machine.

```azurepowershell-interactive
$Cert = Get-ChildItem -Path cert:\LocalMachine\My | Where-Object {($_.Subject-eq "CN=mycert3") } | Select-Object -First 1
$Cert | Export-Certificate -FilePath "$env:temp\DscPublicKey.cer" -Force
```

A good reference for creating GPG keys to use with Linux virtual machines is provided
by an article on GitHub,
[Generating a new GPG key](https://help.github.com/en/articles/generating-a-new-gpg-key).

After your content is published, append a tag with name `GuestConfigPolicyCertificateValidation` and value `enabled` to all virtual
machines where code signing should be required. This tag can be delivered at scale using Azure
Policy. See the [Apply tag and its default value](../samples/apply-tag-default-value.md) sample.
Once this tag is in place, the policy definition generated using the `New-GuestConfigurationPolicy`
cmdlet enables the requirement through the Guest Configuration extension.

## Next steps

- Learn about auditing VMs with [Guest Configuration](../concepts/guest-configuration.md).
- Understand how to [programmatically create policies](programmatically-create.md).
- Learn how to [get compliance data](getting-compliance-data.md).
