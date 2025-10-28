# ChatWait Quickstart Guide

**Feature**: ChatWait Chatbot Endpoints  
**Version**: 1.0.0  
**Last Updated**: 2025-10-28

## Overview

ChatWait is a chatbot service with two interaction modes:

- **Synchronous** (`/v1/chat/wait`): Complete responses in one payload
- **Streaming** (`/v1/chat/streaming`): Real-time token delivery via SSE

This guide walks you through setup, development, testing, and deployment.

---

## Prerequisites

- **Python**: 3.12 or later
- **UV**: Python package manager ([install guide](https://github.com/astral-sh/uv))
- **Gemini API Key**: Get from [Google AI Studio](https://makersuite.google.com/app/apikey)
- **Git**: For version control

---

## Quick Start (5 minutes)

### 1. Clone and Setup

```bash
# Clone repository
git clone https://github.com/your-org/chatwait.git
cd chatwait

# Checkout feature branch
git checkout 001-chatbot-endpoints

# Install dependencies with UV
uv sync

# Copy environment template
cp .env.example .env
```

### 2. Configure Environment

Edit `.env` file:

```bash
# Required
GEMINI_API_KEY=your_gemini_api_key_here

# Optional (defaults provided)
OPENAI_API_BASE=https://generativelanguage.googleapis.com/v1beta/openai
LOG_LEVEL=INFO
MAX_MESSAGE_LENGTH=5000
IDLE_TIMEOUT_SECONDS=60
```

### 3. Run Tests (Test-First Verification)

```bash
# Run all tests
uv run pytest

# Run specific test categories
uv run pytest -m unit          # Unit tests only
uv run pytest -m integration   # Integration tests only
uv run pytest -m contract      # Contract tests only

# With coverage report
uv run pytest --cov=src --cov-report=html
```

**Expected Output**: All tests pass ✅ (80%+ coverage)

### 4. Start API Server

```bash
# Development mode (auto-reload)
uv run uvicorn src.api.main:app --reload --port 8000

# Production mode
uv run uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --workers 4
```

**Verify**: Visit http://localhost:8000/docs (OpenAPI documentation)

### 5. Start Chainlit UI (Demo)

```bash
# In a separate terminal
uv run chainlit run src/ui/app.py --port 8001
```

**Verify**: Visit http://localhost:8001 (Chat interface)

---

## Project Structure

```
chatwait/
├── src/
│   ├── api/           # FastAPI layer (HTTP endpoints)
│   ├── agents/        # OpenAI SDK layer (conversation logic)
│   ├── models/        # Pydantic schemas (shared)
│   ├── ui/            # Chainlit layer (demo UI)
│   └── utils/         # Utilities (errors, logging, config)
├── tests/
│   ├── contract/      # API schema tests
│   ├── integration/   # Streaming + reconnection tests
│   └── unit/          # Business logic tests
├── specs/             # Feature specifications & plans
├── .github/           # CI/CD workflows
├── pyproject.toml     # UV project config
└── .env               # Environment variables (gitignored)
```

---

## Development Workflow

### Test-First Development (Constitution Principle I)

**Workflow**: Red → Green → Refactor

1. **Write failing test** (Red):

```python
# tests/unit/api/test_chat_routes.py
@pytest.mark.asyncio
async def test_chat_wait_returns_response():
    # Arrange
    request = ChatRequest(message="Hello")

    # Act
    response = await chat_wait(request)

    # Assert
    assert response.message.role == "assistant"
    assert len(response.message.content) > 0
```

2. **Run test** (should fail):

```bash
uv run pytest tests/unit/api/test_chat_routes.py -v
```

3. **Implement minimal code** (Green):

```python
# src/api/routers/chat.py
@router.post("/v1/chat/wait", response_model=ChatResponse)
async def chat_wait(request: ChatRequest) -> ChatResponse:
    # Minimal implementation
    agent = get_agent()
    response = await agent.generate(request.message, request.context)
    return response
```

4. **Run test again** (should pass):

```bash
uv run pytest tests/unit/api/test_chat_routes.py -v
```

5. **Refactor** (improve without changing behavior):

```python
# src/api/routers/chat.py
@router.post("/v1/chat/wait", response_model=ChatResponse)
async def chat_wait(
    request: ChatRequest,
    agent: ChatAgent = Depends(get_agent)
) -> ChatResponse:
    """Synchronous chat interaction."""
    return await agent.generate_response(
        message=request.message,
        context=request.context
    )
```

### Pre-Commit Quality Gates

```bash
# Install pre-commit hooks (one-time)
uv run pre-commit install

# Manual run (what CI will check)
uv run ruff check src tests    # Linting
uv run mypy src                # Type checking
uv run black src tests --check # Formatting
uv run pytest --cov=src        # Tests + coverage
```

**Gates**: All must pass before commit ✅

---

## API Usage Examples

### Example 1: Synchronous Chat (Python)

```python
import httpx

async def chat_sync():
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://localhost:8000/v1/chat/wait",
            json={
                "message": "What is Python?",
                "context": None  # First message, no context
            }
        )

        data = response.json()
        print(f"Response: {data['message']['content']}")

        # Save context for next turn
        conversation_context = data['context']

        # Second message (multi-turn)
        response = await client.post(
            "http://localhost:8000/v1/chat/wait",
            json={
                "message": "Can you give me an example?",
                "context": conversation_context
            }
        )

        print(f"Response: {response.json()['message']['content']}")

# Run
import asyncio
asyncio.run(chat_sync())
```

### Example 2: Streaming Chat (Python)

```python
import httpx
import json

async def chat_stream():
    async with httpx.AsyncClient() as client:
        async with client.stream(
            "POST",
            "http://localhost:8000/v1/chat/streaming",
            json={"message": "Explain async programming"},
            headers={"Accept": "text/event-stream"}
        ) as response:
            buffer = ""
            async for chunk in response.aiter_text():
                buffer += chunk

                # Parse SSE events
                while "\n\n" in buffer:
                    event_text, buffer = buffer.split("\n\n", 1)

                    # Parse event fields
                    event_data = {}
                    for line in event_text.split("\n"):
                        if line.startswith("event: "):
                            event_data["event"] = line[7:]
                        elif line.startswith("data: "):
                            event_data["data"] = json.loads(line[6:])

                    # Handle event types
                    if event_data.get("event") == "token":
                        print(event_data["data"]["text"], end="", flush=True)
                    elif event_data.get("event") == "end":
                        print(f"\n\nDone! {event_data['data']['total_tokens']} tokens")
                    elif event_data.get("event") == "error":
                        print(f"\n\nError: {event_data['data']['detail']}")

asyncio.run(chat_stream())
```

### Example 3: Streaming with Reconnection (JavaScript)

```javascript
let lastEventId = null;
let conversationContext = null;

function connectStream(message) {
  const eventSource = new EventSource("/v1/chat/streaming", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Last-Event-ID": lastEventId || "",
    },
    body: JSON.stringify({
      message: message,
      context: conversationContext,
    }),
  });

  eventSource.addEventListener("token", (event) => {
    lastEventId = event.lastEventId;
    const data = JSON.parse(event.data);
    document.getElementById("output").innerText += data.text;
  });

  eventSource.addEventListener("end", (event) => {
    const data = JSON.parse(event.data);
    conversationContext = data.conversation_context;
    eventSource.close();
  });

  eventSource.addEventListener("error", (event) => {
    const data = JSON.parse(event.data);
    console.error("Stream error:", data.detail);
    eventSource.close();

    // Retry with exponential backoff
    setTimeout(() => connectStream(message), 1000);
  });
}

// Usage
connectStream("Explain Python async");
```

---

## Testing Guide

### Unit Tests (Fast, Isolated)

**Purpose**: Test business logic in isolation with mocks.

```python
# tests/unit/agents/test_chat_agent.py
@pytest.mark.unit
@pytest.mark.asyncio
async def test_agent_generates_response(mock_gemini_client):
    # Arrange
    agent = ChatAgent(client=mock_gemini_client)
    mock_gemini_client.generate.return_value = "Mocked response"

    # Act
    response = await agent.generate_response("Hello")

    # Assert
    assert response.message.content == "Mocked response"
    assert response.message.role == "assistant"
```

**Run**: `uv run pytest -m unit --fast`

### Integration Tests (Real Dependencies)

**Purpose**: Test streaming stability, reconnection, multi-turn conversations.

```python
# tests/integration/test_streaming_stability.py
@pytest.mark.integration
@pytest.mark.asyncio
async def test_streaming_handles_disconnection():
    # Arrange
    async with httpx.AsyncClient() as client:
        # Start stream
        response = await client.post("/v1/chat/streaming", json={"message": "Hi"})

        # Simulate disconnect after 3 tokens
        tokens_received = []
        count = 0
        async for event in parse_sse_stream(response):
            if event["event"] == "token":
                tokens_received.append(event["data"]["text"])
                count += 1
                if count == 3:
                    break  # Simulate disconnect

        last_event_id = f"token_abc_003"

        # Reconnect with Last-Event-ID
        response = await client.post(
            "/v1/chat/streaming",
            json={"message": "Hi"},
            headers={"Last-Event-ID": last_event_id}
        )

        # Assert: stream resumes from token 4
        first_event = await parse_next_sse_event(response)
        assert first_event["data"]["sequence"] == 4
```

**Run**: `uv run pytest -m integration`

### Contract Tests (API Schemas)

**Purpose**: Validate OpenAPI spec compliance, RFC 7807 errors.

```python
# tests/contract/test_chat_wait_contract.py
@pytest.mark.contract
@pytest.mark.asyncio
async def test_chat_wait_response_matches_schema():
    # Arrange
    schema = load_openapi_schema()

    # Act
    response = await client.post("/v1/chat/wait", json={"message": "Hi"})

    # Assert
    validate_against_schema(response.json(), schema["ChatResponse"])
    assert response.status_code == 200
    assert response.headers["content-type"] == "application/json"
```

**Run**: `uv run pytest -m contract`

---

## Deployment

### Local Development

```bash
# API server
uv run uvicorn src.api.main:app --reload --port 8000

# Chainlit UI
uv run chainlit run src/ui/app.py --port 8001
```

### Docker Container

```bash
# Build image
docker build -t chatwait:latest .

# Run container
docker run -d \
  -p 8000:8000 \
  -e GEMINI_API_KEY=your_key_here \
  --name chatwait \
  chatwait:latest
```

### Production (Example: Render)

```bash
# Install dependencies
uv sync --no-dev

# Start with gunicorn + uvicorn workers
uv run gunicorn src.api.main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000
```

---

## Troubleshooting

### Issue: Tests Failing

**Symptom**: `pytest` fails with import errors

**Solution**:

```bash
# Ensure dependencies installed
uv sync

# Run from project root
cd /path/to/chatwait
uv run pytest
```

### Issue: Gemini API Errors

**Symptom**: 401 Unauthorized or 403 Forbidden

**Solution**:

1. Verify API key in `.env` file
2. Check key validity at [Google AI Studio](https://makersuite.google.com/app/apikey)
3. Ensure `OPENAI_API_BASE` points to Gemini endpoint

### Issue: Streaming Disconnects

**Symptom**: SSE connection drops frequently

**Solution**:

1. Check network stability
2. Verify `Last-Event-ID` header format
3. Ensure token buffer hasn't expired (60s TTL)
4. Review server logs for timeout errors

### Issue: Type Checking Errors

**Symptom**: `mypy` reports type errors

**Solution**:

```bash
# Install type stubs
uv add --dev types-httpx types-pydantic

# Rerun type check
uv run mypy src
```

---

## Next Steps

1. **Review Constitution**: Read `.specify/memory/constitution.md` to understand development principles
2. **Read Specification**: Review `specs/001-chatbot-endpoints/spec.md` for feature requirements
3. **Explore Contracts**: Check `specs/001-chatbot-endpoints/contracts/` for API schemas
4. **Run Tests**: Execute `uv run pytest -v` to see test structure
5. **Start Development**: Follow Test-First workflow (Red-Green-Refactor)

---

## Resources

- **FastAPI Documentation**: https://fastapi.tiangolo.com/
- **OpenAI Agents SDK**: https://github.com/openai/openai-python
- **Chainlit Documentation**: https://docs.chainlit.io/
- **UV Package Manager**: https://github.com/astral-sh/uv
- **SSE Specification**: https://html.spec.whatwg.org/multipage/server-sent-events.html
- **RFC 7807**: https://datatracker.ietf.org/doc/html/rfc7807

---

## Support

For issues or questions:

1. Check `specs/001-chatbot-endpoints/plan.md` for architecture details
2. Review test examples in `tests/` directory
3. Consult constitution for development principles
4. Open an issue on GitHub (if applicable)

**Happy Coding!** 🚀
