# IME 1.105.103.0 → 1.105.152.0

## Scope and Methodology

MSI-level and managed/native-metadata comparison, consistent with the rest of this tracker:

- MSI database tables diffed via `msidump`; all payload files SHA-256 hashed
- Changed managed assemblies inspected with `dnfile` (TypeDef/MethodDef counts, added/removed types, P/Invoke, AssemblyRef changes) — no IL decompilation
- Changed native DLLs inspected with `pefile` for exports/imports and target machine type
- String diffing across the largest changed files for corroborating detail (embedded cert/PKI URLs)

**Builds:** 1.105.103.0 (built 2026‑08‑25) → 1.105.152.0 (built 2026‑09‑04), both WiX Toolset 5.0.2.0. Note the version jump skips 1.105.1xx numbering entirely — this is a larger release than the previous pair.

---

## Key MSI-Level Findings

- MSI grows ~36 KB (13.754 MB → 13.791 MB)
- **1 file added** (`Microsoft.Management.Clients.Common.SignatureValidation.dll`, 47,440 bytes), **0 removed**
- **92 of 154 files are byte-different; 61 are byte-identical**
- ProductCode changes; UpgradeCode unchanged
- **No changes** to CustomAction, Registry, ServiceInstall, Directory, or InstallExecuteSequence tables — again, no new install-time mechanics

**Architecture change buried in the noise:** two native DLLs — `CoreLib.dll` and `SignatureValidationLibrary.dll` — silently **flip from x64 (PE32+, machine 0x8664) to x86 (PE32, machine 0x14c)**. This is why `CoreLib.dll` shows the single largest size delta in the package (**‑146,944 bytes**) despite being a "changed", not "added", file — it's a smaller architecture's build of the same component, not a trimmed version of the same binary. Every other native/managed file in the package was already x86 (`0x14c`) in both builds; only these two — the WinDC trust libraries previously noted as distinct from the WinRE AMD64 lineage — made the jump this release. Both also carry a coordinated version bump, 7.1.1.28 → 7.1.1.65.

---

## Major Theme: Signature-Validation and Certificate-Pinning Overhaul

This release ties together roughly a dozen files around one subsystem. The chain of evidence:

### New assembly: `Microsoft.Management.Clients.Common.SignatureValidation.dll` (added, 6.0.0.0)
Referenced via new `AssemblyRef` entries from `ClientHealthEval.exe`, `AgentCommon.dll`, and `IntuneWindowsAgent.exe` — this is a new shared library, not an isolated add. String-diffing it turns up embedded PKI metadata: AIA/CDP URLs for **Microsoft Root Cert Authority 2010/2011** and, notably, the **`Microsoft Code Signing PCA 2024`** CRL/cert URLs — strongly suggesting an embedded/pinned certificate resource used to validate signatures against a specific, current Microsoft code-signing root.

### `SignatureValidationLibrary.dll` — rebuilt x64→x86, exports change shape entirely
Old (x64) exports use the `__cdecl`/64-bit name-mangling convention (`YAJ…PEBG…`). New (x86) exports use `__stdcall` with numeric suffixes (`_VerifyTlsCertificatePinning@52`) and add capability that didn't exist before:
- `VerifyDetachedCmsSignatureEx` / `VerifyDetachedCmsSignatureEx2` — CMS/PKCS#7 detached-signature verification
- `VerifyTlsCertificatePinning` / `_VerifyLeafCertDynamicPinning@40` — TLS certificate pinning, including a *dynamic* pinning variant
- `ConfigureSignatureValidationLogging` / `ConfigureSignatureValidationTelemetry` — dedicated logging/telemetry hooks for this subsystem

### `dynpin_cxx.dll` (Rust/cxx-bridge "dynamic pin" library) — new export + new import
- `+ jws_ecdsa_signature_to_der` — a JWS→DER ECDSA signature format converter (consistent with verifying JWS-format signatures against X.509/DER-based chain-validation APIs)
- `+ SetFileTime` (kernel32) — new ability to set a file's timestamp, which shows up again below in the download/staging rework

