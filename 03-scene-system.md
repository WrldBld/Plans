# Phase 3: Scene System (Engine)

## Objective
Enhance the scene system with entry conditions, interaction templates, expanded directorial notes, and implement world export functionality for the Player to consume.

## Status: ✅ COMPLETE

## Tasks

### 3.1 Scene Entry Conditions
**Status:** ⏳ DEFERRED (basic structure in place)
**Files:**
- `Engine/src/domain/entities/scene.rs` - SceneCondition enum exists
- `Engine/src/domain/entities/interaction.rs` - InteractionCondition enum added

**Subtasks:**
- [x] 3.1.1 Basic condition types defined (HasItem, CharacterPresent, etc.)
- [ ] 3.1.2 Condition evaluation logic (deferred to Player runtime)
- [x] 3.1.3 Store conditions in Neo4j (via JSON serialization)
- [ ] 3.1.4 API endpoints for managing scene conditions (done via interaction conditions)

### 3.2 Interaction Templates
**Status:** ✅ COMPLETE
**Files:**
- `Engine/src/domain/entities/interaction.rs`
- `Engine/src/infrastructure/persistence/interaction_repository.rs`
- `Engine/src/infrastructure/http/interaction_routes.rs`

**Subtasks:**
- [x] 3.2.1 Create InteractionTemplate entity
- [x] 3.2.2 Define interaction types (Dialogue, Examine, UseItem, etc.)
- [x] 3.2.3 Link interactions to scenes
- [x] 3.2.4 Store interaction templates in Neo4j
- [x] 3.2.5 API endpoints for managing interactions

### 3.3 Directorial Notes Enhancement
**Status:** ✅ COMPLETE
**Files:**
- `Engine/src/domain/value_objects/directorial.rs`

**Subtasks:**
- [x] 3.3.1 Create structured DirectorialNotes value object
- [x] 3.3.2 Add tone guidance field (ToneGuidance enum)
- [x] 3.3.3 Add NPC motivation hints (NpcMotivation struct)
- [x] 3.3.4 Add forbidden topics list
- [x] 3.3.5 Add allowed tool calls specification
- [x] 3.3.6 Add to_prompt() method for LLM consumption

### 3.4 World Export
**Status:** ✅ COMPLETE
**Files:**
- `Engine/src/infrastructure/export/mod.rs`
- `Engine/src/infrastructure/export/json_exporter.rs`
- `Engine/src/infrastructure/http/export_routes.rs`

**Subtasks:**
- [x] 3.4.1 Define WorldSnapshot JSON structure
- [x] 3.4.2 Implement world serialization
- [x] 3.4.3 Include all related entities (characters, locations, scenes)
- [x] 3.4.4 Include relationships and social network
- [x] 3.4.5 API endpoint: GET /api/worlds/:id/export
- [x] 3.4.6 Compressed export option: GET /api/worlds/:id/export/raw?format=compressed

## New API Routes

> **Note:** Axum 0.8 uses `{param}` syntax for route parameters.

### Interaction Routes
- `GET /api/scenes/{scene_id}/interactions` - List interactions in scene
- `POST /api/scenes/{scene_id}/interactions` - Create interaction
- `GET /api/interactions/{id}` - Get interaction by ID
- `PUT /api/interactions/{id}` - Update interaction
- `DELETE /api/interactions/{id}` - Delete interaction
- `PUT /api/interactions/{id}/availability` - Toggle availability

### Export Routes
- `GET /api/worlds/{id}/export` - Export world as JSON snapshot
- `GET /api/worlds/{id}/export/raw` - Export as raw JSON string

## New Domain Types

### InteractionTemplate
```rust
struct InteractionTemplate {
    id: InteractionId,
    scene_id: SceneId,
    name: String,
    interaction_type: InteractionType,  // Dialogue, Examine, UseItem, etc.
    target: InteractionTarget,          // Character, Item, Environment, None
    prompt_hints: String,
    allowed_tools: Vec<String>,
    conditions: Vec<InteractionCondition>,
    is_available: bool,
    order: u32,
}
```

### DirectorialNotes (Enhanced)
```rust
struct DirectorialNotes {
    general_notes: String,
    tone: ToneGuidance,           // Serious, Lighthearted, Tense, etc.
    npc_motivations: HashMap<String, NpcMotivation>,
    forbidden_topics: Vec<String>,
    allowed_tools: Vec<String>,
    suggested_beats: Vec<String>,
    pacing: PacingGuidance,       // Natural, Fast, Slow, Building, Urgent
}
```

### WorldSnapshot (Export Format)
```rust
struct WorldSnapshot {
    metadata: SnapshotMetadata,
    world: WorldData,
    acts: Vec<ActData>,
    scenes: Vec<SceneData>,
    characters: Vec<CharacterData>,
    locations: Vec<LocationData>,
    relationships: Vec<RelationshipData>,
    connections: Vec<ConnectionData>,
}
```

## Acceptance Criteria
- [x] Scenes can have interaction templates defining available actions
- [x] Directorial notes include structured guidance for LLM
- [x] World can be exported as complete JSON snapshot
- [x] Export includes all entities and relationships
- [x] Export format ready for Player consumption

## Dependencies
- Phase 2 (Core Domain) ✅

## Notes
- Entry condition evaluation deferred to Player runtime
- Interaction templates guide the LLM on how to handle player actions
- World export is the primary way to "publish" a world for the Player
- DirectorialNotes.to_prompt() provides LLM-ready context
- Axum 0.8 route syntax uses `{param}` instead of `:param`

## Domain Entities Reference
See `plans/00-master-plan.md` PART 1 for complete domain model specifications:
- Scene with entry conditions and available interactions
- DirectorialContext for LLM guidance (tone, NPC motivations, forbidden topics)
- WorldSnapshot export format for Player consumption
