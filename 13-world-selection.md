# Phase 13: World Selection & Session Management

## Overview

Worlds act as the "server" concept in WrldBldr. Before entering gameplay, users must select or create a World. This phase introduces a World Selection screen between Role Selection and the actual game views.

**Key Concept**: A World = A Game Session. Players join the same World that the DM is running.

---

## Current Flow

```
MainMenu (Server URL) → RoleSelect (DM/Player/Spectator) → GameView
```

## Proposed Flow

```
MainMenu (Server URL) → RoleSelect → WorldSelect → GameView
                                          ↓
                              DM: Create/Continue World
                              Player: Join Existing World
                              Spectator: View Existing World
```

---

## User Stories

### Epic: World Selection for DM

#### US-13.1: View Available Worlds (DM)
**As a** Dungeon Master
**I want to** see a list of all worlds I have created
**So that** I can choose which campaign to continue running

**Acceptance Criteria:**
- Display list of existing worlds with name, description preview, and last modified date
- Show world statistics (character count, location count, scene count)
- Empty state message when no worlds exist: "No worlds yet. Create your first campaign!"
- Worlds sorted by last modified date (most recent first)
- Loading state while fetching worlds

**API:** `GET /api/worlds`

---

#### US-13.2: Create New World (DM)
**As a** Dungeon Master
**I want to** create a new world/campaign
**So that** I can start building a fresh setting for my players

**Acceptance Criteria:**
- "Create New World" button prominently displayed
- Modal/form with fields:
  - World name (required)
  - Description (optional)
  - Rule system selection (D20, D100, Fate, Custom)
- Validation: World name must be unique and non-empty
- After creation, automatically enter the new world
- Success feedback: "World created! Entering [World Name]..."

**API:** `POST /api/worlds`

---

#### US-13.3: Continue Existing World (DM)
**As a** Dungeon Master
**I want to** select an existing world and enter it
**So that** I can continue running my campaign

**Acceptance Criteria:**
- Click on world card to select
- "Continue" button to enter selected world
- Or double-click to enter directly
- Load world data and transition to DM View
- Show loading indicator during world load

**API:** `GET /api/worlds/{id}` → `GET /api/worlds/{id}/export`

---

#### US-13.4: Delete World (DM)
**As a** Dungeon Master
**I want to** delete a world I no longer need
**So that** I can clean up old or test campaigns

**Acceptance Criteria:**
- Delete button on each world card (trash icon)
- Confirmation modal: "Delete [World Name]? This cannot be undone. All characters, locations, and scenes will be permanently deleted."
- Type world name to confirm (prevent accidental deletion)
- Success feedback and remove from list
- Cannot delete while players are connected (show error)

**API:** `DELETE /api/worlds/{id}`

---

#### US-13.5: Edit World Settings (DM)
**As a** Dungeon Master
**I want to** edit a world's name, description, and rule system
**So that** I can update campaign details without recreating it

**Acceptance Criteria:**
- Edit button on each world card (pencil icon)
- Modal with pre-filled current values
- Save changes without entering the world
- Cancel returns to list unchanged

**API:** `PUT /api/worlds/{id}`

---

### Epic: World Selection for Player

#### US-13.6: View Available Worlds (Player)
**As a** Player
**I want to** see a list of worlds I can join
**So that** I can find the campaign my DM is running

**Acceptance Criteria:**
- Display list of all worlds (or worlds marked as "open")
- Show world name, description, and DM name (if available)
- Show player count: "2 players connected"
- Show status indicator: 🟢 Active (DM online), 🟡 Idle, ⚫ Offline
- Empty state: "No worlds available. Ask your DM to create one!"

**API:** `GET /api/worlds` (filtered for player visibility)

---

#### US-13.7: Join World (Player)
**As a** Player
**I want to** select a world and join it
**So that** I can participate in the campaign

**Acceptance Criteria:**
- Click on world card to select
- "Join" button to enter selected world
- Load world data and transition to PC View
- If DM is not online, show warning: "The DM is not currently online. You can explore but gameplay will be limited."
- Show loading indicator during world load

**API:** `GET /api/worlds/{id}/export` (player-filtered snapshot)

---

#### US-13.8: View World Details (Player)
**As a** Player
**I want to** see more details about a world before joining
**So that** I can verify it's the right campaign

