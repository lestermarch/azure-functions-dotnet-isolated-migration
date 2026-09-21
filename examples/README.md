# Examples

This directory contains a GitHub Copilot Plan mode prompt and the output from an actual manual test run.

## Test repository

The prompt was tested against Microsoft's public [`Azure-Samples/azure-functions-code-testing-sample`](https://github.com/Azure-Samples/azure-functions-code-testing-sample) repository at commit [`632adadfc4f347c71aad277d988967f73c72f434`](https://github.com/Azure-Samples/azure-functions-code-testing-sample/commit/632adadfc4f347c71aad277d988967f73c72f434).

The sample contains a .NET 6 Azure Functions v4 app using the in-process model, including `Microsoft.NET.Sdk.Functions`, `[FunctionName]`, and `FunctionsStartup`.

## Files

- [`migration-analysis.prompt.md`](migration-analysis.prompt.md) — the assessment prompt used in GitHub Copilot Plan mode.
- [`migration-analysis-output.md`](migration-analysis-output.md) — the actual output captured from the run.

The output is intentionally retained as generated, including the activity-summary lines before and after the assessment. It represents a point-in-time example rather than authoritative current guidance. Re-run the prompt against your own repository and verify recommendations against current Microsoft Learn documentation.
