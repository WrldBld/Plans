# Phase 21: Player Character Creation & Starting Location Selection

## Overview

This plan implements the critical missing piece that enables actual gameplay: allowing Player users to create their own characters (PCs) when joining a session, select a starting location, and wire the scene manager to use PC locations for scene determination. This bridges the gap between session joining and actual gameplay participation.

**Status**: [x] **COMPLETE**

**Priority**: **CRITICAL** - Blocks core gameplay features

**Dependencies**:
- ✅ Phase 13 (World Selection) - Players can join sessions
- ✅ Phase 14 (Rule Systems & Challenges) - Character sheet templates available
- ✅ Phase 15 (Routing & Navigation) - Navigation infrastructure
- ✅ Phase 19 (Queue System) - Session-aware queues
- ✅ Anonymous Users & Session Management - Stable user IDs and session flows

**Related Plans**:
- `03-scene-system.md` - Scene system architecture
- `04-player-basics.md` - Player infrastructure
- `13-world-selection.md` - Session joining flows
- `14-rule-systems-and-challenges.md` - Character sheet system

---

## Problem Statement

Currently, the system has:
- ✅ NPCs (`Character` entity) created by DMs in Creator Mode
- ✅ Sessions that players can join
- ✅ Scene system that shows scenes based on location and featured NPCs
- ✅ Character sheet templates for different rule systems
- ❌ **No Player Character (PC) entity** - players have no persistent character representation
- ❌ **No PC creation flow** - players join sessions but can't create their character
- ❌ **No starting location selection** - PCs have no location to start from
- ❌ **Scene manager doesn't account for PC locations** - scenes are determined by featured NPCs, not where PCs are

**Impact**: Players can join sessions but cannot actually participate in gameplay because:
1. They have no character to represent them
2. They have no location to start from
3. The scene manager doesn't know where PCs are, so it can't show appropriate scenes

---

## Solution Architecture

### Core Concept

**Player Characters (PCs)** are distinct from NPCs:
- **NPCs** (`Character` entity): Created by DMs, have archetypes, wants, relationships, controlled by LLM/DM
- **PCs** (`PlayerCharacter` entity): Created by players, have character sheets, current location, owned by `user_id`

**Key Design Decisions**:
1. **Separate Entity**: PCs are a separate entity type, not a variant of `Character`
   - Rationale: PCs have different lifecycle (created by players, tied to sessions), different data (character sheet, location tracking), different permissions
2. **Session-Scoped**: PCs are created per-session, not per-world
   - Rationale: A player might want different characters in different sessions of the same world
3. **Location-Based Scene Resolution**: Scene manager uses PC's current location to determine which scene to show
   - Rationale: Enables location-based gameplay, travel mechanics, scene transitions

### Data Model

```rust
// Engine/src/domain/entities/player_character.rs

/// A player character (PC) - distinct from NPCs
#[derive(Debug, Clone)]
pub struct PlayerCharacter {
    pub id: PlayerCharacterId,
    pub session_id: SessionId,
    pub user_id: String,  // Anonymous user ID from Player
    pub world_id: WorldId,
    
    // Character identity
    pub name: String,
    pub description: Option<String>,
    
    // Character sheet data (matches CharacterSheetData from Phase 14)
    pub sheet_data: Option<CharacterSheetData>,
    
    // Location tracking
    pub current_location_id: LocationId,
    pub starting_location_id: LocationId,  // For reference/history
    
    // Visual assets (optional, can be generated later)
    pub sprite_asset: Option<String>,
    pub portrait_asset: Option<String>,
    
    // Metadata
    pub created_at: DateTime<Utc>,
    pub last_active_at: DateTime<Utc>,
}

/// Player Character ID value object
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct PlayerCharacterId(Uuid);

impl PlayerCharacterId {
    pub fn new() -> Self {
        Self(Uuid::new_v4())
    }
    
    pub fn from_uuid(uuid: Uuid) -> Self {
        Self(uuid)
    }
    
    pub fn to_string(&self) -> String {
        self.0.to_string()
    }
}
```

### Session Integration

```rust
// Engine/src/infrastructure/session.rs

pub struct GameSession {
    // ... existing fields ...
    
    /// Map of user_id -> PlayerCharacter for this session
    pub player_characters: HashMap<String, PlayerCharacter>,
    
    /// Current scene ID (determined by PC locations)
    pub current_scene_id: Option<SceneId>,
}
```

### Scene Resolution Logic

The scene manager will resolve scenes based on:
1. **PC Locations**: Find all PCs in the session, get their current locations
2. **Location-to-Scene Mapping**: For each location, find active scenes at that location
3. **Scene Selection**: 
   - If all PCs are at the same location → show scene for that location
   - If PCs are at different locations → show DM a "split party" view or default to first PC's location
   - If no scene exists for location → show location backdrop with basic interactions

