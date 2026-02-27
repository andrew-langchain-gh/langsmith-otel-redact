# langsmith-otel-redact

Demonstrates using LangChain with OpenTelemetry to send traces through a collector that redacts LLM outputs before forwarding to LangSmith.

## Architecture

```
┌─────────────────────┐
│   Python App        │
│  (LangChain + OTel) │
└────────┬────────────┘
         │ OTLP/HTTP :4318
         ▼
┌─────────────────────────────────────────────────┐
│              OTel Collector                      │
│                                                 │
│  Processors:                                    │
│    batch → transform/mask_llm_output            │
│    (replaces gen_ai.completion with REDACTED)   │
│                                                 │
│  Exporters:                                     │
│  ┌───────────┬────────────────┬──────────────┐  │
│  │  *debug   │  *file         │ otlphttp/    │  │
│  │           │  (collector.   │ langsmith    │  │
│  │           │   log)         │              │  │
│  └───────────┴────────────────┴──────────────┘  │
└─────────────────────────────────┬────────────────┘
                                  │ OTLP/HTTP
                                  ▼
                       ┌────────────────────┐
                       │  LangSmith         │
                       │  (api.smith.lang-   │
                       │   chain.com/otel)   │
                       └────────────────────┘

* debug and file exporters are included for local
  debugging only and would be removed in production.
```

The project runs a single OpenTelemetry collector that receives traces from the application, redacts `gen_ai.completion` span attributes using the `transform/mask_llm_output` processor, and forwards the redacted traces to LangSmith.

## Prerequisites

- [uv](https://docs.astral.sh/uv/)
- [Docker](https://docs.docker.com/get-docker/) and Docker Compose
- An OpenAI API key
- A LangSmith API key

## Setup

1. Copy the `.env` file and fill in your API keys:

```
LANGSMITH_OTEL_ENABLED="true"
LANGSMITH_TRACING="true"
LANGSMITH_OTEL_ONLY="true"
LANGSMITH_API_KEY="<your-langsmith-api-key>"
LANGSMITH_PROJECT="<your-langsmith-project>"
OPENAI_API_KEY="<your-openai-api-key>"
```

The `LANGSMITH_API_KEY` variable is passed through to the OTel collector container via docker-compose, where it is used for the `x-api-key` header when exporting to LangSmith.

## Running

### Start the OTel collector

```sh
docker compose up
```

This starts the collector, which listens on ports 4317 (gRPC) and 4318 (HTTP).

### Install dependencies

```sh
uv sync
```

### Run the application

```sh
uv run main.py
```

This invokes a LangChain chain that asks for a programming joke. The resulting trace is sent via OTLP to the collector, which redacts the LLM output and forwards the trace to LangSmith.
