# WrldBldr Roadmap

This document tracks all remaining work identified during the codebase analysis. Sub-agents should use this to understand context, track progress, and coordinate implementation.

**Last Updated**: 2025-12-15
**Overall Progress**: Core gameplay complete; Queue System complete; **Code Review Fixes Complete**; **Suggestion Queue Integration Complete**; **Phase 20 (Unified Generation Queue UI) PARTIALLY COMPLETE; Phase 16 (Decision Queue) READY**

**Current Priority**: **Phase 20 (Unified Generation Queue UI)** and **Phase 16 (Decision Queue)** - Now unblocked by Phase 19 and code review fixes

**Key Updates** (2025-12-15):
- ✅ Phase 19 fully implemented and code review issues resolved
- ✅ All queue services operational with background workers
- ✅ WebSocket handler refactored to thin adapter (no application logic)
- ✅ Health check endpoint and cleanup worker implemented
- ✅ Code cleanup complete - both Engine and Player compile with no errors
- ✅ Fixed Player clippy error in challenge_library.rs
- ✅ **Code Review Fixes Complete** (2025-12-15):
  - ✅ GenerationService wired to AppState and operational
  - ✅ Generation event broadcasting worker implemented
  - ✅ ChallengeSuggestionDecision handler completed
  - ✅ NarrativeEventSuggestionDecision handler completed
  - ✅ All WebSocket message types synchronized
  - ✅ Both Engine and Player compile successfully
- ✅ **Event Bus Architecture Implemented** (2025-12-15):
  - ✅ SQLite-backed event bus with pub/sub pattern
  - ✅ AppEvent DTO for cross-cutting system events
  - ✅ All producers wired (Story, Narrative, Challenge, Generation, Suggestions)
  - ✅ WebSocketEventSubscriber for real-time client updates
  - ✅ Ready for future Redis backend and advanced consumers
- 🆕 Phase 20 (Unified Generation Queue UI) now UNBLOCKED - ready to implement
- 🆕 Phase 16 (Decision Queue) now UNBLOCKED - ready to implement

---

## Implementation Order (Critical Path)

### ✅ COMPLETED: Phase 19 (Queue System)

Phase 19 is now **COMPLETE** (2025-12-15). All queue infrastructure is operational.

### Immediate Priority: Phase 20 + Phase 16 (Now Unblocked)

With Phase 19 complete, these phases can proceed in parallel:

**Phase 20: Unified Generation Queue UI** (formerly "Phase 15 Generation Queue")
- Builds on LLMReasoningQueue + AssetGenerationQueue (now available)
- Unify image + suggestion queues in Creator Mode sidebar
- Frontend queue UI with real-time WebSocket events
- See Section 2.1b for details

**Phase 16: Director Decision Queue**
- Builds on DMApprovalQueue (now available)
- Frontend decision queue UI in Director Mode sidebar
- History, filtering, and keyboard shortcuts
- See Section 2.1c for details

**Optional Parallel Work**:
- **Phase 18B** (Generation Event Wiring) - Critical for real-time asset generation feedback
- **Phase 17G** (Event Chain Visualizer) - Polish for Story Arc feature

**Completed Dependencies**:
- ✅ Phase 19 (Queue System) - All queue infrastructure operational
- ✅ Phase 18.2.3 (WebSocket refactoring) - Completed via Phase 19C

**Reference Documents**:
- [19-queue-system.md](./19-queue-system.md) - Complete queue system plan
- [queue_system_architecture_validation.md](./queue_system_architecture_validation.md) - Architecture validation
- [lock_refactoring_analysis.md](./lock_refactoring_analysis.md) - Lock pattern analysis

---

## How to Use This Document

1. **Before starting work**: Read the relevant section to understand context and dependencies
2. **During implementation**: Mark tasks as `[🔄]` when in progress
3. **After completion**: Mark tasks as `[✅]` and add completion date
4. **If blocked**: Mark as `[⛔]` and document the blocker

**Status Legend**:
- `[ ]` Not started
- `[🔄]` In progress
- `[✅]` Complete
- `[⛔]` Blocked
- `[⏭️]` Skipped/Deferred

---

## Priority Tiers

### Tier 1: Critical Path (Blocks Core Gameplay)
- LLM Integration (Engine)
- DM Approval Flow (Engine + Player)
- Player Action Sending (Player)

### Tier 2: Feature Completeness
- Creator Mode API Integration (Player)
- Workflow Config Persistence (Player)
- Spectator View (Player)

### Tier 3: Architecture & Quality
- DDD Patterns (Engine)
- Error Handling (Both)
- Testing (Both)
- Security (Engine)

### Tier 4: Session & World Management
- Queue System Architecture (Engine) - Phase 19 ✅ **COMPLETE**
- World Selection Flow (Player) - Phase 13 ✅
- Rule Systems & Challenges (Both) - Phase 14 ✅
- Routing & Navigation (Player) - Phase 15 ✅
- Story Arc (Both) - Phase 17 (partial - 17G remaining)
- ComfyUI Enhancements (Both) - Phase 18 (partial - all sub-phases pending)
- **Unified Generation Queue UI (Both) - Phase 20** ⚠️ **READY TO IMPLEMENT**
- **Director Decision Queue (Both) - Phase 16** ⚠️ **READY TO IMPLEMENT**

### Tier 5: Future Features
- Tactical Combat (Both)
- Audio System (Player)
- Save/Load System (Both)

---

## TIER 1: CRITICAL PATH

### 1.1 LLM Integration (Engine)

**Location**: `Engine/src/infrastructure/websocket.rs`, `Engine/src/application/services/llm_service.rs`

**Context**: The WebSocket handler receives `PlayerAction` messages but has TODOs for LLM processing. The `LlmService` and `OllamaClient` exist but aren't wired into the action flow.

**Current State**:
```rust
// In websocket.rs around line 200+
// TODO: Process action through LLM and send ApprovalRequired to DM
// TODO: Load scene from database and broadcast SceneUpdate
```

**Tasks**:

- [✅] **1.1.1** Wire PlayerAction to LLM processing (2025-12-11)
  - File: `Engine/src/infrastructure/websocket.rs`
  - Implemented `process_player_action_with_llm()` function
  - Extracts scene/character data, builds GamePromptRequest, calls LLM
  - Sends ApprovalRequired to DM only

- [✅] **1.1.2** Implement prompt building in LlmService (2025-12-11)
  - File: `Engine/src/application/services/llm_service.rs`
  - Added `build_system_prompt_with_notes()` with DirectorialNotes integration
  - Added `generate_npc_response_with_direction()` method
  - Includes tone/pacing guidance, character wants, relationship context

- [✅] **1.1.3** Implement tool calling support (2025-12-11)
  - File: `Engine/src/domain/value_objects/game_tools.rs` (new)
  - Defined `GameTool` enum: GiveItem, RevealInfo, ChangeRelationship, TriggerEvent
  - Added tool parsing and validation in llm_service.rs
  - 19 unit tests for tool handling

- [✅] **1.1.4** Send ApprovalRequired to DM (2025-12-11)
  - File: `Engine/src/infrastructure/websocket.rs`
  - Integrated with PlayerAction handler
  - Tracks pending approvals in session
  - Sends LLMProcessing notification to DM

- [✅] **1.1.5** Handle conversation history (2025-12-11)
  - File: `Engine/src/infrastructure/session.rs`
  - Added `ConversationTurn` struct with speaker, content, timestamp
  - Added history management methods with 30-turn limit
  - 9 unit tests for conversation history
  - Include in LLM context

**Dependencies**: None (can start immediately)

**Acceptance Criteria**:
- Player sends action → Engine calls LLM → DM receives approval popup
- LLM response includes contextual dialogue based on scene
- Tool calls are parsed and included in approval request

---

### 1.2 DM Approval Flow (Engine)

**Location**: `Engine/src/infrastructure/websocket.rs`

**Context**: The message types for `ApprovalDecision` exist but handling is incomplete.

**Tasks**:

- [✅] **1.2.1** Handle ApprovalDecision::Accept (2025-12-11)
  - File: `Engine/src/infrastructure/websocket.rs`
  - Broadcasts DialogueResponse to all players
  - Stores NPC response in conversation history
  - Executes approved tool calls

- [✅] **1.2.2** Handle ApprovalDecision::AcceptWithModification (2025-12-11)
  - File: `Engine/src/infrastructure/websocket.rs`
  - Uses DM's modified dialogue
  - Filters tool calls to approved_tools list only

- [✅] **1.2.3** Handle ApprovalDecision::Reject (2025-12-11)
  - File: `Engine/src/infrastructure/websocket.rs`
  - Extracts feedback, increments retry_count
  - Enforces 3-retry maximum
  - Re-sends LLMProcessing notification

- [✅] **1.2.4** Handle ApprovalDecision::TakeOver (2025-12-11)
  - File: `Engine/src/infrastructure/websocket.rs`
  - Uses DM's custom response
  - No tool execution, full DM control

- [✅] **1.2.5** Implement tool call execution (2025-12-11)
  - File: `Engine/src/application/services/tool_execution_service.rs` (new)
  - `ToolExecutionService` with execute_tool() method
  - Implements GiveItem, RevealInfo, ChangeRelationship, TriggerEvent
  - Returns `ToolExecutionResult` with `StateChange` enum for broadcasting
  - 9 unit tests

**Dependencies**: 1.1 (LLM Integration)

**Acceptance Criteria**:
- DM can accept/modify/reject LLM responses
- Approved responses are broadcast to all players
- Tool calls modify game state correctly
- Rejection triggers LLM retry with feedback

---

### 1.3 Player Action Sending (Player)

**Location**: `Player/src/presentation/views/pc_view.rs`, `Player/src/application/services/action_service.rs`

**Context**: The `ActionService` has methods for sending actions, but the PC view may not be fully wiring user interactions to these methods.

**Tasks**:

- [✅] **1.3.1** Verify dialogue choice sends PlayerAction (2025-12-11)
  - File: `Player/src/presentation/views/pc_view.rs`
  - Already implemented via handle_choice_selected()
  - Sets awaiting_input=false, shows loading state

- [✅] **1.3.2** Verify custom input sends PlayerAction (2025-12-11)
  - File: `Player/src/presentation/views/pc_view.rs`
  - Already implemented via handle_custom_input()
  - Clears input after sending

- [✅] **1.3.3** Implement interaction button actions (2025-12-11)
  - File: `Player/src/presentation/views/pc_view.rs`
  - handle_interaction() maps all types: talk, examine, travel, use
  - Unknown types fallback to custom_targeted()

- [✅] **1.3.4** Handle DialogueResponse from server (2025-12-11)
  - File: `Player/src/application/services/session_service.rs`
  - DialogueResponse handler calls dialogue_state.apply_dialogue()
  - Trigger typewriter animation
  - Display choices when animation completes

