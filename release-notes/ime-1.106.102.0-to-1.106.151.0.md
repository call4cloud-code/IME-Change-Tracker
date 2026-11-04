# Intune Management Extension 1.106.102.0 to 1.106.151.0

**Comparison:** MSI payloads and installer tables, with managed assembly metadata and IL comparison.

**Evidence level:** Extracted directly from the two MSI files. Managed IL differences are compared by method name and normalized to exclude metadata token renumbering. Native changes are assessed from binary and symbol/string differences, not a complete native instruction diff. This does not establish which server flights are enabled for a tenant.

## Executive summary

The main changes in `1.106.151.0` are around trust and diagnostics. There is new code for signed component verification, stronger and safer TLS certificate pinning handling, reporting of certificate pinning failures, and additional Windows MDM session telemetry. The native policy storage binary also contains more WinDC document signature validation references.

### At a glance

| Item | 1.106.102.0 | 1.106.151.0 |
| --- | ---: | ---: |
| MSI size | 13,840,384 bytes | 13,873,152 bytes |
| Embedded `media1.cab` | 13,246,472 bytes | 13,279,614 bytes |
| Installer `File` records | 154 | 154 |
| Embedded CAB physical file data records after alias removal | 151 | 151 |
| File entries with different bytes | — | 93 |
| Byte identical file entries | — | 61 |
| Changed localized resource files | — | 42 |
| Changed other file entries | — | 51 |
| `Component` rows | 161 | 161 |
| `CustomAction` rows | 22 | 22 |
| `InstallExecuteSequence` rows | 52 | 52 |
| `Registry` rows | 9 | 9 |
| `ServiceInstall` rows | 1 | 1 |
| `MsiFileHash` rows | 23 | 23 |

The two embedded CABs contain three alias records that share data with other CAB records. The counts above include the alias file entries and compare their resolved file contents. The aliases are `f_appConfigShadow`, `_E9B6082C`, and `_34C9B140`.

## 1. Exact methods checked

| Component | Method | Finding |
| --- | --- | --- |
| `ScriptPlugIn.dll` | `ScriptProcessorV2.ProcessManagedInstallerPolicy` | Exact IL unchanged, 1,417 bytes |
| `ScriptPlugIn.dll` | `EmitManagedInstallerGoldenSignals` | Exact IL unchanged, 435 bytes |
| `ScriptPlugIn.dll` | `EmitAppsSyncManagedInstallerSignal` | Exact IL unchanged, 115 bytes |
| `BootstrapperAgentCore.dll` | `ManagedInstallerProvider.ApplyManagedInstallerPolicy` | Exact IL unchanged, 104 bytes |
| `BootstrapperAgentCore.dll` | `ManagedInstallerProvider.SetManagedInstallerPolicy` | Exact IL unchanged, 237 bytes |
| `BootstrapperAgentCore.dll` | `ManagedInstallerProvider.WaitForPolicyToTakeEffect` | Exact IL unchanged, 55 bytes |
| `Win32AppPlugIn.dll` | `ApplicationPoller.IsWDACSyncRequired` | Exact IL unchanged, 76 bytes |
| `Win32AppPlugIn.dll` | `UnlockWin10SHelper.GetWDACSyncState` | Exact IL unchanged, 171 bytes |
| `AgentCommon.dll` | `ManagedInstallerSynchronizationHelper.NeedToInstallManagedInstallerPolicy` | Same normalized IL, 339 bytes both |
| `AgentCommon.dll` | `ManagedInstallerSynchronizationHelper.LoadManagedInstallerPolicy` | Same normalized IL, 328 bytes both |
| `AgentCommon.dll` | `ManagedInstallerSynchronizationHelper.StartProactiveRemediationCheckIn` | Same normalized IL, 113 bytes both |
| `AgentCommon.dll` | `WdacWMIHelper.GetAllPolicyInfos` | Same normalized IL, 738 bytes both |
| `AgentCommon.dll` | `WdacWMIHelper.CreateToken` | Same normalized IL, 210 bytes both |
| `AgentCommon.dll` | `WdacWMIHelper.DeletePolicy` | Same normalized IL, 342 bytes both |

