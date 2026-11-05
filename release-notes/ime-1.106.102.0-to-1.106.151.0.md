# ime-1.106.102.0-to-1.106.151.0

## Scope

This document describes the changes found between these Intune Management Extension packages:

| Package | Version |
| --- | --- |
| Previous package | `1.106.102.0` |
| New package | `1.106.151.0` |

Evidence sources: MSI database and table comparison, extraction and SHA 256 comparison of the complete embedded CAB payloads, CLR TypeDef and MethodDef comparison, direct managed IL method body comparison, IL normalization to account for metadata token changes, ASCII and UTF 16 string inspection, selected native binary and symbol inspection, direct comparison of the nested WinRE CAB and manifest, and extraction and inspection of the actual `IntuneWindowsAgentCustomActions` SfxCA stream.

The MSI cabinets have the same irregular layout seen in earlier packages. Three file records reference compressed data shared with other entries: `f_appConfigShadow`, `_E9B6082C`, and `_34C9B140`. A working copy of each CAB index was normalized for extraction without changing the compressed file contents. All 154 file entries from each package were resolved and compared, representing 151 unique compressed data records and three aliases per package.

Managed method comparisons include direct IL byte comparison and a second pass that removes metadata token renumbering. Native findings are based on bytes, symbols, and strings, not a complete native-code disassembly. Where a call path, runtime outcome, or feature activation is not directly established, that is identified below.

Presence of code, a new flight name, or a telemetry contract does not mean the behavior is enabled for every tenant. This comparison covers the two endpoint builds and does not establish the exact intermediate build in which every change first appeared.

## Executive summary

`1.106.151.0` updates the component download and policy verification stack, certificate pinning behavior, and telemetry around Windows device management. This is a broader security, reliability, and diagnostics update inside existing IME components, rather than a release that adds new MSI features or introduces a new Win32 application workflow.

The important changes are:

1. **Downloaded components gain a more explicit signature verification path.** `Components.Runtime.dll` adds an `AuthenticodeVerifier` implementation, signer-chain checks, Microsoft root verification, expected signer subject checks, and `ComponentSignatureVerification` telemetry. `Common.Base.dll` adds a signature verification contract to the download path.
2. **TLS certificate pinning is simplified and defaults to enforcement.** `AgentCommon.dll` removes the old `Disabled / Audit / Enforce` mode model from the current flight state. New diagnostics describe enforcing with safe defaults when flight resolution fails, while an optional flight controls the volume of successful validation telemetry.
3. **Certificate pinning failures can be retained and reported.** `AgentCommon.dll`, `DataSensorPlugIn.dll`, and `ClientHealthEval.exe` add a shared registry snapshot, synchronization, reporting, and cleanup path for `LatestWindowsCertPinningFailed`.
4. **DataSensor gains more Windows management diagnostics.** It introduces OMA DM session-ended telemetry and DeviceManagementSync event processing for Windows events 209 and 321, together with additional event metadata and error handling.
5. **The native policy stack contains additional signed WinDC document validation code.** `IntunePolicyStorage.dll` and related policy runtime binaries gain references to document signatures, tenant binding, signing time, certificate chains, and signer verification. Their exact runtime activation has not been established from native disassembly.
6. **Cloud registration and flighting helpers expand.** `Common.Base.dll` adds a native registration environment reader, while `Flighting.dll` gains a flight configuration factory and environment-filter resolution.
7. **Process handling and component management gain supporting changes.** `Common.Base.dll` adds `JobObject.AssignProcess` and more explicit Win32 error diagnostics. `Microsoft.Management.IntuneComponent.exe` and component runtime assemblies also have managed method changes.
8. **The main Win32 app, script, Autopilot bootstrapper, and app status method bodies remain stable.** `Win32AppPlugIn.dll`, `ScriptPlugIn.dll`, `BootstrapperAgentCore.dll`, and `StatusServiceLibrary.dll` contain no changed managed IL method bodies in the direct comparison, despite differences in whole-file hashes.
9. **The existing IC3/Trouter and WinRE packages are unchanged.** Both `Microsoft.IC3.Trouter.dll` and the WinRE CAB and manifest are byte identical. This comparison does not show a new notification transport or recovery bundle implementation.
10. **The installer structure stays the same.** Both MSIs contain 154 file entries and the same component, service, registry, custom-action, and install-sequence inventories. The new build replaces existing files without adding a new MSI setup path.