- [✅] **1.3.5** Handle LLMProcessing indicator (2025-12-11)
  - File: `Player/src/presentation/views/pc_view.rs`
  - Added `is_llm_processing` signal to DialogueState
  - Shows "NPC is thinking..." with animated ellipsis in DialogueBox
  - Disables all action buttons (0.5 opacity) via ActionPanel disabled prop
  - Clears when DialogueResponse arrives

**Dependencies**: 1.1, 1.2 (needs Engine to respond) ✅ Complete

**Acceptance Criteria**:
- Player clicks choice → action sent → response received → dialogue displays
- Custom input works correctly
- All interaction types send appropriate actions
- Loading states display correctly

---

## TIER 2: FEATURE COMPLETENESS

### 2.1 Creator Mode API Integration (Player)

**Location**: `Player/src/presentation/components/creator/`

**Context**: Creator mode UI components exist but have TODO comments for API calls. They display static/mock data instead of fetching from Engine.

**Tasks**:

- [✅] **2.1.1** Wire EntityBrowser to fetch entities (2025-12-11)
  - File: `Player/src/presentation/components/creator/entity_browser.rs`
  - CharacterList and LocationList fetch from REST API
  - Loading and error states implemented

- [✅] **2.1.2** Wire CharacterForm to save characters (2025-12-11)
  - File: `Player/src/presentation/components/creator/character_form.rs`
  - POST for create, PUT for update
  - Form validation and success/error feedback

- [✅] **2.1.3** Wire LocationForm to save locations (2025-12-11)
  - File: `Player/src/presentation/components/creator/location_form.rs`
  - Parent location dropdown populated from API
  - POST for create, PUT for update

- [✅] **2.1.4** Wire AssetGallery to manage assets (2025-12-11)
  - File: `Player/src/presentation/components/creator/asset_gallery.rs`
  - Fetch, activate, delete assets via REST API
  - Context menu for asset actions

- [✅] **2.1.5** Wire SuggestionButton to request suggestions (2025-12-11)
  - File: `Player/src/presentation/components/creator/suggestion_button.rs`
  - Desktop (reqwest) and WASM (gloo_net) implementations
  - All suggestion types supported

- [✅] **2.1.6** Wire GenerationModal to queue generation (2025-12-11)
  - File: `Player/src/presentation/components/creator/asset_gallery.rs`
  - POST /api/assets/generate with prompt, count, workflow
  - Integrated into AssetGallery component

**Dependencies**: Engine REST API (already complete)

**Acceptance Criteria**:
- Entity browser shows real data from Engine
- Creating/editing entities persists to database
- Asset gallery displays and manages real assets
- AI suggestions work and populate form fields
- Generation queue shows real-time progress

---

### 2.1b Unified Generation Queue UI - Creator Mode (Player + Engine) - **Phase 20**

**Plan**: [20-unified-generation-queue.md](./20-unified-generation-queue.md)  
**Analysis**: [unified_generation_queue_analysis.md](./unified_generation_queue_analysis.md)

**Summary**: Unify image generation and LLM suggestion requests into a single queue system in Creator Mode, with WebSocket events for real-time progress visibility.

**Location**: Creator Mode sidebar

**Status**: 🔄 **PARTIALLY COMPLETE** (as of 2025-12-15)  

**Current State**: 
- ✅ Image generation uses queue system (GenerationState, WebSocket events)
- ✅ LLMReasoningQueue infrastructure available (via Phase 19)
- ✅ AssetGenerationQueue infrastructure available (via Phase 19)
- ✅ LLM suggestions are routed through `LLMReasoningQueue` (no longer synchronous HTTP)
- ✅ Suggestion processing is handled in `LLMQueueService` and emits `GenerationEvent::Suggestion*`
- ✅ WebSocket events for suggestions (`SuggestionQueued`, `SuggestionProgress`, `SuggestionComplete`, `SuggestionFailed`) are implemented
- ✅ Player tracks both image batches and suggestion tasks in `GenerationState`
- ✅ Unified Generation Queue UI shows **both** images and suggestions in Creator Mode sidebar
- ❌ Advanced queue UX (ComfyUI health banner, retry/cancel/clear UI, detailed error expansion) still pending

**Dependencies**: 
- ✅ Phase 19 complete - LLMReasoningQueue and AssetGenerationQueue now available
- ✅ Suggestion queue integration implemented (see `suggestion_queue_oversight.md` and `20-unified-generation-queue.md`)

**Tasks**: 
1. **Phase 20B – Player UX polish (remaining)**  
   - Enhance `GenerationQueuePanel` with:
     - Error details expansion
     - Retry/cancel/clear actions for failed/ready batches
     - Visual distinction and filtering between images vs suggestions
   - Add queue badges / counters in Creator Mode header
2. **Phase 20C – Notifications & persistence**  
   - Toasts for completed suggestions/batches
   - Ensure queue state persistence across Creator Mode navigation
3. Keep `20-unified-generation-queue.md` as the detailed design reference

---

### 2.1c Director Decision Queue - Director Mode (Player + Engine) - **Phase 16**

**Plan**: [16-director-decision-queue.md](./16-director-decision-queue.md)  
**Analysis**: [director_decision_queue_analysis.md](./director_decision_queue_analysis.md)

**Summary**: Provide DM visibility and control over all AI gameplay decisions (NPC responses, tool usage, challenge suggestions) before they affect players.

**Location**: Director Mode sidebar

**Status**: [ ] **READY TO IMPLEMENT** - Phase 19 complete, all dependencies met

**Current State**:
- ✅ ApprovalService and PlayerActionService exist (partial implementation)
- ✅ Basic approval workflow (Accept/Modify/Reject/TakeOver) works
- ✅ DMApprovalQueue infrastructure available (via Phase 19)
- ✅ DMApprovalQueueService with history, delay, expiration features
- ❌ No decision queue UI in Director Mode
- ❌ No WebSocket events for decision queue status

**Dependencies**: 
- ✅ Phase 19 complete - DMApprovalQueue now available
- ✅ Services already refactored to use queues

**Key Features** (to be implemented):
- Real-time queue of pending AI decisions
- Approve/Reject/Modify/Delay actions
- Type filtering (Dialogue, Tools, Challenges, Transitions)
- Decision history with undo capability
- Keyboard shortcuts for fast approval workflow

**Tasks**: See plan document for detailed breakdown (16A: Engine enhancements, 16B: Player UI, 16C: Integration)

---

### 2.1d Unified Queue System - Engine Architecture (Phase 19) ✅ **COMPLETE**

**Plan**: [19-queue-system.md](./19-queue-system.md)  
**Architecture Validation**: [queue_system_architecture_validation.md](./queue_system_architecture_validation.md)  
**Lock Pattern Analysis**: [lock_refactoring_analysis.md](./lock_refactoring_analysis.md)

**Summary**: Comprehensive queue system for all game actions, AI processing, and DM approvals. Provides crash recovery, audit trails, and foundation for scaling.

**Status**: ✅ **COMPLETE** (2025-12-15)

**Completed Features**:
- ✅ Solved critical lock pattern problem
- ✅ Enabled Phase 20 (Generation Queue) and Phase 16 (Decision Queue)
- ✅ Crash recovery via persistent SQLite queues
- ✅ Foundation for scaling to multi-server deployments

**Five Queues** (All Implemented):
| Queue | Purpose | Persistence | Status |
|-------|---------|-------------|--------|
| `PlayerActionQueue` | Player actions awaiting processing | Yes | ✅ Complete |
| `DMActionQueue` | DM actions awaiting processing | Yes | ✅ Complete |
| `LLMReasoningQueue` | Ollama requests (BATCH_SIZE controlled) | Optional | ✅ Complete |
| `AssetGenerationQueue` | ComfyUI requests (BATCH_SIZE=1) | Optional | ✅ Complete |
| `DMApprovalQueue` | Decisions awaiting DM approval | Yes | ✅ Complete |

**Key Features Implemented**:
- ✅ Hexagonal architecture with pluggable backends (InMemory, SQLite)
- ✅ Crash recovery via persistent queues
- ✅ Concurrency control (semaphores) for AI services
- ✅ History tracking for approvals
- ✅ WebSocket handlers enqueue and return immediately (no locks held)
- ✅ Background workers for all queue processing
- ✅ Health check endpoint: `GET /api/health/queues`
- ✅ Cleanup worker for old items

**Enabled Phases**:
- ✅ Phase 20 (Unified Generation Queue) - Now unblocked
- ✅ Phase 16 (Director Decision Queue) - Now unblocked
- ✅ Phase 18.2.3 (WebSocket refactoring) - Completed as part of Phase 19C

**Implementation Phases**:

**Phase 19A: Core Queue Infrastructure** (Week 1) ✅ **COMPLETE**
- [✅] **19A.1** Create queue port interfaces (2025-12-14)
  - File: `Engine/src/application/ports/outbound/queue_port.rs` (NEW)
  - `QueuePort<T>`, `ApprovalQueuePort<T>`, `ProcessingQueuePort<T>`
  - `QueueItem<T>`, `QueueItemStatus`, `QueueError`

- [✅] **19A.2** Create queue item types (2025-12-14)
  - File: `Engine/src/domain/value_objects/queue_items.rs` (NEW)
  - `QueueItemId`, `PlayerActionItem`, `DMActionItem`, `LLMRequestItem`, `AssetGenerationItem`, `ApprovalItem`

- [✅] **19A.3** Implement InMemoryQueue (2025-12-14)
  - File: `Engine/src/infrastructure/queues/memory_queue.rs` (NEW)
  - For development and testing
  - Priority-based dequeue

- [✅] **19A.4** Implement SqliteQueue (2025-12-14)
  - File: `Engine/src/infrastructure/queues/sqlite_queue.rs` (NEW)
  - Production persistence
  - Added `sqlx` with SQLite feature to Cargo.toml
  - Schema: queue_items table with indexes

**Phase 19B: Queue Services** (Week 2) ✅ **COMPLETE**
- [✅] **19B.1** Create PlayerActionQueueService (2025-12-14)
  - File: `Engine/src/application/services/player_action_queue_service.rs` (NEW)
  - Enqueue and process player actions
  - Routes to LLMReasoningQueue

- [✅] **19B.2** Create DMActionQueueService (2025-12-14)
  - File: `Engine/src/application/services/dm_action_queue_service.rs` (NEW)
  - Enqueue and process DM actions (approvals, triggers, etc.)

- [✅] **19B.3** Create LLMQueueService (2025-12-14)
  - File: `Engine/src/application/services/llm_queue_service.rs` (NEW)
  - Concurrency-controlled LLM processing (semaphore)
  - Background worker task
  - Routes responses to DMApprovalQueue

