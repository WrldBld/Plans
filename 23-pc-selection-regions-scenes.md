# Phase 23: PC Selection, Regions & Scene System

**Status**: IN PROGRESS (ENGINE COMPLETE - 23A, 23B, 23C, 23D, 23E, 23F; PLAYER PENDING)
**Priority**: HIGH - Core gameplay experience
**Depends On**: Phase 21 (PC Creation), Phase 13 (World Selection)
**Last Updated**: 2024-12-18

---

## Overview

This phase transforms WrldBldr into a JRPG-style exploration experience where:
- Players select/create/import characters before entering gameplay
- Locations contain **Regions** (sub-locations with their own backdrops)
- Scenes are dynamically derived from PC's current region + NPCs present
- NPC presence is determined by LLM reasoning based on relationships
- Players track "last known" NPC positions (fog of war)
- DM can trigger events that bring NPCs to players

**Inspiration**: Front Mission 3, Persona, classic visual novels with location-based exploration.

---

## Current Flow

```
Session Join → Check for PC → (No PC) → PC Creation → Game View
                            → (Has PC) → Game View (no scene displayed)
```

## Proposed Flow

```
Session Join → Character Selection Screen
                    ├── Select Existing PC → Load into last region → Scene View
                    ├── Create New PC → Spawn Point Selection → Scene View  
                    └── Import PC from Other World → Spawn Point Selection → Scene View

Scene View = Region Backdrop + NPCs in Region + Navigation Menu
```

---

## Key Concepts

> **Region**: A sub-location within a Location. Each region has its own backdrop image and represents a distinct "screen" in the JRPG sense. Examples: "Bar Entrance", "Bar Counter", "Table 1", "Bathroom".

> **Scene Derivation**: Scenes are not pre-authored. When PC enters a region, the system queries which NPCs are present (via LLM reasoning) and displays: backdrop + NPC sprites.

> **Observation System**: PCs track where they last saw NPCs. This creates a "fog of war" where player knowledge differs from reality, enabling mystery/investigation gameplay.

> **Game Time**: In-game time used for LLM cache TTL and story progression. Real-time passing doesn't invalidate cache; game-time advancing does.

> **NPC Presence Logic**: NPCs don't simulate travel. Instead, when querying "Is NPC at region?", the LLM reasons based on NPC relationships (WORKS_AT, FREQUENTS, HOME, etc.), time of day, and story context.

---

## Location & Region Hierarchy

```
City of Valdris (Location, has city_map.png)
│
├── [clickable area on city map] → Market District
│
└── Market District (Location, has district_map.png)
    │   parent_map_bounds: {x:100, y:200, w:150, h:100} on city map
    │
    ├── [clickable area on district map] → Rusty Anchor Tavern
    │
    ├── Market Square (Region, is_spawn_point: true)
    │   map_bounds: {x:50, y:50, w:200, h:200} on district map
    │   backdrop: market_square_scene.png
    │
    └── Rusty Anchor Tavern (Location, has tavern_floorplan.png)
        │   parent_map_bounds: {x:300, y:150, w:80, h:80} on district map
        │
        ├── Entrance (Region, is_spawn_point: true)
        │   map_bounds: {x:100, y:200, w:50, h:30} on tavern map
        │   backdrop: entrance_scene.png
        │   CONNECTED_TO_REGION → Bar Counter
        │   EXITS_TO_LOCATION → Market District (arrival: Market Square)
        │
        ├── Bar Counter (Region)
        │   map_bounds: {x:150, y:100, w:100, h:50} on tavern map
        │   backdrop: bar_counter_scene.png
        │   CONNECTED_TO_REGION → Entrance, Tables
        │
        └── Tables (Region)
            map_bounds: {x:50, y:50, w:150, h:100} on tavern map
            backdrop: tables_scene.png
            CONNECTED_TO_REGION → Bar Counter, Entrance
```

### Two Views of Navigation:

1. **Map View (Overview/Navigation)**
   - Shows location's `map_asset` (top-down floorplan)
   - Regions and child locations are **clickable areas** using their bounds
   - Click region/location → PC moves there

2. **Scene View (Gameplay/Interaction)**
   - Shows region's `backdrop_asset` (visual novel style)
   - NPCs present displayed as sprites
   - Player interacts here

---

## Data Model

### Location Entity (Updated)

```rust
pub struct Location {
    pub id: LocationId,
    pub world_id: WorldId,
    pub name: String,
    pub description: String,
    pub location_type: LocationType,
    
    // Visual assets
    pub backdrop_asset: Option<String>,   // Default scene backdrop
    pub map_asset: Option<String>,        // Top-down map image for navigation
    
    // Position on PARENT location's map (if this location is nested)
    pub parent_map_bounds: Option<MapBounds>,
    
    // Default entry point when arriving without specific region
    pub default_region_id: Option<RegionId>,
    
    pub atmosphere: Option<String>,
}
```

### Region Entity (Replaces BackdropRegion)

```rust
/// A region within a location - represents a distinct "screen" or area
/// 
/// Stored as Neo4j node: `(Location)-[:HAS_REGION]->(Region)`
#[derive(Debug, Clone)]
pub struct Region {
    pub id: RegionId,
    pub location_id: LocationId,
    pub name: String,
    pub description: String,
    
    // Scene display (visual novel view)
    pub backdrop_asset: Option<String>,
    pub atmosphere: Option<String>,
    
    // Position on parent location's map (clickable area)
    pub map_bounds: Option<MapBounds>,
    
    pub is_spawn_point: bool,
    pub order: u32,
}

#[derive(Debug, Clone)]
pub struct MapBounds {
    pub x: u32,
    pub y: u32,
    pub width: u32,
    pub height: u32,
}
```

### RegionId (Renamed from BackdropRegionId)

```rust
// In Engine/src/domain/value_objects/ids.rs
define_id!(RegionId);
// Remove: define_id!(BackdropRegionId);
```

### PlayerCharacter Entity (Updated)

```rust
pub struct PlayerCharacter {
    pub id: PlayerCharacterId,
    pub session_id: Option<SessionId>,  // CHANGED: Now optional (PC persists per-world)
    pub user_id: String,
    pub world_id: WorldId,
    
    // Identity
    pub name: String,
    pub description: Option<String>,
    pub sheet_data: Option<CharacterSheetData>,
    
    // Position
    pub current_location_id: LocationId,
    pub current_region_id: RegionId,
    // REMOVED: starting_location_id (redundant)
    
    // Assets
    pub sprite_asset: Option<String>,
    pub portrait_asset: Option<String>,
    
    // Metadata
    pub created_at: DateTime<Utc>,
    pub last_active_at: DateTime<Utc>,
}
```

### Observation System

```rust
/// Observation types for NPC tracking
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ObservationType {
    /// PC directly saw the NPC
    Direct,
    /// PC heard about NPC location from someone
    HeardAbout,
    /// PC deduced NPC location (investigation, logic)
    Deduced,
}

/// A PC's observation of an NPC's location
#[derive(Debug, Clone)]
pub struct NpcObservation {
    pub pc_id: PlayerCharacterId,
    pub npc_id: CharacterId,
    pub location_id: LocationId,
    pub region_id: RegionId,
    pub game_time: GameTime,
    pub observation_type: ObservationType,
    pub notes: Option<String>,
}
```

### Game Time System

