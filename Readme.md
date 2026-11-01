# Customer Support AI Agent

**An intelligent, database-backed customer support agent powered by LLMs, tool calling, and Python.**

Build an AI agent that can understand customer queries, retrieve real order information, and perform support operations through controlled backend tools.

The project focuses on engineering reliable AI agents rather than building a simple chatbot wrapper.

<p align="center">
  <strong>Python · FastAPI · PostgreSQL · SQLAlchemy · OpenAI · Docker</strong>
</p>

> **Status:** Under active development. See the [Development Roadmap](#development-roadmap) for progress.

---

## Table of Contents

- [Overview](#overview)
- [Key Capabilities](#key-capabilities)
- [System Architecture](#system-architecture)
- [Agent Execution Lifecycle](#agent-execution-lifecycle)
- [Technology Stack](#technology-stack)
- [Database Design](#database-design)
- [Agent Tools](#agent-tools)
- [Reliability & Security Design](#reliability--security-design)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [API Usage](#api-usage)
- [Testing](#testing)
- [Development Roadmap](#development-roadmap)
- [Engineering Principles](#engineering-principles)

---

## Overview

Customer support systems often depend on rigid workflows, predefined responses, and manual intervention.

This project explores a more flexible approach using an LLM-powered AI agent that can understand natural-language requests, decide which tools are required, retrieve information from a real database, and execute approved actions.

For example:

- A customer asks about an order.
- The agent identifies that order information is required.
- It calls the appropriate backend tool.
- The backend retrieves the actual order from PostgreSQL.
- The result is passed back to the LLM.
- The agent generates a response based on the retrieved information.

The system is designed around a key principle:

**The LLM handles reasoning and tool selection. The backend handles authorization, validation, and execution.**

The model never gets direct access to the database or unrestricted SQL execution.

## Key Capabilities

### AI Agent & Tool Calling
- LLM-powered request understanding.
- Dynamic tool selection based on user intent.
- Structured function calling.
- Tool execution through a controlled registry.
- Reasoning and observation loop.

### Customer Support Operations
- Retrieve order status from PostgreSQL.
- Verify customer ownership of orders.
- Create support tickets for eligible delayed orders.
- Prevent duplicate ticket creation using idempotency.

### Backend Engineering
- REST API using FastAPI.
- PostgreSQL as the source of truth.
- SQLAlchemy 2.0 for database operations.
- Alembic for database schema migrations.
- Pydantic for request validation.

### Reliability & Observability
- Bounded agent execution.
- Structured tool execution logs.
- Agent run persistence.
- Error handling and controlled retries.
- Database transaction management.
- Execution traceability.

*Features are being implemented incrementally. This list describes the intended project capabilities, not a claim that every feature is already complete.*

---

## System Architecture

```mermaid
flowchart TD
    A[Customer] --> B[FastAPI]
    B --> C[Authentication & Validation]
    C --> D[Agent Runtime]
    D --> E[OpenAI LLM]
    E --> F{Tool Required?}
    F -->|Yes| G[Tool Registry]
    F -->|No| H[Final Response]
    G --> I[Backend Validation]
    I --> J[Service Layer]
    J --> K[(PostgreSQL)]
    K --> J
    J --> L[Tool Result]
    L --> E
    E --> H
    H --> M[Customer Response]
    D --> N[Agent Execution Logs]
```

## Agent Execution Lifecycle

The agent follows a controlled execution loop inspired by the ReAct pattern.

```mermaid
flowchart TD
    A[Receive Customer Query] --> B[Send Context to LLM]
    B --> C{LLM Decision}
    C -->|Tool Call| D[Validate Tool Arguments]
    D --> E{Valid?}
    E -->|No| F[Return Structured Error]
    E -->|Yes| G[Execute Backend Tool]
    G --> H[Query or Update PostgreSQL]
    H --> I[Return Tool Result]
    I --> B
    F --> B
    C -->|Final Answer| J[Persist Agent Run]
    J --> K[Return Response]
```

### Execution Steps

1. Receive a customer request through the API.
2. Resolve the authenticated customer's identity.
3. Send the request and relevant instructions to the LLM.
4. Let the LLM select an available tool if necessary.
5. Validate the requested function and its arguments.
6. Execute the tool through the backend service layer.
7. Return the tool result to the LLM.
8. Continue until a final response is generated or an execution limit is reached.
9. Persist the execution result and tool-call history.

---

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Language | Python 3.11+ | Core application |
| API | FastAPI | REST endpoints |
| LLM | OpenAI API | Reasoning and tool selection |
| Database | PostgreSQL | Persistent application data |
| ORM | SQLAlchemy 2.0 | Database interaction |
| Migrations | Alembic | Schema versioning |
| Validation | Pydantic | Input and output validation |
| Containerization | Docker Compose | Local infrastructure |
| Testing | Pytest | Unit and integration tests |

---

## Database Design

PostgreSQL is the application's source of truth. Customer and order information is retrieved from actual database records rather than hardcoded dictionaries or mock responses.

### Database Entities

| Entity | Responsibility |
|---|---|
| `customers` | Customer identity and account details |
| `orders` | Customer orders and delivery status |
| `support_tickets` | Support requests and ticket status |
| `agent_runs` | Individual agent execution records |
| `agent_tool_calls` | Tool arguments, results, and execution status |

### Entity Relationships

```mermaid
erDiagram
    CUSTOMERS ||--o{ ORDERS : places
    ORDERS ||--o{ SUPPORT_TICKETS : has
    CUSTOMERS ||--o{ AGENT_RUNS : initiates
    AGENT_RUNS ||--o{ AGENT_TOOL_CALLS : contains

    CUSTOMERS {
        uuid id PK
        string name
        string email
        timestamp created_at
    }

    ORDERS {
        uuid id PK
        uuid customer_id FK
        string status
        decimal amount
        timestamp created_at
    }

    SUPPORT_TICKETS {
        uuid id PK
        uuid order_id FK
        string issue
        string status
        string idempotency_key UK
        timestamp created_at
    }

    AGENT_RUNS {
        uuid id PK
        uuid customer_id FK
        text user_message
        text final_response
        string status
        timestamp created_at
    }

    AGENT_TOOL_CALLS {
        uuid id PK
        uuid run_id FK
        string tool_name
        json arguments
        json result
        boolean succeeded
        timestamp created_at
    }
```

### Database Migration Strategy

Database schema changes are managed using Alembic.

- Version-controlled database migrations.
- Incremental schema updates.
- Migration history tracking.
- Controlled upgrades and downgrades.
- Consistent database structure across development and deployment environments.

---

## Agent Tools

The initial agent exposes a restricted set of backend functions.

### 1. `get_order_status`

Retrieves the status of a specific order.

**Input:**

```json
{
  "order_id": "order-uuid"
}
```

**Responsibilities:**
- Validate the order identifier.
- Verify that the order belongs to the authenticated customer.
- Retrieve the actual order status.
- Return structured information to the agent.

### 2. `create_support_ticket`

Creates a support ticket for an eligible delayed order.

**Input:**

```json
{
  "order_id": "order-uuid",
  "issue": "My delivery has been delayed."
}
```

**Responsibilities:**
- Validate the order.
- Verify customer ownership.
- Check whether the order qualifies for a support ticket.
- Prevent duplicate ticket creation.
- Persist the ticket in PostgreSQL.
- Return the ticket identifier and status.

Customer identity is obtained from authenticated backend context, not from LLM-generated tool arguments.

---

## Reliability & Security Design

Reliable agent execution requires more than successful LLM responses.

The project is designed to address the following engineering challenges:

| Challenge | Approach |
|---|---|
| Repeated tool execution | Idempotency |
| Invalid tool arguments | Schema validation |
| Unknown or hallucinated tool names | Tool registry allow-list |
| Infinite agent loops | Maximum iteration limits |
| Temporary dependency failures | Controlled retries |
| Database failures | Transaction handling |
| Unauthorized order access | Customer-scoped queries |
| Prompt injection | Authorization enforced in backend code, not in the prompt |
| External API delays | Request timeouts |
| Debugging agent behavior | Persistent execution traces |
| Sensitive information exposure | Controlled tool outputs |

Additional security measures planned for deployment include authenticated customer sessions, authorization testing, secrets management, rate limiting, and structured audit logging.

---

## Project Structure

```text
customer-support-agent/
│
├── app/
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── runtime.py
│   │   ├── prompts.py
│   │   └── tools.py
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── order_service.py
│   │   └── ticket_service.py
│   │
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── models.py
│   └── schemas.py
│
├── alembic/
│   ├── versions/
│   └── env.py
│
├── tests/
│   ├── test_services.py
│   └── test_agent.py
│
├── .env.example
├── .gitignore
├── alembic.ini
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

## Getting Started

### Prerequisites

Make sure you have:

- Python 3.11 or later
- Docker and Docker Compose
- PostgreSQL-compatible environment (Docker Compose is used here)
- OpenAI API key
- Git

### 1. Clone the repository

```bash
git clone <repository-url>
cd customer-support-agent
```

### 2. Create a virtual environment

```bash
python -m venv .venv
```

Activate it on Windows:

```powershell
.venv\Scripts\Activate.ps1
```

Activate it on macOS / Linux:

```bash
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure environment variables

Create a `.env` file using `.env.example`:

```env
APP_NAME=Customer Support Agent
ENVIRONMENT=development

DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/support_agent

OPENAI_API_KEY=your_openai_api_key
OPENAI_MODEL=your_available_model

MAX_AGENT_ITERATIONS=5
```

Do not commit `.env` or expose API credentials.

### 5. Start PostgreSQL

```bash
docker compose up -d
```

### 6. Apply database migrations

```bash
alembic upgrade head
```

### 7. Start the API server

```bash
uvicorn app.main:app --reload
```

The API will be available at:

```text
http://127.0.0.1:8000
```

Interactive API documentation:

```text
http://127.0.0.1:8000/docs
```

---

## API Usage

### Health Check

```http
GET /health
```

### Chat with the Agent

```http
POST /chat
Content-Type: application/json
X-Customer-ID: <customer-uuid>
```

**Request:**

```json
{
  "message": "Can you check the status of my order?"
}
```

**Illustrative response:**

```json
{
  "run_id": "agent-run-uuid",
  "response": "Your order is currently delayed.",
  "status": "completed"
}
```

The response is illustrative. The actual result depends on the customer's database records and agent execution.

**Development note:** The `X-Customer-ID` header is intended only for local development. Production deployments must resolve customer identity from verified authentication credentials.

---

## Testing

Run the test suite:

```bash
pytest -v
```

Planned test coverage:

- Order lookup.
- Customer ownership validation.
- Invalid order identifiers.
- Support ticket creation.
- Duplicate ticket prevention.
- Tool argument validation.
- Unknown tool rejection.
- Agent execution limits.
- Database failure handling.
- Unauthorized access attempts.
- Prompt-injection attempts to access another customer's data.

Integration tests should use an isolated test database rather than production data.

---

## Development Roadmap

### Phase 1 — Foundation
- [ ] Initialize project structure.
- [ ] Configure PostgreSQL with Docker Compose.
- [ ] Integrate SQLAlchemy.
- [ ] Define database models.
- [ ] Configure Alembic migrations.

### Phase 2 — Backend Services
- [ ] Implement order retrieval.
- [ ] Implement customer ownership checks.
- [ ] Implement support ticket creation.
- [ ] Add idempotency protection.
- [ ] Test database services.

### Phase 3 — Agent Runtime
- [ ] Integrate OpenAI Responses API.
- [ ] Define structured function tools.
- [ ] Implement tool registry.
- [ ] Build the agent execution loop.
- [ ] Handle tool results and final responses.
- [ ] Persist agent runs and tool calls.

### Phase 4 — API Integration
- [ ] Build FastAPI endpoints.
- [ ] Integrate customer authentication.
- [ ] Connect API requests to the agent runtime.
- [ ] Validate request and response schemas.

### Phase 5 — Reliability Engineering
- [ ] Add bounded retries and backoff.
- [ ] Add dependency timeouts.
- [ ] Implement circuit breaker behavior.
- [ ] Improve database transaction handling.
- [ ] Handle concurrent and duplicate operations.
- [ ] Add graceful failure recovery.

### Phase 6 — Testing & Observability
- [ ] Unit tests.
- [ ] Integration tests.
- [ ] Adversarial authorization tests.
- [ ] Agent behavior evaluation.
- [ ] Structured logging and metrics.
- [ ] End-to-end testing.

### Phase 7 — Deployment
- [ ] Dockerize the application.
- [ ] Configure production environment variables.
- [ ] Add deployment health checks.
- [ ] Configure production database.
- [ ] Deploy the application.

---

## Engineering Principles

This project is built around the following principles:

**1. Database as the source of truth**

The agent must retrieve actual application data instead of inventing or assuming customer information.

**2. Controlled tool execution**

The LLM can request operations but cannot directly execute database queries or bypass backend authorization.

**3. Explicit state and traceability**

Agent runs and tool executions should be inspectable and useful for debugging.

**4. Reliability over demos**

Transient failures, repeated requests, invalid arguments, and unexpected tool responses must be considered during implementation.

**5. Incremental complexity**

Start with a reliable single-agent architecture before introducing multi-agent orchestration.

---

## Project Objective

The goal is to build and understand a production-oriented AI agent architecture by implementing the complete lifecycle of an agent: request understanding, reasoning, tool selection, backend execution, state management, persistence, reliability, evaluation, and deployment.

Rather than treating AI agents as simple chatbot interfaces, this project focuses on the backend engineering principles required to make them useful, secure, and reliable in real-world applications.

---

**Status:** Under active development.