- [✅] **19B.4** Create AssetGenerationQueueService (2025-12-14)
  - File: `Engine/src/application/services/asset_generation_queue_service.rs` (NEW)
  - ComfyUI request processing with concurrency control
  - Background worker task

- [✅] **19B.5** Create DMApprovalQueueService (2025-12-14)
  - File: `Engine/src/application/services/dm_approval_queue_service.rs` (NEW)
  - History, delay, expiration features
  - Approval decision processing

**Phase 19C: WebSocket Integration** (Week 2-3) ✅ **COMPLETE** (2025-12-14)
- [✅] **19C.1** Update WebSocket handler to use queues (2025-12-14)
  - File: `Engine/src/infrastructure/websocket.rs`
  - ✅ Added queue status events (`ActionQueued`, `QueueStatus`)
  - ✅ Player actions enqueued to `PlayerActionQueue`
  - ✅ Approval decisions enqueued to `DMActionQueue`
  - ✅ Handler returns immediately after enqueueing (no locks held)
  - ✅ Removed all synchronous processing
  - ✅ Removed dead code (`process_player_action_with_llm` function)
  - ✅ Handler is now thin adapter (parse → enqueue → return)
  - **Completes Phase 18.2.3** (WebSocket refactoring) ✅

- [✅] **19C.2** Add queue status WebSocket events (2025-12-14)
  - File: `Engine/src/infrastructure/websocket.rs`
  - ✅ `QueueStatus` event type added
  - ✅ `ActionQueued` event type added
  - ✅ Events ready for use when queues are wired

- [✅] **19C.3** Start background workers (2025-12-14)
  - File: `Engine/src/main.rs`
  - ✅ Spawned worker tasks for LLM and asset generation queues
  - ✅ Added graceful shutdown handling
  - ✅ Queue services added to AppState
  - ✅ Player action worker implemented with prompt building
  - ✅ Approval notification worker implemented
  - ✅ DM action queue worker implemented
  - ✅ Cleanup worker implemented

**Phase 19D: Configuration & Polish** (Week 3) ✅ **COMPLETE** (Core configuration)
- [✅] **19D.1** Add QueueConfig to AppConfig (2025-12-14)
  - File: `Engine/src/infrastructure/config.rs`
  - ✅ Backend selection (memory/sqlite) with environment variables
  - ✅ Batch sizes, timeouts, retention configurable
  - ✅ SQLite path configuration

- [✅] **19D.2** Queue factory based on backend config (2025-12-14)
  - File: `Engine/src/infrastructure/queues/factory.rs` (NEW)
  - ✅ QueueFactory creates InMemory or SQLite queues based on config
  - ✅ QueueBackendEnum wrapper for runtime backend selection
  - ✅ Modular design allows easy addition of future backends (Redis, etc.)

- [✅] **19D.3** Update AppState to use configured queue backend (2025-12-14)
  - File: `Engine/src/infrastructure/state.rs`
  - ✅ AppState uses QueueFactory to create queues
  - ✅ Supports both InMemory and SQLite backends
  - ✅ Defaults to SQLite for testing (as requested)

- [✅] **19D.4** Cleanup task for old items (2025-12-14)
  - File: `Engine/src/main.rs`
  - ✅ Background task removes completed/failed items older than retention period
  - ✅ Runs every hour, configurable via `history_retention_hours`
  - ✅ Expires old approvals based on `approval_timeout_minutes`

- [✅] **19D.5** Health check for queue status (2025-12-14)
  - File: `Engine/src/infrastructure/http/queue_routes.rs` (NEW)
  - ✅ Endpoint: `GET /api/health/queues`
  - ✅ Shows depth, processing count for all queues
  - ✅ Returns JSON with queue status information

- [ ] **19D.6** Metrics/logging for queue operations
  - Queue depth metrics (can be added via health endpoint)
  - Processing time metrics
  - Error rate tracking

**Tasks**: See plan document for detailed breakdown (19A: Infrastructure, 19B: Services, 19C: WebSocket, 19D: Config)

---

### 2.2 Workflow Configuration Persistence (Player)

**Location**: `Player/src/presentation/components/settings/workflow_config_editor.rs`

**Context**: The workflow configuration editor displays workflow settings but has a TODO for saving changes.

**Tasks**:

- [✅] **2.2.1** Implement save workflow configuration (2024-12-11)
  - File: `Player/src/presentation/components/settings/workflow_config_editor.rs`
  - On save: `POST /api/workflows/{slot}` with configuration
  - Include: workflow JSON, prompt mappings, defaults, locked inputs
  - Show success/error feedback
  - **Note**: Save button wired, but `save_workflow_defaults()` needs full implementation

- [✅] **2.2.2** Implement delete workflow configuration (2025-12-11)
  - File: `Player/src/presentation/components/settings/workflow_config_editor.rs`
  - Red delete button with confirmation modal
  - DELETE /api/workflows/{slot} with refresh

- [✅] **2.2.3** Implement workflow test (2025-12-11)
  - File: `Player/src/presentation/components/settings/workflow_config_editor.rs`
  - Test modal with prompt input
  - Displays generated image and timing info

- [✅] **2.2.4** Load configurations on settings open (2024-12-11)
  - File: `Player/src/presentation/components/settings/workflow_slot_list.rs`
  - `GET /api/workflows` on component mount
  - Display configured vs unconfigured slots
  - Show workflow name and node count

**Dependencies**: Engine workflow API (already complete)

**Acceptance Criteria**:
- Workflow configurations save to Engine
- Test workflow shows generated output
- Slot list reflects current configuration state

---

### 2.3 Spectator View (Player)

**Location**: `Player/src/presentation/views/spectator_view.rs`

**Context**: Currently shows placeholder text. Should display the same scene as PC view but without interaction capabilities.

**Tasks**:

- [✅] **2.3.1** Add scene display to SpectatorView (2025-12-11)
  - File: `Player/src/presentation/views/spectator_view.rs`
  - Copy backdrop and character rendering from pc_view.rs
  - Use same Backdrop and CharacterLayer components
  - Display current scene from GameState

- [✅] **2.3.2** Add dialogue display (read-only) (2025-12-11)
  - File: `Player/src/presentation/views/spectator_view.rs`
  - SpectatorDialogueBox with "Spectating - No choices available"
  - Shows typewriter animation and LLM processing indicator

- [✅] **2.3.3** Add conversation log (2025-12-11)
  - File: `Player/src/presentation/views/spectator_view.rs`
  - ConversationLog component with speaker names
  - Auto-adds entries when dialogue completes

- [✅] **2.3.4** Handle scene updates (2025-12-11)
  - File: `Player/src/presentation/views/spectator_view.rs`
  - Uses same state hooks as pc_view
  - Reactive updates via Dioxus signals

**Dependencies**: Visual novel components (already complete)

**Acceptance Criteria**:
- Spectators see the same scene as players
- Dialogue displays with typewriter effect
- No interaction buttons shown
- Scene updates in real-time

---

## TIER 3: ARCHITECTURE & QUALITY

### 3.0 Engine Architecture Remediation (Phase 18.2)

**Location**: `Engine/src/infrastructure/websocket.rs`, `Engine/src/application/services/`

**Context**: WebSocket adapter contains application orchestration. This violates hexagonal architecture principles.

**Status**: [⏸️] **MERGED INTO Phase 19C** - WebSocket refactoring will be completed as part of Phase 19C (Queue System Integration)

**Current State**:
- ✅ Phase 18.2.1-18.2.2 Complete: SessionManagementPort, PlayerActionService, ApprovalService created
- ✅ Phase 18.2.3 Partial: Ports implemented on SessionManager
- ⏸️ Phase 18.2.3 Remaining: WebSocket handler refactoring **deferred to Phase 19C**

**Why Merged with Phase 19**:
- Phase 19 queue system solves the lock pattern problem that blocked Phase 18.2.3
- WebSocket refactoring requires queue infrastructure anyway
- More efficient to do both together

**What Will Be Done in Phase 19C**:
- Refactor `websocket.rs` to use queue services
- Remove orchestration from WebSocket adapter
- WebSocket becomes thin adapter: parse → enqueue → return
- Worker tasks handle processing (no locks held across async boundaries)

**Reference**: See `architecture_plan.md` Phase 18.2 and `lock_refactoring_analysis.md`

---

### 3.1 Domain-Driven Design Patterns (Engine)

**Location**: `Engine/src/domain/`

**Status**: 🔄 **Partial** - structures exist, need wiring

**Context**: The domain layer has entities and value objects but is missing aggregates, domain events, and domain services. This affects data consistency and extensibility.

**Existing Code** (defined but not wired):
- [✅] `WorldAggregate` defined in `domain/aggregates/world_aggregate.rs` (216 lines)
- [✅] `AggregateError` type for invariant violations
- [✅] `DomainEvent` enum defined in `domain/events/domain_events.rs` (238 lines)
  - 17 event types covering World, Character, Location, Scene, Dialogue, Interaction, Session
- [✅] Use case traits defined in `ports/inbound/use_cases.rs` (177 lines)
  - `ManageWorldUseCase`, `ManageCharacterUseCase`, `ManageLocationUseCase`, `ManageSceneUseCase`
- [✅] Repository port traits fully implemented in `ports/outbound/repository_port.rs`
  - Services already depend on traits, not concrete implementations

**Remaining Tasks**:

- [ ] **3.1.1** Wire WorldAggregate into services (replace direct entity access)
  - Currently: Services access entities directly
  - Goal: All mutations go through aggregate root for invariant enforcement

- [x] **3.1.2** Implement event publisher/subscriber pattern ✅ (2025-12-15)
  - ✅ Created `EventBusPort<AppEvent>` abstraction in `application/ports/outbound/`
  - ✅ Implemented `SqliteEventBus` with `SqliteAppEventRepository`
  - ✅ `AppEvent` DTO with 11 event types (Story, Narrative, Challenge, Generation, Suggestions)
  - ✅ All producers wired: StoryEventService, NarrativeEventService, challenge resolution, generation
  - ✅ WebSocketEventSubscriber maps AppEvents to ServerMessages
  - ✅ Ready for multiple subscribers (analytics, audit, timeline projectors)
  - See `plans/event_bus_architecture.md` for full documentation

- [ ] **3.1.3** Implement use case traits in application services
  - WorldServiceImpl should implement ManageWorldUseCase
  - CharacterServiceImpl should implement ManageCharacterUseCase
  - etc.

- [ ] **3.1.4** Implement Domain Services (if needed)
  - Currently: Most logic is in application services (acceptable)
  - Consider: `RelationshipService`, `MonomythService`, `ArchetypeService` if pure domain logic emerges

**Dependencies**: None (refactoring)

