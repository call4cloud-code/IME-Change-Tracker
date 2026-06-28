# Intune Management Extension — `1.101.111.0` → `1.103.101.0` deep diff

Reverse-engineered from the two MSIs (`IntuneWindowsAgent_1_101_111_0.msi`, `IntuneWindowsAgent_1_103_101_0.msi`). MSI payloads were extracted from the embedded `media1.cab`; managed assemblies were diffed at the metadata + IL level (type/method/string/constant sets, plus targeted IL disassembly of changed methods).

Everything below describes *this build pair only*. Most of the new behaviour is **flight-gated** and self-labelled **"Phase 1"**, so what actually runs in a tenant depends on which ECS account flights are enabled. Microsoft can change any of it before docs/GA.

---

## 0. Identity & packaging

| | 1.101.111 | 1.103.101 |
|---|---|---|
| ProductVersion | 1.101.111.0 | 1.103.101.0 |
| ProductCode | `{CEFAEC57-33DC-4775-BFD2-561699357699}` | `{58976FF3-87CA-4691-BDFC-671F37F07CB6}` |
| UpgradeCode | `{9FE9701C-0F89-40B4-B77A-AA65607E87D8}` (unchanged) | same |
| Payload files | 131 | 134 (+3) |

**Installer-table changes (MSI):**
- `Property`: ProductCode + ProductVersion bumped; `Upgrade` min/max version rows bumped accordingly. UpgradeCode unchanged → normal major-upgrade.
- `CustomAction`: **removed** `RunCopyCatalogSilently` (type 3137); **added** `SyncAppConfigTimestamps` (type 65). This pairs with the new `*.exe.config.bak` file and the agent's new config-state self-check (see §9) — the CA aligns the config file's create/modify timestamps so the runtime "config tamper" heuristic doesn't false-positive on a fresh install.
- `Registry`, `ServiceInstall`, `ServiceControl`, `Feature`, `Shortcut`: **no changes**. Service install/run model is identical.

**New files in 1.103:**
- `Microsoft.Bcl.Memory.dll` — new dependency (pulled in by the Extensions 10.x upgrade).
- `System.Threading.Channels.dll` — new dependency (used by the reworked IC3/push pipeline).
- `Microsoft.Management.Services.IntuneWindowsAgent.exe.config.bak` — backup copy of the agent config, consumed by the config-state check + `SyncAppConfigTimestamps` CA.

**Dependency bumps** (`.exe.config` binding redirects): `Microsoft.Extensions.*` 9.0.0.9 / 10.0.0.3 → **10.0.0.6**, added a `DependencyInjection.Abstractions` redirect, and folded away the separate `System.Runtime.CompilerServices.Unsafe` redirect. Large third-party deltas (`System.Text.Json` +51 KB, `WindowsPackageManager.dll` +425 KB, `Microsoft.IC3.Trouter.dll` +115 KB, vcruntime/concrt/vccorlib) are framework/WinGet/Trouter refreshes, not IME logic.

The full changed first-party file list is in the appendix. The localized `*.resources.dll` churn (±40 bytes each) is resource-string rebuild noise and is not analysed.

---

## 1. Headline feature — RealTimeCompliance / ComplianceMonitor (entirely new)

The whole namespace `Microsoft.Management.Services.IntuneWindowsAgent.ComplianceMonitor` (16 types) **did not exist in 1.101** and is added in 1.103. It runs inside the agent service process. It does **not** execute compliance scripts itself — it's a near-real-time watcher over the built-in Windows device-compliance signals that nudges the rest of the stack to re-evaluate early.

**New types:** `ComplianceMonitor`, `ComplianceMonitorLauncher`, `ComplianceMonitorStore`, `ComplianceMonitorConstants`, `ComplianceSettingHelper`, `ComplianceCycleAggregator`, `ComplianceMonitorAuditing`, `DciHelper`, `DriftRecord`, `CycleTelemetry`, `CycleOutcome`, `HourlyRollup`, `ComplianceSettingId`, `Precheck.ComplianceMonitorPrecheck`, and interfaces `IComplianceMonitorLauncher/IComplianceSettingHelper/IDciHelper/IComplianceMonitorAuditing/IComplianceMonitorPrecheck`.