**Acceptance Criteria:**
- Expand/details button on world card
- Show: Full description, rule system, creation date, character count
- Show connected players list (if any)
- "Join" button in details view

---

### Epic: World Selection for Spectator

#### US-13.9: View Available Worlds (Spectator)
**As a** Spectator
**I want to** see worlds that allow spectating
**So that** I can watch a game in progress

**Acceptance Criteria:**
- Display list of worlds (same as Player view)
- Show spectator count: "3 spectators watching"
- Indicate if world allows spectators (DM setting)

**API:** `GET /api/worlds`

---

#### US-13.10: Watch World (Spectator)
**As a** Spectator
**I want to** select a world and watch it
**So that** I can observe the campaign without participating

**Acceptance Criteria:**
- Click on world card to select
- "Watch" button to enter as spectator
- Load world data and transition to Spectator View
- If no active session, show: "No active scene. Waiting for the DM to start..."

---

### Epic: Session Management

#### US-13.11: Track Connected Users
**As a** DM
**I want to** see who is connected to my world
**So that** I know when my players are ready

**Acceptance Criteria:**
- Player list panel in DM View showing connected users
- Show: Username, role (Player/Spectator), connection status
- Real-time updates when players join/leave
- Notification when player joins: "[Player] has joined the session"

**WebSocket:** `PlayerJoined`, `PlayerLeft` events

---

#### US-13.12: Disconnect from World
**As a** User (any role)
**I want to** leave the current world and return to world selection
**So that** I can switch to a different campaign

**Acceptance Criteria:**
- "Leave World" or back button in game views
- Confirmation if DM: "Leave world? Players will remain connected but gameplay will pause."
- Return to World Selection screen
- WebSocket disconnection handled gracefully

---

#### US-13.13: World Access Control (Future)
**As a** DM
**I want to** control who can join my world
**So that** I can run private campaigns

**Acceptance Criteria:**
- World visibility setting: Public / Private / Invite-Only
- For private: Generate invite code
- For invite-only: Approve player requests
- Block/kick players from world

**Note:** Deferred to future phase - requires authentication system

---

## UI Mockups

### DM World Selection Screen

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  [← Back]                    Select World                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  [+]  CREATE NEW WORLD                                               │   │
│  │       Start a fresh campaign                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  YOUR WORLDS                                                                │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  🏰 The Dragon's Bane                                    [✏️] [🗑️]  │   │
│  │  A dark fantasy campaign in the realm of Valdris                     │   │
│  │                                                                       │   │
│  │  📊 12 Characters • 8 Locations • 5 Scenes                           │   │
│  │  🕐 Last played: 2 hours ago                                         │   │
│  │                                                                       │   │
│  │                                              [Continue →]            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ⚔️ Curse of Strahd                                      [✏️] [🗑️]  │   │
│  │  Gothic horror in Barovia                                            │   │
│  │                                                                       │   │
│  │  📊 8 Characters • 15 Locations • 12 Scenes                          │   │
│  │  🕐 Last played: 3 days ago                                          │   │
│  │                                                                       │   │
│  │                                              [Continue →]            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Player World Selection Screen

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  [← Back]                    Join a World                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  AVAILABLE WORLDS                                                           │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  🟢 The Dragon's Bane                                                │   │
│  │  A dark fantasy campaign in the realm of Valdris                     │   │
│  │                                                                       │   │
│  │  👥 2 players connected • DM online                                  │   │
│  │  📜 D20 System                                                       │   │
│  │                                                                       │   │
│  │                                                   [Join →]           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ⚫ Curse of Strahd                                                  │   │
│  │  Gothic horror in Barovia                                            │   │
│  │                                                                       │   │
│  │  👥 0 players connected • DM offline                                 │   │
│  │  📜 D20 System                                                       │   │
│  │                                                                       │   │
│  │                                                   [Join →]           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────   │
│  💡 Don't see your world? Make sure your DM has created it and you're      │
│     connected to the same server.                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Create World Modal

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Create New World                           [×]   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  World Name *                                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ The Dragon's Bane                                                     │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  Description                                                                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ A dark fantasy campaign where heroes must stop an ancient dragon     │ │
│  │ from awakening and destroying the realm of Valdris.                  │ │
│  │                                                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  Rule System                                                                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ D20 (D&D, Pathfinder)                                             ▼  │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  [Cancel]                                    [Create World →]        │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Model Considerations

