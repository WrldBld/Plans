# WrldBldr Code Review Discoveries

**Date**: 2025-12-16
**Reviewers**: Claude Opus 4.5 with specialized review agents
**Scope**: Complete codebase review of Engine and Player components
**Purpose**: Document all issues, gaps, and improvements for sub-agent implementation

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Summary](#2-architecture-summary)
3. [Critical Architecture Violations](#3-critical-architecture-violations)
4. [Engine Issues](#4-engine-issues)
5. [Player Issues](#5-player-issues)
6. [Feature Gaps](#6-feature-gaps)
7. [Challenge System Analysis](#7-challenge-system-analysis)
8. [Refactoring Opportunities](#8-refactoring-opportunities)
9. [Good Practices to Preserve](#9-good-practices-to-preserve)
10. [Implementation Priority Matrix](#10-implementation-priority-matrix)

---

## 1. Project Overview

### What is WrldBldr?

WrldBldr is a **TTRPG (Tabletop Role-Playing Game) Campaign Management System** consisting of:

- **Engine**: Rust backend server (Axum + Neo4j + Ollama + ComfyUI)
- **Player**: Rust frontend application (Dioxus - Desktop/WASM/Android)

### Core Concept

The system enables AI-assisted tabletop gameplay where:
1. Players interact with NPCs through a visual novel-style interface
2. An LLM (Ollama) suggests NPC responses and skill challenges
3. The Dungeon Master (DM) approves, modifies, or rejects all AI suggestions
4. Skill challenges use configurable dice systems (D20, D100, Fate, custom)
5. Outcomes trigger game state changes through tool calls

### Key Files for Context

| File | Purpose |
|------|---------|
| `/home/otto/repos/WrldBldr/Engine/CLAUDE.md` | Engine architecture rules |
| `/home/otto/repos/WrldBldr/Player/CLAUDE.md` | Player architecture rules |
| `/home/otto/repos/WrldBldr/plans/ROADMAP.md` | Project roadmap and phase tracking |
| `/home/otto/repos/WrldBldr/plans/CLAUDE.md` | Planning conventions |

---

## 2. Architecture Summary

### Hexagonal Architecture (Ports and Adapters)

Both Engine and Player follow hexagonal architecture with these layers:

```
┌─────────────────────────────────────────────────────────────┐
│                        PRESENTATION                         │
│              (HTTP Routes, WebSocket, Dioxus UI)            │
├─────────────────────────────────────────────────────────────┤
│                        INFRASTRUCTURE                        │
│         (Neo4j, Ollama, ComfyUI, HTTP Client, Storage)      │
├─────────────────────────────────────────────────────────────┤
│                         APPLICATION                          │
│              (Services, Ports, DTOs, Use Cases)             │
├─────────────────────────────────────────────────────────────┤
│                           DOMAIN                             │
│            (Entities, Value Objects, Domain Events)          │
└─────────────────────────────────────────────────────────────┘

DEPENDENCY DIRECTION: Outer layers depend on inner layers, NEVER the reverse
```

### Layer Rules (STRICTLY ENFORCED)

| Layer | Can Import | Cannot Import |
|-------|------------|---------------|
| Domain | (nothing external) | application, infrastructure, presentation |
| Application | domain only | infrastructure, presentation |
| Infrastructure | domain, application | presentation |
| Presentation | domain, application | infrastructure (must use services!) |

### Directory Structure

**Engine:**
```
Engine/src/
├── domain/
│   ├── entities/        # World, Character, Scene, Challenge, Skill, etc.
│   ├── value_objects/   # IDs, Archetype, Want, Relationship, Difficulty
│   ├── aggregates/      # WorldAggregate (partially implemented)
│   ├── events/          # DomainEvent enum (defined, not fully wired)
│   └── services/        # Pure domain logic (planned)
├── application/
│   ├── services/        # WorldService, ChallengeService, LlmService, etc.
│   ├── ports/
│   │   ├── inbound/     # Use case traits
│   │   └── outbound/    # LlmPort, QueuePort, RepositoryPort, SessionManagementPort
│   └── dto/             # Request/response types, queue items, app events
└── infrastructure/
    ├── http/            # Axum route handlers
    ├── websocket/       # WebSocket handler and messages
    ├── persistence/     # Neo4j repositories
    ├── queues/          # SQLite/InMemory queue implementations
    ├── ollama.rs        # LLM client
    ├── comfyui.rs       # Asset generation client
    └── session.rs       # SessionManager
```

**Player:**
```
Player/src/
├── domain/
│   ├── entities/        # Character, Scene, Dialogue, PlayerAction
│   └── value_objects/   # IDs
├── application/
│   ├── services/        # SessionService, ActionService, WorldService
│   ├── ports/
│   │   └── outbound/    # ApiPort, GameConnectionPort, Platform
│   └── dto/             # WebSocket messages, WorldSnapshot
├── infrastructure/
│   ├── http_client.rs   # REST API client
│   ├── websocket/       # WebSocket client
│   └── platform/        # Platform-specific code (Desktop/WASM)
└── presentation/
    ├── views/           # MainMenu, DMView, PCView, SpectatorView
    ├── components/      # dm_panel/, creator/, tactical/, settings/
    └── state/           # SessionState, GameState, DialogueState
```

---

## 3. Critical Architecture Violations

### 3.1 Engine: Application Services Importing Infrastructure

**Severity**: CRITICAL
**Impact**: Breaks hexagonal architecture; services cannot be tested in isolation

#### Violation 1: session_join_service.rs

**File**: `/home/otto/repos/WrldBldr/Engine/src/application/services/session_join_service.rs`

```rust
// VIOLATION: Direct infrastructure import
use crate::infrastructure::session::SessionManager;
```

**Fix Required**:
```rust
// CORRECT: Use the existing port trait
use crate::application::ports::outbound::session_management_port::SessionManagementPort;

// Service should accept trait bound
pub struct SessionJoinService<S: SessionManagementPort> {
    session_manager: S,
}
```

#### Violation 2: challenge_resolution_service.rs

**File**: `/home/otto/repos/WrldBldr/Engine/src/application/services/challenge_resolution_service.rs`

```rust
// VIOLATION: Direct infrastructure import
use crate::infrastructure::session::SessionManager;
```

**Fix Required**: Same pattern as above - use `SessionManagementPort` trait bound.

#### Violation 3: narrative_event_approval_service.rs

**File**: `/home/otto/repos/WrldBldr/Engine/src/application/services/narrative_event_approval_service.rs`

```rust
// VIOLATION: Direct infrastructure import
use crate::infrastructure::session::SessionManager;
```

**Fix Required**: Same pattern as above - use `SessionManagementPort` trait bound.

### 3.2 Engine: Serde Derives in Domain Layer

**Severity**: HIGH
**Impact**: Domain layer has external framework dependencies; violates purity

#### Affected Files:

| File | Types with Serde |
|------|------------------|
| `domain/value_objects/ids.rs` | WorldId, CharacterId, SceneId, etc. |
| `domain/value_objects/archetype.rs` | Archetype enum |
| `domain/value_objects/want.rs` | Want, WantIntensity |
| `domain/value_objects/relationship.rs` | Relationship |
| `domain/value_objects/directorial.rs` | DirectorialNotes, NpcMotivation |

**Current (Incorrect)**:
```rust
// In domain/value_objects/ids.rs
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct WorldId(pub Uuid);
```

**Fix Required**:
Create DTO wrappers in application layer:
```rust
// In application/dto/ids.rs
#[derive(Serialize, Deserialize)]
pub struct WorldIdDto(pub String);

impl From<WorldId> for WorldIdDto {
    fn from(id: WorldId) -> Self {
        WorldIdDto(id.0.to_string())
    }
}

impl TryFrom<WorldIdDto> for WorldId {
    type Error = uuid::Error;
    fn try_from(dto: WorldIdDto) -> Result<Self, Self::Error> {
        Ok(WorldId(Uuid::parse_str(&dto.0)?))
    }
}
```

**Note**: This is a significant refactor. For pragmatic reasons, keeping serde on IDs may be acceptable if documented as a known deviation. The priority should be the service-level violations.

### 3.3 Player: Direct HttpClient Usage in Presentation

**Severity**: HIGH
**Impact**: Presentation layer bypasses application services; hard to test/mock

#### Violation 1: world_select.rs

**File**: `/home/otto/repos/WrldBldr/Player/src/presentation/views/world_select.rs`

```rust
// VIOLATION: Direct infrastructure usage
use crate::infrastructure::http_client::HttpClient;

// Inside component:
let worlds = HttpClient::get("/api/worlds").await;
```

**Fix Required**:
```rust
// Use WorldService via context provider
let world_service = use_context::<WorldService>();
let worlds = world_service.list_worlds().await;
```

#### Violation 2: services.rs

**File**: `/home/otto/repos/WrldBldr/Player/src/presentation/services.rs`

```rust
// VIOLATION: Direct infrastructure usage
use crate::infrastructure::http_client::HttpClient;
```

**Fix Required**: Route all HTTP calls through application services.

### 3.4 Player: Serde in Domain Layer

**Severity**: MEDIUM
**Impact**: Domain layer has external dependencies

#### Affected Files:

| File | Types with Serde |
|------|------------------|
| `domain/entities/player_action.rs` | PlayerAction, ActionType |
| `domain/entities/approval.rs` | ApprovalDecision |

**Fix Required**: Move to `application/dto/` as DTOs.

### 3.5 Player: Platform-Specific Code in Application Layer

**Severity**: MEDIUM
**Impact**: Application layer has platform-specific compile flags

**File**: `/home/otto/repos/WrldBldr/Player/src/application/ports/outbound/platform.rs`

```rust
// VIOLATION: cfg attributes in application layer
#[cfg(target_arch = "wasm32")]
impl Platform {
    // WASM implementation
}

#[cfg(not(target_arch = "wasm32"))]
impl Platform {
    // Desktop implementation
}
```

**Fix Required**: Move implementations to `infrastructure/platform/` and expose through trait.

---

## 4. Engine Issues

### 4.1 TODO Items in Challenge Resolution

**File**: `/home/otto/repos/WrldBldr/Engine/src/application/services/challenge_resolution_service.rs`

#### TODO 1: Character Modifiers (Lines ~108, ~224)

```rust
// TODO: integrate real character modifiers
let character_modifier = 0;

// TODO: Look up character's skill modifier from PlayerCharacterService
let character_modifier = 0;
```

**Impact**: All dice rolls always use +0 modifier regardless of character skills.

**Fix Required**:
```rust
// Need to implement:
pub async fn get_skill_modifier(
    &self,
    pc_id: &PlayerCharacterId,
    skill_id: &SkillId,
) -> Result<i32, ServiceError> {
    // 1. Load PlayerCharacter entity
    // 2. Get character sheet data
    // 3. Find skill entry by skill_id
    // 4. Return proficient ? (bonus + proficiency_bonus) : bonus
}
```

#### TODO 2: Character ID Mapping (Line ~120)

```rust
// TODO: derive real character_id from session participant mapping
let character_id = "unknown".to_string();
```

**Impact**: AppEvent::ChallengeResolved always has character_id = "unknown".

**Fix Required**: Look up character from session's participant mapping.

### 4.2 Unimplemented WebSocket Handlers

**File**: `/home/otto/repos/WrldBldr/Engine/src/infrastructure/websocket.rs`

The following ClientMessage handlers exist but are incomplete:

| Message | Status | Notes |
|---------|--------|-------|
| `RegenerateOutcome` | TODO | Handler stub exists, no implementation |
| `DiscardChallenge` | TODO | Handler stub exists, no implementation |
| `CreateAdHocChallenge` | TODO | Handler stub exists, no implementation |

### 4.3 Unimplemented Challenge Types

**File**: `/home/otto/repos/WrldBldr/Engine/src/domain/entities/challenge.rs`

| Challenge Type | Status |
|----------------|--------|
| `SkillCheck` | ✅ Implemented |
| `AbilityCheck` | ✅ Implemented |
| `SavingThrow` | ✅ Implemented |
| `OpposedCheck` | ❌ No resolution logic |
| `ComplexChallenge` | ❌ Multi-roll not implemented |

### 4.4 Unimplemented Difficulty Types

| Difficulty | Status |
|------------|--------|
| `DC(u32)` | ✅ Implemented |
| `Percentage(u32)` | ✅ Implemented |
| `Descriptor(String)` | ⚠️ Partial (no threshold logic) |
| `Opposed` | ❌ No resolution logic |
| `Custom(String)` | ⚠️ Falls through to partial |

### 4.5 Trigger Conditions Not Fully Evaluated

**File**: `/home/otto/repos/WrldBldr/Engine/src/domain/entities/challenge.rs`

```rust
impl TriggerCondition {
    pub fn matches(&self, context: &TriggerContext) -> bool {
        match &self.trigger_type {
            TriggerType::ChallengeComplete { .. } => false, // NOT IMPLEMENTED
            TriggerType::TimeBased { .. } => false,          // NOT IMPLEMENTED
            TriggerType::NpcPresent { .. } => false,         // NOT IMPLEMENTED
            // ... only ObjectInteraction, EnterArea, DialogueTopic work
        }
    }
}
```

### 4.6 Large Service Files

| File | Lines | Issue |
|------|-------|-------|
| `llm_service.rs` | 400+ | Consider splitting prompt building into separate module |
| `websocket.rs` | 600+ | Consider extracting message handlers |
| `character_repository.rs` | 500+ | Acceptable for repository, but could split queries |

---

## 5. Player Issues

### 5.1 Unwrap Usage in Presentation

**Severity**: MEDIUM
**Impact**: Potential runtime panics

Approximately 30+ `.unwrap()` calls found in presentation layer. Example:

```rust
// Dangerous pattern in components
let data = signal.read().as_ref().unwrap();
```

**Fix Required**: Use `if let`, `match`, or `.unwrap_or_default()` patterns.

### 5.2 Large Component Files

| File | Lines | Recommendation |
|------|-------|----------------|
| `dm_view.rs` | 1000+ | Split into DmDirectorTab, DmCreatorTab, etc. |
| `challenge_library.rs` | 500+ | Extract ChallengeCard, ChallengeForm |
| `workflow_config_editor.rs` | 600+ | Extract WorkflowInputEditor, WorkflowTestModal |
| `skills_panel.rs` | 400+ | Extract SkillCard, SkillEditor |

### 5.3 State Types Using Infrastructure DTOs

**File**: `/home/otto/repos/WrldBldr/Player/src/presentation/state/`

Some state types reference infrastructure DTOs directly:

```rust
// In presentation/state/session_state.rs
pub pending_approvals: Signal<Vec<PendingApproval>>,
// PendingApproval contains ProposedTool which is infrastructure DTO
```

**Fix Required**: Create presentation-specific types or use domain types.

### 5.4 Missing Error Handling in Components

Many async operations lack proper error handling:

```rust
// Common pattern
let result = some_service.do_thing().await;
// No .map_err() or error state update
```

**Fix Required**: Add error state signals and display error messages to user.

---

## 6. Feature Gaps

### 6.1 Phase 22: Core Game Loop Gaps

| Feature | Current State | Gap |
|---------|---------------|-----|
| Dice Parsing | ✅ `DiceFormula` works | None |
| Character Modifiers | ❌ Hardcoded to 0 | Need `get_skill_modifier()` |
| Tool Receipts in Outcomes | ❌ Not in LLM suggestions | Need enhanced `ChallengeSuggestionInfo` |
| Outcome Text Editing | ❌ Read-only | Need editable fields in ApprovalPopup |
| Regenerate Outcomes | ⚠️ Message exists | Handler not implemented |
| Discard Challenge | ⚠️ Message exists | Handler not implemented |
| Ad-hoc Challenges | ⚠️ UI exists | Backend not wired |
| New Tool Types | ⚠️ 4 exist | Need 7+ more (NPC motivation, descriptions, etc.) |
| Outcome Trigger Execution | ⚠️ Domain model ready | Not wired after resolution |

### 6.2 Tier 3: Architecture & Quality Gaps

| Feature | Status |
|---------|--------|
| Typed Error System | ❌ Uses `anyhow::Result` everywhere |
| Test Suite | ❌ No tests exist |
| Authentication | ❌ No auth middleware |
| Authorization | ❌ No ownership checks |
| CORS Hardening | ❌ Allows all origins |
| Input Validation | ⚠️ Partial |

### 6.3 Tier 5: Future Features (Not Started)

- Tactical Combat System
- Audio System
- Save/Load System

---

## 7. Challenge System Analysis

### 7.1 Current Flow (Working)

```
PC Action → LLM Processing → ChallengeSuggestion → DM Approval
    → ChallengePrompt → Player Roll → Resolution → Broadcast
```

### 7.2 Domain Model (Complete)

The domain model in `/home/otto/repos/WrldBldr/Engine/src/domain/entities/challenge.rs` is comprehensive:

```rust
pub struct Challenge {
    pub id: ChallengeId,
    pub world_id: WorldId,
    pub scene_id: Option<SceneId>,
    pub name: String,
    pub description: String,
    pub challenge_type: ChallengeType,
    pub skill_id: SkillId,
    pub difficulty: Difficulty,
    pub outcomes: ChallengeOutcomes,
    pub trigger_conditions: Vec<TriggerCondition>,
    pub active: bool,
    pub prerequisite_challenges: Vec<ChallengeId>,
    pub order: u32,
    pub is_favorite: bool,
    pub tags: Vec<String>,
}

pub struct ChallengeOutcomes {
    pub success: Outcome,
    pub failure: Outcome,
    pub partial: Option<Outcome>,
    pub critical_success: Option<Outcome>,
    pub critical_failure: Option<Outcome>,
}

pub struct Outcome {
    pub description: String,
    pub triggers: Vec<OutcomeTrigger>,
}

pub enum OutcomeTrigger {
    RevealInformation { info: String, persist: bool },
    EnableChallenge { challenge_id: ChallengeId },
    DisableChallenge { challenge_id: ChallengeId },
    ModifyCharacterStat { stat: String, modifier: i32 },
    TriggerScene { scene_id: SceneId },
    GiveItem { item_name: String, item_description: String },
    Custom { description: String },
}
```

### 7.3 WebSocket Messages (Complete)

**Engine-side** (`/home/otto/repos/WrldBldr/Engine/src/infrastructure/websocket/messages.rs`):

| Message | Direction | Purpose |
|---------|-----------|---------|
| `ApprovalRequired` | Server→Client | Contains `ChallengeSuggestionInfo` |
| `ChallengeSuggestionDecision` | Client→Server | DM approves/rejects |
| `ChallengePrompt` | Server→Client | Prompts player to roll |
| `ChallengeRollInput` | Client→Server | Player submits dice/result |
| `ChallengeResolved` | Server→Client | Broadcast result |
| `RegenerateOutcome` | Client→Server | DM requests regeneration |
| `DiscardChallenge` | Client→Server | DM discards suggestion |
| `CreateAdHocChallenge` | Client→Server | DM creates custom challenge |
| `OutcomeRegenerated` | Server→Client | Regenerated outcome data |
| `ChallengeDiscarded` | Server→Client | Confirmation |
| `AdHocChallengeCreated` | Server→Client | Confirmation |

**Player-side** (`/home/otto/repos/WrldBldr/Player/src/application/dto/websocket_messages.rs`):

All messages are mirrored and compatible.

### 7.4 Resolution Service (Mostly Complete)

**File**: `/home/otto/repos/WrldBldr/Engine/src/application/services/challenge_resolution_service.rs`

Key methods:
- `handle_suggestion_decision()` - DM approval flow
- `handle_roll_input()` - Dice resolution
- `evaluate_challenge_result()` - Outcome determination

Missing:
- Character modifier integration
- Outcome trigger execution after resolution

### 7.5 Player UI Components

| Component | File | Status |
|-----------|------|--------|
| Challenge Library | `dm_panel/challenge_library.rs` | ✅ Complete |
| Trigger Challenge Modal | `dm_panel/trigger_challenge_modal.rs` | ✅ Complete |
| Ad-hoc Challenge Modal | `dm_panel/adhoc_challenge_modal.rs` | ✅ Complete |
| Approval Popup | `dm_panel/approval_popup.rs` | ⚠️ Needs editing capability |
| Challenge Roll Modal | `tactical/challenge_roll.rs` | ✅ Complete |

---

## 8. Refactoring Opportunities

### 8.1 High Priority

| Area | Current | Proposed | Effort |
|------|---------|----------|--------|
| Service Dependencies | Concrete types | Trait bounds | Medium |
| HTTP Handler Pattern | Some direct repo access | All via services | Medium |
| Error Types | `anyhow::Result` | Typed errors | High |

### 8.2 Medium Priority

| Area | Current | Proposed | Effort |
|------|---------|----------|--------|
| Repository Pattern | Trait per entity | Generic `RepositoryPort<T>` | Medium |
| Component Size | Large view files | Extract sub-components | Medium |
| DTO Mapping | Manual conversions | `From`/`Into` implementations | Low |

### 8.3 Low Priority

| Area | Current | Proposed | Effort |
|------|---------|----------|--------|
| DTO Duplication | Separate in Engine/Player | Shared protocol crate | High |
| Logging | `tracing::info!` everywhere | Structured logging | Low |
| Metrics | None | Prometheus/OpenTelemetry | Medium |

---

## 9. Good Practices to Preserve

### 9.1 Architecture

- **Port Traits**: Well-defined `LlmPort`, `QueuePort`, `RepositoryPort`, `SessionManagementPort`
- **Event Bus**: Clean pub/sub with session scoping
- **Queue System**: Robust crash recovery with SQLite persistence
- **WebSocket Refactor**: Handler is thin adapter (parse → enqueue → return)

### 9.2 Domain Model

- **Rich Entities**: Challenge, Character, Scene have comprehensive fields
- **Value Objects**: Strongly-typed IDs prevent mixing entity types
- **Enums**: ChallengeType, Difficulty, OutcomeTrigger are well-designed

### 9.3 Code Style

- **Async/Await**: Consistent usage throughout
- **Error Propagation**: `?` operator used correctly
- **Documentation**: Most public types have doc comments

---

## 10. Implementation Priority Matrix

### Immediate (Fix Before Phase 22)

1. **Fix 3 service infrastructure imports** (30 minutes)
   - `session_join_service.rs`
   - `challenge_resolution_service.rs`
   - `narrative_event_approval_service.rs`

2. **Fix Player HttpClient usage** (1 hour)
   - `world_select.rs`
   - `presentation/services.rs`

### Phase 22 Implementation

3. **Character modifier integration** (2-3 hours)
4. **Regenerate/Discard handlers** (2-3 hours)
5. **Ad-hoc challenge backend** (2-3 hours)
6. **Outcome trigger execution** (3-4 hours)
7. **Enhanced tool system** (4-6 hours)
8. **Approval UI editing** (3-4 hours)

### Post-Phase 22

9. **Typed error system** (1-2 days)
10. **Test infrastructure** (2-3 days)
11. **Authentication system** (2-3 days)

---

## Appendix A: File Reference

### Engine Critical Files

```
/home/otto/repos/WrldBldr/Engine/src/
├── application/
│   ├── services/
│   │   ├── challenge_resolution_service.rs  # Main challenge logic
│   │   ├── session_join_service.rs          # VIOLATION: infrastructure import
│   │   ├── narrative_event_approval_service.rs  # VIOLATION
│   │   ├── llm_service.rs                   # LLM integration
│   │   └── tool_execution_service.rs        # Tool call execution
│   ├── ports/
│   │   └── outbound/
│   │       └── session_management_port.rs   # Trait to use instead
│   └── dto/
│       ├── queue_items.rs                   # ApprovalItem, ChallengeSuggestionInfo
│       └── app_events.rs                    # AppEvent enum
├── domain/
│   ├── entities/
│   │   ├── challenge.rs                     # Challenge entity
│   │   └── skill.rs                         # Skill entity
│   └── value_objects/
│       ├── ids.rs                           # WARNING: has serde
│       └── game_tools.rs                    # GameTool enum
└── infrastructure/
    ├── websocket/
    │   └── messages.rs                      # ServerMessage, ClientMessage
    ├── websocket.rs                         # WebSocket handler
    └── session.rs                           # SessionManager
```

### Player Critical Files

```
/home/otto/repos/WrldBldr/Player/src/
├── application/
│   ├── dto/
│   │   └── websocket_messages.rs            # Mirrored messages
│   └── ports/
│       └── outbound/
│           └── platform.rs                  # WARNING: cfg in application
├── presentation/
│   ├── views/
│   │   └── world_select.rs                  # VIOLATION: HttpClient
│   ├── services.rs                          # VIOLATION: HttpClient
│   ├── handlers/
│   │   └── session_message_handler.rs       # Message handler
│   └── components/
│       └── dm_panel/
│           ├── approval_popup.rs            # Needs editing capability
│           └── adhoc_challenge_modal.rs     # Needs backend wiring
└── domain/
    └── entities/
        ├── player_action.rs                 # WARNING: has serde
        └── approval.rs                      # WARNING: has serde
```

---

## Appendix B: Environment

- **Platform**: NixOS
- **Rust**: Latest stable
- **Database**: Neo4j 5.x
- **LLM**: Ollama with qwen3-vl model
- **Asset Generation**: ComfyUI

### Development Commands

```bash
# Engine
cd /home/otto/repos/WrldBldr/Engine
nix-shell -p rustc cargo gcc pkg-config openssl.dev --run "cargo check"
nix-shell -p rustc cargo gcc pkg-config openssl.dev --run "cargo build"

# Player
cd /home/otto/repos/WrldBldr/Player
nix-shell -p rustc cargo gcc pkg-config openssl.dev webkitgtk_4_1.dev glib.dev gtk3.dev libsoup_3.dev --run "cargo check"
```