The headline for this build is therefore **IME 1.106.151.0 strengthens component and policy trust validation, makes TLS certificate pinning failure handling more resilient, and adds Windows management health diagnostics, while leaving the main app delivery, script, notification, and WinRE paths structurally unchanged.**

## Package-level changes

| Property | 1.106.102.0 | 1.106.151.0 |
| --- | --- | --- |
| `ProductVersion` | `1.106.102.0` | `1.106.151.0` |
| `ProductCode` | `{AF98F07E-B150-462A-BC23-DE6C1B377D58}` | `{0C295A70-0C7C-451D-8D2D-C15DFCF7B226}` |
| `UpgradeCode` | `{9FE9701C-0F89-40B4-B77A-AA65607E87D8}` | same |
| MSI SHA 256 | `5664b8225d3d58042bb530ddae72e92fb39758837b1cae1cdde895714fa31134` | `2377e0b46a5415b463c042a17940b021ab7854bab85dbdb66b27b6abe37f14fe` |
| MSI size | 13,840,384 bytes | 13,873,152 bytes (`+32,768`) |
| Embedded `media1.cab` size | 13,246,472 bytes | 13,279,614 bytes (`+33,142`) |
| File entries | 154 | 154 |
| Distinct physical CAB payload entries | 151 | 151 |
| Files added | — | 0 |
| Files removed | — | 0 |
| Shared files changed | — | 93 |
| Shared files byte identical | — | 61 |
| Changed localization resource entries | — | 42 |
| Changed nonlocalization file entries | — | 51 |
| MSI `Component` rows | 161 | 161 |
| MSI `CustomAction` rows | 22 | 22 |
| MSI `InstallExecuteSequence` rows | 52 | 52 |
| MSI `Registry` rows | 9 | 9 |
| MSI `ServiceInstall` rows | 1 | 1 |

The installer still contains the same files, components, services, and custom action definitions. The `Signature` and `Upgrade` version boundaries move to `1.106.151.0`, and `ProductCode` changes as expected for a new MSI release. There are 89 changed `File` table rows, chiefly for file version and size metadata, and three changed `MsiFileHash` rows.

The difference between 93 changed file hashes and fewer meaningful behavioral changes is important. A rebuilt assembly can have a different SHA 256 or file version while retaining identical managed method bodies.

## Largest payload deltas

The largest differences among the existing binaries are concentrated in native policy storage, dynamic pinning, the main shared agent library, and the DataSensor/Client Health components.

| File | 1.106.102.0 | 1.106.151.0 | Size delta |
| --- | ---: | ---: | ---: |
| `IntunePolicyStorage.dll` | 1,097,040 | 1,158,456 | `+61,416` |
| `dynpin_cxx.dll` | 971,072 | 1,005,880 | `+34,808` |
| `Microsoft.Management.Services.IntuneWindowsAgent.AgentCommon.dll` | 781,688 | 815,480 | `+33,792` |
| `IntunePolicyAdapter.dll` | 779,088 | 800,568 | `+21,480` |
| `Microsoft.Management.Clients.IntuneManagementExtension.DataSensorPlugIn.dll` | 169,848 | 186,232 | `+16,384` |
| `ClientHealthEval.exe` | 180,640 | 185,760 | `+5,120` |
| `Microsoft.Management.Clients.Components.Runtime.dll` | 297,848 | 302,944 | `+5,096` |
| `Microsoft.Management.Clients.Flighting.dll` | 53,584 | 56,120 | `+2,536` |
| `Microsoft.Management.Clients.Common.Base.dll` | 114,512 | 116,024 | `+1,512` |

`IntunePolicyStorage.dll` and `dynpin_cxx.dll` each have two MSI file records pointing to the same corresponding compressed data in the cabinet. The sizes above count each unique binary once.

## Largest managed-code changes

The managed metadata and IL comparison identifies the following noteworthy expansions:

| Assembly | MethodDefs old → new | Finding |
| --- | ---: | --- |
| `Components.Runtime.dll` | 811 → 844 | New Authenticode verification and component signature telemetry |
| `DataSensorPlugIn.dll` | 401 → 424 | Additional OMA DM, DeviceManagementSync, and certificate failure telemetry |
| `AgentCommon.dll` | 3,326 → 3,348 | Certificate pinning state, synchronization, and reporting helpers |
| `Common.Base.dll` | 491 → 503 | Download verification, registration environment, and process helpers |
| `ClientHealthEval.exe` | 686 → 697 | Certificate pinning failure report construction and sending |
| `Flighting.dll` | 204 → 209 | Additional configuration and environment filter resolution |
| `Policy.Runtime.dll` | 346 → 348 | Changes to WinDC document retrieval and validation support |
| `IntuneWindowsAgent.exe` | 1,371 → 1,373 | Updated certificate pinning flight resolution |

Many raw IL differences are caused by shifted metadata tokens rather than changes to the actual instruction sequence. Targeted comparisons were normalized where necessary before assessing a method as changed. Native binary size changes are not treated as proof of a particular runtime behavior.

# 1. Component downloads add signer verification and outcome telemetry

**Changed components:** `Microsoft.Management.Clients.Components.Runtime.dll`, `Microsoft.Management.Clients.Common.Base.dll`.

`Components.Runtime.dll` introduces a group of types under `Download.Verification`:

```text
AuthenticodeVerifier
ICorrelatedIntegrityVerifier
SignerChainVerification
SignatureVerificationResult
SignatureVerificationTelemetry
SignatureVerificationTelemetryRecord
SignerSubjectValidationStatus
WinTrustNativeMethods
```

The new code includes:

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
```

Telemetry names and properties identify `ComponentSignatureVerification`, an expected signer subject of `Microsoft Corporation`, Authenticode results, Microsoft root policy results, publisher checks, and validation duration.

`Common.Base.dll` adds `Download.ICorrelatedPayloadVerifier.VerifySignature`, `ReliableDownloader.VerifySignature`, and `DownloadRequest.VerificationTargetName` to the download infrastructure.

`ComponentInstaller.ValidatePackage` grows from 172 to 215 IL bytes. Together with the new verifier types, that is direct evidence of changes to the downloadable-component validation path. The exact set of downloads that use this path and their feature-flight conditions still require runtime tracing.

# 2. TLS certificate pinning state and fallback behavior change

**Changed components:** `AgentCommon.dll`, `IntuneWindowsAgent.exe`, `dynpin_cxx.dll`.

The old `TlsCertificatePinningMode` enum and the `Enabled`, `Enforce`, and `Mode` properties on the flight state are no longer present. The new state model contains a `SafeState` fallback and `DisableSuccessfulTelemetry` setting.

Relevant method size changes in `AgentCommon.dll`:

| Method | Old IL bytes | New IL bytes |
| --- | ---: | ---: |
| `TlsCertificatePinningContext.ValidateServerCertificate` | 331 | 254 |
| `TlsCertificatePinningContext.ConfigureRequest` | 115 | 101 |
| `TlsCertificatePinningContext.ResolveFlightState` | 53 | 52 |
| `TlsCertificatePinningContext.EmitResult` | 662 | 673 |

New log messages state that TLS certificate pinning is permanently enforced and that failure to resolve the flight state results in safe enforcement defaults. They also distinguish a missing platform certificate chain and an enforced signature-validation failure.

The main agent adds `ResolveDisableSuccessfulTlsTelemetryFlightAsync` and references `DisableSuccessfulTlsCertificatePinningTelemetry`. The new optional flight appears to control suppression of clean success telemetry, rather than activation of certificate pinning itself.

`dynpin_cxx.dll` also changes, with additional certificate-chain, CMS signer, ECDSA, and trust diagnostics visible in the native strings. Its exact native control flow has not been reconstructed.

This is a change to IME outbound trust validation and its diagnostic behavior. It does not establish the result of any particular validation attempt on a managed device.

# 3. Certificate pinning failures gain a persistent health reporting path

**Changed components:** `AgentCommon.dll`, `DataSensorPlugIn.dll`, `ClientHealthEval.exe`.

The new code connects certificate pinning failures to registry persistence and Client Health reporting.

`AgentCommon.dll` adds:

```text
RegistryHelper.SaveLatestWindowsCertPinningFailed
RegistryHelper.DeleteUploadedWindowsCertPinningFailed
RegistryHelper.CreateWindowsCertPinningFailedMutex
RegistryHelper.WithWindowsCertPinningFailedRegistryLock
RegistryHelperWrapper.GetLatestWindowsCertPinningFailed
LatestWindowsCertPinningFailed
Global\Microsoft.IntuneManagementExtension.WindowsCertPinningFailed
```

`DataSensorPlugIn.dll` adds:

```text
EmitWindowsCertPinningFailedTelemetry
SaveLatestWindowsCertPinningFailed
CreateWindowsCertPinningFailedProperties
WindowsCertPinningFailedEventName
```

`ClientHealthEval.exe` adds:

```text
WindowsCertPinningFailedReportBuilder.CreateReportBody
ClientHealthRuleEngine.SendWindowsCertPinningFailedReport
```

New messages describe saving the latest failure, retaining a record when reporting fails, and deleting the uploaded snapshot only if the registry value has not since been replaced by a newer failure. A named mutex protects concurrent access, with diagnostics for timeouts and abandoned mutexes.

The components point to the following logical flow:

```mermaid
flowchart LR
    A[Certificate pinning failure] --> B[DataSensor collection]
    B --> C[Registry snapshot]
    C --> D[Client Health report]
    D --> E[Upload attempt]
    E --> F[Remove reported snapshot]
