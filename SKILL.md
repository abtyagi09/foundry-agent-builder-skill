---
name: "foundry-agent-builder"
description: "Build Azure AI Foundry / Microsoft Foundry agents end to end with GitHub, Bicep, GitHub Actions, managed identity, and repeatable deployment. Use when asked to create, scaffold, deploy, troubleshoot, or explain a Foundry prompt-based agent or code-first hosted agent."
---

# Foundry Agent Builder

Build or deploy a Microsoft Foundry agent with production-ready defaults: GitHub source control, Bicep infrastructure, GitHub Actions with Azure OIDC, managed identity, RBAC, observability, and smoke tests.

## When to use this

- "Build a Foundry agent for this customer/use case"
- "Scaffold an Azure AI Foundry hosted agent"
- "Deploy this agent with GitHub Actions"
- "Create the Bicep for a Foundry agent"
- "Troubleshoot my Foundry agent deployment"
- "Should this be a prompt agent or a hosted agent?"

## Architecture decision rule

Use a **prompt-based Foundry agent** for simple scenarios where behavior is mostly instructions, model selection, managed tools, file search/RAG, Azure AI Search grounding, and straightforward tool configuration. Use `PromptAgentDefinition`, Foundry agent versioning, or the portal-backed prompt-agent pattern. Do not introduce a custom container or code-first runtime unless the use case needs it.

Use **Microsoft Agent Framework deployed as a Microsoft Foundry Hosted Agent** for custom or complex scenarios. Refer to the Microsoft Hosted Agents documentation before finalizing the design: https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents

Choose Hosted Agents when the use case needs bring-your-own code, Microsoft Agent Framework orchestration, LangGraph/Semantic Kernel/custom framework support, custom business logic, complex tool loops, custom protocols, webhooks, non-OpenAI payloads, controlled CPU/memory, stateful workloads, persistent files/state, long-lived resilient work, or replayable streaming.

Default to Microsoft Agent Framework for new code-first hosted-agent implementations unless the repo already uses another framework or the user requests one.

## Hosted-agent protocol guidance

- **Responses** — conversational assistants, multi-turn Q&A, RAG/tools, background chat-style processing, and agents published to Teams or Microsoft 365.
- **Invocations** — webhooks, classification/extraction/batch jobs, protocol bridges, or arbitrary JSON payloads.
- **Invocations over WebSocket** — real-time voice or bidirectional streaming.

If unsure, start with Responses; a hosted agent can add other protocols later.

## Knowledge grounding pattern

For sample/demo corpora and customer-approved RAG content:

1. Create clearly labeled sample documents under `docs/` or `data/`.
2. Upload those documents to an Azure Storage account / Blob container as the durable source.
3. Index the uploaded documents into Azure AI Search with source metadata, page/section references, and chunk IDs.
4. Connect Azure AI Search to the Foundry project.
5. Expose indexed content through **Foundry IQ / knowledge bases** where supported, and use it as the agent's grounding layer.
6. Attach the Foundry IQ / knowledge base or Azure AI Search tool to prompt-based agents; for code-first Microsoft Agent Framework hosted agents, call the retriever from the agent code.
7. Require citations and explicit refusal/gap statements when evidence is missing.

GitHub Actions should include this RAG sequence when grounding is part of the solution: upload sample docs -> build/update AI Search index -> create/update Foundry IQ or knowledge base -> create/update agent -> smoke test grounded answers.

Important Bicep limitation: Azure Storage accounts, Blob containers, Azure AI Search services, Foundry accounts/projects, Foundry project connections, identities, and RBAC can be created with Bicep. Foundry IQ knowledge sources and knowledge bases are not currently exposed as first-class Bicep/ARM resource types. Create or update them after Bicep using REST/SDK scripts, or wrap those REST calls in a Bicep `deploymentScripts` resource if the user requires a single Bicep-driven deployment. Treat that as Bicep-orchestrated scripting, not native declarative Bicep support.

## Monitoring and evaluation

Always include monitoring and evaluation for deployable agents:

1. Provision Log Analytics and Application Insights with Bicep.
2. Enable Foundry observability/tracing for agent runs, tool calls, latency, and errors.
3. For Container App or API wrappers, configure Azure Monitor OpenTelemetry and log request start/completion, latency, response status, and failures.
4. Add an `evals/` folder with a small golden dataset covering grounding, refusal/evidence gaps, formatting, and at least one core business workflow.
5. Add a GitHub Actions evaluation gate after deployment that calls the live endpoint or Foundry Responses API and fails below a configurable pass rate.
6. Upload evaluation results as a workflow artifact and document where to inspect traces in Foundry and telemetry in Application Insights.

## Default assumptions

