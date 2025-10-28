<!--
SYNC IMPACT REPORT
==================
Version: 1.0.0 (Initial Constitution)
Ratification Date: 2025-10-28
Last Amended: 2025-10-28

Changes:
- ✅ NEW: Constitution created with 7 core principles
- ✅ NEW: Principle I - Test-First Development (NON-NEGOTIABLE)
- ✅ NEW: Principle II - Code Quality & Type Safety
- ✅ NEW: Principle III - Async-First Architecture
- ✅ NEW: Principle IV - Clear Architectural Boundaries
- ✅ NEW: Principle V - Security-First Design
- ✅ NEW: Principle VI - Streaming Resilience
- ✅ NEW: Principle VII - Extensibility & Maintainability
- ✅ NEW: Section on Performance Requirements
- ✅ NEW: Section on User Experience Consistency
- ✅ NEW: Governance rules for technical decision-making

Template Impact Assessment:
- ✅ plan-template.md: Constitution Check section already present, ready for principle validation
- ✅ spec-template.md: Requirements alignment compatible with new principles
- ✅ tasks-template.md: Test-first workflow enforced, task categorization supports principle-driven development

Follow-up Actions:
- None - all placeholders filled with concrete values
- Templates are compatible with new constitution structure
-->

# ChatWait Constitution

## Core Principles

### I. Test-First Development (NON-NEGOTIABLE)

**Tests MUST be written before implementation.** This principle is absolute and non-negotiable.

- Tests are written first and MUST fail before any implementation begins
- User/stakeholder approval on failing tests is required before implementation
- Strict adherence to Red-Green-Refactor cycle: Red (failing test) → Green (minimal passing code) → Refactor (improve without changing behavior)
- No production code is committed without corresponding tests
- Test coverage gates MUST pass before code review approval

**Rationale**: Test-First ensures requirements are clear before implementation, reduces bugs, enables confident refactoring, and creates living documentation of system behavior. It prevents scope creep and ensures features solve actual problems.

### II. Code Quality & Type Safety

**Strong typing and linting are mandatory for all code.**

- Python code MUST use type hints (PEP 484) on all function signatures, class attributes, and return types
- FastAPI dependencies, request/response models MUST be fully typed with Pydantic
- Linting MUST pass before commits: flake8/ruff for style, mypy for type checking, black for formatting
- Code MUST be self-documenting: clear variable names, concise functions (max 50 lines), single responsibility
- Comments explain "why" not "what" - code should be obvious enough to not need "what" comments
- Magic numbers and strings MUST be replaced with named constants

**Rationale**: Strong typing catches errors at development time, improves IDE support, makes refactoring safer, and serves as inline documentation. Linting ensures consistency across team members and reduces cognitive load during code review.

### III. Async-First Architecture

**All FastAPI endpoints and I/O operations MUST be asynchronous.**

- FastAPI route handlers MUST use `async def` unless synchronous behavior is explicitly required and justified
- Database operations, API calls, file I/O MUST use async libraries (asyncpg, httpx, aiofiles)
- Blocking operations (CPU-bound tasks) MUST be offloaded to thread/process pools using `asyncio.to_thread()` or similar
- Async context managers (`async with`) MUST be used for resource management
- No blocking calls (requests, time.sleep, synchronous DB drivers) in async contexts

**Rationale**: Async-first maximizes throughput and resource efficiency in I/O-bound applications. FastAPI is built on async foundations; synchronous code blocks the event loop and degrades performance under load.

### IV. Clear Architectural Boundaries

**Strict separation between UI (Chainlit), Agent Logic (OpenAI SDK), and API Layer (FastAPI).**

- **UI Layer (Chainlit)**: Handles user interaction, rendering, session management. Does NOT contain business logic.
- **Agent Layer (OpenAI Agents SDK)**: Orchestrates AI workflows, tool calls, conversation state. Does NOT handle HTTP concerns.
- **API Layer (FastAPI)**: Exposes HTTP endpoints, request validation, response formatting. Does NOT implement AI logic directly.
- Cross-layer communication MUST go through defined interfaces (dependency injection, abstract protocols)
- Shared models (Pydantic schemas) MAY be used across layers but business logic MUST NOT leak between layers
- Each layer MUST be independently testable with mocks/stubs for other layers

**Rationale**: Clear boundaries enable independent testing, parallel development, technology migration, and cognitive simplicity. Layers can evolve independently without cascading changes.

### V. Security-First Design

**API keys, credentials, and sensitive data MUST NEVER be exposed to users or logs.**

- Environment variables MUST be used for all secrets (API keys, database passwords, signing keys)
- Secrets MUST NOT appear in: API responses, error messages, logs, stack traces, or client-side code
- Use secret managers (dotenv locally, cloud secret managers in production)
- Pydantic models returning responses MUST exclude sensitive fields explicitly
- Logging MUST sanitize or redact sensitive data (use structured logging with field filtering)
- Error responses MUST be generic to external users; detailed errors only in debug mode or internal logs

