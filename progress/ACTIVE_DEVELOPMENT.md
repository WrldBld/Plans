# Active Development

Active implementation tracking for WrldBldr user stories.

**Current Phase**: Phase A - Core Player Experience  
**Last Updated**: 2025-12-18

---

## Phase Overview

| Phase | Focus | Status | Est. Effort |
|-------|-------|--------|-------------|
| A | Core Player Experience | **ACTIVE** | 3-4 days |
| B | Player Knowledge & Agency | Pending | 4-5 days |
| C | DM Tools & Advanced Features | Pending | 5-7 days |

---

## Phase A: Core Player Experience

Critical features for basic player gameplay. These complete the core gameplay loop.

### US-NAV-008: Navigation Options UI

| Field | Value |
|-------|-------|
| **Status** | Not Started |
| **Priority** | Critical (blocking gameplay) |
| **Effort** | 1 day |
| **System** | [Navigation](../systems/navigation-system.md) |

**Description**: Player UI buttons for region movement and location exits.

**Implementation Notes**:
- Engine: Complete (WebSocket handlers for `MoveToRegion`, `ExitToLocation`)
- Player: Not started
- Create `NavigationPanel` component
- Add to `pc_view.rs` action panel area
- Wire up WebSocket messages
- Show locked/unlocked states

---

### US-NAV-009: Game Time Display

| Field | Value |
|-------|-------|
| **Status** | Not Started |
| **Priority** | High |
| **Effort** | 0.5 days |
| **System** | [Navigation](../systems/navigation-system.md) |

**Description**: Display current in-game time to players.

**Implementation Notes**:
- Engine: Complete (`GameTimeUpdated` WebSocket message)
- Player: Not started
- Create `GameTimeDisplay` component
- Add `current_game_time` to session state
- Update from `GameTimeUpdated` handler

---

### US-NPC-008: Approach Event Display

| Field | Value |
|-------|-------|
| **Status** | Partial |
| **Priority** | High |
| **Effort** | 0.5 days |
| **System** | [NPC](../systems/npc-system.md) |

**Description**: Visual notification when NPC approaches player.

**Implementation Notes**:
- Engine: Complete
- Player: Logs to conversation, needs visual overlay
- Create `ApproachEventOverlay` component
- Show NPC sprite sliding in with description
- Add "Continue" button to dismiss

---

### US-NPC-009: Location Event Display

| Field | Value |
|-------|-------|
| **Status** | Partial |
| **Priority** | High |
| **Effort** | 0.5 days |
| **System** | [NPC](../systems/npc-system.md) |

**Description**: Visual notification for location-wide events.

**Implementation Notes**:
- Engine: Complete
- Player: Logs to conversation, needs event banner
- Create `LocationEventBanner` component
- Styled narrative overlay
- Auto-dismiss or click to dismiss

---

## Phase B: Player Knowledge & Agency

Enhance player information and interaction capabilities.

### US-OBS-004/005: Known NPCs Panel

| Field | Value |
|-------|-------|
| **Status** | Not Started |
| **Priority** | Medium |
| **Effort** | 2 days |
| **System** | [Observation](../systems/observation-system.md) |

**Description**: Player view of observed NPCs with last seen info.

**Implementation Notes**:
- Engine: Complete (observation endpoints exist)
- Player: Not started
- Create `KnownNpcsPanel` component
- Query observations endpoint
- Show observation type icons (direct/heard/deduced)
- Display last seen location and game time

---

### US-CHAR-009: Inventory Panel

| Field | Value |
|-------|-------|
| **Status** | Not Started |
| **Priority** | Medium |
| **Effort** | 1.5 days |
| **System** | [Character](../systems/character-system.md) |

**Description**: Player inventory with equipped items and actions.

**Implementation Notes**:
- Engine: Complete (`POSSESSES` edges exist)
- Player: Button placeholder only
- Create `InventoryPanel` component
- Fetch items via PC endpoint
- Show equipped status, quantity
- Wire up use/drop/equip actions

---

### US-NAV-010: Mini-map with Clickable Regions

| Field | Value |
|-------|-------|
| **Status** | Not Started |
| **Priority** | Medium |
| **Effort** | 1.5 days |
| **System** | [Navigation](../systems/navigation-system.md) |

**Description**: Visual map showing regions with click-to-navigate.

