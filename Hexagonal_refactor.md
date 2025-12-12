# Hexagonal Architecture Refactoring Plan

This document tracks the refactoring work needed to bring both Engine and Player codebases into compliance with hexagonal architecture (ports and adapters) and domain-driven design principles.

## Architecture Overview

### Target Architecture

```
                    ┌─────────────────────────────────────┐
                    │         PRESENTATION LAYER          │
                    │  (Views, Components, HTTP Handlers) │
                    └──────────────────┬──────────────────┘
                                       │ calls
                                       ▼
                    ┌─────────────────────────────────────┐
                    │          INBOUND PORTS              │
                    │    (Use Case Interfaces/Traits)     │
                    └──────────────────┬──────────────────┘
                                       │ implemented by
                                       ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         APPLICATION LAYER                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                      APPLICATION SERVICES                            │  │
│  │  (WorldService, CharacterService, SessionService, LLMService, etc.)  │  │
│  └──────────────────────────────┬──────────────────────────────────────┘  │
│                                 │ uses                                     │
│                                 ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                         DOMAIN LAYER                                 │  │
│  │  Entities: World, Character, Scene, Location, Act, Interaction       │  │
│  │  Value Objects: WorldId, CharacterId, Archetype, Relationship        │  │
│  │  Domain Services: Pure business logic                                │  │
│  │  Aggregates: WorldAggregate (World + Acts + Scenes)                  │  │
│  │  Domain Events: CharacterCreated, SceneTransitioned, etc.            │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                 │ depends on                               │
│                                 ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                        OUTBOUND PORTS                                │  │
│  │  (Repository Traits, External Service Traits)                        │  │
│  │  WorldRepository, CharacterRepository, LlmPort, ComfyUIPort          │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────┘
                                       │ implemented by
                                       ▼
                    ┌─────────────────────────────────────┐
                    │        INFRASTRUCTURE LAYER         │
                    │  (Neo4j, HTTP Client, WebSocket,    │
                    │   Ollama, ComfyUI, localStorage)    │
                    └─────────────────────────────────────┘
```

### Key Principles

1. **Dependency Rule**: Inner layers NEVER depend on outer layers
   - Domain → (nothing external)
   - Application → Domain only
   - Infrastructure → Application + Domain
   - Presentation → Application + Domain (NOT Infrastructure directly)

2. **Ports are Interfaces**: Traits that define capabilities needed by application layer
   - Inbound ports: Use cases exposed to outer layers
   - Outbound ports: Capabilities needed from infrastructure

3. **Adapters Implement Ports**: Infrastructure provides concrete implementations
   - `Neo4jWorldRepository` implements `WorldRepository` trait
   - `OllamaClient` implements `LlmPort` trait

4. **No Infrastructure Types in Domain/Application**
   - Domain types have no serde, no framework attributes
   - Infrastructure DTOs map to/from domain types at boundaries

---

## Phase 1: Engine - Wire Existing Repository Ports

**Priority: CRITICAL**
**Estimated Effort: Medium**

The Engine already has `WorldRepository` trait defined but unused. Wire it through.

### 1.1 Make Neo4jRepository Implement WorldRepository Trait

**File**: `Engine/src/infrastructure/persistence/world_repository.rs`

- [ ] Implement `WorldRepository` trait for `Neo4jWorldRepository`
- [ ] Ensure all trait methods are satisfied
- [ ] Add `#[async_trait]` attribute

### 1.2 Modify Application Services to Use Trait Bounds

**Files**:
- [ ] `Engine/src/application/services/world_service.rs`
- [ ] `Engine/src/application/services/character_service.rs`
- [ ] `Engine/src/application/services/scene_service.rs`
- [ ] `Engine/src/application/services/location_service.rs`

**Changes**:
```rust
// BEFORE (violation)
pub struct WorldServiceImpl {
    repository: Neo4jRepository,
}

// AFTER (correct)
pub struct WorldServiceImpl<R: WorldRepository> {
    repository: R,
}

// Or using trait objects:
pub struct WorldServiceImpl {
    repository: Arc<dyn WorldRepository<Error = anyhow::Error>>,
}
```

### 1.3 Update AppState to Hold Trait Objects

**File**: `Engine/src/infrastructure/state.rs`

