[![Workflow](https://img.shields.io/badge/%E2%9C%87%EF%B8%8F_Download_Workflow-Import_to_n8n-EA4B71?style=for-the-badge&logoColor=white)](workflow/echodesk-workflow.json)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Read_the_Post-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/rohitkumardubey)
[![Built with n8n](https://img.shields.io/badge/Built_with-n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io)
[![Groq](https://img.shields.io/badge/Powered_by-Groq-F55036?style=for-the-badge&logoColor=white)](https://groq.com)

# EchoDesk

> **An automated AI workflow that reads incoming customer feedback, classifies it by sentiment, topic and urgency using structured output, and generates a tailored, context-aware reply, all without manual triage.**

---

## Table of Contents

- [Situation](#-situation)
- [Task](#-task)
- [How It Was Built](#-how-it-was-built)
- [Result](#-result)
- [Tech Stack](#-tech-stack)
- [Architecture](#-architecture)
- [Node Flow](#-node-flow)
- [Prompt Engineering](#-prompt-engineering)
- [Structured Output and Retry Logic](#-structured-output-and-retry-logic)
- [Screenshots](#-screenshots)
- [Quick Start](#-quick-start)
- [Limitations](#-limitations)
- [Future Scope](#-future-scope)
- [Links](#-links)

---

## 🔍 Situation

Sales and support teams that receive customer feedback across multiple channels face a recurring bottleneck: every message has to be read, categorised, and matched to a response by hand. Sentiment, topic, and urgency are judged inconsistently from one reviewer to the next, and low-priority praise often receives the same turnaround time as a high-urgency billing complaint.

This becomes harder to sustain as feedback volume grows, since manual review does not scale and free-form AI responses cannot reliably be parsed into fields a downstream system can act on.

---

## 🎯 Task

Build an automated pipeline that could:

- **Fetch** customer feedback from a live endpoint across multiple channels (email, support tickets)
- **Classify** each item by sentiment, topic, urgency, and key issue, in a consistent, machine-readable format
- **Generate** a reply that adapts its tone and content to the classification, rather than a generic acknowledgement
- **Validate** AI output against a defined schema, with automatic retry if the model returns malformed data
- **Route** structured results downstream for reporting, prioritisation, or CRM logging

---

## ⚙️ How It Was Built

The workflow was built in **n8n**, using **Groq** as the AI provider across two purpose-matched open-source models: a smaller model for classification and a larger model for reply generation.

### System Flow

```
Customer Feedback (Academy API)
           │
           ▼
    [ HTTP Request: GetFeedback ]
           │
           ▼
    [ Limit: SetFeedbackItem ]
           │
           ▼
 [ Basic LLM Chain: ClassifyFeedback ]
   Model: gpt-oss-20b
   Structured Output Parser
   Retry on Fail (3 attempts)
           │
           ▼
   [ Edit Fields: SetClassificationResult ]
           │
           ▼
 [ Basic LLM Chain: GenerateReply ]
   Model: gpt-oss-120b
   Tone adapts to sentiment and urgency
           │
           ▼
  [ HTTP Request: SendGeneratedReply ]
```

### Key Engineering Decisions

- **Two models, two jobs.** Classification is a simpler, well-bounded task and runs on a smaller, faster model (gpt-oss-20b). Reply generation needs more nuance and runs on a larger model (gpt-oss-120b). Matching model size to task complexity keeps cost down without sacrificing output quality.
- **Structured Output Parser over free-text parsing.** Asking the model to write a paragraph and then regex-parsing it out is fragile. Defining a JSON schema up front and validating against it makes the output directly usable by downstream nodes.
- **Retry on Fail as a safety net.** LLM output occasionally breaks schema even with a parser in place. Three retries with a short wait between attempts resolves the large majority of malformed responses without failing the whole run.
- **Classification feeds generation.** Rather than treating reply drafting as an isolated step, the classified sentiment, topic, and urgency are passed into the reply prompt, so tone and next steps are grounded in the actual analysis rather than guessed at generically.
- **Header Auth for endpoint access.** Both the feedback source and the reply submission endpoint are authenticated through a shared API key credential plus a per-request assessment ID header, keeping credentials centralised and reusable across nodes.

---

## 📊 Result

| Metric | Outcome |
|---|---|
| Feedback channels supported | Email, support ticket |
| Classification fields | Sentiment, topic, urgency, key issue |
| Classification model | openai/gpt-oss-20b (via Groq) |
| Reply generation model | openai/gpt-oss-120b (via Groq) |
| Output validation | Structured Output Parser with 3x retry on failure |
| Manual triage required | None |
| Reply tone | Adapts automatically to sentiment and urgency |

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | n8n |
| AI Provider | Groq |
| Classification Model | openai/gpt-oss-20b |
| Generation Model | openai/gpt-oss-120b |
| Output Validation | n8n Structured Output Parser |
| Authentication | Header Auth (API key and assessment ID) |
| Data Source | REST API (feedback endpoint) |

---

## 🏗 Architecture

```
┌──────────────────────────────────────────────────────────┐
│              Customer Feedback Source (API)               │
│           Email and support ticket channels                │
└───────────────────────┬──────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│                        n8n                                 │
│                                                              │
│   GetFeedback → SetFeedbackItem                             │
│                                                              │
│   ClassifyFeedback (gpt-oss-20b + Structured Output Parser) │
│                          │                                   │
│                SetClassificationResult                       │
│                          │                                   │
│         GenerateReply (gpt-oss-120b, tone-adaptive)          │
│                          │                                   │
│                  SendGeneratedReply                          │
└───────────────────────┬──────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│         Structured classification and reply delivered      │
│              back to the feedback processing API           │
└──────────────────────────────────────────────────────────┘
```

---

## 🔀 Node Flow

| Step | Node | Purpose |
|---|---|---|
| 1 | `TriggerManual` | Starts the workflow |
| 2 | `GetFeedback` | Fetches feedback items via authenticated GET request |
| 3 | `SetFeedbackItem` | Limits processing to a single feedback item |
| 4 | `ClassifyFeedback` | Classifies sentiment, topic, urgency, and key issue using a structured schema |
| 5 | `SetClassificationResult` | Extracts and consolidates the parsed classification fields |
| 6 | `GenerateReply` | Generates a reply, using the classification to set tone and next steps |
| 7 | `SendGeneratedReply` | Posts the final structured result back to the API |

---

## 🧠 Prompt Engineering

Two prompts were tuned independently for their respective tasks.

**Classification prompt**

| Decision | Reason |
|---|---|
| Fixed category sets for sentiment, topic, and urgency | Prevents free-text drift and keeps output filterable and reportable |
| Explicit schema example embedded in the system message | Reinforces the Structured Output Parser and reduces malformed responses |
| One-sentence key issue summary | Gives a fast, human-readable snapshot without reproducing the full message |

**Reply generation prompt**

| Decision | Reason |
|---|---|
| Classification passed into the prompt as context | Ties the reply's tone and content directly to the analysis rather than guessing at it |
| Conditional tone instructions (negative or urgent → empathetic, positive → appreciative) | Produces a reply that reads as considered rather than templated |
| Topic-based next steps for billing and support | Moves the customer toward resolution instead of a bare acknowledgement |
| Length capped at 2 to 3 sentences | Keeps replies concise and appropriate for direct customer communication |

---

## 🛡 Structured Output and Retry Logic

Classification output is validated against a fixed JSON schema before it is allowed to move downstream:

```
sentiment  → positive | neutral | negative
topic      → billing | product | support | general
urgency    → low | medium | high
key_issue  → one-sentence summary
```

If the model returns output that fails to match this schema, the node retries automatically up to three times with a short delay between attempts, rather than failing the entire run. This keeps the pipeline resilient to the occasional malformed response that even a well-prompted, smaller model can produce.

---

## 📸 Screenshots

**n8n Workflow Canvas**

*The full pipeline: feedback retrieval, classification with structured output, and tone-adaptive reply generation, ending in a call back to the feedback API.*

---

## 🚀 Quick Start

### Prerequisites

- An n8n instance (cloud or self-hosted)
- A Groq API key, available free at [groq.com](https://groq.com)
- Access credentials for the feedback source API

### 1. Import the Workflow

1. Open your n8n instance
2. Click the three-dot menu and select **Import**
3. Select `workflow/echodesk-workflow.json` from this repository
4. Add your credentials: Header Auth (API key and assessment ID) and Groq API
5. Activate the workflow

### 2. Required Credentials

| Credential | Where to get it | Used in node |
|---|---|---|
| Header Auth (API key) | Feedback source provider | `GetFeedback`, `SendGeneratedReply` |
| Groq API key | [groq.com](https://groq.com) | `ClassifyFeedback`, `GenerateReply` |

---

## ⚠️ Limitations

| Limitation | Detail |
|---|---|
| Single item per run | The current version processes one feedback item at a time via the Limit node |
| Model determinism | Smaller models are more prone to schema-breaking output, mitigated by retry logic but not eliminated |
| Language support | Prompts are tuned for English feedback |
| Channel scope | Currently supports email and support ticket text; does not process voice or attachments |

---

## 🚀 Future Scope

- **Batch processing** to classify and reply to multiple feedback items per run
- **CRM integration** to log classification results directly against customer records
- **Priority routing** to escalate high-urgency, negative-sentiment feedback to a human agent
- **Multi-language support** for non-English feedback
- **Analytics dashboard** summarising sentiment and topic trends over time

---

## 🔗 Links

| Resource | URL |
|---|---|
| Workflow JSON | `workflow/echodesk-workflow.json` |
| Groq | [groq.com](https://groq.com) |
| n8n Documentation | [docs.n8n.io](https://docs.n8n.io) |

---

*Built by [Rohit Kumar Dubey](https://www.linkedin.com/in/rohitkumardubey)*