The broader `AgentCommon.dll` is **not** byte identical and does contain real changes elsewhere. For its Managed Installer and WDAC helpers, instruction and constant sequences did not change after filtering metadata tokens. That makes a Managed Installer code fix unlikely in this release, but the comparison does not rule out external Windows or service side behavior changes.

There is no evidence here for a new AppLocker Managed Installer tagging algorithm, new WDAC WMI policy sync workflow, different application trust transition, or new blocking/waiting behavior in the Win32 app plug-in.

## 2. Component downloads gain explicit signature verification and telemetry

**Changed assemblies:** `Microsoft.Management.Clients.Components.Runtime.dll` and `Microsoft.Management.Clients.Common.Base.dll`.

`Components.Runtime.dll` increases from 811 to 844 MethodDefs. It introduces the following types:

```text
Download.Verification.AuthenticodeVerifier
Download.Verification.ICorrelatedIntegrityVerifier
Download.Verification.SignerChainVerification
Download.Verification.SignatureVerificationResult
Download.Verification.SignatureVerificationTelemetry
Download.Verification.SignatureVerificationTelemetryRecord
Download.Verification.SignerSubjectValidationStatus
Download.Verification.WinTrustNativeMethods
```

Key new methods and signals:

```text
AuthenticodeVerifier.VerifySignature
AuthenticodeVerifier.ValidateSignerChains
AuthenticodeVerifier.VerifyMicrosoftRootPolicy
AuthenticodeVerifier.IsExpectedSignerSubject
AuthenticodeVerifier.GetSignerSubjectName
WinTrustNativeMethods.WinVerifyTrust
WinTrustNativeMethods.CertVerifyCertificateChainPolicy
IntegrityVerifier.VerifySignatureCore
IntegrityVerifier.RecordSignatureOutcome
SignatureVerificationTelemetry.Record
ComponentInstaller.ValidatePackage
COMPONENT_SIGNATURE_VERIFICATION_EVENT_NAME
ExpectedSignerSubjectName
```

Visible messages identify the expected subject as `Microsoft Corporation` and distinguish Authenticode results, Microsoft root trust results, signer subject status, publisher matching, and duration in telemetry.

`Common.Base.dll` adds `Download.ICorrelatedPayloadVerifier.VerifySignature`, `ReliableDownloader.VerifySignature`, and a `VerificationTargetName` property on `DownloadRequest`. These changes expose a more explicit path for verifying and reporting the signature of a downloaded component.

**Code level result:** `ComponentInstaller.ValidatePackage` changes from 172 to 215 IL bytes. New verification classes and telemetry contracts are present. This is a meaningful change in component payload verification. **It is not proof that WDAC Managed Installer tagging changed.**

**Limit:** This identifies new code and public method contracts. It does not prove the precise server side feature gate or that every download takes the new path.

## 3. TLS certificate pinning moves to safer enforcement and reporting

**Changed assemblies:** `AgentCommon.dll`, `IntuneWindowsAgent.exe`, and the native `dynpin_cxx.dll`.

The `TlsCertificatePinningMode` type disappears from `AgentCommon.dll`. Old `TlsCertificatePinningFlightState` getters for `Enabled`, `Enforce`, and `Mode` are removed. New members include `TlsCertificatePinningFlightStateProvider.SafeState`, `TlsCertificatePinningContext.IsCleanSuccess`, and `TlsCertificatePinningFlightState.DisableSuccessfulTelemetry`.

Relevant method IL size changes:

| Method | Old IL bytes | New IL bytes |
| --- | ---: | ---: |
| `TlsCertificatePinningContext.ValidateServerCertificate` | 331 | 254 |
| `TlsCertificatePinningContext.ConfigureRequest` | 115 | 101 |
| `TlsCertificatePinningContext.ResolveFlightState` | 53 | 52 |
| `TlsCertificatePinningContext.EmitResult` | 662 | 673 |

New log strings explicitly state that TLS pinning is permanently enforced, that failure to evaluate a flight uses safe defaults, and that an enforced signature validation failure is recorded. The optional flight `DisableSuccessfulTlsCertificatePinningTelemetry` appears to control reporting of clean successes rather than enable or disable the trust check itself.

`IntuneWindowsAgent.exe` adds `ResolveDisableSuccessfulTlsTelemetryFlightAsync`. The main agent also changes its TLS flight resolution path. `dynpin_cxx.dll` changes in size and contains additional crypto/certificate trust strings, but its native control flow has not been fully reconstructed.

**Interpretation:** The release moves toward more consistent TLS certificate pinning enforcement with better failure handling and less unnecessary success telemetry. This is separate from WDAC file trust or Managed Installer.

## 4. Failed TLS certificate pinning now has a durable health report path

The new reporting path spans `AgentCommon.dll`, `DataSensorPlugIn.dll`, and `ClientHealthEval.exe`.

### Registry and synchronization

`AgentCommon.dll` introduces:

```text
RegistryHelper.SaveLatestWindowsCertPinningFailed
RegistryHelper.DeleteUploadedWindowsCertPinningFailed
RegistryHelper.CreateWindowsCertPinningFailedMutex
RegistryHelper.WithWindowsCertPinningFailedRegistryLock
RegistryHelperWrapper.GetLatestWindowsCertPinningFailed
LatestWindowsCertPinningFailed
Global\Microsoft.IntuneManagementExtension.WindowsCertPinningFailed
```

New messages describe a snapshot being retained if it changes during upload, an abandoned mutex being recovered, and a timed out registry lock. This shows explicit protection against clobbering a newer failure while an older one is being reported.

### Client Health

`ClientHealthEval.exe` adds `WindowsCertPinningFailedReportBuilder.CreateReportBody` and `ClientHealthRuleEngine.SendWindowsCertPinningFailedReport`, plus new registry, telemetry, and flighting dependencies. New log strings confirm that a failed report is retained for a retry and registry cleanup happens only after reporting succeeds.

### DataSensor

`DataSensorPlugIn.dll` adds `EmitWindowsCertPinningFailedTelemetry`, `SaveLatestWindowsCertPinningFailed`, and `CreateWindowsCertPinningFailedProperties`. It can capture the event and retain a registry snapshot for later reporting.

A simplified observed component relationship:

```mermaid
flowchart LR
    A[Certificate pinning failure] --> B[DataSensor event collection]
    B --> C[LatestWindowsCertPinningFailed registry snapshot]
    C --> D[ClientHealth report]
    D --> E[Send report]
    E --> F[Clear the uploaded snapshot]
```

The diagram shows the roles visible in the code; exact timing and flight conditions require an on-device trace.

## 5. New Windows MDM event diagnostics

`DataSensorPlugIn.dll` changes from 401 to 424 MethodDefs. Beyond certificate pinning it adds:

```text
TelemetryJob.EmitOmaDmSessionEndedTelemetry
TelemetryJob.EmitDeviceManagementSyncTelemetry
TelemetryJob.GetDeviceManagementSync209Telemetry
TelemetryJob.GetDeviceManagementSync321ExecutionTelemetry
TelemetryJob.GetDeviceManagementSync209ExecutionTelemetry
TelemetryJob.GetDeviceManagementSyncExecutionTelemetry
ListenerEventConfiguration.EnableDeviceManagementSyncEvent209DiagnosticEventQuery
ListenerEventConfiguration.EnableDeviceManagementSyncEvent321DiagnosticEventQuery
ECSFlightKeyEmitOmaDmSessionEndedTelemetry
```

