# Director Decision Queue: Plans vs Implementation Analysis

**Date**: 2025-12-14  
**Status**: **COMPLETELY UNIMPLEMENTED**

## Executive Summary

The Director Decision Queue (Phase 16) is **completely unimplemented**. This is a critical feature for gameplay that allows DMs to approve, reject, or modify AI decisions (NPC responses, tool usage, challenge suggestions) before they affect the game. Currently, AI decisions are executed immediately without DM oversight.

**Implementation Coverage**: **0%** - No code exists for this feature

---

## Detailed Comparison

### Planned Features (from plans/16-director-decision-queue.md)

#### Epic: Decision Visibility

**US-16.1: View Pending AI Decisions**
- Decision Queue panel visible in Director Mode
- Shows all pending decisions with type indicators (NPC Response, Tool Usage, Challenge Suggestion, Scene Transition)
- Decisions ordered by timestamp (oldest first)
- Each decision shows summary and confidence level
- Queue count badge in Director Mode header

**US-16.2: Preview Decision Details**
- Click/tap decision to expand details panel
- NPC Response shows: full dialogue, internal reasoning, tone/emotion, character wants
- Tool Usage shows: tool type, parameters, targets, reasoning
- Challenge Suggestion shows: challenge name, skill, relevance reasoning, difficulty, outcomes
- Collapse to return to queue list

**US-16.3: Approve Decision**
- "Approve" button (green checkmark) on each decision
- Keyboard shortcut: Enter or A
- On approval: decision executed, removed from queue, logged, players see result
- Batch approve: "Approve All" for trusted decisions

**US-16.4: Reject Decision**
- "Reject" button (red X) on each decision
- Keyboard shortcut: Escape or R
- On rejection: decision discarded, removed from queue, optional reason, LLM notified for alternative
- Rejection logged for session review

**US-16.5: Modify Decision Before Approval**
- "Edit" button (pencil icon) on each decision
- Opens inline editor for dialogue, tone, tool parameters, challenge difficulty
- "Save & Approve" commits modified decision
- "Cancel" returns to original decision
- Modified decisions marked with "Edited by DM" indicator

**US-16.6: Delay Decision**
- "Delay" button (clock icon) on each decision
- Keyboard shortcut: D
- Decision moves to "Delayed" section
- Delayed decisions don't block new decisions
- Can return delayed decisions to active queue
- Auto-expire option: discard if not handled within X minutes

#### Epic: Decision Filtering & Organization

**US-16.7: Filter Decisions by Type**
- Filter tabs: All, Dialogue, Tools, Challenges, Transitions
- Quick filter buttons in queue header
- Filter state persists across sessions

**US-16.8: Sort Decisions**
- Sort by: Timestamp (default), Urgency, Confidence, Type
- Ascending/descending toggle

**US-16.9: Decision History**
- History panel showing past decisions
- Shows: decision type, outcome (approved/rejected/modified), timestamp
- Undo functionality for recent decisions (if applicable)
- Search history by type, date range, character

#### Epic: Auto-Approval Settings

**US-16.10: Configure Auto-Approval Rules**
- Settings panel for decision auto-approval
- Per-decision-type settings:
  - Auto-approve NPC responses above confidence threshold
  - Auto-approve tool usage for specific tools
  - Auto-approve challenges below difficulty threshold
- Global toggle: Enable/disable auto-approval
- Confidence threshold slider (0-100%)

---

## Current Implementation Status

### Backend (Engine)

**Status**: ❌ **NOT IMPLEMENTED**

**Missing Components**:
- ❌ `Engine/src/domain/entities/decision.rs` - PendingDecision entity
- ❌ `Engine/src/application/services/decision_queue.rs` - DecisionQueueService
- ❌ Decision queue integration with LLM processing
- ❌ WebSocket events for decision queue (DecisionPending, DecisionUpdated, etc.)
- ❌ DM decision action handlers (ApproveDecision, RejectDecision, DelayDecision)

**Current Behavior**:
- AI decisions (NPC responses, tool calls, challenges) are executed **immediately** without DM approval
- No queue system exists
- No way to review or modify AI decisions before execution

**Evidence**:
- `grep -r "DecisionQueue\|decision.*queue\|PendingDecision"` returns 0 matches in Engine
- `Engine/src/infrastructure/websocket.rs` executes LLM responses directly without queuing

### Frontend (Player)

**Status**: ❌ **NOT IMPLEMENTED**

**Missing Components**:
- ❌ `Player/src/presentation/state/decision_state.rs` - DecisionState management
- ❌ `Player/src/presentation/components/director/decision_queue.rs` - DecisionQueuePanel component
- ❌ `Player/src/presentation/components/director/decision_detail.rs` - DecisionDetail component
- ❌ `Player/src/presentation/components/director/decision_editor.rs` - DecisionEditor component
- ❌ `Player/src/presentation/components/director/decision_history.rs` - DecisionHistory component
- ❌ WebSocket handlers for decision events
- ❌ Queue badge in Director Mode header
- ❌ Keyboard shortcuts for decision actions

