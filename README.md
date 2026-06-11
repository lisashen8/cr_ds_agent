# Stateful Data Agent on Cloud Run with Agent Platform Memory bank service

This repository contains an expert **Data Science Agent** built using the **Google Agent Development Kit (ADK)**. Originally written for Agent Engine, the app has been refactored for **Google Cloud Run** deployment with direct support for the interactive **ADK Web UI** and persistent, long-term memory via **Vertex AI Agent Engine (Reasoning Engine)**.

---

## 📋 Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Local Configuration](#2-local-configuration)
3. [Step 1: Provision the Memory Bank (Agent Engine)](#step-1-provision-the-memory-bank-agent-engine)
4. [Step 2: Deploy to Cloud Run & Link the Memory Bank](#step-2-deploy-to-cloud-run--link-the-memory-bank)
5. [Step 3: Enable OpenTelemetry (OTel) Tracing](#step-3-enable-opentelemetry-otel-tracing)
6. [Interactive Web UI Usage](#interactive-web-ui-usage)

---

## 1. Prerequisites

Before deploying, ensure you have the required CLI tools installed and authorized:

1. **Install and Configure Google Cloud CLI (`gcloud`):**
   ```bash
   gcloud auth login
   gcloud auth application-default login
   gcloud config set project lisa-demo-project-1
   ```

2. **Install Project Dependencies:**
   Ensure you are in the virtual environment and run:
   ```bash
   pip install -r requirements.txt
   ```

---

## 2. Local Configuration

The application uses a `.env` file to manage local environment settings. Verify your `.env` contains the following variables:

```env
# Tells ADK to use Vertex AI instead of Google AI Studio
GOOGLE_GENAI_USE_VERTEXAI=true
GOOGLE_CLOUD_PROJECT=lisa-demo-project-1
GOOGLE_CLOUD_LOCATION=us-central1

# Local OTel tracing settings
GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
```

To run a fast test locally and confirm Vertex AI connectivity:
```bash
python -m google.adk.cli run ./ "hello"
```

---

## Step 1: Provision the Memory Bank (Agent Engine)

Under Agent Engine, each instance of a Reasoning Engine automatically provisions a highly scalable, managed **Memory Bank**. Rather than rewriting any Python code, we use this managed Memory Bank as the backend for our Cloud Run deployment.

To create the Memory Bank, deploy your agent code to Agent Engine:

```bash
python -m google.adk.cli deploy agent_engine \
  --project=lisa-demo-project-1 \
  --region=us-central1 \
  --display_name="cr_ds_agent" \
  ./
```

### 📝 Capturing the ID
Once deployment finishes successfully, the CLI output will print your deployed resource path:
`Deployed to Agent Platform: projects/741251896794/locations/us-central1/reasoningEngines/8973077560640405504`

Save the Reasoning Engine ID (`8973077560640405504`). This is what we will use to link Cloud Run's long-term memory.

---

## Step 2: Deploy to Cloud Run & Link the Memory Bank

Now, deploy the application to Cloud Run with the **ADK Web UI** enabled, and link its memory service directly to your newly provisioned Agent Engine instance ID using the `--memory_service_uri` flag.

```bash
python -m google.adk.cli deploy cloud_run \
  --project=lisa-demo-project-1 \
  --region=us-central1 \
  --with_ui \
  --memory_service_uri="agentengine://<YOUR_REASONING_ENGINE_ID>" \
  ./
```

*Replace `<YOUR_REASONING_ENGINE_ID>` with your captured ID (e.g., `8973077560640405504`).*

---

## Step 3: Enable OpenTelemetry (OTel) Tracing

To monitor latency, tool executions (like BigQuery queries), and callback durations, you can explicitly configure Cloud Run to stream OpenTelemetry signals to **Google Cloud Observability (Cloud Trace)**.

To enable OTel tracing, simply add the `--trace_to_cloud` and `--otel_to_cloud` flags when deploying:

```bash
python -m google.adk.cli deploy cloud_run \
  --project=lisa-demo-project-1 \
  --region=us-central1 \
  --with_ui \
  --trace_to_cloud \
  --otel_to_cloud \
  --memory_service_uri="agentengine://<YOUR_REASONING_ENGINE_ID>" \
  ./
```

### 📊 Viewing the Spans
Every interaction turn inside the Web UI or via the REST API automatically exports OTel spans. 
* View your traces live on the [Google Cloud Trace Console](https://console.cloud.google.com/traces/list?project=lisa-demo-project-1).

---

## Interactive Web UI Usage

Once successfully deployed, the CLI will output your live service URL.
* To launch the **ADK Web UI**, append `/dev-ui/` to your Cloud Run URL:
  ```text
  https://adk-default-service-name-741251896794.us-central1.run.app/dev-ui/
  ```

### 🧠 How Memory Works in the UI:
1. **Retrieval (`PreloadMemoryTool`):** Before the agent processes any message, it queries your linked Agent Engine Memory Bank for past sessions/preferences matching your user, and proactively injects them into the agent's prompt context.
2. **Ingestion (`_save_memory`):** At the end of each conversation turn, your custom callback automatically processes the current dialogue, extracts key facts, and saves them to the Memory Bank.
