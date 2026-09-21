# Azure Functions .NET isolated migration skill

A GitHub Copilot agent skill for assessing, planning, implementing, validating, and safely deploying migrations from the Azure Functions .NET in-process hosting model to the isolated worker model.

Support for the in-process model ends on **10 November 2026**. The skill follows the current [Microsoft Learn migration guidance](https://learn.microsoft.com/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net10) and emphasizes compatibility assessment, reversible changes, staging-slot rollout, and prevention of runtime/payload mismatches such as `AZFD0013`.

## Repository contents

```text
azure-functions-dotnet-isolated-migration/
├── SKILL.md
└── evals/
    └── evals.json
examples/
├── README.md
├── migration-analysis.prompt.md
├── migration-analysis-output.md
└── end-to-end-evaluation-summary.md
```

- [`azure-functions-dotnet-isolated-migration/SKILL.md`](azure-functions-dotnet-isolated-migration/SKILL.md) contains the agent skill.
- [`examples/migration-analysis.prompt.md`](examples/migration-analysis.prompt.md) is a reusable GitHub Copilot Plan mode assessment prompt.
- [`examples/migration-analysis-output.md`](examples/migration-analysis-output.md) is an actual assessment produced from the prompt and skill.
- [`examples/end-to-end-evaluation-summary.md`](examples/end-to-end-evaluation-summary.md) documents the full Azure deployment, migration, and validation flow that was run against a public sample app.
- [`azure-functions-dotnet-isolated-migration/evals/evals.json`](azure-functions-dotnet-isolated-migration/evals/evals.json) contains representative evaluation scenarios.

## Use the skill

Copy the complete `azure-functions-dotnet-isolated-migration` folder into a skills directory supported by your GitHub Copilot environment. Keep `SKILL.md` at the root of that folder.

Then open an in-process Azure Functions repository and either describe the migration task normally or run the example prompt in GitHub Copilot Plan mode. The skill description is designed to trigger for in-process/isolated-worker migration requests even when the skill is not named explicitly.

## Safety

The skill separates repository migration from Azure rollout. It does not assume that production settings can be changed safely. Review the generated plan, validate locally, and use a non-production slot before changing `FUNCTIONS_WORKER_RUNTIME` or deploying an isolated-worker payload.

## Disclaimer

The example output is preserved from a real test run for transparency. It is not a substitute for a fresh assessment, and package versions, supported target frameworks, and Azure Functions guidance can change. Always consult current Microsoft documentation before making production changes.

## License

Licensed under the [MIT License](LICENSE).
