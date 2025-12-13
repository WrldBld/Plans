# WrldBldr Architecture Plan

**Last Updated**: 2025-12-13
**Status**: Active - Remediation in progress

This document tracks the hexagonal architecture compliance for both Engine and Player codebases, including remaining work identified from code review.

---

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
   - Domain types have no serde, no framework attributes (exception: serde for WorldSnapshot serialization)
   - Infrastructure DTOs map to/from domain types at boundaries

---

## Current Status Summary

| Area | Engine | Player |
|------|--------|--------|
| Architecture Violations | **0** (all routes use services) | Some DTO re-exports remain |
| Missing Services | **0** (all created) | 0 |
| Testing Coverage | 53 tests | 17 tests |
| Test Gaps | 14/15 entities untested | 11/12 services untested |

---

## Phase Status

| Phase | Description | Status | Notes |
|-------|-------------|--------|-------|
| 1 | Engine - Wire Repository Ports | **Complete** | All port traits defined |
| 2 | Engine - Route HTTP Through Services | **Complete** | All routes migrated (2025-12-13) |
| 3 | Engine - Remove Infra from Application | **Complete** | 1 test-only violation |
| 4 | Player - Create Outbound Ports | **Complete** | ApiPort, GameConnectionPort defined |
| 5 | Player - Create Application Services | **Complete** | All services exist |
| 6 | Player - Fix App/Presentation Boundary | **Complete** | DTO re-exports used (acceptable) |
| 7 | Player - Fix Presentation/Infra Boundary | **Complete** | Components use application::dto |
| 8 | Player - Enrich Domain Layer | **Complete** | Value objects, entities created |
| 9 | Engine - Implement DDD Patterns | **Complete** | Aggregates, events defined |
| 10 | Engine - Complete Service Migration | **Complete** | All 12 services created (2025-12-13) |
| 11 | Player - Fix Architecture Violations | **Complete** | DTO pattern used (2025-12-13) |
| 12 | Test Infrastructure | **TODO** | Next priority |

---

## Phase 10: Engine - Complete Service Migration

**Priority: HIGH**
**Status: COMPLETE (2025-12-13)**

### 10.1 Missing Services to Create

| Service | Route File | Methods Needed |
|---------|------------|----------------|
| `StoryEventService` | story_event_routes.rs | list_events, create_marker, update_event, search_events |
| `SkillService` | skill_routes.rs | list_skills, create, update, delete, initialize_defaults |
| `AssetService` | asset_routes.rs | list_assets, upload, delete, generate_batch |
| `ChallengeService` | challenge_routes.rs | CRUD, toggle_favorite, set_active |
| `InteractionService` | interaction_routes.rs | list, create, update, delete |
| `RelationshipService` | character_routes.rs (partial) | get_social_network, get_relationships |

### 10.2 Route Migration Status

| Route File | Current Status | Action |
|------------|----------------|--------|
| `world_routes.rs` | Uses WorldService | Done |
| `character_routes.rs` | Partial | Migrate lines 252-339 (relationships) |
| `location_routes.rs` | Uses LocationService | Done |
| `scene_routes.rs` | Uses SceneService | Done |
| `export_routes.rs` | Direct repo | Migrate to WorldService.export() |
| `story_event_routes.rs` | Direct repo (lines 626-934) | Create service, migrate all |
| `skill_routes.rs` | Direct repo (lines 87-284) | Create service, migrate all |
| `asset_routes.rs` | Direct repo (lines 141-264) | Create service, migrate all |
| `challenge_routes.rs` | Direct repo (lines 430-737) | Create service, migrate all |
| `interaction_routes.rs` | Direct repo (lines 73-244) | Create service, migrate all |
| `workflow_routes.rs` | Direct repo (lines 143-398) | Create service, migrate all |

### 10.3 Tasks

