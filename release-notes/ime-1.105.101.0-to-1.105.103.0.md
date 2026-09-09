# IME 1.105.101.0 → 1.105.103.0

## Scope and Methodology

This is an MSI-level and managed-assembly metadata comparison of two consecutive IME builds, in the style established for this tracker:

- MSI database tables (Property, File, CustomAction, Registry, ServiceInstall, Directory, InstallExecuteSequence) diffed via `msidump`
- All 153 payload files hashed with SHA-256 to isolate genuinely changed files from byte-identical ones
- Changed managed assemblies inspected at the metadata level with `dnfile` — TypeDef/MethodDef counts, added/removed type names, P/Invoke (`ImplMap`) entries, AssemblyRef changes
- No IL decompilation was performed; findings come from package structure, assembly metadata shape, and cross-file correlation

**Builds:** 1.105.101.0 (built 2026‑08‑11) → 1.105.103.0 (built 2026‑08‑25), both WiX Toolset 5.0.2.0.

---

## Key MSI-Level Findings

- MSI grows ~24 KB (13.729 MB → 13.754 MB)
- **0 files added, 0 files removed** — payload stays at 153 files
- **75 of 153 files are byte-different; 78 are byte-identical**
- ProductCode changes (`{EA1E4C91-…}` → `{DDF6D6BF-…}`); UpgradeCode unchanged (`{9FE9701C-0F89-40B4-B77A-AA65607E87D8}`)
- **No changes** to CustomAction, Registry, ServiceInstall, Directory, or InstallExecuteSequence tables — install mechanics are untouched

**Noise vs. signal:** of the 75 changed files, 72 (every satellite resource DLL, every plug-in DLL, and most EXEs) shift by ≤48 bytes with **zero** TypeDef/MethodDef delta at the metadata level — consistent with a deterministic-build recompile triggered by the version bump alone, not functional change. Spot-checked (`TamperProtection.dll`, `ImeUI.exe`, `Newtonsoft.Json.dll`): all show 0/0 type and method deltas. Three files carry real code changes.

---

## The One Real Feature: Event-Driven Compliance Monitoring

### `Microsoft.Management.Services.IntuneWindowsAgent.exe` (+65,024 bytes)
TypeDef 251 → 300 (+49), MethodDef 1,144 → 1,348 (+204).

Adds an entire `ComplianceMonitor.EventListening` namespace plus a generic, reusable event-source framework:

- **New generic infrastructure:** `IEventSource`, `IEventPublisher<T>`, `EventSourceBase<T>`, `EventSourceListener`, `EventLogEventSource<T>` — a typed pub/sub layer for reacting to OS events rather than polling for them.
- **Compliance-specific wiring:** `IComplianceEventSourceCatalog` / `ComplianceEventSourceCatalog`, `ComplianceEventPublisher`, `ComplianceEventSignal`, `RegistryEventSourceRegistration` (a `RegistryEventSource<T>`), and `WindowsSecurityCenterEventSourceRegistration` / `WindowsSecurityCenterWatcher` (a `WindowsSecurityCenterEventSource<T>`).
- **New P/Invoke:** `WscRegisterForChanges` / `WscUnRegisterChanges` from `wscapi.dll` — the agent now registers directly with the **Windows Security Center API** to get pushed notifications (AV/firewall/AutoUpdate state changes) instead of only sampling state periodically.
- Internally, the older `Execute/HandleDrifts/HandleTrigger/RunOneCycle` async state machine was renumbered/extended (`d__39`→`d__56` range) rather than replaced, suggesting this event layer feeds into the existing compliance evaluation cycle rather than replacing it outright.

### `Microsoft.Management.Services.IntuneWindowsAgent.AgentCommon.dll` (+1,064 bytes)
TypeDef unchanged (658), MethodDef +2: `WaitUntilStarted`, `SignalMonitoringStarted` — a startup handshake so dependents can block until the new event-listening subsystem has actually come up.

### `…AgentCommon.ServiceContracts.dll` (+984 bytes)
TypeDef unchanged (36), MethodDef +6 — three new config properties (getter+setter each):
- `ComplianceEventProcessorBatchTimeSeconds`
- `ComplianceEventProcessorDelayTimeSeconds`
- `EventPerSetting`

Batching/delay knobs for the new compliance-event pipeline, plus a per-setting toggle for whether individual settings raise their own events.

---

## Custom Action and MSI Mechanics

Unchanged. `CustomAction`, `Registry`, `ServiceInstall`, `Directory`, and `InstallExecuteSequence` tables are byte-for-byte identical between the two builds — this release carries no new install-time logic, permissions, or scheduled-task/service definitions.

---

## Confidence Assessment

**High confidence (directly visible):**
- MSI table diffs (no changes) and file-level hash diffs (75/153 changed)
- New TypeDef/MethodDef surface in the three changed assemblies, and the `wscapi.dll` P/Invoke addition
- Zero-delta confirmation on the 72 "noise" files via metadata inspection, not just file size

**Strong inference (no IL decompilation):**
- That the new `EventListening` framework is *wired into* the existing compliance-evaluation cycle rather than dormant — inferred from the renumbering pattern of the pre-existing async cycle methods, not from call-graph analysis
- Exact trigger conditions for when WSC-sourced events cause an out-of-cycle compliance re-evaluation

**Server/rollout dependent:**
- Whether this event-driven path is active by default or gated behind an ECS/flighting check (no flighting-table or flight-ID string changes were observed in this pair, unlike the 1.105.103.0→1.105.152.0 pair where signature-validation flighting is explicit)

## Headline

*"1.105.103.0 is a narrow, single-feature release. Nearly every other file in the package is a version-stamp recompile with no metadata change. The one real addition is a Windows-Security-Center-backed, event-driven compliance signal path — layered onto the agent's existing polling cycle rather than replacing it — with new batching/delay policy knobs to control it."*
