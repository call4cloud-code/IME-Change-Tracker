# Intune SideCar Flight Tracker

Last run: `2026-08-11 07:00:06 UTC`

| Area | Total flights | Enabled flights |
| --- | ---: | ---: |
| Selfhost | 46 | 44 |
| PE | 22 | 21 |

Selfhost vs PE differences in this run: **24**

Flight events since previous snapshot: **1**

| Environment | Added | Added enabled | Enabled | Disabled | Value changed | Expiry changed | Removed |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Selfhost | 0 | 1 | 0 | 0 | 0 | 0 | 0 |
| PE | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

## Flight Events Since Previous Snapshot

| Environment | Event | Flight | Previous | Current | Previous expiry | Current expiry |
| --- | --- | --- | --- | --- | --- | --- |
| Selfhost | Added enabled | `ValidateRolloutTemplate` |  | `true` |  | `2025-10-01T00:00:00.000Z` |

## Current Selfhost vs PE Differences

| Status | Flight | Selfhost | PE | Different fields |
| --- | --- | --- | --- | --- |
| Only in selfhost | `APv2ScriptNullRefFix` | `true` |  |  |
| Only in selfhost | `CollectAppInvForNonEmptyGuids` | `true` |  |  |
| Only in selfhost | `dnsErrorRetryDelay` | `true` |  |  |
| Only in selfhost | `EmitSidecarMsiInstallerEvent` | `true` |  |  |
| Only in selfhost | `EnableCleanupMsiKeyOnUninstall` | `true` |  |  |
| Only in selfhost | `EnableComplianceMonitorSimulateMode` | `false` |  |  |
| Only in selfhost | `EnableComponentManager` | `true` |  |  |
| Only in selfhost | `EnableExpandedCrashTelemetry` | `true` |  |  |
| Only in selfhost | `EnableHourlyComplianceScriptCadence` | `true` |  |  |
| Only in selfhost | `EnableIC3Telemetry` | `true` |  |  |
| Only in selfhost | `EnableIsAADUserTelemetry` | `true` |  |  |
| Only in selfhost | `EnableNotificationDedupeCache` | `true` |  |  |
| Only in selfhost | `EnableOneDSProxyFeature` | `true` |  |  |
| Only in selfhost | `EnablePutWithTupleResult` | `true` |  |  |
| Only in selfhost | `EnableRealTimeComplianceAccount` | `true` |  |  |
| Only in selfhost | `EnableRealTimeComplianceDevice` | `true` |  |  |
| Only in selfhost | `EnableStartupOptimization` | `true` |  |  |
| Only in selfhost | `featureX` | `true` |  |  |
| Only in selfhost | `OperationSpanLoggingEnabled` | `true` |  |  |
| Only in selfhost | `RunNontrackedAppsCheckinImmediatelyAfterDpp` | `true` |  |  |
| Only in selfhost | `RunNontrackedAppsCheckinImmediatelyAfterEsp` | `true` |  |  |
| Only in selfhost | `ValidateRolloutTemplate` | `true` |  |  |
| Only in selfhost | `win32AppEnforcementTelemetryEnabled` | `true` |  |  |
| Only in selfhost | `wingetRetryOnTransientErrors` | `true` |  |  |

## Selfhost Enabled Flights

