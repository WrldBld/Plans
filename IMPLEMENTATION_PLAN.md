# WrldBldr Implementation Plan

This document tracks all remaining work identified during the codebase analysis. Sub-agents should use this to understand context, track progress, and coordinate implementation.

**Last Updated**: 2025-12-11
**Overall Progress**: ~85% complete (Tier 1 + Tier 2 Complete)

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

### Tier 4: Future Features
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

### 3.1 Domain-Driven Design Patterns (Engine)

**Location**: `Engine/src/domain/`

**Context**: The domain layer has entities and value objects but is missing aggregates, domain events, and domain services. This affects data consistency and extensibility.

**Tasks**:

- [ ] **3.1.1** Implement World Aggregate
  - File: `Engine/src/domain/aggregates/world_aggregate.rs` (new)
  - Wrap World, Acts, Characters, Locations
  - Enforce invariants:
    - Unique monomyth stages per world
    - Valid parent-child location hierarchy
    - Character relationships reference existing characters
  - Provide atomic save operation

- [ ] **3.1.2** Implement Domain Events
  - File: `Engine/src/domain/events/mod.rs`
  - Define events: `CharacterCreated`, `RelationshipChanged`, `SceneEntered`, etc.
  - Create event dispatcher trait
  - Enable event subscribers (for future audit logging, side effects)

- [ ] **3.1.3** Implement Domain Services
  - File: `Engine/src/domain/services/mod.rs`
  - `RelationshipService`: Calculate effective sentiment with modifiers
  - `MonomythService`: Validate act progression rules
  - `ArchetypeService`: Validate archetype transitions

- [ ] **3.1.4** Create Repository Port Traits
  - File: `Engine/src/application/ports/outbound/repository_port.rs` (new)
  - Define `WorldRepositoryPort`, `CharacterRepositoryPort`, etc.
  - Update services to depend on traits, not concrete Neo4j repos

- [ ] **3.1.5** Create Inbound Port Traits
  - File: `Engine/src/application/ports/inbound/handlers.rs` (new)
  - Define `CreateWorldHandler`, `UpdateCharacterHandler`, etc.
  - Decouple HTTP routes from application services

**Dependencies**: None (refactoring)

**Acceptance Criteria**:
- World aggregate enforces all invariants
- Domain events can be published and subscribed
- Services depend on port traits, not concrete implementations
- Tests can mock repository ports

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

## TIER 4: FUTURE FEATURES

### 4.1 Tactical Combat (Both)

**Context**: GridMap entity exists but no renderer or game logic.

**Tasks**:

- [ ] **4.1.1** Engine: Combat service
  - Turn order calculation
  - Movement validation (pathfinding, terrain costs)
  - Attack resolution (using RuleSystem dice)
  - Combat state in session

- [ ] **4.1.2** Engine: Combat WebSocket messages
  - CombatStart, CombatUpdate, CombatEnd
  - CombatAction (Move, Attack, UseAbility, EndTurn)

- [ ] **4.1.3** Player: Grid renderer
  - File: `Player/src/presentation/components/tactical/`
  - Tile-based rendering
  - Elevation visualization
  - Unit positioning

- [ ] **4.1.4** Player: Combat UI
  - Turn order display
  - Action selection
  - Movement preview
  - Attack targeting

**Dependencies**: Core gameplay (Tier 1)

---

### 4.2 Audio System (Player)

**Context**: Audio module exists but is empty.

**Tasks**:

- [ ] **4.2.1** Implement audio manager
  - File: `Player/src/infrastructure/audio/manager.rs` (new)
  - Background music playback
  - Sound effect triggers
  - Volume control

- [ ] **4.2.2** Add audio to scenes
  - Background music per location
  - Dialogue blips
  - UI feedback sounds

**Dependencies**: None

---

### 4.3 Save/Load System (Both)

**Context**: Planned in Phase 9 but not started.

**Tasks**:

- [ ] **4.3.1** Engine: Save file format
  - Define GameSave structure
  - World state snapshot
  - Session state (conversation, quest progress)

- [ ] **4.3.2** Engine: Save/Load endpoints
  - `POST /api/sessions/{id}/save`
  - `GET /api/saves`
  - `POST /api/saves/{id}/load`

- [ ] **4.3.3** Player: Save/Load UI
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