This is evidence of additional observability for Windows MDM synchronization and OMA DM session completion. Do not interpret the event numbers 209/321 as IC3 workload identifiers. The `Microsoft.IC3.Trouter.dll` payload remains byte identical.

## 6. Signed WinDC document validation appears in native policy storage

`IntunePolicyStorage.dll` grows from 1,097,040 to 1,158,456 bytes. New symbol and diagnostic strings include:

```text
Intune::SignatureValidation::VerifyWinDcDocumentSignature
Intune::SignatureValidation::VerifyWinDCSignatureBound
Intune::SignatureValidation::VerifyAadTenantId
Intune::SignatureValidation::ValidateWinDocLeafCertHash
Intune::SignatureValidation::IsSigningTimeWithinCertValidity
Intune::PolicyStorage::UpdatePolicySignature
Intune::PolicyStorage::UpdatePolicyVerificationTimeStamp
Intune::Utility::VerifySignerCertChainTimeStamp
signerCount == 1
```

`Policy.Runtime.dll` also has method body changes in `WinDCUtils.RetrieveDocuments`, `WinDCUtils.RetrieveDMOrchestratorEntries`, and the signature validation mode registry helper.

**Interpretation:** The native component includes additional code for validating signed Windows declared configuration documents and their signer/binding/timestamp information. A complete disassembly is required to establish when these new routines are invoked and whether they change enforcement for a particular policy type.

This is not direct evidence of a WDAC Managed Installer behavior change.

## 7. Cloud environment, flighting and process helpers

Additional managed changes include:

| Component | New members | Evidence |
| --- | --- | --- |
| `Common.Base.dll` | `Cloud.NativeRegistrationEnvironmentReader`, `IRegistrationEnvironmentReader` | New source for registration environment information |
| `Flighting.dll` | `FlightConfigurationFactory`, `FlightEnvironmentFilterResolver.TryGetFilterValue` | Changes to flight configuration/resolution |
| `Common.Base.dll` | `JobObject.AssignProcess`, `BuildWin32FailureDiagnostic` | More explicit process assignment/error diagnostic path |
| `IntuneWindowsAgent.exe` | `ResolveDisableSuccessfulTlsTelemetryFlightAsync` | Optional telemetry flight resolution |

These are code differences, but they do not establish a visible customer facing feature by themselves.

## 8. Installation and signing/package deltas

The installer has a new product code and version. The `Upgrade` version boundaries and `Signature.NewerFileVersionSearch.MinVersion` move from `1.106.102.0` to `1.106.151.0`.

| MSI table or stream | Finding |
| --- | --- |
| `Property` | `ProductVersion` and `ProductCode` changed |
| `File` | Same 154 IDs; 89 rows differ in version, size or other file metadata |
| `MsiFileHash` | Three rows differ: `f_agentCatalog`, `_0178CD2F`, `_34C9B140` |
| `Signature` | `NewerFileVersionSearch.MinVersion` updated |
| `Upgrade` | Both upgrade/downgrade boundaries updated |
| `CustomAction` | Same 22 actions and parameters |
| `InstallExecuteSequence` | Same 52 entries and ordering |
| `Registry`, `ServiceInstall`, `ServiceControl` | Tables unchanged |
| `Component`, `FeatureComponents` | Same 161 rows in both |
| `IntuneWindowsAgent.cat` | Content changed, size remains 143,578 bytes |

The embedded `Binary.IntuneWindowsAgentCustomActions` stream changes from 479,808 to 480,096 bytes. Its nested CAB contains eight files; only `SideCarSetupCustomActions.dll` differs, changing from 56,224 to 56,184 bytes. All **81 managed method bodies** in that DLL remain identical, and the other seven nested files remain byte identical. The extra bytes in the stream do not demonstrate a custom action behavior change.

The MSI digital signature streams differ. This analysis does not perform an external trust-chain verification of either MSI signature.

