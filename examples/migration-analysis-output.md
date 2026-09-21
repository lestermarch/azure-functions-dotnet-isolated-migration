# Example migration assessment output

> Captured from an actual GitHub Copilot Plan mode run against the repository and revision documented in this directory's [README](README.md). The output below is preserved as generated and is a point-in-time example, not current authoritative guidance.

## Executive Summary

**Fta.DemoFunc.Api** is a **.NET 6, in-process model** Azure Functions app (Functions runtime v4) with a single HTTP-triggered function. It **must migrate before 10 November 2026** when in-process support ends. The app is small, has no Durable Functions, no framework-only or native dependencies, and its test suite already calls the function class directly rather than through the WebJobs host — this is one of the lowest-risk migration profiles you'll see. I recommend migrating straight to **.NET 8 or .NET 10 isolated worker** (see rationale below) rather than .NET Framework 4.8 isolated.

No files have been changed. No Azure resources were inspected or altered — this repo has no CI/CD workflows or IaC checked in, so rollout planning below is generic guidance for whatever pipeline/portal process is currently used to deploy this app.

---

## Current State Assessment

| Aspect | Finding |
|---|---|
| Target framework | `net6.0` (`Fta.DemoFunc.Api.csproj`) — itself out of support (net6 EOL Nov 2024), compounding urgency |
| Functions runtime | `AzureFunctionsVersion v4`, `FUNCTIONS_WORKER_RUNTIME=dotnet` (in local settings sample in `README.md`) |
| Hosting model | **In-process** — confirmed by: |
| | • `Microsoft.NET.Sdk.Functions` v4.2.0 package reference |
| | • `Microsoft.Azure.Functions.Extensions` v1.1.0 (in-process DI package) |
| | • `[assembly: FunctionsStartup(typeof(Startup))]` + `FunctionsStartup`/`IFunctionsHostBuilder` in `Startup.cs` |
| | • `[FunctionName("NotesFunction")]` + `Microsoft.Azure.WebJobs`/`Microsoft.Azure.WebJobs.Extensions.Http` in `NotesFunction.cs` |
| | • `ActionResult<T>` / `System.Web.Http` result types (in-process HTTP model) |
| Triggers/bindings | Single HTTP trigger only (`notes` POST). No Service Bus, Event Hubs, Cosmos DB, Storage, Durable Functions, or timer triggers. |
| Dependencies | Dapper 2.1.24, Npgsql 6.0.10, Polly (via `Microsoft.Extensions.Http.Polly`) — all fully supported on modern .NET, no COM/native/Windows-only APIs. |
| DI/config | Custom `AddApplication`/`AddInfrastructure` extension methods in `DependencyInjection.cs`, wired through `FunctionsStartup.Configure`. Plain `IServiceCollection` registrations — portable as-is to isolated worker's `ConfigureServices`. |
| App Insights | Configured via `host.json` `logging.applicationInsights` sampling settings — standard in-process pattern; needs to move to isolated worker's `ConfigureFunctionsWebApplication`/`AddApplicationInsightsTelemetryWorkerService` pattern. |
| Deployment/CI/CD/IaC | **None found in this repo** — no `.github/workflows`, no Bicep/ARM/Terraform, no `azure-pipelines.yml`. Deployment path (slots, publish profile, pipeline) is external to this repo and not assessable here. |
| Tests | Unit tests (`NotesFunctionTests.cs`) instantiate `NotesFunction` directly and call `.Post(...)` — no host/runtime dependency, so these keep working with minimal change. Integration tests use Testcontainers + WireMock against the real app via HTTP, not the in-process pipeline. |
| .NET Framework-only or native blockers | **None identified.** All dependencies are cross-platform, actively maintained NuGet packages. |

---

## Recommended Target and Decision Rationale

**Recommended: .NET 8 (LTS) isolated, with .NET 10 isolated as the preferred if you want the newest STS.**

- The app and every dependency (Dapper, Npgsql, Polly/`Microsoft.Extensions.Http.Polly`) are proven compatible with modern .NET — no COM, no Windows-only P/Invoke, no framework-only APIs.
- Per the migration workflow, **.NET Framework 4.8 isolated is only appropriate when a dependency truly requires .NET Framework**. That's not the case here, so choosing it would be a strictly worse choice: same migration effort, but you'd inherit a second, separate modernization project later (Framework → modern .NET) instead of doing it once.
- Between .NET 8 and .NET 10: since the Learn article you referenced targets `net10`, and .NET 10 is the current isolated-worker baseline, .NET 10 is the more future-proof pick if your Azure Functions plan/OS/tooling supports it today. If you want the more conservative LTS choice for a production cutover this cycle, .NET 8 is equally valid and slightly lower risk on tooling maturity. **No unresolved compatibility gate exists — this is a preference between two supported LTS/STS targets, not a technical blocker.**