```rust
/// In-game time tracking for a session
/// 
/// Used for:
/// - LLM cache TTL (cache invalidates when game time advances)
/// - Story progression
/// - NPC scheduling context (day/night)
#[derive(Debug, Clone)]
pub struct GameTime {
    /// Current in-game date and time
    pub current: DateTime<Utc>,
    /// How fast game time passes relative to real time (0 = paused, 1.0 = real-time)
    pub time_scale: f32,
    /// Last real-world time we updated game time
    pub last_updated: DateTime<Utc>,
}

impl GameTime {
    pub fn new() -> Self {
        Self {
            current: Utc::now(),
            time_scale: 0.0,      // Paused by default - DM controls time
            last_updated: Utc::now(),
        }
    }
    
    /// Advance game time by a fixed amount (DM action)
    pub fn advance(&mut self, duration: chrono::Duration) {
        self.current = self.current + duration;
        self.last_updated = Utc::now();
    }
    
    /// Get time of day for NPC scheduling
    pub fn time_of_day(&self) -> TimeOfDay {
        let hour = self.current.hour();
        match hour {
            6..=11 => TimeOfDay::Morning,
            12..=17 => TimeOfDay::Afternoon,
            18..=21 => TimeOfDay::Evening,
            _ => TimeOfDay::Night,
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TimeOfDay {
    Morning,
    Afternoon,
    Evening,
    Night,
}
```

### LLM Presence Cache

```rust
/// Cache for LLM "is NPC present?" queries
/// 
/// Key: (region_id, npc_id)
/// Invalidation: When game_time advances past cache entry's timestamp + TTL
pub struct PresenceCache {
    entries: HashMap<(RegionId, CharacterId), PresenceCacheEntry>,
}

pub struct PresenceCacheEntry {
    pub is_present: bool,
    pub reasoning: String,       // LLM's explanation (for DM review)
    pub cached_at_game_time: DateTime<Utc>,
    pub ttl_game_hours: u32,     // e.g., 1 hour in-game
}

impl PresenceCache {
    /// Check if cache entry is still valid
    pub fn is_valid(&self, key: &(RegionId, CharacterId), current_game_time: &GameTime) -> bool {
        if let Some(entry) = self.entries.get(key) {
            let elapsed = current_game_time.current - entry.cached_at_game_time;
            elapsed.num_hours() < entry.ttl_game_hours as i64
        } else {
            false
        }
    }
}
```

---

## Neo4j Schema

### Node Labels

```cypher
// Existing
(:World), (:Location), (:Character), (:PlayerCharacter), (:Session)

// New/Updated
(:Region)  // Replaces BackdropRegion
```

### Edge Types

```cypher
// Location hierarchy (existing)
(Location)-[:CONTAINS_LOCATION]->(Location)

// Location contains regions
(Location)-[:HAS_REGION]->(Region)

// Region to region navigation (within or across locations)
(Region)-[:CONNECTED_TO_REGION {
    description: String,
    bidirectional: bool,
    is_locked: bool,
    lock_description: String?
}]->(Region)

// Region exit to location (going "up" or "out")
(Region)-[:EXITS_TO_LOCATION {
    description: String,
    arrival_region_id: String,  // Which region in target location
    bidirectional: bool         // Can you enter from that location too?
}]->(Location)

// NPC relationships to locations (existing)
(Character)-[:WORKS_AT]->(Location)
(Character)-[:FREQUENTS]->(Location)
(Character)-[:HOME_LOCATION]->(Location)
(Character)-[:AVOIDS]->(Location)

// NPC relationships to regions (NEW)
(Character)-[:WORKS_AT_REGION {shift: "day"|"night"|"always"}]->(Region)
(Character)-[:FREQUENTS_REGION {frequency: "often"|"sometimes"|"rarely"}]->(Region)
(Character)-[:HOME_REGION]->(Region)
(Character)-[:AVOIDS_REGION {reason: String}]->(Region)

// PC observation of NPC (NEW)
(PlayerCharacter)-[:OBSERVED_NPC {
    location_id: String,
    region_id: String,
    game_time: DateTime,
    observation_type: "direct"|"heard_about"|"deduced",
    notes: String?
}]->(Character)
```

### Indexes

```cypher
CREATE INDEX region_id_idx IF NOT EXISTS FOR (r:Region) ON (r.id);
CREATE INDEX region_location_idx IF NOT EXISTS FOR (r:Region) ON (r.location_id);
CREATE INDEX region_spawn_idx IF NOT EXISTS FOR (r:Region) ON (r.is_spawn_point);
CREATE INDEX pc_region_idx IF NOT EXISTS FOR (pc:PlayerCharacter) ON (pc.current_region_id);
```

---

## User Stories

### Epic 1: PC Selection & Management

#### US-23.1: View Character Selection Screen
**As a** Player
**I want to** see a character selection screen after joining a session
**So that** I can choose which character to play or create a new one

**Acceptance Criteria:**
- Screen displays after session join (before game view)
- Shows list of existing PCs for this user in this world
- Shows "Create New Character" button
- Shows "Import Character" button
- Empty state: "No characters yet. Create your first adventurer!"
- Each PC card shows: name, description preview, last location/region, last played date

---

#### US-23.2: Select Existing Character
**As a** Player
**I want to** select one of my existing characters
**So that** I can continue playing where I left off

**Acceptance Criteria:**
- Click PC card to select
- "Play" button loads character into game view
- PC appears at their `current_region_id` (last known position)
- Scene displays with region backdrop + NPCs present

**API:** `POST /api/sessions/{id}/select-pc` with `{pc_id}`

---

#### US-23.3: Create New Character with Spawn Point
**As a** Player
**I want to** create a new character and choose where they start
**So that** I can begin my adventure at an appropriate location

**Acceptance Criteria:**
- "Create New Character" opens creation wizard (existing flow)
- Step 3 (Starting Location) shows ONLY regions marked as spawn points
- Regions grouped by parent location
- After creation, PC spawns at selected region
- Scene displays immediately

**API:** `GET /api/worlds/{id}/spawn-points` returns regions with `is_spawn_point=true`

---

#### US-23.4: Import Character from Another World
**As a** Player
**I want to** import a character from a different world with the same rule system
**So that** I can reuse a character I've already built

**Acceptance Criteria:**
- "Import Character" button shows list of worlds with matching rule system
- Select world → shows PCs from that world
- Select PC → creates a COPY in current world
- Copy has new ID, same stats/sheet data
- Must select spawn point for imported PC
- Original PC in source world unchanged

**API:** 
- `GET /api/users/{id}/pcs?rule_system=D20` - List PCs across worlds by rule system
- `POST /api/worlds/{id}/import-pc` with `{source_pc_id, spawn_region_id}`

---

#### US-23.5: Delete Character
**As a** Player
**I want to** delete a character I no longer want
**So that** I can clean up old characters

**Acceptance Criteria:**
- Delete button on PC card (trash icon)
- Confirmation: "Delete [Name]? This cannot be undone."
- PC removed from list
- Cannot delete while PC is active in a session

---

### Epic 2: Region Management (DM)

#### US-23.6: Create Region within Location
**As a** DM
**I want to** create regions within a location
**So that** I can define distinct areas for exploration

**Acceptance Criteria:**
- "Add Region" button in location details
- Form: name, description, backdrop asset, atmosphere, map_bounds, is_spawn_point
- Region appears in location's region list
- Can reorder regions (drag & drop or order field)

**API:** `POST /api/locations/{id}/regions`

---

#### US-23.7: Mark Region as Spawn Point
**As a** DM
**I want to** mark certain regions as spawn points
**So that** players can start their characters there

**Acceptance Criteria:**
- Toggle "Spawn Point" on region
- Only spawn point regions shown during PC creation
- At least one spawn point required per world (validation on PC creation)

**API:** `PUT /api/regions/{id}` with `{is_spawn_point: true}`

---

#### US-23.8: Set Location Default Region
**As a** DM
**I want to** set a default entry region for a location
**So that** players arriving without a specific destination have somewhere to go

**Acceptance Criteria:**
- Dropdown to select default region from location's regions
- Used when: exiting to location without specific arrival region
- Optional - if not set, first spawn point region used

**API:** `PUT /api/locations/{id}` with `{default_region_id}`

---

#### US-23.9: Assign NPC to Region
**As a** DM
**I want to** assign NPCs to work at, frequent, or live in specific regions
**So that** the LLM knows where NPCs are likely to be