**Implementation Notes**:
- Engine: Complete (region bounds data exists)
- Player: Not started
- Create `MiniMap` component
- Render region bounds as clickable areas
- Highlight current region
- Click region to navigate

---

## Phase C: DM Tools & Advanced Features

Improve DM workflow. These don't block player gameplay.

### US-CHAL-010: Region-level Challenge Binding

| Field | Value |
|-------|-------|
| **Status** | Not Started |
| **Priority** | Medium |
| **Effort** | 2 days |
| **System** | [Challenge](../systems/challenge-system.md) |

**Description**: Bind challenges to specific regions, not just locations.

**Implementation Notes**:
- Engine: Schema referenced but not implemented
- Player: Not started
- Add `AVAILABLE_AT_REGION` edge to challenge repository
- Add `list_by_region()` repository method
- Add region filter to challenge service
- Update Director Mode to show region-bound challenges

---

### US-SCN-009: Scene Entry Conditions

| Field | Value |
|-------|-------|
| **Status** | Partial |
| **Priority** | Medium |
| **Effort** | 0.5 days |
| **System** | [Scene](../systems/scene-system.md) |

**Description**: Evaluate conditions before showing a scene.

**Implementation Notes**:
- Engine: `SceneCondition` enum exists, evaluation missing
- Player: N/A (engine feature)
- Add `evaluate_conditions()` helper function
- Call from `scene_resolution_service.resolve_scene()`
- Check CompletedScene, HasItem, KnowsCharacter, FlagSet conditions

---

### US-NAR-009: Visual Trigger Condition Builder

| Field | Value |
|-------|-------|
| **Status** | Not Started |
| **Priority** | Low |
| **Effort** | 3-4 days |
| **System** | [Narrative](../systems/narrative-system.md) |

**Description**: Visual builder for narrative trigger conditions.

**Implementation Notes**:
- Engine: Trigger schema exists
- Player: Not started
- Add `/api/triggers/schema` endpoint for available types
- Create visual builder component with dropdowns
- Support all trigger types (location, NPC, challenge, time, etc.)
- Add AND/OR/AtLeast logic selection

---

### US-AST-010: Advanced Workflow Parameter Editor

| Field | Value |
|-------|-------|
| **Status** | Not Started |
| **Priority** | Low |
| **Effort** | 2 days |
| **System** | [Asset](../systems/asset-system.md) |

**Description**: Edit ComfyUI workflow parameters in UI.

**Implementation Notes**:
- Engine: Complete (workflow config exists)
- Player: Basic config exists
- Add prompt mapping editor
- Add locked inputs configuration
- Add style reference detection display
- Optional: Raw JSON viewer/editor

---

## Completed

Stories moved here when fully implemented.

### US-CHAL-009: Skill Modifiers Display During Rolls

| Field | Value |
|-------|-------|
| **Completed** | 2025-12-18 (discovered already complete) |
| **System** | [Challenge](../systems/challenge-system.md) |

**Implementation**: Full skill modifier display in challenge rolls.
- `SkillsDisplay` component shows all skills with modifiers
- `ChallengeRollModal` shows modifier in header and result breakdown
- Roll display: dice + modifier + skill = total

**Files**:
- `Player/src/presentation/components/tactical/challenge_roll.rs`
- `Player/src/presentation/components/tactical/skills_display.rs`

---

### US-DLG-009: Context Budget Configuration

| Field | Value |
|-------|-------|
| **Completed** | 2025-12-18 (discovered already complete) |
| **System** | [Dialogue](../systems/dialogue-system.md) |

**Implementation**: Full context budget configuration via Settings API.
- `GET/PUT /api/settings` exposes all 10 `ContextBudgetConfig` fields
- Per-world settings at `/api/worlds/{world_id}/settings`
- Metadata endpoint for UI field rendering

**Files**:
- `Engine/src/domain/value_objects/context_budget.rs`
- `Engine/src/infrastructure/http/settings_routes.rs`

---

## Progress Log

| Date | Phase | Story | Change |
|------|-------|-------|--------|
| 2025-12-18 | - | US-CHAL-009 | Marked complete (already implemented) |
| 2025-12-18 | - | US-DLG-009 | Marked complete (already implemented) |
| 2025-12-18 | - | - | Created ACTIVE_DEVELOPMENT.md |