---

## Migration Workstreams

### 1. Project file — `Fta.DemoFunc.Api.csproj`
- Change `<TargetFramework>` from `net6.0` → `net10.0` (or `net8.0`).
- Change SDK to `Microsoft.NET.Sdk` with `<OutputType>Exe</OutputType>` and `AzureFunctionsVersion` → `v4` with isolated worker markers (per current Learn guidance, project uses `Microsoft.Azure.Functions.Worker.Sdk` extension, not `Microsoft.NET.Sdk.Functions`).
- Remove: `Microsoft.NET.Sdk.Functions`, `Microsoft.Azure.Functions.Extensions`.
- Add: `Microsoft.Azure.Functions.Worker`, `Microsoft.Azure.Functions.Worker.Sdk`, `Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore` (for `ActionResult`-style HTTP), `Microsoft.ApplicationInsights.WorkerService` + `Microsoft.Azure.Functions.Worker.ApplicationInsights` if App Insights is used.
- Keep Dapper/Npgsql/Polly references unchanged — no isolated-worker equivalents needed.

### 2. New `Program.cs`
- Add isolated worker host builder using `ConfigureFunctionsWebApplication()` (this app is HTTP-only, so the ASP.NET Core integration is the right choice for `ActionResult`-style responses).
- Move `AddApplication(configuration)` / `AddInfrastructure(configuration)` calls from `Startup.Configure` into the new host's `ConfigureServices`.
- Configure Application Insights explicitly (`AddApplicationInsightsTelemetryWorkerService()` + `ConfigureFunctionsApplicationInsights()`).

### 3. Remove `Startup.cs`
- Delete `FunctionsStartup`/`IFunctionsHostBuilder` class and the assembly attribute once its registrations are confirmed moved into `Program.cs`.

### 4. `NotesFunction.cs`
- `[FunctionName("NotesFunction")]` → `[Function("NotesFunction")]`.
- `Microsoft.Azure.WebJobs`/`Microsoft.Azure.WebJobs.Extensions.Http` → `Microsoft.Azure.Functions.Worker`/`Microsoft.Azure.Functions.Worker.Http`.
- `System.Web.Http` result types (`BadRequestObjectResult`, `InternalServerErrorResult` from that namespace) need review — with the ASP.NET Core HTTP extension, use `Microsoft.AspNetCore.Mvc` result types consistently (the code already mixes `Microsoft.AspNetCore.Mvc.CreatedResult`/`BadRequestObjectResult`, so this is mostly a namespace cleanup, not a rewrite).
- JSON serialization: confirm `System.Text.Json` behavior for `CreateNoteRequest`/`CreateNoteResponse` model binding still matches — isolated ASP.NET Core integration reuses ASP.NET Core's own JSON pipeline, low risk here.
- `ILogger<NotesFunction>` constructor injection pattern is unchanged and portable as-is.

### 5. `host.json`
- Keep `AzureFunctionsVersion`/version block; move the `logging.applicationInsights` sampling settings only if the new isolated App Insights configuration in `Program.cs` needs matching adjustments (worker-side sampling is configured through the SDK, host-side settings mostly still apply for host telemetry).