**Supporting plumbing added in `AgentCommon.dll`:**
- `Mdm` namespace: `DeviceTriggerBase`, `IDeviceTrigger`, `DeviceTriggerResult`, `IMdmSession`, `MdmSessionWrapper`, `MdmSessionState` — the OMA-DM device check-in wrapper.
- `Wmi.WmiHelper` (`QueryScalar`, `ReadFirstScalar`, `ReadLatencyMs`, `RunWithTimeout`, `CanonicalizeWmiValue`) — the WMI access layer the monitor reads through.
- `RegistryAccessor.BuildHardenedDacl` + `EnsureSecuredKeysExist` — DACL hardening for the monitor's registry keys.
- `ServiceContracts.ComplianceMonitor.ComplianceMonitorConfig` + `PerSettingConfig` — the server config contract.

### 1.1 Applicability (all required)
`ComplianceMonitor.IsApplicable` → `ComplianceMonitorPrecheck`:
1. OS build ≥ Windows 10 1703,
2. device is MDM-enrolled,
3. Microsoft Defender is the **active** AV.

Plus the master ECS flight `EnableRealTimeComplianceAccount`. If off, `ComplianceMonitorLauncher` re-arms on a timer and re-attempts later without a service restart.

### 1.2 Config sources
- **Server config** arrives in SessionSettings under value `ComplianceMonitorConfigJson` and deserializes to `ComplianceMonitorConfig`:

```jsonc
{
  "Enabled": true,
  "PollIntervalSeconds": 300,          // clamped 60..3600, default 300
  "DciMaxCheckInsPerHourPerDevice": 2, // clamped 0..10, default 2
  "Settings": [ "BitLockerEnabled", "SecureBootEnabled", ... ],
  "PerSetting": { "BitLockerEnabled": { "Enabled": true } }  // dict<string,PerSettingConfig>
}
```
  `Enabled=false` (or master flight off) → monitor self-stops.
- **Local registry** root `HKLM\SOFTWARE\Microsoft\IntuneManagementExtension\ComplianceMonitor` with subkeys `\Config`, `\Config\PerSetting`, `\Config\RateWindow`, `\Settings`. Values: `Enabled`, `PollIntervalSeconds`, `DciMaxCheckInsPerHourPerDevice`, `Settings`, plus per-setting `LastValue` baselines and the rate window (`WindowStartTicks`, `Count`).

### 1.3 The poll cycle (`ExecuteCycleAsync`)
Each tick: start a stopwatch + record WMI latency baseline → take config's `Settings` (dedup, cap at `MaxSettingsPerCycle`) → for each setting run a drift-check task under a parallelism semaphore → `WhenAll` → collect `DriftRecord`s → build a CSV of drifted names → hand to `DciHelper.HandleDriftsAsync` → emit cycle telemetry (drifted count, duration, WMI latency) via `LogCycle`.

### 1.4 What it watches and where it reads (`ComplianceSettingHelper.ReadSettingValue`)
WMI namespace `root\cimv2\mdm\dmmap` (the MDM bridge), plus Defender/DeviceGuard:

| Setting name | Source |
|---|---|
| `RtpEnabled` | Defender real-time protection |
| `DefenderEnabled` / `DefenderVersion` / `SignatureOutOfDate` | Defender status (`AMServiceEnabled`, `AMProductVersion`, `AntivirusSignatureAge`) |
| `AntivirusRequired` / `AntivirusRequireCurrentSignature` | `MDM_DeviceStatus_Antivirus01` (`Status` / `SignatureStatus`) |
| `AntiSpywareRequired` / `AntiSpywareRequireCurrentSignature` | `MDM_DeviceStatus_Antispyware01` |
| `ActiveFirewallRequired` | `MDM_DeviceStatus_Firewall01.Status` |
| `BitLockerEnabled` | `MDM_DeviceStatus_Compliance01.EncryptionCompliance` |
| `SecureBootEnabled` | `MDM_DeviceStatus.SecureBootState` |
| `CodeIntegrityEnabled` | Device Guard `CodeIntegrityPolicyEnforcementStatus` |
| `TpmRequired` | TPM check |
| `OsBuildVersion` | OS build |

`DetectDriftInSetting`: read current value → compare (ordinal `String.Equals`) to the stored baseline → if different, emit a `DriftRecord {SettingName, OldValue, NewValue}`. The new value is **not** persisted as baseline here; that happens only after a *successful* check-in (§1.5).