---

## User Stories

### Epic 1: PC Creation Flow

**US-1.1**: As a Player, I want to create my character when joining a session so that I have a representation in the game world.

**Acceptance Criteria**:
- After joining a session as `ParticipantRole::Player`, I see a "Create Character" screen
- I can enter my character's name and description
- I can fill out the character sheet (if the world has a sheet template)
- I can select a starting location from available locations in the world
- My character is saved and associated with my `user_id` and `session_id`
- After creation, I'm taken to the PC View showing my starting location

**US-1.2**: As a Player, I want to see available starting locations so I can choose where my character begins.

**Acceptance Criteria**:
- I see a list of all locations in the world
- Each location shows: name, description, location type (Interior/Exterior/Abstract)
- I can filter locations by type
- I can see which locations have scenes available
- I can select a location as my starting point

**US-1.3**: As a Player, I want to fill out my character sheet based on the world's rule system so my character has proper stats.

**Acceptance Criteria**:
- If the world has a character sheet template, I see the sheet form
- I can fill in all required fields (ability scores, skills, etc.)
- Field types match the template (Number, Text, Checkbox, Select, etc.)
- I can see my character's modifiers calculated automatically
- Sheet data is saved with my character

**US-1.4**: As a Player, I want to skip character creation if I've already created a character for this session.

**Acceptance Criteria**:
- If I already have a PC for this session, I'm taken directly to PC View
- I can see my existing character's name and location
- I can edit my character if needed (name, description, sheet data)

### Epic 2: Location-Based Scene Resolution

**US-2.1**: As a Player, I want to always see the scene at my character's current location so I know where I am in the world.

**Acceptance Criteria**:
- When I join a session, I see the scene at my starting location
- The scene shows: backdrop, location name, featured NPCs (if any)
- If no scene exists for my location, I see the location's default backdrop
- Scene updates automatically when my character travels to a new location
- I always see the scene from my character's perspective (never see other locations unless I'm there)
- Scene resolution is automatic and transparent - I don't need to manually select scenes

