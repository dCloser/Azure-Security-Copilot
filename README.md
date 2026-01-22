# Azure-Security-Copilot
This is a repository showing the architecture for an Agentic SOC. Will continue to improve this and add more contexts, scenarios for implementation

## Overview

This architecture describes an agent-assisted SOC workflow that can:

- **Query Microsoft Sentinel / Log Analytics for operational reports (e.g., alert/incident counts)**

- **Summarize and enrich findings using an Azure AI model**

- **Trigger automated actions via Azure Logic Apps (emailing, ticket creation, containment runbooks)**

- **Support context-aware threat hunting by generating KQL investigation queries and packaging results for analysts**

The intent is to combine the strengths of:

  - **Sentinel (telemetry + detections)**

  - **Logic Apps (automation + integrations + approvals)**

  - **Azure AI (summarization, triage, hunt assistance)**

## Goals

- Automate routine reporting (daily/weekly stats, trending counts, severity breakdowns).

- Reduce analyst toil by summarizing alerts/incidents with context.

- Accelerate investigation using AI-assisted KQL generation and “next query” suggestions.

- Enable controlled response (containment actions behind approvals and guardrails).

## Key Components

  1. **SIEM Logs (Microsoft Sentinel / Log Analytics)**

    Source of truth for security telemetry, alerts, and incidents and Queries are executed using KQL (scheduled or on-demand).


  2. **Agentic AI / Logic App Automation (Orchestrator)**

    The orchestration layer that decides what to query, routes results to AI analysis triggers downstream actions (formatting, email, ticketing, containment)

    In practice, this can be a Logic App workflow, an Azure Function/Container App “agent,” or a hybrid approach.

  3. **Azure AI Model (Summarization + Triage)**

    Used to:

    - filter and prioritize (e.g., High/Medium/Low, exclude informational)

    - summarize outcomes and recommended next steps

    - optionally generate investigation KQL (threat hunting support)

  4. **Context-Aware Agentic AI Analysis (Decision Hub)**

    Central “decision” stage that:

    - merges query outputs + alert metadata + enrichment context

    - determines what actions to take

            - send email

            - create ticket
              
            - kick off containment runbook
              
            - generate hunt query set

5. **Agentic Action: Format to HTML (Logic App)**

        Converts raw query output into a consistent HTML report (tables, sections, deep links).

        Produces a final payload that is “email-ready.”

6. **Agentic Action: Send Email (Logic App)**

        Sends email to NOC/SOC distribution lists and includes:

              -  severity counts and trends
                
              -  summarized narrative
                
              -  key IOCs (if applicable)
                
              -  links back to Sentinel queries/incidents

        (optional) relevant external references for investigation

7. **Ticketing Integration (ServiceNow / Other)**

        Creates an incident/case when thresholds or conditions are met.

        Helps ensure findings move into an auditable workflow with ownership/SLAs.

8. **Containment Runbooks (Logic Apps)**

        A set of Logic Apps that can be triggered based on scenario detection, such as:

                 - Malware containment (Storage Account)
                  
                 - Malware containment (Virtual Machine — often handled by Defender for Endpoint)
                  
                 - Add identified malicious traffic (IP/Domain/URL) to blocklists (Firewall + Defender)
                  
                 - Enforce onboarding of Microsoft Defender devices
                  
                 - Generate a ServiceNow incident
                  
                 - Initiate vulnerability remediation (Qualys/Tenable) with Teams approval
                  
                 - Block exploited ports on NSGs with Teams approval
                  
                 - Rotate expired certs/client secrets and store in Key Vault

**End-to-End Flow**
<img width="1741" height="966" alt="image" src="https://github.com/user-attachments/assets/bb2d514e-8200-4f15-a2d5-f54b58ea1b55" />

> **Footnote — implementation notes**  
> Daily reports can be scheduled, but **investigation/containment** should be triggered by:  
> - Sentinel Automation rules  
> - Incident creation  
> - Alert threshold crossings  
> This reduces cost and improves responsiveness.
>
> **“Human-in-the-loop” approvals for containment**  
> Containment actions (blocking, isolation, NSG changes, remediation) should typically require:  
> - Teams/Email approval step in Logic Apps  
> - A change/ticket reference (CRQ/RFC)  
> - An audit trail (who approved, when, what changed)
>
> **Constrain the AI model with a strict action contract**  
> To keep the agent safe and predictable:  
> - Use a fixed JSON schema for actions (e.g., `SendEmail`, `CreateTicket`, `RunContainmentPlaybook`)  
> - Allow AI to recommend actions, but have the orchestrator validate:  
>   - severity thresholds  
>   - allowed connectors  
>   - allowed destinations (approved email lists)  
>   - allowed runbooks
>
> **Validate AI-generated KQL before running**  
> If the model generates KQL:  
> - run it in read-only mode  
> - enforce query limits:  
>   - time range caps  
>   - row limits  
>   - table allow-list  
> - reject queries with risky operators or unexpected joins  
> - log query text + outcomes for review
>
> **Build a “state store” to prevent repeated noise**  
> Use Cosmos DB/Table Storage/Queue storage to track:  
> - last sent report hash  
> - alert/incident IDs already summarized  
> - throttling (don’t spam SOC every time a rule fires)
>
> **Enrichment should be deterministic, not just “LLM guessed”**  
> Augment summaries with structured enrichment:  
> - TI lookups (MDE TI, VirusTotal, internal feeds)  
> - Entity context (asset criticality, owner, location)  
> - Known-good allowlists (scanner IPs, backup jobs)  
> Then let the model summarize based on those facts.
>
> **Observability: treat the agent like production software**  
> Add:  
> - workflow correlation IDs  
> - structured logs (inputs, outputs, decisions)  
> - dead-letter queue for failures  
> - dashboards for report success/failures, send counts, containment triggers
>
> **Security hardening (minimum baseline)**  
> - Use Managed Identity wherever possible  
> - Store secrets in Key Vault only  
> - Least privilege RBAC:  
>   - query access scoped to workspace  
>   - strict connector permissions (ServiceNow, email, firewall APIs)  
> - Add DLP rules to prevent accidental exfiltration of sensitive log contents
>
> **Define “safe to email” rules**  
> Email is convenient but risky. Consider:  
> - removing raw event payloads  
> - redacting usernames/IPs in lower environments  
> - including only summarized findings + links to Sentinel for full detail