## 9. Unchanged binaries worth calling out

The following are byte identical even though they sit in important IME flows:

```text
Microsoft.IC3.Trouter.dll
Microsoft.Intune.WinRe.Bundle.cab
Microsoft.Intune.WinRe.Bundle.json
Microsoft.Management.Deployment.InProc.dll
```

Several managed DLLs, including `Win32AppPlugIn.dll`, `ScriptPlugIn.dll`, `BootstrapperAgentCore.dll`, and `StatusServiceLibrary.dll`, have different whole-file hashes due to packaging or metadata, while their managed IL method bodies remain unchanged.

## 10. Confidence and what is still unknown

**Confirmed from the MSI:** file count, table counts/content, file hashes, embedded CAB files, and nested MSI custom action payloads.

**Confirmed from managed metadata/IL:** new and removed method and type names, IL byte sizes, and changes to IL instruction/operand sequences. For targeted WDAC/Managed Installer methods, normalized instruction sequences match. The normalization removes metadata token integers, so it is not a full resolution of every referenced method/member token.

**Confirmed at the symbol/string level:** the native IntunePolicyStorage additions and dynpin crypto/certificate strings. This is not a complete native disassembly.

**Not established:** which ECS flights are active on a device, the exact certificate failure reporting cadence, whether a particular WDAC policy is impacted, the effective result of runtime signature checks, and any Windows or Intune service side behavior changes independent of the MSI payload.

## 11. Lab checks

1. On a lab device with `1.106.151.0`, observe the component runtime logs during a signed component download. Look for `ComponentSignatureVerification` and the signer, Microsoft root and Authenticode outcomes.
2. Deliberately generate a test certificate pinning failure in a controlled test environment, then inspect `LatestWindowsCertPinningFailed` storage and the Client Health reporting path. Do not introduce a TLS interception on a production tenant to test this.
3. Capture DataSensor event processing and determine whether the Event 209/321 and OMA DM session events are emitted under the relevant flight.
4. Repeat a normal Managed Installer app install under both versions and compare the same AppLocker/Code Integrity events and IME log sequence. The code comparison gives no reason to expect a different Managed Installer tagging outcome.

## Appendix A. Changed nonlocalization file entries

