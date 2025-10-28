# Implementation Plan: ChatWait Chatbot Endpoints

**Branch**: `001-chatbot-endpoints` | **Date**: 2025-10-28 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-chatbot-endpoints/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

ChatWait is a scoped chatbot service exposing two interaction modes: synchronous (`/v1/chat/wait`) for complete request/response and streaming (`/v1/chat/streaming`) for real-time token delivery via SSE. The service features a three-layer architecture with strict boundaries: Chainlit UI for demonstrations, OpenAI Agents SDK for conversation orchestration, and FastAPI for HTTP endpoints. The system is stateless (v1), uses Gemini LLM via OpenAI-compatible API, and implements streaming resilience with token-level reconnection, RFC 7807 error handling, and URL path versioning for future extensibility.

## Technical Context

**Language/Version**: Python 3.12+  
**Primary Dependencies**: FastAPI (async web framework), Chainlit (chat UI), OpenAI Agents SDK (conversation orchestration), Gemini (LLM via OpenAI-compatible API), Pydantic (data validation), httpx (async HTTP), sse-starlette (SSE support)  
**Storage**: None (v1 is stateless - no session persistence, conversation context passed in requests)  
**Testing**: pytest (test framework), pytest-asyncio (async test support), pytest-cov (coverage), httpx (test client)  
**Target Platform**: Linux/macOS/Windows server (Python runtime), containerizable (Docker)  
**Project Type**: Web API (single project with three logical layers: UI demo, Agent logic, API layer)  
**Performance Goals**: <5s response time for /v1/chat/wait, <2s time-to-first-token for /v1/chat/streaming, <200ms inter-token latency (target 100-150ms), 50 concurrent users without degradation  
**Constraints**: Async-first (no blocking I/O), stateless design (no server-side sessions in v1), RFC 7807 error format, 5000 char message limit, 60s idle timeout  
**Scale/Scope**: MVP scope - 2 endpoints, multi-turn conversations (stateless), streaming with reconnection, no authentication/persistence

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-check after Phase 1 design._

### Principle I: Test-First Development (NON-NEGOTIABLE)

- ✅ **PASS**: Tests will be written before implementation using pytest
- ✅ **PASS**: Red-Green-Refactor cycle enforced: contract tests → integration tests → unit tests before code
- ✅ **PASS**: Test coverage gates configured in pytest.ini (minimum 80% coverage required)
- ✅ **PASS**: Test categories: contract tests (API schemas), integration tests (streaming stability, reconnection), unit tests (business logic)

### Principle II: Code Quality & Type Safety

- ✅ **PASS**: All Python code will use PEP 484 type hints (function signatures, returns, class attributes)
- ✅ **PASS**: Pydantic models for all FastAPI request/response validation
- ✅ **PASS**: Linting configured: ruff (style), mypy (type checking), black (formatting)
- ✅ **PASS**: Pre-commit hooks for automated quality checks before commits

### Principle III: Async-First Architecture

- ✅ **PASS**: All FastAPI route handlers use `async def`
- ✅ **PASS**: Async HTTP client (httpx) for Gemini API calls
- ✅ **PASS**: SSE streaming inherently async (sse-starlette)
- ✅ **PASS**: OpenAI Agents SDK async operations (streaming responses)
- ✅ **PASS**: No blocking calls (requests, time.sleep) in codebase

### Principle IV: Clear Architectural Boundaries

- ✅ **PASS**: Three-layer architecture with defined boundaries:
  - UI Layer: `src/ui/` (Chainlit demo app, no business logic)
  - Agent Layer: `src/agents/` (OpenAI SDK orchestration, conversation logic)
  - API Layer: `src/api/` (FastAPI endpoints, request validation, SSE streaming)
- ✅ **PASS**: Shared models in `src/models/` (Pydantic schemas for cross-layer communication)
- ✅ **PASS**: Dependency injection for cross-layer communication (FastAPI Depends)
- ✅ **PASS**: Each layer independently testable with mocks

### Principle V: Security-First Design

- ✅ **PASS**: Environment variables for API keys (GEMINI_API_KEY, OPENAI_API_KEY)
- ✅ **PASS**: No secrets in responses, logs, or error messages
- ✅ **PASS**: RFC 7807 error responses exclude internal details (sanitized error messages)
- ✅ **PASS**: Pydantic response models explicitly exclude sensitive fields

### Principle VI: Streaming Resilience

- ✅ **PASS**: SSE with token-level sequence tracking (unique IDs per token)
- ✅ **PASS**: Reconnection support via Last-Event-ID header (SSE standard)
- ✅ **PASS**: Resume from last received token (no duplication/regeneration)
- ✅ **PASS**: Error events with RFC 7807 payload, graceful stream closure
- ✅ **PASS**: Integration tests for disconnect/reconnect scenarios

### Principle VII: Extensibility & Maintainability