**Acceptance Criteria:**
- In NPC edit screen, section for "Location Relationships"
- Can add: WORKS_AT_REGION (with shift), FREQUENTS_REGION (with frequency), HOME_REGION
- Shows both location-level and region-level relationships
- Visual indicator of assigned regions on location map

**API:** 
- `POST /api/characters/{id}/region-relationships` with `{region_id, relationship_type, metadata}`
- `GET /api/characters/{id}/region-relationships`

---

#### US-23.10: Connect Regions for Navigation
**As a** DM
**I want to** define how regions connect to each other
**So that** players can navigate between them

**Acceptance Criteria:**
- Region edit screen shows "Connections" section
- Can connect to: other regions in same location, regions in connected locations
- Connection has optional description ("A door leads to...", "Stairs going up...")
- Bidirectional by default, can be one-way
- Can mark as locked with description

**API:** `POST /api/regions/{id}/connections` with `{target_region_id, description, bidirectional}`

---

#### US-23.11: Connect Region to Parent Location Exit
**As a** DM
**I want to** connect a region to exit to another location
**So that** players can leave buildings/areas

**Acceptance Criteria:**
- "Add Exit" button in region connections
- Select target location
- Select arrival region in target location
- Specify description ("Step outside into the market")
- Bidirectional option

**API:** `POST /api/regions/{id}/exits` with `{target_location_id, arrival_region_id, description, bidirectional}`

---

### Epic 3: Scene Derivation

#### US-23.12: Display Scene from Region
**As a** Player
**I want to** see a scene based on my current region
**So that** I experience the location visually

**Acceptance Criteria:**
- Scene shows region's backdrop image
- If no backdrop, falls back to parent location's backdrop
- Atmosphere text displayed (if set)
- Scene updates when PC moves to different region

---

#### US-23.13: Show NPCs in Current Region
**As a** Player
**I want to** see NPCs who are present in my current region
**So that** I know who I can interact with

**Acceptance Criteria:**
- NPCs determined by LLM query (based on relationships + game time)
- NPC sprites displayed in scene
- Click NPC to start interaction
- If no NPCs, scene shows empty backdrop
- Loading state while querying NPC presence

---

#### US-23.14: View Navigation Options
**As a** Player
**I want to** see where I can go from my current region
**So that** I can explore the world

**Acceptance Criteria:**
- Navigation panel shows connected regions/locations
- Grouped: "In [Location Name]" vs "Leave to..."
- Shows locked connections with lock icon
- Click destination to move PC there
- Moving updates scene to new region

---

### Epic 4: Observation System

#### US-23.15: Track NPC Observations (Automatic - Direct)
**As a** Player
**I want** the system to remember where I saw NPCs
**So that** I can find them again later

**Acceptance Criteria:**
- When scene displays with NPCs, observation records created (type: direct)
- Records include: region, game time, observation type
- Overwrites previous observation for same NPC (latest wins)

---

#### US-23.16: Record Heard Information (DM - HeardAbout)
**As a** DM
**I want to** tell a player where an NPC was seen/heard about
**So that** players gain information through roleplay

**Acceptance Criteria:**
- DM action: "Share NPC Location" with PC
- Creates observation record (type: heard_about)
- Player sees in their "Known NPC Locations" list
- Can include notes: "The bartender mentioned seeing them at the docks"

**WebSocket:** `ShareNpcLocation { pc_id, npc_id, region_id, notes }`

---

#### US-23.17: Deduce NPC Location (Challenge Result - Deduced)
**As a** Player
**I want to** learn NPC locations through investigation
**So that** my character's skills matter

**Acceptance Criteria:**
- Successful investigation/tracking challenge can reveal NPC location
- Creates observation record (type: deduced)
- Part of challenge outcome triggers

---

#### US-23.18: View Known NPC Locations
**As a** Player
**I want to** see where I last saw/heard about NPCs
**So that** I can plan where to find them

**Acceptance Criteria:**
- "Contacts" or "Known NPCs" panel in player UI
- Shows NPC name, last known location/region, how long ago (game time)
- Observation type icon (eye=direct, ear=heard, brain=deduced)
- Click to see notes/details

---

### Epic 5: Game Time System

#### US-23.19: View Current Game Time
**As a** Player or DM
**I want to** see the current in-game time
**So that** I understand the story's temporal context

**Acceptance Criteria:**
- Time display in session UI (e.g., "Day 3, 2:30 PM")
- Shows time of day indicator (sun/moon icon)
- DM sees additional controls

---

#### US-23.20: Advance Game Time (DM)
**As a** DM
**I want to** advance the in-game time
**So that** I can progress the story and invalidate stale information

**Acceptance Criteria:**
- Quick buttons: "+1 hour", "+6 hours", "+1 day"
- Custom advance: specify hours/days
- Advancing time broadcasts to all players
- Invalidates LLM presence cache entries past TTL
- Scene may update (NPCs leave/arrive based on time of day)

**WebSocket:** `AdvanceGameTime { duration_hours }`

---

### Epic 6: DM Event System

#### US-23.21: Trigger NPC Approach Event
**As a** DM
**I want to** have an NPC approach a player
**So that** I can initiate interactions and drive the story

**Acceptance Criteria:**
- DM selects NPC and target PC
- Enters event description ("A hooded figure slides into the seat across from you...")
- Event sent to PC as dialogue box
- NPC "arrives" in PC's current region
- NPC sprite appears in scene
- Interaction can begin

**WebSocket:** `TriggerApproachEvent { npc_id, target_pc_id, description }`
**Client receives:** `ApproachEvent { npc_name, npc_id, description, npc_sprite }`

---

#### US-23.22: View Approach Event as Player
**As a** Player
**I want to** see when an NPC approaches me
**So that** I can react to the situation

**Acceptance Criteria:**
- Dialogue box appears with event description
- NPC sprite fades into scene
- After dismissing dialogue, can interact with NPC normally
- Creates direct observation record

---

#### US-23.23: Trigger Location Event (DM)
**As a** DM
**I want to** describe something happening in a region
**So that** I can set the scene without NPC involvement

**Acceptance Criteria:**
- DM enters description for a region
- All PCs in that region see the description
- No NPC involved (just narration)
- Examples: "The lights flicker and go out", "You hear a scream from outside"

**WebSocket:** `TriggerLocationEvent { region_id, description }`

---

## UI Mockups

