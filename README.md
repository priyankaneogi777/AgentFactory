# AgentFactory

> **Autonomous AI Agent Builder, Registry & Marketplace powered by Model Context Protocol (MCP) and LangGraph.**

[![Python 3.12](https://img.shields.io/badge/Python-3.12-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115+-009688.svg)](https://fastapi.tiangolo.com/)
[![React 18](https://img.shields.io/badge/React-18-61DAFB.svg)](https://react.dev/)
[![LangGraph](https://img.shields.io/badge/LangGraph-2.0+-orange.svg)](https://www.langchain.com/langgraph)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

AgentFactory allows developers and workspace members to register standardized **Model Context Protocol (MCP)** tool servers, compose autonomous **LangGraph ReAct** agents from natural-language prompts, test them in an interactive playground, and publish verified agents to a community marketplace for instant reuse across the organization.

---

## Key Platform Features

- **Prompt-Driven Agent Builder**: Create fully functional AI agents with custom system prompts and instructions—zero manual graph coding required.
- **Universal MCP Registry**: Connect any Model Context Protocol tool server (GitHub, Slack, GitLab, Postgres, Jira, Tavily) via stdio or Server-Sent Events (SSE).
- **Zero-Trust Credential Vault**: Third-party Personal Access Tokens (PATs) are validated via live API handshakes (`/user`, `auth.test`), stored encrypted in PostgreSQL, masked in the UI, and injected **Just-In-Time (JIT)**. Secrets are never exposed to the LLM context prompt.
- **Dynamic Tool Selection & Deselection**: Select required tools and deselect unused tools to avoid LLM tool sprawl.
- **Missing Token Interruption & Resume**: If an agent requires credentials not yet configured, the workflow pauses, prompts the user for the PAT, verifies it live, and resumes automatically.
- **Interactive Playground with Human-in-the-Loop**: Multi-turn chat testing with live Server-Sent Events (SSE) streaming. Destructive or write operations trigger Human-in-the-Loop interactive approval cards before execution.
- **Automated Agent Scoring Engine**: Automatically scores agents (0–100) and assigns letter grades (A–F) evaluating tool scope, prompt completeness, reliability, and test coverage.
- **Admin Governance Gate (Score $\ge$ 70)**: Platform admins review submissions with durable LangGraph checkpointing (`AsyncPostgresSaver`). Submissions with Score $\ge$ 70 are approved; submissions under 70 are rejected with actionable feedback.
- **Direct Marketplace Reuse**: Once published, any workspace member can discover and run community agents directly with zero rebuilding.

---

## End-to-End System Workflow

```mermaid
flowchart TD
    %% Global Styles
    classDef startStyle fill:#0f172a,stroke:#334155,stroke-width:2px,color:#ffffff,font-weight:bold
    classDef authStyle fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#78350f,font-weight:bold
    classDef userStyle fill:#eff6ff,stroke:#2563eb,stroke-width:2px,color:#1e3a8a
    classDef adminStyle fill:#faf5ff,stroke:#7c3aed,stroke-width:2px,color:#4c1d95
    classDef pauseStyle fill:#fffbeb,stroke:#d97706,stroke-width:2px,stroke-dasharray: 4 4,color:#92400e
    classDef gateStyle fill:#fef2f2,stroke:#dc2626,stroke-width:2px,color:#7f1d1d,font-weight:bold
    classDef successStyle fill:#ecfdf5,stroke:#059669,stroke-width:2px,color:#064e3b,font-weight:bold
    classDef marketStyle fill:#f0fdf4,stroke:#059669,stroke-width:3px,color:#064e3b,font-weight:bold

    %% 1. ENTRY & AUTH
    S1(["1. User or Admin visits Platform"]):::startStyle
    S2["2. Enter Email and Password (LoginPage.tsx)<br/>• Submits to FastAPI /auth/login"]:::authStyle
    S3{"3. Database Check (bcrypt and is_admin)<br/>Role Identification"}:::authStyle

    S1 --> S2
    S2 --> S3

    %% 2. ROLE BRANCHING
    S3 -->|is_admin == false| U1["4A. Lands on MCP Registry (MCPRegistry.tsx)<br/>• Views ONLY Admin-approved & published servers"]:::userStyle
    S3 -->|is_admin == true| A1["4B. Admin Identified (AdminReview.tsx)<br/>• Unlocks '+ Register MCP Server' & Portal"]:::adminStyle

    %% 3. USER WORKFLOW
    U1 --> U2{"Tokens connected?"}:::userStyle
    U2 -->|No / First time| U3["5A. Connections Page (Connections.tsx)<br/>• Connects PATs (GitHub, Slack, GitLab)<br/>• Live API test probe validates keys<br/>• Tokens ENCRYPTED & HIDDEN in UI"]:::userStyle
    U2 -->|Yes| U4["6A. Agent Builder Page (AgentBuilder.tsx)<br/>• Inputs Agent Name & System Prompt<br/>• Selects required MCP servers/tools<br/>• DESELECTS unneeded tools"]:::userStyle
    U3 --> U4

    U4 --> U5{"All tokens active<br/>for selected tools?"}:::userStyle
    U5 -->|Missing Tokens| U6["7A. INTERRUPT GATE<br/>• Workflow paused: prompts for token<br/>• User inputs PAT & validated live<br/>RESUMES automatically!"]:::pauseStyle
    U6 --> U7["8A. Agent Created (POST /agents)<br/>• Saved to DB under owner_id<br/>• Appears in My Agents Page"]:::userStyle
    U5 -->|All Active| U7

    U7 --> U8["9A. My Agents Overview Tab (AgentDetail.tsx)<br/>• Renders visual backend flowchart<br/>• Shows ReAct loop & MCP tool bindings"]:::userStyle
    U8 --> U9["10A. Playground Testing (Playground.tsx)<br/>• Tests prompts & use cases with chat<br/>• Backend injects PAT JIT (zero LLM leaks)<br/>• Human-in-the-Loop for write actions"]:::userStyle
    U9 --> U10["11A. Automated Scoring Engine (scoring.py)<br/>• Evaluates: Tool count, Prompt length,<br/>Reliability & Test execution history<br/>• Computes Score (0-100) & Grade (A-F)"]:::userStyle
    U10 --> U11["12A. User Submits for Publishing<br/>• POST /marketplace/publish<br/>• LangGraph calls interrupt()<br/>• State paused durably in PostgreSQL"]:::userStyle

    %% 4. ADMIN WORKFLOW
    A1 --> A2["5B. Admin Registers New MCP Server<br/>• Inputs Name, SSE/stdio URL, provider"]:::adminStyle
    A2 --> A3["6B. Test Probe with Admin PAT<br/>• Admin provides personal PAT token<br/>• Live handshake tests tool introspection"]:::adminStyle
    A3 --> A4["7B. Server Approved and Published<br/>• Commits to mcp_servers & mcp_tools<br/>• Instantly visible on MCP Registry!"]:::successStyle
    A4 -.->|Now available in catalog| U1

    %% 5. ADMIN REVIEW & GOVERNANCE GATE
    U11 --> R1["13. Admin Review Portal (AdminReview.tsx)<br/>• Inspects pending agent submission<br/>• Reviews prompt, tool bindings & score"]:::adminStyle
    A1 --> R1
    
    R1 --> R2{"14. Score Evaluation Gate<br/>Does Score >= 70?"}:::gateStyle
    
    R2 -->|< 70: Fails Gate| R3["15A. REJECT AGENT<br/>• Resume LangGraph: Command(resume='rejected')<br/>• Sent back to user with feedback!"]:::gateStyle
    R3 -.->|User refines prompt & tools| U4

    R2 -->|>= 70: Passes Gate| R4["15B. APPROVE AGENT<br/>• Resume LangGraph: Command(resume='approved')<br/>• Status set to Published"]:::successStyle

    %% 6. COMMUNITY MARKETPLACE REUSE
    R4 --> M1["16. Community Marketplace (/marketplace)<br/>• Visible to all workspace members<br/>• Trust Badge, Score, Grade & Tools"]:::marketStyle
    
    M1 --> M2["17. DIRECT AGENT REUSE (Zero Rebuilding)<br/>• ANY USER can discover and use this agent directly<br/>• No need to build or configure again!<br/>• Runs with user's own connected PAT tokens"]:::marketStyle
```

---

## Technology Stack

| Layer | Technology | Purpose |
|:---|:---|:---|
| **Frontend** | React 18, Vite, TypeScript, TailwindCSS | Single Page Application (SPA) with responsive UI, Lucide icons, SSE streaming |
| **API Gateway** | Python 3.12, FastAPI, Starlette | Async REST endpoints, Pydantic v2 validation, HS256 JWT auth |
| **Orchestration** | LangGraph, LangChain | ReAct agent execution loop, state graphs, durable checkpointing (`AsyncPostgresSaver`) |
| **Tool Protocol** | Model Context Protocol (MCP) | Universal standard for exposing tools/resources over stdio and SSE |
| **Database** | PostgreSQL 16 (Supabase) | Multi-tenant persistence: users, connections, agents, runs, marketplace reviews |
| **LLMs Supported** | Google Gemini (2.5 Flash, 3.5 Flash Lite), OpenAI | LLM reasoning, code generation, tool parameter extraction |
| **Dependency Manager** | `uv` / `pip` (Python), `npm` (Node) | High-speed virtual environments and packaging |

---

## Repository Structure

```text
AgentFactory/
|-- backend/
|   |-- app/
|   |   |-- main.py              # FastAPI application entrypoint & middleware
|   |   |-- deps.py              # Auth & database dependency injection
|   |   |-- routers/             # API routes
|   |   |   |-- auth.py          # Login, signup, JWT issuance
|   |   |   |-- mcp.py           # MCP server registration & catalog
|   |   |   |-- connections.py   # User PAT credential vault & validation
|   |   |   |-- agents.py        # Agent builder CRUD & runtime execution
|   |   |   `-- marketplace.py   # Publishing submission & admin reviews
|   |   `-- services/            # Core business logic
|   |       |-- agent_builder.py # LangGraph ReAct constructor & JIT token injector
|   |       |-- publish_graph.py # LangGraph approval state graph with interrupt()
|   |       |-- scoring.py       # Automated scoring & grading engine (0-100)
|   |       `-- github_tools.py  # Live GitHub API tools (issues, PRs, commits)
|   `-- requirements.txt         # Backend Python dependencies
|-- frontend/
|   |-- src/
|   |   |-- pages/
|   |   |   |-- LoginPage.tsx    # Authentication & role routing
|   |   |   |-- MCPRegistry.tsx  # Catalog of approved MCP servers
|   |   |   |-- Connections.tsx  # Masked PAT credential connection
|   |   |   |-- AgentBuilder.tsx # Prompt input, tool selection/deselection
|   |   |   |-- MyAgents.tsx     # User's agent portfolio
|   |   |   |-- AgentDetail.tsx  # Visual flowchart overview of backend execution
|   |   |   |-- Playground.tsx   # Interactive chat & Human-in-the-Loop approvals
|   |   |   |-- AdminReview.tsx  # Admin moderation & Score 70 threshold gate
|   |   |   `-- Marketplace.tsx  # Community marketplace & instant reuse
|   |   `-- services/api.ts      # Typed client API layer
|   `-- package.json
|-- docs/                        # Architecture diagrams & technical specifications
|   |-- high-level-architecture.html # Standalone vertical interactive architecture
|   |-- system-design-lld.html       # Full Low-Level Design document
|   `-- system-design-lld.md         # Companion markdown documentation
`-- supabase/
    `-- migrations/              # Database schemas & SQL tables
```

---

## Quick Start Guide

### Prerequisites
- **Python**: 3.12 or higher
- **Node.js**: 18.x or higher
- **Package Managers**: `npm` and [`uv`](https://github.com/astral-sh/uv) (recommended) or `pip`
- **Database**: PostgreSQL 16 instance (local or Supabase)

---

### 1. Clone the Repository

```bash
git clone https://github.com/priyankaneogi777/AgentFactory.git
cd AgentFactory
```

---

### 2. Configure Environment Variables

Create `.env` in the root (or copy `.env.example`):

```bash
cp .env.example .env
```

Set the required environment keys:
```ini
# Backend Gateway
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/agentfactory
JWT_SECRET=your_super_secret_jwt_key_here
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=1440

# LLM Keys
GEMINI_API_KEY=your_gemini_api_key_here
OPENAI_API_KEY=your_openai_api_key_here

# Frontend
VITE_API_URL=http://localhost:8000
```

---

### 3. Run Backend (FastAPI)

```bash
cd backend

# Create virtual environment and install dependencies
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Start FastAPI server on port 8000
uvicorn app.main:app --reload --port 8000
```

*The API documentation is available at [http://localhost:8000/docs](http://localhost:8000/docs).*

---

### 4. Run Frontend (React + Vite)

In a new terminal:

```bash
cd frontend

# Install npm packages
npm install

# Start development server
npm run dev
```

*Open your browser at [http://localhost:5173](http://localhost:5173).*

---

## Viewing System Design and Architecture

Interactive visual architecture diagrams and low-level design specifications are included in the repository:

```bash
# Open the pure vertical auto-fit architecture flowchart
open docs/high-level-architecture.html

# Open the comprehensive low-level design document
open docs/system-design-lld.html
```

---

## Security Architecture

1. **Vault Isolation**: External credentials (GitHub PATs, Slack Bot Tokens, GitLab Tokens) are never stored in browser state and are never passed to the LLM. The LLM only receives abstract tool signatures (e.g. `github_list_issues`). When executing, the backend intercepts the call and injects the user's encrypted token JIT.
2. **Stateless JWT Gatekeeper**: API routes authenticate requests using signed `HS256` tokens. The `is_admin` claim governs platform administrative endpoints.
3. **Human-in-the-Loop**: Destructive tool actions (e.g., delete branch, create PR, post message) pause the ReAct loop and display an approval card in the Playground before execution.

---

## License

This project is licensed under the [MIT License](LICENSE).