- [ ] Change `repository: Neo4jRepository` to trait object
- [ ] Update all state access patterns

### 1.4 Define Missing Repository Traits

**File**: `Engine/src/application/ports/outbound/repository_port.rs`

- [ ] `CharacterRepository` trait
- [ ] `SceneRepository` trait
- [ ] `LocationRepository` trait
- [ ] `ActRepository` trait
- [ ] `InteractionRepository` trait

---

## Phase 2: Engine - Route HTTP Through Services

**Priority: CRITICAL**
**Estimated Effort: Medium**

HTTP handlers currently bypass application services and call repositories directly.

### 2.1 Audit All HTTP Routes

**Files in** `Engine/src/infrastructure/http/`:

| File | Direct Repo Access | Fix Required |
|------|-------------------|--------------|
| `world_routes.rs` | Yes (lines 79-258) | Route through WorldService |
| `character_routes.rs` | Check | TBD |
| `scene_routes.rs` | Check | TBD |
| `location_routes.rs` | Check | TBD |
| `story_arc_routes.rs` | Check | TBD |
| `challenge_routes.rs` | Check | TBD |
| `skill_routes.rs` | Check | TBD |
| `asset_routes.rs` | Check | TBD |

### 2.2 Update Route Handlers

**Pattern**:
```rust
// BEFORE (violation)
pub async fn list_worlds(State(state): State<Arc<AppState>>) -> Result<...> {
    let worlds = state.repository.worlds().list().await?;
    // ...
}

// AFTER (correct)
pub async fn list_worlds(State(state): State<Arc<AppState>>) -> Result<...> {
    let worlds = state.world_service.list_worlds().await?;
    // ...
}
```

### 2.3 Add Missing Service Methods

Ensure application services expose all operations needed by HTTP handlers.

---

## Phase 3: Engine - Remove Infrastructure from Application Layer

**Priority: HIGH**
**Estimated Effort: Medium**

### 3.1 Extract WorldSnapshotBuilder Dependency

**File**: `Engine/src/application/services/world_service.rs`

- [ ] Define `WorldExporter` port trait in outbound ports
- [ ] Move `WorldSnapshotBuilder` to implement this trait
- [ ] Service depends on trait, not concrete builder

### 3.2 Extract Duplicate Parsing Logic

**Files**:
- `Engine/src/infrastructure/http/world_routes.rs` (lines 260-276)
- `Engine/src/infrastructure/persistence/world_repository.rs` (lines 208-222)

- [ ] Implement `FromStr` for `MonomythStage` in domain layer
- [ ] Remove duplicate parsing code

---

## Phase 4: Player - Create Outbound Ports

**Priority: CRITICAL**
**Estimated Effort: High**

The Player has empty port modules. Define proper abstractions.

### 4.1 Define GameConnectionPort

**File**: `Player/src/application/ports/outbound/game_connection.rs`

```rust
#[async_trait]
pub trait GameConnectionPort: Send + Sync {
    async fn connect(&self, url: &str) -> Result<(), ConnectionError>;
    async fn disconnect(&self) -> Result<(), ConnectionError>;
    async fn send_message(&self, msg: GameMessage) -> Result<(), ConnectionError>;
    fn is_connected(&self) -> bool;
}
```

### 4.2 Define ApiPort

**File**: `Player/src/application/ports/outbound/api_port.rs`

```rust
#[async_trait]
pub trait ApiPort: Send + Sync {
    // World operations
    async fn list_worlds(&self) -> Result<Vec<WorldSummary>, ApiError>;
    async fn get_world(&self, id: &str) -> Result<WorldSnapshot, ApiError>;
    async fn create_world(&self, request: CreateWorldRequest) -> Result<World, ApiError>;

    // Character operations
    async fn list_characters(&self, world_id: &str) -> Result<Vec<Character>, ApiError>;
    async fn get_character(&self, id: &str) -> Result<Character, ApiError>;
    async fn save_character(&self, character: &Character) -> Result<Character, ApiError>;

    // Similar for Location, Scene, Skill, Challenge, Asset, etc.
}
```

### 4.3 Have HttpClient Implement ApiPort

**File**: `Player/src/infrastructure/http_client.rs`