### Character Selection Screen

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  [← Leave Session]              Choose Your Character                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  YOUR CHARACTERS IN "THE DRAGON'S BANE"                                     │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  [Portrait]  Kira Shadowmend                               [🗑️]    │   │
│  │              Human Rogue • Level 5                                   │   │
│  │              Last seen: Rusty Anchor Bar - Bar Counter              │   │
│  │              Last played: 2 hours ago                               │   │
│  │                                                      [Play →]       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  [Portrait]  Brother Marcus                                [🗑️]    │   │
│  │              Dwarf Cleric • Level 3                                  │   │
│  │              Last seen: Temple District - Main Hall                 │   │
│  │              Last played: 5 days ago                                │   │
│  │                                                      [Play →]       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐        │
│  │  [+] Create New Character    │  │  [↓] Import from Other World │        │
│  └──────────────────────────────┘  └──────────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Import Character Modal

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Import Character                              [×]   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Select a world with the same rule system (D20):                           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ○ Curse of Strahd                                                   │   │
│  │    2 characters available                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ● Lost Mines of Phandelver                                          │   │
│  │    1 character available                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  CHARACTERS IN "LOST MINES OF PHANDELVER":                                 │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ● [Portrait]  Theron Brightblade                                    │   │
│  │                Half-Elf Paladin • Level 4                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ⚠️ This will create a COPY. The original character stays in its world.   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  [Cancel]                                    [Import & Select Spawn] │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Spawn Point Selection (PC Creation Step 3 - Updated)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  [← Back]                  Create Your Character                     3/4   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CHOOSE YOUR STARTING LOCATION                                              │
│  Where will your adventure begin?                                          │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  📍 CITY OF VALDRIS                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ○ Market Square                                                     │   │
│  │    A bustling plaza where merchants hawk their wares                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ● Rusty Anchor Bar - Entrance                                       │   │
│  │    The worn steps lead to a popular tavern among adventurers        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  📍 OUTSKIRTS                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ○ Forest Road - Crossroads                                          │   │
│  │    Where the road from the capital meets the wilderness path        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  [Back]                                              [Next →]        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Scene View with Navigation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Rusty Anchor Bar - Bar Counter                    🕐 Day 3, 7:30 PM 🌙    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │                      [BAR COUNTER BACKDROP]                          │   │
│  │                                                                       │   │
│  │                    ┌─────────┐                                       │   │
│  │                    │ GRUFF   │                                       │   │
│  │                    │(Bartender)                                      │   │
│  │                    └─────────┘                                       │   │
│  │                                                                       │   │
│  │   The smell of ale and woodsmoke fills the air. Gruff polishes      │   │
│  │   a mug behind the counter, eyeing you with mild interest.          │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────┐  ┌─────────────────────────────────────────┐  │
│  │  CHARACTERS HERE        │  │  GO TO...                               │  │
│  │  ─────────────────────  │  │  ─────────────────────────────────────  │  │
│  │  👤 Gruff the Bartender │  │  📍 In Rusty Anchor Bar:                │  │
│  │     [Talk]              │  │     → Entrance                          │  │
│  │                         │  │     → Tables                            │  │
│  │                         │  │     → Back Room (🔒 Locked)             │  │
│  │                         │  │  🚪 Leave to:                           │  │
│  │                         │  │     → Market District                   │  │
│  └─────────────────────────┘  └─────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### DM: Region Management

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  [← Back]            Rusty Anchor Bar - Regions                    [Edit]  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LOCATION MAP                                         [Upload Map Image]   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  ┌──────────┐    ┌────────────────────────┐                         │   │
│  │  │ Entrance │    │      Tables            │                         │   │
│  │  │  (spawn) │    │                        │                         │   │
│  │  └────┬─────┘    └───────────┬────────────┘                         │   │
│  │       │                      │                                       │   │
│  │       └──────┬───────────────┘                                       │   │
│  │              │                                                        │   │
│  │       ┌──────┴──────┐    ┌────────────┐                              │   │
│  │       │ Bar Counter │────│ Back Room  │                              │   │
│  │       └─────────────┘    │  (locked)  │                              │   │
│  │                          └────────────┘                              │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  REGIONS                                                   [+ Add Region]  │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  📍 Entrance                                           [✏️] [🗑️]   │   │
│  │  ┌────────────┐  The worn wooden steps lead into the tavern         │   │
│  │  │ [backdrop] │  ☑️ Spawn Point • 🚪 Exits to: Market District      │   │
│  │  │  preview   │  NPCs: Hostess Mira (works here, day)               │   │
│  │  └────────────┘  Connects to: Bar Counter, Tables                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  📍 Bar Counter                                        [✏️] [🗑️]   │   │
│  │  ┌────────────┐  A long oak counter stained with years of use       │   │
│  │  │ [backdrop] │  ☐ Spawn Point                                      │   │
│  │  │  preview   │  NPCs: Gruff (works here, always)                   │   │
│  │  └────────────┘  Connects to: Entrance, Tables, Back Room           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### DM: Trigger Approach Event

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Trigger NPC Approach                           [×]   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Select NPC:                                                                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ Mysterious Stranger                                                ▼  │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  Target Player:                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ Kira Shadowmend (at Bar Counter)                                   ▼  │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  Event Description:                                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ A hooded figure slides onto the stool next to you. Without looking   │ │
│  │ up, they mutter: "You're the one asking about the missing merchant,  │ │
│  │ aren't you?" The figure's hand rests near a concealed blade.         │ │
│  │                                                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ℹ️ The NPC will appear in the player's current region after this event.  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  [Cancel]                                    [Trigger Event →]       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Player: Approach Event Dialogue

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Rusty Anchor Bar - Bar Counter                    🕐 Day 3, 7:30 PM 🌙    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │                      [BAR COUNTER BACKDROP]                          │   │
│  │                         (slightly dimmed)                            │   │
│  │                                                                       │   │
│  │        ┌───────────────────────────────────────────────────┐        │   │
│  │        │                                                    │        │   │
│  │        │   A hooded figure slides onto the stool next to   │        │   │
│  │        │   you. Without looking up, they mutter: "You're   │        │   │
│  │        │   the one asking about the missing merchant,      │        │   │
│  │        │   aren't you?" The figure's hand rests near a     │        │   │
│  │        │   concealed blade.                                │        │   │
│  │        │                                                    │        │   │
│  │        │                              [Continue →]         │        │   │
│  │        └───────────────────────────────────────────────────┘        │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Player: Known NPCs Panel

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  👥 Known NPCs                                                        [×]   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  👁️ Gruff the Bartender                                              │   │
│  │  Last seen: Bar Counter • Just now                                   │   │
│  │  "Serves drinks at the Rusty Anchor. Knows everyone."               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  👂 Marcus the Merchant                                              │   │
│  │  Last seen: Market Square • 2 days ago (heard about)                │   │
│  │  "The bartender mentioned he hasn't been at his stall."             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  🧠 Mysterious Stranger                                              │   │
│  │  Last seen: Bar Counter • 10 minutes ago (deduced)                  │   │
│  │  "Was asking about the missing merchant. Might know something."     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Legend: 👁️ Saw directly  👂 Heard about  🧠 Deduced                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### DM: Game Time Controls

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🕐 GAME TIME                                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Current: Day 3, 7:30 PM                                    🌙 Evening     │
│                                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                       │
│  │ +1 Hour  │ │ +6 Hours │ │ +1 Day   │ │ Custom   │                       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘                       │
│                                                                             │
│  ⚠️ Advancing time may change which NPCs are present in scenes.            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## API Endpoints

### New REST Endpoints

```
# Region Management
GET    /api/locations/{id}/regions           # List regions in location
POST   /api/locations/{id}/regions           # Create region
GET    /api/regions/{id}                     # Get region details
PUT    /api/regions/{id}                     # Update region
DELETE /api/regions/{id}                     # Delete region
POST   /api/regions/{id}/connections         # Connect to another region
POST   /api/regions/{id}/exits               # Add exit to location

# Spawn Points
GET    /api/worlds/{id}/spawn-points         # List spawn point regions (grouped by location)

# PC Selection & Import
GET    /api/sessions/{id}/available-pcs      # List user's PCs for this world
POST   /api/sessions/{id}/select-pc          # Select PC to play
GET    /api/users/{id}/pcs                   # List all user's PCs (with ?rule_system filter)
POST   /api/worlds/{id}/import-pc            # Import PC from another world

# NPC Region Relationships
GET    /api/characters/{id}/region-relationships
POST   /api/characters/{id}/region-relationships
DELETE /api/characters/{id}/region-relationships/{rel_id}

# Observations
GET    /api/pcs/{id}/observations            # Get PC's NPC observations
POST   /api/pcs/{id}/observations            # Create observation (internal/DM)

# Game Time
GET    /api/sessions/{id}/game-time          # Get current game time
POST   /api/sessions/{id}/game-time/advance  # Advance game time

# NPC Presence (Internal)
POST   /api/internal/query-presence          # Query if NPCs are in region
```

### WebSocket Messages

