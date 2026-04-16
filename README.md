<div align="center">

# 🌍 Nearby Chat

### **Location-based temporary chatrooms. Zero-friction anonymous. AI-native development.**

<p>
  <a href="#-quick-start"><img alt="Stage" src="https://img.shields.io/badge/stage-design_complete-blue?style=flat-square"></a>
  <a href="#-architecture"><img alt="Architecture" src="https://img.shields.io/badge/architecture-modular_monolith-brightgreen?style=flat-square"></a>
  <a href="./backend/CLAUDE.md"><img alt="Backend" src="https://img.shields.io/badge/backend-Spring_Boot_3_+_Netty-6DB33F?style=flat-square&logo=spring"></a>
  <a href="./frontend/CLAUDE.md"><img alt="Frontend" src="https://img.shields.io/badge/frontend-Vue_3_+_TS-42b883?style=flat-square&logo=vue.js"></a>
  <a href="./CLAUDE.md"><img alt="AI-driven" src="https://img.shields.io/badge/dev-AI_driven-7c3aed?style=flat-square"></a>
  <a href="#-license"><img alt="License" src="https://img.shields.io/badge/license-TBD-lightgrey?style=flat-square"></a>
</p>

<p>
  <a href="#-highlights">Highlights</a> ·
  <a href="#-quick-start">Quick Start</a> ·
  <a href="#-architecture">Architecture</a> ·
  <a href="#-ai-native-workflow">AI Workflow</a> ·
  <a href="#-documentation">Docs</a> ·
  <a href="#-roadmap">Roadmap</a> ·
  <a href="#-contributing">Contributing</a>
</p>

</div>

---

## 📖 What is this?

**Nearby Chat** is a geo-based temporary chatroom service. Users open the webpage—no signup, no login—and are automatically placed into a chatroom with people physically nearby. When the room empties, it's gone.

But this repo is **more than the product**. It's a **showcase of AI-native software engineering**: every file here is authored, maintained, and governed through a structured multi-agent workflow designed around the realities of large language models (stateless sessions, bounded context, bias toward confident hallucination).

```
       You (the user)
            │
            ▼
         CTO Agent  ◄──── Your single entry point
      ╱   │   │   │   ╲
     ▼    ▼   ▼   ▼    ▼
  backend front  ops  qa  code-reviewer
   -dev  -end
```

Five specialized agents, one orchestrator, strict contracts, immutable decision records, and a filesystem designed as **persistent memory** for the AIs that will lose their conversation history tomorrow.

---

## ✨ Highlights

### For the Product

- 🌐 **Zero-friction onboarding** — open and chat, no account
- 📍 **Location-driven matching** — one chatroom per geographic area at a time
- 💨 **Ephemeral by design** — empty rooms close instantly
- 🛡️ **Anonymous but accountable** — `deviceId + IP` ban scope, audit-grade message logs
- 🪪 **Persistent anonymous identity** — nicknames survive tab refreshes via localStorage

### For the Engineering

- 🧱 **Modular Monolith** — single deployable, module-clean code, zero-rewrite path to microservices
- 📜 **Contract-first** — `API_SPEC.md` is the single source of truth between frontend and backend
- 🧠 **ADR discipline** — every decision recorded, immutable, indexed
- 🧪 **Test pyramid** — unit tests owned by devs, integration tests owned by dedicated QA agent
- 🚧 **ArchUnit from day one** — architectural rules enforced at CI, not in review comments

### For the AI-Native Workflow

- 🤖 **Multi-agent orchestration** — CTO + 5 sub-agents, each with a bounded role and its own spec file
- 💾 **Filesystem as memory** — docs are persistent, conversations are not; design reflects this ruthlessly
- 📋 **Structured task briefings** — every sub-agent dispatch uses a required template (no ad-hoc prompts)
- 🎯 **Two-layer review** — automated rule-check by `code-reviewer`, business review by CTO; concerns stay separated
- 📚 **Self-learning via LESSONS.md** — recurring failures get codified into rules, not lost in chat history