**Rationale**: Security breaches often result from accidental exposure of credentials in logs, error messages, or responses. Proactive prevention is easier than reactive remediation.

### VI. Streaming Resilience

**Streaming responses MUST be stable, resilient to network issues, and retry-friendly.**

- Use Server-Sent Events (SSE) or WebSocket connections for streaming data from agents/LLMs
- Implement client-side reconnection logic with exponential backoff
- Include message IDs or sequence numbers to detect dropped messages
- Server MUST support resumption from last successful message ID
- Timeout handling: client must timeout idle streams, server must detect disconnected clients
- Graceful degradation: if streaming fails, fall back to polling or batch responses where applicable
- Test streaming under network failures (disconnect mid-stream, slow connections, packet loss)

**Rationale**: Real-world networks are unreliable. Streaming is a core user-facing feature; failures here destroy user experience. Resilience builds user trust and reduces support burden.

### VII. Extensibility & Maintainability

**New endpoints or agent capabilities MUST be addable with minimal changes to existing code.**

- Use dependency injection for services and configuration (FastAPI's `Depends` pattern)
- OpenAI agent tools MUST be registered via plugins or discoverable modules (avoid hardcoding tool lists)
- New API routes MUST be isolated in separate router files (APIRouter), not monolithic main.py
- Configuration-driven behavior: feature flags, model selection, tool availability should be configurable without code changes
- Follow Open/Closed Principle: open for extension (add new tools/endpoints), closed for modification (don't edit stable code)
- Breaking changes MUST be versioned (e.g., `/v1/`, `/v2/` API prefixes)

**Rationale**: Systems evolve through additions more often than rewrites. Extensibility reduces regression risk and enables rapid experimentation. Maintainability reduces cognitive load and onboarding time.

## Performance Requirements

**Performance is a feature, not an afterthought.**

- **Endpoint Latency**: Non-streaming endpoints MUST respond within 500ms at p95 under normal load
- **Streaming Time-to-First-Token**: Agent responses MUST begin streaming within 2 seconds of user input
- **Concurrent Users**: System MUST handle 100 concurrent users without degradation (target for v1.0)
- **Resource Limits**: Server processes MUST stay under 512MB memory per worker under normal load
- **Database Query Optimization**: Queries MUST use indexes; N+1 query patterns are forbidden
- **Caching**: Frequent read-heavy operations MUST use caching (Redis, in-memory LRU) with TTL
- **Monitoring**: All endpoints MUST emit metrics (latency, error rate, throughput) for observability

**Rationale**: Poor performance drives users away. Setting explicit targets enables proactive optimization and prevents performance regressions during development.

## User Experience Consistency

**Users experience a cohesive, predictable, and delightful interface.**

- **Error Messages**: User-facing errors MUST be clear, actionable, and non-technical (e.g., "Unable to process request" not "NoneType has no attribute 'id'")
- **Loading States**: All async operations MUST show progress indicators (spinners, progress bars, streaming status)
- **Keyboard & Accessibility**: UI MUST support keyboard navigation; ARIA labels for screen readers where applicable
- **Response Formatting**: Agent responses MUST maintain consistent markdown formatting, code blocks syntax-highlighted
- **Session Persistence**: User conversations MUST persist across page reloads unless explicitly cleared
- **Mobile Responsiveness**: UI MUST be usable on mobile devices (responsive design, touch-friendly controls)
- **Dark Mode**: UI MUST support dark mode with accessible contrast ratios

**Rationale**: Consistency reduces cognitive load. Predictability builds user confidence. Accessibility is both ethical and expands user base. Poor UX erodes trust even if technical implementation is solid.

## Governance

**This constitution supersedes all other development practices and guides all technical decisions.**

### Amendment Process

- Constitution amendments MUST be documented with rationale and version bump justification
- MAJOR version: Principle removal, redefinition, or backward-incompatible governance changes
- MINOR version: New principle added or material expansion of existing principle
- PATCH version: Clarifications, wording improvements, typo fixes
- Amendments MUST include Sync Impact Report assessing affected templates and docs

### Compliance & Review

- All pull requests MUST verify compliance with applicable principles in PR description
- Code reviewers MUST check constitution adherence before approval
- Constitution violations MUST be explicitly justified and documented in `plan.md` Complexity Tracking section
- Complexity that cannot be justified MUST be refactored to comply

### Technical Decision Authority

- When choosing between approaches, constitution principles are tiebreakers
- Test-First (Principle I) overrides all other considerations - no exceptions
- Security (Principle V) overrides performance optimization attempts
- Simplicity and extensibility (Principle VII) should guide architecture choices when multiple valid options exist

### Living Document

- This constitution evolves with the project
- Regular retrospectives SHOULD review whether principles still serve the project
- Proposed changes trigger discussion before adoption
- Constitution version history maintained in git

**Version**: 1.0.0 | **Ratified**: 2025-10-28 | **Last Amended**: 2025-10-28
