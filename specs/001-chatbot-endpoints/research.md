# Research Document: ChatWait Technical Decisions

**Feature**: ChatWait Chatbot Endpoints  
**Created**: 2025-10-28  
**Phase**: 0 (Outline & Research)

## Overview

This document captures research findings and technical decisions for the ChatWait chatbot service implementation. All unknowns from the Technical Context have been resolved through best practices research and architectural analysis.

---

## Decision 1: FastAPI for Async Web Framework

### Decision

Use **FastAPI** as the async web framework for HTTP endpoints.

### Rationale

- **Async-first**: Built on Starlette/ASGI, native async support aligns with Principle III (Async-First Architecture)
- **Type Safety**: Automatic request validation via Pydantic aligns with Principle II (Code Quality & Type Safety)
- **Performance**: High throughput (5,000-10,000 requests/second) meets SC-005 (50 concurrent users)
- **SSE Support**: Works seamlessly with sse-starlette for streaming responses
- **OpenAPI**: Automatic API documentation generation supports contract testing
- **Ecosystem**: Large community, extensive documentation, proven production use

### Alternatives Considered

- **Flask**: Synchronous by default, requires extensions (Flask-SocketIO) for streaming, less performant
- **Django**: Heavier framework, async support less mature, overkill for MVP scope
- **Sanic**: Async-first but smaller community, less ecosystem support

### Implementation Notes

- Use `@app.get()` / `@app.post()` decorators with `async def` handlers
- Leverage `Depends()` for dependency injection (agent instances, configuration)
- Use `APIRouter` for endpoint grouping (/v1/chat/\* routes)

---

## Decision 2: OpenAI Agents SDK for Conversation Orchestration

### Decision

Use **OpenAI Agents SDK** for agent logic and conversation management.

### Rationale

- **Abstraction**: Provides high-level API for multi-turn conversations (handles context, tool calls)
- **Streaming**: Native streaming support with `stream()` method
- **Gemini Compatibility**: Works with Gemini via OpenAI-compatible API (via `base_url` parameter)
- **Session Management**: Built-in session support (SQLiteSession) - not used in v1 but enables future extensibility (Principle VII)
- **Tool Integration**: Extensible tool framework for future features (search, database queries)
- **Async Support**: Async methods (`await agent.run_async()`) align with Principle III

### Alternatives Considered

- **LangChain**: More complex, heavier dependencies, overkill for stateless chatbot
- **Direct Gemini API**: Lower-level, would require custom conversation management logic
- **Custom Agent Framework**: Reinventing the wheel, higher maintenance burden

### Implementation Notes

- Initialize agent with Gemini config: `Agent(model="gemini-pro", base_url="https://generativelanguage.googleapis.com")`
- For v1: Pass conversation context in each request (stateless)
- For v2+: Enable `SQLiteSession` or `RedisSession` for server-side session persistence

---

## Decision 3: Gemini LLM via OpenAI-Compatible API

### Decision

Use **Gemini** as the LLM, accessed via Gemini's OpenAI-compatible API endpoint.

### Rationale

- **Compatibility**: OpenAI-compatible API allows using OpenAI SDK clients without modification
- **Performance**: Gemini Pro offers fast inference (<2s TTFT) meeting SC-002
- **Cost-Effective**: Competitive pricing for production usage
- **Streaming**: Native streaming support via SSE
- **Reliability**: Enterprise-grade SLA and uptime guarantees

### Alternatives Considered

- **OpenAI GPT-4**: Higher cost, may exceed budget constraints
- **Local LLM (Ollama)**: Slower inference, requires GPU infrastructure, deployment complexity
- **Anthropic Claude**: Different API format, requires custom integration

### Implementation Notes

- Configure OpenAI client with Gemini base URL: `openai.api_base = "https://generativelanguage.googleapis.com/v1beta/openai"`
- API key via environment variable: `GEMINI_API_KEY`
- Model selection: `gemini-pro` for balance of speed and quality

---

## Decision 4: SSE (Server-Sent Events) for Streaming

### Decision

Use **SSE (Server-Sent Events)** via `sse-starlette` for /v1/chat/streaming endpoint.

### Rationale

- **Simplicity**: Unidirectional server→client, simpler than WebSocket for this use case
- **HTTP/1.1**: Works over standard HTTP, no protocol upgrade required
- **Browser Support**: Native EventSource API in all modern browsers
- **Reconnection**: Built-in reconnection logic via Last-Event-ID header (Principle VI - Streaming Resilience)
- **Event IDs**: Native support for unique event IDs (token-level tracking)
- **Text-Based**: Simple JSON payloads, easy to debug

### Alternatives Considered

- **WebSocket**: Bidirectional (unnecessary overhead), more complex connection management
- **HTTP Polling**: Inefficient, high latency (>200ms), doesn't meet inter-token latency target
- **gRPC Streaming**: Requires Protocol Buffers, overkill for simple chat streaming

### Implementation Notes

- Use `sse_starlette.EventSourceResponse` for streaming responses
- Event format: `{ id: "token_123", event: "token", data: JSON.stringify({text: "..."})} `
- Event types: `token` (normal), `end` (completion), `error` (RFC 7807 payload)
- Last-Event-ID header for reconnection resume

---

## Decision 5: Chainlit for UI Demo Layer

### Decision

Use **Chainlit** for the UI demonstration layer.

### Rationale

- **Chat-Focused**: Purpose-built for conversational AI interfaces
- **Python Integration**: Pure Python, integrates seamlessly with FastAPI backend
- **Streaming Support**: Native streaming message display
- **Quick Prototyping**: Minimal boilerplate for MVP demos
- **Multi-Turn**: Built-in conversation history UI

### Alternatives Considered

