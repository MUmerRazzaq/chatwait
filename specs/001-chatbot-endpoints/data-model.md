# Data Model: ChatWait Entities

**Feature**: ChatWait Chatbot Endpoints  
**Created**: 2025-10-28  
**Phase**: 1 (Design & Contracts)

## Overview

This document defines the data entities, their relationships, validation rules, and state transitions for the ChatWait chatbot service. All entities are represented as Pydantic models to ensure type safety (Principle II) and automatic validation.

---

## Entity Diagram

```
┌─────────────────┐
│  ChatRequest    │
│                 │
│  - message      │──────┐
│  - context?     │      │
│  - request_id?  │      │
└─────────────────┘      │
                         │ contains
                         ▼
                  ┌──────────────────┐
                  │ ConversationCtx  │
                  │                  │
                  │  - messages[]    │◄─────┐
                  │  - metadata      │      │
                  └──────────────────┘      │
                         │                  │
                         │ contains         │ part of
                         ▼                  │
                  ┌──────────────────┐      │
                  │    Message       │──────┘
                  │                  │
                  │  - role          │
                  │  - content       │
                  │  - timestamp     │
                  │  - id?           │
                  └──────────────────┘
                         ▲
                         │ produces
                         │
┌─────────────────┐      │
│  ChatResponse   │──────┘
│                 │
│  - message      │
│  - context      │
│  - metadata     │
└─────────────────┘

┌─────────────────┐
│   TokenEvent    │  (Streaming only)
│                 │
│  - event_type   │
│  - token_id     │
│  - data         │
│  - timestamp    │
└─────────────────┘

┌─────────────────┐
│ ProblemDetails  │  (Errors - RFC 7807)
│                 │
│  - type         │
│  - title        │
│  - status       │
│  - detail       │
│  - instance     │
└─────────────────┘
```

---

## Entity Definitions

### 1. Message

Represents a single conversational turn (user input or assistant response).

**Fields:**

| Field     | Type                         | Required | Validation                               | Description               |
| --------- | ---------------------------- | -------- | ---------------------------------------- | ------------------------- |
| role      | Literal["user", "assistant"] | Yes      | Must be "user" or "assistant"            | Message author role       |
| content   | str                          | Yes      | 1-5000 characters, non-empty after strip | Message text content      |
| timestamp | datetime                     | Yes      | ISO 8601 format                          | When message was created  |
| id        | Optional[str]                | No       | UUID v4 format if provided               | Unique message identifier |

**Validation Rules:**

- `content` must not be empty or whitespace-only
- `content` length ≤ 5000 characters (per FR-010)
- `timestamp` must be UTC timezone
- `id` is auto-generated if not provided (UUIDv4)

**Pydantic Model:**

```python
from pydantic import BaseModel, Field, field_validator
from datetime import datetime
from typing import Literal, Optional
import uuid

class Message(BaseModel):
    role: Literal["user", "assistant"]
    content: str = Field(..., min_length=1, max_length=5000)
    timestamp: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    id: Optional[str] = Field(default_factory=lambda: str(uuid.uuid4()))

    @field_validator("content")
    @classmethod
    def validate_content(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Message content cannot be empty or whitespace")
        return v
```

---

### 2. ConversationContext

Represents the history of messages in a multi-turn dialog.

**Fields:**

| Field    | Type           | Required | Validation                            | Description                  |
| -------- | -------------- | -------- | ------------------------------------- | ---------------------------- |
| messages | list[Message]  | Yes      | 1-100 messages, alternating roles     | Ordered conversation history |
| metadata | dict[str, Any] | No       | Max 10 keys, values JSON-serializable | Additional context metadata  |

**Validation Rules:**

- `messages` list must not be empty
- Messages should alternate between "user" and "assistant" roles (soft validation, warning only)
- Maximum 100 messages per context (prevent excessive context size)
- `metadata` values must be JSON-serializable (str, int, float, bool, list, dict)

**State Transitions:**

- Initial: Empty list (new conversation)
- Active: 1-100 messages
- No explicit "ended" state (stateless design)

**Pydantic Model:**

```python
from pydantic import BaseModel, Field, field_validator
from typing import Any

class ConversationContext(BaseModel):
    messages: list[Message] = Field(..., min_length=1, max_length=100)
    metadata: dict[str, Any] = Field(default_factory=dict)

    @field_validator("messages")
    @classmethod
    def validate_messages(cls, v: list[Message]) -> list[Message]:
        if len(v) > 100:
            raise ValueError("Conversation context cannot exceed 100 messages")
        return v
```

---

### 3. ChatRequest

Represents an incoming request to either /v1/chat/wait or /v1/chat/streaming.

**Fields:**

