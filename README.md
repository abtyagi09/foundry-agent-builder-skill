# Foundry Agent Builder Skill

Standalone Microsoft Scout skill for building, deploying, and troubleshooting Microsoft Foundry agents with GitHub, Bicep, GitHub Actions, managed identity, RBAC, Azure Storage, Azure AI Search, and Foundry IQ / knowledge bases.

## Install in Scout

Clone this repo and copy it into your Scout skills folder:

```powershell
git clone https://github.com/abtyagi09/foundry-agent-builder-skill.git
Copy-Item .\foundry-agent-builder-skill "$env:USERPROFILE\.scout\m-skills\foundry-agent-builder" -Recurse -Force
```

Restart Scout, then invoke:

```text
/foundry-agent-builder
```

## Contents

```text
SKILL.md
references/deployment-lessons.md
```

`SKILL.md` is intentionally kept under Scout's 20 KB skill-loading limit. Longer deployment notes live in `references/`.

## What it covers

- Prompt-based Foundry agents for simple instruction/tool/RAG scenarios.
- Microsoft Agent Framework deployed as Foundry Hosted Agents for code-first and complex scenarios.
- Azure Storage -> Azure AI Search -> Foundry IQ / knowledge base grounding.
- GitHub Actions deployment with Azure OIDC.
- Current Microsoft Foundry RBAC roles and troubleshooting notes.