**Acceptance Criteria**:
- World aggregate enforces all invariants
- Domain events can be published and subscribed
- Services depend on port traits, not concrete implementations ✅ (already done)
- Tests can mock repository ports ✅ (already possible)

---

### 3.2 Error Handling (Engine)

**Location**: Throughout Engine codebase

**Context**: Currently uses `anyhow::Result` everywhere, making error handling generic and hard to match on specific error types.

**Tasks**:

- [ ] **3.2.1** Define Domain Errors
  - File: `Engine/src/domain/errors.rs` (new)
  - `WorldError`: NotFound, InvalidName, DuplicateAct
  - `CharacterError`: NotFound, InvalidArchetype, RelationshipToSelf
  - `LocationError`: NotFound, CircularHierarchy, InvalidConnection
  - `SceneError`: NotFound, InvalidCondition

- [ ] **3.2.2** Define Application Errors
  - File: `Engine/src/application/errors.rs` (new)
  - `ServiceError`: Validation, NotFound, Conflict, ExternalService
  - Wrap domain errors

- [ ] **3.2.3** Define Infrastructure Errors
  - File: `Engine/src/infrastructure/errors.rs` (new)
  - `PersistenceError`: Connection, Query, Serialization
  - `LlmError`: Timeout, RateLimit, InvalidResponse
  - `ComfyUIError`: QueueFull, WorkflowFailed

- [ ] **3.2.4** Update services to use typed errors
  - Files: All files in `Engine/src/application/services/`
  - Replace `anyhow::Result` with specific error types
  - Use `thiserror` derive macro

- [ ] **3.2.5** Map errors to HTTP responses
  - File: `Engine/src/infrastructure/http/error_handler.rs` (new)
  - 400 Bad Request: Validation errors
  - 404 Not Found: Entity not found
  - 409 Conflict: Duplicate resources
  - 500 Internal Server Error: Infrastructure errors
  - Include error details in JSON response

**Dependencies**: None

**Acceptance Criteria**:
- All errors have specific types
- HTTP responses have appropriate status codes
- Error messages are helpful for debugging
- No `unwrap()` or `expect()` in production paths

---

### 3.3 Testing (Engine)

**Location**: `Engine/tests/` (new directory)

**Context**: No tests exist. Need unit tests for domain logic and integration tests for repositories and API.

**Tasks**:

- [ ] **3.3.1** Set up test infrastructure
  - File: `Engine/tests/common/mod.rs` (new)
  - Test database setup/teardown helpers
  - Mock implementations for LlmPort
  - Test fixture factories (create_test_world, etc.)

- [ ] **3.3.2** Domain entity unit tests
  - File: `Engine/tests/domain/` (new directory)
  - Test archetype transitions
  - Test relationship sentiment clamping
  - Test want intensity validation
  - Test directorial notes to_prompt()

- [ ] **3.3.3** Repository integration tests
  - File: `Engine/tests/integration/` (new directory)
  - Test World CRUD with Neo4j
  - Test Character relationships
  - Test Location hierarchy and connections
  - Requires test Neo4j instance

- [ ] **3.3.4** API endpoint tests
  - File: `Engine/tests/api/` (new directory)
  - Test each REST endpoint
  - Test error responses
  - Use axum-test or similar

- [ ] **3.3.5** WebSocket tests
  - File: `Engine/tests/websocket/` (new directory)
  - Test session join/leave
  - Test message routing
  - Test approval flow

**Dependencies**: 3.2 (Error Handling - for proper assertions)

**Acceptance Criteria**:
- `cargo test` passes
- >70% code coverage on domain layer
- Integration tests run in CI

---

### 3.4 Security (Engine)

**Location**: `Engine/src/main.rs`, `Engine/src/infrastructure/`

**Context**: CORS allows all origins, no authentication, no authorization.

**Tasks**:

- [ ] **3.4.1** Fix CORS configuration
  - File: `Engine/src/main.rs`
  - Allow specific origins from environment variable
  - Allow credentials for WebSocket
  - Restrict methods and headers

- [ ] **3.4.2** Add authentication middleware
  - File: `Engine/src/infrastructure/auth/mod.rs` (new)
  - JWT token validation
  - Extract user ID from token
  - Add to request extensions

- [ ] **3.4.3** Add authorization checks
  - File: `Engine/src/infrastructure/http/*.rs`
  - Check world ownership before mutations
  - DM-only endpoints (directorial updates)
  - Admin-only endpoints (if applicable)

- [ ] **3.4.4** Secure WebSocket connections
  - File: `Engine/src/infrastructure/websocket.rs`
  - Require auth token on connect
  - Validate token before joining session
  - Rate limit messages per client

- [ ] **3.4.5** Input validation
  - File: `Engine/src/application/` (various)
  - Validate string lengths
  - Sanitize HTML/XSS in text fields
  - Validate IDs exist before operations

**Dependencies**: None

**Acceptance Criteria**:
- CORS only allows configured origins
- All endpoints require valid auth token
- Users can only modify their own worlds
- Invalid input returns 400 with details

---

### 3.5 Testing (Player)

**Location**: `Player/tests/` (new directory)

**Tasks**:

- [ ] **3.5.1** Set up test infrastructure
  - File: `Player/tests/common/mod.rs` (new)
  - Mock WebSocket server
  - Mock world snapshot fixtures

- [ ] **3.5.2** State management tests
  - File: `Player/tests/state/` (new directory)
  - Test SessionState transitions
  - Test GameState updates
  - Test DialogueState typewriter logic

- [ ] **3.5.3** Component tests
  - Consider using dioxus testing utilities
  - Test visual novel components render correctly
  - Test form validation

**Dependencies**: None

**Acceptance Criteria**:
- `cargo test` passes
- State transitions tested
- Components render without panic

---

## TIER 4: SESSION & WORLD MANAGEMENT

### 4.1 World Selection Flow (Player) - Phase 13

**Location**: `Player/src/presentation/views/world_select.rs`, `Player/src/main.rs`

**Plan Document**: `plans/13-world-selection.md`

**Context**: Worlds act as "servers" in WrldBldr. Before gameplay, users must select or create a World. This adds a World Selection screen between Role Selection and Game Views.

**Flow**:
```
MainMenu → RoleSelect → WorldSelect → GameView
                            ↓
                DM: Create/Continue World
                Player: Join Existing World
                Spectator: Watch Existing World
```

**Tasks**:

- [✅] **4.1.1** Add WorldSelect to AppView enum (2025-12-11)
  - File: `Player/src/main.rs`
  - Added WorldSelect variant
  - Track selected_role signal

- [✅] **4.1.2** Create WorldSelectView component (2025-12-11)
  - File: `Player/src/presentation/views/world_select.rs`
  - Role-aware UI (DM: create/continue, Player: join, Spectator: watch)
  - World list from GET /api/worlds
  - Create world form (DM only)
  - Load world snapshot on selection

- [✅] **4.1.3** Update navigation flow (2025-12-11)
  - File: `Player/src/main.rs`
  - RoleSelect → WorldSelect (not directly to game views)
  - Connection to Engine happens after world selection
  - Back button returns to role select

- [✅] **4.1.4** Fix DiceSystem deserialization (2025-12-11)
  - File: `Player/src/infrastructure/asset_loader/world_snapshot.rs`
  - Match Engine's DiceSystem enum exactly
  - Support D20, D100, DicePool, Fate, Custom variants

- [ ] **4.1.5** Add world statistics display
  - Show character count, location count, scene count
  - Show last modified date
  - DM online indicator for players

- [ ] **4.1.6** Implement world delete (DM)
  - Confirmation modal with world name typing
  - DELETE /api/worlds/{id}
  - Cannot delete while players connected

- [ ] **4.1.7** Implement world edit (DM)
  - Edit name, description without entering
  - PUT /api/worlds/{id}

**Dependencies**: Engine World API (existing)

**Acceptance Criteria**:
- Users must select a world before entering gameplay
- DMs can create new worlds or continue existing ones
- Players/Spectators can join available worlds
- World loads correctly with rule system

---

### 4.2 Rule Systems & Challenges (Both) - Phase 14

**Location**: Engine domain + Player views

**Plan Document**: `plans/14-rule-systems-and-challenges.md`

**Context**: Support multiple TTRPG systems (D&D 5e, Call of Cthulhu, Kids on Bikes, etc.) through modular rule systems, character sheets, skills, and challenge mechanics. LLM analyzes scenes and suggests challenges; DM makes final decisions.

**Architecture**:
```
World Creation → Select Rule System → Customize Skills → Define Challenges
                        ↓
Director Mode: LLM suggests challenge → DM approves → Resolution → Narrative
```

**Phase 14A: Rule System Selection** ✅ (2025-12-12)

- [✅] **4.2.1** Add RuleSystem to World entity (2025-12-12)
  - File: `Engine/src/domain/entities/world.rs`
  - RuleSystemType enum: D20, D100, Narrative, Custom
  - RuleSystemVariant enum with 9 presets
  - SuccessComparison enum for different resolution mechanics

- [✅] **4.2.2** Create rule system presets (2025-12-12)
  - File: `Engine/src/domain/value_objects/rule_system.rs`
  - D20: D&D 5e, Pathfinder 2e, Generic D20
  - D100: Call of Cthulhu 7e, RuneQuest, Generic D100
  - Narrative: Kids on Bikes, FATE Core, PbtA
  - Full stat definitions for each preset

- [✅] **4.2.3** Update world creation UI (2025-12-12)
  - File: `Player/src/presentation/views/world_select.rs`
  - Rule system type dropdown with descriptions
  - Preset dropdown that updates based on type
  - Description text for selected preset

- [✅] **4.2.4** API: Rule system endpoints (2025-12-12)
  - File: `Engine/src/infrastructure/http/rule_system_routes.rs`
  - GET /api/rule-systems (list available types with presets)
  - GET /api/rule-systems/{type} (details for a system type)
  - GET /api/rule-systems/{type}/presets (list presets with configs)
  - GET /api/rule-systems/{type}/presets/{variant} (full preset config)

**Phase 14B: Skill System** (2025-12-12) ✅

- [✅] **4.2.5** Create Skill entity (2025-12-12)
  - File: `Engine/src/domain/entities/skill.rs` (new)
  - Skill struct with id, name, category, base_attribute, is_custom, is_hidden
  - SkillCategory enum for UI organization
  - SkillId value object

- [✅] **4.2.6** Populate default skills per system (2025-12-12)
  - `default_skills_for_variant()` function
  - D&D 5e: 18 skills across STR/DEX/CON/INT/WIS/CHA
  - Pathfinder 2e: 16 skills
  - Call of Cthulhu 7e: 33 skills in 5 categories
  - Kids on Bikes: 6 approaches
  - FATE Core: 18 skills
  - PbtA: 9 basic moves