```rust
// Client → Server
enum ClientMessage {
    // ... existing ...
    
    // PC Selection
    SelectPlayerCharacter { pc_id: String },
    
    // Navigation
    MoveToRegion { region_id: String },
    ExitToLocation { location_id: String, arrival_region_id: Option<String> },
    
    // DM Events
    TriggerApproachEvent { npc_id: String, target_pc_id: String, description: String },
    TriggerLocationEvent { region_id: String, description: String },
    ShareNpcLocation { pc_id: String, npc_id: String, region_id: String, notes: Option<String> },
    
    // Game Time
    AdvanceGameTime { hours: u32 },
}

// Server → Client
enum ServerMessage {
    // ... existing ...
    
    // PC Selection
    PcSelected { pc: PlayerCharacterData },
    
    // Scene Updates
    SceneChanged { 
        region: RegionData, 
        npcs_present: Vec<NpcPresenceData>,
        navigation_options: Vec<NavigationOption>,
    },
    NpcPresenceUpdated { region_id: String, npcs: Vec<NpcPresenceData> },
    
    // Events
    ApproachEvent { npc_id: String, npc_name: String, npc_sprite: Option<String>, description: String },
    LocationEvent { description: String },
    NpcLocationShared { npc_id: String, npc_name: String, region_name: String, notes: Option<String> },
    
    // Game Time
    GameTimeUpdated { game_time: GameTimeData },
}
```

---

## Implementation Phases

### Phase 23A: Region Entity & Basic CRUD (Engine) - Priority 1 ✅ COMPLETE

**Goal**: Establish Region as a first-class entity with spawn point support.

**Duration**: 2-3 days
**Completed**: 2024-12-18

#### 23A.1: Rename BackdropRegionId → RegionId ✅

**Files**:
- `Engine/src/domain/value_objects/ids.rs`

**Tasks**:
- [x] Rename `BackdropRegionId` to `RegionId`
- [x] Update all imports/usages

#### 23A.2: Create Region Entity ✅

**Files**:
- `Engine/src/domain/entities/region.rs` (NEW - replaces BackdropRegion in location.rs)
- `Engine/src/domain/entities/mod.rs`
- `Engine/src/domain/entities/location.rs` (remove BackdropRegion)

**Tasks**:
- [x] Create `Region` entity with all fields (backdrop, map_bounds, is_spawn_point)
- [x] Create `MapBounds` struct
- [x] Create `RegionConnection` and `RegionExit` structs
- [x] Remove `BackdropRegion` struct from location.rs
- [x] Export from entities mod

#### 23A.3: Update Location Entity ✅

**Files**:
- `Engine/src/domain/entities/location.rs`

**Tasks**:
- [x] Add `map_asset: Option<String>` field
- [x] Add `parent_map_bounds: Option<MapBounds>` field
- [x] Add `default_region_id: Option<RegionId>` field
- [x] Update constructor and builder methods

#### 23A.4: Region Repository ✅

**Files**:
- `Engine/src/infrastructure/persistence/region_repository.rs` (NEW)
- `Engine/src/infrastructure/persistence/mod.rs`

**Tasks**:
- [x] Implement `Neo4jRegionRepository`
- [x] CRUD operations: create, get, list_by_location, update, delete
- [x] Region connections: create_connection, get_connections, delete_connection, unlock_connection
- [x] Region exits: create_exit, get_exits, delete_exit
- [x] Query spawn points: list_spawn_points

#### 23A.5: Update Location Repository ✅

**Files**:
- `Engine/src/infrastructure/persistence/location_repository.rs`

**Tasks**:
- [x] Remove old BackdropRegion methods (moved to region_repository)
- [x] Add `create_region` and `get_regions` methods (basic, via LocationRepositoryPort)
- [x] Add support for new Location fields (map_asset, parent_map_bounds, default_region_id)

#### 23A.6: Region HTTP Routes ✅

**Files**:
- `Engine/src/infrastructure/http/region_routes.rs` (NEW)
- `Engine/src/infrastructure/http/mod.rs`

**Tasks**:
- [x] `GET /api/locations/{location_id}/regions`
- [x] `POST /api/locations/{location_id}/regions`
- [x] `GET /api/regions/{region_id}`
- [x] `PATCH /api/regions/{region_id}`
- [x] `DELETE /api/regions/{region_id}`
- [x] `GET /api/worlds/{world_id}/spawn-points`
- [x] `GET /api/regions/{region_id}/connections`
- [x] `POST /api/regions/{region_id}/connections`
- [x] `DELETE /api/regions/{from_region_id}/connections/{to_region_id}`
- [x] `POST /api/regions/{from_region_id}/connections/{to_region_id}/unlock`
- [x] `GET /api/regions/{region_id}/exits`
- [x] `POST /api/regions/{region_id}/exits`
- [x] `DELETE /api/regions/{region_id}/exits/{location_id}`

**Acceptance Criteria**: ✅ All Complete
- [x] Regions can be created, read, updated, deleted
- [x] Regions can be marked as spawn points
- [x] Regions can connect to other regions
- [x] Regions can have exits to locations (with arrival_region_id)
- [x] Spawn points can be queried by world
- [x] Engine compiles with no errors

---

### Phase 23B: PC Selection & Import (Engine + Player) - Priority 1 ✅ ENGINE COMPLETE

**Goal**: Players can select existing PCs or import from other worlds.

**Duration**: 3-4 days
**Engine Completed**: 2024-12-18
**Player Status**: Pending (frontend implementation)

#### 23B.1: Update PlayerCharacter Entity ✅

**Files**:
- `Engine/src/domain/entities/player_character.rs`

**Tasks**:
- [x] Make `session_id` optional (`Option<SessionId>`)
- [x] Add `current_region_id: Option<RegionId>`
- [x] Keep `starting_location_id` (useful for reference/history)
- [x] Update constructors: `new()` for standalone, `new_in_session()` for session-bound
- [x] Add methods: `bind_to_session()`, `unbind_from_session()`, `is_bound_to_session()`
- [x] Add methods: `with_starting_region()`, `update_region()`, `update_position()`

#### 23B.2: Update PC Repository ✅

**Files**:
- `Engine/src/infrastructure/persistence/player_character_repository.rs`

**Tasks**:
- [x] Update Neo4j queries for new schema (optional session_id, current_region_id)
- [x] Add `get_by_user_and_world(user_id, world_id)` method
- [x] Add `get_unbound_by_user(user_id)` method
- [x] Add `update_region(id, region_id)` method
- [x] Add `update_position(id, location_id, region_id)` method
- [x] Add `bind_to_session(id, session_id)` method
- [x] Add `unbind_from_session(id)` method

#### 23B.3: Update PC Service ✅

**Files**:
- `Engine/src/application/services/player_character_service.rs`

**Tasks**:
- [x] Update `CreatePlayerCharacterRequest` to have optional `session_id`
- [x] Update `create_pc()` to handle standalone and session-bound PCs

#### 23B.4: Update PlayerCharacterRepositoryPort ✅

**Files**:
- `Engine/src/application/ports/outbound/repository_port.rs`

**Tasks**:
- [x] Add `get_by_user_and_world()` method
- [x] Add `get_unbound_by_user()` method
- [x] Add `update_region()` method
- [x] Add `update_position()` method
- [x] Add `bind_to_session()` method
- [x] Add `unbind_from_session()` method

#### 23B.5: Update PC Routes/DTOs ✅

**Files**:
- `Engine/src/infrastructure/http/player_character_routes.rs`

**Tasks**:
- [x] Update `PlayerCharacterResponseDto` with optional `session_id` and `current_region_id`
- [x] Update route handler to pass `Some(session_id)` for session-bound creation

#### 23B.6: PC Selection HTTP Routes ✅ COMPLETE

**Files**:
- `Engine/src/infrastructure/http/player_character_routes.rs`
- `Engine/src/infrastructure/http/mod.rs`

**Tasks**:
- [x] `GET /api/sessions/{id}/available-pcs` - List PCs for selection
- [x] `POST /api/sessions/{id}/select-pc` - Select PC for session
- [x] `GET /api/users/{id}/pcs?rule_system=X` - List PCs across worlds
- [x] `POST /api/worlds/{id}/import-pc` - Import/copy PC

