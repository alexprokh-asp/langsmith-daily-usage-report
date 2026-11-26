# LangSmith Daily Usage Report

Automated AI usage and cost monitoring with **LangSmith, n8n, and Telegram**.

Track token consumption, model usage, workflow activity, and estimated costs with automated daily reports delivered directly to Telegram.

## 📸 Workflow Overview

![LangSmith Daily Usage Report - n8n Workflow](doc/workflow.jpg)

## 📌 About the Project

When working with multiple AI-powered workflows, it is easy to lose track of token consumption and API costs.

**LangSmith Daily Usage Report** automates usage monitoring by collecting execution data from the LangSmith API, processing it in n8n, and sending a summarized report to Telegram every evening.

The report provides a convenient overview of AI usage over the previous 24 hours without having to open the LangSmith dashboard every day.

## ✨ Key Features

- **Automated daily reports** - scheduled execution every day at 22:00 (configurable).
- **Token consumption tracking** - total, prompt, and completion tokens.
- **Model usage breakdown** - calls, tokens, and costs grouped by AI model.
- **Workflow usage breakdown** - run counts, tokens, and costs grouped by root run name.
- **Cost monitoring** - estimated usage costs in USD.
- **Telegram notifications** - readable HTML-formatted reports.
- **Pagination support** - retrieves data across multiple pages of LangSmith API results.
- **Rolling 24-hour reporting** - covers the previous 24 hours rather than a calendar day.

## 🏗️ Architecture

The solution consists of two separate parts:

1. **Tracing integration** - sends AI execution traces from supported applications to LangSmith.
2. **Reporting workflow** - retrieves usage data from LangSmith, aggregates statistics, and delivers the report to Telegram.

### Execution Flow

The n8n workflow runs sequentially:

```text
Schedule Trigger (22:00)
          |
          v
Query LangSmith Runs (24h)
          |
          v
Query LangSmith LLM Runs (24h)
          |
          v
    Aggregate Tokens
       (JavaScript)
          |
          v
 Send Daily Report to TG
```

### Workflow Components

| Component | Responsibility |
|---|---|
| Schedule Trigger (22:00) | Starts the workflow every day at 22:00 in the workflow timezone. |
| Query LangSmith Runs (24h) | Retrieves root runs and their token and cost information. |
| Query LangSmith LLM Runs (24h) | Retrieves individual LLM runs for model-level statistics. |
| Aggregate Tokens | Processes and groups the collected data using JavaScript. |
| Send Daily Report to TG | Sends the formatted report to Telegram. |

## ⚙️ How It Works

### 1. Schedule Trigger

The workflow starts automatically every day at 22:00.

- Timezone: taken from the workflow settings (the exported file uses `Europe/Kyiv`; change it to your own).
- Cron expression:

```cron
0 22 * * *
```

### 2. Retrieve Root Runs

The first HTTP Request node sends a POST request to the LangSmith API:

```text
POST https://api.smith.langchain.com/api/v1/runs/query
```

It retrieves root runs for the configured LangSmith project.

The query includes:

- Run ID and name
- Run type
- Start time
- Prompt, completion, and total tokens
- Prompt, completion, and total costs
- Session ID

The reporting period is calculated dynamically:

```javascript
$now.minus({ hours: 24 }).toUTC().toISO()
```

Pagination is enabled to retrieve results across multiple pages.

### 3. Retrieve LLM Runs

After the root runs request completes, the second HTTP Request node queries LLM runs using the same API endpoint.

It retrieves additional information required for model-level statistics:

- Token consumption
- Model metadata
- Provider metadata
- Invocation parameters
- Cost information

This request also supports pagination.

**Important:** Both API requests execute sequentially, not in parallel.

### 4. Aggregate Usage Data

The JavaScript Code node processes the results from both HTTP requests.

It accesses root run data directly from the first request node:

```javascript
$('Query LangSmith Runs (24h)').all()
```

LLM run data is received through the current node input:

```javascript
$input.all()
```

No Merge node is required.

> **Note:** the Code node references the first request node by its name. If you rename `Query LangSmith Runs (24h)`, update this reference in the code as well.

The aggregation logic performs three main tasks.

**Root run statistics**

- Groups runs by workflow or chain name.
- Calculates run counts.
- Aggregates prompt, completion, and total tokens.
- Calculates costs.
- Builds overall usage statistics.

**LLM model statistics**

- Identifies models using available metadata and invocation parameters.
- Groups LLM runs by model.
- Collects provider information when available.
- Aggregates token consumption and costs.

**Report formatting**

- Sorts workflows and models by token consumption.
- Displays the top 10 workflows and the top 5 models.
- Collapses the remaining entries into a single summary line (count, tokens, cost).
- Formats token counts into readable K/M values.
- Formats costs in USD.
- Escapes HTML characters for Telegram message formatting.

### 5. Send Telegram Report

The final Telegram node sends the generated report using HTML formatting.

The report includes:

- Total estimated cost.
- Total token consumption.
- Number of root runs.
- Prompt and completion tokens.
- Workflow-level statistics.
- Model-level statistics.

## 📊 Report Preview

The report is generated dynamically from LangSmith data. The labels are currently in Russian; you can change them in the `Aggregate Tokens` Code node.

```text
📊 LangSmith за 24 часа
💰 $X.XX · X.XXM tok · N запусков
in X.XXM / out XXX.XK

Цепочки
• Workflow A
  N запусков · XXX.XK tok · $X.XXX
• Workflow B
  N запусков · XX.XK tok · $X.XXX
…и ещё N: XX.XK tok · $X.XXX

Модели
• model-a: N вызовов · X.XXM tok (in X.XXM / out XX.XK) · $X.XX
• model-b: N вызовов · XXX.XK tok (in XXX.XK / out XX.XK) · $X.XXX
```

## 🔌 Tracing Integration

The reporting workflow relies on execution data already collected in LangSmith.

To monitor applications such as self-hosted n8n and Langflow, configure their supported LangSmith tracing integrations.

Typical LangSmith environment variables include:

```env
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your_langsmith_api_key
LANGCHAIN_PROJECT=your_project_name
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
```

The exact configuration depends on the application and its LangSmith integration.

For Langflow, use its supported LangSmith tracing configuration. Refer to the official documentation for the environment variables and startup options applicable to your installed version.

**Note:** Tracing configuration and the daily reporting workflow are separate. The reporting workflow reads data from LangSmith through its API; it does not configure tracing for other applications.

## 🛠️ Technology Stack

| Technology | Purpose |
|---|---|
| n8n | Workflow automation and scheduling |
| LangSmith | LLM tracing and usage data |
| LangSmith API | Retrieving run and token information |
| JavaScript | Data aggregation and report generation |
| Telegram | Daily notifications |

## 🔐 Configuration

The exported workflow contains placeholders instead of real values. Replace them before activating the workflow:

| Placeholder | Where | Description |
|---|---|---|
| `YOUR_LANGSMITH_API_KEY` | Both HTTP Request nodes, `x-api-key` header | Authenticates LangSmith API requests. |
| `YOUR_LANGSMITH_PROJECT_ID` | Both HTTP Request nodes, `session` field in the JSON body | ID of the LangSmith project whose runs should be monitored. |
| `YOUR_TELEGRAM_CREDENTIAL` / `YOUR_CREDENTIAL_ID` | Telegram node | Select your own Telegram Bot credential in n8n. |
| `YOUR_TELEGRAM_CHAT_ID` | Telegram node, `chatId` field | Destination chat for the report. |

Also check the **timezone** in the workflow settings and set your own.

### Security Recommendations

- Store API keys in n8n Credentials (for example, a Header Auth credential for the LangSmith key) or in protected environment variables, rather than typing them directly into node parameters.
- Never commit API keys, Telegram bot tokens, or other secrets to GitHub.
- Remove sensitive values from exported workflow JSON files before publishing.
- Consider using a dedicated LangSmith project for a specific group of workflows.

## ⚠️ Limitations

- The report covers a rolling 24-hour period, not a calendar day.
- Each query is limited by pagination settings (100 runs per page, about 50 pages), so very high-volume projects may be truncated at roughly 5,000 runs per query.
- Costs are based on the cost information returned by LangSmith. Actual provider billing may differ.
- LangSmith usage statistics do not directly represent the remaining balance of an AI provider account.
- Model-level statistics depend on the availability of model and provider metadata in LLM runs.
- Root run statistics and LLM run statistics represent different execution levels and should not be added together.

## 📁 Repository Structure

```text
langsmith-daily-usage-report/
│
├── README.md
├── LICENSE
│
├── doc/
│   └── workflow.jpg
│
└── workflow/
    └── LangSmith_Daily_Tokens_Report_to_TG.json
```

The repository contains:

- `README.md` - project documentation.
- `LICENSE` - license file.
- `doc/workflow.jpg` - n8n workflow architecture screenshot.
- `workflow/LangSmith_Daily_Tokens_Report_to_TG.json` - exported n8n workflow, ready for import into another n8n instance.

## 🚀 Import Workflow

1. Download `LangSmith_Daily_Tokens_Report_to_TG.json` from the `workflow/` directory.
2. Open your n8n instance.
3. Create a new workflow or open the workflow editor.
4. Select **Import from File**.
5. Select the downloaded JSON file.
6. Replace all placeholders described in the [Configuration](#-configuration) section.
7. Set your timezone in the workflow settings.
8. Run the workflow once manually to verify the report.
9. Activate the workflow (the exported file is inactive by default).

## 🎯 Project Goal

Make AI usage and cost monitoring accessible without manually checking dashboards and billing pages: a small automated workflow that delivers a daily overview of token consumption, model activity, and estimated costs directly in Telegram.

## 📄 License

This project is licensed under the MIT License - see the `LICENSE` file for details.