| Field      | Type                          | Required | Validation                   | Description                      |
| ---------- | ----------------------------- | -------- | ---------------------------- | -------------------------------- |
| message    | str                           | Yes      | 1-5000 characters, non-empty | User's current message           |
| context    | Optional[ConversationContext] | No       | Valid conversation context   | Previous conversation history    |
| request_id | Optional[str]                 | No       | UUID v4 format               | Idempotency key (for future use) |

**Validation Rules:**

- `message` follows same rules as Message.content (1-5000 chars, non-empty)
- If `context` provided, must be valid ConversationContext
- `request_id` is optional (for future idempotency support)

**Pydantic Model:**

```python
from pydantic import BaseModel, Field
from typing import Optional

class ChatRequest(BaseModel):
    message: str = Field(..., min_length=1, max_length=5000)
    context: Optional[ConversationContext] = None
    request_id: Optional[str] = None

    @field_validator("message")
    @classmethod
    def validate_message(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Message cannot be empty or whitespace")
        return v
```

---

### 4. ChatResponse

Represents the chatbot's generated output from /v1/chat/wait.

**Fields:**

| Field    | Type                     | Required | Validation                   | Description                             |
| -------- | ------------------------ | -------- | ---------------------------- | --------------------------------------- |
| message  | Message                  | Yes      | Valid assistant message      | Generated response message              |
| context  | ConversationContext      | Yes      | Updated conversation context | Full conversation including new message |
| metadata | Optional[dict[str, Any]] | No       | JSON-serializable            | Response metadata (tokens, latency)     |

**Validation Rules:**

- `message.role` must be "assistant"
- `context.messages` must include the new response message
- `metadata` may include: `token_count`, `generation_time_ms`, `model_used`

**Pydantic Model:**

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional, Any

class ChatResponse(BaseModel):
    message: Message
    context: ConversationContext
    metadata: Optional[dict[str, Any]] = None

    @field_validator("message")
    @classmethod
    def validate_message_role(cls, v: Message) -> Message:
        if v.role != "assistant":
            raise ValueError("Response message must have role 'assistant'")
        return v
```

---

### 5. TokenEvent (SSE Streaming)

Represents a single server-sent event in streaming mode.

**Fields:**

| Field      | Type                             | Required        | Validation                         | Description                            |
| ---------- | -------------------------------- | --------------- | ---------------------------------- | -------------------------------------- |
| event_type | Literal["token", "end", "error"] | Yes             | One of: token, end, error          | SSE event type                         |
| token_id   | str                              | Yes (token/end) | Format: `token_{request_id}_{seq}` | Unique token identifier                |
| data       | str \| dict                      | Yes             | JSON-serializable                  | Event payload (token text or metadata) |
| timestamp  | datetime                         | Yes             | ISO 8601 UTC                       | Event generation time                  |

**Event Type Payloads:**

**token** event:

```json
{
  "event_type": "token",
  "token_id": "token_abc123_001",
  "data": { "text": "Hello", "sequence": 1 },
  "timestamp": "2025-10-28T10:30:00Z"
}
```

**end** event:

```json
{
  "event_type": "end",
  "token_id": "token_abc123_final",
  "data": { "total_tokens": 42, "generation_time_ms": 1234 },
  "timestamp": "2025-10-28T10:30:05Z"
}
```

**error** event:

```json
{
  "event_type": "error",
  "token_id": "token_abc123_error",
  "data": {<RFC 7807 ProblemDetails>},
  "timestamp": "2025-10-28T10:30:03Z"
}
```

**Pydantic Model:**

```python
from pydantic import BaseModel, Field
from datetime import datetime, timezone
from typing import Literal, Union, Any

class TokenEvent(BaseModel):
    event_type: Literal["token", "end", "error"]
    token_id: str = Field(..., pattern=r"^token_[a-zA-Z0-9_]+$")
    data: Union[str, dict[str, Any]]
    timestamp: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
