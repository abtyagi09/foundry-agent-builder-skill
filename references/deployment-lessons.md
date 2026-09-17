# Foundry Agent Builder deployment lessons

These notes came from deploying the Hudson Advisors agent through GitHub Actions on September 17, 2026.

## Prompt-based vs code-first hosted agents

Use a prompt-based Foundry agent for simple instruction/tool/RAG scenarios. Use Microsoft Agent Framework deployed as a Microsoft Foundry Hosted Agent for code-first, complex, custom-protocol, stateful, long-running, or controlled-compute scenarios.

Hosted Agents are containerized agentic applications. You package custom code as a container image, push it to Azure Container Registry, and Foundry Agent Service manages endpoint, identity, scaling, session state, isolation, lifecycle, and observability while your code handles orchestration.

Protocol guidance:

- Responses: conversational assistants, multi-turn Q&A, RAG/tools, background chat-style processing, Teams/Microsoft 365 publication.
- Invocations: webhooks, classification/extraction/batch work, protocol bridges, arbitrary JSON payloads.
- Invocations over WebSocket: real-time voice or bidirectional streaming.

Reference: https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents

## Grounding documents, Azure Storage, AI Search, and Foundry IQ

For any sample/demo corpus or approved customer RAG content, do not leave documents as local-only files in the repo.

Recommended grounding flow:

1. Create clearly labeled sample documents under `docs/` or `data/`.
2. Provision an Azure Storage account and Blob container for source documents.
3. Upload the sample documents to Blob Storage as the durable source of truth.
4. Create or update an Azure AI Search index from the blob-backed content.
5. Preserve metadata in the index: source file, page or section, chunk ID, document type, access tags, and last indexed timestamp.
6. Connect Azure AI Search to the Foundry project with managed identity/RBAC.
7. Expose the indexed content through Foundry IQ / knowledge bases where supported.
8. Attach the Foundry IQ / knowledge base or Azure AI Search tool to the prompt-based agent, or call the same retriever from Microsoft Agent Framework code for code-first hosted agents.
9. Require grounded answers, citations/source references, and explicit gap statements when the evidence is missing.

GitHub Actions should automate the sequence:

```text
deploy storage/search/foundry infra
upload sample docs to Blob Storage
create/update Azure AI Search index
create/update Foundry IQ or knowledge base
create/update agent
smoke test grounded answer with citation/source metadata
```

This keeps demo content repeatable, avoids local-file drift, and makes the same pattern usable for customer-approved knowledge later.

## Monitoring and evaluation

For every deployable Foundry agent, include both runtime monitoring and an evaluation gate.

Recommended monitoring:

1. Provision Log Analytics and Application Insights in Bicep.
2. Configure Azure Monitor OpenTelemetry in any Container App/API wrapper.
3. Log request start/completion, latency, status, and failures.
4. Use Foundry observability/tracing for agent runs, model calls, MCP/knowledge-base tool calls, latency, and errors.
5. For code-first Microsoft Agent Framework hosted agents, enable the framework's OpenTelemetry integration and tag spans with the agent name/environment.

Recommended evaluation:

1. Add an `evals/` folder with a small golden dataset.
2. Cover source-grounding, citation/source behavior, refusal/evidence gaps, formatting, and one or more business-critical workflows.
3. Run a live evaluation after deployment against `/ask`, the Foundry Responses API, or the selected hosted-agent protocol.
4. Fail the GitHub Actions run when pass rate is below a configurable threshold such as `EVAL_MIN_PASS_RATE`.
5. Upload JSON results as a workflow artifact.

The Hudson deployment used this pattern: Application Insights + Log Analytics for runtime telemetry, Foundry observability for hosted-agent and MCP tool traces, and a GitHub Actions live evaluation gate over `evals/golden.jsonl`.

## Current Foundry RBAC roles

Do not rely on legacy/misleading Azure AI roles for Foundry hosted-agent publish and endpoint access. Microsoft documentation states that roles such as Azure AI Developer and Cognitive Services roles do not apply to Foundry hosted-agent project work.

Use these role IDs in Bicep and scripts:

| Role | ID | Scope | Purpose |
| --- | --- | --- | --- |
| Foundry Project Manager | `eadc314b-1a2d-4efa-be10-5d325db5065e` | Foundry resource | Publish/create agents from GitHub Actions |
| Foundry User | `53ca6127-db72-4b80-b1b0-d745d6d5456d` | Foundry project | Build/develop in a project |
| Foundry Agent Consumer | `eed3b665-ab3a-47b6-8f48-c9382fb1dad6` | Project and agent | Invoke agent endpoints |
| ACR Pull | `7f951dda-4ed3-4680-a7ca-43fe172d538d` | ACR | Pull runtime containers |

For prompt-based agent creation with `project.agents.create_version`, assign Foundry Project Manager at the Foundry resource scope and Foundry User at the project scope to the GitHub OIDC service principal.