- ✅ **PASS**: URL path versioning (/v1/) for future breaking changes
- ✅ **PASS**: Dependency injection pattern (FastAPI Depends) for easy extension
- ✅ **PASS**: Agent logic isolated (future: add SQLiteSession without API changes)
- ✅ **PASS**: Configuration-driven (environment variables, no hardcoded values)
- ✅ **PASS**: Separate routers for endpoint groups (APIRouter pattern)

### Performance Requirements

- ✅ **PASS**: Async design supports 50 concurrent users (event loop efficiency)
- ✅ **PASS**: Streaming latency targets: <2s TTFT, <200ms inter-token (monitored)
- ✅ **PASS**: Resource limits: stateless design minimizes memory (no session storage)

### User Experience Consistency

- ✅ **PASS**: RFC 7807 error format provides clear, actionable user-facing messages
- ✅ **PASS**: SSE events include metadata for UI progress indicators (token count, status)
- ✅ **PASS**: Consistent behavior between sync and streaming modes (same agent logic)

**GATE STATUS**: ✅ **ALL GATES PASS** - No constitution violations. Proceed to Phase 0.

## Project Structure

### Documentation (this feature)

```text
specs/001-chatbot-endpoints/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
│   ├── openapi.yaml     # OpenAPI 3.1 spec for FastAPI endpoints
│   └── sse-events.md    # SSE event format specifications
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
chatwait/
├── src/
│   ├── api/                    # API Layer (FastAPI)
│   │   ├── __init__.py
│   │   ├── main.py            # FastAPI app initialization, CORS, middleware
│   │   ├── routers/           # Endpoint routers
│   │   │   ├── __init__.py
│   │   │   ├── chat.py        # /v1/chat/wait and /v1/chat/streaming endpoints
│   │   │   └── health.py      # /health endpoint
│   │   ├── dependencies.py    # FastAPI dependencies (agent injection, validation)
│   │   └── middleware.py      # Request logging, error handling middleware
│   │
│   ├── agents/                # Agent Layer (OpenAI SDK)
│   │   ├── __init__.py
│   │   ├── chat_agent.py      # OpenAI agent orchestration, conversation logic
│   │   ├── streaming.py       # Streaming response handling, token tracking
│   │   └── config.py          # Agent configuration (Gemini API setup)
│   │
│   ├── models/                # Shared Models (Pydantic)
│   │   ├── __init__.py
│   │   ├── requests.py        # ChatRequest, StreamingRequest schemas
│   │   ├── responses.py       # ChatResponse, TokenEvent, ErrorResponse (RFC 7807)
│   │   └── conversation.py    # Message, ConversationContext schemas
│   │
│   ├── ui/                    # UI Layer (Chainlit demo)
│   │   ├── __init__.py
│   │   ├── app.py             # Chainlit app entry point
│   │   └── handlers.py        # UI event handlers (message, streaming)
│   │
│   └── utils/                 # Shared Utilities
│       ├── __init__.py
│       ├── errors.py          # RFC 7807 error builder, exception handlers
│       ├── logging.py         # Structured logging setup
│       └── config.py          # Environment variable loading, validation
│
├── tests/
│   ├── conftest.py            # Pytest fixtures (test client, mock agents)
│   ├── contract/              # Contract tests (API schemas, RFC 7807)
│   │   ├── test_chat_wait_contract.py
│   │   └── test_chat_streaming_contract.py
│   ├── integration/           # Integration tests (streaming, reconnection)
│   │   ├── test_streaming_stability.py
│   │   ├── test_reconnection.py
│   │   └── test_multi_turn.py
│   └── unit/                  # Unit tests (business logic)
│       ├── api/
│       │   └── test_routers.py
│       ├── agents/
│       │   ├── test_chat_agent.py
│       │   └── test_streaming.py
│       └── utils/
│           └── test_errors.py
│
├── .github/
│   └── workflows/
│       └── ci.yml             # CI/CD: linting, type checking, tests, coverage
│
├── pyproject.toml             # UV project configuration, dependencies
├── uv.lock                    # UV lockfile
├── pytest.ini                 # Pytest configuration (coverage, markers)
├── ruff.toml                  # Ruff linting configuration
├── mypy.ini                   # Mypy type checking configuration
├── .env.example               # Example environment variables
├── README.md                  # Setup instructions, architecture overview
└── .gitignore                 # Python, UV, IDE ignores
```

**Structure Decision**: Single project (web API) structure selected. The codebase is organized into three logical layers (API, Agent, UI) within a unified Python project managed by UV. This structure aligns with Principle IV (Clear Architectural Boundaries) while maintaining simplicity for the MVP scope. The UI layer (Chainlit) is included for demonstration purposes but can be deployed separately if needed. Tests are organized by type (contract, integration, unit) to support Test-First development (Principle I).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

**No violations detected.** All constitution principles are satisfied by the proposed architecture and technical approach.

```

```