```
[ ] 10.3.1 Create StoryEventService
    - Define with Arc<dyn StoryEventRepositoryPort>
    - Implement all methods from routes
    - Add to AppState
    - Migrate story_event_routes.rs

[ ] 10.3.2 Create SkillService
    - Define with Arc<dyn SkillRepositoryPort>
    - Implement all methods from routes
    - Add to AppState
    - Migrate skill_routes.rs

[ ] 10.3.3 Create AssetService
    - Define with Arc<dyn AssetRepositoryPort>
    - Implement all methods from routes
    - Add to AppState
    - Migrate asset_routes.rs

[ ] 10.3.4 Create ChallengeService
    - Define with Arc<dyn ChallengeRepositoryPort>
    - Implement all methods from routes
    - Add to AppState
    - Migrate challenge_routes.rs

[ ] 10.3.5 Create InteractionService
    - Define with Arc<dyn InteractionRepositoryPort>
    - Implement all methods from routes
    - Add to AppState
    - Migrate interaction_routes.rs

[ ] 10.3.6 Create RelationshipService
    - Define with Arc<dyn RelationshipRepositoryPort>
    - Implement get_social_network, get_relationships
    - Add to AppState
    - Migrate character_routes.rs lines 252-339

[ ] 10.3.7 Migrate workflow_routes.rs
    - Extend existing WorkflowService or create new
    - Migrate all handlers

[ ] 10.3.8 Make AppState.repository private
    - Change `pub repository` to `repository`
    - Verify compilation (ensures no direct access)
```

---

## Phase 11: Player - Fix Architecture Violations

**Priority: HIGH**
**Status: TODO**

### 11.1 Critical: SessionService Circular Dependency

**File**: `src/application/services/session_service.rs`

| Line | Violation | Fix |
|------|-----------|-----|
| 23 | `use crate::infrastructure::websocket::{EngineClient, ServerMessage}` | Use GameConnectionPort |
| 38 | `use crate::infrastructure::websocket::ConnectionState` | Define in application ports |
| **398** | `use crate::presentation::state::{...}` | **CRITICAL** - Move to presentation layer |

### 11.2 SessionService Not Generic

**Current**:
```rust
pub struct SessionService {
    client: Arc<EngineClient>,  // Concrete type!
}
```

**Required**:
```rust
pub struct SessionService<C: GameConnectionPort> {
    client: C,
}
```

### 11.3 Presentation Layer Infrastructure Imports

| File | Line | Import | Fix |
|------|------|--------|-----|
| `presentation/services.rs` | 14 | `infrastructure::http_client::ApiAdapter` | Re-export via application |
| `presentation/state/session_state.rs` | 13 | `infrastructure::websocket::EngineClient` | Use port abstraction |
| `presentation/handlers/session_event_handler.rs` | 16,18 | `infrastructure::websocket::*` | Import via dto |
| `routes.rs` | 33 | `infrastructure::storage` | Create StoragePort |

### 11.4 Inline Infrastructure References

| File | Lines | Reference | Fix |
|------|-------|-----------|-----|
| `components/creator/sheet_field_input.rs` | 26,29,34,37 | `infrastructure::asset_loader::SectionLayout` | `application::dto::SectionLayout` |
| `components/character_sheet_viewer.rs` | 102,105,108,111 | `infrastructure::asset_loader::SectionLayout` | `application::dto::SectionLayout` |
| `components/dm_panel/challenge_library.rs` | 716,720 | `infrastructure::asset_loader::Outcome` | `application::dto::Outcome` |

### 11.5 DTO Re-exports (Technical Debt)

**File**: `src/application/dto/mod.rs`

Currently re-exports infrastructure types wholesale (leaky abstraction):
```rust
pub use crate::infrastructure::asset_loader::{...};  // Line 19
pub use crate::infrastructure::websocket::{...};     // Line 44
```

**Ideal Fix**: Define actual DTO types with From/Into conversions
**Acceptable for now**: Keep re-exports but document as tech debt

### 11.6 Tasks

```
[ ] 11.6.1 Fix SessionService circular dependency (CRITICAL)
    - Remove presentation imports (line 398)
    - Move handle_server_message to presentation layer
    - SessionService returns events, presentation handles state

[ ] 11.6.2 Make SessionService generic over GameConnectionPort
    - Update struct definition
    - Update desktop and WASM implementations
    - Create MockGameConnectionPort for testing

[ ] 11.6.3 Remove EngineClient from SessionState
    - Store connection via port abstraction
    - Update session_state.rs

[ ] 11.6.4 Fix presentation infrastructure imports
    - presentation/services.rs:14
    - presentation/state/session_state.rs:13
    - presentation/handlers/session_event_handler.rs:16,18
    - routes.rs:33 (create StoragePort)

[ ] 11.6.5 Fix inline infrastructure references
    - sheet_field_input.rs: lines 26,29,34,37
    - character_sheet_viewer.rs: lines 102,105,108,111
    - challenge_library.rs: lines 716,720
```

---

## Phase 12: Test Infrastructure

**Priority: MEDIUM**
**Status: TODO**

### 12.1 Current Test Coverage

