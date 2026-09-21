# Azure Functions in-process to isolated-worker assessment

Assess this Azure Functions C# app for migration from the in-process hosting model to the isolated worker model, using the current Microsoft Learn guidance:
https://learn.microsoft.com/en-gb/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net10

## Goal

- Determine whether the app is currently using the .NET in-process model and whether it must be migrated before the retirement date of 10 November 2026.
- Recommend the safest target model and framework.
- Produce a repository-aware migration assessment and rollout plan without editing files.

## Instructions

1. Inspect the repository first and identify:
   - Target framework and Functions runtime version.
   - In-process indicators: `Microsoft.NET.Sdk.Functions`, `Microsoft.Azure.WebJobs.*`, `Microsoft.Azure.Functions.Extensions`, `FunctionName`, `FunctionsStartup`, and `FUNCTIONS_WORKER_RUNTIME=dotnet`.
   - Trigger and binding inventory, including HTTP, Service Bus, Event Hubs, Cosmos DB, Storage, Durable Functions, timers, and custom extensions.
   - Deployment model, slots, CI/CD, infrastructure as code, app settings, and Application Insights configuration.
   - Any .NET Framework-only or native dependencies that may block a modern .NET isolated migration.

2. Determine the recommended target:
   - Prefer .NET 10 isolated when the app and all dependencies are compatible with modern .NET.
   - Use .NET Framework 4.8 isolated only if a dependency or API truly requires .NET Framework.
   - Explain the trade-offs and any unresolved compatibility gate.

3. Produce a migration assessment covering:
   - Current state of the app and runtime.
   - Recommended migration target and rationale.
   - Blockers or unsupported dependencies.
   - Files and project packages likely to change.
   - Function code areas to update, including `FunctionName`, `Program.cs`, startup/DI migration, logging, HTTP, JSON, Durable Functions, and binding namespace/package changes.
   - Local validation plan.
   - Azure rollout plan using a staging slot and coordinated worker-runtime change.
   - Rollback plan.
   - `AZFD0013` risk and prevention strategy.

4. Be specific about operational risk:
   - Explain the difference between the in-process and isolated worker models.
   - Explain the runtime/payload mismatch risk when `FUNCTIONS_WORKER_RUNTIME` and the deployed artifact do not match.
   - Recommend using a staging slot or parallel app before production cutover.

5. Do not edit the codebase. This is an assessment only.
6. Before any production or destructive Azure change, explicitly state what would change and require confirmation.
7. Keep the output concise but actionable, including a prioritized list of risks and next steps.

## Return format

- Executive summary
- Current state assessment
- Recommended target and decision rationale
- Migration workstreams
- Validation checklist
- Azure rollout and rollback plan
- Remaining risks and follow-up actions
