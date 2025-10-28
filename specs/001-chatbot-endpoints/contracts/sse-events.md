# SSE Event Specifications

**Feature**: ChatWait Streaming Endpoint  
**Protocol**: Server-Sent Events (SSE) / EventSource  
**Endpoint**: `/v1/chat/streaming`

## Overview

This document specifies the Server-Sent Events (SSE) format for the streaming chat endpoint. SSE provides unidirectional real-time communication from server to client with built-in reconnection support.

---

## SSE Protocol Basics

### Connection Flow

```
1. Client sends POST to /v1/chat/streaming with ChatRequest body
2. Server validates request
3. Server responds with:
   - Status: 200 OK
   - Content-Type: text/event-stream
   - Cache-Control: no-cache
   - Connection: keep-alive
4. Server sends events as they are generated
5. Server closes connection after final event
```

### Event Format

SSE events follow this text-based format:

```
id: <event_id>
event: <event_type>
data: <json_payload>
<blank line>
```

- **id**: Unique identifier for reconnection (Last-Event-ID)
- **event**: Event type (token, end, error)
- **data**: JSON payload (can span multiple lines)
- **blank line**: Signals end of event

---

## Event Types

### 1. Token Event

**Purpose**: Deliver a single generated token to the client.

**Event Type**: `token`

**ID Format**: `token_{request_id}_{sequence_number}`

- `request_id`: Unique identifier for this streaming request (UUID)
- `sequence_number`: Zero-padded 3-digit sequence (001, 002, ...)

**Data Payload**:

```json
{
  "text": "token_text",
  "sequence": 1
}
```

**Fields**:

- `text` (string, required): The generated token text
- `sequence` (integer, required): Token sequence number (1-indexed)

**Example**:

```
id: token_abc123_001
event: token
data: {"text": "Python", "sequence": 1}

id: token_abc123_002
event: token
data: {"text": " is", "sequence": 2}

id: token_abc123_003
event: token
data: {"text": " a", "sequence": 3}
```

**Performance Target**: <200ms between consecutive token events (target: 100-150ms average)

---

### 2. End Event

**Purpose**: Signal completion of response generation.

**Event Type**: `end`

**ID Format**: `token_{request_id}_final`

**Data Payload**:

```json
{
  "total_tokens": 87,
  "generation_time_ms": 1234,
  "conversation_context": { ... }
}
```

**Fields**:

- `total_tokens` (integer, required): Total number of tokens generated
- `generation_time_ms` (integer, required): Total generation time in milliseconds
- `conversation_context` (ConversationContext, required): Updated conversation context including the new assistant message

**Example**:

```
id: token_abc123_final
event: end
data: {"total_tokens": 87, "generation_time_ms": 1234, "conversation_context": {"messages": [...]}}
```

**Client Action**: After receiving `end` event, close EventSource connection.

---

### 3. Error Event

**Purpose**: Communicate errors that occur during streaming (after connection established).

**Event Type**: `error`

**ID Format**: `token_{request_id}_error`

**Data Payload**: RFC 7807 Problem Details

```json
{
  "type": "/errors/generation-failed",
  "title": "Generation Failed",
  "status": 500,
  "detail": "The AI model encountered an error during generation. Please try again.",
  "instance": "/v1/chat/streaming"
}
```

**Example**:

```
id: token_abc123_error
event: error
data: {"type": "/errors/generation-failed", "title": "Generation Failed", "status": 500, "detail": "The AI model encountered an error during generation. Please try again.", "instance": "/v1/chat/streaming"}
```

**Client Action**:

- Display error message to user
- Close EventSource connection
- Optionally retry with exponential backoff

**Server Action**: After sending `error` event, close SSE stream gracefully.

---

## Reconnection Mechanism

### How Reconnection Works

SSE provides built-in reconnection via the `Last-Event-ID` header:

1. **Client receives events**: Stores last received event `id`
2. **Connection drops**: Network interruption, server timeout, etc.
3. **Client reconnects**: Sends `Last-Event-ID` header with last received `id`
4. **Server resumes**: Continues streaming from the next token after `Last-Event-ID`

### Last-Event-ID Header

**Format**: `Last-Event-ID: token_{request_id}_{sequence_number}`

**Example**:

```
POST /v1/chat/streaming HTTP/1.1
Content-Type: application/json
Last-Event-ID: token_abc123_005

{
  "message": "What is Python?",
  "context": { ... }
}
```

**Server Behavior**:

- Parse `Last-Event-ID` to extract `sequence_number`
- Resume streaming from `sequence_number + 1`
- If `sequence_number` is too old (buffer expired), send `error` event with `type: "/errors/reconnection-failed"`

### Reconnection Buffer

**Server-Side Temporary Buffer**:

- Store last N tokens (e.g., 100 tokens) per request in memory
- TTL: 60 seconds (matches idle timeout)
- Use LRU eviction policy
- Buffer key: `request_id`

**Buffer Expiry**:
If client reconnects but buffer has expired:

```
id: token_abc123_error
event: error
data: {"type": "/errors/reconnection-failed", "title": "Reconnection Failed", "status": 410, "detail": "The streaming session has expired. Please start a new request.", "instance": "/v1/chat/streaming"}
```

---

## Timeout and Idle Handling

### Idle Connection Timeout

**Timeout Duration**: 60 seconds of no activity

**Server Behavior**:

- If no tokens generated for 60 seconds, send timeout event:

```
id: token_abc123_timeout
event: error
data: {"type": "/errors/timeout", "title": "Request Timeout", "status": 504, "detail": "The generation process took too long and was terminated. Please try again.", "instance": "/v1/chat/streaming"}
```

- Close SSE stream after timeout event

**Client Behavior**:

- Detect timeout event
- Display error to user
- Optionally retry with exponential backoff

---

## Client Implementation Example (JavaScript)

```javascript
// Establish SSE connection
const eventSource = new EventSource("/v1/chat/streaming", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    // Include Last-Event-ID for reconnection
    "Last-Event-ID": lastEventId || "",
  },
  body: JSON.stringify({
    message: "What is Python?",
    context: conversationContext,
  }),
});

// Track last event ID for reconnection
let lastEventId = null;

// Handle token events
eventSource.addEventListener("token", (event) => {
  lastEventId = event.lastEventId;
  const data = JSON.parse(event.data);

  // Append token to UI
  appendToken(data.text);

  console.log(`Received token ${data.sequence}: ${data.text}`);
});

// Handle end event
eventSource.addEventListener("end", (event) => {
  const data = JSON.parse(event.data);

  // Update conversation context
  conversationContext = data.conversation_context;

  console.log(
    `Stream complete: ${data.total_tokens} tokens in ${data.generation_time_ms}ms`
  );

  // Close connection
  eventSource.close();
});

// Handle error events
eventSource.addEventListener("error", (event) => {
  const data = JSON.parse(event.data);

  // Display error to user
  displayError(data.title, data.detail);

  console.error(`Stream error: ${data.type} - ${data.detail}`);

  // Close connection
  eventSource.close();

  // Optionally retry with exponential backoff
  if (retryCount < maxRetries) {
    setTimeout(() => reconnect(), Math.pow(2, retryCount) * 1000);
    retryCount++;
  }
});

// Handle connection errors (network issues, before stream starts)
eventSource.onerror = (error) => {
  console.error("SSE connection error:", error);

  // EventSource will automatically retry connection
  // If you want manual control, close and implement custom retry
  if (eventSource.readyState === EventSource.CLOSED) {
    // Connection closed permanently
    handleConnectionClosed();
  }
};
```

---

## Error Scenarios

### 1. Validation Error (Before Stream Starts)

**Scenario**: Request validation fails before SSE stream is established.

**HTTP Response**: 400 Bad Request (JSON, not SSE)

```json
{
  "type": "/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "The 'message' field is required and cannot be empty.",
  "instance": "/v1/chat/streaming"
}
```

**Client Action**: Do NOT establish EventSource, handle as regular HTTP error.

---

### 2. Generation Failure (Mid-Stream)

**Scenario**: LLM fails during token generation (after stream started).

**SSE Event**:

```
id: token_abc123_error
event: error
data: {"type": "/errors/generation-failed", "title": "Generation Failed", "status": 500, "detail": "The AI model encountered an error. Please try again.", "instance": "/v1/chat/streaming"}
```

**Server**: Close stream after error event.  
**Client**: Close EventSource, optionally retry.

---

### 3. Timeout (No Tokens for 60s)

**Scenario**: Generation takes longer than 60 seconds with no progress.

**SSE Event**:

```
id: token_abc123_timeout
event: error
data: {"type": "/errors/timeout", "title": "Request Timeout", "status": 504, "detail": "The generation process took too long. Please try again.", "instance": "/v1/chat/streaming"}
```

**Server**: Close stream after timeout event.  
**Client**: Close EventSource, optionally retry.

---

### 4. Reconnection Buffer Expired

**Scenario**: Client reconnects but buffer has been garbage collected.

**SSE Event**:

```
id: token_abc123_error
event: error
data: {"type": "/errors/reconnection-failed", "title": "Reconnection Failed", "status": 410, "detail": "The streaming session has expired. Please start a new request.", "instance": "/v1/chat/streaming"}
```

**Server**: Close stream after error event.  
**Client**: Start new request (cannot resume).

---

## Performance Monitoring

### Metrics to Track

**Server-Side**:

- **Time-to-First-Token (TTFT)**: Time from request receipt to first token event (target: <2s)
- **Inter-Token Latency**: Time between consecutive token events (target: 100-150ms average, max 200ms)
- **Total Generation Time**: Time from first token to end event
- **Reconnection Rate**: Percentage of requests with at least one reconnection
- **Buffer Hit Rate**: Percentage of successful reconnections vs. expired buffers

**Client-Side**:

- **Perceived Latency**: Time from user message to first token displayed
- **Reconnection Success Rate**: Percentage of successful reconnections
- **Error Rate**: Percentage of streams ending in error events

### Logging

**Example Log Entry**:

```json
{
  "timestamp": "2025-10-28T10:00:00Z",
  "event": "streaming_complete",
  "request_id": "abc123",
  "ttft_ms": 1234,
  "avg_inter_token_ms": 125,
  "total_tokens": 87,
  "total_time_ms": 12000,
  "reconnections": 1
}
```

---

## Summary

SSE event specifications align with constitution principles:

- ✅ **Streaming Resilience (Principle VI)**: Token-level tracking + Last-Event-ID reconnection
- ✅ **Security (Principle V)**: RFC 7807 errors exclude internal details
- ✅ **Performance Requirements**: <2s TTFT, <200ms inter-token targets
- ✅ **User Experience Consistency**: Clear event types, actionable error messages

**Contract Complete** - Ready for implementation.