| Flight | Value | Expiry |
| --- | --- | --- |
| `APv2EnableAutoReconnectOnRestart` | `true` | `2027-03-17T00:00:00.0402432Z` |
| `APv2ScriptNullRefFix` | `true` | `2027-06-01T00:00:00.000Z` |
| `APv2UseStandardUserProviderNative` | `true` | `2026-06-20T00:00:00.0402432Z` |
| `ClearGRSInESP` | `true` | `2026-04-01T00:00:00.000Z` |
| `CollectAppInvForNonEmptyGuids` | `true` | `2026-04-01T00:00:00.000Z` |
| `CommonSchemaEventsProcessing` | `true` | `2025-08-01T00:00:00.000Z` |
| `dnsErrorRetryDelay` | `true` | `2025-09-01T00:00:00.000Z` |
| `EmitSidecarMsiInstallerEvent` | `true` | `2026-12-18T00:00:00.000Z` |
| `EnableCleanupMsiKeyOnUninstall` | `true` | `2026-12-18T00:00:00.000Z` |
| `EnableComponentManager` | `true` | `2026-10-18T00:00:00.000Z` |
| `EnableDiskUsageMonitor` | `true` | `2026-10-18T00:00:00.000Z` |
| `EnableExpandedCrashTelemetry` | `true` | `2026-05-18T00:00:00.000Z` |
| `EnableGoldenSignals` | `true` | `2025-10-18T00:00:00.000Z` |
| `EnableHourlyComplianceScriptCadence` | `true` | `2026-11-07T00:00:00Z` |
| `EnableIC3Feature` | `true` | `2027-06-01T00:00:00.000Z` |
| `EnableIC3Telemetry` | `true` | `2026-06-01T00:00:00.000Z` |
| `EnableIsAADUserTelemetry` | `true` | `2026-06-01T00:00:00.000Z` |
| `EnableNotificationDedupeCache` | `true` | `2026-06-01T00:00:00.000Z` |
| `EnableNotificationWorkloadDispatch` | `true` | `2026-06-01T00:00:00.000Z` |
| `EnableOneDSProxyFeature` | `true` | `2026-12-31T00:00:01.000Z` |
| `EnableProcessMonitoring` | `true` | `2026-10-18T00:00:00.000Z` |
| `EnablePutWithTupleResult` | `true` | `2025-08-18T00:00:00.000Z` |
| `EnableRealTimeComplianceAccount` | `true` | `2026-11-07T00:00:00Z` |
| `EnableRealTimeComplianceDevice` | `true` | `2026-11-07T00:00:00Z` |
| `EnableStartupOptimization` | `true` | `2026-10-01T00:00:00.000Z` |
| `EnableWin32PowerMgmt` | `true` | `2025-11-01T00:00:01.000Z` |
| `ETWEventsAggregation` | `true` | `2025-06-01T00:00:00.000Z` |
| `featureX` | `true` | `2025-04-23T18:25:43.511Z` |
| `FileStreamForDetectionScripts` | `true` | `2025-05-01T00:00:01.000Z` |
| `IsPowershellScriptConcurrentProcessingEnabled` | `true` | `2026-06-01T00:00:00.000Z` |
| `OperationSpanLoggingEnabled` | `true` | `2027-12-31T23:59:59.0000000Z` |
| `redirectFsForScriptsEnabled` | `true` | `2026-11-01T00:00:01.000Z` |
| `RemediationSendResultFirst` | `true` | `2025-04-28T00:00:00.0402432Z` |
| `RouteInventoryEventsToSidecarDB` | `true` | `2026-12-31T00:00:01.000Z` |
| `RunNontrackedAppsCheckinImmediatelyAfterDpp` | `true` | `2027-03-17T00:00:00.0402432Z` |
| `RunNontrackedAppsCheckinImmediatelyAfterEsp` | `true` | `2027-03-17T00:00:00.0402432Z` |
| `SetIMELogACL` | `true` | `2026-03-04T00:00:01.000Z` |
| `SingletonBootstrapperFlight` | `true` | `2026-06-19T00:00:00.0402432Z` |
| `skipUserContextAppsWithoutUserLoggedInEnabled` | `true` | `2026-06-01T00:00:01.000Z` |
| `TcpProbe` | `true` | `2026-08-01T00:00:00.000Z` |
| `ValidateRolloutTemplate` | `true` | `2025-10-01T00:00:00.000Z` |
| `win32AppEnforcementTelemetryEnabled` | `true` | `2027-12-31T23:59:59.0000000Z` |
| `wingetAppTimeoutHandlerEnabled` | `true` | `2025-06-30T23:59:00Z` |
| `wingetRetryOnTransientErrors` | `true` | `2026-06-04T00:00:00.000Z` |

## PE Enabled Flights

| Flight | Value | Expiry |
| --- | --- | --- |
| `APv2EnableAutoReconnectOnRestart` | `true` | `2027-03-17T00:00:00.0402432Z` |
| `APv2UseStandardUserProviderNative` | `true` | `2026-06-20T00:00:00.0402432Z` |
| `ClearGRSInESP` | `true` | `2026-04-01T00:00:00.000Z` |
| `CommonSchemaEventsProcessing` | `true` | `2025-08-01T00:00:00.000Z` |
| `EnableDiskUsageMonitor` | `true` | `2026-10-18T00:00:00.000Z` |
| `EnableGoldenSignals` | `true` | `2025-10-18T00:00:00.000Z` |
| `EnableIC3Feature` | `true` | `2027-06-01T00:00:00.000Z` |
| `EnableNotificationWorkloadDispatch` | `true` | `2026-06-01T00:00:00.000Z` |
| `EnableProcessMonitoring` | `true` | `2026-10-18T00:00:00.000Z` |
| `EnableWin32PowerMgmt` | `true` | `2025-11-01T00:00:01.000Z` |
| `ETWEventsAggregation` | `true` | `2025-06-01T00:00:00.000Z` |
| `FileStreamForDetectionScripts` | `true` | `2025-05-01T00:00:01.000Z` |
| `IsPowershellScriptConcurrentProcessingEnabled` | `true` | `2026-06-01T00:00:00.000Z` |
| `redirectFsForScriptsEnabled` | `true` | `2026-11-01T00:00:01.000Z` |
| `RemediationSendResultFirst` | `true` | `2025-04-28T00:00:00.0402432Z` |
| `RouteInventoryEventsToSidecarDB` | `true` | `2026-12-31T00:00:01.000Z` |
| `SetIMELogACL` | `true` | `2026-03-04T00:00:01.000Z` |
| `SingletonBootstrapperFlight` | `true` | `2026-06-19T00:00:00.0402432Z` |
| `skipUserContextAppsWithoutUserLoggedInEnabled` | `true` | `2026-06-01T00:00:01.000Z` |
| `TcpProbe` | `true` | `2026-08-01T00:00:00.000Z` |
| `wingetAppTimeoutHandlerEnabled` | `true` | `2025-06-30T23:59:00Z` |

## Files

| File | Purpose |
| --- | --- |
| `latest/sidecar_tracker_events.csv` | Added, enabled, disabled, removed, value changed, and expiry changed events compared with the previous snapshot. |
| `latest/sidecar_tracker_changes.csv` | Current selfhost vs PE differences. |
| `latest/sidecar_tracker_comparison.csv` | Full current comparison. |
| `latest/sidecar_selfhost_enabled.csv` | Current enabled selfhost flights. |
| `latest/sidecar_pe_enabled.csv` | Current enabled PE flights. |
| `latest/sidecar_selfhost.json` | Raw normalized selfhost ECS response. |
| `latest/sidecar_pe.json` | Raw normalized PE ECS response. |
