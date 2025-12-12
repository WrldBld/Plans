# Phase 16: Director Decision Queue

## Overview

During gameplay, the LLM continuously analyzes player actions and generates responses, tool calls, and challenge suggestions. The Director Decision Queue provides the DM with real-time visibility into all AI decision-making, allowing approval, modification, or rejection of each decision before it affects the game.

**Key Distinction**:
- **Generation Queue** (Creator Mode, [Phase 15](./15-unified-generation-queue.md)): Content creation (images, text suggestions)
- **Decision Queue** (Director Mode, this phase): Gameplay decisions (NPC responses, tool usage, challenges)

---

## User Stories

### Epic: Decision Visibility

#### US-16.1: View Pending AI Decisions
**As a** Dungeon Master
**I want to** see all pending AI decisions in a unified queue
**So that** I have complete visibility into what the AI is proposing

**Acceptance Criteria:**
- Decision Queue panel visible in Director Mode
- Shows all pending decisions with type indicators:
  - NPC Response (dialogue bubble icon)
  - Tool Usage (wrench icon)
  - Challenge Suggestion (dice icon)
  - Scene Transition (arrow icon)
- Decisions ordered by timestamp (oldest first)
- Each decision shows summary and confidence level
- Queue count badge in Director Mode header

---

#### US-16.2: Preview Decision Details
**As a** Dungeon Master
**I want to** expand a decision to see full details
**So that** I can make informed approval choices

**Acceptance Criteria:**
- Click/tap decision to expand details panel
- NPC Response shows:
  - Full dialogue text
  - Internal reasoning (why the AI chose this response)
  - Tone/emotion indicators
  - Character wants being addressed
- Tool Usage shows:
  - Tool type (GiveItem, RevealInfo, ChangeRelationship, TriggerEvent)
  - Parameters and targets
  - Reasoning for tool use
- Challenge Suggestion shows:
  - Challenge name and skill
  - Why LLM thinks it's relevant
  - Difficulty and potential outcomes
- Collapse to return to queue list

---

#### US-16.3: Approve Decision
**As a** Dungeon Master
**I want to** quickly approve an AI decision
**So that** gameplay can proceed smoothly

**Acceptance Criteria:**
- "Approve" button (green checkmark) on each decision
- Keyboard shortcut: Enter or A
- On approval:
  - Decision executed immediately
  - Removed from queue
  - Logged to conversation history
  - Players see result (dialogue, item, challenge prompt, etc.)
- Batch approve: "Approve All" for trusted decisions

---

#### US-16.4: Reject Decision
**As a** Dungeon Master
**I want to** reject an AI decision
**So that** I can prevent unwanted narrative directions

**Acceptance Criteria:**
- "Reject" button (red X) on each decision
- Keyboard shortcut: Escape or R
- On rejection:
  - Decision discarded
  - Removed from queue
  - Optional: Add reason for rejection (helps AI learn)
  - LLM notified to generate alternative (if applicable)
- Rejection logged for session review

---

#### US-16.5: Modify Decision Before Approval
**As a** Dungeon Master
**I want to** edit an AI decision before approving
**So that** I can fine-tune the narrative

**Acceptance Criteria:**
- "Edit" button (pencil icon) on each decision
- Opens inline editor:
  - NPC Response: Edit dialogue text, adjust tone
  - Tool Usage: Change parameters, targets
  - Challenge: Adjust difficulty, change skill
- "Save & Approve" commits modified decision
- "Cancel" returns to original decision
- Modified decisions marked with "Edited by DM" indicator

---

#### US-16.6: Delay Decision
**As a** Dungeon Master
**I want to** delay a decision for later
**So that** I can handle more urgent matters first

**Acceptance Criteria:**
- "Delay" button (clock icon) on each decision
- Keyboard shortcut: D
- Decision moves to "Delayed" section of queue
- Delayed decisions don't block new decisions
- Can return delayed decisions to active queue
- Auto-expire option: discard if not handled within X minutes

---

### Epic: Decision Filtering & Organization

#### US-16.7: Filter Decisions by Type
**As a** Dungeon Master
**I want to** filter the queue by decision type
**So that** I can focus on specific types of decisions

**Acceptance Criteria:**
- Filter tabs: All | Dialogue | Tools | Challenges | Transitions
- Active filter highlighted
- Count shown per filter
- Keyboard shortcuts: 1-5 for each filter
- Filter persists during session