### `IntunePolicyStorage.dll` (both the root and `PolicyAdapter\` copies) — new imports
`CertCreateCertificateChainEngine` / `CertFreeCertificateChainEngine` (CRYPT32) — building a **custom certificate chain engine**, i.e. validating against a specific trust root set rather than the default system store. Consistent with the pinned-cert material found above.

### `Microsoft.Management.Clients.Components.Runtime.dll` (+124,888 bytes — largest managed delta)
TypeDef 108 → 195 (+87), MethodDef 344 → 811 (+467). New P/Invoke: `WinVerifyTrust`, `WTHelperGetProvSignerFromChain`, `WTHelperProvDataFromStateData` (wintrust.dll), `CertVerifyCertificateChainPolicy` (crypt32.dll) — the managed component host now calls into WinTrust/Authenticode verification directly. The old flat `SignatureValidator` / `ICertificateValidator` / `ComponentDownloader` types are **removed** and replaced by a much larger, layered design (see next section) that includes a `Download.Verification.IntegrityVerifier` / `PayloadVerifierAdapter`.

### `Microsoft.Management.Clients.Policy.Runtime.dll` — new `Flighting` namespace
`ISignatureValidationModeRegistry`, `PolicySignatureValidationMode`, `SignatureValidationModeRegistry`, `SignatureValidationModeApplyResult/Outcome`, `SignatureValidationFlightSyncJob`, `SignatureValidationTelemetry` — signature validation now has a **policy-controlled rollout mode** with its own sync job and telemetry, i.e. this is being staged/flighted rather than switched on unconditionally.

### `Microsoft.Management.Services.IntuneWindowsAgent.AgentCommon.dll` (+9,176 bytes)
New `TlsCertificatePinningContext` / `TlsCertificatePinningFlightStateProvider` / `TlsCertificatePinningMode` types, wired into the outbound web-request pipeline (`SendWebRequest*` methods renumbered/extended). TLS certificate pinning is now flight-gated at the HTTP-client level, not just at the package-signature level.

### `Microsoft.Management.IntuneComponent.exe` (+5,120 bytes)
New `ValidatedAssemblyPath` type plus new P/Invoke `CreateFile` / `GetFinalPathNameByHandle` (kernel32) — resolves a file's *final* path (through reparse points/symlinks/junctions) before validating it. This is a standard mitigation against path-spoofing tricks that bypass a signature check by swapping the target after validation — a hardening step for whatever gets fed into the new signature-verification path.

**Taken together:** this release replaces an ad-hoc, per-component signature check with a centralized, WinTrust/CMS-based verification subsystem (native x86 rebuild of the WinDC libraries, a new managed wrapper assembly carrying a pinned Microsoft code-signing cert, custom cert-chain engines, TLS certificate pinning down to the HTTP client, and a path-canonicalization hardening step) — rolled out under an explicit feature-flight rather than turned on outright.

---

## Second Theme: Component Download/Update Runtime Rewritten

Independent of the signature work above, `Components.Runtime.dll` and `Common.Base.dll` show a broader architectural rewrite of how the agent's own components (the WinDC libraries, etc.) get configured, downloaded, staged, and enforced:

- **`Configuration`** namespace (new): `ComponentConfigurationPolicy(Parser)`, `ComponentConfigurationReconciler/Resolver`, `ComponentDriftStatus`, `LocalPackageCatalog/Baseline`, `ReconciliationSummary`, `ResolvedComponent/DesiredState` — a declarative desired-state-vs-actual-state reconciliation model.
- **`Download.Orchestration`**: `DownloadOrchestrator`, `DownloadDeliveryStatus`
- **`Download.Retention`**: `StagedComponentCleaner`, `RetentionFileSystem`, `ComponentRetentionConfiguration`
- **`Download.Staging`**: `StagingManager`, `StagedComponentLayout`
- **`Download.Verification`**: `IntegrityVerifier`, `PayloadVerifierAdapter` (ties back into the signature-validation subsystem above)
- **`Job`**: `ComponentConfigurationReconcileJob`, `ComponentMaintenanceJob`, `DownloadManagerJob` — background scheduling for the above