---

## 🚀 Quick Start

### Option A · AI-driven (recommended)

```bash
# 1. Clone
git clone <repo> nearby-chat && cd nearby-chat

# 2. Launch Claude Code from the project root
claude

# 3. Claude Code auto-loads CLAUDE.md and becomes your CTO agent.
#    It will report current project state and ask what you want to do.
```

You'll then converse with the CTO agent in natural language:

> **You:** "Add rate limiting to the message endpoint, 10 msg/sec per user."
>
> **CTO:** "Before I dispatch this, 3 questions... [clarifies] ... Now dispatching `backend-dev` for T-042 (details in `docs/tasks/active/T-042.md`). Frontend parallel dispatch not needed—no UI change. I'll report back once review is complete."

### Option B · Human-driven (pending `ops` deliverables)

```bash
# Start dependencies (Redis / MongoDB / MySQL)
docker-compose up -d

# Backend
cd backend && ./mvnw spring-boot:run

# Frontend
cd frontend && npm install && npm run dev

# Integration tests
cd integration-tests && ./run.sh
```

> Detailed ops setup will be populated by the `ops` sub-agent once the corresponding task ships.

---

## 🏗 Architecture

### Runtime

```
┌──────────────────┐         ┌─────────────────────────────────────┐
│  Vue 3 Frontend  │ ──WS──► │  Netty Gateway  ──┐                │
│  (browser)       │ ◄─HTTP─ │                   │                │
└──────────────────┘         │                   ▼                │
                             │            ┌──────────────┐        │
                             │            │  Spring Boot │        │
                             │            │  (modules)   │        │
                             │            └──────┬───────┘        │
                             │                   │                │
                             │  ┌──────────┐ ┌───┴────┐ ┌───────┐ │
                             │  │  Redis   │ │MongoDB │ │ MySQL │ │
                             │  │ GEO·Sess │ │  msgs  │ │ users │ │
                             │  │ Pub/Sub  │ │        │ │       │ │
                             │  └──────────┘ └────────┘ └───────┘ │
                             └─────────────────────────────────────┘
```

### Module Layout (backend)

```
app → gateway | message | chatroom | auth | user
         │        │         │         │      │
         └────────┴────────┬┴─────────┴──────┘
                          im-core
                             │
                         location
                             │
                      infrastructure
                             │
                          common
```

- **Dependencies strictly flow downward.** Enforced by ArchUnit from Phase 1.
- Inter-module comm via `api/` package only. `internal/` is invisible across modules.
- See [`backend/CLAUDE.md`](./backend/CLAUDE.md) for the full spec.

---

## 🤖 AI-Native Workflow

### The Cast

| Agent | Role | Spec |
|---|---|---|
| 🎯 **CTO** (main) | Sole user interface; needs analysis, docs, dispatch, review, bridge | [`CLAUDE.md`](./CLAUDE.md) |
| 🟢 **backend-dev** | Java / Spring / Netty implementation, unit tests | [`backend/CLAUDE.md`](./backend/CLAUDE.md) |
| 🟣 **frontend-dev** | Vue 3 / TS, contract-bound consumer of `API_SPEC.md` | [`frontend/CLAUDE.md`](./frontend/CLAUDE.md) |
| 🟠 **ops** | Docker / CI / k8s / migrations / observability | [`ops/CLAUDE.md`](./ops/CLAUDE.md) |
| 🔵 **qa** | End-to-end integration tests, contract tests, bug reports with attribution | [`integration-tests/CLAUDE.md`](./integration-tests/CLAUDE.md) |
| ⚪ **code-reviewer** | Automated rule-check first-pass review; frees CTO for business review | [`code-review/CLAUDE.md`](./code-review/CLAUDE.md) |

### The Rules