### 6. `local.settings.json`
- Update `FUNCTIONS_WORKER_RUNTIME` from `dotnet` → `dotnet-isolated` (local dev only — never commit this file, per README guidance it's already gitignored/sample-only).

### 7. Tests
- **Unit tests**: no structural change needed — they instantiate `NotesFunction` and call `.Post(...)` directly, independent of the hosting model. Only update if any `FunctionName`/attribute-level assertions exist (none found).
- **Integration tests**: exercise the deployed HTTP endpoint via WireMock/Testcontainers — unaffected by the hosting model change, but must be re-run against the migrated app to confirm behavior parity.

---

## Validation Checklist

- [ ] Restore packages against the new TFM/SDK.
- [ ] Build in Release configuration — resolve all compiler errors from removed in-process types first pass, then a second pass for warnings.
- [ ] Confirm build output contains `Fta.DemoFunc.Api.exe`/isolated worker host artifacts, with **no** residual `Microsoft.Azure.WebJobs.*`/`Microsoft.NET.Sdk.Functions` references.
- [ ] Run `func start` (Core Tools v4) locally with `local.settings.json` set to `dotnet-isolated`; call `POST /api/notes` and confirm 201/400/500 paths all behave as before.
- [ ] Run the full unit test suite (`Fta.DemoFunc.Api.Tests.Unit`) — expect no regressions.
- [ ] Run the integration test suite (`Fta.DemoFunc.Api.Tests.Integration`, requires Docker) — confirm Postgres persistence and the WireMock notification call both still succeed.
- [ ] Verify Application Insights telemetry (request, dependency, log correlation) appears correctly from the isolated worker.
- [ ] Confirm no secrets are present in `local.settings.json` before any commit/publish (`CopyToPublishDirectory: Never` is already set — keep it).

---

## Azure Rollout and Rollback Plan

This repo has **no CI/CD or IaC checked in**, so the following is the safe generic sequence — confirm with you which parts apply to your actual deployment pipeline before executing anything against Azure:

1. **Identify current deployment target** (Function App name, resource group, subscription, whether a staging slot exists). *I have not queried Azure — say the word and I'll enumerate the actual Function App and its current `FUNCTIONS_WORKER_RUNTIME`/stack settings before proposing exact values.*
2. Create (or reuse) a **non-production deployment slot**.
3. On that slot only, set `FUNCTIONS_WORKER_RUNTIME=dotnet-isolated` and update the .NET stack/version setting to match the chosen target (`net10`/`net8`).
4. Deploy the migrated isolated build to that slot — **not production**.
5. Smoke-test the `notes` HTTP endpoint, Postgres connectivity, notification HTTP call, and Application Insights telemetry on the slot.
6. Only after validation, **swap** the slot into production.
7. Re-verify production immediately post-swap (endpoint, logs, App Insights).
8. Update any external CI/CD pipeline and IaC (not present in this repo) so future deployments always pair the isolated build artifact with `dotnet-isolated`.

**Rollback:** keep the pre-migration production slot/artifact untouched until the swap is confirmed healthy; swap back if problems appear. Do not delete the original in-process deployment or its slot until the isolated version has run successfully in production for a validation period you're comfortable with.

**I will not make any Azure change (slot creation, app setting change, swap) without first stating the exact resource, setting, and value, and getting your explicit confirmation.**

---

## AZFD0013 Risk and Prevention

`AZFD0013` occurs when `FUNCTIONS_WORKER_RUNTIME` says `dotnet-isolated` but the deployed payload is still the in-process build (or vice versa) — a mismatch between the runtime setting and the artifact actually running.

**Prevention for this app:**
- Never change `FUNCTIONS_WORKER_RUNTIME` and deploy the new build as two separate, unordered steps against the same slot.
- Always change the runtime setting **and** deploy the isolated artifact **together**, on a slot, before swapping.
- Because this repo has no CI/CD pipeline checked in, there's an elevated real-world risk of someone flipping the app setting manually in the portal ahead of a deployment (or vice versa) — call this out explicitly to whoever owns the actual deployment process.

---

## Remaining Risks and Follow-up Actions (prioritized)

1. **High — retirement deadline**: in-process support ends 10 Nov 2026; this is a small, low-risk app, so there's no reason to delay starting the migration.
2. **High — net6.0 is itself out of support**: independent of the hosting-model migration, the app is running an EOL framework today.
3. **Medium — no CI/CD/IaC in repo**: I can't confirm today's actual deployment mechanism, slot topology, or app settings from this repo alone. Before rollout, identify and share (or let me query) the actual Function App resource.
4. **Low — HTTP result-type cleanup**: mixed `System.Web.Http`/`Microsoft.AspNetCore.Mvc` usage in `NotesFunction.cs` should be tidied to pure ASP.NET Core types as part of the migration, not left mixed.
5. **Low — App Insights sampling config**: confirm worker-side vs host-side sampling settings produce equivalent telemetry volume/fidelity post-migration.

**Next step:** confirm you want .NET 10 vs .NET 8 as the target, and let me know whether you'd like me to (a) proceed with the actual code changes (still no Azure changes without separate confirmation), or (b) first inventory the real Azure Function App/deployment pipeline for this app so the rollout plan above can be made concrete.

Created 7 todos
