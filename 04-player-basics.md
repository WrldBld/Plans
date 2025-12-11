# Phase 4: Player Basics

## Objective
Implement the core Player functionality: WebSocket connection to Engine, world data loading, enhanced visual novel renderer, and PC action input system.

## Status: ✅ COMPLETED (2025-12-11)

## Tasks

### 4.1 WebSocket Client
**Status:** ✅ COMPLETE
**Files:**
- `Player/src/infrastructure/websocket/mod.rs`
- `Player/src/infrastructure/websocket/client.rs`
- `Player/src/infrastructure/websocket/messages.rs`

**Subtasks:**
- [x] 4.1.1 Create WebSocket client (EngineClient) using tokio-tungstenite (desktop) / gloo-net (WASM)
- [x] 4.1.2 Implement connection handling (connect, reconnect, disconnect)
- [x] 4.1.3 Define message serialization/deserialization (serde_json)
- [x] 4.1.4 Handle server messages and dispatch to state
- [ ] 4.1.5 Implement heartbeat/ping mechanism

### 4.2 World Loading
**Status:** ✅ COMPLETE
**Files:**
- `Player/src/infrastructure/asset_loader/mod.rs`
- `Player/src/infrastructure/asset_loader/world_snapshot.rs`

**Subtasks:**
- [x] 4.2.1 Define WorldSnapshot matching Engine export format
- [x] 4.2.2 Implement JSON deserialization
- [x] 4.2.3 Create world state from snapshot
- [x] 4.2.4 Load world via WebSocket or direct JSON file
- [x] 4.2.5 Handle loading errors gracefully

### 4.3 Visual Novel Renderer Enhancement
**Status:** ✅ COMPLETE
**Files:**
- `Player/src/presentation/components/visual_novel/mod.rs`
- `Player/src/presentation/components/visual_novel/backdrop.rs`
- `Player/src/presentation/components/visual_novel/character_sprite.rs`
- `Player/src/presentation/components/visual_novel/dialogue_box.rs`
- `Player/src/presentation/components/visual_novel/choice_menu.rs`

**Subtasks:**
- [x] 4.3.1 Backdrop component with image loading
- [x] 4.3.2 Character sprite positioning (left/center/right)
- [x] 4.3.3 Dialogue box with typewriter effect (with punctuation-aware delays)
- [x] 4.3.4 Speaker name display
- [x] 4.3.5 Choice menu with custom input option
- [ ] 4.3.6 Transition animations between scenes (deferred to polish phase)

### 4.4 PC Action Input
**Status:** ✅ COMPLETE
**Files:**
- `Player/src/presentation/components/action_panel.rs`
- `Player/src/domain/entities/player_action.rs`
- `Player/src/application/services/action_service.rs`

**Subtasks:**
- [x] 4.4.1 Define PlayerAction types (Talk, Examine, UseItem, Travel, Custom, DialogueChoice)
- [x] 4.4.2 Create action panel UI component (with disabled state for LLM processing)
- [x] 4.4.3 Interaction button list from scene
- [x] 4.4.4 Custom text input for dialogue
- [x] 4.4.5 Send actions to Engine via WebSocket

### 4.5 State Management
**Status:** ✅ COMPLETE
**Files:**
- `Player/src/presentation/state/mod.rs`
- `Player/src/presentation/state/game_state.rs`
- `Player/src/presentation/state/session_state.rs`
- `Player/src/presentation/state/dialogue_state.rs`
- `Player/src/presentation/state/generation_state.rs`

**Subtasks:**
- [x] 4.5.1 Create GameState with Dioxus signals (world, current_scene, characters, interactions)
- [x] 4.5.2 SessionState for connection status (ConnectionStatus enum, user role, error handling)
- [x] 4.5.3 DialogueState for typewriter animation (is_llm_processing signal added)
- [x] 4.5.4 GenerationState for asset generation tracking

## WebSocket Protocol

### Client → Engine Messages
```rust
enum ClientMessage {
    JoinSession { user_id: String, role: String },
    PlayerAction { action_type: String, target: Option<String>, dialogue: Option<String> },
    RequestSceneChange { scene_id: String },
    Heartbeat,
}
```

### Engine → Client Messages
```rust
enum ServerMessage {
    SessionJoined { session_id: String, world_snapshot: WorldSnapshot },
    SceneUpdate { scene: SceneData, characters: Vec<CharacterData> },
    DialogueResponse { speaker: String, text: String, choices: Vec<String> },
    Error { message: String },
    Pong,
}
```

## Component Structure

```
Player/src/presentation/
├── components/
│   ├── visual_novel/
│   │   ├── mod.rs
│   │   ├── backdrop.rs       # Background image
│   │   ├── character_sprite.rs # Character display
│   │   ├── dialogue_box.rs   # Text display with typewriter
│   │   └── choice_menu.rs    # Player choices
│   ├── action_panel.rs       # Interaction buttons
│   └── shared/
│       ├── loading.rs
│       └── error_display.rs
├── views/
│   ├── pc_view.rs           # Main player view
│   ├── dm_view.rs           # DM control panel
│   └── lobby_view.rs        # Session join
└── state/
    ├── mod.rs
    ├── game_state.rs
    └── session_state.rs
```

## Acceptance Criteria
- [x] Player can connect to Engine via WebSocket
- [x] World data loads successfully from Engine (WorldSnapshot)
- [ ] Visual novel displays backdrop and character sprites
- [ ] Dialogue displays with typewriter effect
- [ ] Player can select choices or type custom input
- [ ] Actions are sent to Engine and responses received

## Dependencies
- Phase 1 (Foundation) ✅
- Phase 3 (Scene System - for WorldSnapshot format) ✅

## Notes
- Use Dioxus 0.7.2 signals for reactive state
- WebSocket connection should auto-reconnect on disconnect
- Visual novel should support mobile touch and desktop click
- Consider offline mode with cached world data (future)
- Player targets: Desktop (Linux via GTK/WebKit), Web (WASM via trunk), Android
- Tailwind CSS for styling with TTRPG theme (parchment, ink, blood, gold)
- Build commands: `task desktop`, `task web`, `task serve`

## Domain Entities Reference
See `plans/00-master-plan.md` PART 2 for complete Player specifications:
- GameSession, Participant, PlayerAction domain model
- DirectorialContext for DM view
- LLM integration flow (request → response → approval)
- ApprovalDecision workflow (Accept/Modify/Reject/TakeOver)
- UI wireframes for PC View, DM View, Tactical Combat View