- 🚫 **Sub-agents don't talk to each other.** All cross-team info routes through CTO.
- 🚫 **Sub-agents don't talk to the user.** The CTO is the only voice you hear.
- 🚫 **No one skips review.** Even trivial tasks pass through `code-reviewer` + CTO.
- ✅ **Frontend and backend tasks can run in parallel**—provided `API_SPEC.md` is fully finalized (see [`CLAUDE.md`](./CLAUDE.md) §4.3).
- ✅ **Bug attribution rebuttal is explicit and evidence-gated.** Sub-agents can push back on blame, but must prove their side is clean ([`CLAUDE.md`](./CLAUDE.md) §6.6).
- ✅ **Three-strike rule on bugs.** Same bug failing QA three times forces an escalation to the user ([`CLAUDE.md`](./CLAUDE.md) §6.7.2).

### Memory as a Filesystem

Claude sessions get reset, lose context, swap models. **We design for this:**

- Every decision → written into `docs/ADR/`
- Every task → a file in `docs/tasks/active/` that survives the session
- Every recurring failure → harvested into `docs/LESSONS.md`
- Every rule → lives in a `CLAUDE.md`, not in anyone's head

When a new Claude session starts, it reads the filesystem and reconstructs full context. No chat history required.

---

## 📚 Documentation

### Start Here

| Topic | Doc |
|---|---|
| 🗺 **Project overview** (you are here) | `README.md` |
| 🏃 **Just looking for current status** | [`docs/FEATURES.md`](./docs/FEATURES.md) |
| 🧾 **API contract (shared truth)** | [`docs/API_SPEC.md`](./docs/API_SPEC.md) |
| 📋 **Active tasks** | [`docs/TASKS.md`](./docs/TASKS.md) + [`docs/tasks/`](./docs/tasks/) |
| 🏛 **Why we built it this way** | [`docs/ADR/`](./docs/ADR/) |

### By Role

| Role | Reading Order |
|---|---|
| **Product / stakeholder** | `README.md` → `docs/FEATURES.md` |
| **Backend dev (human or AI)** | `backend/CLAUDE.md` → `docs/API_SPEC.md` → `docs/FEATURES.md` |
| **Frontend dev** | `frontend/CLAUDE.md` → `docs/API_SPEC.md` → `docs/FEATURES.md` |
| **Ops** | `ops/CLAUDE.md` → `backend/CLAUDE.md` §8–§9 |
| **QA** | `integration-tests/CLAUDE.md` → `docs/API_SPEC.md` |
| **CTO (Claude Code main agent)** | `CLAUDE.md` (auto-loaded) |

---

## 📁 Repository Layout

```
./
├── README.md                         ← you are here
├── CLAUDE.md                         ← CTO main agent spec
│
├── docs/
│   ├── FEATURES.md                   ← feature catalog + status
│   ├── API_SPEC.md                   ← frontend↔backend contract
│   ├── TASKS.md                      ← task index
│   ├── tasks/
│   │   ├── README.md
│   │   ├── TEMPLATE.md
│   │   ├── active/                   ← ongoing tasks (1 file each)
│   │   └── archive/YYYY-MM/          ← archived tasks
│   ├── ADR/                          ← architecture decisions (immutable)
│   │   ├── README.md
│   │   ├── TEMPLATE.md
│   │   └── ADR-XXXX-*.md
│   └── LESSONS.md                    ← hard-won lessons for future sessions
│
├── backend/                          ← Spring Boot modular monolith
│   └── CLAUDE.md                     ← backend dev spec
│
├── frontend/                         ← Vue 3 + TS
│   └── CLAUDE.md                     ← frontend dev spec
│
├── ops/
│   └── CLAUDE.md                     ← ops spec
│
├── integration-tests/
│   └── CLAUDE.md                     ← QA spec
│
├── code-review/
│   └── CLAUDE.md                     ← code-reviewer spec
│
└── .claude/
    └── agents/                       ← sub-agent registration entries
        ├── backend-dev.md
        ├── frontend-dev.md
        ├── ops.md
        ├── qa.md
        └── code-reviewer.md
```