### 1.5 Drift → device check-in (`DciHelper`)
Decision order in `HandleDriftsAsync`:
1. flight `EnableComplianceMonitorSimulateMode` on → `decision=Simulated`: commit baselines, fire nothing;
2. hourly cap exhausted (`IsAllowedThisHour` vs `DciMaxCheckInsPerHourPerDevice`) → `decision=DeviceCapReachedDeferred` (rate window persisted, **fails closed** if the persisted window can't be validated);
3. else → `HandleTriggerAsync`.

`HandleTriggerAsync` → `AgentCommon.Mdm.DeviceTriggerBase.TriggerAsync` → `MdmSessionWrapper.RunCheckInAsync`, which is a **Windows OMA-DM generic alert**, not anything Intune-proprietary:

```
Windows.Management.MdmSessionManager.TryCreateSession()
  → MdmAlert { Source="IntuneRealTimeCompliance", Type="DriftRetry",
               Target="LocalDevice", Data="RealTimeCompliance drift retry" }
  → MdmSession.StartAsync(alerts)   // with a hard timeout
```
This forces an on-demand MDM sync with Intune. On `ApiSuccess`: `CommitDriftsToStore` (write new baselines) + `RecordSuccess` (rate-window++). On failure: `decision=TriggerFailed` with the HRESULT, baselines untouched, retried next cycle.

### 1.6 Hardening (clearly written against local tampering)
DACL-hardened registry root; setting names must match `[A-Za-z0-9_]{1,64}` (else "possible tamper"); baselines > 256 chars treated as missing/tamper; `PollIntervalSeconds` and the DCI cap clamped on read with a tamper warning; max 64 stored settings; rate window fails closed.

### 1.7 Auditing
`ComplianceMonitorAuditing.EmitHourlyCycleSummary` emits `CycleHourlyRollup` (Cycles, Drifted, DriftedSettingIds, DciCount, TotalDciDurationMs, AvgDciDurationMs, AvgWmiLatencyMs) under telemetry component `ComplianceMonitor`.

---

## 2. Compliance-script scheduling — Minute / Hourly / Defer (new cadence ladder)

The custom-compliance engine is still the `[HS]` ("Health Scripts") engine in `ScriptPlugIn.dll`. 1.103 adds a tighter cadence ladder driven by the ComplianceMonitor effort.

**New schedule contract:** `ServiceContracts.HealthScripts.MinuteSchedule` (alongside the existing `DailySchedule` / `HourlySchedule`, all extend `ScheduleBase`).

**New handlers in `ScriptPlugIn.Scheduling`:**
- `MinuteScheduleHandler.Process` — handles server-pushed `ScheduleType.Minute` (gated by flight `EnableMinuteScheduleHandling`). Logs `[HS] inspect minute schedule for policy …`; rejects non-minute payloads (`the incoming type is not minute (got {0}).`) and validates interval (`Invalid interval was detected, {0} (allowed range: {1}-{2}).`).
- `DeferOneHourScheduleHandler.Process` — fallback: if a `Minute` schedule arrives while `EnableMinuteScheduleHandling` is **off**, the policy is deferred one hour (`[HS] ScheduleType.Minute received with EnableMinuteScheduleHandling flight OFF; deferring policy 1h.`).

**Hourly override ("Phase 1"):** `ScheduleManager.Process` now checks `policyType == 8` (compliance-script policy) and the new `IsHourlyComplianceScriptCadenceEnabled()` (reads ECS flight `EnableHourlyComplianceScriptCadence`, defaults OFF). If both true, it **rewrites the schedule from `DailySchedule` (1 day) to `HourlySchedule` (1 hour)**, re-serializes the payload, and logs `[ComplianceMonitor] Phase 1 override applied for compliance-script policy {id}: forcing 1h cadence.`

**Notification-driven immediate run:** `PolicyPoller.ScheduleForNotification` (new) wakes the engine on a push notification rather than waiting for the scheduler, with NotificationID dedup (`starting / dropped, already running / idle`). This is the receiving end of the `intunemanagementextension://synccompliance` command (§3).

Net cadence ladder for custom compliance: **Daily (default) → Hourly (`EnableHourlyComplianceScriptCadence`) → Minute (server `ScheduleType.Minute` + `EnableMinuteScheduleHandling`)**, with real-time drift check-ins layered on top from §1.

---

## 3. Push-notification multi-workload dispatch (new)

1.103 reworks how a single Sidecar push notification fans out to work. New ECS flight: `EnableNotificationWorkloadDispatch`.

**`SidecarNotification.dll`** — new payload extensions `SidecarNotificationPayloadExtensions.GetAdditionalWorkloads` and `GetSyncType`. The payload can now carry a list of *additional* workloads plus a sync type.

**`AgentCommon.NotificationWorkloadHelper`** (new: `CreateClientContextFromNotification`, `TriggerAdditionalWorkloads`, `DispatchWorkload`) parses that list and dispatches each, with guardrails visible in the strings:
- caps the payload to **20** entries (`Payload contains {1} entries, capping to 20`),
- de-dupes (`Skipping duplicate workload`), skips the primary intent (`Skipping primary intent`), validates values (`Invalid/Undefined/Unsupported workload value/type`),
- dispatches via delegates and logs `[AdditionalWorkloads][{corr}] Triggering {workload}` / `… failed` / `… delegate not provided`.

Workload kinds seen: `PowerShellScriptWorkload`, `Win32AppWorkload` (and the `…_ComplianceScripts`, `…_IntunePivot`, `…_UserSession` device-action callbacks).

**Agent side (`EmsAgentService`)** gains the matching trigger methods that those delegates point at: `TriggerPowerShellScriptAsync`, `TriggerProactiveRemediationAsync`, `TriggerWin32AppAsync`, `Win32AppCheckInAsync`, and a refactor of `Win32AppAsync` into `Win32AppCoreAsync`.

Effect: one notification can now drive several workloads in one pass (e.g. push scripts + Win32 check-in together), instead of one intent per notification.

---

## 4. IC3 / Trouter push channel — reliability & telemetry rework

Large rework of the IC3 (Trouter) real-time channel. `Microsoft.IC3.Trouter.dll` grew +115 KB; the agent's IC3 code is substantially expanded.

**Proxy awareness (new):** `IC3Client.GetSystemWebProxy`, `CheckAndUpdateProxy`, `UpdateProxy`, `GetRedactedProxy`, `OnNetworkIssue`, plus interface `IIC3Client.UpdateProxy`. The channel now reads the system web proxy, reacts to proxy changes, and redacts proxy detail in logs.

**On-demand check-in (new):** `IC3Manager.TriggerImmediateCheck` / `IC3ManagerService.TriggerImmediateCheck` (+ interfaces), and `IC3GenericWorkloadHelper.ShouldPerformOnDemandCheckIn`.

**Statistics object (new `IC3Statistics`)** with counters: `StartAttempt`, `StartFailure`, `TotalMessagesReceived`, `CallbackFailure`, `ConfigInvalid`, `ParseError`, `VersionMismatch`, `DisconnectReason`, `EcsFallbackUsed`, `NetworkIssue`, `ProxyChanged`, `SystemProxyEnabled`. New `IC3DisconnectStats` / `IC3LifecycleStats` types.

**Health buckets (new strings/telemetry):** `IC3LifecycleHealth`, `IC3ConnectionHealth`, `IC3MessageHealth` ("IC3 lifecycle and configuration health"), with a disconnect-reason breakdown and average latency.

This is a robustness/observability pass on the push channel — proxy handling, fallback to ECS, and detailed disconnect/parse/version failure accounting.

---

## 5. Win32 apps — ESP registry watcher (new)

`Win32AppPlugIn.EspHelper` is new: `SetupEspRegistryWatcher`, `OnEspRegistryValueChanged`, `FindFirstSyncKeyPath`, `DisposeEspRegistryWatcher`. New flight/string `RunNontrackedAppsCheckinImmediatelyAfterEsp`.

Behaviour (from the trace strings): during Autopilot it installs a `RegistryWatcher` on the OMA-DM **FirstSync** key subtree; when ESP completes (a value change under that subtree, re-validated via `Re-evaluated ESP phase`), it fires an **immediate Win32 app workload check-in** for non-tracked apps instead of waiting for the next poll (`ESP completed. Triggering immediate app workload check-in.`). It self-disposes on a safety timeout (`Safety timeout ({0}h) reached`) and handles the race where ESP already finished before the watcher started (`Firing callback directly`).

Effect: faster delivery of non-tracked Win32 apps right after enrollment.

---

## 6. ClientHealthEval.exe — per-process ProcessMonitoring telemetry (new)

`ClientHealthEval.exe` grew +36 KB with a new `ClientHealth.ProcessMonitoring` subsystem: `DiskUsageCollector`, `PointInTimeCollector`, `SystemProcessProvider`/`IProcessProvider`, and metric records `ProcessEntry`, `ProcessMetrics`, `MemoryMetrics`, `HandleMetrics`, `ThreadMetrics`, `PerPidMetrics`, `DiskUsageEntry/Result`, `CollectionState`, plus `RuleHandler.ProcessMonitoringRuleHandler` and `Telemetry.ClientHealthTelemetryManager`.

It collects the IME's **own** resource footprint — per-PID memory/handles/threads and disk usage — emitting telemetry buckets `ClientHealth.Memory`, `ClientHealth.Handles`, `ClientHealth.Threads`, `ClientHealth.DiskUsage`, `ClientHealth.CollectionState`.

Registered in **`HealthCheck.xml`** as a new check:
```xml
<HealthCheck Description="Collect IME process resource metrics (memory, handles, threads, disk)."
             ID="d4f7a2c1-8e3b-4f5a-9c6d-2b1e0f8a7d3c" Type="ProcessMonitoring">
  <Applicability OS="ALL" ClientVersion="ALL"/>
  <PARAM Order="1" Description="Process Monitoring Check">ProcessMonitoring</PARAM>
</HealthCheck>
```

---

## 7. Azure Scale Unit (ASU) resolution rewrite

The scale-unit logic moved out of `DiscoveryService` into a dedicated `AgentCommon.CommonHelper.EnvironmentResolver`.

- **Removed:** `DiscoveryService.PopulateScaleUnitInEnvironment`.
- **Added:** `EnvironmentResolver.NormalizeAndResolveScaleUnitName`, `ParseScaleUnitFromUrl`, `NormalizeScaleUnit`, `CacheScaleUnitToRegistry`, `TryLoadScaleUnitFromCache` (+ interface `IEnvironmentResolver`).

It now parses the scale unit from the `SideCarGatewayServiceUrl`, normalizes it, and **caches it to registry** under value `AzureScaleUnit`. If the gateway URL isn't available yet at startup, it loads the scale unit from the registry cache (`SideCarGatewayServiceUrl not available yet, attempting to load scale unit from registry cache` / `Loaded scale unit from registry cache`). This makes ASU determination resilient to early-boot ordering and offline starts.

---

## 8. Telemetry consolidation

- **Removed (EXE):** `SidecarAgentCommonClientTelemetry` (`LogInfoEvent/LogErrorEvent/LogOperationEvent` + `IsSidecarGoldenSignalFlightEnabled` + `TryGettingBooleanECSFlight`) and `Telemetry.SidecarAgentHeartbeatSignal.EmitSidecarHeartbeatSignal`.
- **Added (AgentCommon):** `AgentCommonClientTelemetry` (`LogInfoEvent/LogErrorEvent/LogOperationEvent`) + interface `IAgentCommonClientTelemetry`. The logging surface moved down into AgentCommon so plugins share one path.
- **Flighting:** `FlightingManager` loses `EnableClientTelemetry` and gains `EnableRealTimeCompliance`, `EnableComplianceMonitorSimulateMode`, `EnableHourlyComplianceScriptCadence`, `EnableMinuteScheduleHandling`.
- `SetupCommonTelemetryAsync` → `SetupCommonTelemetry` (now synchronous), and a new `GetCrashCollectionLevelFromEcsAsync` (crash-collection level pulled from ECS).

---

## 9. Config-file state self-check (new)

`EmsAgentService.LogConfigFileState` is new, with strings like `ConfigFileState`, `Exists={0}; CreationTimeUtc={1}; LastWriteTimeUtc={2}; CreatedEqualsModified={3}; ReadSucceeded={4}`, and `Agent heartbeat failed during initialization`. Backed by new `AgentCommon.FileSystemWrapper` helpers (`GetFileCreationTimeUtc`, `GetFileLastWriteTimeUtc`, `GetAttributes`, `IsReparsePoint`, `TryGetDiskSpaceInfo`, `EnumerateFiles/Directories`).

At startup the agent records the state of its `.exe.config` (exists, timestamps, whether created==modified, read success). This is why 1.103 ships `*.exe.config.bak` and the `SyncAppConfigTimestamps` MSI custom action — to give the check a known-good baseline and avoid false tamper signals on first run.

---

## 10. Smaller first-party changes

- `UserAccount` rework: `RetrieveAsync`, `GetUserAccountType`, `GetUserAccountTypeBySessionId`, `EmitTelemetry`, `GetExceptionCode`, and use of a `DeviceCheckInAppId` path for AAD user retrieval in userless sessions; `NativeMethods.LookupAccountSid` added.
- `WindowsUnattendedUtilities.ResolveRemoteDesktopUsersGroupName` (new) — resolves the localized "Remote Desktop Users" group via SID for the unattended Remote Help flow.
- `EnvironmentHelper.IsEnrolledWithIntuneBLP` (new) — additional enrollment-type check.
- `SessionSettingsStore`/`SessionSettingsPersister` expanded: `PersistFromGatewayResponse`, `TryExtractSettings`, `ShouldParse`, `DeserializeSettings`, `PersistDictionary`, `LogUnexpectedPersistFault`, `SetTelemetrySink` — this is the path that lands `ComplianceMonitorConfigJson` (and other gateway settings) into SessionSettings.
- `CustomPropertiesCollectionExtensions.Clone` (new).
- `WinGetLibrary.dll` (+8.7 KB), `Microsoft.Management.Deployment.*` (+14 KB / winmd +3 KB / manifest +2.4 KB) — WinGet/COM deployment surface refresh that tracks the `WindowsPackageManager.dll` bump.
- `DataSensorPlugIn.dll` (+1 KB), `Common.Base.dll` (+4.5 KB), `Common.Telemetry.dll` (+7 KB) — supporting changes for the telemetry/health work above.
- Hash-changed but size-identical / ±8-byte first-party DLLs (`AppEnforcementProcessor`, `IntunePivotPlugIn`, `PivotParser`, `SideCarETWProvider`, `ProcessingAnalysis`, `StatusServiceLibrary`, `UnlockWin10SPlugIn`, `RebootCoordinator`, `Win32AppInventoryCollector`, etc.) are recompiles/re-signs with no observable type/method/string change.

---

## 11. New ECS flights introduced in 1.103 (quick reference)

| Flight (method / account-flight string) | Effect |
|---|---|
| `EnableRealTimeCompliance` / `EnableRealTimeComplianceAccount` | master switch for the ComplianceMonitor |
| `EnableComplianceMonitorSimulateMode` | drift detected but no check-in fired (dry run) |
| `EnableHourlyComplianceScriptCadence` | forces compliance-script policies to 1h cadence |
| `EnableMinuteScheduleHandling` | enables `ScheduleType.Minute` handling (else defer 1h) |
| `EnableNotificationWorkloadDispatch` | enables multi-workload push dispatch |
| `RunNontrackedAppsCheckinImmediatelyAfterEsp` | ESP-completion → immediate Win32 check-in |

Removed: `EnableClientTelemetry`.

## Registry surface added in 1.103

- `HKLM\SOFTWARE\Microsoft\IntuneManagementExtension\ComplianceMonitor\{Config, Config\PerSetting, Config\RateWindow, Settings}` (DACL-hardened).
- Scale-unit cache value `AzureScaleUnit` (under the IME environment key).

---

## Appendix — changed first-party binaries (by delta)

(Localized `*.resources.dll` ±40-byte churn and pure-resign no-delta DLLs omitted from the narrative; full list of changed first-party modules below.)

See the accompanying table; the modules carrying real logic change, in order of significance, are:
`IntuneWindowsAgent.exe` (+62,976), `AgentCommon.dll` (+44,544), `ClientHealthEval.exe` (+36,352), `WinGetLibrary.dll` (+8,744), `Win32AppPlugIn.dll` (+7,688), `Common.Telemetry.dll` (+7,168), `ScriptPlugIn.dll` (+5,120), `Common.Base.dll` (+4,568), `AgentCommon.ServiceContracts.dll` (+1,536), `DataSensorPlugIn.dll` (+1,024), `SidecarNotification.dll` (+552), `Flighting.dll` (+472), `AgentExecutor.exe` (+560), `BootstrapperAgentCore.dll` (+2,048).

> Method of analysis: MSI CAB extraction (`msitools`), .NET metadata/IL parsing (`dnfile` + `dncil`), set-diff of TypeDef/MethodDef/#US/#Constant streams, and targeted IL disassembly of changed methods. All findings are from the uploaded binaries; flight-gated behaviour is inferred from code paths and log/constant strings, not from runtime observation.
