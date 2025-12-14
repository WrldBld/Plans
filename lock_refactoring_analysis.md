# Lock Pattern Refactoring Analysis

## Executive Summary

The WebSocket integration with application services faces a fundamental architectural mismatch:
- **Current pattern**: Async lock acquisition → release → LLM call → re-acquire
- **Service expectation**: Single mutable reference throughout operation

This document analyzes the data flow and presents the solution: **Phase 19: Unified Queue System**.

**See [19-queue-system.md](./19-queue-system.md) for the complete queue architecture plan.**

---

## Solution: Phase 19 Queue System

The lock pattern problem is solved by moving all operations through queues:

```
WebSocket Handler → Enqueue → Worker Processes → Result
     (fast)          (fast)      (can take time)
```

No locks are held across async boundaries because:
1. WebSocket handler just enqueues and returns
2. Worker tasks acquire locks only when needed (phased pattern)
3. Each queue controls its own concurrency

---

## Compatibility with Existing Queue Plans

### Three Queue Systems

WrldBldr's plans define **three distinct but related queue concepts**:

| Queue | Phase | Mode | Purpose | Status |
|-------|-------|------|---------|--------|
| **Generation Queue** | Phase 15 | Creator | Unify image + LLM suggestions | ~50% done (images only) |
| **Decision Queue** | Phase 16 | Director | DM approval for gameplay decisions | Not started |
| **Infrastructure Queues** | NEW | Both | Concurrency control (LLM, ComfyUI) | Proposed |

### Architecture Layers

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                           APPLICATION LAYER                                     │
│                                                                                 │
│  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐   │
│  │     DIRECTOR MODE (Phase 16)    │    │      CREATOR MODE (Phase 15)    │   │
│  │                                 │    │                                 │   │
│  │  ┌─────────────────────────┐   │    │   ┌─────────────────────────┐   │   │
│  │  │ DecisionQueueService    │   │    │   │ SuggestionQueueService  │   │   │
│  │  │ - pending decisions     │   │    │   │ - pending suggestions   │   │   │
│  │  │ - approve/reject/delay  │   │    │   │ - async processing      │   │   │
│  │  │ - history tracking      │   │    │   │ - WebSocket events      │   │   │
│  │  └───────────┬─────────────┘   │    │   └───────────┬─────────────┘   │   │
│  │              │                 │    │               │                 │   │
│  │  ┌───────────▼─────────────┐   │    │   ┌───────────▼─────────────┐   │   │
│  │  │ PlayerActionService     │   │    │   │ GenerationService       │   │   │
│  │  │ - gather_context()      │   │    │   │ - queue_batch()         │   │   │
│  │  │ - process_with_llm()    │   │    │   │ - process_batch()       │   │   │
│  │  │ - apply_result()        │   │    │   │                         │   │   │
│  │  └───────────┬─────────────┘   │    │   └───────────┬─────────────┘   │   │
│  │              │                 │    │               │                 │   │
│  │  ┌───────────▼─────────────┐   │    │               │                 │   │
│  │  │ ApprovalService         │   │    │               │                 │   │
│  │  │ - process_decision()    │   │    │               │                 │   │
│  │  └───────────┬─────────────┘   │    │               │                 │   │
│  │              │                 │    │               │                 │   │
│  └──────────────┼─────────────────┘    └───────────────┼─────────────────┘   │
│                 │                                      │                      │
├─────────────────┼──────────────────────────────────────┼──────────────────────┤
│                 │      INFRASTRUCTURE LAYER            │                      │
│                 │                                      │                      │
│  ┌──────────────▼──────────────────────────────────────▼──────────────────┐  │
│  │                     QUEUE INFRASTRUCTURE (NEW)                         │  │
│  │                                                                        │  │
│  │  ┌────────────────────────────┐    ┌────────────────────────────────┐ │  │
│  │  │ LLMQueue                   │    │ ComfyUIQueue                   │ │  │
│  │  │ (Concurrency Control)      │    │ (Concurrency Control)          │ │  │
│  │  │                            │    │                                │ │  │
│  │  │ - batch_size: configurable │    │ - batch_size: 1 (always)       │ │  │
│  │  │ - semaphore: Semaphore     │    │ - semaphore: Semaphore         │ │  │
│  │  │ - submit() → await result  │    │ - submit() → await result      │ │  │
│  │  │                            │    │                                │ │  │
│  │  └─────────────┬──────────────┘    └───────────────┬────────────────┘ │  │
│  │                │                                   │                   │  │
│  │                ▼                                   ▼                   │  │
│  │         OllamaClient                        ComfyUIClient              │  │
│  │                                                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