---

#### US-16.8: Priority Indicators
**As a** Dungeon Master
**I want to** see which decisions are time-sensitive
**So that** I can prioritize appropriately

**Acceptance Criteria:**
- Urgency indicators:
  - Normal (no indicator)
  - Awaiting Player (yellow - player is waiting for response)
  - Scene Critical (red - blocks scene progression)
- Sort by urgency option
- Time elapsed shown (e.g., "30s ago")

---

### Epic: Decision History

#### US-16.9: View Decision History
**As a** Dungeon Master
**I want to** review past decisions
**So that** I can understand the AI's behavior patterns

**Acceptance Criteria:**
- "History" tab in Decision Queue panel
- Shows last 50 decisions with outcomes
- Indicators: Approved, Rejected, Modified, Expired
- Click to view full decision details
- Filter by outcome type

---

#### US-16.10: Undo Recent Decision
**As a** Dungeon Master
**I want to** undo a recently approved decision
**So that** I can correct mistakes

**Acceptance Criteria:**
- "Undo" available for last 3 approved decisions
- Only for decisions < 2 minutes old
- Undo removes effects:
  - Dialogue removed from conversation
  - Items returned
  - Relationship changes reverted
- Confirmation required: "Undo will remove this from the story"
- Some decisions non-undoable (scene transitions)

---

## UI Mockups