- Cloud: Azure
- IaC: Bicep
- Source control and CI/CD: GitHub + GitHub Actions
- GitHub-to-Azure auth: OpenID Connect federation, never stored client secrets
- Runtime language: Python unless the repo is clearly TypeScript/JavaScript
- App hosting: Azure Container Apps for separate app/API surfaces; Microsoft Foundry Hosted Agents for code-first agent runtimes
- Secrets: Key Vault and managed identity where possible
- Observability: Application Insights + Log Analytics and Foundry hosted-agent observability
- Environment: dev first, with prod separated by parameters and GitHub environments

## Workflow

1. **Clarify only blockers** — target subscription/resource group if not inferable, preferred language when code must be generated, whether to create files locally or provide guidance, and prompt-based vs code-first only if ambiguous.
2. **Inspect the repo first** — detect language/runtime, existing infra, GitHub workflows, app hosting, and any Agent Framework/LangGraph/Semantic Kernel/FastAPI patterns.
3. **Choose architecture** — prompt-based for simple instruction/tool/RAG agents; Microsoft Agent Framework Hosted Agent for code-first or complex scenarios.
4. **Scaffold precisely** — create modular infra, agent code, sample documents, storage/search/Foundry IQ grounding, tests/evals, GitHub Actions, and a README with exact deployment commands.
5. **Deploy with OIDC** — configure GitHub environment variables, Azure federated credential, least-privilege Foundry RBAC, and no client secrets.
6. **Validate** — run local syntax checks, Bicep build/validate, tests/evals, then live smoke tests.

## Prompt-based repo shape

```text
infra/main.bicep
infra/modules/*.bicep
infra/parameters/dev.bicepparam
src/agent/                 # prompt-agent setup/update/invoke logic
src/api/                   # optional API wrapper
docs/ or data/             # sample grounding content uploaded to Azure Storage
tests/
.github/workflows/deploy-dev.yml
README.md
```

## Code-first hosted-agent repo shape

```text
infra/main.bicep
infra/modules/*.bicep
infra/parameters/dev.bicepparam
src/agent/                 # Microsoft Agent Framework code + protocol handlers
Dockerfile
.dockerignore
tests/
evals/
.github/workflows/deploy-dev.yml
README.md
```

Use `src/api/` only for a separate API/front end. Do not confuse an app-hosted API with the Foundry hosted-agent container.

## RBAC requirements

Use current Microsoft Foundry roles, not legacy/misleading Azure AI roles:

- **Foundry Project Manager** (`eadc314b-1a2d-4efa-be10-5d325db5065e`) at the Foundry resource scope for publishing/creating agents from GitHub Actions.
- **Foundry User** (`53ca6127-db72-4b80-b1b0-d745d6d5456d`) at the Foundry project scope for the GitHub OIDC service principal.
- **Foundry Agent Consumer** (`eed3b665-ab3a-47b6-8f48-c9382fb1dad6`) at the project scope and, after the agent exists, at the agent scope for runtime invocation.
- **ACR Pull** for any runtime identity pulling a container.

Add a 60-120 second wait after deploying Foundry role assignments before calling Foundry agent APIs.

## GitHub Actions pattern

1. `azure/login` using OIDC.
2. `az bicep build` and `az deployment group validate`.
3. `az deployment group create`.
4. Capture camelCase deployment outputs and fail fast if any required output is empty.
5. Install Python dependencies.
6. Wait for Foundry RBAC propagation.
7. Upload sample documents to Azure Storage, index them into Azure AI Search, and create/update Foundry IQ / knowledge bases when using RAG.
8. Create/update the prompt-based agent, or build/push/deploy the Microsoft Agent Framework hosted-agent container.
9. Grant runtime identity Foundry Agent Consumer at the agent scope.
10. Build/push a separate API/front-end image only if one exists.
11. Run live evaluations and upload the results artifact.
12. Smoke test `/health`, `/ask`, the Responses API, or the selected hosted-agent protocol endpoint.

## Common failure fixes

Read [references/deployment-lessons.md](references/deployment-lessons.md) for details.

- `AADSTS700213` during `azure/login`: copy the exact OIDC subject claim from the Actions log and add it as a federated credential. Some environments emit `repo:<owner>@<ownerNumericId>/<repo>@<repoNumericId>:environment:<env>` instead of the classic owner/repo subject.
- `Bearer token authentication is not permitted for non-TLS protected URLs`: deployment outputs were probably empty or mis-cased; use camelCase Bicep outputs and `test -n`.
- `agents/write` permission error: check Foundry Project Manager and Foundry User assignments for the GitHub service principal.
- `/ask` or hosted-agent endpoint returns 500/403: check runtime managed identity and Foundry Agent Consumer assignments at project and agent scopes.
- `RoleAssignmentExists`: remove manually-created duplicate assignments or let Bicep own all role assignments from the start.

## Output

Lead with the concrete result: files created, deployed URL, successful GitHub Actions run, or exact blocker. Include exact commands only when the user must run them.