- [✅] **4.2.7** Skills management UI (2025-12-12)
  - File: `Player/src/presentation/components/settings/skills_panel.rs` (new)
  - SkillsPanel modal component with skills grouped by category
  - SkillRow component with visibility toggle, edit, delete actions
  - AddSkillForm and EditSkillForm for custom skills
  - API integration (fetch, create, update, delete)
  - Integrated into DM Director view via "Manage Skills" button

- [✅] **4.2.8** API: Skill endpoints (2025-12-12)
  - File: `Engine/src/infrastructure/http/skill_routes.rs` (new)
  - GET /api/worlds/{world_id}/skills (list with fallback to defaults)
  - POST /api/worlds/{world_id}/skills (create custom skill)
  - PUT /api/worlds/{world_id}/skills/{skill_id} (update)
  - DELETE /api/worlds/{world_id}/skills/{skill_id} (delete custom only)
  - POST /api/worlds/{world_id}/skills/initialize (populate defaults)

**Phase 14C: Character Sheet Templates** (In Progress)

- [✅] **4.2.9** Create CharacterSheetTemplate entity (2025-12-12)
  - File: `Engine/src/domain/entities/sheet_template.rs` (new)
  - SheetTemplateId, CharacterSheetTemplate, SheetSection, SheetField
  - FieldType: Number, Text, Checkbox, Select, SkillReference, Derived, Resource, ItemList, SkillList
  - SectionLayout: Vertical, Grid, Flow, TwoColumn
  - CharacterSheetData and FieldValue for storing actual values

- [✅] **4.2.10** Default templates per rule system (2025-12-12)
  - D&D 5e: Ability Scores, Modifiers, Combat (HP/AC/Initiative), Skills, Features, Inventory
  - Pathfinder 2e: Ability Scores, Combat, Skills
  - Call of Cthulhu 7e: Characteristics, Derived (HP/Sanity/MP), Skills
  - Kids on Bikes: Stats (dice types d4-d20), Adversity, Strengths/Flaws
  - FATE Core: Aspects, Skills, Stress/Consequences, Refresh, Stunts
  - PbtA: Stats (-2 to +3), Harm, XP, Basic Moves, Playbook Moves
  - RuneQuest, Generic D20, Generic D100, Custom minimal templates

- [✅] **4.2.10b** API: Sheet template endpoints (2025-12-12)
  - File: `Engine/src/infrastructure/http/sheet_template_routes.rs` (new)
  - GET /api/worlds/{id}/sheet-template (get default or generate)
  - GET /api/worlds/{id}/sheet-templates (list all)
  - POST /api/worlds/{id}/sheet-template/initialize (persist default)
  - POST /api/worlds/{id}/sheet-templates/{id}/sections (add section)
  - POST /api/worlds/{id}/sheet-templates/{id}/sections/{id}/fields (add field)
  - DELETE /api/worlds/{id}/sheet-templates/{id}

- [✅] **4.2.10c** Player-side types (2025-12-12)
  - File: `Player/src/infrastructure/asset_loader/world_snapshot.rs`
  - SheetTemplate, SheetSection, SheetField, SectionLayout
  - FieldType, SelectOption, ItemListType
  - CharacterSheetData, FieldValue

- [x] **4.2.11** Update character creation (2025-12-12)
  - File: `Player/src/presentation/components/creator/character_form.rs`
  - Created `sheet_field_input.rs` with CharacterSheetForm, SheetSectionInput, SheetFieldInput components
  - Renders fields dynamically based on template (Number, Text, Checkbox, Select, Resource, Derived)
  - Fetches template from `/api/worlds/{id}/sheet-template`
  - Saves sheet_data with character
  - Collapsible "Character Sheet" section after narrative fields

- [x] **4.2.12** Character sheet viewer (PC View) (2025-12-12)
  - File: `Player/src/presentation/components/character_sheet_viewer.rs` (new)
  - Created CharacterSheetViewer modal component with sections and fields
  - Read-only view with resource bars, formatted values
  - Integrated into PC View via ActionPanel "character" button
  - Placeholder shown when no template available

**Phase 14D: Challenge System Core**

- [✅] **4.2.13** Create Challenge entity (2025-12-12)
  - File: `Engine/src/domain/entities/challenge.rs` (new)
  - ChallengeType: SkillCheck, AbilityCheck, SavingThrow, OpposedCheck, ComplexChallenge
  - Difficulty: DC, Percentage, Descriptor, Opposed, Custom
  - ChallengeOutcomes: success, failure, partial, critical_success, critical_failure
  - TriggerCondition and TriggerType for LLM-driven suggestions
  - ChallengeResult and OutcomeType for resolution

- [✅] **4.2.14** Challenge outcomes and triggers (2025-12-12)
  - OutcomeTrigger: RevealInformation, EnableChallenge, DisableChallenge, ModifyCharacterStat, TriggerScene, GiveItem, Custom
  - TriggerType: ObjectInteraction, EnterArea, DialogueTopic, ChallengeComplete, TimeBased, NpcPresent, Custom
  - Basic keyword matching for trigger evaluation

- [✅] **4.2.15** Challenge library UI (2025-12-12)
  - File: `Player/src/presentation/components/dm_panel/challenge_library.rs` (new)
  - Browse by type and skill with filtering
  - Quick search across name/description/tags
  - Favorites section with toggle
  - Challenge cards grouped by type
  - Active/inactive status toggle

- [✅] **4.2.16** Challenge creation modal (2025-12-12)
  - File: `Player/src/presentation/components/dm_panel/challenge_library.rs` (ChallengeFormModal)
  - Name, description, skill selection, difficulty
  - Success/failure outcomes with triggers
  - Tags for organization
  - Edit mode for existing challenges

- [✅] **4.2.17** API: Challenge endpoints (2025-12-12)
  - File: `Engine/src/infrastructure/http/challenge_routes.rs` (new)
  - Neo4jChallengeRepository with CRUD operations
  - GET /api/worlds/{id}/challenges - list all
  - GET /api/worlds/{id}/challenges/active - list active
  - GET /api/worlds/{id}/challenges/favorites - list favorites
  - GET /api/scenes/{id}/challenges - list by scene
  - POST /api/worlds/{id}/challenges - create
  - GET/PUT/DELETE /api/challenges/{id}
  - PUT /api/challenges/{id}/favorite - toggle
  - PUT /api/challenges/{id}/active - set status

**Phase 14E: LLM Challenge Integration**

- [✅] **4.2.18** Add challenges to LLM context (2025-12-12)
  - File: `Engine/src/application/services/llm_service.rs`
  - Added `ActiveChallengeContext` struct with trigger hints
  - Extended `GamePromptRequest` with `active_challenges` field
  - System prompt includes "Active Challenges" section
  - Instructs LLM to use `<challenge_suggestion>` XML tags

- [✅] **4.2.19** LLM challenge suggestion parsing (2025-12-12)
  - File: `Engine/src/application/services/llm_service.rs`
  - `ChallengeSuggestion` struct with challenge_id, confidence, reasoning
  - `SuggestionConfidence` enum (High, Medium, Low)
  - `parse_response()` extracts challenge suggestions from LLM output
  - WebSocket `ApprovalRequired` includes optional challenge suggestion

- [✅] **4.2.20** Challenge suggestion approval UI (2025-12-12)
  - File: `Player/src/presentation/components/dm_panel/approval_popup.rs`
  - Challenge suggestion section with amber styling
  - Shows challenge name, skill, difficulty, confidence, reasoning
  - "Approve Challenge" and "Skip Challenge" buttons

- [✅] **4.2.21** Challenge resolution flow (2025-12-12)
  - File: `Player/src/presentation/components/tactical/challenge_roll.rs` (new)
  - `ChallengeRollModal` with d20 roll interface
  - Displays challenge name, skill, difficulty, character modifier
  - Roll result breakdown (roll + modifier = total)
  - Platform-specific random generation (WASM/native)
  - WebSocket message types: `ChallengePrompt`, `ChallengeResolved`

- [✅] **4.2.22** Manual challenge trigger (2025-12-12)
  - File: `Player/src/presentation/components/dm_panel/trigger_challenge_modal.rs` (new)
  - `TriggerChallengeModal` with challenge and character selection
  - "Trigger Challenge" button in DM Quick Actions
  - Filters to active challenges only
  - WebSocket `TriggerChallenge` message type

**Phase 14F: Player Experience**

- [✅] **4.2.23** Player challenge prompt (2025-12-12)
  - File: `Player/src/presentation/views/pc_view.rs`
  - WebSocket messages: `ChallengePrompt`, `ChallengeResolved`, `ChallengeRoll`
  - `ChallengePromptData` and `ChallengeResultData` structs in session state
  - `ChallengeRollModal` shown when active challenge exists
  - `send_challenge_roll()` function sends roll result to Engine

- [✅] **4.2.24** Player skill display (2025-12-12)
  - File: `Player/src/presentation/components/tactical/skills_display.rs` (new)
  - `SkillsDisplay` component with skills grouped by category
  - `PlayerSkillData` struct with modifier and proficiency
  - Color-coded modifiers (green positive, red negative)
  - Visual indicator for proficient skills (golden border)
  - Session state `player_skills` signal

---

### 4.3 Routing & Navigation (Player) - Phase 15

**Location**: `Player/src/main.rs`, `Player/Cargo.toml`

**Plan Document**: `plans/14-routing-navigation.md`

**Context**: The Player app currently uses signal-based view switching without proper routing. This causes navigation issues:
- **Web**: Refresh loses location, no browser history, no shareable URLs
- **Mobile**: No deep linking support
- **Desktop**: No custom URL scheme handling

**Proposed URL Structure**:
```
/                           → MainMenu
/roles                      → RoleSelect
/worlds                     → WorldSelect
/worlds/:world_id/dm        → DMView (Director)
/worlds/:world_id/dm/creator → DMView (Creator mode)
/worlds/:world_id/play      → PCView
/worlds/:world_id/watch     → SpectatorView
```

**Phase 15A: Basic Router Setup**

- [✅] **4.3.1** Add router feature to Dioxus (2025-12-12)
  - File: `Player/Cargo.toml`
  - Added `router` feature to both desktop and web dioxus dependencies

- [✅] **4.3.2** Define Route enum (2025-12-12)
  - File: `Player/src/routes.rs` (new, 469 lines)
  - Route enum with `#[derive(Routable)]` macro
  - 7 routes: MainMenuRoute, RoleSelectRoute, WorldSelectRoute, DMViewRoute, PCViewRoute, SpectatorViewRoute, NotFoundRoute
  - world_id parameter for game views

- [✅] **4.3.3** Update App to use Router (2025-12-12)
  - File: `Player/src/main.rs` (reduced from 538 to 66 lines)
  - Replaced signal-based view switching with `Router::<Route>`
  - Context providers for state preserved

