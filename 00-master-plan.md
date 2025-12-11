# WrldBldr - Master Implementation Plan

## Project Overview

WrldBldr is a TTRPG management system consisting of two applications:
- **Engine**: World creation tool for DMs (Rust backend with Axum + Neo4j + Ollama + ComfyUI)
- **Player**: Gameplay client with PC and DM views (Dioxus frontend - Desktop, WASM, Android)

## Architecture Decisions

- **Hexagonal architecture** (ports and adapters) for both Engine and Player
- **JSON over WebSockets** for Engine ↔ Player communication
- **Neo4j** for graph-based world data
- **Ollama** (qwen3-vl:30b) for AI-assisted NPC responses
- **ComfyUI** for generative asset creation
- **System-agnostic rule engine** for flexible TTRPG support

---

# PART 1: ENGINE SPECIFICATION

## Purpose
The Engine is where DMs build their worlds - defining locations, characters, relationships, story arcs, and game rules before play begins.

## Technical Stack
- **Language**: Rust
- **Backend**: Axum (REST API + WebSocket)
- **Database**: Neo4j (graph database)
- **Communication**: JSON over WebSockets (to Player)

## Domain Model

### Core Entities

#### World
The top-level container for a campaign setting.
```rust
struct World {
    id: WorldId,
    name: String,
    description: String,
    rule_system: RuleSystemConfig,
    created_at: DateTime,
}
```

#### Act / Story Arc
Campbell's monomyth structure for narrative organization.
```rust
struct Act {
    id: ActId,
    world_id: WorldId,
    name: String,
    stage: MonomythStage,  // Ordinary World, Call to Adventure, etc.
    scenes: Vec<SceneId>,
}

enum MonomythStage {
    OrdinaryWorld, CallToAdventure, RefusalOfTheCall, MeetingTheMentor,
    CrossingTheThreshold, TestsAlliesEnemies, ApproachToInnermostCave,
    Ordeal, Reward, TheRoadBack, Resurrection, ReturnWithElixir,
}
```

#### Scene
Complete storytelling unit (location + time + events/dialogue).
```rust
struct Scene {
    id: SceneId,
    act_id: ActId,
    name: String,
    location: LocationId,
    time_context: TimeContext,
    backdrop: AssetRef,
    entry_conditions: Vec<Condition>,
    available_interactions: Vec<InteractionTemplate>,
    directorial_notes: String,
}
```

#### Location
Physical or conceptual places in the world. Locations form a hierarchy - a Town contains a Bar, the Bar contains rooms.
```rust
struct Location {
    id: LocationId,
    world_id: WorldId,
    parent_id: Option<LocationId>,  // Hierarchical containment (Bar inside Town)
    name: String,
    description: String,
    location_type: LocationType,    // Interior, Exterior, Abstract
    backdrop_asset: Option<AssetRef>,
    grid_map_id: Option<GridMapId>,
    backdrop_regions: Vec<BackdropRegion>,
}

struct BackdropRegion {
    id: String,
    name: String,
    bounds: RegionBounds,        // (x, y, width, height)
    backdrop_asset: AssetRef,
    description: Option<String>,
}

struct LocationConnection {
    from_location: LocationId,
    to_location: LocationId,
    connection_type: SpatialRelationship,
    description: String,
    bidirectional: bool,
    requirements: Vec<ConnectionRequirement>,
    travel_time: Option<u32>,
}

enum SpatialRelationship {
    ConnectsTo,   // Standard connection (door, path, road)
    Enters,       // Going inside (Town -> Bar)
    Exits,        // Going outside (Bar -> Town)
    LeadsTo,      // Travel/transition
    AdjacentTo,   // Shared border
}
```

#### Character (NPC)
```rust
struct Character {
    id: CharacterId,
    name: String,
    description: String,
    sprite_asset: AssetRef,
    portrait_asset: AssetRef,
    base_archetype: CampbellArchetype,
    current_archetype: CampbellArchetype,
    archetype_history: Vec<ArchetypeChange>,
    wants: Vec<Want>,
    relationships: Vec<RelationshipId>,
    stats: StatBlock,
    inventory: Vec<ItemId>,
}

enum CampbellArchetype {
    Hero, Mentor, ThresholdGuardian, Herald,
    Shapeshifter, Shadow, Trickster, Ally,
}

struct Want {
    description: String,
    target: Option<ActantTarget>,
    intensity: f32,
    known_to_player: bool,
}
```