Every sub-agent's **registration file** in `.claude/agents/` is a thin shell. The **full spec** lives at `<role>/CLAUDE.md`. Rule changes edit one place; the registration doesn't need to change.

---

## 🧪 Testing Strategy

| Layer | Owner | Where |
|---|---|---|
| Unit tests (backend) | `backend-dev` | `backend/**/src/test/java/` |
| Unit tests (frontend) | `frontend-dev` | `frontend/tests/` |
| Integration tests | `qa` | `integration-tests/` |
| Contract tests | `qa` (vs `API_SPEC.md`) | `integration-tests/contract/` |
| Regression tests | `qa` | `integration-tests/regression/` |
| Architecture tests | `backend-dev` (ArchUnit) | `backend/**/src/test/java/architecture/` |

A Feature is **not** considered `✅ DONE` until `qa` integration tests pass. Unit tests alone are insufficient signal.

---

## 🗺 Roadmap

### Current Phase: **Design Complete → Phase 1 (Backend skeleton + ArchUnit)**

- [ ] **Phase 1** · Parent POM, empty modules, dependency graph, ArchUnit baseline, `/actuator/health`
- [ ] **Phase 2** · Anonymous identity (`/session/init`, nickname, ban)
- [ ] **Phase 3** · `im-core` abstractions
- [ ] **Phase 4** · Location + chatroom (with concurrency tests)
- [ ] **Phase 5** · Netty WebSocket gateway
- [ ] **Phase 6** · Message pipeline (persist + broadcast)
- [ ] **Phase 7** · Polish, observability, full ArchUnit coverage
- [ ] **Phase A–E (frontend)** · Scaffold → contract layer → session init → chat core → polish

Full plan in [`backend/CLAUDE.md`](./backend/CLAUDE.md) §10 and [`frontend/CLAUDE.md`](./frontend/CLAUDE.md) §11.

---

## 🎯 Design Principles (one-liners)

1. **Docs are memory** — nothing lives "in the conversation"
2. **Contract is truth** — `API_SPEC.md` outranks anyone's opinion
3. **Single source of truth (SSOT)** — one fact, one document, everywhere else cite don't copy
4. **Distributed-ready from day one** — even when deploying as a single instance
5. **ADRs are immutable** — wrong decisions get *superseded*, not *deleted*
6. **Architecture rules run in CI** — not in code review comments
7. **Unit ≠ integration** — both required, owned by different roles
8. **Bug attribution needs evidence** — opinions don't count, proofs do

---

## 🤝 Contributing

Whether you're a human or an AI, the flow is the same:

```
 user request
      │
      ▼
  CTO clarifies ──────── clarification loop until unambiguous
      │
      ▼
  CTO writes to docs (FEATURES / API_SPEC / TASKS / ADR)
      │
      ▼
  CTO dispatches to sub-agent(s) via Task tool (parallelizable if rules met)
      │
      ▼
  sub-agent delivers ─── code-reviewer (layer 1) ─── CTO business review (layer 2)
      │                         │                          │
      │◄─── REWORK ─────────────┘ (up to 3 rounds) ◄──────┘
      │
      ▼
  qa integration tests ──── PASS → FEATURE DONE
      │
      └── FAIL → attribution → dispatch fix → repeat (3-strike escalation)
```

- ❌ **Do not edit code outside this flow.** If you must (hotfix), tell the CTO afterward so it can update the docs.
- ✅ **Report API_SPEC mismatches.** Don't mock around them.
- ✅ **Prove your rebuttals.** "I don't think this is my bug" needs evidence.

Full rules in [`CLAUDE.md`](./CLAUDE.md).

---

## 🧭 Status

Check `docs/FEATURES.md` for the feature catalog and `docs/TASKS.md` for what's being worked on right now.

---

## 📜 License

`TBD`

---

<div align="center">
<sub>
Built with discipline by a CTO agent and friends · Designed to outlive individual Claude sessions · Contributions follow the flow.
</sub>
</div>
