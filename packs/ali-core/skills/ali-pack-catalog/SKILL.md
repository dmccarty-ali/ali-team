---
name: ali-pack-catalog
description: |
  Catalog of available ali-team plugin packs. Use when developers ask what packs
  are available, what a pack contains, or how to install additional packs.
triggers:
  planning:
    - "what agent packs are available for my project?"
    - "what domain packs exist?"
    - "which packs should I install?"
  guidance:
    - "how do I add a pack to my project?"
    - "how do I install a plugin pack?"
  review:
    - "is my plugin-config.json complete?"
    - "what packs should this project have?"
  reference:
    - "what agents are in pack X?"
    - "what does ali-data contain?"
---

# Pack Catalog

## Available Packs

| Pack | Description | Install As |
|------|-------------|------------|
| ali-core | Core agents + all skills. Ships to every project automatically. | (automatic) |
| ali-data | Data platform agents: dbt, Kafka, Iceberg, Parquet, data architecture, etc. | ali-data |
| ali-healthcare | Healthcare data agents: FHIR/HL7, EDI, RCM. | ali-healthcare |
| ali-cloud | Cloud infrastructure agents: AWS, Terraform, Docker. | ali-cloud |
| ali-ai | AI/ML development agents: LangGraph, ML, Airflow, Streamlit. | ali-ai |
| ali-governance | Data governance agents: Collibra, MDM, SAP MDG, WhereScape. | ali-governance |
| ali-manufacturing | Manufacturing domain agents. | ali-manufacturing |
| ali-consulting | Consulting delivery agents: RFP, SOW, OCM. | ali-consulting |
| ali-azure | Azure platform agents: Fabric, Copilot, Graph API. | ali-azure |
| ali-office | Office document agents: PPTX, DOCX, XLSX. Prerequisites not yet created — see below. | ali-office |

## How to Install a Pack

Edit .claude-local/plugin-config.json:

```json
{ "packs": ["ali-core", "ali-data"] }
```

Then run: claude-ali

The launcher auto-installs any listed pack on next launch.

## Pack Contents

### ali-core

Core agents shipped to every project automatically. Includes orchestration, code development, documentation, testing, and infrastructure agents.

Agents:
- ali-plan-agent
- ali-plan-builder
- ali-plan-reviewer
- ali-expert-panel-expert
- ali-merge-synthesis-expert
- ali-technical-feasibility-expert
- ali-bash-developer
- ali-bash-expert
- ali-backend-developer
- ali-backend-expert
- ali-frontend-developer
- ali-frontend-expert
- ali-design-review-expert
- ali-document-developer
- ali-documentation-expert
- ali-doc-review-expert
- ali-doc-review-pass
- ali-doc-review-merge
- ali-test-developer
- ali-testing-expert
- ali-security-expert
- ali-github-admin
- ali-github-expert
- ali-macos-admin
- ali-claude-developer
- ali-claude-code-expert

### ali-data

Data platform agents for dbt, Kafka, Iceberg, Parquet, data architecture, and more.

Agents:
- ali-database-expert
- ali-dbt-expert
- ali-kafka-expert
- ali-iceberg-expert
- ali-parquet-expert
- ali-data-architecture-expert
- ali-data-science-expert
- ali-starburst-expert

### ali-healthcare

Healthcare data agents covering FHIR/HL7, EDI, and revenue cycle management.

Agents:
- ali-fhir-hl7-expert
- ali-edi-expert
- ali-rcm-expert

### ali-cloud

Cloud infrastructure agents for AWS, Terraform, and Docker.

Agents:
- ali-aws-admin
- ali-aws-expert
- ali-terraform-admin
- ali-terraform-expert
- ali-docker-admin
- ali-docker-expert

### ali-ai

AI and machine learning development agents for LangGraph, ML, Airflow, and Streamlit.

Agents:
- ali-langgraph-expert
- ali-machine-learning-expert
- ali-ai-expert
- ali-streamlit-expert
- ali-airflow-expert

### ali-governance

Data governance agents for Collibra, MDM strategy, SAP MDG, and WhereScape.

Agents:
- ali-collibra-expert
- ali-data-governance-expert
- ali-mdm-strategy-expert
- ali-sap-mdg-expert
- ali-wherescape-expert

### ali-manufacturing

Manufacturing domain agents.

Agents:
- ali-manufacturing-expert

### ali-consulting

Consulting delivery agents for RFP responses, SOW scoping, and organizational change management.

Agents:
- ali-rfp-response-expert
- ali-sow-scoping-expert
- ali-ocm-expert
- ali-sow-orchestrator

### ali-azure

Azure platform agents for Fabric, Copilot, and Graph API.

Agents:
- ali-azure-fabric-expert
- ali-copilot-expert
- ali-graph-api-expert

### ali-office

Office document agents for PPTX, DOCX, and XLSX.

**Note: ali-office is gated on prerequisite agents that have not yet been created.** The ali-office-developer and ali-office-expert agents must be created before this pack can be built and installed. Until then, listing ali-office in plugin-config.json will fail at install time.

Agents (pending):
- ali-office-developer (prerequisite — not yet created)
- ali-office-expert (prerequisite — not yet created)