### Director Mode with Decision Queue

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  DIRECTOR MODE                                     [Director] [Creator] [Settings]      │
├───────────────────────────────────────────────────────┬─────────────────────────────────┤
│                                                       │                                 │
│  SCENE: The Dusty Library                             │  DECISION QUEUE          [3]   │
│  ─────────────────────────────────────────────────    │  ───────────────────────────── │
│                                                       │                                 │
│  [Scene preview with backdrop and characters]         │  [All] [💬] [🔧] [🎲] [→]       │
│                                                       │                                 │
│                                                       │  ┌─────────────────────────┐   │
│  CONVERSATION LOG                                     │  │ 💬 Jasper's Response    │   │
│  ─────────────────────────────────────────────────    │  │ "The tome? Dangerous..."│   │
│                                                       │  │ ⏱ 15s  ●●●○○            │   │
│  Marcus: "I want to examine the old books on         │  │ [✓] [✗] [✏️] [⏰]        │   │
│  that shelf over there."                              │  └─────────────────────────┘   │
│                                                       │                                 │
│  [Waiting for DM to approve response...]              │  ┌─────────────────────────┐   │
│                                                       │  │ 🔧 RevealInfo           │   │
│                                                       │  │ "Map in book binding"   │   │
│                                                       │  │ ⏱ 12s  ●●○○○            │   │
│                                                       │  │ [✓] [✗] [✏️] [⏰]        │   │
│                                                       │  └─────────────────────────┘   │
│                                                       │                                 │
│  ─────────────────────────────────────────────────    │  ┌─────────────────────────┐   │
│  DIRECTORIAL CONTROLS                                 │  │ 🎲 Challenge Suggested  │   │
│  Tone: [Mysterious ▼]  Pacing: [Slow ▼]              │  │ Library Use (Hard)      │   │
│  Active NPCs: Jasper, Guard                           │  │ ⏱ 8s   ●●●●○            │   │
│                                                       │  │ [✓] [✗] [✏️] [⏰]        │   │
│                                                       │  └─────────────────────────┘   │
│                                                       │                                 │
│                                                       │  [Approve All Safe]            │
│                                                       │                                 │
└───────────────────────────────────────────────────────┴─────────────────────────────────┘
```

### Expanded Decision Detail - NPC Response

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  💬 NPC RESPONSE: Jasper the Bartender                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROPOSED DIALOGUE                                                          │
│  ─────────────────────────────────────────────────────────────────────────  │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ "The tome? Ah, you've got a keen eye, friend. That one's been here   │ │
│  │ longer than I have. Some say it whispers at night... but that's just │ │
│  │ old tavern talk, surely."                                             │ │
│  │                                                                       │ │
│  │ *Jasper's eyes flicker nervously toward the bookshelf*                │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  AI REASONING                                                               │
│  ─────────────────────────────────────────────────────────────────────────  │
│  • Character want: "Keep the book's secrets hidden" → Deflecting           │
│  • Archetype: Shapeshifter → Revealing partial truth                       │
│  • Directorial note: "Build suspense around the tome"                      │
│  • Tone setting: Mysterious → Added atmospheric detail                     │
│                                                                             │
│  TONE               CONFIDENCE                                              │
│  [Mysterious ▼]     ●●●●○ High (82%)                                       │
│                                                                             │
│  ATTACHED TOOLS                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ 🔧 RevealInfo: "The bartender knows more than he's letting on"       │ │
│  │    Target: Player Marcus                                              │ │
│  │    [Include ✓] [Remove]                                               │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  [Cancel]                      [Edit Response]         [Approve Response]   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Expanded Decision Detail - Challenge Suggestion

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🎲 CHALLENGE SUGGESTION                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SUGGESTED CHALLENGE                                                        │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  📚 Library Use                                                             │
│  "Research the Ancient Tome"                                                │
│                                                                             │
│  Difficulty: Hard (skill/2)                                                 │
│  Target: Marcus (skill: 45%, needs ≤22)                                    │
│                                                                             │
│  AI REASONING                                                               │
│  ─────────────────────────────────────────────────────────────────────────  │
│  • Player action: "examine the old books" matches trigger                   │
│  • Scene has active challenge "Research the Tome" ready                     │
│  • This could reveal the hidden map (story progression)                     │
│  • Confidence: ●●●●○ (85%)                                                 │
│                                                                             │
│  POTENTIAL OUTCOMES                                                         │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  ✓ Success:                                                                 │
│    "Discovers this is a rare copy of the Necronomicon. The binding         │
│    contains a hidden map showing a location in the Miskatonic Valley."     │
│    → Enables: "Notice Hidden Door" challenge                                │
│                                                                             │
│  ✗ Failure:                                                                 │
│    "The text is in a language you don't recognize. You feel uneasy         │
│    looking at the strange symbols."                                         │
│    → Triggers: Sanity Check (0/1)                                          │
│                                                                             │
│  MODIFY BEFORE APPROVAL                                                     │
│  ─────────────────────────────────────────────────────────────────────────  │
│  Difficulty: [Hard ▼]     Skill: [Library Use ▼]                           │
│                                                                             │
│  [Reject]  [Delay - Not Yet]               [Trigger Challenge]              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Editing a Decision

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ✏️ EDIT NPC RESPONSE: Jasper the Bartender                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  EDIT DIALOGUE                                                              │
│  ─────────────────────────────────────────────────────────────────────────  │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ "The tome? Ah, you've got a keen eye, friend. That one's been here   │ │
│  │ longer than I have. Some say it whispers at night... but that's just │ │
│  │ old tavern talk, surely. Though I wouldn't read it after dark, if I  │ │
│  │ were you."                                                            │ │
│  │                                                                       │ │
│  │ *Jasper's eyes flicker nervously toward the bookshelf*                │ │
│  │ |                                                                     │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  [🪄 Suggest Alternative]  ← LLM suggestion appears in Generation Queue    │
│                                                                             │
│  TONE                                                                       │
│  [Mysterious ▼]  [Threatening]  [Friendly]  [Nervous]                      │
│                                                                             │
│  INCLUDE ACTIONS                                                            │
│  [✓] Eye movement toward bookshelf                                         │
│  [ ] Lower voice to whisper                                                 │
│  [ ] Clean glass nervously                                                  │
│                                                                             │
│  [Cancel Edit]                                          [Save & Approve]    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Decision History

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DECISION HISTORY                                          [← Back to Queue] │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Filter: [All ▼]                                    Session: 45 decisions   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ✓ APPROVED  5:23 PM                                                 │   │
│  │ 💬 Jasper: "The tome? Ah, you've got a keen eye..."                │   │
│  │ Modified by DM: Added warning about reading at night               │   │
│  │                                                         [View] [Undo]│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ✗ REJECTED  5:22 PM                                                 │   │
│  │ 🔧 ChangeRelationship: Marcus → Jasper (Trust +2)                  │   │
│  │ Reason: "Too early for trust increase"                              │   │
│  │                                                              [View] │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ✓ APPROVED  5:20 PM                                                 │   │
│  │ 💬 Guard: "Move along, nothing to see here."                       │   │
│  │                                                              [View] │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ⏰ DELAYED  5:18 PM → EXPIRED                                       │   │
│  │ 🎲 Challenge: Perception check to notice shadowy figure             │   │
│  │ Auto-expired after 5 minutes                                        │   │
│  │                                                              [View] │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  [Load More...]                                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Queue Badge in Director Mode Header

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  [Director Mode]  [Creator Mode]  [Settings]                       [← Back] │
│       ┌───┐                                                                 │
│       │ 3 │  ← Red badge when decisions pending                            │
│       └───┘                                                                 │
│            ┌───┐                                                            │
│            │ 2 │  ← Yellow badge for generation queue (Creator Mode)       │
│            └───┘                                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture

```
Player Action
      │
      ▼