#### Relationship (Graph Edge)
```rust
struct Relationship {
    id: RelationshipId,
    from_character: CharacterId,
    to_character: CharacterId,
    relationship_type: RelationshipType,
    sentiment: f32,           // -1.0 (hatred) to 1.0 (love)
    history: Vec<RelationshipEvent>,
    known_to_player: bool,
}
```

#### GridMap (Tactical Combat)
```rust
struct GridMap {
    id: GridMapId,
    name: String,
    width: u32,
    height: u32,
    tilesheet: AssetRef,
    tiles: Vec<Vec<Tile>>,
}

struct Tile {
    terrain_type: TerrainType,
    elevation: i32,
    tile_index: u32,
    passable: bool,
    cover_value: Option<u8>,
}
```

### Neo4j Graph Schema

```cypher
// Nodes
(:World {id, name, description})
(:Act {id, name, stage})
(:Scene {id, name, time_context, directorial_notes})
(:Location {id, name, description, parent_id, location_type, backdrop_asset, grid_map_id, backdrop_regions})
(:Character {id, name, base_archetype, current_archetype})
(:Item {id, name, description})
(:Event {id, description, timestamp})

// Relationships
(:World)-[:CONTAINS_ACT]->(:Act)
(:World)-[:CONTAINS_LOCATION]->(:Location)
(:Act)-[:CONTAINS_SCENE]->(:Scene)
(:Scene)-[:TAKES_PLACE_AT]->(:Location)
(:Scene)-[:FEATURES]->(:Character)
(:Location)-[:CONNECTS_TO {connection_type, travel_time, requirements, bidirectional}]->(:Location)
(:Character)-[:RELATES_TO {type, sentiment, history}]->(:Character)
(:Character)-[:WANTS {intensity, description}]->(:Character|:Item|:Goal)
(:Character)-[:POSSESSES]->(:Item)
```

## Hexagonal Architecture (Engine)

The Engine is a **backend-only server** (Axum + WebSocket). No UI - the Player application provides the frontend for both PCs and DMs.

```
Engine/
├── src/
│   ├── main.rs               # Axum server entry point
│   ├── domain/               # Core business logic (no external deps)
│   │   ├── entities/         # World, Scene, Character, etc.
│   │   ├── value_objects/    # Archetype, Want, Relationship types
│   │   ├── aggregates/       # World aggregate root
│   │   └── services/         # Domain services
│   ├── application/          # Use cases / orchestration
│   │   ├── ports/
│   │   │   ├── inbound/      # WorldService, SceneService
│   │   │   └── outbound/     # Repository traits, LLMPort
│   │   ├── services/
│   │   └── dto/
│   └── infrastructure/       # External adapters
│       ├── persistence/      # Neo4j adapter
│       ├── websocket/        # WebSocket server
│       ├── http/             # REST API routes
│       ├── ollama/           # LLM client
│       ├── comfyui/          # Asset generation
│       ├── asset_manager/    # Asset storage
│       └── export/           # World export
```

## Engine User Stories (MVP)

### World Management
- [ ] US-E001: Create world with name and description
- [ ] US-E002: Configure rule system (stats, dice, formulas)
- [ ] US-E003: Save and load worlds

### Location Building
- [ ] US-E010: Create locations with backdrops
- [ ] US-E011: Connect locations with spatial relationships (enters, exits, connects)
- [ ] US-E012: Create hierarchical locations (Town contains Bar contains Rooms)
- [ ] US-E013: Define backdrop regions within a location
- [ ] US-E014: Create tactical grid maps from tilesheets
- [ ] US-E015: Paint elevation levels on grid tiles
- [ ] US-E016: Set connection requirements (item needed, scene completed)

### Character Creation
- [ ] US-E020: Create NPCs with sprites and portraits
- [ ] US-E021: Assign Campbell archetypes to characters
- [ ] US-E022: Define character "Wants" (actantial goals)
- [ ] US-E023: Create relationships between characters
- [ ] US-E024: View social network as graph