- [ ] Implement `ApiPort` trait
- [ ] Keep low-level HTTP methods private
- [ ] Export only the trait implementation

### 4.4 Have EngineClient Implement GameConnectionPort

**File**: `Player/src/infrastructure/websocket/mod.rs`

- [ ] Implement `GameConnectionPort` trait
- [ ] Define `GameMessage` enum in application/domain layer (not infrastructure)

---

## Phase 5: Player - Create Application Services

**Priority: HIGH**
**Estimated Effort: High**

Views currently call HttpClient directly. Create proper services.

### 5.1 Create WorldService

**File**: `Player/src/application/services/world_service.rs`

```rust
pub struct WorldService<A: ApiPort> {
    api: A,
}

impl<A: ApiPort> WorldService<A> {
    pub async fn list_worlds(&self) -> Result<Vec<WorldSummary>, ServiceError>;
    pub async fn load_world(&self, id: &str) -> Result<WorldSnapshot, ServiceError>;
    pub async fn create_world(&self, request: CreateWorldRequest) -> Result<World, ServiceError>;
}
```

### 5.2 Create CharacterService

**File**: `Player/src/application/services/character_service.rs`

### 5.3 Create LocationService

**File**: `Player/src/application/services/location_service.rs`

### 5.4 Create SkillService

**File**: `Player/src/application/services/skill_service.rs`

### 5.5 Create ChallengeService

**File**: `Player/src/application/services/challenge_service.rs`

### 5.6 Create AssetService

**File**: `Player/src/application/services/asset_service.rs`

---

## Phase 6: Player - Fix Application/Presentation Boundary

**Priority: CRITICAL**
**Estimated Effort: High**

### 6.1 Remove Infrastructure Imports from session_service.rs

**File**: `Player/src/application/services/session_service.rs`

- [ ] Remove imports of `SessionState`, `GameState`, `DialogueState`
- [ ] Return domain events/DTOs instead of mutating presentation state
- [ ] Presentation layer handles state updates based on returned events

### 6.2 Make ActionService Depend on Port

**File**: `Player/src/application/services/action_service.rs`

```rust
// BEFORE (violation)
pub struct ActionService {
    client: Arc<EngineClient>,
}

// AFTER (correct)
pub struct ActionService<C: GameConnectionPort> {
    connection: C,
}
```

---

## Phase 7: Player - Fix Presentation/Infrastructure Boundary

**Priority: HIGH**
**Estimated Effort: High**

### 7.1 Remove Infrastructure Types from Presentation State

**Files**:
- [ ] `Player/src/presentation/state/session_state.rs`
- [ ] `Player/src/presentation/state/game_state.rs`
- [ ] `Player/src/presentation/state/dialogue_state.rs`

**Changes**:
- Define presentation-specific types or use domain types
- Remove `EngineClient` from `SessionState`
- Replace `ProposedTool`, `ChallengeSuggestionInfo` with domain/presentation types

### 7.2 Update View Components to Use Services

**Files to update**:
- [ ] `Player/src/presentation/views/dm_view.rs`
- [ ] `Player/src/presentation/views/world_select.rs`
- [ ] `Player/src/presentation/components/creator/entity_browser.rs`
- [ ] `Player/src/presentation/components/creator/character_form.rs`
- [ ] `Player/src/presentation/components/creator/location_form.rs`
- [ ] `Player/src/presentation/components/creator/suggestion_button.rs`
- [ ] `Player/src/presentation/components/creator/asset_gallery.rs`
- [ ] `Player/src/presentation/components/settings/skills_panel.rs`
- [ ] `Player/src/presentation/components/settings/workflow_*.rs`
- [ ] `Player/src/presentation/components/dm_panel/challenge_library.rs`
- [ ] `Player/src/presentation/components/story_arc/*.rs`

**Pattern**:
```rust
// BEFORE (violation)
async fn fetch_characters(world_id: &str) -> Result<Vec<Character>, String> {
    HttpClient::get(&format!("/api/worlds/{}/characters", world_id)).await
}

// AFTER (correct) - inject service via context
let character_service = use_context::<CharacterService>();
let characters = character_service.list_characters(world_id).await;
```

---

## Phase 8: Player - Enrich Domain Layer