| File | CAB file ID | Old bytes | New bytes | SHA256 old → new, first 12 chars |
| --- | --- | ---: | ---: | --- |
| `AgentExecutor.exe` | `f_A2FC1319` | 75,640 | 75,680 | `a66b246436d0` → `71bb50adf5b9` |
| `ClientCertCheck.exe` | `f_F5C054BE` | 30,584 | 30,584 | `dded99b23fba` → `785b8119b49e` |
| `ClientHealthEval.exe` | `f_2598C59C` | 180,640 | 185,760 | `a9e5242d5fd1` → `3e1ae7426cf2` |
| `CoreLib.dll` | `_495A5463` | 1,084,240 | 1,082,168 | `26c355ed4b1d` → `844dd5fb61fd` |
| `ImeUI.exe` | `f_3f93074f` | 257,912 | 257,912 | `55043c8d7070` → `4f85d9d64e4a` |
| `IntunePolicyStorage.dll` | `_1C89D395` | 1,097,040 | 1,158,456 | `70f5299e99f0` → `69e93b32c777` |
| `IntuneWindowsAgent.cat` | `f_agentCatalog` | 143,578 | 143,578 | `e2351878a406` → `af97b7bbd30e` |
| `Microsoft.IdentityModel.Extensions.dll` | `f_9776E9F8` | 68,872 | 68,872 | `3b912aeada26` → `b03ceccbd14c` |
| `Microsoft.IdentityModel.dll` | `f_4204427F` | 1,113,408 | 1,113,408 | `a2a588d11e05` → `e7cff1f06a30` |
| `Microsoft.Management.Clients.Common.Base.dll` | `f_77B5BEE6` | 114,512 | 116,024 | `e7ee6a20c361` → `34f566209ff6` |
| `Microsoft.Management.Clients.Common.EtwProvider.Sidecar.dll` | `f_84292245` | 38,264 | 38,240 | `ac069757f4d8` → `09eb017a3683` |
| `Microsoft.Management.Clients.Common.SignatureValidation.dll` | `_8D2F6A41` | 47,440 | 47,456 | `09349449fa08` → `739197bc3da2` |
| `Microsoft.Management.Clients.Common.Telemetry.dll` | `f_CD7048E1` | 83,360 | 82,744 | `507932063751` → `e44866da7ab6` |
| `Microsoft.Management.Clients.Components.Base.dll` | `_BEB8EAE9` | 37,200 | 37,728 | `665dbe8dd5b5` → `e8167c4f86aa` |
| `Microsoft.Management.Clients.Components.Runtime.dll` | `_75F257E9` | 297,848 | 302,944 | `f6f4d8cbf463` → `38a2711026ec` |
| `Microsoft.Management.Clients.CopyAgentCatalog.exe` | `f_678F63F6` | 20,856 | 20,344 | `a7f01faee828` → `33595d0c1887` |
| `Microsoft.Management.Clients.Flighting.dll` | `B3B71F52` | 53,584 | 56,120 | `4a2c3f7616b0` → `5366638c6941` |
| `Microsoft.Management.Clients.IntuneManagementExtension.AppEnforcementProcessor.dll` | `A5CB4B9F` | 124,832 | 124,832 | `6b6094cc5c3f` → `0994c827508a` |
| `Microsoft.Management.Clients.IntuneManagementExtension.DataSensorPlugIn.dll` | `bf36af75` | 169,848 | 186,232 | `e27936680976` → `36712791911f` |
| `Microsoft.Management.Clients.IntuneManagementExtension.IntunePivotPlugIn.dll` | `AA0684F9` | 143,736 | 143,736 | `1b2e323f4e2a` → `f237b1df3d2d` |
| `Microsoft.Management.Clients.IntuneManagementExtension.PivotParser.dll` | `ac1b8a01` | 217,464 | 217,504 | `cf2b4f6ed03e` → `d2812cb0ea86` |
| `Microsoft.Management.Clients.IntuneManagementExtension.ProcessingAnalysis.dll` | `A23ECE02` | 28,536 | 28,576 | `886892f3b3f7` → `df710e0b04db` |
| `Microsoft.Management.Clients.IntuneManagementExtension.ScriptPlugIn.dll` | `a6011b1f` | 251,768 | 251,808 | `356a24eb2c92` → `77026e1e278b` |
| `Microsoft.Management.Clients.IntuneManagementExtension.SidecarNotification.dll` | `e34410f8` | 18,336 | 18,296 | `a69b03b07b04` → `dd499692e3be` |
| `Microsoft.Management.Clients.IntuneManagementExtension.StatusServiceLibrary.dll` | `A3185EB3` | 28,536 | 28,536 | `0ba2c494fd93` → `779cc6c223e6` |
| `Microsoft.Management.Clients.IntuneManagementExtension.TamperProtection.dll` | `AF1B1D37` | 61,816 | 61,856 | `5fca8e1d2d84` → `a9fa8001028e` |
| `Microsoft.Management.Clients.IntuneManagementExtension.UnlockWin10SPlugIn.dll` | `C4CAFBAF` | 51,576 | 51,576 | `7612b257335e` → `975beb693c3f` |
| `Microsoft.Management.Clients.IntuneManagementExtension.Win32AppInventoryCollector.dll` | `AD0B9F84` | 29,560 | 29,600 | `5a0a0ed197a2` → `86269ecd45c8` |
| `Microsoft.Management.Clients.IntuneManagementExtension.Win32AppPlugIn.dll` | `BAEA6E21` | 961,440 | 961,400 | `bc15c896a547` → `8583176dbfec` |
| `Microsoft.Management.Clients.IntuneManagementExtension.WinGetLibrary.dll` | `B8003E57` | 82,296 | 82,336 | `e56029f84503` → `66d8875f47b8` |
| `Microsoft.Management.Clients.IntuneWindowsAgent.RebootCoordinator.dll` | `f_69e59da0` | 36,256 | 36,216 | `08417a388c42` → `06b87e7bd999` |
| `Microsoft.Management.Clients.Policy.Base.dll` | `_7BB2E691` | 16,248 | 16,696 | `e686ac16ad49` → `345bcf8f55ce` |
| `Microsoft.Management.Clients.Policy.Runtime.dll` | `_60FB0C1A` | 111,992 | 111,928 | `a56135417df9` → `f36d2d9ccdbd` |
| `Microsoft.Management.IntuneComponent.exe` | `_B1FCA5F9` | 53,112 | 53,048 | `83ef5ef4e41e` → `ffca838fb3b8` |
| `Microsoft.Management.Services.BootstrapperAgentCore.dll` | `A8830AB4` | 189,856 | 189,856 | `a3695a2bc939` → `cb6a6f708f6f` |
| `Microsoft.Management.Services.BootstrapperAgentEtwProvider.dll` | `f_8F4C15B5` | 60,792 | 60,792 | `55ed5ca6d689` → `947e7ad4fe7a` |
| `Microsoft.Management.Services.DevicePreparationProviderCommon.dll` | `B6B6949B` | 43,896 | 43,896 | `cdd406f086cb` → `b260d7384f1b` |
| `Microsoft.Management.Services.IntuneWindowsAgent.AgentCommon.Logging.dll` | `A358EF66` | 18,296 | 18,296 | `5060ae55b09b` → `6651c7f4b86b` |
| `Microsoft.Management.Services.IntuneWindowsAgent.AgentCommon.ServiceContracts.dll` | `A183BCCE` | 45,944 | 45,984 | `d2c2bc382e97` → `18753c8fdd0d` |
| `Microsoft.Management.Services.IntuneWindowsAgent.AgentCommon.dll` | `f_0D61D816` | 781,688 | 815,480 | `1d220af13054` → `3963a2334833` |
| `Microsoft.Management.Services.IntuneWindowsAgent.SideCarETWProvider.dll` | `f_318F9F4C` | 24,992 | 24,952 | `a4657da4f594` → `d6c732be5fdf` |
| `Microsoft.Management.Services.IntuneWindowsAgent.exe` | `f_IntuneWindowsAgent` | 538,016 | 538,488 | `9fcd17741c69` → `ca99274ed20b` |
| `Newtonsoft.Json.dll` | `c1ae5c06` | 731,440 | 731,440 | `697e3cdc2ed7` → `77ad9d866550` |
| `OneDSApi.dll` | `F6D8B905` | 191,392 | 191,352 | `ecf20b495a87` → `24e4f7e285be` |
| `PolicyStorageCmdlets.dll` | `_D7A3B1E4` | 40,312 | 39,736 | `3df715297c10` → `6ab1dcf3fcb2` |
| `Polly.dll` | `A6866986` | 278,888 | 278,928 | `890537a59dbf` → `ebbae9babf22` |
| `SignatureValidationLibrary.dll` | `_A35BE4C7` | 519,504 | 517,944 | `2369ec53f7e7` → `d0ecc2bbca6f` |
| `dynpin_cxx.dll` | `_0178CD2F` | 971,072 | 1,005,880 | `aacb62652f2a` → `c7039a9f87c9` |
| `IntunePolicyAdapter.dll` | `_39E14C68` | 779,088 | 800,568 | `525a80c22eb9` → `9da45ebd52a8` |
| `IntunePolicyStorage.dll` | `_E9B6082C` | 1,097,040 | 1,158,456 | `70f5299e99f0` → `69e93b32c777` |
| `dynpin_cxx.dll` | `_34C9B140` | 971,072 | 1,005,880 | `aacb62652f2a` → `c7039a9f87c9` |