- [✅] **4.3.4** Convert views to accept route params (2025-12-12)
  - Route components wrap presentation views
  - DMViewRoute, PCViewRoute, SpectatorViewRoute receive world_id
  - Connection logic moved to route components

**Phase 15B: Route Guards & Navigation**

- [✅] **4.3.5** Implement route guards (2025-12-12)
  - Routes handle missing state by redirecting
  - Connection initiated on world selection
  - Disconnect handled on back navigation

- [✅] **4.3.6** Update navigation calls (2025-12-12)
  - All `current_view.set(...)` replaced with `navigator().push(...)`
  - Back buttons use navigator with disconnect logic
  - All navigation handlers updated

- [✅] **4.3.7** Create NotFound component (2025-12-12)
  - 404 page with route path display
  - Link back to main menu
  - Clean styling

**Phase 15C: Web History Integration**

- [✅] **4.3.8** Verify browser history works (2025-12-12)
  - Automatic via Dioxus Router
  - URL updates on navigation
  - Back/forward buttons work

- [✅] **4.3.9** Add state persistence localStorage (2025-12-12)
  - File: `Player/src/infrastructure/storage.rs` (new)
  - Saves server URL, role, last world
  - Platform-agnostic API (WASM + desktop stubs)
  - Clear on disconnect

- [✅] **4.3.10** Update page titles (2025-12-12)
  - File: `Player/src/routes.rs`
  - `set_page_title()` helper for dynamic titles
  - Per-route titles: "Main Menu", "Select Role", "Dungeon Master", etc.

- [✅] **4.3.11** Test deep links (2025-12-12)
  - Documented in code comments
  - Direct URLs load correct view
  - Missing state redirects appropriately

**Phase 15D: Desktop URL Scheme**

- [✅] **4.3.12** Register `wrldbldr://` scheme (2025-12-12)
  - macOS: `assets/macos/Info.plist` with CFBundleURLTypes
  - Windows: `assets/windows/url-scheme.reg` registry entry
  - Linux: `assets/linux/wrldbldr.desktop` with MimeType

- [✅] **4.3.13** Handle incoming URLs (2025-12-12)
  - File: `Player/src/infrastructure/url_handler.rs` (new)
  - `parse_url_scheme()` maps URLs to Route variants
  - 8 unit tests for URL parsing

**Phase 15E: Mobile Deep Linking (Documentation)**

- [✅] **4.3.14** Android intent filters (2025-12-12)
  - Documented in `docs/DEEP_LINKING.md`
  - Example AndroidManifest.xml configuration

- [✅] **4.3.15** iOS URL types (2025-12-12)
  - Documented in `docs/DEEP_LINKING.md`
  - Example Info.plist configuration

**Dependencies**:
- Phase 13 (World Selection) - for world-based routes
- Dioxus 0.7+ with router feature

**Acceptance Criteria**:
- URL reflects current location
- Refresh preserves navigation state (web)
- Browser back/forward buttons work
- Deep links navigate correctly
- Desktop app handles URL scheme
- Protected routes redirect properly
- 404 page for invalid routes

---

### 4.4 Story Arc (Both) - Phase 17

**Location**: Engine domain + Player views (new Story Arc tab)

**Plan Document**: `plans/17-story-arc.md`

**Context**: The Story Arc feature introduces two complementary systems:
1. **StoryEvent (Past Events Timeline)** - Automatic logging of ALL gameplay events (dialogue, movement, combat, items, relationships)
2. **NarrativeEvent (Future Events)** - Designer-defined events with triggers, outcomes, and branching chains

**Architecture**:
```
Gameplay → StoryEvents logged automatically → Timeline View
DM designs → NarrativeEvents with triggers → LLM detects conditions → DM approves → Event fires
Events chain together → Branching storylines based on outcomes
```

**New Entities**:
- `StoryEvent`: Immutable log of past gameplay events (13 event types)
- `NarrativeEvent`: Future event hooks with triggers and outcomes
- `EventChain`: Connected sequences of narrative events

**UI Location**: New "Story Arc" tab (4th tab in DM View) + small widget in Director view showing relevant pending events

**Phase 17A: Domain Foundation (Engine)** ✅ (2025-12-12)

- [✅] **4.4.1** Add new ID types to ids.rs
  - StoryEventId, NarrativeEventId, EventChainId

- [✅] **4.4.2** Create StoryEvent entity
  - File: `Engine/src/domain/entities/story_event.rs` (new)
  - StoryEventType enum: 18 event types including LocationChange, DialogueExchange, CombatEvent, ChallengeAttempted, etc.

- [✅] **4.4.3** Create NarrativeEvent entity
  - File: `Engine/src/domain/entities/narrative_event.rs` (new)
  - NarrativeTrigger with 14 trigger types
  - TriggerLogic: All (AND), Any (OR), AtLeast(n)
  - EventOutcome with conditions, effects, and chain links

- [✅] **4.4.4** Create EventChain entity
  - File: `Engine/src/domain/entities/event_chain.rs` (new)
  - Links NarrativeEvents in sequences with branching

**Phase 17B: Story Events Backend (Engine)** ✅ (2025-12-12)

- [✅] **4.4.5** Create Neo4jStoryEventRepository
  - CRUD operations for story events
  - Query by session, world, filters

- [✅] **4.4.6** Create story_event_routes.rs
  - GET /api/sessions/{id}/story-events (list with filters)
  - POST /api/sessions/{id}/story-events (DM markers)
  - PUT /api/story-events/{id} (update summary, hidden, tags)
  - GET /api/worlds/{id}/story-events/search

- [✅] **4.4.7** Create StoryEventService for recording events
  - record_dialogue_exchange, record_challenge_attempted, etc.
  - 10 convenience methods for recording different event types

**Phase 17C: Narrative Events Backend (Engine)** ✅ (2025-12-12)

- [✅] **4.4.8** Create Neo4jNarrativeEventRepository
  - CRUD operations with trigger/outcome storage

- [✅] **4.4.9** Create narrative_event_routes.rs
  - GET/POST/PUT/DELETE for narrative events
  - PUT /api/narrative-events/{id}/active (toggle)
  - PUT /api/narrative-events/{id}/favorite (toggle)
  - POST /api/narrative-events/{id}/trigger (manual)

- [✅] **4.4.10** Create event_chain_routes.rs
  - CRUD for chains
  - POST /api/event-chains/{id}/events (add to chain)
  - DELETE /api/event-chains/{id}/events/{event_id}

**Phase 17D: LLM Integration (Engine)** ✅ (2025-12-12)

- [✅] **4.4.12** Add narrative events to LLM context
  - ActiveNarrativeEventContext in GamePromptRequest
  - Narrative events section in system prompt
  - Instruct LLM to suggest triggers via <narrative_event_suggestion> tags

- [✅] **4.4.13** Parse event triggers from LLM responses
  - NarrativeEventSuggestion struct with confidence and reasoning
  - Parse <narrative_event_suggestion> tags

- [✅] **4.4.14** WebSocket event suggestion flow
  - NarrativeEventSuggestionInfo in ApprovalRequired message
  - NarrativeEventSuggestionDecision client message
  - Handler for DM approval/rejection

- [⏭️] **4.4.15** Custom trigger LLM evaluation (deferred)

**Phase 17E: Story Arc Tab UI (Player)** ✅ (2025-12-12)

- [✅] **4.4.16** Create StoryArc mode in DMView
  - Added StoryArc to DMMode enum
  - StoryArcContent with three sub-views: Timeline, Narrative Events, Event Chains
  - Tab navigation within Story Arc

- [✅] **4.4.17** Create TimelineView with event cards
  - Vertical scrollable timeline with TimelineEventCard components
  - TimelineFilters with event type, search, date range, show hidden toggle
  - Event cards with type icons, timestamps, summaries

- [✅] **4.4.18** Create EventDetailModal
  - Full event information popup
  - Related characters, event-type-specific details
  - Visibility toggle

- [✅] **4.4.19** Create AddDmMarkerModal
  - Title, note, importance, marker type
  - Tags support

**Phase 17F: Narrative Event UI (Player)** ✅ (2025-12-12)

- [✅] **4.4.20** Create NarrativeEventLibrary view
  - Grid/list with search and status filters
  - Favorites toggle
  - Stats bar with counts

- [✅] **4.4.21** Create NarrativeEventCard component
  - Status badges (Active, Triggered, Pending)
  - Priority indicator, tag display
  - Favorite and active toggles

- [ ] **4.4.22** Create TriggerConditionBuilder component (deferred)
- [ ] **4.4.23** Create OutcomeBuilder component (deferred)
- [ ] **4.4.24** Create EventTriggerApproval popup (deferred)

**Phase 17G: Event Chains & Polish (Player)** 🔄 (In Progress)

- [ ] **4.4.25** Create EventChainVisualizer component
  - Placeholder created, full implementation pending

- [✅] **4.4.26** Create PendingEventsWidget for Director view
  - Shows 3-5 most relevant pending events by priority
  - Compact display with event names and trigger counts

- [✅] **4.4.27** Add Story Arc tab to DM View
  - 4th tab after Director, Creator, Settings
  - "Story Arc" label with proper routing

- [ ] **4.4.28** Export/import functionality (deferred)

**Dependencies**:
- Phase 14D (Challenge System) - Reuse TriggerCondition patterns
- Existing LLM approval flow infrastructure

**Acceptance Criteria**:
- All gameplay events automatically logged to timeline
- DM can filter, search, and annotate timeline
- DM can design narrative events with complex triggers
- LLM detects trigger conditions and suggests events
- DM approves/rejects event triggers
- Event chains visualized as flowcharts
- Events branch based on outcome selection
- Pending events widget shows relevant events in Director view

---

### 4.5 ComfyUI Enhancements (Both) - Phase 18

**Location**: Engine infrastructure + Player components

**Plan Document**: `plans/18-comfyui-enhancements.md`

**Context**: The current ComfyUI integration has critical gaps:
- GenerationService emits events but they're NOT broadcast via WebSocket (disconnected)
- Player's session_service has TODO: "Update GenerationState when passed to this function"
- GenerationQueuePanel only shows "Failed" label, not error message
- No retry logic, circuit breaker, or health monitoring
- Style references not implemented (deferred from Phase 11)

**Architecture**:
```
Phase 18A: ComfyUI Resilience (Backend)
    │
    └──> Phase 18B: Generation Event Wiring (Full Stack) [CRITICAL]
              │
              ├──> Phase 18C: Style Reference System (Full Stack)
              │
              ├──> Phase 18D: Director Mode Quick Generate (Frontend)
              │
              └──> Phase 18E: Batch Management UI (Frontend)
```

**Phase 18A: ComfyUI Resilience (Engine)**