### World Entity Additions

```rust
pub struct World {
    pub id: WorldId,
    pub name: String,
    pub description: Option<String>,
    pub rule_system: RuleSystemConfig,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,

    // New fields for session management
    pub visibility: WorldVisibility,      // Public, Private, InviteOnly
    pub allow_spectators: bool,
    pub max_players: Option<u32>,
}

pub enum WorldVisibility {
    Public,      // Anyone can see and join
    Private,     // Only visible to owner (future: with auth)
    InviteOnly,  // Requires invite code (future)
}
```

### World Statistics (Computed)

```rust
pub struct WorldStats {
    pub character_count: u32,
    pub location_count: u32,
    pub scene_count: u32,
    pub act_count: u32,
    pub last_modified: DateTime<Utc>,
}
```

### Session State

```rust
pub struct WorldSession {
    pub world_id: WorldId,
    pub dm_connected: bool,
    pub players: Vec<ConnectedPlayer>,
    pub spectators: Vec<ConnectedSpectator>,
    pub current_scene: Option<SceneId>,
}

pub struct ConnectedPlayer {
    pub user_id: String,
    pub username: String,
    pub connected_at: DateTime<Utc>,
    pub character_id: Option<CharacterId>,  // Their PC
}
```

---

## API Additions

### REST Endpoints

```
# World listing with stats
GET  /api/worlds                    # List all worlds with stats
GET  /api/worlds/{id}               # Get world details
GET  /api/worlds/{id}/stats         # Get world statistics
POST /api/worlds                    # Create new world
PUT  /api/worlds/{id}               # Update world settings
DELETE /api/worlds/{id}             # Delete world

# Session status
GET  /api/worlds/{id}/session       # Get current session state (who's connected)
```

### WebSocket Messages

```rust
// Client → Server
enum WorldMessage {
    JoinWorld { world_id: WorldId, role: ParticipantRole },
    LeaveWorld,
}

// Server → Client
enum SessionEvent {
    PlayerJoined { user_id: String, username: String, role: ParticipantRole },
    PlayerLeft { user_id: String },
    DMConnected,
    DMDisconnected,
    SessionStarted { scene_id: SceneId },
}
```

---

## Implementation Phases

### Phase 13A: World List UI (Player)
- [ ] Create `WorldSelectView` component
- [ ] DM variant: List worlds with create/edit/delete
- [ ] Player variant: List worlds with join
- [ ] Spectator variant: List worlds with watch
- [ ] Loading and empty states

### Phase 13B: World CRUD (Engine)
- [ ] Add `WorldStats` endpoint
- [ ] Add world listing with stats
- [ ] Implement delete with cascade (characters, locations, etc.)
- [ ] Add validation for world operations

### Phase 13C: Create World Flow
- [ ] Create World modal component
- [ ] Form validation
- [ ] API integration
- [ ] Auto-enter after creation

### Phase 13D: Session Awareness
- [ ] Track connected users per world
- [ ] WebSocket events for join/leave
- [ ] Display connection status in world cards
- [ ] DM online indicator for players

### Phase 13E: Navigation Integration
- [ ] Update app routing to include WorldSelect
- [ ] Back navigation from game views
- [ ] World context in game state
- [ ] Leave world functionality

---

## Dependencies

- **Requires**: Role selection (existing)
- **Requires**: World CRUD API (existing, needs stats endpoint)
- **Optional**: Authentication system (for access control)

---

## Out of Scope (Future)

- User authentication and accounts
- World access control (private/invite-only)
- World search and filtering
- World templates/presets
- World import/export
- Character persistence across sessions

---

## Questions for Clarification

1. **World Visibility**: Should all worlds be visible to all players by default, or should DMs control visibility?

2. **Player Names**: Without authentication, how do we identify players? Just a text input username on join?

3. **Concurrent DMs**: Can multiple users connect as DM to the same world, or is it locked to one DM?

4. **World Persistence**: When DM disconnects, does the world session end or can players remain?

5. **Character Selection**: Should players select/create their character as part of joining a world?