### Scene Authoring
- [ ] US-E030: Create scenes at locations
- [ ] US-E031: Write directorial notes for LLM guidance
- [ ] US-E032: Define scene entry conditions
- [ ] US-E033: Organize scenes into acts (monomyth stages)

### Export & Serve
- [ ] US-E040: Export world as JSON for Player
- [ ] US-E041: Host game session via WebSocket

---

# PART 2: PLAYER SPECIFICATION

## Purpose
The Player app provides two distinct views:
1. **PC View**: Visual novel gameplay experience for players
2. **DM View**: Directorial control panel for running the game

## Key Innovation: AI-Assisted Directing

The DM doesn't directly roleplay NPCs. Instead:
1. PC performs an action (talks to NPC, interacts with object)
2. Action + DM's directorial notes sent to Engine
3. Engine calls LLM (qwen3-vl:30b multimodal) with context
4. LLM generates response + potential tool calls (give item, change state)
5. DM receives approval popup before execution
6. DM can accept OR provide feedback for regeneration
7. Approved response shown to PC

## Domain Model

### Session State
```rust
struct GameSession {
    id: SessionId,
    world_snapshot: WorldSnapshot,
    current_scene: SceneId,
    participants: Vec<Participant>,
    game_state: GameState,
    event_log: Vec<GameEvent>,
}

struct Participant {
    id: ParticipantId,
    user_id: UserId,
    role: ParticipantRole,  // DungeonMaster, Player, Spectator
    character: Option<PlayerCharacter>,
}
```

### PC Interaction
```rust
struct PlayerAction {
    player_id: ParticipantId,
    action_type: ActionType,  // Speak, Examine, UseItem, Move, Attack
    target: Option<InteractionTarget>,
    dialogue: Option<String>,
}
```

### DM Directorial System
```rust
struct DirectorialContext {
    scene_notes: String,
    dm_live_notes: String,
    character_motivations: Vec<CharacterMotivation>,
    tone_guidance: ToneGuidance,
    forbidden_topics: Vec<String>,
    allowed_tool_calls: Vec<ToolPermission>,
}
```

### LLM Integration
```rust
struct LLMRequest {
    conversation_history: Vec<ConversationTurn>,
    player_action: PlayerAction,
    directorial_context: DirectorialContext,
    available_tools: Vec<ToolDefinition>,
    scene_image: Option<ImageData>,
}

struct LLMResponse {
    npc_dialogue: String,
    internal_reasoning: String,      // Hidden from PC, shown to DM
    proposed_tool_calls: Vec<ProposedToolCall>,
    suggested_next_beats: Vec<String>,
}
```

### DM Approval Flow
```rust
enum ApprovalDecision {
    Accept,
    AcceptWithModification { modified_dialogue: String, approved_tools: Vec<ToolCallId> },
    Reject { feedback: String },
    TakeOver { dm_response: String },
}
```

### LLM Tool Definitions
```rust
enum GameTool {
    GiveItem { recipient: CharacterId, item: ItemId },
    TakeItem { from: CharacterId, item: ItemId },
    UpdateRelationship { character_a: CharacterId, character_b: CharacterId, change: RelationshipChange },
    TriggerEvent { event_type: String, details: String },
    ChangeScene { new_scene: SceneId },
    StartCombat { participants: Vec<CharacterId>, map: GridMapId },
    RevealInformation { to: CharacterId, information: String },
    UpdateCharacterState { character: CharacterId, state_change: StateChange },
}
```

## Hexagonal Architecture (Player)

```
Player/
├── src/
│   ├── main.rs
│   ├── domain/
│   │   ├── entities/         # Session, Participant, Action
│   │   ├── value_objects/
│   │   └── services/
│   ├── application/
│   │   ├── ports/
│   │   │   ├── inbound/      # GameService, CombatService
│   │   │   └── outbound/     # EngineConnection, LLMPort
│   │   └── services/
│   ├── infrastructure/
│   │   ├── websocket/        # Engine connection
│   │   ├── audio/            # Sound effects, music
│   │   └── asset_loader/     # Sprites, backdrops, WorldSnapshot
│   └── presentation/
│       ├── components/
│       │   ├── visual_novel/ # Backdrop, dialogue, sprites
│       │   ├── tactical/     # Grid map, units
│       │   ├── dm_panel/     # Directorial controls
│       │   └── shared/
│       ├── views/
│       │   ├── pc_view.rs
│       │   └── dm_view.rs
│       └── state/
```