- [ ] **4.5.1** Create ComfyUIConfig value object
  - File: `Engine/src/domain/value_objects/comfyui_config.rs` (NEW)
  - max_retries (1-5), base_delay_seconds (1-30), timeouts

- [ ] **4.5.2** Extend ComfyUIError enum
  - Add ServiceUnavailable, CircuitOpen, Timeout, MaxRetriesExceeded

- [ ] **4.5.3** Add CircuitBreaker struct
  - Track failure count, open/close state
  - 5 failures → open for 60s → half-open probe

- [ ] **4.5.4** Add cached health check (5s TTL)
  - Check before queue_prompt(), fail fast if unhealthy

- [ ] **4.5.5** Add retry wrapper with exponential backoff
  - Default: 5s, 15s, 45s delays
  - Only retry transient errors (5xx, timeout)

- [ ] **4.5.6** Add config API endpoints
  - GET/PUT /api/config/comfyui

**Phase 18B: Generation Event Wiring (Full Stack)** [CRITICAL]

**Status**: ✅ **PARTIALLY COMPLETE** (2025-12-15) - GenerationService wired to AppState and event broadcasting implemented. Remaining tasks are frontend integration.

**Note**: `GenerationService` in `application/services/generation_service.rs` (509 lines) exists with full event system (`GenerationEvent` enum, event channel). ✅ **NOW INSTANTIATED** in `AppState` and wired to WebSocket via generation event broadcaster worker.

- [✅] **4.5.7** Wire GenerationService events to WebSocket (2025-12-15)
  - ✅ GenerationService added to AppState
  - ✅ Generation event broadcaster worker implemented in main.rs
  - ✅ Events converted to ServerMessage and broadcast to all sessions
  - ✅ All generation event types (BatchQueued, BatchProgress, BatchComplete, BatchFailed) supported

- [ ] **4.5.8** Add ComfyUIStateChanged WebSocket message
  - Both Engine and Player message types
  - state, message, retry_in_seconds fields

- [ ] **4.5.9** Handle generation events in session_service
  - File: `Player/src/application/services/session_service.rs`
  - Fix TODO at line ~484: call GenerationState methods

- [ ] **4.5.10** Update GenerationQueuePanel error display
  - File: `Player/src/presentation/components/creator/generation_queue.rs`
  - Show error messages, expandable details, retry button

- [ ] **4.5.11** Create ComfyUI disconnected banner
  - File: `Player/src/presentation/components/creator/comfyui_banner.rs` (NEW)
  - Show when disconnected, countdown to retry

**Phase 18C: Style Reference System (Full Stack)**

- [ ] **4.5.12** Add StyleReferenceMapping to workflow config
  - AutoDetect, Specific{node_id, input_name}, PromptInjection, Disabled

- [ ] **4.5.13** Implement IPAdapter auto-detection
  - Scan workflow for nodes with "IPAdapter" in class_type

- [ ] **4.5.14** Inject style reference into workflow
  - IPAdapter: set image input to reference path
  - Fallback: append style keywords to prompt

- [ ] **4.5.15** Add "Use as Reference" to asset context menu
  - Select reference, show in generation modal

- [ ] **4.5.16** Add style reference input selector to workflow editor
  - Dropdown with detected image inputs

**Phase 18D: Director Mode Quick Generate (Player)**

- [ ] **4.5.17** Add generate buttons to NPC panel
  - File: `Player/src/presentation/components/dm_panel/npc_motivation.rs`
  - "Generate Portrait", "Generate Sprite" buttons

- [ ] **4.5.18** Create DirectorGenerateModal
  - File: `Player/src/presentation/components/dm_panel/director_generate_modal.rs` (NEW)
  - Pre-populate prompt from character description

- [ ] **4.5.19** Add queue badge to Director Mode header
  - Show active generation count

- [ ] **4.5.20** Create minimal DirectorQueuePanel
  - Slide-in panel showing active batches

**Phase 18E: Batch Management UI (Player)**

- [ ] **4.5.21** Add cancel batch endpoint
  - DELETE /api/assets/batch/{batch_id}

- [ ] **4.5.22** Add retry batch endpoint
  - POST /api/assets/batch/{batch_id}/retry

- [ ] **4.5.23** Add clear batch endpoint
  - DELETE /api/assets/batch/{batch_id}/clear

- [ ] **4.5.24** Add batch management buttons to queue UI
  - Cancel (queued), Retry/Clear (failed), Select/Clear (ready)

- [ ] **4.5.25** Add batch details expansion
  - Show prompt, workflow, timestamps when expanded

- [ ] **4.5.26** Add "Clear All Completed" bulk action

**Dependencies**:
- Phase 11/12 (Asset Gallery, Workflow Settings) - existing infrastructure
- Existing WebSocket message types (need wiring, not creation)

**Acceptance Criteria**:
- Real-time generation progress via WebSocket
- Detailed error messages with expandable details
- ComfyUI disconnected banner with reconnect countdown
- Style references work via IPAdapter or prompt injection
- Generate assets from Director Mode NPC panel
- Cancel/retry/clear batches from queue UI

---

## TIER 5: FUTURE FEATURES

### 5.1 Tactical Combat (Both)

**Context**: GridMap entity exists but no renderer or game logic.

**Tasks**:

- [ ] **5.1.1** Engine: Combat service
  - Turn order calculation
  - Movement validation (pathfinding, terrain costs)
  - Attack resolution (using RuleSystem dice)
  - Combat state in session

- [ ] **5.1.2** Engine: Combat WebSocket messages
  - CombatStart, CombatUpdate, CombatEnd
  - CombatAction (Move, Attack, UseAbility, EndTurn)

- [ ] **5.1.3** Player: Grid renderer
  - File: `Player/src/presentation/components/tactical/`
  - Tile-based rendering
  - Elevation visualization
  - Unit positioning

- [ ] **5.1.4** Player: Combat UI
  - Turn order display
  - Action selection
  - Movement preview
  - Attack targeting

**Dependencies**: Core gameplay (Tier 1)

---

### 5.2 Audio System (Player)

**Context**: Audio module exists but is empty.

**Tasks**:

- [ ] **5.2.1** Implement audio manager
  - File: `Player/src/infrastructure/audio/manager.rs` (new)
  - Background music playback
  - Sound effect triggers
  - Volume control

- [ ] **5.2.2** Add audio to scenes
  - Background music per location
  - Dialogue blips
  - UI feedback sounds

**Dependencies**: None

---

### 5.3 Save/Load System (Both)

**Context**: Planned in Phase 9 but not started.

**Tasks**:

- [ ] **5.3.1** Engine: Save file format
  - Define GameSave structure
  - World state snapshot
  - Session state (conversation, quest progress)

- [ ] **5.3.2** Engine: Save/Load endpoints
  - `POST /api/sessions/{id}/save`
  - `GET /api/saves`
  - `POST /api/saves/{id}/load`

- [ ] **5.3.3** Player: Save/Load UI
  - Save button in menu
  - Load game screen
  - Save file list with timestamps

**Dependencies**: Core gameplay (Tier 1)

---

## APPENDIX A: File Quick Reference

### Engine - Key Files to Modify

| Task Area | Primary Files |
|-----------|---------------|
| LLM Integration | `websocket.rs`, `llm_service.rs`, `session.rs` |
| DM Approval | `websocket.rs`, `messages.rs` |
| DDD Patterns | `domain/aggregates/`, `domain/events/`, `domain/services/` |
| Error Handling | New files in each layer |
| Security | `main.rs`, `auth/` (new) |
| Testing | `tests/` (new directory) |

### Player - Key Files to Modify

| Task Area | Primary Files |
|-----------|---------------|
| Action Sending | `pc_view.rs`, `action_service.rs` |
| Creator Mode | `entity_browser.rs`, `character_form.rs`, `location_form.rs`, `asset_gallery.rs` |
| Workflow Config | `workflow_config_editor.rs` |
| Spectator View | `spectator_view.rs` |
| Testing | `tests/` (new directory) |

---

## APPENDIX B: Environment Setup

### Engine Development

```bash
# Enter Nix shell
cd Engine
nix-shell

# Or install dependencies manually
# - Rust toolchain
# - Neo4j 5.x running on localhost:7687
# - Ollama running on configured host

# Set environment
export NEO4J_PASSWORD="your_password"
export OLLAMA_BASE_URL="http://localhost:11434/v1"

# Run
cargo run
```

### Player Development

```bash
# Enter Nix shell
cd Player
nix-shell

# Install JS dependencies
npm install

# Run desktop
cargo run

# Run web
trunk serve
```

### Docker Development

```bash
# From repository root
docker-compose up

# Engine at http://localhost:3000
# Neo4j at http://localhost:7474
```

---

## APPENDIX C: Coordination Notes

### Parallel Work Opportunities

These task groups can be worked on simultaneously:

1. **Group A**: LLM Integration (1.1) + DM Approval (1.2) - Engine focused
2. **Group B**: Creator Mode (2.1) + Workflow Config (2.2) - Player focused
3. **Group C**: DDD Patterns (3.1) + Error Handling (3.2) - Refactoring
4. **Group D**: Testing (3.3, 3.5) - Can start after respective features

### Sequential Dependencies

```
1.1 LLM Integration
    └── 1.2 DM Approval
        └── 1.3 Player Action Sending (verification)

3.1 DDD Patterns
    └── 3.2 Error Handling
        └── 3.3 Testing

2.1 Creator Mode API
    └── 2.2 Workflow Config (both need API integration patterns)
```

### Communication Points

Sub-agents working on different areas should sync on:
- WebSocket message format changes (affects both Engine and Player)
- API response format changes (affects Player API calls)
- State structure changes (affects multiple Player components)

---

## APPENDIX D: Definition of Done

A task is complete when:

1. **Code**: Implementation compiles without warnings
2. **Tests**: Related tests pass (when test infrastructure exists)
3. **Integration**: Works with connected components
4. **Documentation**: Code comments for non-obvious logic
5. **Review**: No obvious bugs or security issues

---

## Change Log

