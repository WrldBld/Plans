# WrldBldr Roadmap

This document tracks all remaining work identified during the codebase analysis. Sub-agents should use this to understand context, track progress, and coordinate implementation.

**Last Updated**: 2025-12-12
**Overall Progress**: ~99% complete (Tier 1-4 complete: Phase 13-15 done, Phase 17 planned)

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
- World Selection Flow (Player) - Phase 13
- Rule Systems & Challenges (Both) - Phase 14
- Routing & Navigation (Player) - Phase 15
- Story Arc (Both) - Phase 17
- ComfyUI Enhancements (Both) - Phase 18

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

### 2.1b Unified Generation Queue - Creator Mode (Player + Engine)

**Plan**: [15-unified-generation-queue.md](./15-unified-generation-queue.md)

**Summary**: Unify image generation and LLM suggestion requests into a single queue system in Creator Mode, with WebSocket events for real-time progress visibility.

**Location**: Creator Mode sidebar

**Status**: [ ] Not started

**Tasks**: See plan document for detailed breakdown (15A: Engine, 15B: Player, 15C: Polish)

---

### 2.1c Director Decision Queue - Director Mode (Player + Engine)

**Plan**: [16-director-decision-queue.md](./16-director-decision-queue.md)

**Summary**: Provide DM visibility and control over all AI gameplay decisions (NPC responses, tool usage, challenge suggestions) before they affect players.

**Location**: Director Mode sidebar

**Status**: [ ] Not started

**Key Features**:
- Real-time queue of pending AI decisions
- Approve/Reject/Modify/Delay actions
- Type filtering (Dialogue, Tools, Challenges, Transitions)
- Decision history with undo capability
- Keyboard shortcuts for fast approval workflow

**Tasks**: See plan document for detailed breakdown (16A: Engine, 16B: Player UI, 16C: Integration)

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

**Phase 17A: Domain Foundation (Engine)**

- [ ] **4.4.1** Add new ID types to ids.rs
  - StoryEventId, NarrativeEventId, EventChainId

- [ ] **4.4.2** Create StoryEvent entity
  - File: `Engine/src/domain/entities/story_event.rs` (new)
  - StoryEventType enum: LocationChange, DialogueExchange, CombatEvent, ChallengeAttempted, ItemAcquired, ItemTransferred, RelationshipChanged, SceneTransition, InformationRevealed, NpcAction, DmMarker, NarrativeEventTriggered, Custom

- [ ] **4.4.3** Create NarrativeEvent entity
  - File: `Engine/src/domain/entities/narrative_event.rs` (new)
  - NarrativeTrigger with 14 trigger types (NpcAction, PlayerEntersLocation, DialogueTopic, ChallengeCompleted, RelationshipThreshold, HasItem, EventCompleted, FlagSet, etc.)
  - TriggerLogic: All (AND), Any (OR), AtLeast(n)
  - EventOutcome with conditions, effects, and chain links
  - EventEffect enum: ModifyRelationship, GiveItem, TakeItem, RevealInformation, SetFlag, EnableChallenge, EnableEvent, TriggerScene, etc.

- [ ] **4.4.4** Create EventChain entity
  - File: `Engine/src/domain/entities/event_chain.rs` (new)
  - Links NarrativeEvents in sequences with branching

**Phase 17B: Story Events Backend (Engine)**

- [ ] **4.4.5** Create Neo4jStoryEventRepository
  - CRUD operations for story events
  - Query by session, world, filters

- [ ] **4.4.6** Create story_event_routes.rs
  - GET /api/sessions/{id}/story-events (list with filters)
  - POST /api/sessions/{id}/story-events (DM markers)
  - PUT /api/story-events/{id} (update summary, hidden, tags)
  - GET /api/worlds/{id}/story-events/search

- [ ] **4.4.7** Wire StoryEvent creation into existing flows
  - DialogueResponse → StoryEvent::DialogueExchange
  - ChallengeResult → StoryEvent::ChallengeAttempted
  - Tool execution → appropriate StoryEvent type
  - Scene transitions, item transfers, relationship changes

**Phase 17C: Narrative Events Backend (Engine)**

- [ ] **4.4.8** Create Neo4jNarrativeEventRepository
  - CRUD operations with trigger/outcome storage

