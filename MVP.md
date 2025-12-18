# WrldBldr MVP Plan

**Created:** 2025-12-17
**Status:** ACTIVE - Primary Implementation Reference
**Target:** Playable TTRPG game loop without tactical combat

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Project Vision](#2-project-vision)
3. [System Architecture](#3-system-architecture)
4. [Neo4j Data Model Foundation](#4-neo4j-data-model-foundation)
5. [LLM Context System Design](#5-llm-context-system-design)
   - 5.1 [Region & Navigation System](#51-region--navigation-system)
   - 5.2 [NPC Presence System](#52-npc-presence-system)
   - 5.3 [Game Time System](#53-game-time-system)
   - 5.4 [Observation System](#54-observation-system)
   - 5.5 [DM Event System](#55-dm-event-system)
6. [Current State Assessment](#6-current-state-assessment)
7. [MVP Scope Definition](#7-mvp-scope-definition)
8. [Implementation Roadmap](#8-implementation-roadmap)
9. [Architecture Compliance Rules](#9-architecture-compliance-rules)
10. [Acceptance Criteria](#10-acceptance-criteria)
11. [Progress Tracking](#11-progress-tracking)

---

## 1. Executive Summary

### Goal

Deliver a **playable TTRPG game loop** where:
1. Players interact with NPCs through a visual novel interface
2. An LLM (Ollama) generates NPC responses informed by deep narrative context
3. The DM approves/modifies all AI-generated content before players see it
4. Character motivations, relationships, and narrative arcs drive emergent storytelling

### Key Innovations

1. **Pure Neo4j Graph Model**: All entities and their relationships stored as nodes and edges - no JSON blobs for relational data
2. **Per-Character Actantial Model**: Every character (PC/NPC) has their own view of who helps them, opposes them, and what they desire
3. **Rich Location Relationships**: Characters have HOME_LOCATION, WORKS_AT, FREQUENTS relationships with locations
4. **Challenge-Location Binding**: Challenges can be location-bound and unlocked by events
5. **Context-Aware LLM**: Rich narrative context with configurable token budgets and automatic summarization
6. **Dual Trigger System**: Both Engine and LLM can suggest narrative event triggers

### Out of Scope

- Tactical combat (grid maps, turn order, pathfinding)
- Multi-platform builds (WASM, Android) - desktop only
- Save/Load system (session state persists in Neo4j, no explicit save files)

---

## 2. Project Vision

### The Game Loop

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           WrldBldr Game Loop                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  PLAYER  │───▶│  ENGINE  │───▶│   LLM    │───▶│    DM    │              │
│  │  Action  │    │ + Context│    │ Response │    │ Approval │              │
│  └──────────┘    └──────────┘    └──────────┘    └────┬─────┘              │
│       ▲                                               │                     │
│       │         ┌─────────────────────────────────────┘                     │
│       │         ▼                                                           │
│       │    ┌──────────┐    ┌──────────┐    ┌──────────┐                    │
│       │    │ Approved │───▶│  Tools   │───▶│  State   │                    │
│       └────│ Response │    │ Execute  │    │ Updated  │                    │
│            └──────────┘    └──────────┘    └──────────┘                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Detailed Flow

1. **Player performs action** (speak to NPC, examine object, use item)
2. **Engine builds rich context** via graph traversal:
   - Scene information (location, time, atmosphere)
   - NPC's actantial model (wants, helpers, opponents)
   - NPC's location relationships (where they live, work, frequent)
   - NPC's relationships and sentiment toward PC
   - Active narrative events and their triggers
   - Available challenges at this location
   - Conversation history
   - DM's directorial notes
3. **Context budget enforced**:
   - Each category has a MAX_TOKENS limit
   - If exceeded, a summarization LLM call condenses the content
4. **LLM generates response**:
   - NPC dialogue (shown to player)
   - Internal reasoning (shown to DM only)
   - Tool call suggestions (give item, change relationship, etc.)
   - Challenge suggestions (skill checks)
   - Narrative event trigger suggestions
5. **DM reviews and decides**:
   - Accept as-is
   - Modify dialogue or tool calls
   - Reject and provide feedback for regeneration
   - Take over and write response manually
6. **Approved content delivered**:
   - Player sees NPC dialogue
   - Tool calls execute (items given, relationships updated)
   - StoryEvent recorded to timeline
   - Narrative events triggered if conditions met

### Campbell's Monomyth Integration

Each **Act** in a world corresponds to a stage of the Hero's Journey:

| Stage | Name | Narrative Function |
|-------|------|-------------------|
| 1 | Ordinary World | Hero's normal life before adventure |
| 2 | Call to Adventure | Something disrupts the ordinary world |
| 3 | Refusal of the Call | Hero hesitates, showing humanity |
| 4 | Meeting the Mentor | A guide appears to prepare the hero |
| 5 | Crossing the Threshold | Hero commits to the adventure |
| 6 | Tests, Allies, Enemies | Hero learns rules of the new world |
| 7 | Approach to Inmost Cave | Hero prepares for central ordeal |
| 8 | Ordeal | Hero faces greatest fear |
| 9 | Reward | Hero gains the treasure |
| 10 | The Road Back | Hero deals with consequences |
| 11 | Resurrection | Hero is tested once more, transformed |
| 12 | Return with Elixir | Hero returns with new wisdom |

Characters have **Campbell Archetypes**:
- **Hero** - Protagonist undergoing transformation
- **Mentor** - Wise guide providing knowledge/training
- **Threshold Guardian** - Tests hero's commitment at boundaries
- **Herald** - Announces change and call to adventure
- **Shapeshifter** - Changes loyalties, creates doubt
- **Shadow** - Represents dark side, often the villain
- **Trickster** - Brings humor/chaos, challenges thinking
- **Ally** - Provides support and companionship

### Greimas Actantial Model Integration

Each character has their own **actantial model** - their view of who plays what role in their personal narrative:

```
        SENDER ──────────────▶ OBJECT ◀────────────── RECEIVER
     (who motivated           (what they              (who benefits
      the quest)               desire)                if they succeed)
           │                      ▲                        │
           │                      │                        │
           ▼                      │                        ▼
        HELPER ◀──────────── SUBJECT ─────────────▶ OPPONENT
     (who aids them)        (the character         (who opposes them)
                             themselves)
```

**Key Insight**: The same person can be a HELPER in one character's model and an OPPONENT in another's. These are **one-way, per-character relationships**.

---

## 3. System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              WrldBldr System                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐│
│  │           ENGINE                │    │           PLAYER                ││
│  │         (Rust/Axum)             │◀──▶│        (Rust/Dioxus)            ││
│  │                                 │ WS │                                 ││
│  │  ┌───────────────────────────┐  │    │  ┌───────────────────────────┐  ││
│  │  │     Application Layer     │  │    │  │     Presentation Layer    │  ││
│  │  │  - Services               │  │    │  │  - PC View (Visual Novel) │  ││
│  │  │  - Use Cases              │  │    │  │  - DM View (Director)     │  ││
│  │  │  - DTOs                   │  │    │  │  - Creator Mode           │  ││
│  │  └───────────────────────────┘  │    │  └───────────────────────────┘  ││
│  │              │                  │    │              │                  ││
│  │  ┌───────────────────────────┐  │    │  ┌───────────────────────────┐  ││
│  │  │       Domain Layer        │  │    │  │     Application Layer     │  ││
│  │  │  - Entities               │  │    │  │  - Services               │  ││
│  │  │  - Value Objects          │  │    │  │  - Ports                  │  ││
│  │  │  - Domain Services        │  │    │  │  - DTOs                   │  ││
│  │  └───────────────────────────┘  │    │  └───────────────────────────┘  ││
│  │              │                  │    │              │                  ││
│  │  ┌───────────────────────────┐  │    │  ┌───────────────────────────┐  ││
│  │  │   Infrastructure Layer    │  │    │  │   Infrastructure Layer    │  ││
│  │  │  - Neo4j Repositories     │  │    │  │  - WebSocket Client       │  ││
│  │  │  - Ollama Client          │  │    │  │  - HTTP Client            │  ││
│  │  │  - ComfyUI Client         │  │    │  │  - Asset Loader           │  ││
│  │  │  - WebSocket Server       │  │    │  └───────────────────────────┘  ││
│  │  └───────────────────────────┘  │    │                                 ││
│  └─────────────────────────────────┘    └─────────────────────────────────┘│
│                 │                                                           │
│  ┌──────────────┴──────────────┐    ┌─────────────────────────────────────┐│
│  │           Neo4j             │    │           Ollama                    ││
│  │      (Graph Database)       │    │        (LLM Server)                 ││
│  └─────────────────────────────┘    └─────────────────────────────────────┘│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Hexagonal Architecture (Engine)

```
                    ┌─────────────────────────────────────┐
                    │         Infrastructure Layer        │
                    │  ┌─────────┐  ┌─────────┐          │
      HTTP ────────▶│  │  HTTP   │  │WebSocket│ ◀─────── WebSocket
                    │  │ Routes  │  │ Server  │          │
                    │  └────┬────┘  └────┬────┘          │
                    │       │            │               │
                    └───────┼────────────┼───────────────┘
                            │            │
                    ┌───────┴────────────┴───────────────┐
                    │         Application Layer          │
                    │  ┌─────────────────────────────┐   │
                    │  │         Services            │   │
                    │  │  - WorldService             │   │
                    │  │  - CharacterService         │   │
                    │  │  - LocationService          │   │
                    │  │  - ChallengeService         │   │
                    │  │  - NarrativeEventService    │   │
                    │  │  - LLMContextService (NEW)  │   │
                    │  └─────────────────────────────┘   │
                    │               │                    │
                    │  ┌────────────┴────────────────┐   │
                    │  │      Ports (Traits)         │   │
                    │  │  - CharacterRepositoryPort  │   │
                    │  │  - LocationRepositoryPort   │   │
                    │  │  - WantRepositoryPort (NEW) │   │
                    │  │  - ChallengeRepositoryPort  │   │
                    │  │  - LlmPort                  │   │
                    │  └─────────────────────────────┘   │
                    └───────────────┬────────────────────┘
                                    │
                    ┌───────────────┴────────────────────┐
                    │          Domain Layer              │
                    │  ┌─────────────────────────────┐   │
                    │  │        Entities             │   │
                    │  │  - Character (refactored)   │   │
                    │  │  - Location (refactored)    │   │
                    │  │  - Want (NEW entity)        │   │
                    │  │  - Challenge (refactored)   │   │
                    │  │  - NarrativeEvent           │   │
                    │  │  - StoryEvent               │   │
                    │  └─────────────────────────────┘   │
                    │  ┌─────────────────────────────┐   │
                    │  │      Value Objects          │   │
                    │  │  - ActantialRole (NEW)      │   │
                    │  │  - CampbellArchetype        │   │
                    │  │  - ContextBudget (NEW)      │   │
                    │  └─────────────────────────────┘   │
                    └────────────────────────────────────┘
                                    │
                    ┌───────────────┴────────────────────┐
                    │       Infrastructure Layer         │
                    │  ┌─────────┐  ┌─────────┐         │
                    │  │  Neo4j  │  │ Ollama  │         │
                    │  │  Repos  │  │ Client  │         │
                    │  └─────────┘  └─────────┘         │
                    └────────────────────────────────────┘
```

### Data Flow Rules

1. **Domain Layer**: Pure business logic, no external dependencies
2. **Application Layer**: Orchestrates use cases, depends only on domain and ports
3. **Infrastructure Layer**: Implements ports, handles external systems
4. **Presentation Layer** (Player only): UI components, depends on application services

**Import Rules**:
- Domain NEVER imports from Application, Infrastructure, or Presentation
- Application NEVER imports from Infrastructure or Presentation
- Infrastructure implements Application ports
- Presentation uses Application services

---

## 4. Neo4j Data Model Foundation

This section defines the **complete graph schema** for WrldBldr. The goal is to maximize Neo4j's graph capabilities by storing all relationships as edges rather than JSON blobs.

### Design Principles

1. **Edges for Relationships**: Any reference to another entity becomes a Neo4j edge
2. **Properties on Edges**: Relationship metadata (timestamps, reasons, quantities) stored on edges
3. **JSON Only for Non-Relational Data**: Configuration blobs, spatial data (GridMap tiles), deeply nested templates
4. **Query-First Design**: Schema optimized for common graph traversals

### Acceptable JSON Blobs (Per ADR-001)

These remain as JSON properties because they don't represent relationships:

| Entity | JSON Field | Reason |
|--------|------------|--------|
| GridMap | `tiles` | 2D spatial data, not relational |
| CharacterSheetTemplate | `sections` | Deeply nested template structure |
| CharacterSheetData | Full sheet | Per ADR-001, form data |
| WorkflowConfiguration | `workflow_json` | ComfyUI workflow format |
| RuleSystemConfig | Full config | System configuration |
| DirectorialNotes | Full notes | Scene metadata |

---

### 4.1 Node Types

#### Core World Structure

```cypher
// World - Top-level container
(:World {
    id: "uuid",
    name: "The Shattered Realms",
    description: "A world torn apart by ancient magic...",
    rule_system: "{...}",  // JSON - RuleSystemConfig
    created_at: datetime(),
    updated_at: datetime()
})

// Act - Story arc (Monomyth stage)
(:Act {
    id: "uuid",
    name: "The Call",
    stage: "CallToAdventure",  // MonomythStage enum
    description: "The heroes receive their summons...",
    order: 1
})

// Goal - Abstract desire target
(:Goal {
    id: "uuid",
    name: "Family Honor Restored",
    description: "The stain on the family name is cleansed"
})
```

#### Location System

```cypher
// Location - Physical or conceptual place
(:Location {
    id: "uuid",
    name: "The Rusty Anchor Tavern",
    description: "A dimly lit tavern frequented by sailors...",
    location_type: "Interior",  // Interior, Exterior, Abstract
    backdrop_asset: "/assets/backdrops/tavern.png",
    atmosphere: "Smoky, raucous, smells of ale and salt"
})

// Region - Sub-location within a location (replaces BackdropRegion)
// Regions are the "screens" players navigate between in JRPG-style exploration
(:Region {
    id: "uuid",
    name: "The Bar Counter",
    description: "A worn wooden counter with brass fittings",
    backdrop_asset: "/assets/backdrops/bar_counter.png",
    atmosphere: "Smoky, the barkeep polishes glasses",
    // Position on parent location's map (clickable area)
    map_bounds_x: 100,
    map_bounds_y: 200,
    map_bounds_width: 300,
    map_bounds_height: 150,
    is_spawn_point: false,  // Can PCs start here?
    order: 1
})

// GridMap - Tactical map (combat - out of MVP scope but schema included)
(:GridMap {
    id: "uuid",
    name: "Tavern Ground Floor",
    width: 20,
    height: 15,
    tile_size: 32,
    tilesheet_asset: "/assets/tilesets/interior.png",
    tiles: "[...]"  // JSON - 2D tile array (acceptable)
})
```

#### Character System

```cypher
// Character - NPC
(:Character {
    id: "uuid",
    name: "Marcus the Redeemed",
    description: "A former mercenary seeking redemption...",
    sprite_asset: "/assets/sprites/marcus.png",
    portrait_asset: "/assets/portraits/marcus.png",
    base_archetype: "Ally",
    current_archetype: "Mentor",
    is_alive: true,
    is_active: true
})

// PlayerCharacter - PC
(:PlayerCharacter {
    id: "uuid",
    user_id: "user-123",
    name: "Kira Shadowblade",
    description: "A vengeful warrior...",
    sprite_asset: "/assets/sprites/kira.png",
    portrait_asset: "/assets/portraits/kira.png",
    sheet_data: "{...}",  // JSON - CharacterSheetData (acceptable per ADR-001)
    created_at: datetime(),
    last_active_at: datetime()
})

// Want - Character desire (Actantial model)
(:Want {
    id: "uuid",
    description: "Avenge my family's murder",
    intensity: 0.9,           // 0.0 = mild, 1.0 = obsession
    known_to_player: false,
    created_at: datetime()
})

// Item - Object that can be possessed
(:Item {
    id: "uuid",
    name: "Sword of the Fallen",
    description: "A blade that once belonged to a fallen hero",
    item_type: "Weapon",
    is_unique: true,
    properties: "{...}"  // JSON - item-specific properties
})

// Skill - Ability that can be tested
(:Skill {
    id: "uuid",
    name: "Persuasion",
    description: "The ability to influence others through words",
    category: "Social",
    base_attribute: "Charisma",
    is_custom: false
})
```

#### Scene & Interaction System

```cypher
// Scene - Narrative unit
(:Scene {
    id: "uuid",
    name: "Meeting the Informant",
    time_context: "Evening",
    backdrop_override: null,  // Use location's backdrop if null
    directorial_notes: "{...}",  // JSON - DirectorialNotes (acceptable)
    order: 1
})

// InteractionTemplate - Available action in scene
(:InteractionTemplate {
    id: "uuid",
    name: "Ask about the Baron",
    interaction_type: "Dialogue",
    prompt_hints: "The informant knows secrets about the Baron's past",
    is_available: true,
    order: 1
})
```

#### Challenge System

```cypher
// Challenge - Skill check
(:Challenge {
    id: "uuid",
    name: "Convince the Guard",
    description: "Persuade the guard to let you pass",
    challenge_type: "Social",
    difficulty: "Medium",
    difficulty_class: 15,
    active: true,
    is_favorite: false,
    order: 1
})

// ChallengeOutcome - Possible result (stored on Challenge as JSON for now)
// Note: Outcomes are complex nested structures, keeping as JSON
```

#### Event System

```cypher
// NarrativeEvent - DM-designed future event
(:NarrativeEvent {
    id: "uuid",
    name: "The Baron's Arrival",
    description: "The Baron unexpectedly arrives at the tavern",
    scene_direction: "The door swings open, and silence falls...",
    suggested_opening: "Well, well... what have we here?",
    is_active: true,
    is_triggered: false,
    is_repeatable: false,
    trigger_count: 0,
    delay_turns: 0,
    priority: 10,
    is_favorite: true,
    created_at: datetime()
})

// StoryEvent - Immutable record of past event
(:StoryEvent {
    id: "uuid",
    event_type: "DialogueExchange",
    timestamp: datetime(),
    game_time: "Day 3, Evening",
    summary: "Kira spoke with Marcus about the Baron",
    is_hidden: false,
    tags: ["plot", "baron", "marcus"]
})

// EventChain - Linked sequence of narrative events
(:EventChain {
    id: "uuid",
    name: "The Baron's Downfall",
    description: "Events leading to the Baron's defeat",
    is_active: true,
    current_position: 0,
    color: "#FF5733",
    is_favorite: true
})
```

---

### 4.2 Edge Types

#### World Structure Edges

```cypher
// World contains entities
(world:World)-[:CONTAINS_ACT {order: 1}]->(act:Act)
(world:World)-[:CONTAINS_LOCATION]->(location:Location)
(world:World)-[:CONTAINS_CHARACTER]->(character:Character)
(world:World)-[:CONTAINS_SKILL]->(skill:Skill)
(world:World)-[:CONTAINS_ITEM]->(item:Item)
(world:World)-[:CONTAINS_CHALLENGE]->(challenge:Challenge)
(world:World)-[:CONTAINS_NARRATIVE_EVENT]->(event:NarrativeEvent)
(world:World)-[:CONTAINS_EVENT_CHAIN]->(chain:EventChain)
(world:World)-[:CONTAINS_GOAL]->(goal:Goal)

// Act contains scenes
(act:Act)-[:CONTAINS_SCENE {order: 1}]->(scene:Scene)
```

#### Location Hierarchy & Navigation

```cypher
// Location hierarchy (parent contains child)
(parent:Location)-[:CONTAINS_LOCATION]->(child:Location)
// Example: Town CONTAINS Tavern, Tavern CONTAINS Back Room

// Location connections (navigation)
(from:Location)-[:CONNECTED_TO {
    connection_type: "Door",      // Door, Path, Stairs, Portal, etc.
    description: "A heavy oak door",
    bidirectional: true,
    travel_time: 0,               // In game-time units
    is_locked: false,
    unlock_requirements: null     // JSON if complex requirements
}]->(to:Location)

// Location has tactical map
(location:Location)-[:HAS_TACTICAL_MAP]->(map:GridMap)

// Location has regions (sub-locations)
(location:Location)-[:HAS_REGION]->(region:Region)

// Region to region navigation (within same location)
(region:Region)-[:CONNECTED_TO_REGION {
    description: "A door leads to the back room",
    bidirectional: true,
    is_locked: false,
    lock_description: null
}]->(other:Region)

// Region exits to parent/sibling location
(region:Region)-[:EXITS_TO_LOCATION {
    description: "Step outside into the market",
    arrival_region_id: "uuid",  // Which region in target location
    bidirectional: true
}]->(location:Location)
```

#### Character-Location Relationships

```cypher
// PC current position (location + region)
(pc:PlayerCharacter)-[:CURRENTLY_AT]->(location:Location)
(pc:PlayerCharacter)-[:CURRENTLY_IN_REGION]->(region:Region)

// PC starting position
(pc:PlayerCharacter)-[:STARTED_AT]->(location:Location)
(pc:PlayerCharacter)-[:STARTED_IN_REGION]->(region:Region)

// NPC home location
(npc:Character)-[:HOME_LOCATION {
    description: "Lives in the apartment above the tavern"
}]->(location:Location)

// NPC work location
(npc:Character)-[:WORKS_AT {
    role: "Bartender",
    schedule: "Evenings"
}]->(location:Location)

// NPC frequents location (with schedule)
(npc:Character)-[:FREQUENTS {
    frequency: "Often",           // Rarely, Sometimes, Often, Always
    time_of_day: "Evening",       // Morning, Afternoon, Evening, Night, Any
    day_of_week: "Any",           // Specific days or "Any"
    reason: "Meets contacts here",
    since: datetime()
}]->(location:Location)

// NPC avoids location
(npc:Character)-[:AVOIDS {
    reason: "Bad memories of a fight"
}]->(location:Location)

// NPC region-level relationships (more granular than location)
(npc:Character)-[:WORKS_AT_REGION {
    shift: "day",              // day, night, always
    role: "Bartender"
}]->(region:Region)

(npc:Character)-[:FREQUENTS_REGION {
    frequency: "often",        // often, sometimes, rarely
    time_of_day: "Evening"
}]->(region:Region)

(npc:Character)-[:HOME_REGION]->(region:Region)

(npc:Character)-[:AVOIDS_REGION {
    reason: "Was beaten here once"
}]->(region:Region)
```

#### Character-Character Relationships (Social)

```cypher
// Social relationship (sentiment-based)
(from:Character)-[:RELATES_TO {
    relationship_type: "Rivalry",  // Family, Romantic, Professional, Rivalry, Friendship, Mentorship, Enmity
    sentiment: -0.7,               // -1.0 (hatred) to 1.0 (love)
    known_to_player: true,
    established_at: datetime()
}]->(to:Character)

// Archetype change history (self-referential with timestamp)
(character:Character)-[:ARCHETYPE_CHANGED {
    from_archetype: "Hero",
    to_archetype: "Shadow",
    reason: "Consumed by vengeance",
    changed_at: datetime(),
    order: 1
}]->(character:Character)
```

#### Character-Item Relationships (Inventory)

```cypher
// Character possesses item
(character:Character)-[:POSSESSES {
    quantity: 1,
    equipped: true,
    acquired_at: datetime(),
    acquisition_method: "Found"   // Found, Purchased, Gifted, Looted, Crafted
}]->(item:Item)

// PC possesses item (same structure)
(pc:PlayerCharacter)-[:POSSESSES {
    quantity: 3,
    equipped: false,
    acquired_at: datetime()
}]->(item:Item)
```

#### Actantial Model Relationships

```cypher
// Character has a want
(character:Character)-[:HAS_WANT {
    priority: 1,                  // 1 = primary want
    acquired_at: datetime()
}]->(want:Want)

// Want targets something (the OBJECT in actantial terms)
(want:Want)-[:TARGETS]->(target)
// Where target can be:
//   (:Character) - wants something FROM a person
//   (:Item) - wants a specific item
//   (:Goal) - wants an abstract outcome

// Actantial role assignments (per want, directional, from subject's POV)
(subject:Character)-[:VIEWS_AS_HELPER {
    want_id: "uuid",              // Which want this relates to
    reason: "Saved my life",
    assigned_at: datetime()
}]->(helper:Character)

(subject:Character)-[:VIEWS_AS_OPPONENT {
    want_id: "uuid",
    reason: "Killed my family",
    assigned_at: datetime()
}]->(opponent:Character)

(subject:Character)-[:VIEWS_AS_SENDER {
    want_id: "uuid",
    reason: "My father's dying wish",
    assigned_at: datetime()
}]->(sender:Character)

(subject:Character)-[:VIEWS_AS_RECEIVER {
    want_id: "uuid",
    reason: "My village will be safe",
    assigned_at: datetime()
}]->(receiver:Character)
```

#### Scene & Interaction Edges

```cypher
// Scene is at a location
(scene:Scene)-[:AT_LOCATION]->(location:Location)

// Scene belongs to act
(scene:Scene)-[:BELONGS_TO_ACT]->(act:Act)

// Scene features characters
(scene:Scene)-[:FEATURES_CHARACTER {
    role: "Primary",              // Primary, Secondary, Background
    entrance_description: "Already present when scene begins"
}]->(character:Character)

// Interaction belongs to scene
(interaction:InteractionTemplate)-[:BELONGS_TO_SCENE]->(scene:Scene)

// Interaction targets entity
(interaction:InteractionTemplate)-[:TARGETS_CHARACTER]->(character:Character)
(interaction:InteractionTemplate)-[:TARGETS_ITEM]->(item:Item)
(interaction:InteractionTemplate)-[:TARGETS_REGION]->(region:BackdropRegion)

// Interaction requires conditions
(interaction:InteractionTemplate)-[:REQUIRES_ITEM]->(item:Item)
(interaction:InteractionTemplate)-[:REQUIRES_CHARACTER_PRESENT]->(character:Character)
(interaction:InteractionTemplate)-[:REQUIRES_FLAG {flag_name: "met_baron", value: true}]->(interaction)
```

#### Challenge Edges

```cypher
// Challenge requires skill
(challenge:Challenge)-[:REQUIRES_SKILL]->(skill:Skill)

// Challenge is available at location (location-wide)
(challenge:Challenge)-[:AVAILABLE_AT_LOCATION {
    always_available: false,
    time_restriction: "Evening"   // null for any time
}]->(location:Location)

// Challenge is available at specific region (more granular)
(challenge:Challenge)-[:AVAILABLE_AT_REGION {
    always_available: false,
    time_restriction: null
}]->(region:Region)

// Challenge prerequisite (must complete first)
(challenge:Challenge)-[:REQUIRES_COMPLETION_OF]->(prerequisite:Challenge)

// Challenge tied to scene (optional)
(challenge:Challenge)-[:TIED_TO_SCENE]->(scene:Scene)

// Challenge success unlocks location connection
(challenge:Challenge)-[:ON_SUCCESS_UNLOCKS]->(location:Location)
// This would unlock a CONNECTED_TO edge's is_locked property

// Challenge enabled/disabled by events
(event:NarrativeEvent)-[:ENABLES_CHALLENGE]->(challenge:Challenge)
(event:NarrativeEvent)-[:DISABLES_CHALLENGE]->(challenge:Challenge)
```

#### Narrative Event Edges

```cypher
// Narrative event tied to location/region/scene/act (optional)
(event:NarrativeEvent)-[:TIED_TO_LOCATION]->(location:Location)
(event:NarrativeEvent)-[:TIED_TO_REGION]->(region:Region)
(event:NarrativeEvent)-[:TIED_TO_SCENE]->(scene:Scene)
(event:NarrativeEvent)-[:BELONGS_TO_ACT]->(act:Act)

// Narrative event features NPCs
(event:NarrativeEvent)-[:FEATURES_NPC]->(character:Character)

// Event chain membership
(chain:EventChain)-[:CONTAINS_EVENT {
    position: 1,
    is_completed: false
}]->(event:NarrativeEvent)

// Event chains to another event
(event:NarrativeEvent)-[:CHAINS_TO {
    delay_turns: 2,
    chain_reason: "Baron retaliates after being exposed"
}]->(next:NarrativeEvent)

// Event trigger conditions (edges for entity-based conditions)
(event:NarrativeEvent)-[:TRIGGERED_BY_ENTERING_LOCATION]->(location:Location)
(event:NarrativeEvent)-[:TRIGGERED_BY_ENTERING_REGION]->(region:Region)
(event:NarrativeEvent)-[:TRIGGERED_BY_TALKING_TO]->(character:Character)
(event:NarrativeEvent)-[:TRIGGERED_BY_CHALLENGE_COMPLETE {success_required: true}]->(challenge:Challenge)
(event:NarrativeEvent)-[:TRIGGERED_BY_EVENT_COMPLETE {outcome_required: "success"}]->(prev:NarrativeEvent)

// Event effects (edges for entity-based effects)
(event:NarrativeEvent)-[:EFFECT_GIVES_ITEM {outcome: "success", quantity: 1}]->(item:Item)
(event:NarrativeEvent)-[:EFFECT_MODIFIES_RELATIONSHIP {
    outcome: "success",
    sentiment_change: 0.3
}]->(character:Character)
(event:NarrativeEvent)-[:EFFECT_TRIGGERS_SCENE]->(scene:Scene)
```

#### Story Event (Timeline) Edges

```cypher
// Story event occurred in session
(event:StoryEvent)-[:OCCURRED_IN_SESSION]->(session:Session)

// Story event occurred at location
(event:StoryEvent)-[:OCCURRED_AT]->(location:Location)

// Story event occurred in scene
(event:StoryEvent)-[:OCCURRED_IN_SCENE]->(scene:Scene)

// Story event involves characters
(event:StoryEvent)-[:INVOLVES {
    role: "Speaker"               // Speaker, Target, Witness, etc.
}]->(character:Character)

// Story event triggered by narrative event
(event:StoryEvent)-[:TRIGGERED_BY]->(narrative:NarrativeEvent)

// Story event records challenge attempt
(event:StoryEvent)-[:RECORDS_CHALLENGE]->(challenge:Challenge)
```

#### Session Edges

```cypher
// Session uses world
(session:Session)-[:USES_WORLD]->(world:World)

// Session has player characters
(session:Session)-[:HAS_PLAYER_CHARACTER]->(pc:PlayerCharacter)

// Player character is in world
(pc:PlayerCharacter)-[:PLAYS_IN]->(world:World)
```

---

### 4.3 Example: Complete Character Graph

Here's how Kira the Avenger would be represented:

```cypher
// Create Kira
CREATE (kira:Character {
    id: "kira-001",
    name: "Kira the Avenger",
    description: "A warrior consumed by vengeance...",
    base_archetype: "Hero",
    current_archetype: "Shadow",
    is_alive: true,
    is_active: true
})

// Create her want
CREATE (revenge:Want {
    id: "want-001",
    description: "Avenge my family's murder",
    intensity: 0.9,
    known_to_player: false
})

// Create the target of her want (the Baron)
CREATE (baron:Character {
    id: "baron-001",
    name: "Baron Valdris",
    base_archetype: "Shadow",
    current_archetype: "Shadow"
})

// Connect want to Kira
CREATE (kira)-[:HAS_WANT {priority: 1}]->(revenge)

// Want targets the Baron
CREATE (revenge)-[:TARGETS]->(baron)

// Kira views Baron as opponent
CREATE (kira)-[:VIEWS_AS_OPPONENT {
    want_id: "want-001",
    reason: "Murdered my family"
}]->(baron)

// Create Marcus (ally)
CREATE (marcus:Character {
    id: "marcus-001",
    name: "Marcus the Redeemed"
})

// Kira views Marcus as helper
CREATE (kira)-[:VIEWS_AS_HELPER {
    want_id: "want-001",
    reason: "Saved my life from the Baron's guards"
}]->(marcus)

// Kira's location relationships
CREATE (tavern:Location {id: "loc-001", name: "The Rusty Anchor"})
CREATE (kira)-[:FREQUENTS {
    frequency: "Often",
    time_of_day: "Evening",
    reason: "Gathering information on the Baron"
}]->(tavern)

// Kira possesses a sword
CREATE (sword:Item {id: "item-001", name: "Father's Blade"})
CREATE (kira)-[:POSSESSES {
    quantity: 1,
    equipped: true,
    acquisition_method: "Inherited"
}]->(sword)

// Archetype change history
CREATE (kira)-[:ARCHETYPE_CHANGED {
    from_archetype: "Hero",
    to_archetype: "Shadow",
    reason: "Consumed by vengeance after witnessing family's murder",
    changed_at: datetime(),
    order: 1
}]->(kira)
```

---

### 4.4 Key Graph Queries

#### Get Character's Full Context for LLM

```cypher
// Get NPC with all context for LLM
MATCH (npc:Character {id: $npc_id})

// Get their wants and targets
OPTIONAL MATCH (npc)-[hw:HAS_WANT]->(want:Want)
OPTIONAL MATCH (want)-[:TARGETS]->(target)

// Get their actantial relationships
OPTIONAL MATCH (npc)-[vh:VIEWS_AS_HELPER]->(helper:Character)
OPTIONAL MATCH (npc)-[vo:VIEWS_AS_OPPONENT]->(opponent:Character)
OPTIONAL MATCH (npc)-[vs:VIEWS_AS_SENDER]->(sender:Character)
OPTIONAL MATCH (npc)-[vr:VIEWS_AS_RECEIVER]->(receiver:Character)

// Get their location relationships
OPTIONAL MATCH (npc)-[home:HOME_LOCATION]->(homeLoc:Location)
OPTIONAL MATCH (npc)-[work:WORKS_AT]->(workLoc:Location)
OPTIONAL MATCH (npc)-[freq:FREQUENTS]->(freqLoc:Location)

// Get social relationships
OPTIONAL MATCH (npc)-[rel:RELATES_TO]->(other:Character)

// Get inventory
OPTIONAL MATCH (npc)-[poss:POSSESSES]->(item:Item)

RETURN npc,
       collect(DISTINCT {want: want, target: target, priority: hw.priority}) as wants,
       collect(DISTINCT {helper: helper.name, reason: vh.reason}) as helpers,
       collect(DISTINCT {opponent: opponent.name, reason: vo.reason}) as opponents,
       collect(DISTINCT {home: homeLoc.name}) as home,
       collect(DISTINCT {work: workLoc.name, role: work.role}) as workplaces,
       collect(DISTINCT {location: freqLoc.name, when: freq.time_of_day, reason: freq.reason}) as frequents,
       collect(DISTINCT {character: other.name, type: rel.relationship_type, sentiment: rel.sentiment}) as relationships,
       collect(DISTINCT {item: item.name, equipped: poss.equipped}) as inventory
```

#### Find NPCs at Location (with schedule awareness)

```cypher
// Find NPCs who should be at this location at this time
MATCH (location:Location {id: $location_id})

// NPCs who live here
OPTIONAL MATCH (npc:Character)-[:HOME_LOCATION]->(location)
WHERE npc.is_active = true

// NPCs who work here (check schedule)
OPTIONAL MATCH (worker:Character)-[w:WORKS_AT]->(location)
WHERE worker.is_active = true
  AND (w.schedule IS NULL OR w.schedule = $time_of_day)

// NPCs who frequent here (check schedule)
OPTIONAL MATCH (visitor:Character)-[f:FREQUENTS]->(location)
WHERE visitor.is_active = true
  AND (f.time_of_day = "Any" OR f.time_of_day = $time_of_day)

RETURN collect(DISTINCT npc) + collect(DISTINCT worker) + collect(DISTINCT visitor) as npcs_present
```

#### Find Available Challenges at Location

```cypher
// Get challenges available at this location
MATCH (location:Location {id: $location_id})
MATCH (challenge:Challenge)-[:AVAILABLE_AT]->(location)
WHERE challenge.active = true

// Check prerequisites are met
OPTIONAL MATCH (challenge)-[:REQUIRES_COMPLETION_OF]->(prereq:Challenge)
WITH challenge, collect(prereq) as prerequisites

// Filter to only those with all prerequisites completed (or no prerequisites)
WHERE all(p in prerequisites WHERE p.id IN $completed_challenge_ids)
  OR size(prerequisites) = 0

// Check if enabled by events
OPTIONAL MATCH (event:NarrativeEvent)-[:ENABLES_CHALLENGE]->(challenge)
WHERE event.is_triggered = true

// Check if disabled by events
OPTIONAL MATCH (disabler:NarrativeEvent)-[:DISABLES_CHALLENGE]->(challenge)
WHERE disabler.is_triggered = true

WITH challenge, event, disabler
WHERE (event IS NOT NULL OR NOT exists((challenge)<-[:ENABLES_CHALLENGE]-()))
  AND disabler IS NULL

RETURN challenge
```

#### Find Narrative Tension (Mutual Opponents)

```cypher
// Find pairs of characters who oppose each other
MATCH (a:Character)-[:VIEWS_AS_OPPONENT]->(b:Character)-[:VIEWS_AS_OPPONENT]->(a)
WHERE a.world_id = $world_id AND a.id < b.id  // Avoid duplicates
RETURN a.name as character1, b.name as character2
```

#### Find Potential Allies (Shared Opponents)

```cypher
// Find characters who share an opponent with the given character
MATCH (char:Character {id: $character_id})-[:VIEWS_AS_OPPONENT]->(enemy:Character)
MATCH (potential:Character)-[:VIEWS_AS_OPPONENT]->(enemy)
WHERE potential.id <> char.id
RETURN potential.name as ally, enemy.name as shared_enemy, count(*) as shared_enemies
ORDER BY shared_enemies DESC
```

---

## 5. LLM Context System Design

### Context Categories

| Category | Description | Default Max Tokens | Priority |
|----------|-------------|-------------------|----------|
| `scene` | Location, time, atmosphere, present characters | 500 | 1 (highest) |
| `npc_identity` | Name, description, archetype, behaviors | 400 | 2 |
| `npc_actantial` | Wants, helpers, opponents, senders, receivers | 800 | 3 |
| `npc_locations` | Where NPC lives, works, frequents | 300 | 4 |
| `npc_relationships` | Sentiment toward PC and others in scene | 500 | 5 |
| `narrative_events` | Active events and their trigger conditions | 600 | 6 |
| `challenges` | Active challenges at this location | 400 | 7 |
| `conversation` | Recent dialogue turns | 1500 | 8 |
| `directorial` | DM's guidance, tone, forbidden topics | 400 | 9 |

### Configuration

```rust
// domain/value_objects/context_budget.rs

/// Configuration for LLM context budget
#[derive(Debug, Clone)]
pub struct ContextBudgetConfig {
    pub scene_max_tokens: u32,
    pub npc_identity_max_tokens: u32,
    pub npc_actantial_max_tokens: u32,
    pub npc_locations_max_tokens: u32,
    pub npc_relationships_max_tokens: u32,
    pub narrative_events_max_tokens: u32,
    pub challenges_max_tokens: u32,
    pub conversation_max_tokens: u32,
    pub directorial_max_tokens: u32,
    pub total_max_tokens: u32,
}

impl Default for ContextBudgetConfig {
    fn default() -> Self {
        Self {
            scene_max_tokens: 500,
            npc_identity_max_tokens: 400,
            npc_actantial_max_tokens: 800,
            npc_locations_max_tokens: 300,
            npc_relationships_max_tokens: 500,
            narrative_events_max_tokens: 600,
            challenges_max_tokens: 400,
            conversation_max_tokens: 1500,
            directorial_max_tokens: 400,
            total_max_tokens: 8000,
        }
    }
}
```

### Summarization Flow

When a category exceeds its token budget:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Context Summarization Flow                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Build raw context for category                                          │
│     │                                                                       │
│     ▼                                                                       │
│  2. Count tokens (tiktoken or approximation)                                │
│     │                                                                       │
│     ▼                                                                       │
│  3. If tokens <= max_tokens: USE AS-IS                                      │
│     │                                                                       │
│     ▼                                                                       │
│  4. If tokens > max_tokens: SUMMARIZE                                       │
│     │                                                                       │
│     ├──▶ Build summarization prompt:                                        │
│     │    "We need to collect relevant information for an interaction        │
│     │     between {player_name} and {npc_name}.                             │
│     │     Here is all the {category_name} information.                      │
│     │     Summarize the most relevant details for this interaction          │
│     │     in under {target_tokens} tokens.                                  │
│     │                                                                       │
│     │     Full {category_name}:                                             │
│     │     {raw_content}"                                                    │
│     │                                                                       │
│     ├──▶ Call LLM with summarization prompt                                 │
│     │                                                                       │
│     └──▶ Use summarized content                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### System Prompt Structure

```
You are roleplaying as {npc_name}, a {archetype} in this scene.

=== SCENE ===
Location: {location_name}
Time: {time_context}
Atmosphere: {atmosphere}
Others Present: {character_list}

=== YOUR CHARACTER ===
{npc_description}

As a {archetype}, you typically:
{archetype_behaviors}

=== YOUR DESIRES AND GOALS ===
{actantial_context}

=== YOUR PLACES ===
Home: {home_location}
Work: {work_location}
You frequent: {frequented_locations}

=== YOUR RELATIONSHIPS ===
Your feelings toward the player character ({pc_name}):
{pc_relationship}

Other relationships:
{other_relationships}

=== ACTIVE STORY THREADS ===
{narrative_events_context}

=== CHALLENGES AVAILABLE HERE ===
{challenges_context}

=== DIRECTOR'S GUIDANCE ===
{directorial_notes}

=== CONVERSATION SO FAR ===
{conversation_history}

=== RESPONSE FORMAT ===
Respond with:
1. <reasoning>Your internal thoughts (hidden from player, shown to DM)</reasoning>
2. <dialogue>What you say to the player</dialogue>
3. Optionally: <challenge_suggestion>JSON for skill check</challenge_suggestion>
4. Optionally: <narrative_event_suggestion>JSON for event trigger</narrative_event_suggestion>

You may also call tools to:
- give_item: Give an item to the player
- reveal_info: Reveal plot information
- change_relationship: Update relationship sentiment
- trigger_event: Trigger a narrative event
{additional_tools}
```

---

## 5.1 Region & Navigation System

### Overview

Regions are sub-locations within a Location, representing distinct "screens" in JRPG-style exploration. Players navigate between regions, and scenes are derived from the PC's current region.

```
Location: Rusty Anchor Tavern
├── Region: Entrance (is_spawn_point: true)
│   └── CONNECTED_TO_REGION → Bar Counter
│   └── EXITS_TO_LOCATION → Market District
├── Region: Bar Counter
│   └── CONNECTED_TO_REGION → Entrance, Tables, Back Room
├── Region: Tables
│   └── CONNECTED_TO_REGION → Bar Counter, Entrance
└── Region: Back Room (is_locked: true)
    └── CONNECTED_TO_REGION → Bar Counter
```

### Navigation Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Region Navigation Flow                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Player clicks navigation option                                             │
│       │                                                                     │
│       ▼                                                                     │
│  MoveToRegion { region_id } ──────────────────────▶ Engine validates:      │
│       │                                             - Is connected?         │
│       │                                             - Is locked?            │
│       │                                                                     │
│       │  ExitToLocation { location_id } ─────────▶ Engine validates:       │
│       │       │                                    - Exit exists?           │
│       │       │                                    - arrival_region_id?     │
│       │       │                                                             │
│       ▼       ▼                                                             │
│  Update PC position (location_id + region_id)                               │
│       │                                                                     │
│       ▼                                                                     │
│  Query NPC presence at new region                                           │
│       │                                                                     │
│       ▼                                                                     │
│  SceneChanged { region, npcs_present, navigation_options }                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### WebSocket Messages

```rust
// Client → Server
MoveToRegion { region_id: String }
ExitToLocation { location_id: String, arrival_region_id: Option<String> }

// Server → Client
SceneChanged {
    region: RegionData,
    npcs_present: Vec<NpcPresenceData>,
    navigation_options: NavigationOptions,
}
MovementBlocked { reason: String }
```

---

## 5.2 NPC Presence System

### Overview

NPCs don't simulate travel. Instead, when querying "Is NPC at region?", the system reasons based on NPC relationships and time of day.

### Presence Determination

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       NPC Presence Determination                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Query: "Which NPCs are present in this region?"                            │
│       │                                                                     │
│       ▼                                                                     │
│  1. Check WORKS_AT_REGION edges                                             │
│     - If NPC works here AND shift matches time_of_day → PRESENT             │
│                                                                             │
│  2. Check HOME_REGION edges                                                 │
│     - If NPC lives here AND time is Night → LIKELY PRESENT                  │
│                                                                             │
│  3. Check FREQUENTS_REGION edges                                            │
│     - If NPC frequents here AND time matches → CHECK frequency              │
│       - "always" → PRESENT                                                  │
│       - "often" → 70% chance                                                │
│       - "sometimes" → 40% chance                                            │
│       - "rarely" → 10% chance                                               │
│                                                                             │
│  4. Check AVOIDS_REGION edges                                               │
│     - If NPC avoids here → NOT PRESENT (overrides above)                    │
│                                                                             │
│  5. Optional: LLM reasoning for edge cases                                  │
│     - "Given the context, would [NPC] be here?"                             │
│                                                                             │
│  Result: List of NPCs present with reasoning                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Presence Cache

```rust
/// Cache for presence queries, invalidated when game time advances
pub struct PresenceCache {
    entries: HashMap<(RegionId, CharacterId), PresenceCacheEntry>,
}

pub struct PresenceCacheEntry {
    pub is_present: bool,
    pub reasoning: String,       // For DM review
    pub cached_at_game_time: DateTime<Utc>,
    pub ttl_game_hours: u32,     // Invalidate after N in-game hours
}
```

---

## 5.3 Game Time System

### Overview

Game time is separate from real time. The DM controls when time advances, affecting NPC presence and event triggers.

### GameTime Value Object

```rust
pub struct GameTime {
    pub current: DateTime<Utc>,      // Current in-game date/time
    pub time_scale: f32,             // 0.0 = paused (default)
    pub last_updated: DateTime<Utc>, // Real-world timestamp
}

pub enum TimeOfDay {
    Morning,    // 6:00 - 11:59
    Afternoon,  // 12:00 - 17:59
    Evening,    // 18:00 - 21:59
    Night,      // 22:00 - 5:59
}
```

### DM Controls

- **Advance by hours**: "+1 hour", "+6 hours" quick buttons
- **Advance by days**: "+1 day" for time skips
- **Custom advance**: Set specific in-game time

### Effects of Time Advancement

1. **NPC Presence Changes**: Day workers leave at evening, night NPCs appear
2. **Presence Cache Invalidation**: Queries re-evaluated for new time
3. **Event Triggers**: Time-based narrative events may fire
4. **Challenge Availability**: Some challenges only available at certain times

### WebSocket Messages

```rust
// Client → Server (DM only)
AdvanceGameTime { hours: u32 }

// Server → Client (broadcast)
GameTimeUpdated {
    display: String,      // "Day 3, 7:30 PM"
    time_of_day: String,  // "Evening"
    is_paused: bool,
}
```

---

## 5.4 Observation System

### Overview

PCs track where they last saw NPCs, creating a "fog of war" where player knowledge differs from reality. This enables mystery and investigation gameplay.

### Observation Types

| Type | Source | Example |
|------|--------|---------|
| `Direct` | PC saw NPC in region | "You see Marcus at the bar" |
| `HeardAbout` | DM shared information | "The bartender mentions Marcus was here earlier" |
| `Deduced` | Challenge result | "Investigation success: Marcus frequents the docks at night" |

### Data Model

```rust
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

### Neo4j Edge

```cypher
(pc:PlayerCharacter)-[:OBSERVED_NPC {
    location_id: "uuid",
    region_id: "uuid",
    game_time: datetime(),
    observation_type: "direct",  // direct, heard_about, deduced
    notes: "Saw them arguing with the bartender"
}]->(npc:Character)
```

### Automatic Observation Recording

- When a scene displays NPCs, `Direct` observations are auto-created
- Observations overwrite previous for same NPC (latest wins)
- DM can share `HeardAbout` observations via WebSocket
- Challenge outcomes can create `Deduced` observations

### Player UI: Known NPCs Panel

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Known NPCs                                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  👁️ Marcus the Bartender                                                    │
│     Last seen: Bar Counter • Just now                                       │
│                                                                             │
│  👂 Suspicious Stranger                                                     │
│     Last heard: Docks • 2 days ago (game time)                             │
│     "The bartender mentioned seeing him at the docks"                       │
│                                                                             │
│  🧠 Baron Valdris                                                           │
│     Deduced: Castle • 1 day ago                                            │
│     "Investigation revealed his evening routine"                            │
│                                                                             │
│  Legend: 👁️ Saw directly  👂 Heard about  🧠 Deduced                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5.5 DM Event System

### Overview

The DM can trigger events that bring NPCs to players or narrate environmental changes.

### Event Types

#### Approach Event
NPC approaches a specific PC, triggering an interaction.

```rust
// DM triggers
TriggerApproachEvent {
    npc_id: String,
    target_pc_id: String,
    description: String,  // "A hooded figure slides into the seat next to you..."
}

// Player receives
ApproachEvent {
    npc_id: String,
    npc_name: String,
    npc_sprite: Option<String>,
    description: String,
}
```

**Effects:**
- NPC appears in PC's current region
- Direct observation created
- Dialogue box shows description
- NPC available for interaction

#### Location Event
Narration affecting all PCs in a region.

```rust
// DM triggers
TriggerLocationEvent {
    region_id: String,
    description: String,  // "The lights flicker and go out..."
}

// Players in region receive
LocationEvent {
    region_id: String,
    description: String,
}
```

**Effects:**
- All PCs in region see description
- No NPC involvement
- Pure narration/atmosphere

---

## 6. Current State Assessment

### What's Complete

| Component | Status | Notes |
|-----------|--------|-------|
| Engine hexagonal architecture | ✅ Complete | Clean port/adapter separation |
| Player hexagonal architecture | ✅ Complete | Controlled violation documented |
| Neo4j persistence | ✅ Complete | Graph-first model with edges |
| WebSocket communication | ✅ Complete | Full bidirectional |
| LLM integration (Ollama) | ✅ Complete | Tool calling works |
| DM approval flow | ✅ Complete | Accept/modify/reject/takeover |
| Challenge system | ✅ Complete | With character modifiers, location/region binding |
| Outcome branches | ✅ Complete | Multiple outcomes per challenge |
| Story timeline | ✅ Complete | 18 event types |
| Narrative events | ✅ Complete | Triggers, outcomes, chains, region binding |
| Visual novel UI | ✅ Complete | Backdrop, sprites, dialogue |
| DM director panel | ✅ Complete | Notes, approvals, queue |
| Character archetypes | ✅ Complete | 8 Campbell archetypes |
| Region system | ✅ Complete | Sub-locations, navigation, spawn points |
| NPC presence system | ✅ Complete | Rule-based + LLM presence queries |
| Game time system | ✅ Complete | DM-controlled, affects NPC presence |
| Observation system | ✅ Complete | Direct, HeardAbout, Deduced tracking |
| DM event system | ✅ Complete | Approach events, location events |
| PC selection/creation | ✅ Complete | Per-world PCs, spawn point selection |

### Remaining Work

| Subsystem | Status | Notes |
|-----------|--------|-------|
| **Player UI - Region Navigation** | ⏳ Pending | Engine complete, Player pending |
| **Player UI - Game Time Display** | ⏳ Pending | Engine complete, Player pending |
| **Player UI - Observation Panel** | ⏳ Pending | Engine complete, Player pending |
| **Player UI - PC Selection** | ⏳ Pending | Engine complete, Player pending |
| **E2E Testing** | ⏳ Pending | Integration tests needed |
| **Documentation** | ⏳ Pending | Schema docs, API docs |

---

## 7. MVP Scope Definition

### In Scope

| Feature | Priority | Effort |
|---------|----------|--------|
| **Phase 0: Data Model** | P0 | X-Large |
| Location graph (hierarchy, connections) | P0 | Large |
| Character graph (wants, actantial, inventory) | P0 | Large |
| Character-Location relationships | P0 | Medium |
| Scene graph (location, features) | P0 | Medium |
| Challenge graph (skill, location, prerequisites) | P0 | Medium |
| Narrative Event graph (triggers, effects) | P0 | Large |
| Story Event graph (timeline) | P0 | Medium |
| Migration scripts | P0 | Medium |
| **Phase 1: LLM Context** | P1 | Large |
| Context budget configuration | P1 | Small |
| Graph-based context building | P1 | Large |
| Context summarization | P1 | Medium |
| **Phase 2: Trigger System** | P1 | Medium |
| Engine trigger evaluation | P1 | Medium |
| LLM trigger suggestions | P1 | Small |
| **Phase 3: Session Flow** | P2 | Medium |
| World selection UI | P2 | Medium |
| Session creation | P2 | Small |

### Out of Scope

| Feature | Reason |
|---------|--------|
| Tactical Combat | Deferred to post-MVP |
| GridMap editing | Combat is out of scope |
| WASM/Android builds | Desktop-first |
| Save/Load files | Session state persists in Neo4j |
| User authentication | Local/trusted use |

---

## 8. Implementation Roadmap

### Phase 0: Neo4j Data Model Foundation

**Goal:** Establish a solid graph-first data model for all entities

**Duration:** 7-10 days

---

#### Phase 0.A: Core Schema Design

**Duration:** 1 day

##### Task 0.A.1: Document Complete Graph Schema
- [ ] Finalize all node types with properties
- [ ] Finalize all edge types with properties
- [ ] Create visual schema diagram
- [ ] Document query patterns for each use case
- [ ] **Acceptance:** Schema document reviewed and approved

##### Task 0.A.2: Create Migration Strategy
- [ ] Identify all entities needing migration
- [ ] Define migration order (dependencies)
- [ ] Plan backward compatibility (if needed)
- [ ] **Acceptance:** Migration plan approved

---

#### Phase 0.B: Location System (Graph-First)

**Duration:** 1-2 days

##### Task 0.B.1: Add Location Hierarchy Edges
- [ ] Add `CONTAINS_LOCATION` edge support to repository
- [ ] Update `Location` entity (remove `parent_id` field)
- [ ] Update location creation to use edges
- [ ] Add `get_children`, `get_parent` methods
- [ ] **Acceptance:** Location hierarchy works via edges

##### Task 0.B.2: Add Location Connection Edges
- [ ] Create `LocationConnection` as edge properties (not entity)
- [ ] Add `CONNECTED_TO` edge support to repository
- [ ] Methods: `connect_locations`, `disconnect`, `get_connections`
- [ ] Support bidirectional flag
- [ ] **Acceptance:** Location navigation works via edges

##### Task 0.B.3: Add Location-GridMap Edge
- [ ] Add `HAS_TACTICAL_MAP` edge support
- [ ] Update `Location` entity (remove `grid_map_id` field)
- [ ] **Acceptance:** GridMap association via edge

##### Task 0.B.4: Add BackdropRegion as Nodes
- [ ] Create `BackdropRegion` entity
- [ ] Create `BackdropRegionRepositoryPort`
- [ ] Add `HAS_REGION` edge support
- [ ] Update `Location` entity (remove `backdrop_regions` field)
- [ ] **Acceptance:** Backdrop regions as graph nodes

##### Task 0.B.5: Migration - Locations
- [ ] Script to migrate existing `parent_id` to edges
- [ ] Script to migrate `LocationConnection` records to edges
- [ ] Script to migrate `backdrop_regions` JSON to nodes
- [ ] **Acceptance:** Existing location data migrated

---

#### Phase 0.C: Character System (Graph-First)

**Duration:** 2-3 days

##### Task 0.C.1: Create Want Entity & Repository
- [ ] Create `Want` entity in `domain/entities/want.rs`
- [ ] Create `WantId` in `domain/value_objects/ids.rs`
- [ ] Create `WantRepositoryPort` trait
- [ ] Implement `Neo4jWantRepository`
- [ ] Add `HAS_WANT` edge support
- [ ] Add HTTP routes for Want CRUD
- [ ] **Acceptance:** Wants as separate nodes

##### Task 0.C.2: Create Goal Entity & Repository
- [ ] Create `Goal` entity in `domain/entities/goal.rs`
- [ ] Create `GoalId` in `domain/value_objects/ids.rs`
- [ ] Create `GoalRepositoryPort` trait
- [ ] Implement `Neo4jGoalRepository`
- [ ] Add `TARGETS` edge support (Want → Goal/Character/Item)
- [ ] **Acceptance:** Goals as separate nodes

##### Task 0.C.3: Create Actantial Relationships
- [ ] Create `ActantialRole` enum in value objects
- [ ] Add `VIEWS_AS_HELPER` edge support
- [ ] Add `VIEWS_AS_OPPONENT` edge support
- [ ] Add `VIEWS_AS_SENDER` edge support
- [ ] Add `VIEWS_AS_RECEIVER` edge support
- [ ] Create `ActantialRepositoryPort` trait
- [ ] Implement repository with all actantial queries
- [ ] **Acceptance:** Full actantial model via edges

##### Task 0.C.4: Create Inventory Edges
- [ ] Add `POSSESSES` edge support
- [ ] Update `Character` entity (remove `inventory` field)
- [ ] Methods: `add_item`, `remove_item`, `get_inventory`
- [ ] Support quantity, equipped, acquisition_method
- [ ] **Acceptance:** Inventory via edges

##### Task 0.C.5: Create Archetype History Edges
- [ ] Add `ARCHETYPE_CHANGED` edge support (self-referential)
- [ ] Update `Character` entity (remove `archetype_history` field)
- [ ] Methods: `change_archetype`, `get_archetype_history`
- [ ] **Acceptance:** Archetype history via edges

##### Task 0.C.6: Refactor Character Entity
- [ ] Remove `wants: Vec<Want>` field
- [ ] Remove `inventory: Vec<ItemId>` field
- [ ] Remove `archetype_history: Vec<ArchetypeChange>` field
- [ ] Keep `stats: StatBlock` as JSON (acceptable)
- [ ] Update all Character queries
- [ ] **Acceptance:** Character entity simplified

##### Task 0.C.7: Create Character-Location Edges
- [ ] Add `HOME_LOCATION` edge support
- [ ] Add `WORKS_AT` edge support (with role, schedule)
- [ ] Add `FREQUENTS` edge support (with frequency, time_of_day, reason)
- [ ] Add `AVOIDS` edge support
- [ ] Methods: `set_home`, `set_workplace`, `add_frequents`, `get_locations`
- [ ] **Acceptance:** Rich character-location relationships

##### Task 0.C.8: Update PC Location Edges
- [ ] Ensure `CURRENTLY_AT` edge works for PlayerCharacter
- [ ] Ensure `STARTED_AT` edge works
- [ ] Update `move_to_location` to update edge
- [ ] **Acceptance:** PC location via edges

##### Task 0.C.9: Create Social Relationship Edges
- [ ] Update `RELATES_TO` edge support (already exists, verify)
- [ ] Ensure sentiment, relationship_type on edge
- [ ] Add relationship history tracking (optional)
- [ ] **Acceptance:** Social relationships via edges

##### Task 0.C.10: Migration - Characters
- [ ] Script to extract `wants` JSON to Want nodes + edges
- [ ] Script to extract `inventory` JSON to POSSESSES edges
- [ ] Script to extract `archetype_history` to edges
- [ ] **Acceptance:** Existing character data migrated

---

#### Phase 0.D: Scene & Interaction System

**Duration:** 1 day

##### Task 0.D.1: Create Scene Location Edge
- [ ] Add `AT_LOCATION` edge support
- [ ] Update `Scene` entity (remove `location_id` field)
- [ ] **Acceptance:** Scene location via edge

##### Task 0.D.2: Create Scene Act Edge
- [ ] Add `BELONGS_TO_ACT` edge support
- [ ] Update `Scene` entity (remove `act_id` field)
- [ ] **Acceptance:** Scene-Act relationship via edge

##### Task 0.D.3: Create Scene Features Edges
- [ ] Add `FEATURES_CHARACTER` edge support
- [ ] Update `Scene` entity (remove `featured_characters` field)
- [ ] Support role property on edge
- [ ] **Acceptance:** Scene-Character features via edges

##### Task 0.D.4: Create Interaction Target Edges
- [ ] Add `TARGETS_CHARACTER` edge support
- [ ] Add `TARGETS_ITEM` edge support
- [ ] Add `TARGETS_REGION` edge support
- [ ] Update `InteractionTemplate` entity
- [ ] **Acceptance:** Interaction targets via edges

##### Task 0.D.5: Create Interaction Condition Edges
- [ ] Add `REQUIRES_ITEM` edge support
- [ ] Add `REQUIRES_CHARACTER_PRESENT` edge support
- [ ] Add `REQUIRES_FLAG` edge support (or keep as JSON)
- [ ] Update `InteractionTemplate` entity
- [ ] **Acceptance:** Interaction conditions via edges

##### Task 0.D.6: Migration - Scenes & Interactions
- [ ] Script to migrate scene `location_id` to edges
- [ ] Script to migrate scene `featured_characters` to edges
- [ ] Script to migrate interaction targets/conditions
- [ ] **Acceptance:** Existing data migrated

---

#### Phase 0.E: Challenge System (Graph-First)

**Duration:** 1-2 days

##### Task 0.E.1: Create Challenge-Skill Edge
- [ ] Add `REQUIRES_SKILL` edge support
- [ ] Update `Challenge` entity (remove `skill_id` field)
- [ ] **Acceptance:** Challenge-Skill via edge

##### Task 0.E.2: Create Challenge-Location Edge
- [ ] Add `AVAILABLE_AT` edge support
- [ ] Support `always_available`, `time_restriction` properties
- [ ] Methods: `get_challenges_at_location`
- [ ] **Acceptance:** Challenge location binding

##### Task 0.E.3: Create Challenge Prerequisite Edges
- [ ] Add `REQUIRES_COMPLETION_OF` edge support
- [ ] Update `Challenge` entity (remove `prerequisite_challenges` field)
- [ ] Methods: `get_prerequisites`, `check_prerequisites_met`
- [ ] **Acceptance:** Challenge prerequisites via edges

##### Task 0.E.4: Create Challenge Unlock Edge
- [ ] Add `ON_SUCCESS_UNLOCKS` edge support
- [ ] Link to Location (unlocks CONNECTED_TO edge)
- [ ] Methods: `get_unlocks`
- [ ] **Acceptance:** Challenge can unlock locations

##### Task 0.E.5: Create Challenge-Event Edges
- [ ] Add edge from NarrativeEvent to Challenge (ENABLES, DISABLES)
- [ ] Methods: `get_enabling_events`, `get_disabling_events`
- [ ] **Acceptance:** Events can enable/disable challenges

##### Task 0.E.6: Migration - Challenges
- [ ] Script to migrate `skill_id` to edges
- [ ] Script to migrate `prerequisite_challenges` to edges
- [ ] **Acceptance:** Existing challenge data migrated

---

#### Phase 0.F: Narrative Event System (Graph-First)

**Duration:** 2 days

##### Task 0.F.1: Create Event Location/Scene/Act Edges
- [ ] Add `TIED_TO_LOCATION` edge support
- [ ] Add `TIED_TO_SCENE` edge support
- [ ] Add `BELONGS_TO_ACT` edge support
- [ ] Update `NarrativeEvent` entity (remove ID fields)
- [ ] **Acceptance:** Event associations via edges

##### Task 0.F.2: Create Event Features Edge
- [ ] Add `FEATURES_NPC` edge support
- [ ] Update `NarrativeEvent` entity (remove `featured_npcs` field)
- [ ] **Acceptance:** Event-NPC features via edges

##### Task 0.F.3: Create Event Chain Edges
- [ ] Add `CONTAINS_EVENT` edge on EventChain with position
- [ ] Add `CHAINS_TO` edge between events
- [ ] Update `NarrativeEvent` entity (remove chain fields)
- [ ] Update `EventChain` entity (remove `events` array)
- [ ] **Acceptance:** Event chains via edges

##### Task 0.F.4: Create Event Trigger Edges
- [ ] Add `TRIGGERED_BY_ENTERING` (Location) edge
- [ ] Add `TRIGGERED_BY_TALKING_TO` (Character) edge
- [ ] Add `TRIGGERED_BY_CHALLENGE_COMPLETE` edge
- [ ] Add `TRIGGERED_BY_EVENT_COMPLETE` edge
- [ ] Keep complex triggers (flags, stats) as JSON for now
- [ ] **Acceptance:** Entity-based triggers via edges

##### Task 0.F.5: Create Event Effect Edges
- [ ] Add `EFFECT_GIVES_ITEM` edge
- [ ] Add `EFFECT_MODIFIES_RELATIONSHIP` edge
- [ ] Add `EFFECT_TRIGGERS_SCENE` edge
- [ ] Add `ENABLES_CHALLENGE` edge
- [ ] Add `DISABLES_CHALLENGE` edge
- [ ] Keep complex effects as JSON for now
- [ ] **Acceptance:** Entity-based effects via edges

##### Task 0.F.6: Migration - Narrative Events
- [ ] Script to migrate association IDs to edges
- [ ] Script to migrate featured_npcs to edges
- [ ] Script to migrate event chains to edges
- [ ] Script to extract entity-based triggers to edges
- [ ] Script to extract entity-based effects to edges
- [ ] **Acceptance:** Existing event data migrated

---

#### Phase 0.G: Story Event System (Timeline)

**Duration:** 1 day

##### Task 0.G.1: Create Story Event Location/Scene Edges
- [ ] Add `OCCURRED_AT` (Location) edge support
- [ ] Add `OCCURRED_IN_SCENE` edge support
- [ ] Update `StoryEvent` entity (remove ID fields)
- [ ] **Acceptance:** StoryEvent location via edges

##### Task 0.G.2: Create Story Event Involves Edge
- [ ] Add `INVOLVES` edge support with role property
- [ ] Update `StoryEvent` entity (remove `involved_characters` field)
- [ ] **Acceptance:** StoryEvent-Character via edges

##### Task 0.G.3: Create Story Event Trigger Edge
- [ ] Add `TRIGGERED_BY` (NarrativeEvent) edge support
- [ ] Update `StoryEvent` entity (remove `triggered_by` field)
- [ ] **Acceptance:** StoryEvent-NarrativeEvent via edge

##### Task 0.G.4: Create Story Event Challenge Edge
- [ ] Add `RECORDS_CHALLENGE` edge support
- [ ] Extract challenge_id from event_type variants
- [ ] **Acceptance:** StoryEvent-Challenge via edge

##### Task 0.G.5: Keep Event Type as JSON
- [ ] Keep `StoryEventType` as JSON (complex enum with data)
- [ ] Document why (acceptable per ADR)
- [ ] **Acceptance:** Event type remains JSON

##### Task 0.G.6: Migration - Story Events
- [ ] Script to migrate location/scene IDs to edges
- [ ] Script to migrate involved_characters to edges
- [ ] Script to migrate triggered_by to edge
- [ ] **Acceptance:** Existing story event data migrated

---

#### Phase 0.H: Service Updates & Validation

**Duration:** 1-2 days

##### Task 0.H.1: Update CharacterService
- [ ] Use new Want repository
- [ ] Use new Actantial repository
- [ ] Use new inventory methods
- [ ] Update all character queries
- [ ] **Acceptance:** CharacterService works with graph model

##### Task 0.H.2: Update LocationService
- [ ] Use hierarchy edges
- [ ] Use connection edges
- [ ] Add `get_npcs_at_location` method
- [ ] **Acceptance:** LocationService works with graph model

##### Task 0.H.3: Update SceneService
- [ ] Use location/act edges
- [ ] Use features edges
- [ ] **Acceptance:** SceneService works with graph model

##### Task 0.H.4: Update ChallengeService
- [ ] Use skill edge
- [ ] Use location edge
- [ ] Use prerequisite edges
- [ ] Add `get_available_at_location` method
- [ ] **Acceptance:** ChallengeService works with graph model

##### Task 0.H.5: Update NarrativeEventService
- [ ] Use all new edges
- [ ] Update trigger evaluation to use edges
- [ ] Update effect execution to use edges
- [ ] **Acceptance:** NarrativeEventService works with graph model

##### Task 0.H.6: Update StoryEventService
- [ ] Use all new edges
- [ ] Update event recording to create edges
- [ ] **Acceptance:** StoryEventService works with graph model

##### Task 0.H.7: Run Full Migration
- [ ] Execute all migration scripts in order
- [ ] Validate data integrity
- [ ] Run integration tests
- [ ] **Acceptance:** All data migrated successfully

##### Task 0.H.8: Update HTTP Routes & DTOs
- [ ] Update DTOs for new graph model
- [ ] Update routes to return/accept new structures
- [ ] **Acceptance:** API works with new model

---

### Phase 1: LLM Context Enhancement

**Duration:** 3-4 days

#### Task 1.1: Context Budget Configuration
- [ ] Create `ContextBudgetConfig` value object
- [ ] Add to `AppSettings` entity
- [ ] Add to settings repository
- [ ] Create settings UI in Player
- [ ] **Acceptance:** Can configure token budgets per category

#### Task 1.2: Token Counting
- [ ] Add token counting utility (tiktoken or approximation)
- [ ] Test accuracy against known token counts
- [ ] **Acceptance:** Token counts are reasonably accurate

#### Task 1.3: LLM Context Service (Graph-Based)
- [ ] Create `LLMContextService` in application layer
- [ ] Build context via graph queries (not JSON parsing)
- [ ] Implement all category builders
- [ ] **Acceptance:** Context built from graph traversal

#### Task 1.4: Context Summarization
- [ ] Implement `summarize_category` method
- [ ] Create summarization prompt template
- [ ] Wire summarization when budget exceeded
- [ ] **Acceptance:** Long contexts automatically summarized

#### Task 1.5: Update Prompt Builder
- [ ] Refactor `build_system_prompt_with_notes`
- [ ] Use new LLMContextService
- [ ] Include NPC location relationships
- [ ] Include available challenges
- [ ] **Acceptance:** LLM receives rich, graph-based context

---

### Phase 2: Trigger System

**Duration:** 2-3 days

#### Task 2.1: Engine Trigger Evaluation Service
- [ ] Create `TriggerEvaluationService`
- [ ] Evaluate triggers via graph queries
- [ ] Check entity-based triggers (edges)
- [ ] Check flag/stat triggers (JSON)
- [ ] **Acceptance:** Engine detects satisfied triggers

#### Task 2.2: LLM Trigger Suggestion Parsing
- [ ] Parse `<narrative_event_suggestion>` from LLM
- [ ] Validate against known narrative events
- [ ] Queue for DM approval
- [ ] **Acceptance:** LLM can suggest triggers

#### Task 2.3: Trigger Queue & UI
- [ ] Add trigger suggestions to DM decision queue
- [ ] Show trigger details, source (Engine/LLM)
- [ ] Accept/reject actions
- [ ] **Acceptance:** DM can review trigger suggestions

#### Task 2.4: Trigger Execution
- [ ] On approval, execute narrative event
- [ ] Apply effects (via graph operations)
- [ ] Record StoryEvent
- [ ] Enable chained events
- [ ] **Acceptance:** Approved triggers execute correctly

---

### Phase 3: World Selection & Session Flow

**Duration:** 2 days

#### Task 3.1: World Selection UI
- [ ] List available worlds
- [ ] Show world details
- [ ] Select world to play
- [ ] **Acceptance:** Can browse and select worlds

#### Task 3.2: Session Creation
- [ ] Create session from selected world
- [ ] Initialize session state
- [ ] Connect via WebSocket
- [ ] **Acceptance:** Session starts successfully

#### Task 3.3: PC Selection/Creation
- [ ] List available PCs
- [ ] Assign PC to player
- [ ] Show starting scene
- [ ] **Acceptance:** Player has character and sees starting scene

---

### Phase 4: Integration & Testing

**Duration:** 2-3 days

#### Task 4.1: End-to-End Game Loop Test
- [ ] Test: Player speaks to NPC
- [ ] Test: LLM generates response with full context
- [ ] Test: DM approves response
- [ ] Test: Tool calls execute via graph operations
- [ ] Test: Narrative event triggers
- [ ] Test: Challenge flow
- [ ] **Acceptance:** All integration tests pass

#### Task 4.2: Graph Query Validation
- [ ] Log and review graph queries
- [ ] Verify context includes all expected data
- [ ] Check query performance
- [ ] **Acceptance:** Graph queries are correct and performant

#### Task 4.3: Bug Fixes & Polish
- [ ] Address issues found in testing
- [ ] Performance optimization if needed
- [ ] **Acceptance:** Game loop is stable

#### Task 4.4: Documentation
- [ ] Update ROADMAP.md
- [ ] Update README with data model overview
- [ ] Document graph schema
- [ ] **Acceptance:** Documentation is current

---

## 9. Architecture Compliance Rules

### Hexagonal Architecture Principles

```
┌─────────────────────────────────────────────────────────────────┐
│                      DEPENDENCY RULES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Domain Layer (innermost)                                      │
│   ├── Contains: Entities, Value Objects, Domain Services        │
│   ├── Depends on: NOTHING external                              │
│   └── Rule: Pure Rust, no framework dependencies                │
│                                                                 │
│   Application Layer                                             │
│   ├── Contains: Services, Use Cases, DTOs, Ports (traits)       │
│   ├── Depends on: Domain only                                   │
│   └── Rule: Orchestrates domain logic via ports                 │
│                                                                 │
│   Infrastructure Layer (outermost)                              │
│   ├── Contains: Repositories, External clients, HTTP/WS         │
│   ├── Depends on: Application (implements ports)                │
│   └── Rule: Adapts external systems to ports                    │
│                                                                 │
│   Presentation Layer (Player only)                              │
│   ├── Contains: UI Components, Views, State                     │
│   ├── Depends on: Application services                          │
│   └── Rule: Calls services, never repositories directly         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Import Rules

**NEVER ALLOWED:**
```rust
// Domain importing from application/infrastructure
use crate::application::*;  // FORBIDDEN in domain/
use crate::infrastructure::*;  // FORBIDDEN in domain/

// Application importing from infrastructure
use crate::infrastructure::*;  // FORBIDDEN in application/
```

**ALWAYS REQUIRED:**
```rust
// Infrastructure implements application ports
use crate::application::ports::outbound::SomeRepositoryPort;

impl SomeRepositoryPort for Neo4jSomeRepository {
    // ...
}
```

### Violation Approval Process

1. **Propose in Plan:**
   - Document the violation in this MVP.md
   - Explain why it's necessary
   - Describe the trade-off
   - Propose mitigation

2. **Await Approval:**
   - User must explicitly approve
   - No implementation until approved

3. **Mark in Code:**
   ```rust
   // ARCHITECTURE VIOLATION: [APPROVED 2025-12-17]
   // Reason: <explanation>
   // Mitigation: <how we limit the damage>
   // Approved by: <user>
   use crate::infrastructure::SomeType;  // Normally forbidden
   ```

### Currently Accepted Violations

| Location | Description | Justification | Status |
|----------|-------------|---------------|--------|
| `Player/src/presentation/services.rs` | Type aliases expose `ApiAdapter` | Composition layer pattern - components use service methods only | ACCEPTED |
| `Engine/src/domain/entities/*.rs` | Serde derives on some domain types | Required for JSON serialization to Neo4j | ACCEPTED with ADR comments |

### Proposed Violations (Awaiting Approval)

*None currently proposed*

---

## 10. Acceptance Criteria

### MVP Complete When:

1. **Data Model is Pure Graph**
   - [ ] No JSON blobs for entity references
   - [ ] All relationships are Neo4j edges
   - [ ] Acceptable JSON only for non-relational data

2. **Location System Works**
   - [ ] Hierarchy via CONTAINS edges
   - [ ] Navigation via CONNECTED_TO edges
   - [ ] NPCs have location relationships (HOME, WORKS_AT, FREQUENTS)

3. **Character System Works**
   - [ ] Wants as separate nodes
   - [ ] Actantial model via VIEWS_AS_* edges
   - [ ] Inventory via POSSESSES edges
   - [ ] Archetype history via edges

4. **Challenge System Works**
   - [ ] Challenges bound to locations
   - [ ] Challenges can unlock locations
   - [ ] Challenges enabled/disabled by events

5. **LLM Receives Rich Context**
   - [ ] Context built via graph queries
   - [ ] All categories included
   - [ ] Summarization works when needed

6. **Triggers Work Both Ways**
   - [ ] Engine evaluates triggers via graph
   - [ ] LLM can suggest triggers
   - [ ] All triggers go to DM approval

7. **Game Loop is Playable**
   - [ ] Can select world and start session
   - [ ] Can speak to NPCs
   - [ ] LLM generates contextual responses
   - [ ] DM can approve/modify/reject
   - [ ] Tool calls execute via graph operations
   - [ ] Timeline records events

8. **Architecture is Clean**
   - [ ] No unapproved violations
   - [ ] All ports properly implemented
   - [ ] Services use dependency injection

---

## 11. Progress Tracking

### Phase 0.A: Core Schema Design

| Task | Status | Notes |
|------|--------|-------|
| 0.A.1 Document Schema | [x] | neo4j-graph-schema.md complete |
| 0.A.2 Migration Strategy | [x] | migration-strategy.md complete |

### Phase 0.B: Location System

| Task | Status | Notes |
|------|--------|-------|
| 0.B.1 Hierarchy Edges | [x] | CONTAINS_LOCATION edge implemented |
| 0.B.2 Connection Edges | [x] | CONNECTED_TO edge with properties |
| 0.B.3 GridMap Edge | [x] | HAS_TACTICAL_MAP edge implemented |
| 0.B.4 BackdropRegion Nodes | [x] | HAS_REGION edge implemented |
| 0.B.5 Migration | [x] | Repository and service updated |

### Phase 0.C: Character System

| Task | Status | Notes |
|------|--------|-------|
| 0.C.1 Want Entity | [x] | Want as node with HAS_WANT edge |
| 0.C.2 Goal Entity | [x] | Goal node for abstract targets |
| 0.C.3 Actantial Edges | [x] | VIEWS_AS_* edges implemented |
| 0.C.4 Inventory Edges | [x] | POSSESSES edge with properties |
| 0.C.5 Archetype History | [x] | Tracking archetype changes |
| 0.C.6 Refactor Character | [x] | Removed embedded wants/inventory |
| 0.C.7 Character-Location | [x] | HOME/WORKS_AT/FREQUENTS/AVOIDS edges |
| 0.C.8 PC Location | [x] | PlayerCharacter location tracking |
| 0.C.9 Social Relationships | [x] | Already used edges - no changes |
| 0.C.10 Migration | [x] | Repository methods complete |

### Phase 0.D: Scene & Interaction

| Task | Status | Notes |
|------|--------|-------|
| 0.D.1 Scene Location | [x] | AT_LOCATION edge implemented |
| 0.D.2 Scene Act | [x] | BELONGS_TO_ACT edge implemented |
| 0.D.3 Scene Features | [x] | FEATURES_CHARACTER edge with role/cue |
| 0.D.4 Interaction Targets | [x] | TARGETS_* edges implemented |
| 0.D.5 Interaction Conditions | [x] | REQUIRES_* edges implemented |
| 0.D.6 Migration | [x] | Repository and service updated |

### Phase 0.E: Challenge System

| Task | Status | Notes |
|------|--------|-------|
| 0.E.1 Challenge-Skill | [x] | REQUIRES_SKILL edge implemented |
| 0.E.2 Challenge-Location | [x] | AVAILABLE_AT edge with properties |
| 0.E.3 Prerequisites | [x] | REQUIRES_COMPLETION_OF edge |
| 0.E.4 Unlock Edge | [x] | ON_SUCCESS_UNLOCKS edge |
| 0.E.5 Event Edges | [x] | TIED_TO_SCENE edge |
| 0.E.6 Migration | [x] | Service uses all edge methods |

### Phase 0.F: Narrative Event

| Task | Status | Notes |
|------|--------|-------|
| 0.F.1 Association Edges | [x] | TIED_TO_SCENE/LOCATION, BELONGS_TO_ACT |
| 0.F.2 Features Edge | [x] | FEATURES_NPC with role property |
| 0.F.3 Chain Edges | [x] | CONTAINS_EVENT from EventChain |
| 0.F.4 Trigger Edges | [x] | JSON kept (complex nested data) |
| 0.F.5 Effect Edges | [x] | JSON kept (complex nested data) |
| 0.F.6 Migration | [x] | Entity, repository, DTO all updated |

### Phase 0.G: Story Event

| Task | Status | Notes |
|------|--------|-------|
| 0.G.1 Location/Scene Edges | [x] | OCCURRED_AT, OCCURRED_IN_SCENE edges implemented |
| 0.G.2 Involves Edge | [x] | INVOLVES edge with role property (actor, target, speaker, witness) |
| 0.G.3 Trigger Edge | [x] | TRIGGERED_BY_NARRATIVE, OCCURRED_IN_SESSION edges implemented |
| 0.G.4 Challenge Edge | [x] | RECORDS_CHALLENGE edge implemented |
| 0.G.5 Keep Event Type JSON | [x] | StoryEventType remains as JSON (complex discriminated union) |
| 0.G.6 Migration | [x] | StoryEvent entity, repository, service, DTO all updated |

### Phase 0.H: Service Updates

| Task | Status | Notes |
|------|--------|-------|
| 0.H.1 CharacterService | [x] | Uses edge methods for wants; location/inventory/actantial deferred |
| 0.H.2 LocationService | [x] | ALREADY COMPLETE - all edge methods used |
| 0.H.3 SceneService | [x] | Updated to use AT_LOCATION and FEATURES_CHARACTER edges |
| 0.H.4 ChallengeService | [x] | ALREADY COMPLETE - all edge methods used |
| 0.H.5 NarrativeEventService | [x] | Added 18 edge methods (scene, location, act, NPCs, chains) |
| 0.H.6 StoryEventService | [x] | Done in 0.G - uses all edge methods |
| 0.H.7 Run Migration | [x] | No data migration needed - new code uses edges |
| 0.H.8 Update Routes/DTOs | [x] | DTOs have edge constructors; routes use minimal for lists |

### Phase 1: LLM Context ✅ COMPLETE

| Task | Status | Notes |
|------|--------|-------|
| 1.1 Budget Config | [x] | `ContextBudgetConfig` in value_objects/context_budget.rs |
| 1.2 Token Counting | [x] | `TokenCounter` with CharacterApprox, WordApprox, Hybrid methods |
| 1.3 Context Service | [x] | `LLMContextService` builds context from graph traversal |
| 1.4 Summarization | [x] | `SummarizationPlanner`, `SummarizationRequest`, category-specific prompts |
| 1.5 Update Prompt Builder | [x] | `build_system_prompt_from_assembled()`, `merge_conversation_history()` |

### Phase 2: Trigger System ✅ COMPLETE

| Task | Status | Notes |
|------|--------|-------|
| 2.1 Engine Evaluation | [x] | `TriggerEvaluationService` evaluates triggers against `GameStateSnapshot` |
| 2.2 LLM Parsing | [x] | `<narrative_event_suggestion>` already parsed in LLMService |
| 2.3 Queue & UI | [x] | `NarrativeEventSuggestionInfo` flows through DM approval queue |
| 2.4 Execution | [x] | `EventEffectExecutor` executes all `EventEffect` types |

### Phase 3: Session Flow ✅ ENGINE COMPLETE (via Phase 23)

| Task | Status | Notes |
|------|--------|-------|
| 3.1 World Selection | [x] | Session routes with world_id, world snapshot export |
| 3.2 Session Creation | [x] | `create_or_get_dm_session` in session_routes.rs |
| 3.3 PC Selection | [x] | Phase 23B.6 - available-pcs, select-pc, import-pc routes |

**Phase 23 Implementation** (plans/23-pc-selection-regions-scenes.md):
- Region entity with spawn points, connections, exits
- PC position tracking (location + region)
- Navigation system (WebSocket: MoveToRegion, ExitToLocation, SceneChanged)
- Game time system (DM-controlled, affects NPC presence)
- Observation system (Direct, HeardAbout, Deduced)
- DM event system (ApproachEvent, LocationEvent)

### Phase 4: Integration

| Task | Status | Notes |
|------|--------|-------|
| 4.1 E2E Tests | [ ] | |
| 4.2 Query Validation | [ ] | |
| 4.3 Bug Fixes | [x] | See Bug Fix Session below |
| 4.4 Documentation | [ ] | |

### Bug Fix Session (2025-12-18)

Critical bugs fixed and MVP TODOs addressed:

| Task | Status | Description |
|------|--------|-------------|
| A.1 Preset Parsing Bug | [x] | Player failed to parse preset: Added `RuleSystemPresetDetails` struct |
| A.2 Generation Queue World Scoping | [x] | Queue showed all worlds; added world_id to GenerationBatch and scoped queries |
| B.1 get_by_scene() | [x] | Implemented character lookup by scene (was returning empty vec) |
| B.2 Fetch Wants for LLM | [x] | `build_prompt_from_action` now fetches wants via `character_repo.get_wants()` |
| B.3 Challenge Description/Skill | [x] | Added challenge_description and skill_name to approval DTOs |
| C.1 Character Name on Join | [x] | `SessionParticipantInfo` now includes character_name from session |
| C.2 target_pc_id Through Flow | [x] | Added pc_id to PlayerActionItem, LLMRequestItem, ChallengeSuggestionInfo |
| C.3 Re-enqueue Discarded Challenges | [x] | Documented approach; manual DM trigger for now |

**Files Modified (Engine):**
- `src/infrastructure/websocket_helpers.rs` - Added character_repo param, fetch wants
- `src/main.rs` - Pass character_repo to build_prompt_from_action worker
- `src/application/dto/queue_items.rs` - Added pc_id, challenge_description, skill_name fields
- `src/application/dto/challenge.rs` - Added challenge_description, skill_name to PendingChallengeResolutionDto
- `src/application/services/challenge_resolution_service.rs` - Pass skill_id, look up skill name
- `src/application/services/challenge_outcome_approval_service.rs` - Use new DTO fields
- `src/application/services/player_action_queue_service.rs` - Pass pc_id through queue
- `src/application/services/llm_queue_service.rs` - Use pc_id in ChallengeSuggestionInfo
- `src/application/services/dm_approval_queue_service.rs` - Documented re-enqueue approach
- `src/application/ports/outbound/async_session_port.rs` - Added character_name to SessionParticipantInfo
- `src/infrastructure/session_adapter.rs` - Look up character_name from session
- `src/infrastructure/session/game_session.rs` - Added get_character_name_for_user()
- `src/infrastructure/session/mod.rs` - Improved TODO comment for wants
- `src/infrastructure/http/suggestion_routes.rs` - Added pc_id to LLMRequestItem
- `src/infrastructure/websocket.rs` - Look up pc_id for player actions

**Files Modified (Player):**
- `src/application/dto/world_snapshot.rs` - Added RuleSystemPresetDetails struct
- `src/application/dto/mod.rs` - Export RuleSystemPresetDetails
- `src/presentation/views/world_select.rs` - Deserialize preset correctly

---

## Appendix A: File Locations

### New Files to Create (Phase 0)

```
Engine/
├── src/
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── want.rs              # NEW
│   │   │   ├── goal.rs              # NEW
│   │   │   └── backdrop_region.rs   # NEW
│   │   └── value_objects/
│   │       ├── actantial.rs         # NEW
│   │       └── context_budget.rs    # NEW
│   ├── application/
│   │   ├── ports/
│   │   │   └── outbound/
│   │   │       ├── want_repository_port.rs      # NEW
│   │   │       ├── goal_repository_port.rs      # NEW
│   │   │       └── actantial_repository_port.rs # NEW
│   │   └── services/
│   │       ├── llm_context_service.rs       # NEW
│   │       └── trigger_evaluation_service.rs # NEW
│   └── infrastructure/
│       └── persistence/
│           ├── want_repository.rs       # NEW
│           ├── goal_repository.rs       # NEW
│           ├── actantial_repository.rs  # NEW
│           └── backdrop_region_repository.rs # NEW
```

### Files to Modify (Phase 0)

```
Engine/
├── src/
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── character.rs         # Remove wants, inventory, archetype_history
│   │   │   ├── location.rs          # Remove parent_id, grid_map_id, backdrop_regions
│   │   │   ├── scene.rs             # Remove location_id, act_id, featured_characters
│   │   │   ├── challenge.rs         # Remove skill_id, prerequisite_challenges
│   │   │   ├── narrative_event.rs   # Remove many ID fields
│   │   │   ├── story_event.rs       # Remove location_id, involved_characters, etc.
│   │   │   ├── event_chain.rs       # Remove events array
│   │   │   └── mod.rs               # Export new entities
│   │   └── value_objects/
│   │       ├── ids.rs               # Add WantId, GoalId, BackdropRegionId
│   │       └── mod.rs               # Export new VOs
│   ├── application/
│   │   ├── ports/
│   │   │   └── outbound/
│   │   │       └── mod.rs           # Export new ports
│   │   └── services/
│   │       ├── character_service.rs # Major updates
│   │       ├── location_service.rs  # Major updates
│   │       ├── scene_service.rs     # Updates
│   │       ├── challenge_service.rs # Updates
│   │       ├── narrative_event_service.rs # Major updates
│   │       ├── story_event_service.rs # Updates
│   │       └── mod.rs               # Export new services
│   └── infrastructure/
│       ├── persistence/
│       │   ├── character_repository.rs  # Major refactor
│       │   ├── location_repository.rs   # Major refactor
│       │   ├── scene_repository.rs      # Updates
│       │   ├── challenge_repository.rs  # Updates
│       │   ├── narrative_event_repository.rs # Major refactor
│       │   ├── story_event_repository.rs # Updates
│       │   └── mod.rs                   # Export new repos
│       └── state/
│           └── mod.rs               # Wire new services
```

---

## Appendix B: Related Documents

| Document | Purpose |
|----------|---------|
| `plans/00-master-plan.md` | Original project specification |
| `plans/CONSOLIDATED_PLAN_FORWARD.md` | Previous progress tracking |
| `plans/17-story-arc.md` | Narrative event system design |
| `plans/22-core-game-loop.md` | Game loop design |
| `Engine/specs/first-draft.md` | Campbell/Greimas theory background |
| `Engine/README.md` | Engine architecture overview |

---

## Revision History

| Date | Change |
|------|--------|
| 2025-12-17 | Initial MVP plan created |
| 2025-12-17 | Expanded to comprehensive Neo4j data model foundation |
