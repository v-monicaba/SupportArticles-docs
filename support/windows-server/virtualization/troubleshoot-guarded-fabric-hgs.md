---
title: Troubleshoot the Host Guardian Service
description: Provides resolutions to common problems encountered when deploying or operating a Host Guardian Service (HGS) server in a guarded fabric.
ms.date: 01/15/2025
manager: dcscontentpm
audience: itpro
ms.topic: troubleshooting
ms.reviewer: kaushika, dongill, inhenkel, v-lianna
ms.custom:
- sap:virtualization and hyper-v\shielded virtual machines
- pcy:WinComm Storage High Avail
ms.assetid: 424b8090-0692-49a6-9dc4-3c0e77d74b80
---
# Troubleshoot the Host Guardian Service

This article describes resolutions to common problems encountered when deploying or operating a Host Guardian Service (HGS) server in a guarded fabric.

_Applies to:_ &nbsp; Windows Server 2022, Windows Server 2019, Windows Server 2016

If you're unsure of the nature of your problem, first try running the [guarded fabric diagnostics](troubleshoot-guarded-fabric-diagnostics.md) on your HGS servers and Hyper-V hosts to narrow down the potential causes.

## Certificates

HGS requires several certificates in order to operate, including the admin-configured encryption and signing certificate as well as an attestation certificate managed by HGS itself. If these certificates are incorrectly configured, HGS is unable to serve requests from Hyper-V hosts wishing to attest or unlock key protectors for shielded VMs.
The following sections cover common problems related to certificates configured on HGS.

### Certificate permissions

HGS must be able to access both the public and private keys of the encryption and signing certificates added to HGS by the certificate thumbprint.
Specifically, the group managed service account (gMSA) that runs the HGS service needs access to the keys.
To find the gMSA used by HGS, run the following command in an elevated PowerShell prompt on your HGS server:

```powershell
(Get-IISAppPool -Name KeyProtection).ProcessModel.UserName
```

How you grant the gMSA account access to use the private key depends on where the key is stored: on the machine as a local certificate file, on a hardware security module (HSM), or using a custom third-party key storage provider.

#### Grant access to software-backed private keys

If you're using a self-signed certificate or a certificate issued by a certificate authority that isn't stored in a hardware security module or custom key storage provider, you can change the private key permissions by performing the following steps:

1. Open local certificate manager (*certlm.msc*).
2. Expand **Personal** > **Certificates** and find the signing or encryption certificate that you want to update.
3. Right-click the certificate and select **All Tasks** > **Manage Private Keys**.
4. Select **Add** to grant a new user access to the certificate's private key.
5. In the object picker, enter the gMSA account name for HGS found earlier, then select **OK**.
6. Ensure the gMSA has **Read** access to the certificate.
7. Select **OK** to close the permission window.

