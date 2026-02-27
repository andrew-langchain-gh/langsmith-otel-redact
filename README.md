# langchain-otel-redact

Demonstrates using LangChain with OpenTelemetry to send traces through a multi-collector pipeline that redacts LLM outputs before forwarding to LangSmith.

## Architecture

```
┌─────────────────────┐
│   Python App        │
│  (LangChain + OTel) │
└────────┬────────────┘
         │ OTLP/HTTP :4318
         ▼
┌─────────────────────────────────────────────────┐
│              OTel Agent Collector                │
│                                                 │
│  Processors:                                    │
│    batch → resource/stamping →                  │
│    transform/mask_llm_output                    │
│    (replaces gen_ai.completion with REDACTED)   │
│                                                 │
│  Exporters:                                     │
│  ┌───────────┬───────────────┐                  │
│  │  *debug   │  *file        │                  │
│  │           │  (agent.log)  │                  │
│  └───────────┴───────────────┘                  │
└──────────┬──────────────────────────┬───────────┘
           │ OTLP/gRPC                │ OTLP/HTTP
           ▼                          ▼
┌─────────────────────┐    ┌────────────────────┐
│  OTel Gateway       │    │  LangSmith         │
│  Collector          │    │  (api.smith.lang-   │
│                     │    │   chain.com/otel)   │
│  Processors:        │    └────────────────────┘
│   attributes/       │
│   from_header       │
│                     │
│  Exporters:         │
│   *debug, *file     │
│   (gateway.log)     │
└─────────────────────┘

* debug and file exporters are included for local
  debugging only and would be removed in production.
```

The project runs two OpenTelemetry collectors in a pipeline:

1. **Agent collector** — Receives traces from the application, redacts `gen_ai.completion` span attributes, and forwards to both LangSmith and the gateway collector.
2. **Gateway collector** — Receives forwarded traces from the agent, extracts metadata headers, and logs to file.

The redaction is handled by the `transform/mask_llm_output` processor in the agent collector config, which replaces all LLM completion content with `REDACTED`.

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

The `LANGSMITH_API_KEY` variable is passed through to the OTel agent collector container via docker-compose, where it is used for the `x-api-key` header in both the gateway and LangSmith exporters.

## Running

### Start the OTel collectors

```sh
docker compose up
```

This starts both the agent and gateway collectors. The agent collector listens on ports 4317 (gRPC) and 4318 (HTTP).

### Install dependencies

```sh
uv sync
```

### Run the application

```sh
uv run main.py
```

This invokes a LangChain chain that asks for a programming joke. The resulting trace is sent via OTLP to the agent collector, which redacts the LLM output and forwards the trace onward.