**Implementation Notes**:
- `list_available_pcs` returns PCs for user in session's world with location/region names
- `select_pc` binds PC to session, verifies ownership and world match
- `list_user_pcs` filters by rule system type (D20, D100, Narrative, Custom)
- `import_pc` copies PC to new world, requires spawn point region

#### 23B.7: Character Selection View (Player) - Pending

**Files**:
- `Player/src/presentation/views/character_select.rs` (NEW)
- `Player/src/routes/mod.rs`
- `Player/src/routes/player_routes.rs`

**Tasks**:
- [ ] Create `CharacterSelectView` component
- [ ] List existing PCs with details
- [ ] "Create New" button → PC Creation wizard
- [ ] "Import" button → Import modal
- [ ] "Play" button → Select PC and enter game
- [ ] Update routing to show character select after session join

#### 23B.8: Import Character Modal (Player) - Pending

**Files**:
- `Player/src/presentation/components/character_select/import_modal.rs` (NEW)

**Tasks**:
- [ ] List worlds with matching rule system
- [ ] Show PCs in selected world
- [ ] Import creates copy + redirects to spawn selection
- [ ] Error handling for import failures

#### 23B.9: Update PC Creation Wizard - Pending

**Files**:
- `Player/src/presentation/views/pc_creation.rs`

**Tasks**:
- [ ] Step 3: Show only spawn point REGIONS (not all locations)
- [ ] Group regions by parent location
- [ ] Update API calls to use spawn-points endpoint
- [ ] Pass region_id (not location_id) to PC creation

**Acceptance Criteria**:
- [x] PlayerCharacter entity supports optional session binding
- [x] PlayerCharacter entity tracks current_region_id
- [x] Repository methods exist for PC selection workflows
- [x] Engine compiles with no errors
- [ ] (Player) Character selection screen after joining session
- [ ] (Player) Can select existing PC and enter game
- [ ] (Player) Can create new PC with spawn region selection
- [ ] (Player) Can import PC from another world (copy)

---

### Phase 23C: Scene Derivation & Navigation (Engine + Player) - Priority 2

**Goal**: Scenes display based on PC's region + NPCs present.

**Duration**: 4-5 days
**Status**: IN PROGRESS (ENGINE COMPLETE, PLAYER PENDING)

#### 23C.1: NPC Region Relationships ✅

**Files**:
- `Engine/src/infrastructure/persistence/character_repository.rs`

**Tasks**:
- [x] Add methods for region relationships:
  - `set_home_region(char_id, region_id)`
  - `set_work_region(char_id, region_id, shift)`
  - `add_frequented_region(char_id, region_id, frequency)`
  - `add_avoided_region(char_id, region_id, reason)`
  - `remove_*_region` methods
  - `list_region_relationships(char_id)`
  - `get_npcs_related_to_region(region_id)` - All NPCs with any relationship
- [x] Created `RegionShift` enum (day, night, always)
- [x] Created `RegionFrequency` enum (often, sometimes, rarely)
- [x] Created `RegionRelationship` and `RegionRelationshipType` structs

#### 23C.2: NPC Region Relationship Routes ✅

**Files**:
- `Engine/src/infrastructure/http/character_routes.rs`
- `Engine/src/infrastructure/http/region_routes.rs`

**Tasks**:
- [x] `GET /api/characters/{id}/region-relationships`
- [x] `POST /api/characters/{id}/region-relationships`
- [x] `DELETE /api/characters/{character_id}/region-relationships/{region_id}/{rel_type}`
- [x] `GET /api/regions/{region_id}/npcs` - NPCs related to a region

#### 23C.3: Game Time System ✅

**Files**:
- `Engine/src/domain/value_objects/game_time.rs` (NEW)

**Tasks**:
- [x] Create `GameTime` struct with current time, time_scale, last_updated
- [x] Create `TimeOfDay` enum (Morning, Afternoon, Evening, Night)
- [x] Implement methods: `advance()`, `advance_hours()`, `advance_days()`, `time_of_day()`, `display_time()`, `display_date()`
- [x] Support paused mode (time_scale = 0, default)
- [x] Unit tests for time advancement and time-of-day calculation

#### 23C.4: Add game_time to Session ✅

**Files**:
- `Engine/src/infrastructure/session/game_session.rs`

**Tasks**:
- [x] Add `game_time: GameTime` field to `GameSession`
- [x] Initialize game_time in constructors
- [x] Add accessor methods: `game_time()`, `game_time_mut()`, `advance_time_hours()`, `advance_time_days()`, `time_of_day()`, `display_game_time()`, `is_time_paused()`

#### 23C.5: NPC Presence Query Service ✅

**Files**:
- `Engine/src/application/services/presence_service.rs` (NEW)

**Tasks**:
- [x] Create `PresenceService<L: LlmPort>` for querying NPC presence
- [x] `query_presence(region_id, game_time)` method
- [x] `PresenceServiceConfig` for configuration (TTL, temperature, LLM toggle)
- [x] Simple rule-based presence determination (no LLM dependency)
- [x] LLM-based presence determination with prompt building
- [x] `PresenceCache` with game-time TTL
- [x] `invalidate_cache()` and `invalidate_all_cache()` methods
- [x] `force_npc_present()` for DM approach events
- [x] `NpcPresenceResult` struct with character_id, name, is_present, reasoning, sprite_asset

#### 23C.6: Scene Query Endpoint ✅

**Files**:
- `Engine/src/infrastructure/http/region_routes.rs`

**Tasks**:
- [x] `POST /api/regions/{region_id}/scene` - Get derived scene for a region
- [x] `DerivedSceneDto` with region, location_name, backdrop, atmosphere, npcs_present, navigation, game_time
- [x] `NpcPresenceDto` with optional DM-only reasoning
- [x] `NavigationOptionsDto` with connected_regions and exits
- [x] `GameTimeDto` with display, time_of_day, is_paused
- [x] Rule-based NPC presence determination based on relationships and time of day

#### 23C.6: Scene View Updates (Player)

**Files**:
- `Player/src/presentation/views/pc_view.rs`
- `Player/src/presentation/components/scene_display.rs` (NEW or update)

**Tasks**:
- [ ] Display region backdrop (fall back to location backdrop)
- [ ] Query NPC presence when region changes
- [ ] Show NPC sprites for present NPCs
- [ ] Show atmosphere text
- [ ] Click NPC to interact
- [ ] Loading state while querying presence

#### 23C.7: Navigation Panel (Player)

**Files**:
- `Player/src/presentation/components/navigation_panel.rs` (NEW)

**Tasks**:
- [ ] List connected regions (within same location)
- [ ] List exits to other locations
- [ ] Show locked connections with lock icon
- [ ] Click to move PC to region/location
- [ ] Send `MoveToRegion` or `ExitToLocation` WebSocket message

#### 23C.8: Handle Movement & Scene Updates ✅ ENGINE COMPLETE

**Files**:
- `Engine/src/infrastructure/websocket.rs`
- `Engine/src/infrastructure/websocket/messages.rs`
- `Player/src/presentation/handlers/session_message_handler.rs`
- `Player/src/application/dto/websocket_messages.rs`

**Tasks**:
- [x] Add `SelectPlayerCharacter`, `MoveToRegion`, `ExitToLocation` to ClientMessage
- [x] Add `PcSelected`, `SceneChanged`, `MovementBlocked` to ServerMessage
- [x] Add supporting DTOs: `RegionData`, `NpcPresenceData`, `NavigationData`, `NavigationTarget`, `NavigationExit`
- [x] Handle `SelectPlayerCharacter` message - validate PC ownership, return PcSelected
- [x] Handle `MoveToRegion` message - validate connection, check locks, update position
- [x] Handle `ExitToLocation` message - use arrival_region_id or default_region_id
- [x] Update PC's `current_location_id` and `current_region_id` via repository
- [x] Query NPC presence using rule-based `is_npc_present()` function
- [x] Send `SceneChanged` with region, npcs_present, navigation
- [x] Send `MovementBlocked` if path is locked
- [x] Sync messages to Player's websocket_messages.rs
- [x] Add handlers for new messages in Player's session_message_handler.rs
- [ ] (Player) Scene view updates on SceneChanged