**UI Mockup - Player View**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│  PC VIEW - The Dusty Library                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [Backdrop Image: Ancient library with dusty tomes]                    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Location: The Dusty Library                                │       │
│  │  Your Character: Aragorn                                    │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  [NPC Sprites: Librarian (left), Scholar (right)]                      │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Dialogue Box                                                │       │
│  │  Librarian: "Welcome, traveler. What knowledge do you seek?"│       │
│  │  [Choice 1] [Choice 2] [Custom Input...]                    │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  [Actions: Talk] [Examine] [Travel] [Character Sheet]                  │
└─────────────────────────────────────────────────────────────────────────┘
```

**US-2.2**: As a DM, I want to navigate to any location to preview how scenes look so I can verify the player experience.

**Acceptance Criteria**:
- In Director Mode, I have a "Location Navigator" panel
- I can see a list of all locations in the world
- I can click a location to "preview" it - see the scene as players would see it
- Preview shows: backdrop, location name, any NPCs at that location
- Preview mode is clearly indicated ("Previewing: The Dusty Library")
- I can exit preview mode to return to my normal DM view
- Preview doesn't affect the actual game state

**UI Mockup - DM Location Navigator**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│  DIRECTOR MODE - Location Navigator                                     │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────┐  ┌──────────────────────────────────────────┐│
│  │  LOCATIONS           │  │  PREVIEW: The Dusty Library              ││
│  │  ─────────────────   │  │  ─────────────────────────────────────── ││
│  │                      │  │                                          ││
│  │  🔍 Search...        │  │  [Backdrop: Ancient library]             ││
│  │                      │  │                                          ││
│  │  📍 The Dusty Library│  │  Location: The Dusty Library             ││
│  │     (2 PCs, 1 NPC)   │  │  NPCs: Librarian, Scholar                ││
│  │                      │  │                                          ││
│  │  📍 The Market Square│  │  [Exit Preview] [View as PC]             ││
│  │     (0 PCs, 3 NPCs)  │  │                                          ││
│  │                      │  │  This is how players see this location   ││
│  │  📍 The Tavern       │  │                                          ││
│  │     (1 PC, 2 NPCs)   │  │                                          ││
│  │                      │  │                                          ││
│  │  📍 The Forest       │  │                                          ││
│  │     (0 PCs, 0 NPCs)  │  │                                          ││
│  └──────────────────────┘  └──────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

**US-2.3**: As a DM, I want to see the point of view of any character (PC or NPC) so I can understand what they're experiencing.

**Acceptance Criteria**:
- In Director Mode, I can select any character from a list
- I can click "View as [Character Name]" to see the scene from their perspective
- The view shows exactly what that character sees:
  - Their current location's scene
  - NPCs visible to them
  - Available interactions from their perspective
- The view is clearly labeled ("Viewing as: Aragorn")
- I can switch between different characters' perspectives
- I can exit character view to return to normal DM view
- Character view doesn't affect the actual game state

**UI Mockup - DM Character View**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│  DIRECTOR MODE - Viewing as: Aragorn                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────┐  ┌──────────────────────────────────────────┐│
│  │  CHARACTERS          │  │  PC VIEW (Aragorn's Perspective)         ││
│  │  ─────────────────   │  │  ─────────────────────────────────────── ││
│  │                      │  │                                          ││
│  │  👤 Aragorn (PC)     │  │  [Backdrop: The Dusty Library]          ││
│  │     Location: Library│  │                                          ││
│  │     [View as this] ✓ │  │  Location: The Dusty Library             ││
│  │                      │  │  Your Character: Aragorn                 ││
│  │  👤 Legolas (PC)     │  │                                          ││
│  │     Location: Market │  │  [NPC Sprites visible to Aragorn]       ││
│  │     [View as this]   │  │                                          ││
│  │                      │  │  [Dialogue box as Aragorn sees it]       ││
│  │  👤 Librarian (NPC)  │  │                                          ││
│  │     Location: Library│  │  [Actions available to Aragorn]         ││
│  │     [View as this]   │  │                                          ││
│  │                      │  │  [Exit Character View]                   ││
│  │  👤 Scholar (NPC)    │  │                                          ││
│  │     Location: Library│  │                                          ││
│  │     [View as this]   │  │                                          ││
│  └──────────────────────┘  └──────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

**US-2.4**: As a DM, I want to see where all PCs are located so I can manage the session effectively.

**Acceptance Criteria**:
- In Director Mode, I see a "PC Locations" panel
- Each PC shows: name, current location, last active time
- I can see which PCs are at the same location vs. split across locations
- Split party is highlighted (e.g., "⚠️ Party split across 2 locations")
- I can manually move PCs to different locations if needed
- I can see a count of PCs per location

**US-2.5**: As the system, I want to resolve scenes based on PC locations so gameplay flows naturally.

**Acceptance Criteria**:
- When a PC joins, the scene manager finds the scene at their location
- If multiple PCs are at the same location, they see the same scene
- If PCs are at different locations, each sees their own location's scene
- Scene transitions happen automatically when PCs travel between locations
- Scene resolution is transparent to players - they always see their character's scene

### Epic 3: PC Management

**US-3.1**: As a Player, I want to view my character's information so I can see my stats and location.

**Acceptance Criteria**:
- I can open a "Character" panel in PC View
- I see: name, description, current location, character sheet
- I can see my inventory (future feature)
- I can see my relationships with NPCs (future feature)

**US-3.2**: As a Player, I want to edit my character's basic information so I can update my character sheet.

**Acceptance Criteria**:
- I can edit: name, description, character sheet fields
- Changes are saved to the server
- Other players see my updated character name
- DM sees my updated character information

**US-3.3**: As a DM, I want to see all PCs in the session so I can manage the party.

**Acceptance Criteria**:
- In Director Mode, I see a "Player Characters" panel
- I can see all PCs with their locations and basic info
- I can view a PC's full character sheet
- I can manually set a PC's location (for scene management)

---

## Implementation Plan

### Phase 21A: Domain Foundation (Engine)

**Goal**: Create the `PlayerCharacter` entity and repository infrastructure.

#### 21A.1: Create PlayerCharacter Entity

**Files**:
- `Engine/src/domain/value_objects/ids.rs` - Add `PlayerCharacterId`
- `Engine/src/domain/entities/player_character.rs` - New file

**Tasks**:
- [ ] Define `PlayerCharacterId` value object (UUID-based)
- [ ] Create `PlayerCharacter` struct with all fields
- [ ] Add builder methods: `new()`, `with_description()`, `with_sheet_data()`, etc.
- [ ] Add validation: name must not be empty, location_id must exist

**Acceptance Criteria**:
- `PlayerCharacter` entity compiles
- Can create a PC with name, location, sheet data
- Validation prevents invalid PCs

#### 21A.2: Create PlayerCharacter Repository

**Files**:
- `Engine/src/application/ports/outbound/repository_port.rs` - Add `PlayerCharacterRepositoryPort`
- `Engine/src/infrastructure/persistence/player_character_repository.rs` - New file

**Tasks**:
- [ ] Define `PlayerCharacterRepositoryPort` trait:
  - `create(pc: &PlayerCharacter) -> Result<()>`
  - `get(id: PlayerCharacterId) -> Result<Option<PlayerCharacter>>`
  - `get_by_session(session_id: SessionId) -> Result<Vec<PlayerCharacter>>`
  - `get_by_user_and_session(user_id: &str, session_id: SessionId) -> Result<Option<PlayerCharacter>>`
  - `update(pc: &PlayerCharacter) -> Result<()>`
  - `update_location(id: PlayerCharacterId, location_id: LocationId) -> Result<()>`
  - `delete(id: PlayerCharacterId) -> Result<()>`
- [ ] Implement `Neo4jPlayerCharacterRepository`:
  - Store PCs as `(User)-[:HAS_PC]->(PlayerCharacter)-[:IN_SESSION]->(Session)`
  - Store location relationship: `(PlayerCharacter)-[:AT_LOCATION]->(Location)`
  - Query by session, user, location

**Acceptance Criteria**:
- Repository trait defined
- Neo4j implementation compiles
- Can CRUD player characters
- Can query by session, user, location

#### 21A.3: Add PC to Session State

**Files**:
- `Engine/src/infrastructure/session.rs` - Extend `GameSession`

**Tasks**:
- [ ] Add `player_characters: HashMap<String, PlayerCharacter>` to `GameSession`
  - Key: `user_id`, Value: `PlayerCharacter`
- [ ] Add methods:
  - `add_player_character(pc: PlayerCharacter) -> Result<()>`
  - `get_player_character(user_id: &str) -> Option<&PlayerCharacter>`
  - `get_all_pcs() -> Vec<&PlayerCharacter>`
  - `update_pc_location(user_id: &str, location_id: LocationId) -> Result<()>`
- [ ] Update `join_session()` to check for existing PC

**Acceptance Criteria**:
- Session tracks player characters
- Can add/get/update PCs in session
- Session join checks for existing PC

---

### Phase 21B: Service Layer (Engine)

**Goal**: Create services for PC management and location-based scene resolution.

#### 21B.1: Create PlayerCharacterService

**Files**:
- `Engine/src/application/services/player_character_service.rs` - New file

**Tasks**:
- [ ] Create `PlayerCharacterService` struct:
  - Dependencies: `PlayerCharacterRepositoryPort`, `LocationRepositoryPort`, `WorldRepositoryPort`
- [ ] Implement methods:
  - `create_pc(request: CreatePlayerCharacterRequest) -> Result<PlayerCharacter>`
    - Validate: name not empty, location exists, session exists
    - Create PC entity
    - Save to repository
    - Add to session
  - `get_pc(id: PlayerCharacterId) -> Result<Option<PlayerCharacter>>`
  - `get_pc_by_user_and_session(user_id: &str, session_id: SessionId) -> Result<Option<PlayerCharacter>>`
  - `update_pc(id: PlayerCharacterId, request: UpdatePlayerCharacterRequest) -> Result<PlayerCharacter>`
  - `update_pc_location(id: PlayerCharacterId, location_id: LocationId) -> Result<()>`
    - Validate location exists
    - Update PC's current_location_id
    - Update in repository
    - Update in session
    - Trigger scene resolution (see 21B.2)
  - `delete_pc(id: PlayerCharacterId) -> Result<()>`

**Acceptance Criteria**:
- Service compiles and is testable
- All CRUD operations work
- Validation prevents invalid operations
- Location updates trigger scene resolution

#### 21B.2: Create SceneResolutionService

**Files**:
- `Engine/src/application/services/scene_resolution_service.rs` - New file

**Tasks**:
- [ ] Create `SceneResolutionService` struct:
  - Dependencies: `SceneRepositoryPort`, `LocationRepositoryPort`, `SessionManager`
- [ ] Implement `resolve_scene_for_session(session_id: SessionId) -> Result<Option<Scene>>`:
  1. Get all PCs in session
  2. Group PCs by `current_location_id`
  3. If all PCs at same location:
     - Find active scene at that location (via `SceneRepositoryPort::list_by_location()`)
     - If multiple scenes, pick first active one or scene with matching entry conditions
     - Return scene
  4. If PCs at different locations:
     - Return `None` (DM must choose which scene to show, or show first PC's location)
  5. If no PCs:
     - Return `None` (no scene to show)
- [ ] Implement `resolve_scene_for_pc(pc_id: PlayerCharacterId) -> Result<Option<Scene>>`:
  - Get PC's current location
  - Find scene at that location
  - Return scene

**Acceptance Criteria**:
- Service resolves scenes based on PC locations
- Handles single-location and split-party scenarios
- Returns appropriate scene or None

#### 21B.3: Wire Scene Resolution to Session Join

**Files**:
- `Engine/src/application/services/session_join_service.rs`
- `Engine/src/infrastructure/websocket.rs`

**Tasks**:
- [ ] After PC creation, call `SceneResolutionService::resolve_scene_for_pc()`
- [ ] If scene found, send `SceneUpdate` message to player
- [ ] Update session's `current_scene_id`
- [ ] When PC location changes, re-resolve scene and broadcast update

**Acceptance Criteria**:
- Players see scene at their starting location after PC creation
- Scene updates when PC location changes
- Multiple PCs at same location see same scene

---

### Phase 21C: HTTP API (Engine)

**Goal**: Expose PC management and location selection via REST API.

#### 21C.1: Create PlayerCharacter Routes

**Files**:
- `Engine/src/infrastructure/http/player_character_routes.rs` - New file
- `Engine/src/infrastructure/http/mod.rs` - Wire routes

**Tasks**:
- [ ] `POST /api/sessions/{session_id}/player-characters`
  - Body: `CreatePlayerCharacterRequest { name, description, starting_location_id, sheet_data }`
  - Response: `PlayerCharacterResponse`
  - Creates PC, adds to session, resolves scene
- [ ] `GET /api/sessions/{session_id}/player-characters`
  - Response: `Vec<PlayerCharacterResponse>`
  - Lists all PCs in session
- [ ] `GET /api/sessions/{session_id}/player-characters/me`
  - Response: `PlayerCharacterResponse`
  - Gets current user's PC (via `user_id` from session)
- [ ] `GET /api/player-characters/{pc_id}`
  - Response: `PlayerCharacterResponse`
  - Gets PC by ID
- [ ] `PUT /api/player-characters/{pc_id}`
  - Body: `UpdatePlayerCharacterRequest { name, description, sheet_data }`
  - Response: `PlayerCharacterResponse`
  - Updates PC (name, description, sheet data)
- [ ] `PUT /api/player-characters/{pc_id}/location`
  - Body: `{ location_id: String }`
  - Response: `{ success: bool, scene_id: Option<String> }`
  - Updates PC's location, triggers scene resolution
- [ ] `DELETE /api/player-characters/{pc_id}`
  - Response: `{ success: bool }`
  - Deletes PC from session

**Acceptance Criteria**:
- All routes compile and are wired
- Routes handle errors appropriately (404, 400, 500)
- Location update triggers scene resolution
- Routes require valid session membership

#### 21C.2: Create Location Selection Helper Endpoint

**Files**:
- `Engine/src/infrastructure/http/location_routes.rs` - Extend existing

**Tasks**:
- [ ] `GET /api/worlds/{world_id}/locations/available-for-starting`
  - Response: `Vec<LocationSummary>`
  - Lists all locations in world that can be starting locations
  - Filters out locations with entry conditions that PCs can't meet
  - Includes location type, description, whether it has scenes

**Acceptance Criteria**:
- Endpoint returns available starting locations
- Filters appropriately
- Includes helpful metadata

---

### Phase 21D: Player UI - PC Creation Flow

**Goal**: Build the PC creation UI in Player application.

#### 21D.1: Create PlayerCharacterService (Player)

**Files**:
- `Player/src/application/services/player_character_service.rs` - New file

**Tasks**:
- [ ] Create `PlayerCharacterService<A: ApiPort>`:
  - `create_pc(session_id: &str, request: CreatePlayerCharacterRequest) -> Result<PlayerCharacterData>`
  - `get_my_pc(session_id: &str) -> Result<Option<PlayerCharacterData>>`
  - `update_pc(pc_id: &str, request: UpdatePlayerCharacterRequest) -> Result<PlayerCharacterData>`
  - `update_location(pc_id: &str, location_id: &str) -> Result<()>`
- [ ] Define DTOs:
  - `PlayerCharacterData` (matches Engine response)
  - `CreatePlayerCharacterRequest`
  - `UpdatePlayerCharacterRequest`

**Acceptance Criteria**:
- Service compiles
- Methods call correct API endpoints
- Error handling works

#### 21D.2: Create PC Creation View

**Files**:
- `Player/src/presentation/views/pc_creation.rs` - New file
- `Player/src/routes.rs` - Add `PCCreationRoute`

**Tasks**:
- [ ] Create `PCCreationView` component:
  - Step 1: Character Basics
    - Name input (required)
    - Description textarea (optional)
  - Step 2: Character Sheet (if world has template)
    - Fetch sheet template from `/api/worlds/{id}/sheet-template`
    - Render sheet fields dynamically (reuse `SheetFieldInput` from Phase 14)
    - Validate required fields
  - Step 3: Starting Location
    - Fetch available locations from `/api/worlds/{id}/locations/available-for-starting`
    - Display location cards with: name, type, description, "Has scenes" indicator
    - Location selection (radio buttons or cards)
  - Step 4: Review & Create
    - Summary of character info
    - "Create Character" button
    - On success: navigate to `PCViewRoute`
- [ ] Add route guard: check if PC already exists, redirect to PC View if so
- [ ] Handle errors: show error messages, allow retry

**Acceptance Criteria**:
- Multi-step form works
- Character sheet renders correctly
- Location selection shows available locations
- PC creation succeeds and navigates to PC View
- Existing PC check works

#### 21D.3: Integrate PC Creation into Session Join Flow

**Files**:
- `Player/src/routes.rs` - Update `PCViewRoute`
- `Player/src/presentation/views/pc_view.rs`

**Tasks**:
- [ ] In `PCViewRoute`, after session join:
  - Check if user has PC for this session (`GET /api/sessions/{id}/player-characters/me`)
  - If no PC: navigate to `PCCreationRoute`
  - If PC exists: proceed to `PCView`
- [ ] In `PCView`, display PC's current location in UI
- [ ] Add "Edit Character" button (opens edit modal)

**Acceptance Criteria**:
- Players without PCs are directed to creation flow
- Players with PCs go directly to PC View
- PC View shows character location

#### 21D.4: Create PC Management Components

**Files**:
- `Player/src/presentation/components/pc/character_panel.rs` - New file
- `Player/src/presentation/components/pc/edit_character_modal.rs` - New file

**Tasks**:
- [ ] `CharacterPanel`:
  - Shows PC name, description, current location
  - Displays character sheet (read-only)
  - "Edit" button opens edit modal
- [ ] `EditCharacterModal`:
  - Edit name, description
  - Edit character sheet fields
  - Save button calls `update_pc()`
  - Success: close modal, refresh character panel

**Acceptance Criteria**:
- Character panel displays PC info
- Edit modal allows updates
- Changes persist to server

---

### Phase 21E: Scene Manager Integration

**Goal**: Wire PC locations into scene resolution and display.

#### 21E.1: Update Scene Resolution in Queue Workers

**Files**:
- `Engine/src/infrastructure/queue_workers.rs`

**Tasks**:
- [ ] When processing player actions that change location:
  - Update PC's location via `PlayerCharacterService::update_pc_location()`
  - Call `SceneResolutionService::resolve_scene_for_pc()`
  - Broadcast `SceneUpdate` to player
- [ ] When PC joins session:
  - After PC creation, resolve scene
  - Send `SceneUpdate` with scene at starting location

**Acceptance Criteria**:
- Location changes trigger scene updates
- New PCs see scene at starting location
- Scene updates broadcast correctly

#### 21E.2: Handle Split Party Scenario

**Files**:
- `Engine/src/application/services/scene_resolution_service.rs`
- `Engine/src/infrastructure/websocket.rs`

**Tasks**:
- [ ] When PCs are at different locations:
  - Option 1: Show first PC's location (default)
  - Option 2: Send special message to DM: "Party is split across locations"
  - Option 3: Allow DM to choose which location to show
- [ ] Add `SplitPartyNotification` WebSocket message type
- [ ] DM sees list of locations with PC counts

**Acceptance Criteria**:
- Split party scenario handled gracefully
- DM is notified when party splits
- System doesn't crash on split party

#### 21E.3: Update PC View to Show Location-Based Scene

**Files**:
- `Player/src/presentation/views/pc_view.rs`

**Tasks**:
- [ ] When `SceneUpdate` received:
  - Update `GameState` with scene
  - Display scene backdrop, location name, featured NPCs
  - Show interactions available at location
- [ ] Display PC's current location in UI (header or sidebar)
- [ ] Show location name in scene display

**Acceptance Criteria**:
- PC View shows scene at PC's location
- Location name displayed
- Scene updates when location changes

---

### Phase 21F: DM Tools for PC Management & Scene Navigation

**Goal**: Give DMs visibility, control over PCs, and ability to preview scenes from any perspective.

#### 21F.1: Create PC Management Panel (DM View)

**Files**:
- `Player/src/presentation/components/dm_panel/pc_management.rs` - New file

**Tasks**:
- [ ] `PCManagementPanel` component:
  - Lists all PCs in session
  - Each PC shows: name, user_id, current location, last active
  - "View Character Sheet" button (opens modal)
  - "Set Location" button (opens location picker)
  - "View as this Character" button (switches to character perspective)
  - "Remove from Session" button (with confirmation)
- [ ] Character sheet viewer modal (read-only)
- [ ] Location picker modal (lists all locations, sets PC location)

**Acceptance Criteria**:
- DM can see all PCs
- DM can view character sheets
- DM can manually set PC locations
- DM can view game from any PC's perspective
- Changes broadcast to affected players

#### 21F.2: Create Location Navigator (DM View)

**Files**:
- `Player/src/presentation/components/dm_panel/location_navigator.rs` - New file

**Tasks**:
- [ ] `LocationNavigator` component:
  - Lists all locations in the world
  - Shows location name, type, description
  - Shows PC count and NPC count per location
  - "Preview Location" button - shows scene as players would see it
  - Preview mode shows: backdrop, location name, NPCs, available interactions
  - "Exit Preview" button to return to normal DM view
- [ ] Preview mode is clearly labeled and doesn't affect game state

**Acceptance Criteria**:
- DM can see all locations
- DM can preview any location's scene
- Preview shows exactly what players see
- Preview mode is clearly indicated
- Preview doesn't affect game state

#### 21F.3: Create Character Perspective Viewer (DM View)

**Files**:
- `Player/src/presentation/components/dm_panel/character_perspective.rs` - New file

**Tasks**:
- [ ] `CharacterPerspectiveViewer` component:
  - Lists all characters (PCs and NPCs) in the session/world
  - "View as [Character Name]" button for each character
  - When viewing as a character:
    - Shows the scene from that character's location
    - Shows NPCs visible to that character
    - Shows interactions available to that character
    - Displays dialogue as that character would see it
  - "Exit Character View" button to return to normal DM view
  - Character view is clearly labeled
- [ ] Character view doesn't affect game state

**Acceptance Criteria**:
- DM can see all characters
- DM can view game from any character's perspective
- Character view shows exactly what that character sees
- Character view is clearly labeled
- Character view doesn't affect game state

#### 21F.4: Add PC Location Display to Director View

**Files**:
- `Player/src/presentation/views/dm_view.rs`
- `Player/src/presentation/components/dm_panel/quick_actions.rs`

**Tasks**:
- [ ] Add "PC Locations" widget to Director View:
  - Shows count of PCs per location
  - Highlights if party is split (⚠️ indicator)
  - Click to open PC Management Panel
- [ ] Add quick actions:
  - "Move PC to Location"
  - "Preview Location" (opens Location Navigator)
  - "View as Character" (opens Character Perspective Viewer)

**Acceptance Criteria**:
- DM sees PC location summary
- Split party is highlighted
- Quick actions work
- Navigation to preview tools is easy

---

## Technical Details

### Database Schema (Neo4j)

```cypher
// Player Character node
(:PlayerCharacter {
  id: String,
  session_id: String,
  user_id: String,
  world_id: String,
  name: String,
  description: String?,
  sheet_data: String,  // JSON
  current_location_id: String,
  starting_location_id: String,
  sprite_asset: String?,
  portrait_asset: String?,
  created_at: String,  // ISO 8601
  last_active_at: String
})

