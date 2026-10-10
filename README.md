<div align="center">

# 🏥 Healthcare Voice Contact Center

**A production-quality, real-time voice AI agent for healthcare clinics.**  
Patients call in, speak naturally, and the AI handles bookings, FAQs, and emergencies — no hold music, no menus.

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![Claude](https://img.shields.io/badge/Claude-Haiku_4.5-D4A017?style=for-the-badge&logo=anthropic&logoColor=white)](https://anthropic.com)
[![Pipecat](https://img.shields.io/badge/Pipecat-1.3.0-FF6B6B?style=for-the-badge)](https://pipecat.ai)
[![Cartesia](https://img.shields.io/badge/Cartesia-STT_%2F_TTS-00C4B4?style=for-the-badge)](https://cartesia.ai)
[![License](https://img.shields.io/badge/License-MIT-22C55E?style=for-the-badge)](LICENSE)

</div>

---

## ✨ What it does

| Capability | How |
|---|---|
| 📅 **Appointment booking** | Checks live availability, confirms name & phone, books the slot |
| 💬 **FAQ answering** | Clinic hours, insurance, billing, directions — instant |
| 🚨 **Emergency escalation** | Detects life-threatening keywords → switches to Supervisor voice + prompt |
| 🎙️ **Natural voice** | Warm human-like TTS with emotional tone, not robotic |
| 🔄 **Personality switching** | Front-desk Aria → Supervisor Marcus, live mid-call |

---

## 🏗️ Current Architecture (v1)

> Real-time single-pipeline voice agent — everything runs in one process.

```mermaid
flowchart LR
    A([🎙️ Caller]) -->|WebRTC / Daily\naudio stream| B

    subgraph PIPELINE ["⚙️  Pipecat Pipeline"]
        direction LR
        B[🔊 Cartesia STT\nink-whisper] --> C[🧠 Claude Haiku\nTool Calling]
        C --> D[🔈 Cartesia TTS\nsonic-3 · emotion=content]
    end

    D -->|audio out| E([🎙️ Caller])

    C -->|check_availability| F[(📄 slots.json)]
    C -->|book_appointment| F
    C -->|lookup_faq| G[(📄 faq.json)]
    C -->|escalate_to_human| H[🚨 Supervisor\nVoice Swap]

    style PIPELINE fill:#0f172a,stroke:#334155,color:#e2e8f0
    style F fill:#1e3a5f,stroke:#3b82f6,color:#93c5fd
    style G fill:#1e3a5f,stroke:#3b82f6,color:#93c5fd
    style H fill:#450a0a,stroke:#ef4444,color:#fca5a5
```

**Key characteristics:**
- Single process, no queue, no external services (beyond APIs)
- Slots stored in `data/slots.json` — state persists across restarts
- VAD (Silero) waits **2.5 seconds** of silence before responding — comfortable for any speaking pace
- On escalation: `TTSUpdateSettingsFrame` swaps voice live + injects Supervisor system prompt

---

## 🚀 Production Architecture (v2 — Roadmap)

> Event-driven, multi-agent system with MCP tool layer, vector RAG, and LangGraph orchestration.

```mermaid
flowchart TD
    IN1([📞 Phone\nTwilio]) & IN2([🌐 Web\nWebRTC]) & IN3([📱 Mobile\nApp]) --> GW

    GW[🔀 API Gateway\nLoad Balancer] --> EB

    subgraph EB_BOX ["📨 Event Bus — Redis Streams / Kafka"]
        EB{{Events\ncall.started · user.spoke\ntool.requested · call.escalated}}
    end

    EB --> ORCH

    subgraph ORCH_BOX ["🧠 Orchestrator — LangGraph StateGraph"]
        ORCH[Intent Classifier\nClaude Sonnet] -->|intent=booking| BW
        ORCH -->|intent=faq| FW
        ORCH -->|urgency=emergency| SW
    end

    subgraph BW_BOX ["📅 Booking Worker"]
        BW[Claude Sonnet] --> MCP
        subgraph MCP ["🔌 MCP Server"]
            T1[get_availability]
            T2[book_slot]
            T3[get_patient_history]
            T4[check_insurance]
        end
        MCP --> DB[(🐘 PostgreSQL\ndoctors · slots\npatients)]
    end

    subgraph FW_BOX ["💬 FAQ Worker"]
        FW[Claude Haiku] --> RAG
        subgraph RAG ["🔍 RAG Pipeline"]
            R1[Query Embedding] --> R2[Vector Search]
            R2 --> R3[Rerank + Generate]
        end
        RAG --> VDB[(🗄️ Qdrant\nclinic docs\ninsurance · policies)]
    end

    subgraph SW_BOX ["🚨 Supervisor Worker"]
        SW[Escalation Handler] --> N1[Slack / SMS Alert]
        SW --> N2[CRM Ticket]
        SW --> N3[Voice Switch]
    end

    style EB_BOX fill:#0f172a,stroke:#6366f1,color:#e2e8f0
    style ORCH_BOX fill:#0f172a,stroke:#8b5cf6,color:#e2e8f0
    style BW_BOX fill:#0f172a,stroke:#3b82f6,color:#e2e8f0
    style FW_BOX fill:#0f172a,stroke:#10b981,color:#e2e8f0
    style SW_BOX fill:#0f172a,stroke:#ef4444,color:#e2e8f0
    style DB fill:#1e3a5f,stroke:#3b82f6,color:#93c5fd
    style VDB fill:#14532d,stroke:#22c55e,color:#86efac
```

### LangGraph State Flow

```mermaid
stateDiagram-v2
    [*] --> classify_intent

    classify_intent --> booking_node : intent = booking
    classify_intent --> faq_node : intent = faq
    classify_intent --> escalate_node : urgency = emergency

    faq_node --> booking_node : needs appointment
    faq_node --> [*] : answered

    booking_node --> mcp_tool_node : slot confirmed
    booking_node --> booking_node : needs more info

    mcp_tool_node --> [*] : booked ✓
    mcp_tool_node --> escalate_node : error / conflict

    escalate_node --> [*] : supervisor notified
```

---

## 📊 v1 vs v2 Comparison

| | v1 · Now | v2 · Production |
|---|:---:|:---:|
| **LLM agents** | 1 agent | Orchestrator + Workers |
| **Tool calling** | Direct Python fn | Via MCP Server |
| **Data store** | JSON file | PostgreSQL |
| **Knowledge base** | Keyword search | Vector RAG (Qdrant) |
| **Agent communication** | None | Event Bus (Redis/Kafka) |
| **Workflow engine** | Linear | LangGraph StateGraph |
| **Scalability** | Single process | Independent microservices |
| **Escalation** | Log to file | Webhook → CRM + Slack |

---

## ⚡ Quick Start

### Prerequisites

- Python **3.11+** · [uv](https://github.com/astral-sh/uv)
- [Anthropic API key](https://console.anthropic.com) — `ANTHROPIC_API_KEY`
- [Cartesia API key](https://cartesia.ai) — `CARTESIA_API_KEY`

### Setup

```bash
cd voice_agents/healthcare_contact_center
cp .env.example .env
# fill in your API keys in .env
uv sync
```

### Run

```bash
# WebRTC (default) — opens browser client
uv run python main.py

# Daily transport
uv run python main.py -t daily
```

Open **http://localhost:7860** in your browser and click Connect.

---

## 🎯 Try It

```
"I'd like to book an appointment with Dr. Patel."
"What insurance plans do you accept?"
"I want to speak to a manager."
"I'm having chest pain right now."   ← triggers emergency escalation
```

---

## 📁 Project Layout

```
healthcare_contact_center/
├── main.py               # Pipeline, VAD config, tool registration
├── personalities.py      # Prompts + Cartesia voice IDs (Aria & Marcus)
├── tools/
│   ├── appointments.py   # check_availability, book_appointment
│   ├── knowledge_base.py # lookup_faq — keyword search
│   └── escalation.py     # escalate_to_human — ticket + voice swap
└── data/
    ├── slots.json         # Mock appointment store (persisted on disk)
    └── faq.json           # Clinic policies and FAQ
```

---

## 🔧 Customizing

- **VAD sensitivity** — change `stop_secs` in `main.py` (default `2.5s`)
- **Voice tone** — change `emotion` in `GenerationConfig` (`content`, `happy`, `neutral`, ...)
- **LLM model** — swap `claude-haiku-4-5-20251001` for `claude-sonnet-4-6` for higher quality
- **Voices** — swap `FRONT_DESK_VOICE` / `SUPERVISOR_VOICE` with any [Cartesia voice ID](https://docs.cartesia.ai)
- **Real backend** — replace JSON ops in `tools/appointments.py` with your scheduling API

---

<div align="center">

Built with [Pipecat](https://pipecat.ai) · [Anthropic Claude](https://anthropic.com) · [Cartesia](https://cartesia.ai)

</div>
