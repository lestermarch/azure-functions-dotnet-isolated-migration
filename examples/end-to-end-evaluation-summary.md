# End-to-end evaluation: Azure Functions in-process to isolated migration

This document captures the end-to-end migration evaluation run against the public sample repository:

- `https://github.com/Azure-Samples/azure-functions-code-testing-sample`
- Commit tested: `632adadfc4f347c71aad277d988967f73c72f434`

The goal was to validate the real-world migration path from the Azure Functions .NET in-process hosting model to the isolated worker model, using the Microsoft Learn guidance and the migration skill created for this repository.

## Scope

The evaluation included:

1. Assessing the in-process app to confirm the hosting model and migration risk.
2. Deploying the baseline in-process app to Azure in the target subscription and region.
3. Running the migration assessment using the skill and prompt.
4. Migrating the code to the isolated worker model.
5. Redeploying the migrated application to Azure.
6. Verifying the function still responded successfully.
7. Recording the operational changes needed to make the migration work in a policy-constrained Azure subscription.

## Baseline: what the sample app looked like

The sample repository was a .NET 6 Azure Functions v4 application using the in-process model. Key indicators included:

- `Microsoft.NET.Sdk.Functions`
- `Microsoft.Azure.Functions.Extensions`
- `FunctionsStartup`
- `[FunctionName]`
- `FUNCTIONS_WORKER_RUNTIME=dotnet`
- HTTP-triggered function app with PostgreSQL-backed business logic

It was a good migration candidate because it was relatively small, had no Durable Functions or native dependencies, and used standard NuGet packages that are compatible with modern .NET.

## Azure environment constraints

The real Azure validation surfaced platform constraints that were not visible from a repository-only review:

- Azure Policy disabled storage shared-key authentication.
- Azure Policy disabled public access to the storage account.
- Windows Consumption was unsuitable because its Azure Files content storage relies on shared-key access.
- The baseline therefore used Linux Consumption with identity-based `AzureWebJobsStorage` and Blob, Queue, and Table data-plane role assignments.
- Conventional ZIP deployment was unavailable in this policy combination. Because the sample and package contained no secrets and were already public, the evaluation temporarily used a versioned GitHub release asset as the remote package source. This was an evaluation workaround, not a production recommendation.
- .NET 10 is not supported on Linux Consumption. Microsoft documents .NET 9 as the final .NET version added to that plan, so the isolated deployment was moved to a Linux B1 Dedicated plan. For a new serverless production deployment, Flex Consumption would normally be preferred.

These constraints materially changed the rollout architecture. They demonstrate why a migration assessment must include Azure Policy, storage networking, deployment mechanics, operating system, hosting plan, and runtime-version compatibility—not only source code.

## Migration process

The migration used the Microsoft Learn guidance and the skill created for this scenario.

### Assessment outcome

The app was judged to be low-to-medium risk for migration because:

- The app was a single HTTP-triggered Function.
- There were no Durable Functions, service bus triggers, or custom native dependencies.
- The logic was isolated enough to test directly.
- Dependency packages were compatible with modern .NET.

The recommendation was to migrate to the isolated worker model using a modern supported target, with `.NET 10` as the preferred target and `.NET 8` as a perfectly acceptable conservative alternative if tooling maturity or platform support was a concern.

### Code changes

The migration applied the current .NET 10 isolated-worker pattern:

- Changed the project SDK to `Azure.Functions.Sdk/1.0.0` and the target framework to `net10.0`.
- Removed `Microsoft.NET.Sdk.Functions` and `Microsoft.Azure.Functions.Extensions`.
- Added the isolated worker, ASP.NET Core HTTP integration, and worker-side Application Insights packages.
- Added `Program.cs` with `ConfigureFunctionsWebApplication()` and moved dependency registrations from `FunctionsStartup`.
- Replaced `FunctionName` with `Function` and removed the obsolete startup class.
- Replaced Newtonsoft.Json attributes with `System.Text.Json` equivalents.
- Changed the HTTP return contract from `ActionResult<T>` to `IActionResult`.
- Updated the unit and integration-test projects to `net10.0` and refreshed vulnerable integration-test dependencies.
- Upgraded Npgsql from the advisory-affected 6.0.10 release to a supported modern version.

Azure validation exposed a behavior that the successful local build did not: the original POCO trigger parameter reached the function as `null`, resulting in HTTP 500 responses. Adding `[FromBody]` was not sufficient in this app. The reliable fix was to bind `HttpRequest`, call `ReadFromJsonAsync<CreateNoteRequest>()`, and delegate to a separate processing method that remained easy to unit test.

This is exactly why the skill should require trigger-level behavior tests rather than treating a clean compilation as proof of migration success.

## Azure validation

The end-to-end validation included:

1. Creating the Azure Function App infrastructure in the target subscription.
2. Deploying the original in-process app.
3. Verifying the deployed app responded successfully to the expected HTTP POST.
4. Migrating the codebase to isolated worker.
5. Redeploying the isolated version.
6. Validating the result with an HTTP request and platform telemetry.

The final functional validation confirmed:

| Check | Baseline in-process | Migrated isolated |
|---|---|---|
| Target/runtime | .NET 6, `dotnet` | .NET 10, `dotnet-isolated` |
| Azure stack | `DOTNET|6.0` | `DOTNET-ISOLATED|10.0` |
| Hosting plan | Linux Consumption | Linux B1 Dedicated |
| Build | Passed | Passed |
| Unit tests | Baseline build validated | 3/3 passed |
| HTTP POST `/api/notes` | HTTP 201 | HTTP 201 |
| Persistence path | PostgreSQL-backed create succeeded | PostgreSQL-backed create succeeded |
| Telemetry | Request and host traces present | Successful HTTP 201 request in Application Insights |

The isolated app used a versioned remote package URL so the runtime, stack, and payload changed together and the platform could not continue using a cached earlier package.

The run established that the application migration was technically viable. It also left one infrastructure finding that would need resolution before production sign-off: host telemetry continued to report `Unable to access AzureWebJobsStorage` after the move to Dedicated hosting. The HTTP-only function still indexed and completed successfully, but a production rollout should fix storage network reachability and obtain a clean host-health result before cutover.

## Operational lessons learned

The most important outcomes from the evaluation were not purely code-related.

### 1. Deployments must be coordinated with runtime settings

The migration skill had to emphasize `AZFD0013` prevention as a real operational risk. The worker runtime, platform stack, and deployed payload must change as one coordinated release. Changing only part of that set can leave the app in a mismatch state.

### 2. Policy and plan constraints can dominate the migration

The subscription environment did not allow default storage or deployment assumptions. Policy required managed identity, and the .NET 10 target required leaving Linux Consumption. A useful migration plan must therefore validate policy, network access, hosting-plan support, deployment method, and runtime together.

### 3. HTTP request handling differs between models

The app’s initial isolated version failed in Azure because request-body binding and null handling differed from the in-process model. This was a practical code-level change, not just a packaging update.

### 4. Validation must include the full deployment artifact

A successful local build and unit test run are necessary but not sufficient. The Azure deployment also had to be validated for:

- runtime setting
- platform stack
- storage connectivity
- trigger indexing
- package contents
- live HTTP behavior
- App Insights telemetry

### 5. Dependency review should happen during migration

The sample app’s dependency set included an outdated Npgsql package with an advisory warning. Updating the dependency as part of migration improved the overall security and support posture and was a useful addition to the skill.

## Skill review

The migration skill was already strong on the source-code changes described by Microsoft Learn. The evaluation showed that success at build time did not cover several deployment and runtime concerns.

The skill did not need a large rewrite, but it did need targeted additions that are now reflected in the guidance:

- subscription-level policy and storage-network constraints
- explicit hosting-plan compatibility checks for the selected .NET version
- managed-identity storage configuration and host-health validation
- coordinated runtime, platform stack, and versioned artifact deployment to prevent `AZFD0013` and stale package reuse
- explicit request-body parsing when ASP.NET Core integration does not preserve the original POCO binding behavior
- validating the full deployed artifact, not just the source code
- dependency hygiene and vulnerability review as part of migration readiness

This makes the skill more realistic for operators who are migrating apps in Azure environments with non-default platform policies.

## Outcome

The functional end-to-end evaluation was successful:

- The original in-process Function App was deployed and returned HTTP 201.
- The code was migrated to .NET 10 isolated and built successfully.
- All three unit tests passed.
- The migrated app was redeployed on a hosting plan that supports .NET 10.
- The live endpoint returned HTTP 201 with the expected response body.
- Application Insights recorded the successful request.
- The migration skill was refined using the issues found during the run.

This should be described as a successful proof of concept, not an unconditional production readiness result. Host storage health still required remediation, Docker was unavailable for the full Testcontainers integration suite, and the evaluation used a temporary public remote-package mechanism that should be replaced by an approved private deployment path.

## Evaluation limitations

- The Azure environment was an ephemeral non-production evaluation environment; no production slot swap was performed.
- Docker Desktop was unavailable, so the Testcontainers integration suite was updated and compiled but not executed locally.
- The HTTP create operation provided functional evidence of PostgreSQL access, but this was not a database performance or resilience test.
- The source sample used anonymous HTTP authorization. A production implementation should use an appropriate authentication and authorization model.
- The temporary GitHub release package contained public sample code and no secrets. Production packages should use an approved private, immutable deployment source.
- The Dedicated plan and PostgreSQL resources incur cost until removed.

## References

- [Migrate C# apps from in-process to isolated worker](https://learn.microsoft.com/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net10)
- [Supported languages and .NET versions in Azure Functions](https://learn.microsoft.com/azure/azure-functions/supported-languages)
- [Migrate Linux Consumption apps to Flex Consumption](https://learn.microsoft.com/azure/azure-functions/migration/migrate-plan-consumption-to-flex)
- [Azure Functions isolated worker guide](https://learn.microsoft.com/azure/azure-functions/dotnet-isolated-process-guide)

## Recommended next step

For production work, turn the proof of concept into a repeatable migration checklist for the owning team:

- confirm the target framework, operating system, and hosting-plan support matrix,
- inventory applicable Azure Policy assignments before designing deployment storage,
- validate storage RBAC, network reachability, and host health,
- deploy an immutable package to a non-production slot or separate app,
- change worker runtime, platform stack, and payload as one coordinated release,
- test every trigger, dependency, failure path, and telemetry flow end to end,
- swap only after validation,
- keep the prior settings and deployment artifact available for rollback.

This keeps the migration aligned with the retirement deadline while minimizing the risk of runtime mismatch or deployment failure.