In `Common.Base.dll` (+33,792 bytes), a new **Delivery Optimization / BITS** download path is added alongside the existing HTTP downloader: `BackgroundCopyManager`, `BitsDownloadJob`, `BitsInterop`, `IBackgroundCopyJob/Manager/Callback/Error` (BITS COM interop), plumbed through a new `PayloadDownloaderFactory` / `IReliableDownloader` / `ReliableDownloaderFactory` abstraction with `DownloadRetryOptions`. A `NetworkCostChecker` (`INetworkCostManager`, wrapping `NetworkListManager`) is also added — the downloader can now check for metered/constrained connections. The old inline `CertificateValidator` / `ICertificateValidator` is removed here too (consolidated into the new SignatureValidation assembly, per the theme above). `System.Net.Http.WinHttpHandler` bumps 9.0.0.16 → 9.0.0.17.

---

## Third Theme: Expanded Network/TLS Health Probing

`ClientHealthEval.exe` (+40,960 bytes — TypeDef 93→133, MethodDef 481→686) gains an entire `ClientHealth.Network` namespace: a generic, pluggable connectivity-probe framework — `IConnectivityProbe`, `ConnectivityProbeFactory/Runner/Registry`, `IProbeExecutor` with `HttpProbeExecutor`/`TcpProbeExecutor` implementations, `AfdHttpProbe`/`AfdTcpProbe` (Azure Front Door-specific probes), `IC3EndpointResolver`, `TlsCertAuditProbeExecutor` / `TlsCertAuditEndpointResolver` (a dedicated TLS-certificate audit probe), and `SslStreamTlsHandshakeClient` implementing `ITlsHandshakeClient`. It also picks up the new `SignatureValidation` AssemblyRef via a `TlsCertAuditSignatureValidationLogAdapter` — the new TLS audit probe logs through the same pipeline as the signature-validation subsystem above. This reads as infrastructure for pre-flight/ongoing network and TLS-trust health checks, generalized well beyond a single hardcoded connectivity test.

---

## Custom Action and MSI Mechanics

Unchanged, as with the previous release. All new capability here is code shipped inside existing files (plus one new DLL added to the existing `INSTALLLOCATION` component) — no new custom actions, registry rows, services, or install sequence entries.

---

## Confidence Assessment

**High confidence (directly visible):**
- MSI structural changes, the one added file, x64→x86 architecture flip on the two native trust libraries (confirmed independently via `file`/PE machine field, not inferred from size alone)
- New types, P/Invoke tables, and AssemblyRef changes across all files listed above
- Embedded PKI/cert URL strings in the new SignatureValidation assembly

**Strong inference (no IL decompilation):**
- That TLS certificate pinning, CMS signature verification, and the component-download integrity verifier are all one coordinated effort rather than coincidental — inferred from shared AssemblyRefs, shared native library dependencies, and a shared `Flighting`-gated rollout mechanism, not from a traced call graph
- Which exact code paths call the new `WinVerifyTrust`/CMS verification first at runtime

**Server/rollout dependent:**
- Actual enforcement of TLS pinning and the new signature-validation mode depends on the `PolicySignatureValidationMode` flight value the client resolves at runtime — this release ships the mechanism, not necessarily an enabled-by-default behavior change
- BITS/Delivery-Optimization download usage vs. the existing HTTP path likely depends on `DownloadPolicyParser`-resolved policy, not observed here

## Headline

*"1.105.152.0 is a substantial release built around one coordinated effort: replacing ad-hoc signature checks with a centralized, WinTrust/CMS-based validation and TLS-pinning subsystem — including a quiet x64→x86 rebuild of the two native trust libraries — rolled out behind an explicit feature flight. Riding alongside it: a full rewrite of the component download/staging/retention pipeline (adding BITS/Delivery-Optimization support) and a generalized network/TLS health-probing framework. None of it required a single new custom action, registry key, or service definition."*