// Relationships
(:User {user_id: String})-[:HAS_PC]->(:PlayerCharacter)
(:PlayerCharacter)-[:IN_SESSION]->(:Session {id: String})
(:PlayerCharacter)-[:AT_LOCATION]->(:Location {id: String})
(:PlayerCharacter)-[:STARTED_AT]->(:Location {id: String})
```

### API Request/Response Examples

**Create PC**:
```json
POST /api/sessions/{session_id}/player-characters
{
  "name": "Aragorn",
  "description": "Ranger of the North",
  "starting_location_id": "loc-123",
  "sheet_data": {
    "values": {
      "strength": {"Number": 16},
      "dexterity": {"Number": 14},
      "constitution": {"Number": 15}
    }
  }
}

Response:
{
  "id": "pc-456",
  "session_id": "session-789",
  "user_id": "user-abc",
  "world_id": "world-xyz",
  "name": "Aragorn",
  "description": "Ranger of the North",
  "current_location_id": "loc-123",
  "starting_location_id": "loc-123",
  "sheet_data": {...},
  "created_at": "2025-12-15T10:00:00Z"
}
```

**Update Location**:
```json
PUT /api/player-characters/{pc_id}/location
{
  "location_id": "loc-456"
}

Response:
{
  "success": true,
  "scene_id": "scene-789"  // If scene found at new location
}
```

### WebSocket Message Updates

**New Message Types**:
```rust
// Engine -> Player
enum ServerMessage {
    // ... existing ...
    