## Player UI Wireframes

### PC View (Visual Novel Mode)
```
┌─────────────────────────────────────────────────────────────┐
│  [Backdrop Image - Full Screen]                             │
│         ┌─────────┐     ┌─────────┐                         │
│         │  NPC    │     │   PC    │                         │
│         │ Sprite  │     │ Sprite  │                         │
│         └─────────┘     └─────────┘                         │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ NPC Name:                                               │ │
│ │ "Dialogue text appears here, typewriter style..."       │ │
│ │ [Choice 1] [Choice 2] [Custom input...]                 │ │
│ └─────────────────────────────────────────────────────────┘ │
│ [Inventory]  [Character]  [Map]  [Log]                      │
└─────────────────────────────────────────────────────────────┘
```

### DM View (Director Mode)
```
┌───────────────────────────────────────┬─────────────────────┐
│  [Scene Preview]                      │  DIRECTORIAL PANEL  │
│  ┌─────────┐   ┌─────────┐            │  Scene Notes: [...]  │
│  │  NPC    │   │   PC    │            │  NPC Motivations     │
│  └─────────┘   └─────────┘            │  Tone: [Dropdown]   │
├───────────────────────────────────────┤                     │
│  CONVERSATION LOG                     │  ACTIVE NPCs:       │
│  ┌─────────────────────────────────┐  │  [x] Bartender      │
│  │ PC: "Hello there"               │  │  [ ] Guard          │
│  │ [Awaiting LLM response...]      │  │                     │
│  └─────────────────────────────────┘  │  [Social Graph]     │
├───────────────────────────────────────┤                     │
│  ┌─ APPROVAL POPUP ────────────────┐  │                     │
│  │ NPC will say: "..."             │  │                     │
│  │ Proposed: [✓] Reveal info       │  │                     │
│  │ [Accept] [Modify] [Reject]      │  │                     │
│  └─────────────────────────────────┘  │                     │
└───────────────────────────────────────┴─────────────────────┘
```