**Current Behavior**:
- Director Mode has no decision queue UI
- No visibility into pending AI decisions
- No way to approve/reject/modify decisions

**Evidence**:
- `grep -r "DecisionQueue\|decision.*queue\|PendingDecision"` returns 0 matches in Player
- `Player/src/presentation/views/dm_view.rs` has no decision queue panel
- No decision-related components exist

---

## Domain Model Comparison

### Planned Domain Model

```rust
// Engine/src/domain/entities/decision.rs

pub struct PendingDecision {
    pub id: DecisionId,
    pub session_id: SessionId,
    pub decision_type: DecisionType,
    pub urgency: DecisionUrgency,
    pub status: DecisionStatus,
    pub created_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
}

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

pub enum DecisionUrgency {
    Normal,
    AwaitingPlayer,
    SceneCritical,
}

pub enum DecisionStatus {
    Pending,
    Approved { by_dm: bool, modified: bool },
    Rejected { reason: Option<String> },
    Delayed { until: Option<DateTime<Utc>> },
    Expired,
}
```

### Current Implementation

**Status**: ❌ **DOES NOT EXIST**

- No decision entity in domain model
- No decision queue service
- No decision-related types

---

## API Contract Comparison

### Planned WebSocket Events

```rust
// Engine → Player (DM only)
DecisionPending {
    decision: PendingDecision,
}
DecisionUpdated {
    decision_id: DecisionId,
    status: DecisionStatus,
}
DecisionExpired {
    decision_id: DecisionId,
}

// Player → Engine (DM actions)
ApproveDecision {
    decision_id: DecisionId,
    modifications: Option<DecisionModification>,
}
RejectDecision {
    decision_id: DecisionId,
    reason: Option<String>,
}
DelayDecision {
    decision_id: DecisionId,
    expire_after_minutes: Option<u32>,
}
```

### Current Implementation

**Status**: ❌ **DOES NOT EXIST**

- No decision-related WebSocket messages
- Current WebSocket messages only handle:
  - Player actions
  - NPC responses (executed immediately)
  - Tool calls (executed immediately)
  - Challenge triggers (executed immediately)

---

## UI Mockup Comparison

### Planned UI (from plans/16-director-decision-queue.md)

**Decision Queue Panel**:
- Sidebar panel in Director Mode
- List of pending decisions with type icons
- Expandable detail view
- Approve/Reject/Edit/Delay buttons
- Filter tabs (All, Dialogue, Tools, Challenges)
- Queue count badge in header
- History panel

### Current UI

**Status**: ❌ **DOES NOT EXIST**

- Director Mode has no decision queue panel
- No decision-related UI components
- No queue badge in header
- No decision history view

---

## Impact Assessment

### Critical Impact

**Blocking Core Gameplay Feature**: 
- DMs have **no control** over AI decisions during gameplay
- NPC responses, tool usage, and challenges execute immediately
- No way to review, modify, or reject AI decisions
- This is a **fundamental gap** in the DM control system

**User Story Coverage**: **0%**
- All 10 user stories (US-16.1 through US-16.10) are **completely unimplemented**

**Severity**: **CRITICAL**
- This feature is essential for DM control during gameplay
- Without it, DMs cannot prevent unwanted narrative directions
- AI decisions happen without oversight, which can break immersion and story consistency

---

## Recommendations

### Immediate Actions (Critical)

1. **Implement Backend Decision Queue**
   - Create `PendingDecision` entity
   - Create `DecisionQueueService` for queue management
   - Modify LLM processing to queue decisions instead of executing immediately
   - Add WebSocket events for decision queue

2. **Implement Frontend Decision Queue UI**
   - Create `DecisionState` for state management
   - Create `DecisionQueuePanel` component
   - Create `DecisionDetail` component for expanded view
   - Add queue badge to Director Mode header

3. **Implement Decision Actions**
   - Approve decision handler
   - Reject decision handler
   - Edit decision handler
   - Delay decision handler

### Short-term (High Priority)

4. **Add Decision History**
   - History panel component
   - Undo functionality for recent decisions

5. **Add Filtering & Sorting**
   - Filter by decision type
   - Sort by timestamp, urgency, confidence

6. **Add Keyboard Shortcuts**
   - Enter/A: Approve
   - Escape/R: Reject
   - E: Edit
   - D: Delay

### Medium-term

7. **Auto-Approval Settings**
   - Settings panel for auto-approval rules
   - Per-decision-type configuration
   - Confidence threshold settings

---

## Conclusion

The Director Decision Queue is **completely unimplemented** and represents a **critical gap** in the DM control system. Currently, all AI decisions execute immediately without DM oversight, which prevents DMs from maintaining narrative control during gameplay.

**Estimated Effort**: 3-4 weeks
- Backend: 1.5-2 weeks
- Frontend: 1.5-2 weeks

**Priority**: **CRITICAL** - This is a core gameplay feature that should be implemented before or alongside other gameplay features.