## Appendix B. Changed localization file entries

The following 42 resource file entries also changed. They are included in the 93 changed entries total.

| CAB ID | Resource file | Old bytes | New bytes |
| --- | --- | ---: | ---: |
| `csAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,344 | 20,384 |
| `csImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,808 |
| `daAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 19,832 | 19,832 |
| `daImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,848 |
| `deAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,344 | 20,384 |
| `deImeUI.resources.dll` | `ImeUI.resources.dll` | 18,848 | 18,808 |
| `elAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 22,392 | 22,392 |
| `elImeUI.resources.dll` | `ImeUI.resources.dll` | 20,384 | 20,344 |
| `esAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,344 | 20,384 |
| `esImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,808 |
| `fiAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,344 | 20,344 |
| `fiImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,808 |
| `frAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,384 | 20,344 |
| `frImeUI.resources.dll` | `ImeUI.resources.dll` | 18,800 | 18,808 |
| `huAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,384 | 20,344 |
| `huImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,808 |
| `itAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,344 | 20,344 |
| `itImeUI.resources.dll` | `ImeUI.resources.dll` | 18,848 | 18,800 |
| `jaAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 21,368 | 21,368 |
| `jaImeUI.resources.dll` | `ImeUI.resources.dll` | 19,320 | 19,360 |
| `koAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,376 | 20,384 |
| `koImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,808 |
| `nlAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,344 | 20,344 |
| `nlImeUI.resources.dll` | `ImeUI.resources.dll` | 18,848 | 18,840 |
| `nbAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 19,832 | 19,872 |
| `nbImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,808 |
| `plAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,384 | 20,344 |
| `plImeUI.resources.dll` | `ImeUI.resources.dll` | 18,848 | 18,848 |
| `ptAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,344 | 20,344 |
| `ptImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,808 |
| `roAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,344 | 20,344 |
| `roImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,808 |
| `ruAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 21,880 | 21,880 |
| `ruImeUI.resources.dll` | `ImeUI.resources.dll` | 20,344 | 20,344 |
| `svAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 19,832 | 19,832 |
| `svImeUI.resources.dll` | `ImeUI.resources.dll` | 18,808 | 18,848 |
| `trAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 20,344 | 20,344 |
| `trImeUI.resources.dll` | `ImeUI.resources.dll` | 18,848 | 18,848 |
| `zhcnAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 19,320 | 19,360 |
| `zhcnImeUI.resources.dll` | `ImeUI.resources.dll` | 18,336 | 18,296 |
| `zhtwAgentExecutor.resources.dll` | `AgentExecutor.resources.dll` | 19,872 | 19,872 |
| `zhtwImeUI.resources.dll` | `ImeUI.resources.dll` | 18,840 | 18,848 |

## Appendix C. Inputs and SHA256

| File | Bytes | SHA256 |
| --- | ---: | --- |
| `IntuneWindowsAgent_1.106.102.0_manual_ffad3f11(2).msi` | 13,840,384 | `5664b8225d3d58042bb530ddae72e92fb39758837b1cae1cdde895714fa31134` |
| `IntuneWindowsAgent_1.106.151.0.msi` | 13,873,152 | `2377e0b46a5415b463c042a17940b021ab7854bab85dbdb66b27b6abe37f14fe` |

**Method:** Compound File/MSI table and string pool parsing, extraction of LZX CAB folders, direct file hashing, CLR TypeDef/MethodDef metadata comparison, and IL operand normalization. Three shared CAB data entries were deduplicated for extraction and restored by alias ID for the installed file comparison.

**Suggested tracker path:** `release-notes/ime-1.106.102.0-to-1.106.151.0.md`.
