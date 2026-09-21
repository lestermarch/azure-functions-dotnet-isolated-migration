---
name: azure-functions-dotnet-isolated-migration
description: Migrate Azure Functions C# apps from the in-process hosting model to the isolated worker model using Microsoft's current migration guidance. Use this skill whenever a user mentions Azure Functions in-process, isolated worker, dotnet-isolated, Functions retirement on November 10, 2026, FunctionsStartup, Microsoft.NET.Sdk.Functions, or asks to assess, plan, implement, test, or deploy this migration. It handles repository assessment, target framework selection, project/code/configuration changes, local validation, slot-based Azure rollout, and rollback-aware troubleshooting.
---

# Azure Functions: in-process to isolated worker migration

Use the official Microsoft Learn procedure as the source of truth:
https://learn.microsoft.com/en-gb/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net10

Support for the in-process model ends on **November 10, 2026**. Treat migration as a compatibility and supportability project, not as an automatic platform conversion. Preserve the existing app until the isolated version is validated.

## Operating principles

- Inspect before editing. Read repository instructions and the relevant project files first.
- Establish the current model from the project and app settings; don't infer it from naming alone.
- Prefer the latest stable worker and extension packages compatible with the chosen target. Do not blindly copy version numbers from examples in the Learn article.
- Keep changes reversible. Do not delete the original app, production slot, or deployment path without explicit approval.
- Separate code migration from Azure rollout. Validate locally before changing `FUNCTIONS_WORKER_RUNTIME` in production.
- Explain assumptions, unsupported dependencies, and any required manual decisions.

## Workflow

### 1. Assess the app

Identify:

- Target framework and Functions runtime version.
- In-process indicators: `Microsoft.NET.Sdk.Functions`, `Microsoft.Azure.WebJobs.*`, `Microsoft.Azure.Functions.Extensions`, `FunctionName`, `FunctionsStartup`, and `FUNCTIONS_WORKER_RUNTIME=dotnet`.
- Triggers and bindings, including Durable Functions, Service Bus, Event Hubs, Cosmos DB, storage, HTTP, and custom extensions.
- Deployment method, slots, CI/CD, infrastructure-as-code, stack settings, and Application Insights configuration.
- Dependencies that require .NET Framework or APIs unavailable on modern .NET.

If Azure inventory is requested, apps whose runtime is `dotnet` are candidates for migration; `dotnet-isolated` apps are already migrated. Use the subscription context explicitly when inventorying multiple subscriptions.

Produce an assessment with: current state, target recommendation, blockers, files to change, test plan, rollout plan, and rollback plan.

### 2. Choose the target

- Use **.NET 10 isolated** when the app and all dependencies are proven compatible with modern .NET. This is the preferred strategic target in the current guidance.
- For an existing **.NET Framework 4.8** app with unknown or framework-only dependencies, use **.NET Framework 4.8 isolated** as the lowest-risk retirement migration. Treat a later move to .NET 10 as a separate modernization unless compatibility is already proven.
- Audit direct and transitive packages, native/COM dependencies, Windows-only APIs, hosting plan, and OS before choosing. Do not assume that the current target alone proves a dependency requirement or that all dependencies can move to modern .NET.
- If the app is C# script (`.csx`), convert it to the project model before applying the project migration.
- If the app is Durable Functions, use the Durable migration guidance as well as this workflow.

State the recommendation, evidence, trade-offs, and any unresolved compatibility gate. Confirm the target before broad edits when more than one target is viable.

### 3. Migrate the project

For modern .NET isolated apps, update the project to use the Azure Functions SDK and worker packages:

- Set the project SDK to `Azure.Functions.Sdk`.
- Set the target framework to the selected version (`net10.0` by default, or `net48` for the Framework path).
- Replace `Microsoft.NET.Sdk.Functions` with `Microsoft.Azure.Functions.Worker` and the appropriate worker extension packages.
- Add Application Insights worker-service integration when the app uses Application Insights.
- Remove all remaining `Microsoft.Azure.WebJobs.*` and `Microsoft.Azure.Functions.Extensions` package references.
- Replace each binding package with its `Microsoft.Azure.Functions.Worker.Extensions.*` equivalent. Consult the binding's current documentation because extension versions can require `host.json` changes.
- Preserve `host.json` and `local.settings.json` copy behavior; never publish secrets from `local.settings.json`.

Common replacements include:

| Existing | Isolated worker direction |
|---|---|
| `Microsoft.Azure.WebJobs.Extensions.Storage` | Blob, Queues, and Tables worker extensions as needed |
| `Microsoft.Azure.WebJobs.Extensions.CosmosDB` / `DocumentDB` | `Microsoft.Azure.Functions.Worker.Extensions.CosmosDB` |
| `Microsoft.Azure.WebJobs.Extensions.ServiceBus` | `Microsoft.Azure.Functions.Worker.Extensions.ServiceBus` |
| `Microsoft.Azure.WebJobs.Extensions.EventHubs` | `Microsoft.Azure.Functions.Worker.Extensions.EventHubs` |
| `Microsoft.Azure.WebJobs.Extensions.DurableTask` | `Microsoft.Azure.Functions.Worker.Extensions.DurableTask` |

### 4. Add the isolated host

Add `Program.cs` and move startup/DI configuration from `FunctionsStartup` into the host builder's `ConfigureServices` block.

For HTTP-triggered modern .NET apps, use `ConfigureFunctionsWebApplication()` and the ASP.NET Core HTTP extension. For non-HTTP apps, `ConfigureFunctionsWorkerDefaults()` is acceptable. Configure Application Insights explicitly when used.

Remove the `FunctionsStartup` attribute and obsolete startup class only after its registrations and configuration have been moved.

### 5. Update function code

Review every function, not just the project file:

- Replace `FunctionName` with `Function`.
- Update `ILogger` acquisition to the isolated worker pattern, normally constructor injection or `FunctionContext`/`ILogger<T>`.
- Update trigger and binding attributes, parameter types, return types, and binding data access to the isolated equivalents.
- Review HTTP functions for ASP.NET Core integration and request/response types.
- Review JSON serialization: isolated worker defaults and serializer configuration can differ.
- Move application log filtering to `Program.cs`; `host.json` primarily controls Functions host logs in isolated mode.
- Review middleware, dependency injection, configuration sources, and any code that depended on host-process behavior.
- For asynchronous HTTP streaming, follow the current isolated-worker requirements rather than copying in-process code.

After each logical batch, build to expose the next set of migration errors. Do not suppress warnings or leave both in-process and isolated attributes/packages in place.

### 6. Validate locally

Use Azure Functions Core Tools v4 and the target .NET SDK. At minimum:

1. Restore packages.
2. Build the project in the target configuration.
3. Run the Functions host locally.
4. Exercise every trigger and binding, including failure/retry paths.
5. Verify Application Insights/log filtering, configuration, serialization, and authentication behavior.
6. Run the repository's tests and deployment/package validation.

Check that generated output contains the worker host and no stale in-process package references. Record any behavior changes and unresolved warnings.

### 7. Roll out safely in Azure

Before publishing, plan these changes together:

1. Set `FUNCTIONS_WORKER_RUNTIME` to `dotnet-isolated`.
2. Deploy the migrated isolated project.

Changing only one creates an interim runtime/payload mismatch (`AZFD0013`). Prefer a staging slot:

1. Create or use a non-production slot.
2. Set the slot's runtime setting to `dotnet-isolated` and update the stack version if needed.
3. Deploy the migrated project to that slot.
4. Test triggers, bindings, health, logs, and dependencies.
5. Swap the validated slot into production.
6. Recheck production and monitoring.

Update CI/CD and IaC so future deployments keep the correct worker runtime and target framework. Keep the original deployment artifact and rollback path until production validation is complete.

### 8. Report completion

Summarize:

- Target framework and worker model.
- Files and packages changed.
- Local and Azure validation performed.
- Any behavior or configuration differences.
- Deployment/slot and rollback status.
- Remaining risks, unsupported dependencies, or follow-up actions.

## Troubleshooting priorities

- `AZFD0013`: runtime setting and deployed payload are temporarily mismatched; complete both rollout operations.
- Startup failures: check target framework, worker SDK, extension versions, `Program.cs`, and stale WebJobs packages.
- Missing triggers/bindings: verify the worker extension package and corresponding `host.json` schema.
- Missing or noisy telemetry: move application log filtering and Application Insights configuration into `Program.cs`.
- DI/configuration failures: migrate `FunctionsStartup` registrations and ensure platform trigger settings are in app settings or supported references.

## Deliverable format

For assessment-only requests, return a prioritized migration plan without editing files. For implementation requests, make surgical changes, show a concise change summary, and run the smallest meaningful validation commands. For Azure changes, state the exact settings and deployment sequence before execution and ask for confirmation before production or destructive actions.

## Related official guidance

- Migration guide: https://learn.microsoft.com/en-gb/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net10
- Model differences: https://learn.microsoft.com/en-gb/azure/azure-functions/dotnet-isolated-in-process-differences
- Isolated worker guide: https://learn.microsoft.com/en-gb/azure/azure-functions/dotnet-isolated-process-guide
- Retirement notice: https://aka.ms/azure-functions-retirements/in-process-model
- Durable migration: https://learn.microsoft.com/en-gb/azure/durable-task/durable-functions-migrate
