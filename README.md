# CalmTalk Therapist Onboarding

**Updated Business Case & Technical Solution**

> AI-driven email understanding, document processing, and controlled onboarding automation.
>
> _Updated November 2026 to reflect the solution as built and under evaluation._

![Power Automate](https://img.shields.io/badge/Orchestration-Power%20Automate-0066FF?logo=powerautomate&logoColor=white)
![Azure AI Foundry](https://img.shields.io/badge/AI-Azure%20AI%20Foundry-0078D4?logo=microsoftazure&logoColor=white)
![Content Understanding](https://img.shields.io/badge/Docs-Content%20Understanding-0078D4)
![Azure AI Search](https://img.shields.io/badge/Knowledge-Azure%20AI%20Search-0078D4)
![Model](https://img.shields.io/badge/Model-gpt--4.1--mini-412991)
![Status](https://img.shields.io/badge/Status-In%20Evaluation-yellow)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Process](#2-current-process)
3. [Current Pain Points](#3-current-pain-points)
4. [Recommended Automation](#4-recommended-automation)
5. [End-to-End Workflow](#5-end-to-end-workflow)
6. [Expected Agent Output](#6-expected-agent-output)
7. [Onboarding Status Lifecycle](#7-onboarding-status-lifecycle)
8. [High-Level Architecture](#8-high-level-architecture)
9. [Roles and Responsibilities](#9-roles-and-responsibilities)
10. [Governance and Control Principles](#10-governance-and-control-principles)
11. [Expected Business Benefits](#11-expected-business-benefits)
12. [Success Measures](#12-success-measures)
13. [Implementation Status](#13-implementation-status)
14. [Conclusion](#14-conclusion)

---

## 1. Executive Summary

CalmTalk's therapist onboarding process starts when a therapist signs an agreement and submits the required information and supporting documents to `therapist@ilexsystem.com`. The current process relies on manual email review, follow-up communication, document tracking, and coordination by the Operations team.

The proposed solution uses:

- **Microsoft Power Automate** as the connection and workflow orchestration layer.
- **CalmTalk-Therapist-Intake-Agent** in Azure AI Foundry to understand incoming email content and determine intents.
- **Azure AI Content Understanding** analyzer to classify submitted documents and extract their content.
- **CalmTalk onboarding API** (Azure App Service), **Azure SQL Database**, and **Azure Blob Storage** for business processing, tracking, and secure file storage.
- **Super-admin CMS** giving the Operations team a single place to review therapists and approve submitted files.

The agent's output guides workflow decisions, including the proposed reply, recommended action, and whether human intervention is required. Automated actions remain subject to workflow rules and API controls. **Final therapist approval and public activation remain under human control.**

Since the original proposal:

- The Power Automate flow (`TherapistOnboarding`) has been built.
- The intake agent has been deployed and connected to the flow.
- A therapist knowledge base has been created on Azure AI Search.
- The agent is being evaluated and optimized for token usage and cost.

See [Implementation Status](#13-implementation-status) for the current summary.

---

## 2. Current Process

After signing the agreement, the therapist sends mandatory onboarding information and supporting materials to `therapist@ilexsystem.com`. These may include:

- Signed contract or agreement
- Professional documents and certifications
- Introductory video
- Professional biography and background information
- Profile picture
- Other information required to complete the therapist profile

The same email address is used for follow-up when information or documents are missing, incomplete, or require clarification. Mandatory requirements must be received and reviewed before the therapist can be activated.

---

## 3. Current Pain Points

| Pain point | Description |
| --- | --- |
| **Incomplete submissions** | Therapists may send only part of the required information or documents. |
| **Repeated follow-ups** | Staff manually request missing items and clarification. |
| **Manual email interpretation** | Messages may contain documents, questions, payment inquiries, or multiple intents. |
| **Manual tracking** | Received files, outstanding requirements, and pending actions are difficult to track across messages. |
| **Operational dependency** | Onboarding can be delayed when the Operations Manager is unavailable. |
| **Scaling limitations** | Administrative workload increases as the therapist network grows. |

---

## 4. Recommended Automation

The proposed solution combines AI-assisted email understanding, document processing, and workflow automation. Each component has a distinct responsibility.

### 4.1 Microsoft Power Automate — Email Intake and Orchestration

- Monitor the onboarding mailbox (`therapist@ilexsystem.com`) for incoming emails; the flow is triggered by a **Gmail trigger**.
- Capture sender, subject, email body, message ID, received date, and attachment metadata.
- Upload attachments to Azure Blob Storage and route supported attachments to document classification through the Azure AI Content Understanding analyzer.
- Send the email text to the Azure AI Foundry intake agent first, using an HTTP action that calls the agent's **Responses endpoint**, and route the flow according to the returned intent and action.
- Emails with no body text are being excluded from agent calls (flow update in progress) to save calls and tokens.
- Authenticate to the agent using a **Microsoft Entra app registration with a client secret**, because Azure AI Foundry and Power Automate sit in different Entra tenants. Copilot Studio was not used.
- Coordinate downstream actions, send approved replies, invoke the CalmTalk API, and handle exceptions.

### 4.2 Azure AI Foundry Intake Agent — Email Understanding and Intent Classification

The agent analyzes the email body and relevant available context to identify one or more intents. It prepares a suitable reply, recommends the next operational action, and indicates whether human intervention is required.

`CalmTalk-Therapist-Intake-Agent` is a Foundry **prompt agent** running on **gpt-4.1-mini** and returning **strict JSON**. It is grounded on an MCP knowledge base (**Foundry IQ**) built over the CalmTalk therapist knowledge index in Azure AI Search, so replies draw on approved CalmTalk information rather than the model's general knowledge.

**Supported intents**

| Intent | Purpose |
| --- | --- |
| `DOCUMENT_SUBMISSION` | Therapist submits onboarding documents or profile information. |
| `ONBOARDING_QUERY` | Question about onboarding steps, required documents, or submission status. |
| `GENERAL_INFORMATION` | General inquiry about CalmTalk or onboarding. |
| `HUMAN_REVIEW` | Case requires human attention because of ambiguity, exception, sensitivity, or a decision outside agent authority. |
| `OTHER` | Message does not match a defined intent or cannot be confidently classified. |

> [!NOTE]
> The agent must support **multiple intents in one email**. For example, an email may include documents and a question about the onboarding steps. The agent should return both relevant intents rather than forcing the message into one category.

### 4.3 Reply Generation and Action Recommendation

Based on the identified intents and approved business information, the agent drafts an email response and recommends an operational action. Power Automate validates the response and routes the action according to defined business rules.

| Action | Purpose |
| --- | --- |
| `PROCESS_DOCUMENTS_AND_ANSWER_QUERY` | Process submitted documents and respond to an accompanying question. |
| `SEND_FOLLOW_UP` | Request missing information or documents. |
| `ANSWER_QUERY` | Respond using approved information and verified context. |
| `UPDATE_ONBOARDING_STATUS` | Request or initiate a status update based on verified processing results. |
| `ESCALATE_TO_HUMAN` | Route a case to the Operations team for review or a business decision. |
| `NO_ACTION` | No immediate operational action is required. |

The agent output includes the identified intents, the proposed reply to the therapist (`response_to_therapist`), the recommended action, and the `requires_human_review` flag.

> [!IMPORTANT]
> The workflow must not execute unsupported or unauthorized actions solely because they appear in model output.

### 4.4 Azure AI Content Understanding — Document Classification and Extraction

An Azure AI Content Understanding analyzer, called from the Power Automate flow, processes each submitted attachment. It:

- Classifies the document type (CV, passport or ID, certificate, experience letter, or other).
- Extracts the file content as JSON.
- Returns a confidence rate per file.

The flow then calls the CalmTalk API to update the file submission. Azure Document Intelligence was considered as an alternative to an LLM-based classification prompt before this approach was chosen.

> [!WARNING]
> Document receipt, content extraction, and validation are separate stages. Extracted content can support a review, but does not by itself prove that a professional credential is authentic or that a therapist is eligible for activation.

### 4.5 CalmTalk API, Azure SQL Database, and Azure Blob Storage

| Component | Role |
| --- | --- |
| **Azure Blob Storage** | Securely stores submitted files and maintains file references. |
| **Azure SQL Database** | Stores email and submission metadata, document-processing results, requirement statuses, and audit history. |
| **CalmTalk onboarding API** (Azure App Service) | Applies business rules, looks up therapists, creates submissions, accepts file submissions from the flow (including the classification result), updates onboarding records, and integrates with CRM. |
| **CalmTalk CRM** | Maintains the therapist master profile and operational onboarding status. |

### 4.6 Therapist Knowledge Base — Azure AI Search

- Azure AI Search service `calmtalkdb` holds the therapist knowledge index (`calmtalk-therapist-knowledge`), populated by an indexer and skillset that split content into chunks and generate embeddings with `text-embedding-3-large`.
- The index is exposed to the intake agent as a **Foundry IQ knowledge base** (retrieval reasoning effort set to **Minimal**) so the agent can answer onboarding questions from approved content.
- Access between the search service and the embedding model uses the search service's **system-assigned managed identity** with the **Cognitive Services OpenAI User** role.

### 4.7 Super-Admin CMS — Therapist Review and File Approval

A web CMS, built in **AngularJS** on top of the existing Azure SQL database and the updated onboarding API, supports the human review step:

1. A super admin logs in and views the list of therapists.
2. The super admin opens a therapist profile to see the submitted information and the files uploaded to Azure Blob Storage.
3. The super admin approves the submission of the files, which updates the onboarding record through the API.

This keeps document approval and activation decisions with authorized people while the automation handles intake and classification.

---

## 5. End-to-End Workflow

| # | Step | Description |
| --- | --- | --- |
| 1 | **Monitor inbox** | Power Automate detects an incoming email and captures email metadata, body, and attachments. |
| 2 | **Associate submission** | Identify the therapist and associate the email with the correct existing onboarding submission where possible. |
| 3 | **Store email and files** | Store attachments in Azure Blob Storage and save submission metadata through the CalmTalk API. |
| 4 | **Analyze email** | Send the email body and relevant context to the Foundry intake agent for intent classification, reply drafting, action recommendation, and human-review assessment. The returned intent routes the email: document submissions continue to document processing; onboarding questions trigger a lookup of the therapist's data and the documents received so far, followed by a reply; other messages follow a reply or escalation path. |
| 5 | **Process documents** | Send applicable attachments to the Azure AI Content Understanding analyzer for document classification, content extraction, and a confidence rate per file. |
| 6 | **Combine results** | Use the agent output and document-processing results with stored onboarding requirements to determine the next workflow step. |
| 7 | **Execute permitted action** | Power Automate applies workflow rules to send a reply, request missing information, route a question, or update records through the API. |
| 8 | **Track status** | Maintain submission status and requirement results, with an audit trail of processing and actions. |
| 9 | **Human review and activation** | Escalate defined exceptions and route completed profiles to the super-admin CMS for authorized human approval before activation. |

---

## 6. Expected Agent Output

The agent returns structured JSON so Power Automate can parse the result and route the workflow consistently. The following illustrates the response structure of the deployed agent; the values are an example only:

```json
{
  "intents": [
    "DOCUMENT_SUBMISSION",
    "ONBOARDING_QUERY"
  ],
  "action": "PROCESS_DOCUMENTS_AND_ANSWER_QUERY",
  "requires_human_review": false,
  "response_to_therapist": "Thank you for your email. We have received your submission. Our team will review the information and contact you if anything else is required."
}
```

---

## 7. Onboarding Status Lifecycle

Example lifecycle:

```mermaid
flowchart LR
    A[New Submission] --> B[Processing]
    B --> C[Information Missing]
    C --> D[Awaiting Therapist]
    D --> E[Information Complete]
    E --> F[Pending Human Review]
    F --> G[Approved]
    G --> H[Online]
```

- The agent can recommend actions, but actual status changes must be based on verified processing results, API business rules, and authorized workflow transitions.
- Final approval and online activation remain under human control.
- File approval and the move to **Approved** are performed by an authorized super admin in the CMS.

---

## 8. High-Level Architecture

```mermaid
flowchart TD
    T[Therapist] --> M[therapist@ilexsystem.com]
    M --> PA[Microsoft Power Automate<br/>Email intake and orchestration]
    PA --> AG[Azure AI Foundry Intake Agent<br/>Email intents, reply, action,<br/>human-review assessment]
    PA --> CU[Azure AI Content Understanding<br/>Document classification and<br/>content extraction]
    AG --> PV[Power Automate<br/>Validate and route workflow actions]
    CU --> PV
    KB[(Azure AI Search<br/>Knowledge Base)] -.-> AG
    PV --> API[CalmTalk API]
    API --> DB[(Azure SQL)]
    API --> BL[(Azure Blob Storage)]
    API --> CRM[(CalmTalk CRM)]
    DB --> CMS[Super-admin CMS<br/>Operations human review and final approval]
    BL --> CMS
    CMS --> ON[Therapist profile activated / online]
```

---

## 9. Roles and Responsibilities

| Component / Role | Responsibility |
| --- | --- |
| **Power Automate** | Email intake, workflow orchestration, action routing, reply delivery, and exception handling. |
| **Azure AI Foundry Intake Agent** | Email interpretation, intent classification, reply drafting, action recommendation, and human-review assessment. |
| **Azure AI Content Understanding** | Document classification, content extraction, and per-file confidence rate. |
| **CalmTalk Onboarding API** | Business logic, authorized onboarding updates, and CRM integration. |
| **Azure SQL / Blob Storage** | Submission tracking, audit data, and secure document storage. |
| **Azure AI Search knowledge base** | Approved therapist knowledge used to ground agent replies. |
| **Super-admin CMS** | Therapist list and profiles, review of submitted files, and file approval. |
| **Operations team** | Exception handling, business decisions, and final therapist approval. |

---

## 10. Governance and Control Principles

- The agent **recommends and structures** actions; Power Automate and the API **execute only permitted actions**.
- Replies must be based on approved knowledge and verified records; the agent must not invent policies, payment terms, or onboarding statuses.
- Email bodies and attachments are **untrusted input** and must not override system instructions or business rules.
- Document receipt, extraction, and validation are separate states.
- Human review is required for defined exceptions and final therapist activation.
- Use least-privilege access, secure document storage, appropriate retention, and audit logging.
- Handle duplicate emails, invalid agent output, API errors, and document-processing failures through explicit exception paths.
- Calls from Power Automate to the Foundry agent cross Entra tenants and are authenticated through a dedicated app registration.

---

## 11. Expected Business Benefits

- Reduce manual email interpretation and repetitive follow-ups.
- Improve consistency and relevance of therapist communications.
- Route emails to the correct workflow, including emails containing multiple intents.
- Improve visibility into submitted documents, missing requirements, and pending actions.
- Reduce administrative dependency on the Operations Manager for routine activities.
- Support a more scalable onboarding process while preserving human accountability.

---

## 12. Success Measures

Establish a baseline before rollout and agree targets with the Operations team. Suggested measures include:

- [ ] Average time from initial submission to onboarding completion
- [ ] Email intent-classification accuracy and multiple-intent detection accuracy
- [ ] Accuracy and completeness of drafted replies
- [ ] Percentage of routine emails processed without manual intervention
- [ ] Number of follow-up emails required per therapist
- [ ] Human-review rate and average exception-resolution time
- [ ] Percentage of document classifications and extracted fields confirmed as correct
- [ ] Duplicate or failed workflow executions
- [ ] Agent cost and efficiency per email: input and output tokens and cache hit rate
- [ ] Pass rate on agent evaluation scenarios, and file-approval turnaround time in the CMS
- [ ] Time from complete documentation to final activation

---

## 13. Implementation Status

Where each component stands against the original proposal:

| Component | Status | Notes |
| --- | --- | --- |
| **Power Automate flow** (`TherapistOnboarding`) | ✅ Built | Gmail trigger, attachments uploaded to Blob, classification, and API calls for therapist lookup, submission creation, and file submission. |
| **Document classification** (Content Understanding) | ✅ Integrated in flow | Returns file content as JSON with a confidence rate per file. |
| **Intake agent** (Foundry) | 🧪 Deployed; in evaluation | Connected through an HTTP action (first test call returned `200`). Evaluated with 15 simulated conversation scenarios. |
| **Therapist knowledge base** (Azure AI Search) | ✅ Built | Index, indexer, and skillset with `text-embedding-3-large`; exposed to the agent through a Foundry IQ knowledge base. |
| **Agent cost optimization** | 🚧 In progress | Reduce input and output tokens and maximize cache hits; review tools (web search, MCP) and retrieval settings; skip empty email bodies. |
| **Onboarding API updates** | 🚧 In progress | Existing API on Azure App Service being extended to support the CMS. |
| **Super-admin CMS** (AngularJS) | 🚧 In development | Therapist list, profile view, and file approval on the existing Azure SQL database. |

---

## 14. Conclusion

Integrating the Azure AI Foundry intake agent and Azure AI Content Understanding into CalmTalk's onboarding workflow extends the process beyond basic email and document automation.

- The agent interprets therapist communications, identifies one or more intents, drafts replies, recommends actions, and flags cases requiring human intervention.
- Content Understanding separately classifies documents and extracts their content.
- Power Automate coordinates these capabilities, while the CalmTalk API, Azure SQL, and Blob Storage support controlled processing and traceability.
- The super-admin CMS gives Operations a clear place to review and approve submissions.

**Final therapist approval and public activation remain under human control.**

---

<p align="center"><sub>CalmTalk | Therapist Onboarding Business Case</sub></p>