```

This flow summarizes the responsibilities visible in the code. The exact reporting cadence, retry schedule, and active flights were not established in this package comparison.

# 4. DataSensor adds Windows device management diagnostics

`DataSensorPlugIn.dll` increases from 401 to 424 MethodDefs. Besides the certificate failure path, it adds event processing and reporting for Windows MDM synchronization.

New members include:

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

Error strings also describe missing execution metadata, failed event parsing, cancelled collection, and unavailable telemetry infrastructure. This is more targeted diagnostic coverage for Windows device management activity.

Event IDs 209 and 321 are Windows event identifiers in this context, **not** IC3 notification workload IDs. `Microsoft.IC3.Trouter.dll` is byte identical between these packages.

# 5. WinDC policy signature and certificate validation expand

**Changed components:** `IntunePolicyStorage.dll`, `IntunePolicyAdapter.dll`, `Policy.Runtime.dll`, and related signature validation binaries.

The native `IntunePolicyStorage.dll` increases from 1,097,040 to 1,158,456 bytes. Symbol and diagnostic strings include:

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

`Policy.Runtime.dll` also has changes in `WinDCUtils.RetrieveDocuments`, `WinDCUtils.RetrieveDMOrchestratorEntries`, and a signature validation mode registry helper.

This is direct evidence of additional signed Windows declared configuration document validation logic, including signer identity, certificate trust, tenant binding, and time checks. It does not prove which policy document types invoke every new routine or whether the checks are active in a given environment. That needs native call-flow analysis and on-device traces.

# 6. Registration environment and ECS flight configuration change

**Changed components:** `Common.Base.dll`, `Flighting.dll`.

`Common.Base.dll` gains `Cloud.NativeRegistrationEnvironmentReader` and `IRegistrationEnvironmentReader`, providing another code path for resolving the device registration environment.

`Flighting.dll` adds:

```text
FlightConfigurationFactory
FlightEnvironmentFilterResolver.TryGetFilterValue
```

These changes are part of the existing environment-aware configuration framework. They provide more explicit registration and flight-filter resolution, but their presence alone does not show whether a particular tenant is receiving a new ECS setting.

# 7. Component management and process support change

`Common.Base.dll` adds `JobObject.AssignProcess` and `BuildWin32FailureDiagnostic`. These are useful additions for managing processes and identifying Windows API failure conditions in the shared runtime.

`Microsoft.Management.IntuneComponent.exe`, `Components.Base.dll`, `Policy.Base.dll`, and `Policy.Runtime.dll` also have changes in managed method bodies or metadata. They belong to the same shared component and policy management stack as the new signature validation code.

The metadata and IL differences are confirmed, but the comparison does not justify assigning every change in these assemblies to a new standalone customer feature.

# 8. Core app processing, Autopilot, IC3 and WinRE remain largely unchanged

Several prominent IME workloads have different assembly hashes but no changed managed IL method bodies in this comparison:

| Component | Result |
| --- | --- |
| `Win32AppPlugIn.dll` | No changed managed IL method bodies |
| `ScriptPlugIn.dll` | No changed managed IL method bodies |
| `BootstrapperAgentCore.dll` | No changed managed IL method bodies |
| `StatusServiceLibrary.dll` | No changed managed IL method bodies |
| `AppEnforcementProcessor.dll` | No changed managed IL method bodies |
| `AgentExecutor.exe` | No changed managed IL method bodies |
| `ImeUI.exe` | No changed managed IL method bodies |

This includes existing Win32 app evaluation and status logic, the Autopilot bootstrapper's Managed Installer provider, and the script processing methods. The normalized instruction sequences for the existing `ManagedInstallerSynchronizationHelper` and `WdacWMIHelper` methods in `AgentCommon.dll` also match.

Other unchanged payloads include:

```text
Microsoft.IC3.Trouter.dll
Microsoft.Intune.WinRe.Bundle.cab
Microsoft.Intune.WinRe.Bundle.json
Microsoft.Management.Deployment.InProc.dll
```

This comparison does not show a new IC3 transport implementation, WinRE update, Win32 app enforcement workflow, or Managed Installer policy synchronization algorithm. These observations concern the local binaries only; they do not rule out service-side or Windows changes.

# 9. MSI installation and custom action comparison

The MSI structure remains consistent between the two releases.

| MSI table or stream | Finding |
| --- | --- |
| `Property` | `ProductVersion` and `ProductCode` updated |
| `File` | Same 154 file IDs; 89 rows differ in version, size, or other metadata |
| `Component` and `FeatureComponents` | Same 161 component records and component mapping |
| `CustomAction` | Same 22 action definitions |
| `InstallExecuteSequence` | Same 52 entries and order |
| `Registry` | Same 9 entries |
| `ServiceInstall` and `ServiceControl` | Unchanged |
| `Signature` | `NewerFileVersionSearch.MinVersion` updated |
| `Upgrade` | New version boundaries for upgrade and downgrade detection |
| `MsiFileHash` | Three changed rows |
| `IntuneWindowsAgent.cat` | Different contents; same 143,578-byte length |

The embedded `Binary.IntuneWindowsAgentCustomActions` stream changes from 479,808 to 480,096 bytes. Its nested CAB contains eight files; only `SideCarSetupCustomActions.dll` differs in bytes, changing from 56,224 to 56,184 bytes. All 81 managed method bodies in that DLL remain identical. The other seven nested CAB files are byte identical.

The outer MSI digital signature streams also differ. This investigation did not independently validate the external signing chains of the MSI packages.

# 10. Evidence, limitations and lab checks

The MSI comparison confirms file inventory, changed hashes, installer table differences, CAB content, and the nested custom action payload. The managed assembly comparison confirms the visible type and method changes as well as the IL differences described above. Normalizing metadata tokens avoids treating simple token renumbering as new logic, but does not replace fully resolving every referenced symbol.

The native `IntunePolicyStorage.dll`, `IntunePolicyAdapter.dll`, `SignatureValidationLibrary.dll`, and `dynpin_cxx.dll` require deeper disassembly for exact execution paths. Static comparisons cannot establish server-side rollout status, the effective outcome of certificate validation on a device, or the timing of health report uploads.

Useful controlled lab checks:

1. Capture `ComponentSignatureVerification` during a downloadable-component update and compare the Authenticode, Microsoft root, and signer subject outcomes.
2. Trace certificate pinning state resolution and validate the safe fallback behavior when configuration cannot be resolved.
3. Inspect `LatestWindowsCertPinningFailed` persistence and `ClientHealthEval` reporting on a test device, including behavior when upload fails. Avoid modifying production TLS traffic to generate a failure.
4. Check whether DataSensor emits OMA DM session-ended events and DeviceManagementSync 209/321 diagnostics under the relevant configuration.
5. Capture WinDC policy document retrieval and validation to determine which document types enter the new native signature verification path.
6. Check registration environment resolution and the resulting ECS filter values in a controlled cloud configuration.

## Summary of confirmed changes

The directly supported changes are concentrated in three areas: **component and WinDC signature validation, TLS certificate pinning and health reporting, and Windows management event telemetry**. Supporting changes exist in flight configuration, registration environment resolution, and process helpers. The installer layout and several core workload engines remain unchanged.

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