| Date | Changes |
|------|---------|
| 2024-12-11 | Initial plan created from codebase analysis |
| 2024-12-11 | Completed Phase 12 Workflow Settings: Engine API routes, Player Settings UI with slot list, upload modal, and config editor. Marked 2.2.1 and 2.2.4 as complete. |
| 2025-12-11 | **TIER 1 COMPLETE**: All 15 Critical Path tasks implemented via rust-pro sub-agents. LLM Integration (1.1.1-1.1.5), DM Approval Flow (1.2.1-1.2.5), Player Action Sending (1.3.1-1.3.5). New files: game_tools.rs, tool_execution_service.rs. Major changes to websocket.rs, session.rs, llm_service.rs, pc_view.rs, dialogue_state.rs, action_panel.rs, dialogue_box.rs. Both Engine and Player compile successfully. |
| 2025-12-11 | **TIER 2 COMPLETE**: All Feature Completeness tasks implemented. Creator Mode API Integration (2.1.1-2.1.6): EntityBrowser, CharacterForm, LocationForm fetch/save via REST API; AssetGallery manages assets; SuggestionButton calls LLM. Workflow Config (2.2.2-2.2.3): Delete with confirmation modal, Test with image preview. Spectator View (2.3.1-2.3.4): Full scene display, read-only dialogue, conversation log. Fixed compilation errors in spectator_view.rs, asset_gallery.rs, character_form.rs, location_form.rs. |
| 2025-12-11 | **Phase 13 (World Selection)**: Added WorldSelect to app flow (4.1.1-4.1.4). WorldSelectView with role-aware UI. Fixed DiceSystem deserialization mismatch between Engine/Player. Navigation: RoleSelect → WorldSelect → GameView. |
| 2025-12-11 | **Phase 14 (Rule Systems & Challenges)**: Added comprehensive plan for modular TTRPG systems. 24 tasks across 6 sub-phases: Rule System Selection (14A), Skill System (14B), Character Sheet Templates (14C), Challenge System Core (14D), LLM Challenge Integration (14E), Player Experience (14F). UI mockups in plans/14-rule-systems-and-challenges.md. |
| 2025-12-12 | **Phase 14A Complete**: Rule System Selection implemented. RuleSystemType/Variant enums with 9 presets (D&D 5e, Pathfinder 2e, CoC 7e, Kids on Bikes, FATE, PbtA). API endpoints in rule_system_routes.rs. World creation UI with type/preset dropdowns. Full RuleSystemConfig editing with advanced configuration section. |
| 2025-12-12 | **Phase 14B Complete**: Skill System fully implemented including UI. Skill entity with category, base_attribute, is_custom, is_hidden. Default skills for all 9 system presets (18 D&D skills, 33 CoC skills, etc.). Neo4j repository and API endpoints. **Skills management UI** in `skills_panel.rs`: modal with skills grouped by category, visibility toggle, add/edit/delete custom skills. Integrated into DM Director view via "Manage Skills" button in Quick Actions. |
| 2025-12-12 | Renamed IMPLEMENTATION_PLAN.md to ROADMAP.md for clarity. |
| 2025-12-12 | **Phase 14C Backend Complete**: Character Sheet Templates. `sheet_template.rs` with comprehensive field type system (Number, Text, Checkbox, Select, SkillReference, Derived, Resource, ItemList, SkillList). Default templates for all 9 rule system variants with appropriate sections (D&D 5e has 6 sections including ability scores, modifiers, combat, skills, features, inventory). Neo4j repository stores templates as JSON. API routes for CRUD operations. Player-side types added. UI components (4.2.11, 4.2.12) pending. |
| 2025-12-12 | **Phase 15 (Routing & Navigation) Added**: Comprehensive plan for Dioxus Router integration. 15 tasks across 5 sub-phases: Basic Router Setup (15A), Route Guards & Navigation (15B), Web History Integration (15C), Desktop URL Scheme (15D), Mobile Deep Linking (15E). URL structure: `/`, `/roles`, `/worlds`, `/worlds/:world_id/dm\|play\|watch`. Detailed plan in `plans/14-routing-navigation.md`. |
| 2025-12-12 | **Phase 14D Complete**: Challenge System Core. Challenge entity with ChallengeType (SkillCheck, AbilityCheck, SavingThrow, OpposedCheck, ComplexChallenge), Difficulty (DC, Percentage, Descriptor, Opposed, Custom), ChallengeOutcomes, OutcomeTrigger, TriggerCondition. Neo4jChallengeRepository with CRUD operations. REST API endpoints (list, get, create, update, delete, toggle favorite, set active). ChallengeLibrary UI component with search, filtering by type/favorites/active, challenge cards grouped by type, and ChallengeFormModal for create/edit. Integrated into DM View with "Manage Challenges" button. |
| 2025-12-12 | **Phase 14E Complete**: LLM Challenge Integration. `ActiveChallengeContext` added to LLM prompts with trigger hints. LLM outputs `<challenge_suggestion>` tags parsed into `ChallengeSuggestion` (challenge_id, confidence, reasoning). ApprovalPopup shows challenge suggestions with amber styling. `ChallengeRollModal` for d20 rolls with modifier calculation and platform-specific random generation. `TriggerChallengeModal` for manual challenge triggering. WebSocket message types: `ChallengePrompt`, `ChallengeResolved`, `ChallengeRoll`, `TriggerChallenge`, `ChallengeSuggestionDecision`. |
| 2025-12-12 | **Phase 14F Complete**: Player Experience. Player-side WebSocket messages for `ChallengePrompt`, `ChallengeResolved`, `ChallengeRoll`. Session state extended with `active_challenge`, `challenge_results`, `player_skills` signals. PC View shows `ChallengeRollModal` when challenge is active, sends roll via `send_challenge_roll()`. New `SkillsDisplay` component groups skills by category with color-coded modifiers and proficiency indicators. Phase 14 (Rule Systems & Challenges) is now complete! |
| 2025-12-12 | **Phase 15A-B Complete**: Routing & Navigation. Added Dioxus Router with 7 routes (MainMenu, RoleSelect, WorldSelect, DMView, PCView, SpectatorView, NotFound). New `routes.rs` (469 lines) with route components that wrap presentation views. main.rs reduced from 538 to 66 lines. Navigation uses `navigator().push()` instead of signal-based switching. Route guards handle missing state, connection logic moved to route components. NotFound page with 404 display and home link. |
| 2025-12-12 | **Phase 15 Complete**: All routing tasks done. **15C**: localStorage persistence (`storage.rs`), dynamic page titles via `set_page_title()`. **15D**: `wrldbldr://` URL scheme assets for macOS (Info.plist), Windows (registry), Linux (.desktop). URL parser in `url_handler.rs` with 8 unit tests. **15E**: Mobile deep linking documented in `docs/DEEP_LINKING.md` with Android/iOS examples. |
| 2025-12-12 | **Phase 17 (Story Arc) Added**: Comprehensive plan for past events timeline and future narrative events system. 28 tasks across 7 sub-phases: Domain Foundation (17A), Story Events Backend (17B), Narrative Events Backend (17C), LLM Integration (17D), Story Arc Tab UI (17E), Narrative Event UI (17F), Event Chains & Polish (17G). New entities: StoryEvent (13 event types), NarrativeEvent (14 trigger types, branching outcomes), EventChain. New UI: Story Arc tab (4th DM tab), Timeline view, Narrative Event Designer, Event Chain Visualizer, Pending Events Widget. Detailed plan in `plans/17-story-arc.md`. |
| 2025-12-14 | **Phase 19 (Queue System) Analysis**: Validated queue system architecture. Created comprehensive analysis documents. Identified Phase 19 as current priority - solves lock pattern issues and enables Phase 15/16. Phase 18.2.3 (WebSocket refactoring) merged into Phase 19C. |
| 2025-12-15 | **Phase 19 COMPLETE + Code Cleanup**: All 5 queue services operational (PlayerActionQueue, DMActionQueue, LLMReasoningQueue, AssetGenerationQueue, DMApprovalQueue). Background workers spawned in main.rs. Health check endpoint at `/api/health/queues`. Fixed compilation errors in Engine (lifetime issues in run_worker methods, 'static bounds). Fixed Player clippy error in challenge_library.rs (`*value <= 0` → `*value == 0` for u32). Renamed "Phase 15 Generation Queue" to **Phase 20** to avoid conflict with "Phase 15 Routing & Navigation". **Phase 20 and Phase 16 now UNBLOCKED.** |
| 2025-12-15 | **Code Review Remediation**: Removed unused config fields (`worker_poll_interval_ms`, `worker_error_delay_ms`). Fixed empty challenge/skill lookups in LLMQueueService - now looks up full challenge details (name, skill, difficulty) and narrative event details (name, description, scene_direction) before sending to DM approval queue. Removed 4 empty modules (`domain/services`, `infrastructure/asset_manager`, `Player/domain/services`, `Player/infrastructure/audio`). Annotated unused DDD structures (WorldAggregate, DomainEvent, use_case traits) with `#[allow(dead_code)]` and documentation explaining Phase 3.1 implementation plan. Updated Phase 3.1 status to "Partial" with existing code inventory. Added note about orphaned GenerationService in Phase 18B. |
| 2025-12-15 | **Code Review Next Steps Complete**: Fixed all critical issues identified in code review. **GenerationService wired**: Added to AppState, instantiated with event channel, generation event broadcaster worker implemented in main.rs. **ChallengeSuggestionDecision handler completed**: Full implementation with approval item lookup, challenge loading, difficulty modification, and ChallengePrompt broadcasting. **NarrativeEventSuggestionDecision handler completed**: Full implementation with event loading, outcome selection, event triggering, StoryEvent recording, and NarrativeEventTriggered broadcasting. **WebSocket message synchronization**: Added NarrativeEventTriggered to both Engine and Player. Added get_by_id() method to DMApprovalQueueService. Both Engine and Player compile successfully. **Phase 20 and Phase 16 now fully unblocked.** |
| 2025-12-15 | **Phase 20A (Unified Generation Queue Core)**: Routed LLM suggestions through `LLMReasoningQueue`, implemented suggestion processing in `LLMQueueService`, added `GenerationEvent::Suggestion*` and corresponding `ServerMessage::Suggestion*` WebSocket events, extended Player `GenerationState` to track suggestion tasks, updated `SuggestionButton` to enqueue instead of synchronous HTTP, and updated `GenerationQueuePanel` to show both image batches and suggestion tasks. Phase 20 now **PARTIALLY COMPLETE** (Engine complete, Player core UI implemented; advanced UX pending). |
| 2025-12-15 | **Event Bus Architecture Implemented**: Full pub/sub infrastructure for cross-cutting system events. Created `EventBusPort<AppEvent>` abstraction with SQLite backend (`SqliteEventBus`, `SqliteAppEventRepository`), `InProcessEventNotifier` + 30s polling for resilience. Defined `AppEvent` DTO with 11 event types (Story, Narrative, Challenge, Generation, Suggestions). Refactored generation pipeline: `GenerationEvent` → `GenerationEventPublisher` → `AppEvent` → `WebSocketEventSubscriber` → `ServerMessage`. Extended all producers: `StoryEventService` (10+ methods), `NarrativeEventService.mark_triggered()`, challenge resolution in websocket, generation/suggestion events. Enhanced `GenerationEvent` with entity context. Both Engine and Player compile. Ready for Redis backend and advanced consumers (analytics, timeline projectors). Comprehensive documentation in `event_bus_architecture.md`. |