For runtime invocation, assign Foundry Agent Consumer at the project scope and, after the agent exists, at the agent scope:

```text
/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>/agents/<agentName>
```

If a Container App, API, or hosted-agent runtime uses a user-assigned managed identity to invoke an agent endpoint, grant that managed identity Foundry Agent Consumer on the project and agent scopes. If 403 persists immediately after assignment, restart the app revision/runtime and allow RBAC propagation time.

## RBAC propagation

Add explicit waits after deploying role assignments:

- 60-120 seconds before calling Foundry agent APIs.
- Another short wait after adding agent-scope runtime access if the next step immediately invokes the agent.

Common missing/propagating RBAC errors:

```text
Identity(object id: <id>) does not have permissions for Microsoft.CognitiveServices/accounts/AIServices/agents/write actions
```

Runtime endpoint failures often appear as API 500 with Container App logs showing:

```text
OpenAI/Responses API Error code: 403
```

## Stable Bicep outputs

Avoid uppercase/underscore Bicep outputs for values consumed by GitHub Actions. ARM can surface output names with unexpected mixed casing, such as:

```text
foundrY_PROJECT_ENDPOINT
azurE_SEARCH_ENDPOINT
```

Prefer camelCase output names:

```bicep
output foundryProjectEndpoint string = foundry.outputs.projectEndpoint
output azureSearchEndpoint string = search.outputs.searchEndpoint
output azureSearchIndex string = search.outputs.searchIndexName
output azureSearchConnectionId string = foundry.outputs.searchConnectionId
output acrName string = app.outputs.acrName
output containerAppName string = app.outputs.containerAppName
output appIdentityPrincipalId string = app.outputs.identityPrincipalId
```

Read them in GitHub Actions with:

```bash
get_output() {
  az deployment group show \
    --resource-group "$AZURE_RESOURCE_GROUP" \
    --name main \
    --query "properties.outputs.$1.value" \
    --output tsv
}

echo "FOUNDRY_PROJECT_ENDPOINT=$(get_output foundryProjectEndpoint)" >> "$GITHUB_ENV"
test -n "$(get_output foundryProjectEndpoint)"
```

If outputs are empty, SDKs can fail with misleading downstream errors such as:

```text
Bearer token authentication is not permitted for non-TLS protected (non-https) URLs
```

## GitHub OIDC subject mismatch

If `azure/login` fails with:

```text
AADSTS700213: No matching federated identity record found for presented assertion subject
```

copy the exact subject claim from the failed GitHub Actions log and add a federated credential for that exact subject.

Some environments emit a numeric subject format:

```text
repo:<owner>@<ownerNumericId>/<repo>@<repoNumericId>:environment:<env>
```

instead of:

```text
repo:<owner>/<repo>:environment:<env>
```

Keep the classic subject if useful, but add the emitted numeric subject as an additional federated credential when required.

## Duplicate role assignments

If a role assignment was manually created and Bicep later creates the same role/scope/principal with a different deterministic GUID, ARM can fail with:

```text
RoleAssignmentExists
```

Fix it by deleting the manual duplicate assignment or by aligning Bicep's `guid(...)` naming with the existing assignment. Prefer letting Bicep own all role assignments from the beginning.

## Container App / runtime permission pattern

For a separate Container App API that invokes a Foundry agent:

1. Use a user-assigned managed identity.
2. Pass `AZURE_CLIENT_ID`, `FOUNDRY_PROJECT_ENDPOINT`, and `AGENT_NAME`.
3. Grant ACR Pull on ACR.
4. Grant Foundry Agent Consumer at project scope.
5. After the agent exists, grant Foundry Agent Consumer at agent scope.
6. Rebuild/update or restart the app revision to refresh managed-identity tokens.

For code-first Microsoft Foundry Hosted Agents, use the same identity/RBAC mindset, but do not deploy a separate Container App unless the solution has a separate API or front end.

## GitHub Actions sequence that worked

1. Azure OIDC login.
2. `az bicep build` and `az deployment group validate`.
3. `az deployment group create` for infra.
4. Capture camelCase outputs and fail fast if required values are empty.
5. Install Python dependencies.
6. Wait for Foundry RBAC propagation.
7. Upload sample documents to Azure Storage, index them into Azure AI Search, and create/update Foundry IQ / knowledge bases when using RAG.
8. Create/update a prompt-based agent, or build/push/deploy a Microsoft Agent Framework hosted-agent container for code-first scenarios.
9. Grant runtime identity Foundry Agent Consumer at the agent scope.
10. Wait briefly again if needed.
11. Build/push a separate API image with ACR if a separate API/front end exists.
12. Update Container App or hosted-agent version.
13. Smoke test `/health`, `/ask`, Responses API, or the selected hosted-agent protocol endpoint.