┌─────────────────┐
│  Engine (LLM)   │
│  Processes action,
│  generates response,
│  proposes tools,
│  suggests challenges
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│                    DECISION QUEUE (Engine)                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ NPC Response│  │ Tool Usage  │  │ Challenge   │  ...    │
│  │ (pending)   │  │ (pending)   │  │ (pending)   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└────────────────────────────┬────────────────────────────────┘
                             │
                             │ WebSocket Events
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                 DECISION QUEUE (Player UI)                   │
│                      Director Mode                           │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  [All] [💬 Dialogue] [🔧 Tools] [🎲 Challenges]        ││
│  │                                                         ││
│  │  ┌─────────────────────────────────────────────────┐   ││
│  │  │ Pending Decision                    [✓][✗][✏️]  │   ││
│  │  └─────────────────────────────────────────────────┘   ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                             │
                             │ DM Approval/Rejection
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    EXECUTION                                 │
│  • Dialogue sent to players                                  │
│  • Tools executed (items given, info revealed, etc.)        │
│  • Challenges triggered                                      │
│  • Scene transitions                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Technical Design

### Engine: Decision Queue State

```rust
// Engine/src/domain/entities/decision.rs

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PendingDecision {
    pub id: DecisionId,
    pub session_id: SessionId,
    pub decision_type: DecisionType,
    pub created_at: DateTime<Utc>,
    pub urgency: DecisionUrgency,
    pub status: DecisionStatus,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum DecisionType {
    NpcResponse {
        character_id: CharacterId,
        dialogue: String,
        reasoning: String,
        tone: String,
        attached_tools: Vec<ProposedTool>,
    },
    ToolUsage {
        tool: GameTool,
        reasoning: String,
        target: Option<CharacterId>,
    },
    ChallengeSuggestion {
        challenge_id: ChallengeId,
        reasoning: String,
        confidence: f32,
    },
    SceneTransition {
        from_scene: SceneId,
        to_scene: SceneId,
        reasoning: String,
    },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DecisionUrgency {
    Normal,
    AwaitingPlayer,  // Player is waiting for response
    SceneCritical,   // Blocks scene progression
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DecisionStatus {
    Pending,
    Approved { by_dm: bool, modified: bool },
    Rejected { reason: Option<String> },
    Delayed { until: Option<DateTime<Utc>> },
    Expired,
}
```

### WebSocket Events

```rust
// Engine/src/domain/events/decision_events.rs

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum DecisionEvent {
    // Engine → Player (DM only)
    DecisionPending {
        decision: PendingDecision,
    },
    DecisionUpdated {
        decision_id: DecisionId,
        status: DecisionStatus,
    },
    DecisionExpired {
        decision_id: DecisionId,
    },

    // Player → Engine (DM actions)
    ApproveDecision {
        decision_id: DecisionId,
        modifications: Option<DecisionModification>,
    },
    RejectDecision {
        decision_id: DecisionId,
        reason: Option<String>,
    },
    DelayDecision {
        decision_id: DecisionId,
        expire_after_minutes: Option<u32>,
    },
}

#[derive(Serialize, Deserialize)]
pub struct DecisionModification {
    pub dialogue: Option<String>,
    pub tone: Option<String>,
    pub tool_changes: Option<Vec<ToolModification>>,
    pub difficulty: Option<String>,
}
```

### Player: Decision Queue State

```rust
// Player/src/presentation/state/decision_state.rs

pub struct DecisionState {
    pub pending: Signal<Vec<PendingDecision>>,
    pub delayed: Signal<Vec<PendingDecision>>,
    pub history: Signal<VecDeque<DecisionHistoryEntry>>,
    pub filter: Signal<DecisionFilter>,
    pub expanded_decision: Signal<Option<DecisionId>>,
}

#[derive(Clone, Default)]
pub enum DecisionFilter {
    #[default]
    All,
    Dialogue,
    Tools,
    Challenges,
    Transitions,
}

pub struct DecisionHistoryEntry {
    pub decision: PendingDecision,
    pub outcome: DecisionOutcome,
    pub timestamp: DateTime<Utc>,
    pub can_undo: bool,
}
```