**Acceptance Criteria** (Engine Complete):
- [x] NPC-Region relationships stored in Neo4j (WORKS_AT_REGION, FREQUENTS_REGION, HOME_REGION, AVOIDS_REGION)
- [x] HTTP routes for managing NPC-Region relationships
- [x] Game time system with time of day (Morning, Afternoon, Evening, Night)
- [x] Session tracks game time (paused by default, DM advances)
- [x] PresenceService determines NPC presence (rule-based + LLM-ready)
- [x] Presence cache with game-time TTL
- [x] Derived scene endpoint returns region + NPCs present + navigation
- [x] WebSocket handlers for SelectPlayerCharacter, MoveToRegion, ExitToLocation
- [x] SceneChanged, PcSelected, MovementBlocked messages implemented
- [x] Navigation uses region connections and checks locks
- [x] Exits to locations use arrival_region_id or default_region_id
- [x] Engine compiles with no errors
- [ ] (Player) Scene displays region backdrop + present NPCs
- [ ] (Player) Navigation panel shows connected regions/locations
- [ ] (Player) Scene updates when moving

---

### Phase 23D: Observation System (Engine + Player) - Priority 2 ✅ ENGINE COMPLETE

**Goal**: Track and display PC's knowledge of NPC locations.

**Duration**: 2-3 days
**Engine Completed**: 2024-12-18
**Player Status**: Pending (frontend implementation)

#### 23D.1: Observation Entity & Repository ✅

**Files**:
- `Engine/src/domain/entities/observation.rs` (NEW)
- `Engine/src/infrastructure/persistence/observation_repository.rs` (NEW)

**Tasks**:
- [x] Create `NpcObservation` struct
- [x] Create `ObservationType` enum (Direct, HeardAbout, Deduced)
- [x] Create `ObservationSummary` struct for display
- [x] Neo4j repository with:
  - `upsert(observation)` - Create or update
  - `get_for_pc(pc_id)`
  - `get_summaries_for_pc(pc_id)` - With NPC/location names
  - `get_latest(pc_id, npc_id)`
  - `delete(pc_id, npc_id)`
  - `batch_upsert(observations)` - For scene entry

#### 23D.2: Auto-Create Observations (Direct) ✅

**Files**:
- `Engine/src/infrastructure/http/region_routes.rs`

**Tasks**:
- [x] When derived scene endpoint called with `pc_id`, create Direct observations
- [x] Include region_id, location_id, game_time
- [x] Uses `batch_upsert()` for all NPCs present

#### 23D.3: DM Share Location (HeardAbout) ✅

**Files**:
- `Engine/src/infrastructure/websocket.rs`
- `Engine/src/infrastructure/websocket/messages.rs`

**Tasks**:
- [x] Add `ShareNpcLocation` message to `ClientMessage` enum
- [x] Handle `ShareNpcLocation` message in websocket handler
- [x] Validate DM permission
- [x] Create HeardAbout observation for target PC
- [x] Log success (future: broadcast to PC)

#### 23D.4: Deduced Observations (Challenge Integration)

**Files**:
- `Engine/src/application/services/challenge_resolution_service.rs`
- `Engine/src/domain/entities/challenge.rs` (if needed)

**Tasks**:
- [ ] Add `OutcomeTrigger::RevealNpcLocation { npc_id, region_id }`
- [ ] When trigger executes, create Deduced observation

#### 23D.5: Observation HTTP Routes ✅

**Files**:
- `Engine/src/infrastructure/http/observation_routes.rs` (NEW)

**Tasks**:
- [x] `GET /api/player-characters/{pc_id}/observations` - List all observations
- [x] `POST /api/player-characters/{pc_id}/observations` - Create observation
- [x] `GET /api/player-characters/{pc_id}/observations/{npc_id}` - Get specific
- [x] `DELETE /api/player-characters/{pc_id}/observations/{npc_id}` - Delete

#### 23D.6: Known NPCs Panel (Player)

**Files**:
- `Player/src/presentation/components/known_npcs_panel.rs` (NEW)

**Tasks**:
- [ ] Fetch PC's observations on load
- [ ] Display list with: NPC name, last region, time ago, observation type icon
- [ ] Show notes if present
- [ ] Legend for observation types

**Acceptance Criteria** (Engine Complete):
- [x] Direct observations created when PC sees NPC in scene (via derived scene endpoint with pc_id)
- [x] DM can share NPC locations (HeardAbout) via WebSocket
- [ ] Challenges can reveal NPC locations (Deduced) - Future enhancement
- [x] HTTP routes for observation CRUD
- [x] Engine compiles with no errors
- [ ] (Player) Player can view all known NPC locations

---

### Phase 23E: DM Event System (Engine + Player) - Priority 3 ✅ ENGINE COMPLETE

**Goal**: DM can trigger approach events and location events.

**Duration**: 2-3 days
**Engine Completed**: 2024-12-18
**Player Status**: Pending (frontend implementation)

#### 23E.1: Approach Event Handler ✅

**Files**:
- `Engine/src/infrastructure/websocket.rs`
- `Engine/src/infrastructure/websocket/messages.rs`

**Tasks**:
- [x] Add `TriggerApproachEvent` to ClientMessage enum
- [x] Add `ApproachEvent` to ServerMessage enum
- [x] Handle `TriggerApproachEvent { npc_id, target_pc_id, description }`
- [x] Validate DM permission
- [x] Get NPC details (name, sprite)
- [x] Send `ApproachEvent` to session (broadcast)
- [x] Create Direct observation for PC

#### 23E.2: Location Event Handler ✅

**Files**:
- `Engine/src/infrastructure/websocket.rs`
- `Engine/src/infrastructure/websocket/messages.rs`

**Tasks**:
- [x] Add `TriggerLocationEvent` to ClientMessage enum
- [x] Add `LocationEvent` to ServerMessage enum
- [x] Handle `TriggerLocationEvent { region_id, description }`
- [x] Validate DM permission
- [x] Send `LocationEvent` to all clients in session (they filter by region)

#### 23E.3: Approach Event UI (Player)

**Files**:
- `Player/src/presentation/components/event_dialogue.rs` (NEW)

**Tasks**:
- [ ] Modal/overlay with event description
- [ ] "Continue" button dismisses
- [ ] After dismiss, NPC appears in scene (refresh NPC list)
- [ ] Trigger scene NPC refresh

#### 23E.4: Location Event UI (Player)

**Files**:
- `Player/src/presentation/components/event_dialogue.rs`

**Tasks**:
- [ ] Same modal but for location events (no NPC)
- [ ] Just displays narration text

#### 23E.5: DM Trigger Event Modal

**Files**:
- `Player/src/presentation/components/dm_panel/trigger_event_modal.rs` (NEW)

**Tasks**:
- [ ] Select NPC dropdown (for approach events)
- [ ] Select target PC dropdown
- [ ] Event description textarea
- [ ] "Trigger Approach Event" button
- [ ] "Trigger Location Event" button (no NPC selection)
- [ ] Success confirmation

**Acceptance Criteria** (Engine Complete):
- [x] DM can trigger approach events via WebSocket
- [x] ApproachEvent message sent to session
- [x] Direct observation created when NPC approaches
- [x] DM can trigger location events via WebSocket
- [x] LocationEvent message sent to session
- [ ] (Player) Player sees event dialogue
- [ ] (Player) NPC appears in scene after approach event

---