If you're running HGS on Server Core or are managing the server remotely, you won't be able to manage private keys using the local certificate manager.
Instead, you need to download the [Guarded Fabric Tools PowerShell module](https://www.powershellgallery.com/packages/GuardedFabricTools), which will allow you to manage the permissions in PowerShell.

1. Open an elevated PowerShell console on the Server Core machine or use PowerShell Remoting with an account that has local administrator permissions on HGS.
2. Run the following commands to install the Guarded Fabric Tools PowerShell module and grant the gMSA account access to the private key.

```powershell
$certificateThumbprint = '<ENTER CERTIFICATE THUMBPRINT HERE>'

# Install the Guarded Fabric Tools module, if necessary
Install-Module -Name GuardedFabricTools -Repository PSGallery

# Import the module into the current session
Import-Module -Name GuardedFabricTools

# Get the certificate object
$cert = Get-Item "Cert:\LocalMachine\My\$certificateThumbprint"

# Get the gMSA account name
$gMSA = (Get-IISAppPool -Name KeyProtection).ProcessModel.UserName

# Grant the gMSA read access to the certificate
$cert.Acl = $cert.Acl | Add-AccessRule $gMSA Read Allow
```

#### Grant access to HSM or custom provider-backed private keys

If your certificate's private keys are backed by a hardware security module (HSM) or a custom key storage provider (KSP), the permission model depends on your specific software vendor.
For the best results, consult your vendor's documentation or support site for information on how private key permissions are handled for your specific device/software.
In all cases, the gMSA used by HGS requires read permissions on the encryption, signing, and communications certificate private keys so that it can perform signing and encryption operations.

Some hardware security modules don't support granting specific user accounts access to a private key; rather, they allow the computer account access to all keys in a specific key set.
For such devices, it's usually sufficient to give the computer access to your keys and HGS is able to leverage that connection.

##### Tips for HSMs

Below are suggested configuration options to help you successfully use HSM-backed keys with HGS based on Microsoft and its partners' experiences.
These tips are provided for your convenience and aren't guaranteed to be correct at the time of reading, nor are they endorsed by the HSM manufacturers.
Contact your HSM manufacturer for accurate information pertaining to your specific device if you have further questions.

|HSM Brand/Series      | Suggestion|
|----------------------|-------------|
|Gemalto SafeNet       | Ensure the Key Usage Property in the certificate request file is set to 0xa0, allowing the certificate to be used for signing and encryption. Additionally, you must grant the gMSA account read access to the private key using the local certificate manager tool (see steps above).|
|nCipher nShield        | Ensure each HGS node has access to the security world containing the signing and encryption keys. You may additionally need to grant the gMSA read access to the private key using the local certificate manager (see steps above).|
|Utimaco CryptoServers | Ensure the Key Usage Property in the certificate request file is set to 0x13, allowing the certificate to be used for encryption, decryption, and signing.|

### Certificate requests

If you're using a certificate authority to issue your certificates in a public key infrastructure (PKI) environment, you need to ensure your certificate request includes the minimum requirements for HGS' usage of those keys.

#### Signing certificates

|CSR property | Required value|
|-------------|---------------|
|Algorithm    | RSA|
|Key size     | At least 2048 bits|
|Key usage    | Signature/Sign/DigitalSignature|

#### Encryption certificates

|CSR property | Required value|
|-------------|---------------|
|Algorithm    | RSA|
|Key size     | At least 2048 bits|
|Key usage    | Encryption/Encrypt/DataEncipherment|

#### Active Directory Certificate Services templates

If you're using Active Directory Certificate Services (ADCS) certificate templates to create the certificates, we recommended you use a template with the following settings:

|ADCS template property | Required value|
|-----------------------|---------------|
|Provider category      | Key storage provider|
|Algorithm name         | RSA|
|Minimum key size       | 2048|
|Purpose                | Signature and encryption|
|Key usage extension    | Digital Signature, Key Encipherment, Data Encipherment ("Allow encryption of user data")|

### Time drift

If your server's time has drifted significantly from that of other HGS nodes or Hyper-V hosts in your guarded fabric, you may encounter issues with the attestation signer certificate validity.
The attestation signer certificate is created and renewed behind the scenes on HGS and is used to sign health certificates issued to guarded hosts by the Attestation Service.

To refresh the attestation signer certificate, run the following command in an elevated PowerShell prompt.

```powershell
Start-ScheduledTask -TaskPath \Microsoft\Windows\HGSServer -TaskName
AttestationSignerCertRenewalTask
```

Alternatively, you can manually run the scheduled task by opening **Task Scheduler** (*taskschd.msc*), navigating to **Task Scheduler Library** > **Microsoft** > **Windows** > **HGSServer** and running the task named **AttestationSignerCertRenewalTask**.

## Switching attestation modes

If you switch HGS from TPM mode to Active Directory mode or vice versa using the [Set-HgsServer](/powershell/module/hgsserver/set-hgsserver) cmdlet, it may take up to 10 minutes for every node in your HGS cluster to start enforcing the new attestation mode.

This is normal behavior.

It's advised that you don't remove any policies allowing hosts from the previous attestation mode until you have verified that all hosts are attesting successfully using the new attestation mode.

### Known issue when switching from TPM to AD mode

If you initialized your HGS cluster in TPM mode and later switch to Active Directory mode, there's a known issue that prevents other nodes in your HGS cluster from switching to the new attestation mode.
To ensure all HGS servers are enforcing the correct attestation mode, run `Set-HgsServer -TrustActiveDirectory` on each node of your HGS cluster.

This issue doesn't apply if you're switching from TPM mode to AD mode and the cluster was originally set up in AD mode.

You can verify the attestation mode of your HGS server by running [Get-HgsServer](/powershell/module/hgsserver/get-hgsserver).

## Memory dump encryption policies

If you're trying to configure memory dump encryption policies and don't see the default HGS dump policies (Hgs\_NoDumps, Hgs\_DumpEncryption and Hgs\_DumpEncryptionKey) or the dump policy cmdlet (`Add-HgsAttestationDumpPolicy`), it's likely that you don't have the latest cumulative update installed.

To fix this, [update your HGS server](/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-manage-hgs#patching-hgs) to the latest cumulative Windows update and [activate the new attestation policies](/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-manage-hgs#updates-requiring-policy-activation).

Ensure you update your Hyper-V hosts to the same cumulative update before activating the new attestation policies, as hosts that don't have the new dump encryption capabilities installed will likely fail attestation once the HGS policy is activated.

## Endorsement Key Certificate error messages

When you register a host using the [Add-HgsAttestationTpmHost](/powershell/module/hgsattestation/add-hgsattestationtpmhost) cmdlet, two TPM identifiers are extracted from the provided platform identifier file: the endorsement key certificate (EKcert) and the public endorsement key (EKpub). The EKcert identifies the manufacturer of the TPM, providing assurances that the TPM is authentic and manufactured through the normal supply chain. The EKpub uniquely identifies that specific TPM, and is one of the measures HGS uses to grant a host access to run shielded VMs.

You'll receive an error when you register a TPM host if either of the two conditions are true:

- The platform identifier file doesn't contain an endorsement key certificate.
- The platform identifier file contains an endorsement key certificate, but that certificate isn't trusted on your system.

Certain TPM manufacturers don't include EKcerts in their TPMs.

If you suspect that this is the case with your TPM, confirm with your OEM that your TPMs should not have an EKcert and use the `-Force` flag to manually register the host with HGS.
If your TPM should have an EKcert but one was not found in the platform identifier file, ensure you're using an administrator (elevated) PowerShell console when running [Get-PlatformIdentifier](/powershell/module/platformidentifier/get-platformidentifier) on the host.

If you received the error that your EKcert is untrusted, ensure that you have [installed the trusted TPM root certificates package](/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-install-trusted-tpm-root-certificates) on each HGS server and that the root certificate for your TPM vendor is present in the local machine's "TrustedTPM\_RootCA" store. Any applicable intermediate certificates also need to be installed in the "TrustedTPM\_IntermediateCA" store on the local machine.
After installing the root and intermediate certificates, you should be able to run `Add-HgsAttestationTpmHost` successfully.

## Group managed service account (gMSA) privileges

HGS service account (gMSA used for Key Protection Service application pool in IIS) needs to be granted [Generate security audits](/windows/security/threat-protection/security-policy-settings/generate-security-audits) privilege, also known as `SeAuditPrivilege`. If this privilege is missing, initial HGS configuration succeeds and IIS starts, however the Key Protection Service is non-functional and returns HTTP error 500 ("Server Error in /KeyProtection Application"). You may also observe the following warning messages in Application event log.

```output
System.ComponentModel.Win32Exception (0x80004005): A required privilege is not held by the client
at Microsoft.Windows.KpsServer.Common.Diagnostics.Auditing.NativeUtility.RegisterAuditSource(String pszSourceName, SafeAuditProviderHandle& phAuditProvider)
at Microsoft.Windows.KpsServer.Common.Diagnostics.Auditing.SecurityLog.RegisterAuditSource(String sourceName)
```

or

```output
Failed to register the security event source.
   at System.Web.HttpApplicationFactory.EnsureAppStartCalledForIntegratedMode(HttpContext context, HttpApplication app)
   at System.Web.HttpApplication.RegisterEventSubscriptionsWithIIS(IntPtr appContext, HttpContext context, MethodInfo[] handlers)
   at System.Web.HttpApplication.InitSpecial(HttpApplicationState state, MethodInfo[] handlers, IntPtr appContext, HttpContext context)
   at System.Web.HttpApplicationFactory.GetSpecialApplicationInstance(IntPtr appContext, HttpContext context)
   at System.Web.Hosting.PipelineRuntime.InitializeApplication(IntPtr appContext)

Failed to register the security event source.
   at Microsoft.Windows.KpsServer.Common.Diagnostics.Auditing.SecurityLog.RegisterAuditSource(String sourceName)
   at Microsoft.Windows.KpsServer.Common.Diagnostics.Auditing.SecurityLog.ReportAudit(EventLogEntryType eventType, UInt32 eventId, Object[] os)
   at Microsoft.Windows.KpsServer.KpsServerHttpApplication.Application_Start()

A required privilege is not held by the client
   at Microsoft.Windows.KpsServer.Common.Diagnostics.Auditing.NativeUtility.RegisterAuditSource(String pszSourceName, SafeAuditProviderHandle& phAuditProvider)
   at Microsoft.Windows.KpsServer.Common.Diagnostics.Auditing.SecurityLog.RegisterAuditSource(String sourceName)
```

Additionally, you may notice that none of the Key Protection Service cmdlets (for example, [Get-HgsKeyProtectionCertificate](/powershell/module/hgskeyprotection/get-hgskeyprotectioncertificate)) work and instead return errors.

To resolve this issue, you need to grant gMSA the "Generate security audits" (`SeAuditPrivilege`). To do that, you may use either Local security policy *SecPol.msc* on every node of the HGS cluster, or Group Policy. Alternatively, you could use [SecEdit.exe](/windows-server/administration/windows-commands/secedit) tool to export the current Security policy, make the necessary edits in the configuration file (which is a plain text) and then import it back.

> [!NOTE]
> When configuring this setting, the list of security principles defined for a privilege fully overrides the defaults (it doesn't concatenate). Hence, when defining this policy setting, be sure to include both default holders of this privilege (Network service and Local service) in addition to the gMSA that you're adding.