### Tactical Combat View
```
┌─────────────────────────────────────────────────────────────┐
│  ┌─────────────────────────────────────┐  ┌───────────────┐ │
│  │  GRID MAP                           │  │ TURN ORDER    │ │
│  │  ┌──┬──┬──┬──┬──┬──┬──┬──┐          │  │ > Fighter     │ │
│  │  │  │▲▲│  │  │░░│░░│  │  │ Elev: 2  │  │   Goblin 1    │ │
│  │  ├──┼──┼──┼──┼──┼──┼──┼──┤          │  │   Mage        │ │
│  │  │  │▲▲│PC│  │░░│E1│  │  │ Elev: 1  │  └───────────────┘ │
│  │  └──┴──┴──┴──┴──┴──┴──┴──┘          │  ┌───────────────┐ │
│  │  ▲▲=High ground  ░░=Cover  E=Enemy  │  │ Fighter HP:24 │ │
│  └─────────────────────────────────────┘  │ [Move][Attack]│ │
│  [End Turn]  [Inventory]  [Abilities]     └───────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Player User Stories (MVP)

### Session Management
- [ ] US-P001: Connect to game session via WebSocket
- [ ] US-P002: Select role (DM, Player, Spectator)
- [ ] US-P003: Claim player character slot

### PC View - Visual Novel
- [ ] US-P010: See scene backdrop and character sprites
- [ ] US-P011: Read NPC dialogue with typewriter effect
- [ ] US-P012: Select dialogue choices or type custom responses
- [ ] US-P013: Examine objects and characters
- [ ] US-P014: Access inventory and character sheet

### PC View - Navigation
- [ ] US-P020: Move between connected locations
- [ ] US-P021: See map of known locations
- [ ] US-P022: Trigger scene transitions

### DM View - Directing
- [ ] US-P030: See current scene with PC actions in real-time
- [ ] US-P031: Edit live directorial notes
- [ ] US-P032: Set individual NPC motivations and moods
- [ ] US-P033: Set tone guidance for LLM responses
- [ ] US-P034: Receive approval popups for LLM responses
- [ ] US-P035: Accept, modify, or reject LLM responses
- [ ] US-P036: Take over and write responses manually
- [ ] US-P037: View social network graph
- [ ] US-P038: See NPC internal reasoning (hidden from PCs)

### LLM Integration
- [ ] US-P040: System sends PC actions + context to LLM
- [ ] US-P041: LLM can propose tool calls (give item, change state)
- [ ] US-P042: Tool calls require DM approval
- [ ] US-P043: Rejected responses trigger LLM retry with feedback

### Tactical Combat
- [ ] US-P050: See tactical grid with elevation
- [ ] US-P051: Move character on grid
- [ ] US-P052: Perform attacks and abilities
- [ ] US-P053: DM controls NPC units during combat
- [ ] US-P054: Turn order and movement rules enforced

---

# PART 3: IMPLEMENTATION PHASES

## Phase 1: Foundation ✅ COMPLETED
- [x] Engine Cargo workspace with hexagonal structure
- [x] Player Cargo workspace with Dioxus
- [x] Neo4j connection and Docker setup
- [x] Basic WebSocket protocol types
- [x] Ollama and ComfyUI client stubs

## Phase 2: Core Domain (Engine) ✅ COMPLETED
**Details:** `plans/02-core-domain-engine.md`
- [x] Neo4j repository implementation (CRUD)
- [x] World aggregate with full persistence
- [x] Character with archetype system
- [x] Location and connection graph
- [x] Scene management
- [x] REST API endpoints for all entities

## Phase 3: Scene System (Engine) ✅ COMPLETED
**Details:** `plans/03-scene-system.md`
- [x] InteractionTemplate entity with types and conditions
- [x] DirectorialNotes enhanced with tone, NPC motivations, pacing
- [x] Interaction API endpoints (CRUD + availability)
- [x] World export to JSON (WorldSnapshot)

## Phase 4: Player Basics ✅ COMPLETED
**Details:** `plans/04-player-basics.md`
- [x] WebSocket client (EngineClient) - tokio-tungstenite (desktop) / gloo-net (WASM)
- [x] WorldSnapshot loader with JSON deserialization
- [x] Visual novel renderer (backdrop, sprites, dialogue box, choices)
- [x] PC action input system (dialogue choices, custom input)
- [x] State management with Dioxus 0.7.2 signals

## Phase 5: LLM Integration ✅ COMPLETED
**Details:** `plans/IMPLEMENTATION_PLAN.md` Section 1.1
- [x] Prompt construction from context (LLMService.build_system_prompt_with_notes)
- [x] Tool definition system for game actions (GameTool enum in domain/value_objects)
- [x] Response parsing and validation (LLMService.parse_response)
- [x] Conversation history management (ConversationTurn in session.rs, 30-turn limit)

## Phase 6: DM Directing System ✅ COMPLETED
**Details:** `plans/IMPLEMENTATION_PLAN.md` Section 1.2
- [x] DM View UI completion (approval popup, directorial panel)
- [x] Approval popup flow (ApprovalRequired → ApprovalDecision)
- [x] Accept/Modify/Reject/TakeOver handling (all 4 decision types)
- [x] LLM retry with feedback (3-retry maximum with feedback injection)

## Phase 7: Tactical Combat
- [ ] Grid map renderer
- [ ] Elevation display
- [ ] Unit movement and pathfinding
- [ ] Turn order system

## Phase 8: ComfyUI Asset Generation 🔄 IN PROGRESS
**Details:** `plans/11-director-creation-ui.md`, `plans/12-workflow-settings.md`
- [x] Workflow templates (Engine: WorkflowConfiguration entity, 9 workflow slots)
- [x] Generation queue (Engine: GenerationBatch, ComfyUIClient, REST + WebSocket APIs)
- [ ] Asset preview and selection UI (Player: UI exists, needs API wiring)
- [ ] Style reference system

## Phase 9: Save/Load System
- [ ] Save file format definition
- [ ] World state serialization
- [ ] Session state persistence
- [ ] Auto-save implementation

## Phase 10: Polish & Platforms
- [ ] Mobile responsive UI (Android, Web Mobile)
- [ ] WASM optimization
- [ ] Audio integration (music, SFX)
- [ ] Performance profiling

---

# PART 4: WebSocket Protocol

## Message Types

```rust
// Engine → Player
enum EngineMessage {
    WorldData(WorldSnapshot),
    SceneUpdate(SceneState),
    CharacterUpdate(CharacterState),
    CombatStart(CombatInitData),
    CombatUpdate(CombatState),
    SessionEvent(SessionEvent),
}