**Priority: MEDIUM**
**Estimated Effort: Medium**

### 8.1 Move Business Types to Domain Layer

**Current location**: `Player/src/infrastructure/websocket/messages.rs`, `Player/src/infrastructure/asset_loader/`

**Target location**: `Player/src/domain/entities/`

Types to move:
- [ ] `Character` → `domain/entities/character.rs`
- [ ] `Scene` → `domain/entities/scene.rs`
- [ ] `Location` → `domain/entities/location.rs`
- [ ] `World` → `domain/entities/world.rs`
- [ ] `Dialogue`, `DialogueChoice` → `domain/entities/dialogue.rs`
- [ ] `Interaction` → `domain/entities/interaction.rs`

### 8.2 Create Domain Value Objects

**File**: `Player/src/domain/value_objects/`

- [ ] `WorldId`, `CharacterId`, `SceneId`, `LocationId`
- [ ] Keep infrastructure types as DTOs that map to/from domain types

---

## Phase 9: Engine - Implement Missing DDD Patterns

**Priority: LOW**
**Estimated Effort: Medium**

### 9.1 Define Inbound Ports (Use Cases)

**File**: `Engine/src/application/ports/inbound/`

```rust
// use_cases.rs
#[async_trait]
pub trait ManageWorldUseCase {
    async fn create_world(&self, request: CreateWorldRequest) -> Result<World>;
    async fn list_worlds(&self) -> Result<Vec<WorldSummary>>;
    async fn get_world(&self, id: WorldId) -> Result<World>;
}

#[async_trait]
pub trait PlayGameUseCase {
    async fn join_session(&self, world_id: WorldId, user: User) -> Result<Session>;
    async fn submit_action(&self, session_id: SessionId, action: PlayerAction) -> Result<ActionResult>;
}
```

### 9.2 Define Domain Events

**File**: `Engine/src/domain/events/`

```rust
pub enum DomainEvent {
    WorldCreated { world_id: WorldId },
    CharacterCreated { character_id: CharacterId, world_id: WorldId },
    SceneTransitioned { from: SceneId, to: SceneId },
    DialogueProgressed { character_id: CharacterId, dialogue: Dialogue },
}
```

### 9.3 Define Aggregates

**File**: `Engine/src/domain/aggregates/`

```rust
pub struct WorldAggregate {
    world: World,
    acts: Vec<Act>,
    scenes: Vec<Scene>,
    characters: Vec<Character>,
    locations: Vec<Location>,
}
```

---

## Verification Checklist

After completing all phases, verify:

### Engine
- [ ] No `use crate::infrastructure::` in `application/services/*.rs`
- [ ] No `use crate::infrastructure::` in `domain/**/*.rs`
- [ ] All services use trait bounds or trait objects for dependencies
- [ ] HTTP routes call services, not repositories
- [ ] `cargo check` passes
- [ ] All tests pass

### Player
- [ ] No `use crate::infrastructure::` in `application/services/*.rs`
- [ ] No `use crate::infrastructure::` in `domain/**/*.rs`
- [ ] No `use crate::infrastructure::` in `presentation/**/*.rs` (except through services)
- [ ] Views use services via context, not direct HttpClient calls
- [ ] `cargo check` passes

---

## Progress Tracking

| Phase | Description | Status | Completion Date |
|-------|-------------|--------|-----------------|
| 1 | Engine - Wire Repository Ports | Not Started | |
| 2 | Engine - Route HTTP Through Services | Not Started | |
| 3 | Engine - Remove Infra from Application | Not Started | |
| 4 | Player - Create Outbound Ports | Not Started | |
| 5 | Player - Create Application Services | Not Started | |
| 6 | Player - Fix App/Presentation Boundary | Not Started | |
| 7 | Player - Fix Presentation/Infra Boundary | Not Started | |
| 8 | Player - Enrich Domain Layer | Not Started | |
| 9 | Engine - Implement DDD Patterns | Not Started | |

---

## References

- [Hexagonal Architecture (Alistair Cockburn)](https://alistair.cockburn.us/hexagonal-architecture/)
- [Domain-Driven Design (Eric Evans)](https://www.domainlanguage.com/ddd/)
- [Clean Architecture (Robert C. Martin)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