### How The Queues Relate

**Infrastructure Queues (LLMQueue, ComfyUIQueue)**:
- Control **concurrency** of external service calls
- BATCH_SIZE controls how many simultaneous calls
- Used **internally** by application services
- Do NOT provide user-visible queue UI

**Phase 15 Generation Queue (SuggestionQueueService)**:
- Provides **user-visible** queue in Creator Mode
- Unifies image generation and LLM suggestions
- Sends WebSocket events (SuggestionQueued, SuggestionComplete, etc.)
- Uses LLMQueue internally for actual Ollama calls

**Phase 16 Decision Queue (DecisionQueueService)**:
- Provides **DM-visible** queue in Director Mode
- Manages pending NPC responses, tool calls, challenge suggestions
- Uses LLMQueue internally for generating NPC responses
- **The current ApprovalService + PlayerActionService are a partial implementation of this!**

---

## Key Realization: Phase 16 Overlap

**The current `ApprovalService` and `PlayerActionService` are essentially an incomplete implementation of Phase 16's Decision Queue.**

| Phase 16 Feature | Current Implementation | Status |
|------------------|----------------------|--------|
| Pending decisions queue | `pending_approvals` in GameSession | Partial |
| Approve decision | `ApprovalDecision::Accept` | Done |
| Reject decision | `ApprovalDecision::Reject` | Done |
| Modify decision | `ApprovalDecision::AcceptWithModification` | Done |
| TakeOver | `ApprovalDecision::TakeOver` | Done |
| Delay decision | Not implemented | Missing |
| Decision urgency | Not implemented | Missing |
| Decision history | Not implemented | Missing |
| Undo recent decision | Not implemented | Missing |
| Filter by type | Not implemented | Missing |
| Batch approve | Not implemented | Missing |
| WebSocket events | `ApprovalRequired` only | Partial |

---

## Current Architecture Analysis

### Data Structures

```
AppState
└── sessions: RwLock<SessionManager>
    ├── sessions: HashMap<SessionId, GameSession>
    ├── client_sessions: HashMap<ClientId, SessionId>
    └── world_sessions: HashMap<WorldId, SessionId>

GameSession
├── participants: HashMap<ClientId, SessionParticipant>
│   └── sender: mpsc::UnboundedSender<ServerMessage>
├── pending_approvals: HashMap<String, PendingApproval>
├── conversation_history: Vec<ConversationTurn>
└── world_snapshot: Arc<WorldSnapshot>
```

### Current Lock Patterns

#### Pattern 1: PlayerAction Flow (websocket.rs:394-477)

```rust
// Step 1: Read lock to get session info and send notifications
let sessions = state.sessions.read().await;
let session_id = sessions.get_client_session(client_id)...;
session.send_to_dm(&processing_msg);
drop(sessions);  // Implicit drop when calling async

// Step 2: Process with LLM (NO LOCK - this is slow!)
process_player_action_with_llm(...).await;  // 1-30+ seconds
```

#### Pattern 2: LLM Processing (websocket.rs:119-335)

```rust
// Phase A: Read lock to gather context
let (session_data, npc_name) = {
    let sessions = state.sessions.read().await;
    let scene_context = build_context(&session.world_snapshot);
    (prompt_request, responding_character.name)
};  // Read lock RELEASED

// Phase B: Call LLM (no lock, takes seconds)
let response = llm_service.generate_npc_response(session_data).await;

// Phase C: Write lock to store result
let mut sessions = state.sessions.write().await;
session.add_pending_approval(pending);
session.send_to_dm(&approval_msg);
```

---

## The Problem

The new `PlayerActionService.process_action()` expects a single mutable reference:

```rust
pub async fn process_action<S: SessionManagementPort>(
    &self,
    session: &mut S,  // Expects single mutable reference
    ...
) -> Result<PlayerActionResult, PlayerActionError>
```

This is incompatible with the async lock pattern because:
1. `RwLockReadGuard` cannot give `&mut`
2. `RwLockWriteGuard` cannot be held across `await` points
3. Blocking all sessions during LLM calls is unacceptable

---

## Recommended Solution: Phased Operations + Infrastructure Queues

### Step 1: Create Infrastructure Queues

```rust
// infrastructure/queues/llm_queue.rs
pub struct LLMQueue {
    tx: mpsc::Sender<LLMJob>,
    semaphore: Arc<Semaphore>,
}

pub struct LLMJob {
    pub request: LLMRequest,
    pub response_tx: oneshot::Sender<Result<LLMResponse, LLMError>>,
}

impl LLMQueue {
    pub fn new(ollama_client: OllamaClient, batch_size: usize) -> Self {
        let (tx, rx) = mpsc::channel(100);
        let semaphore = Arc::new(Semaphore::new(batch_size.max(1)));

        tokio::spawn(Self::run_workers(rx, ollama_client, semaphore.clone()));

        Self { tx, semaphore }
    }

    /// Submit job and wait for result (respects batch_size concurrency)
    pub async fn submit(&self, request: LLMRequest) -> Result<LLMResponse, LLMError> {
        let (response_tx, response_rx) = oneshot::channel();
        self.tx.send(LLMJob { request, response_tx }).await?;
        response_rx.await?
    }

    async fn run_workers(
        mut rx: mpsc::Receiver<LLMJob>,
        ollama_client: OllamaClient,
        semaphore: Arc<Semaphore>,
    ) {
        while let Some(job) = rx.recv().await {
            let client = ollama_client.clone();
            let sem = semaphore.clone();

            tokio::spawn(async move {
                let _permit = sem.acquire().await.unwrap();
                let result = client.generate(&job.request).await;
                let _ = job.response_tx.send(result);
            });
        }
    }
}

// Similar for ComfyUIQueue with batch_size=1
```

### Step 2: Split Session Ports

```rust
// application/ports/outbound/session_query_port.rs
pub trait SessionQueryPort: Send + Sync {
    fn get_client_session(&self, client_id: &str) -> Option<SessionId>;
    fn is_client_dm(&self, client_id: &str) -> bool;
    fn get_pending_approval(&self, session_id: SessionId, request_id: &str)
        -> Option<PendingApprovalInfo>;
    fn get_session_world_context(&self, session_id: SessionId)
        -> Option<SessionWorldContext>;
    fn session_has_dm(&self, session_id: SessionId) -> bool;
}

// application/ports/outbound/session_command_port.rs
pub trait SessionCommandPort: Send + Sync {
    fn add_pending_approval(&mut self, session_id: SessionId, approval: PendingApprovalInfo)
        -> Result<(), SessionManagementError>;
    fn remove_pending_approval(&mut self, session_id: SessionId, request_id: &str)
        -> Result<(), SessionManagementError>;
    fn add_to_conversation_history(&mut self, session_id: SessionId, speaker: &str, text: &str)
        -> Result<(), SessionManagementError>;

    // Broadcast methods (only need &self internally, but part of command port)
    fn broadcast_to_players(&self, session_id: SessionId, message: &BroadcastMessage)
        -> Result<(), SessionManagementError>;
    fn send_to_dm(&self, session_id: SessionId, message: &BroadcastMessage)
        -> Result<(), SessionManagementError>;
}
```

### Step 3: Phased Service Operations