    /// PC creation successful, includes resolved scene
    PCCreated {
        pc: PlayerCharacterData,
        scene: Option<SceneData>,  // Scene at starting location
    },
    
    /// PC location updated, includes new scene
    PCLocationUpdated {
        pc_id: String,
        location_id: String,
        scene: Option<SceneData>,
    },
    
    /// Party is split across locations (DM only)
    SplitPartyNotification {
        locations: Vec<LocationPCCount>,
    },
}

struct LocationPCCount {
    location_id: String,
    location_name: String,
    pc_count: usize,
    pc_names: Vec<String>,
}
```

---

## Testing Strategy

### Unit Tests

1. **PlayerCharacter Entity**:
   - Test validation (empty name, invalid location)
   - Test builder methods
   - Test location updates

2. **PlayerCharacterService**:
   - Test PC creation with valid/invalid data
   - Test location updates
   - Test scene resolution triggers

3. **SceneResolutionService**:
   - Test single PC at location → scene found
   - Test multiple PCs at same location → same scene
   - Test PCs at different locations → split party handling
   - Test no scene at location → returns None

### Integration Tests

1. **PC Creation Flow**:
   - Create PC → verify saved in Neo4j
   - Create PC → verify added to session
   - Create PC → verify scene resolved and sent to player

2. **Location Updates**:
   - Update PC location → verify scene re-resolved
   - Update PC location → verify SceneUpdate broadcast

3. **Session Join**:
   - Join session without PC → redirected to creation
   - Join session with PC → goes to PC View

### Manual Testing Scenarios

1. **Happy Path**:
   - Player joins session
   - Creates PC with name, sheet, starting location
   - Sees scene at starting location
   - Can interact with scene

2. **Existing PC**:
   - Player joins session where they already have a PC
   - Goes directly to PC View
   - Sees their existing character

3. **Split Party**:
   - Two players at different locations
   - Each sees scene at their location
   - DM sees split party notification

4. **Location Change**:
   - Player travels to new location
   - Scene updates to new location's scene
   - Backdrop and NPCs update

---

## Migration & Rollout

### Database Migration

No migration needed - new entity type, no existing data to migrate.

### Backward Compatibility

- Existing sessions without PCs will work (no scene shown until PC created)
- Existing NPCs unaffected
- Scene system continues to work for NPC-based scenes

### Rollout Plan

1. **Phase 21A-B**: Engine domain and services (no UI changes)
2. **Phase 21C**: API endpoints (testable via Postman/curl)
3. **Phase 21D**: Player UI (PC creation flow)
4. **Phase 21E**: Scene manager integration (gameplay enabled)
5. **Phase 21F**: DM tools (polish)

---

## Success Metrics

- ✅ Players can create characters when joining sessions
- ✅ Players can select starting locations
- ✅ Scene manager shows scenes based on PC locations
- ✅ PCs are distinct from NPCs in the system
- ✅ DMs can see and manage PCs
- ✅ Location changes trigger scene updates
- ✅ Split party scenarios handled gracefully

---

## Future Enhancements

1. **PC Persistence Across Sessions**: Allow PCs to persist across multiple sessions of the same world
2. **PC Inventory**: Track items owned by PCs
3. **PC Relationships**: Track PC relationships with NPCs
4. **PC Portraits/Sprites**: Generate or upload PC visual assets
5. **PC Backstory Integration**: Use PC backstory in LLM prompts
6. **Multiple PCs Per User**: Allow players to have multiple PCs in the same session (with DM approval)
7. **PC Death/Retirement**: Handle PC removal from sessions
8. **PC Leveling**: Track PC progression (XP, level ups)

---

## Dependencies & Blockers

**Blocked By**:
- None (all dependencies complete)

**Blocks**:
- Actual gameplay participation
- Location-based travel mechanics
- Scene transitions based on movement
- Party management features

---

## Appendix: File Structure

### Engine

```
Engine/src/
├── domain/
│   ├── entities/
│   │   └── player_character.rs          # NEW
│   └── value_objects/
│       └── ids.rs                        # EXTEND (add PlayerCharacterId)
├── application/
│   ├── services/
│   │   ├── player_character_service.rs   # NEW
│   │   └── scene_resolution_service.rs  # NEW
│   └── ports/
│       └── outbound/
│           └── repository_port.rs       # EXTEND (add PlayerCharacterRepositoryPort)
└── infrastructure/
    ├── persistence/
    │   └── player_character_repository.rs # NEW
    ├── http/
    │   └── player_character_routes.rs    # NEW
    └── session.rs                        # EXTEND (add player_characters map)
```

### Player

```
Player/src/
├── application/
│   ├── services/
│   │   └── player_character_service.rs  # NEW
│   └── dto/
│       └── player_character.rs           # NEW
├── presentation/
│   ├── views/
│   │   └── pc_creation.rs               # NEW
│   └── components/
│       └── pc/
│           ├── character_panel.rs       # NEW
│           └── edit_character_modal.rs # NEW
└── routes.rs                            # EXTEND (add PCCreationRoute)
```

---

**Plan Status**: Ready for implementation
**Estimated Effort**: 3-4 weeks (6 phases, ~30 tasks)
**Priority**: CRITICAL - Enables core gameplay