### Phase 23F: Game Time UI & Cache (Engine + Player) - Priority 3 ✅ ENGINE COMPLETE

**Goal**: DM can view and advance game time.

**Duration**: 1-2 days
**Engine Completed**: 2024-12-18
**Player Status**: Pending (frontend implementation)

#### 23F.1: Game Time HTTP Routes ✅

**Files**:
- `Engine/src/infrastructure/http/session_routes.rs`
- `Engine/src/infrastructure/http/mod.rs`

**Tasks**:
- [x] `GET /api/sessions/{session_id}/game-time` - Get current game time
- [x] `POST /api/sessions/{session_id}/game-time/advance` - Advance by hours/days
- [x] `GameTimeResponse` DTO with display, time_of_day, is_paused
- [x] `AdvanceGameTimeRequest` DTO with hours and days

#### 23F.2: Game Time WebSocket ✅

**Files**:
- `Engine/src/infrastructure/websocket.rs`
- `Engine/src/infrastructure/websocket/messages.rs`

**Tasks**:
- [x] Add `AdvanceGameTime` to ClientMessage enum
- [x] Add `GameTimeUpdated` to ServerMessage enum
- [x] Handle `AdvanceGameTime { hours }`
- [x] Update session's game time
- [x] Broadcast `GameTimeUpdated` to all clients in session

#### 23F.3: Game Time Display (Player)

**Files**:
- `Player/src/presentation/components/game_time_display.rs` (NEW)

**Tasks**:
- [ ] Show current game time in header
- [ ] Time of day icon (sun/moon)
- [ ] Update when `GameTimeUpdated` received

#### 23F.4: DM Time Controls

**Files**:
- `Player/src/presentation/components/dm_panel/time_controls.rs` (NEW)

**Tasks**:
- [ ] Quick advance buttons: +1h, +6h, +1d
- [ ] Custom hours input
- [ ] Warning about NPC presence changes
- [ ] Send `AdvanceGameTime` message

**Acceptance Criteria** (Engine Complete):
- [x] Game time HTTP routes for get and advance
- [x] AdvanceGameTime WebSocket handler
- [x] GameTimeUpdated broadcast to session
- [x] Engine compiles with no errors
- [ ] (Player) Game time displayed to all players
- [ ] (Player) DM time controls in UI
- [ ] Time advancement invalidates presence cache (future enhancement)
- [ ] Scenes may update after time change (future enhancement)

---

## Files to Remove

### Engine Files to Delete

| File | Reason |
|------|--------|
| `BackdropRegion` struct in `location.rs` | Replaced by `Region` entity |

### ID Renames

| Old | New |
|-----|-----|
| `BackdropRegionId` | `RegionId` |

---

## Files Summary

### New Files (Engine)

| File | Purpose |
|------|---------|
| `src/domain/entities/region.rs` | Region entity (replaces BackdropRegion) |
| `src/domain/entities/observation.rs` | NPC observation entity |
| `src/domain/value_objects/game_time.rs` | Game time system |
| `src/infrastructure/persistence/region_repository.rs` | Region Neo4j repo |
| `src/infrastructure/persistence/observation_repository.rs` | Observation repo |
| `src/infrastructure/http/region_routes.rs` | Region HTTP routes |
| `src/infrastructure/http/observation_routes.rs` | Observation HTTP routes |
| `src/application/services/presence_service.rs` | NPC presence queries + cache |

### Modified Files (Engine)

| File | Changes |
|------|---------|
| `src/domain/value_objects/ids.rs` | Rename `BackdropRegionId` → `RegionId` |
| `src/domain/entities/location.rs` | Add map_asset, parent_map_bounds, default_region_id; Remove BackdropRegion |
| `src/domain/entities/player_character.rs` | Add current_region_id, make session_id optional, remove starting_location_id |
| `src/domain/entities/session.rs` | Add game_time |
| `src/infrastructure/persistence/location_repository.rs` | Update for new fields, remove BackdropRegion methods |
| `src/infrastructure/persistence/player_character_repository.rs` | New query methods, schema updates |
| `src/infrastructure/persistence/character_repository.rs` | Region relationship methods |
| `src/infrastructure/http/player_character_routes.rs` | Selection/import routes |
| `src/infrastructure/http/session_routes.rs` | Game time routes |
| `src/infrastructure/websocket.rs` | New message handlers |
| `src/application/services/challenge_resolution_service.rs` | RevealNpcLocation trigger |

### New Files (Player)

| File | Purpose |
|------|---------|
| `src/presentation/views/character_select.rs` | Character selection screen |
| `src/presentation/components/character_select/import_modal.rs` | Import PC modal |
| `src/presentation/components/navigation_panel.rs` | Region navigation |
| `src/presentation/components/known_npcs_panel.rs` | NPC observations |
| `src/presentation/components/event_dialogue.rs` | Approach/location event display |
| `src/presentation/components/game_time_display.rs` | Time display |
| `src/presentation/components/dm_panel/trigger_event_modal.rs` | DM event trigger |
| `src/presentation/components/dm_panel/time_controls.rs` | DM time controls |

### Modified Files (Player)

| File | Changes |
|------|---------|
| `src/presentation/views/pc_creation.rs` | Spawn point regions, region_id selection |
| `src/presentation/views/pc_view.rs` | Scene display integration |
| `src/routes/player_routes.rs` | Character select routing |
| `src/presentation/handlers/session_message_handler.rs` | New message handlers |

---

## Testing Strategy

### Unit Tests

- Region CRUD operations
- MapBounds containment logic
- PC selection and import logic
- Game time calculations and TTL
- Observation CRUD

### Integration Tests

- Region connections and navigation
- Region exits to locations with arrival_region_id
- NPC presence query with LLM mock
- Observation creation on scene view
- Time advancement and cache invalidation

### Manual Testing Scenarios

1. **New Player Flow**
   - Join session → See character select → Create PC → Select spawn region → Enter scene
   
2. **Returning Player Flow**
   - Join session → See character select → Select existing PC → Enter scene at last region

3. **Import Flow**
   - Character select → Import → Select world → Select PC → Select spawn → Play

4. **Navigation Flow**
   - In bar entrance → Click "Bar Counter" → Scene changes → See bartender
   - In bar counter → Click "Market District" (exit) → Arrive at Market Square

5. **Approach Event Flow**
   - DM triggers approach → Player sees dialogue → NPC appears in scene

6. **Time Advancement Flow**
   - DM advances 6 hours → Day becomes night → Query new NPC presence

---

## Dependencies

### Completed Dependencies
- ✅ Phase 0: Neo4j Data Model
- ✅ Phase 3: Scene System (export, interactions)
- ✅ Phase 13: World Selection
- ✅ Phase 21: PC Creation

### New Dependencies
- LLM service must support presence queries
- WebSocket must support new message types

---

## Out of Scope (Future Phases)

- Real-time game time scaling (auto-advance)
- NPC pathfinding/travel simulation
- Combat system (separate phase)
- PC-to-PC observation sharing
- Region fog of war (unexplored regions hidden)
- NPC schedules with multiple shifts per day
- Map editor with drag-and-drop region bounds

---

## Design Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| PC persistence | Per-user, per-world | Players can have multiple PCs, persist across sessions |
| Character import | Copy from worlds with same rule system | Reuse characters without affecting source |
| Region vs BackdropRegion | Replace entirely | Cleaner design, no backward compat needed |
| Regions nest? | No, only locations nest | Simpler hierarchy, regions are leaf nodes |
| NPC position tracking | LLM-determined, not simulated | Story-driven, no scheduling complexity |
| Observation types | Direct, HeardAbout, Deduced | Covers all discovery methods |
| LLM cache TTL | Game-time based | Real time doesn't matter, story time does |
| Exit connections | Specify arrival_region_id | Deterministic navigation |
| Default region | Per-location fallback | Graceful handling of unspecified arrivals |