---

## Implementation Tasks

### Phase 16A: Engine Decision Queue

- [ ] **16A.1** Create PendingDecision entity
  - File: `Engine/src/domain/entities/decision.rs`
  - DecisionType enum with all variants
  - DecisionUrgency and DecisionStatus

- [ ] **16A.2** Create DecisionQueueService
  - File: `Engine/src/application/services/decision_queue.rs`
  - Queue management (add, approve, reject, delay)
  - Expiration handling for delayed decisions
  - History tracking

- [ ] **16A.3** Integrate with LLM processing
  - File: `Engine/src/infrastructure/websocket.rs`
  - Instead of executing LLM responses immediately, queue them
  - Send DecisionPending events to DM

- [ ] **16A.4** Handle DM decision actions
  - File: `Engine/src/infrastructure/websocket.rs`
  - Process ApproveDecision, RejectDecision, DelayDecision
  - Execute approved decisions
  - Notify players of results

### Phase 16B: Player Decision Queue UI

- [ ] **16B.1** Create DecisionState
  - File: `Player/src/presentation/state/decision_state.rs`
  - Signals for pending, delayed, history
  - Filter and expansion state

- [ ] **16B.2** Handle decision WebSocket events
  - File: `Player/src/infrastructure/websocket/handlers.rs`
  - Parse DecisionPending, DecisionUpdated, DecisionExpired
  - Update DecisionState

- [ ] **16B.3** Create DecisionQueuePanel component
  - File: `Player/src/presentation/components/director/decision_queue.rs`
  - List pending decisions with type icons
  - Filter tabs
  - Urgency indicators

- [ ] **16B.4** Create DecisionDetail component
  - File: `Player/src/presentation/components/director/decision_detail.rs`
  - Expandable detail view for each decision type
  - Approve/Reject/Edit/Delay buttons

- [ ] **16B.5** Create DecisionEditor component
  - File: `Player/src/presentation/components/director/decision_editor.rs`
  - Inline editing for dialogue and parameters
  - Integration with suggestion button for alternatives

- [ ] **16B.6** Create DecisionHistory component
  - File: `Player/src/presentation/components/director/decision_history.rs`
  - List past decisions with outcomes
  - Undo functionality for recent decisions

### Phase 16C: Integration

- [ ] **16C.1** Add queue badge to Director Mode header
  - File: `Player/src/presentation/views/dm_view.rs`
  - Show pending count badge
  - Visual indicator for urgency

- [ ] **16C.2** Keyboard shortcuts
  - File: `Player/src/presentation/views/dm_view.rs`
  - Enter/A: Approve selected
  - Escape/R: Reject selected
  - E: Edit selected
  - D: Delay selected
  - 1-5: Filter by type

- [ ] **16C.3** Auto-approval settings
  - File: `Player/src/presentation/components/settings/decision_settings.rs`
  - Option to auto-approve low-risk decisions
  - Confidence threshold for auto-approval
  - Per-decision-type settings

---

## Dependencies

- LLM Integration (Phase 1) - generates decisions
- Challenge System (Phase 14) - challenge suggestions
- WebSocket infrastructure - real-time updates

---

## Relationship with Generation Queue

| Aspect | Generation Queue (Phase 15) | Decision Queue (Phase 16) |
|--------|---------------------------|---------------------------|
| **Location** | Creator Mode | Director Mode |
| **Purpose** | Create content | Control gameplay |
| **Contents** | Images, text suggestions | NPC responses, tools, challenges |
| **Timing** | Asynchronous (can take minutes) | Real-time (seconds) |
| **Urgency** | Low (DM can work on other things) | High (players waiting) |
| **Result** | Assets/suggestions for forms | Actions affecting players |

Both queues share:
- WebSocket-based real-time updates
- Progress/status visibility
- Approve/reject patterns
- History tracking

---

## Future Enhancements

- **AI Learning**: Track rejection patterns to improve suggestions
- **Batch Decisions**: Group related decisions for bulk approval
- **Templates**: Save common modifications as reusable templates
- **Voice Commands**: Approve/reject via voice in VR/AR scenarios
- **Delegation**: Allow trusted players to approve certain decision types