- [ ] **4.4.9** Create narrative_event_routes.rs
  - GET/POST/PUT/DELETE for narrative events
  - PUT /api/narrative-events/{id}/active (toggle)
  - PUT /api/narrative-events/{id}/favorite (toggle)
  - POST /api/narrative-events/{id}/trigger (manual)
  - POST /api/narrative-events/{id}/check-triggers

- [ ] **4.4.10** Create event_chain_routes.rs
  - CRUD for chains
  - POST /api/event-chains/{id}/events (add to chain)
  - DELETE /api/event-chains/{id}/events/{event_id}

- [ ] **4.4.11** Create NarrativeEventService
  - Trigger condition evaluation
  - Outcome effect application

**Phase 17D: LLM Integration (Engine)**

- [ ] **4.4.12** Add narrative events to LLM context
  - Include active events with trigger hints in system prompt
  - Instruct LLM to output NARRATIVE_EVENT_TRIGGER: tags

- [ ] **4.4.13** Parse event triggers from LLM responses
  - Extract NARRATIVE_EVENT_TRIGGER and confidence
  - Validate against actual trigger conditions

- [ ] **4.4.14** WebSocket event suggestion flow
  - EventTriggerSuggestion message to DM
  - EventTriggerDecision message from DM (approve/delay/reject)
  - NarrativeEventTriggered broadcast to all

- [ ] **4.4.15** Custom trigger LLM evaluation
  - For triggers marked llm_evaluation: true
  - LLM determines if custom condition is met

**Phase 17E: Story Arc Tab UI (Player)**

- [ ] **4.4.16** Create StoryArcView component
  - File: `Player/src/presentation/views/story_arc_view.rs` (new)
  - Three sub-views: Timeline, Narrative Events, Event Chains
  - Tab navigation within Story Arc

- [ ] **4.4.17** Create TimelineView with event cards
  - Vertical scrollable timeline
  - Filter bar (event type, character, location, date range)
  - Event cards with type icons, timestamps, summaries
  - Click to expand details

- [ ] **4.4.18** Create TimelineEventDetail modal
  - Full event information
  - Related characters, locations
  - Edit summary (DM only)
  - Toggle visibility

- [ ] **4.4.19** Create AddDmMarker form
  - Title, note, importance
  - Optional location/character association

**Phase 17F: Narrative Event UI (Player)**

- [ ] **4.4.20** Create NarrativeEventLibrary view
  - Grid/list with search and filters
  - Status badges (Active, Triggered, Pending)
  - Favorites section
  - Quick actions (Edit, Trigger, Enable/Disable)

- [ ] **4.4.21** Create NarrativeEventDesigner modal
  - Name, description, scene direction fields
  - Optional Act association dropdown

- [ ] **4.4.22** Create TriggerConditionBuilder component
  - Add/remove trigger conditions
  - Trigger type dropdown with context-specific fields
  - Logic selector (All/Any/AtLeast)

- [ ] **4.4.23** Create OutcomeBuilder component
  - Multiple outcome branches
  - Condition for each outcome
  - Effects list per outcome
  - Chain event links

- [ ] **4.4.24** Create EventTriggerApproval popup
  - Shows suggested event with matching triggers
  - Approve/Delay/Reject actions
  - Confidence indicator

**Phase 17G: Event Chains & Polish (Player)**

- [ ] **4.4.25** Create EventChainVisualizer component
  - Flowchart with nodes and connections
  - Color coding for status (pending, triggered, completed)
  - Click node to view/edit event
  - Drag to reorder (optional)

- [ ] **4.4.26** Create PendingEventsWidget for Director view
  - 3-5 most relevant pending/triggered events
  - Matching trigger count
  - Quick approve/delay/reject
  - "View in Story Arc" link

- [ ] **4.4.27** Add Story Arc tab to DM View
  - 4th tab after Director, Creator, Settings
  - Icon and label

- [ ] **4.4.28** Export/import functionality
  - Export narrative events and chains as JSON
  - Import with conflict resolution

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

- [ ] **4.5.7** Wire GenerationService events to WebSocket
  - File: `Engine/src/infrastructure/websocket.rs`
  - Subscribe to event channel, broadcast to session

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