- **Gradio**: More generic ML demo tool, less chat-focused
- **Streamlit**: Page-reload model, doesn't support smooth streaming
- **Custom React Frontend**: Higher development cost, unnecessary for MVP

### Implementation Notes

- Chainlit app in `src/ui/app.py` with `@cl.on_message` handlers
- Calls FastAPI endpoints (localhost or deployed API)
- Display sync vs streaming mode toggle for demos

---

## Decision 6: UV for Project Management

### Decision

Use **UV** (Python package installer and resolver) for dependency and environment management.

### Rationale

- **Speed**: 10-100x faster than pip, faster than poetry
- **Lock Files**: Deterministic builds via `uv.lock`
- **Script Management**: `[tool.uv.scripts]` for common commands (test, lint, run)
- **Modern**: Replaces pip, pip-tools, virtualenv, poetry in one tool
- **PEP 621**: Uses standard `pyproject.toml` format

### Alternatives Considered

- **Poetry**: Slower, more complex dependency resolution
- **Pip + venv**: Manual dependency management, no lock files
- **Pipenv**: Deprecated, slow dependency resolution

### Implementation Notes

```bash
# Install dependencies
uv sync

# Run tests
uv run pytest

# Start API server
uv run uvicorn src.api.main:app --reload

# Start Chainlit UI
uv run chainlit run src/ui/app.py
```

---

## Decision 7: RFC 7807 Problem Details for Errors

### Decision

Use **RFC 7807 Problem Details** format for all error responses.

### Rationale

- **Standard**: IETF standard (RFC 7807), industry-wide adoption
- **Machine + Human Readable**: Structured (type, status) + actionable (title, detail)
- **Extensible**: Supports additional fields (instance, custom extensions)
- **Framework Support**: FastAPI exception handlers easily map to RFC 7807

### Alternatives Considered

- **Plain JSON**: Less structured, no standard semantics
- **Custom Format**: Reinventing the wheel, requires documentation

### Implementation Notes

```python
# Error response schema (Pydantic)
class ProblemDetails(BaseModel):
    type: str  # URI identifying error category
    title: str  # Short human-readable summary
    status: int  # HTTP status code
    detail: str  # Actionable explanation
    instance: str  # Request path where error occurred
```

---

## Decision 8: Token-Level Sequence Tracking for Reconnection

### Decision

Implement **token-level sequence tracking** with unique IDs for streaming reconnection.

### Rationale

- **Seamless Resumption**: Client can resume from last received token (no duplication)
- **SSE Native**: Last-Event-ID header is SSE standard for reconnection
- **Efficient**: Avoids regenerating entire response on reconnect
- **Testable**: Integration tests can verify resume behavior

### Alternatives Considered

- **Message-Level Tracking**: Coarse-grained, would duplicate partial responses
- **No Tracking**: Client must restart, poor UX for network interruptions
- **Server-Side Buffer**: Requires state management, violates stateless design

### Implementation Notes

- Token ID format: `token_{request_id}_{sequence_number}`
- Store last N tokens in temporary buffer (60s TTL, memory-only)
- Client sends `Last-Event-ID` header on reconnect
- Server resumes from sequence number + 1

---

## Decision 9: Pytest for Test Framework

### Decision

Use **pytest** with async support (`pytest-asyncio`) for all testing.

### Rationale

- **Async Testing**: `pytest-asyncio` supports async test functions
- **Fixtures**: Powerful fixture system for test setup (test clients, mocks)
- **Coverage**: `pytest-cov` integration for coverage reports
- **Markers**: Custom markers for test categories (unit, integration, contract)
- **Ecosystem**: Large plugin ecosystem, widely adopted

### Alternatives Considered

- **unittest**: Built-in but less powerful fixtures, less async support
- **nose2**: Less maintained, smaller community

### Implementation Notes

```python
# pytest.ini configuration
[pytest]
asyncio_mode = auto
testpaths = tests
markers =
    unit: Unit tests (fast, isolated)
    integration: Integration tests (slower, real dependencies)
    contract: Contract tests (API schema validation)

# Test structure
@pytest.mark.asyncio
async def test_streaming_endpoint():
    async with httpx.AsyncClient() as client:
        # Test streaming response
```

---

## Decision 10: Environment Variables for Configuration

### Decision

Use **environment variables** with `.env` file support for all configuration.

### Rationale

- **Security**: Secrets never in code (Principle V - Security-First)
- **12-Factor App**: Standard configuration pattern for cloud deployments
- **Flexibility**: Different configs for dev/staging/prod without code changes

### Implementation Notes

```bash
# .env.example
GEMINI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://generativelanguage.googleapis.com/v1beta/openai
LOG_LEVEL=INFO
MAX_MESSAGE_LENGTH=5000
IDLE_TIMEOUT_SECONDS=60
```

```python
# src/utils/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    gemini_api_key: str
    openai_api_base: str
    log_level: str = "INFO"
    max_message_length: int = 5000
    idle_timeout_seconds: int = 60

    class Config:
        env_file = ".env"
```

---

## Summary

All technical decisions align with the ChatWait Constitution principles:

- ✅ **Test-First**: pytest framework with contract/integration/unit test categories
- ✅ **Type Safety**: Pydantic models, mypy type checking
- ✅ **Async-First**: FastAPI + httpx + OpenAI SDK async methods
- ✅ **Clear Boundaries**: Separate layers (API/Agent/UI) with dependency injection
- ✅ **Security**: Environment variables, RFC 7807 sanitized errors
- ✅ **Streaming Resilience**: SSE + token tracking + reconnection
- ✅ **Extensibility**: URL versioning + session support (future)

**Research Complete** - Ready for Phase 1 (Design & Contracts).
