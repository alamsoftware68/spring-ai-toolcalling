# DateTime AI Assistant (LangChain4j + Spring Boot 4)

A Spring Boot REST service that answers natural-language questions about the current date and time in a given city, using an LLM (OpenAI's `gpt-4o-mini`) orchestrated through **LangChain4j's tool-calling (function-calling) framework**.

This project demonstrates how to wire an LLM into a Java backend so it can reliably call real application code (instead of hallucinating an answer) whenever it needs live, factual data — the same pattern used in production AI agents.

## What it does

A user asks something like:

> "What time is it in Tokyo right now?"

Instead of letting the LLM guess (LLMs have no real-time clock and no live data), the assistant is instructed to **always invoke a Java tool method** to fetch the actual current time for the requested location, then phrase a natural-language reply around that fact.

Example:

```
GET /api/datetime/time?question=What is the current time in Bangalore?

"Current date and time in Bangalore: Sunday, 16 November 2026, 03:45:12 PM IST"
```

## Key features

- **LLM tool-calling (function calling)** — `DateTimeTools.getCurrentDateTime` is exposed to the model via LangChain4j's `@Tool` annotation. The model decides *when* to call it and *what argument* to pass, based solely on the tool's description and parameter docs.
- **Grounded responses, not hallucinations** — a system prompt (`DateTimeAssistant`) explicitly forbids the model from guessing the date/time and forces it to use the tool, and to say so plainly when a location isn't supported.
- **Declarative AI service interface** — `DateTimeAssistant` is a plain Java interface; LangChain4j's `AiServices` generates the implementation at runtime (proxy pattern), wiring together the chat model, the system prompt, and the available tools with no boilerplate glue code.
- **Spring Boot dependency injection for AI components** — the chat model, tools, and assistant are all configured as regular Spring `@Bean`s in `AiConfig`, so they're testable and swappable like any other Spring component.
- **REST API** — a minimal `DateTimeController` exposes the assistant over HTTP (`GET /api/datetime/time`).
- **Timezone resolution** — supports lookups by city name or postal/ZIP code against an internal `city/zip → IANA timezone` map, formatted with `java.time` (`ZonedDateTime` + `DateTimeFormatter`).
- **Externalized configuration** — the OpenAI API key is read from an environment variable via Spring's `@Value`, never hard-coded.

## Tech stack

| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 4.1.2 (Spring MVC) |
| AI orchestration | [LangChain4j](https://github.com/langchain4j/langchain4j) 1.8.0 |
| LLM provider | OpenAI (`gpt-4o-mini`) via `langchain4j-open-ai` |
| Date/time | `java.time` (JSR-310) |
| Build | Maven |

## How it's wired together

```
HTTP request
    │
    ▼
DateTimeController          (REST endpoint: GET /api/datetime/time)
    │
    ▼
DateTimeAssistant           (LangChain4j AI service interface + system prompt)
    │
    ▼
OpenAiChatModel (gpt-4o-mini)   ──decides to call──▶  DateTimeTools.getCurrentDateTime
    │                                                        │
    └────────────────── natural-language answer ◀───────────┘
```

- `AiConfig` — Spring `@Configuration` that builds the `OpenAiChatModel`, registers `DateTimeTools`, and assembles the `DateTimeAssistant` via `AiServices.builder(...)`.
- `DateTimeAssistant` — the AI service contract; its `@SystemMessage` is the only "prompt engineering" needed to keep the model grounded and tool-using.
- `DateTimeTools` — plain Java method annotated with `@Tool`/`@P`, invoked automatically by the model when it needs real data.
- `DateTimeController` — thin REST layer that forwards the user's question to the assistant.

## Running it locally

### Prerequisites
- Java 21+
- An [OpenAI API key](https://platform.openai.com/api-keys)

### Setup

1. Set your API key as an environment variable (the app never reads a key from a committed file):

   **PowerShell**
   ```powershell
   $env:OPENAI_API_KEY = "sk-..."
   ```

   **bash**
   ```bash
   export OPENAI_API_KEY="sk-..."
   ```

2. Run the app:

   ```bash
   ./mvnw spring-boot:run
   ```

3. Call the API:

   ```bash
   curl "http://localhost:8080/api/datetime/time?question=What time is it in London?"
   ```

### Supported locations

Currently resolves a small demo set of cities/ZIP codes (Bangalore, Mumbai, New York, Los Angeles, London, Tokyo, Dubai, and a few matching postal codes) — see `DateTimeTools.ZONES`. Extending it to any IANA timezone is a matter of enriching that lookup (or swapping it for a geocoding API).

## Why this project is interesting

It's a compact, end-to-end example of **grounding an LLM with real tools in a Java/Spring stack** — the core technique behind practical AI agents (RAG, function calling, agentic workflows) rather than a toy chatbot wrapper. It shows comfort with:
- Spring Boot configuration and dependency injection
- Integrating a third-party LLM SDK (LangChain4j) idiomatically into a Spring app
- Designing prompts/tools so an LLM's output stays factually reliable
- Clean separation of concerns (controller / AI service / tools / config)