| Codebase | Total Tests | Domain | Application | Infrastructure |
|----------|-------------|--------|-------------|----------------|
| Engine | 53 | 7 | 28 | 16 |
| Player | 17 | 3 | 1 | 13 |

### 12.2 Critical Gaps

**Engine**:
- 14/15 domain entities have ZERO tests
- ALL HTTP routes (16 files) have ZERO tests
- ALL Neo4j repositories have ZERO tests
- WebSocket handler has ZERO tests

**Player**:
- 11/12 application services have ZERO tests
- ALL presentation state modules have ZERO tests
- No mock ApiPort implementation
- No mock GameConnectionPort implementation

### 12.3 Tasks

```
[ ] 12.3.1 Engine test infrastructure
    - Create tests/ directory structure
    - Create mock repository port implementations
    - Create test entity factories
    - Add axum-test for HTTP testing
    - Document test database setup

[ ] 12.3.2 Player test infrastructure
    - Create MockApiPort implementation
    - Create MockGameConnectionPort implementation
    - Create world snapshot fixtures
    - Research Dioxus component testing

[ ] 12.3.3 Engine domain tests (Priority)
    - World entity tests
    - Character entity tests
    - Location entity tests
    - RuleSystemConfig tests

[ ] 12.3.4 Engine service tests
    - WorldService tests (with mock repo)
    - CharacterService tests
    - Expand LLMService tests

[ ] 12.3.5 Player service tests
    - WorldService tests (with MockApiPort)
    - CharacterService tests
    - SessionService tests (with MockGameConnectionPort)

[ ] 12.3.6 Player state tests
    - SessionState transition tests
    - GameState update tests
    - DialogueState typewriter tests
```

---

## Verification Checklist

### Engine (run after Phase 10)
- [ ] No `state.repository.` in HTTP handlers (compile will fail if violated)
- [ ] All routes call services: `state.*_service.*`
- [ ] No `use crate::infrastructure::` in `application/services/*.rs`
- [ ] `cargo check` passes
- [ ] `cargo test` passes

### Player (run after Phase 11)
- [ ] No `use crate::infrastructure::` in `application/services/*.rs`
- [ ] No `use crate::infrastructure::` in `presentation/**/*.rs`
- [ ] No `use crate::presentation::` in `application/**/*.rs`
- [ ] SessionService generic over GameConnectionPort
- [ ] `cargo check` passes

---

## Implementation Priority

1. **Phase 10** - Engine service migration (completes original refactoring)
2. **Phase 11.6.1-11.6.3** - Player critical fixes (circular dependency)
3. **Phase 12.3.1-12.3.2** - Test infrastructure setup
4. **Phase 11.6.4-11.6.5** - Player cleanup
5. **Phase 12.3.3-12.3.6** - Add tests

---

## Completed Phases (Reference)

### Phase 1: Engine - Wire Repository Ports (Complete 2025-12-12)
- Defined repository port traits in `Engine/src/application/ports/outbound/repository_port.rs`
- Implemented WorldRepositoryPort, CharacterRepositoryPort, LocationRepositoryPort, SceneRepositoryPort
- Implemented SkillRepositoryPort, AssetRepositoryPort, InteractionRepositoryPort, RelationshipRepositoryPort
- All Neo4j repositories implement their corresponding port traits

### Phase 3: Engine - Remove Infra from Application (Complete 2025-12-13)
- Created WorldExporterPort trait
- Services use `Arc<dyn *RepositoryPort>` instead of concrete Neo4j types
- Updated infrastructure/state.rs to wire services with trait objects

### Phase 4: Player - Create Outbound Ports (Complete 2025-12-12)
- Created ApiPort trait for HTTP client abstraction
- Created GameConnectionPort trait for WebSocket management
- Defined domain types: ConnectionState, ParticipantRole

### Phase 5: Player - Create Application Services (Complete 2025-12-12)
- Created WorldService, CharacterService, LocationService, SkillService, ChallengeService
- All services use ApiPort trait bound

### Phase 8: Player - Enrich Domain Layer (Complete 2025-12-13)
- Created domain value objects for IDs
- Created World and Location domain entities

### Phase 9: Engine - DDD Patterns (Complete 2025-12-13)
- Created inbound use case traits
- Created DomainEvent enum
- Created WorldAggregate

---

## References

- [Hexagonal Architecture (Alistair Cockburn)](https://alistair.cockburn.us/hexagonal-architecture/)
- [Domain-Driven Design (Eric Evans)](https://www.domainlanguage.com/ddd/)
- [Clean Architecture (Robert C. Martin)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