```

---

### 6. ProblemDetails (RFC 7807 Error Response)

Represents structured error information for all failure scenarios.

**Fields:**

| Field    | Type | Required | Validation                       | Description                       |
| -------- | ---- | -------- | -------------------------------- | --------------------------------- |
| type     | str  | Yes      | URI format                       | Error category identifier         |
| title    | str  | Yes      | 1-100 characters                 | Short human-readable summary      |
| status   | int  | Yes      | Valid HTTP status code (400-599) | HTTP status code                  |
| detail   | str  | Yes      | 1-500 characters                 | Actionable explanation            |
| instance | str  | Yes      | URI path format                  | Request path where error occurred |

**Common Error Types:**

| Type URI                      | Title               | Status | Use Case                               |
| ----------------------------- | ------------------- | ------ | -------------------------------------- |
| `/errors/validation-error`    | Validation Error    | 400    | Invalid request format, missing fields |
| `/errors/message-too-long`    | Message Too Long    | 400    | Message exceeds 5000 characters        |
| `/errors/invalid-context`     | Invalid Context     | 400    | Malformed conversation context         |
| `/errors/generation-failed`   | Generation Failed   | 500    | LLM generation error                   |
| `/errors/service-unavailable` | Service Unavailable | 503    | Gemini API unavailable                 |
| `/errors/timeout`             | Request Timeout     | 504    | Generation exceeded timeout            |

**Validation Rules:**

- `type` should start with `/errors/` for consistency
- `title` must be concise (≤100 chars)
- `detail` must be actionable (≤500 chars), no internal stack traces
- `status` must match HTTP response status code

**Pydantic Model:**

```python
from pydantic import BaseModel, Field, HttpUrl

class ProblemDetails(BaseModel):
    type: str = Field(..., pattern=r"^/errors/[a-z-]+$")
    title: str = Field(..., min_length=1, max_length=100)
    status: int = Field(..., ge=400, le=599)
    detail: str = Field(..., min_length=1, max_length=500)
    instance: str = Field(..., pattern=r"^/v\d+/.*$")

    class Config:
        json_schema_extra = {
            "example": {
                "type": "/errors/validation-error",
                "title": "Validation Error",
                "status": 400,
                "detail": "The 'message' field is required and cannot be empty.",
                "instance": "/v1/chat/wait"
            }
        }
```

---

## Relationships

### Message → ConversationContext

- **Cardinality**: Many-to-One (many messages in one context)
- **Ownership**: ConversationContext owns Message instances
- **Cascade**: Deleting context removes all messages (in-memory, no persistence)

### ChatRequest → ConversationContext

- **Cardinality**: One-to-One (optional)
- **Ownership**: ChatRequest references existing context
- **Mutability**: Context is immutable in request (read-only)

### ChatResponse → ConversationContext

- **Cardinality**: One-to-One (required)
- **Ownership**: ChatResponse returns updated context
- **Mutability**: New context instance created (immutable pattern)

### TokenEvent → None

- **Standalone**: TokenEvent is ephemeral (streaming only)
- **No Persistence**: Events are not stored, only transmitted

---

## Data Flow Example

### Synchronous Flow (/v1/chat/wait)

```
1. Client sends ChatRequest:
   {
     "message": "What is Python?",
     "context": {
       "messages": [
         {"role": "user", "content": "Hello", "timestamp": "..."},
         {"role": "assistant", "content": "Hi! How can I help?", "timestamp": "..."}
       ]
     }
   }

2. API validates request (Pydantic automatic)

3. API passes to Agent Layer

4. Agent appends user message to context

5. Agent calls Gemini LLM with full context

6. Agent receives complete response

7. Agent creates new Message (role="assistant")

8. API returns ChatResponse:
   {
     "message": {
       "role": "assistant",
       "content": "Python is a high-level programming language...",
       "timestamp": "2025-10-28T10:30:00Z",
       "id": "uuid-here"
     },
     "context": {
       "messages": [
         ... previous messages ...,
         {"role": "user", "content": "What is Python?", ...},
         {"role": "assistant", "content": "Python is...", ...}
       ]
     },
     "metadata": {
       "token_count": 87,
       "generation_time_ms": 1234
     }
   }
```

### Streaming Flow (/v1/chat/streaming)

```
1. Client sends ChatRequest (same as above)

2. API validates request

3. API establishes SSE connection (EventSourceResponse)

4. Agent streams tokens from Gemini

5. For each token:
   - Agent emits TokenEvent(event_type="token", token_id="token_req123_001", data={"text": "Python"})
   - API sends SSE event: `id: token_req123_001\nevent: token\ndata: {"text": "Python"}\n\n`

6. On completion:
   - Agent emits TokenEvent(event_type="end", data={"total_tokens": 87})
   - API sends SSE event: `id: token_req123_final\nevent: end\ndata: {...}\n\n`

7. API closes SSE stream

8. Client has full response + conversation context for next turn
```

---

## Summary

All entities support the constitution principles:

- ✅ **Type Safety (Principle II)**: All entities are Pydantic models with full type hints
- ✅ **Validation**: Field validators enforce FR-010 (message length), RFC 7807 format
- ✅ **Stateless**: No database persistence (v1) - context passed in requests
- ✅ **Extensibility (Principle VII)**: Models support future fields via `metadata` dicts
- ✅ **Security (Principle V)**: ProblemDetails excludes sensitive details

**Phase 1 (Data Model) Complete** - Ready for API contract generation.