// Player → Engine
enum PlayerMessage {
    JoinSession { user_id: UserId, role: ParticipantRole },
    PlayerAction(PlayerAction),
    DirectorialUpdate(DirectorialContext),
    ApprovalDecision(ApprovalDecision),
    CombatAction(CombatAction),
    RequestSceneChange(SceneId),
}

// LLM Flow Messages
enum LLMFlowMessage {
    LLMProcessing { action_id: ActionId },
    ApprovalRequired(ApprovalRequest),
    ResponseApproved { npc_dialogue: String, executed_tools: Vec<ToolResult> },
    ResponseRejected { retry_in_progress: bool },
}
```

---

# PART 5: ComfyUI Integration

## Workflow Types

- **CharacterSprite**: Character description → PNG sprite with transparency
- **CharacterPortrait**: Character + expression → Portrait for dialogue
- **SceneBackdrop**: Location description → Wide backdrop image
- **Tilesheet**: Environment type → Tilesheet with consistent tiles

## User Stories (Asset Generation)
- [ ] US-A001: Generate character sprite from text description
- [ ] US-A002: Generate multiple portraits for NPC (different expressions)
- [ ] US-A003: Generate scene backdrops from location descriptions
- [ ] US-A004: Generate tilesheets for tactical maps
- [ ] US-A005: Queue multiple generation requests
- [ ] US-A006: Preview and select from variations
- [ ] US-A007: Use existing images as style references

---

# PART 6: Save/Load System

## Save File Structure
```rust
struct GameSave {
    metadata: SaveMetadata,   // id, name, created_at, thumbnail
    world_state: WorldState,  // character states, relationships, inventory
    session_state: SessionState,  // conversation, dm_notes, quests
    player_data: Vec<PlayerSaveData>,
}
```

## User Stories (Save/Load)
- [ ] US-S001: Save current game state with a name
- [ ] US-S002: Load previous save and resume
- [ ] US-S003: See list of saves with timestamps and thumbnails
- [ ] US-S004: Delete old saves
- [ ] US-S005: Auto-save periodically during play

---

# Resolved Decisions

1. **Multiplayer**: Multi-PC from start - support multiple players simultaneously
2. **Assets**: Standard PNG images + ComfyUI workflows for generative creation
3. **Persistence**: Full save/load system for game sessions
4. **Voice/Video**: External tools (Discord, etc.) - no integration needed
5. **Offline Mode**: Always online - requires active Engine connection

---

# Key Files Reference

## Engine
- `Engine/src/main.rs` - Axum server entry point
- `Engine/src/domain/entities/` - Core domain models
- `Engine/src/infrastructure/persistence/` - Neo4j repositories
- `Engine/src/infrastructure/http/` - REST API routes
- `Engine/src/infrastructure/websocket.rs` - WebSocket handlers
- `Engine/src/infrastructure/export/` - World export (JSON)

## Player
- `Player/src/main.rs` - Dioxus app entry point
- `Player/src/presentation/views/pc_view.rs` - Player character view
- `Player/src/presentation/views/dm_view.rs` - Dungeon master view
- `Player/src/infrastructure/websocket/` - Engine connection
- `Player/src/infrastructure/asset_loader/` - WorldSnapshot loader

## External Services
- Neo4j: Docker container (bolt://localhost:7687)
- Ollama: 10.8.0.6:11434/v1 (qwen3-vl:30b)
- ComfyUI: 10.8.0.6:8188