```rust
// Context/Result types
pub struct PlayerActionContext {
    pub session_id: SessionId,
    pub action_id: String,
    pub world_context: SessionWorldContext,
    pub responding_character: CharacterContextInfo,
    pub action_type: String,
    pub target: Option<String>,
    pub dialogue: Option<String>,
    pub has_dm: bool,
}

pub struct PlayerActionResult {
    pub session_id: SessionId,
    pub pending_approval: PendingApprovalInfo,
    pub dm_notification: BroadcastMessage,
}

impl PlayerActionService {
    /// Phase 1: Gather context (caller holds read lock)
    pub fn gather_context<Q: SessionQueryPort>(
        &self,
        session: &Q,
        session_id: SessionId,
        action_id: String,
        action_type: String,
        target: Option<String>,
        dialogue: Option<String>,
    ) -> Result<PlayerActionContext, PlayerActionError>;

    /// Phase 2: Process with LLM (no lock, uses LLMQueue)
    pub async fn process_with_llm(
        &self,
        context: PlayerActionContext,
    ) -> Result<PlayerActionResult, PlayerActionError>;

    /// Phase 3: Apply result (caller holds write lock)
    pub fn apply_result<C: SessionCommandPort>(
        &self,
        session: &mut C,
        result: PlayerActionResult,
    ) -> Result<(), PlayerActionError>;
}
```

### Step 4: WebSocket Handler Uses Phases

```rust
ClientMessage::PlayerAction { action_type, target, dialogue } => {
    let action_id = ActionId::new().to_string();

    // Phase 1: Gather context (read lock)
    let context = {
        let sessions = state.sessions.read().await;
        let session_id = sessions.get_client_session(&client_id.to_string())
            .ok_or(Error::NotInSession)?;

        // Notify DM that processing started
        if sessions.session_has_dm(session_id) {
            sessions.send_to_dm(session_id, &processing_notification)?;
        }

        state.player_action_service.gather_context(
            &*sessions,
            session_id,
            action_id,
            action_type,
            target,
            dialogue,
        )?
    };  // Read lock released

    if !context.has_dm {
        return None;  // No DM, skip LLM processing
    }

    // Phase 2: Process with LLM (no lock, goes through LLMQueue)
    let result = state.player_action_service.process_with_llm(context).await?;

    // Phase 3: Apply result (write lock)
    {
        let mut sessions = state.sessions.write().await;
        state.player_action_service.apply_result(&mut *sessions, result)?;
    }  // Write lock released

    None
}
```

---

## Integration with Phase 15 & 16

### Phase 15: SuggestionQueueService Uses LLMQueue

```rust
impl SuggestionQueueService {
    pub async fn enqueue(&self, request: SuggestionRequest) -> String {
        let request_id = Uuid::new_v4().to_string();

        // Send SuggestionQueued event
        self.broadcast_event(SuggestionQueued {
            request_id: request_id.clone(),
            field_type: request.field_type.clone(),
        });

        // Submit to LLMQueue (respects concurrency limits)
        let result = self.llm_queue.submit(request.to_llm_request()).await;

        // Send result event
        match result {
            Ok(response) => {
                self.broadcast_event(SuggestionComplete {
                    request_id,
                    suggestions: response.suggestions,
                });
            }
            Err(e) => {
                self.broadcast_event(SuggestionFailed {
                    request_id,
                    error: e.to_string(),
                });
            }
        }

        request_id
    }
}
```

### Phase 16: DecisionQueueService Uses PlayerActionService

```rust
impl DecisionQueueService {
    /// Add a new pending decision from player action
    pub async fn add_from_player_action<Q: SessionQueryPort, C: SessionCommandPort>(
        &self,
        query_session: &Q,
        command_session: &mut C,
        session_id: SessionId,
        action: PlayerAction,
    ) -> Result<DecisionId, Error> {
        // Phase 1: Gather context
        let context = self.player_action_service.gather_context(
            query_session,
            session_id,
            action.id,
            action.action_type,
            action.target,
            action.dialogue,
        )?;

        // Phase 2: Process with LLM (through LLMQueue)
        let result = self.player_action_service.process_with_llm(context).await?;

        // Phase 3: Add to pending decisions
        let decision = PendingDecision {
            id: DecisionId::new(),
            session_id,
            decision_type: DecisionType::NpcResponse {
                character_id: result.pending_approval.npc_name.clone(),
                dialogue: result.pending_approval.proposed_dialogue.clone(),
                reasoning: result.pending_approval.internal_reasoning.clone(),
                attached_tools: result.pending_approval.proposed_tools.clone(),
            },
            urgency: DecisionUrgency::AwaitingPlayer,
            status: DecisionStatus::Pending,
            created_at: Utc::now(),
        };

        command_session.add_pending_decision(session_id, decision.clone())?;

        // Broadcast DecisionPending event
        self.broadcast_event(DecisionPending { decision: decision.clone() });

        Ok(decision.id)
    }
}
```

---

## BATCH_SIZE Configuration

```rust
// config.rs
pub struct QueueConfig {
    /// Max concurrent LLM calls. 0 = unlimited, 1 = sequential (default)
    pub llm_batch_size: usize,
    /// Max concurrent ComfyUI generations. Always 1.
    pub comfyui_batch_size: usize,
}

impl Default for QueueConfig {
    fn default() -> Self {
        Self {
            llm_batch_size: 1,   // Single Ollama instance
            comfyui_batch_size: 1,  // ComfyUI queues internally
        }
    }
}
```

| BATCH_SIZE | Behavior | Use Case |
|------------|----------|----------|
| 0 | Unlimited concurrency | Development/testing |
| 1 | Sequential (default) | Single GPU/Ollama |
| N | Up to N concurrent | Multiple GPUs or API |

---

## Implementation Plan

### Phase A: Infrastructure Queues (Foundation)

- [ ] **A.1** Create `LLMQueue` infrastructure
  - File: `Engine/src/infrastructure/queues/llm_queue.rs`
  - Semaphore-based concurrency control
  - Submit/await pattern

- [ ] **A.2** Create `ComfyUIQueue` infrastructure
  - File: `Engine/src/infrastructure/queues/comfyui_queue.rs`
  - Fixed batch_size=1 (ComfyUI queues internally)

- [ ] **A.3** Add QueueConfig to AppConfig
  - File: `Engine/src/infrastructure/config.rs`
  - Environment variable support

- [ ] **A.4** Wire queues into AppState
  - File: `Engine/src/infrastructure/state.rs`

### Phase B: Split Ports & Phased Services

- [ ] **B.1** Create `SessionQueryPort` trait
  - File: `Engine/src/application/ports/outbound/session_query_port.rs`

- [ ] **B.2** Create `SessionCommandPort` trait
  - File: `Engine/src/application/ports/outbound/session_command_port.rs`

- [ ] **B.3** Define context/result types
  - `PlayerActionContext`, `PlayerActionResult`
  - `ApprovalContext`, `ApprovalOutcome`

- [ ] **B.4** Refactor PlayerActionService to phased operations
  - `gather_context()`, `process_with_llm()`, `apply_result()`

- [ ] **B.5** Update ApprovalService to return ApprovalOutcome

### Phase C: WebSocket Integration

- [ ] **C.1** Update PlayerAction handler to use phased pattern
- [ ] **C.2** Update ApprovalDecision handler
- [ ] **C.3** Add LLMQueue usage to services

### Phase D: Align with Phase 16 (Optional - Can Be Deferred)

- [ ] **D.1** Rename to DecisionQueueService
- [ ] **D.2** Add DecisionUrgency, DecisionStatus
- [ ] **D.3** Add delay functionality
- [ ] **D.4** Add history tracking
- [ ] **D.5** Add WebSocket events (DecisionPending, DecisionUpdated)

---

## Decision Point: Phase 16 Alignment

**Question**: Should we align the current services with Phase 16 naming/structure now, or keep them minimal and do Phase 16 as a separate enhancement?

### Option A: Align Now
- Rename ApprovalService → part of DecisionQueueService
- Add missing Phase 16 features
- More work now, but cleaner architecture

### Option B: Keep Minimal
- Current services work as-is
- Phase 16 built on top later
- Less work now, potential refactoring later

### Option C: Hybrid
- Do lock refactoring + infrastructure queues now
- Keep current naming
- Phase 16 features added incrementally

**Recommendation**: Option C (Hybrid) - Get the infrastructure right, then add features incrementally.

---

## Testing Strategy

### Unit Tests
- LLMQueue concurrency (verify semaphore limits)
- Each service phase independently
- Context building logic
- Result application logic

### Integration Tests
- Full flow with real lock acquisition
- Verify no deadlocks
- Test concurrent operations

### Load Tests
- Multiple concurrent player actions
- Verify queue backpressure works
- Verify batch_size is respected
