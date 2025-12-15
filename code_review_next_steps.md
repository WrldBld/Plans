# Code Review: Next Steps Plan

**Created**: 2025-12-15
**Completed**: 2025-12-15
**Purpose**: Detailed plan for addressing code issues before feature development
**Target Audience**: AI agents implementing these changes

---

## ✅ Completion Status

**All critical issues have been resolved** (2025-12-15):

1. ✅ **GenerationService Wired** - Added to AppState, instantiated with event channel, generation event broadcaster worker implemented
2. ✅ **ChallengeSuggestionDecision Handler Complete** - Full implementation with approval lookup, challenge loading, and ChallengePrompt broadcasting
3. ✅ **NarrativeEventSuggestionDecision Handler Complete** - Full implementation with event loading, outcome selection, StoryEvent recording, and NarrativeEventTriggered broadcasting
4. ✅ **WebSocket Message Synchronization** - All message types synchronized between Engine and Player
5. ✅ **Compilation Verified** - Both Engine and Player compile successfully with `cargo check`

**Next Steps**: Phase 20 (Unified Generation Queue UI) and Phase 16 (Director Decision Queue) are now fully unblocked and ready for implementation.

---

## Bug Fixes

### Bug: Asset Gallery 404 Error During Character Creation (2025-12-15)

**Issue**: When creating a new character, the AssetGallery component attempts to fetch assets from `/api/character//gallery` (empty character_id), resulting in a 404 error.

**Root Cause**: The AssetGallery component always makes an API call on mount, even when `entity_id` is empty (new entity being created).

**Fix Applied**:
- ✅ Added check to skip API call when `entity_id.is_empty()`
- ✅ Updated UI to show helpful message: "Save the {entity_type} first to generate assets"
- ✅ Hide generate button when entity_id is empty (can't generate assets for non-existent entity)

**Files Modified**:
- `Player/src/presentation/components/creator/asset_gallery.rs`

**Note**: This same pattern likely affects Location and Item forms as well. The fix handles all entity types generically.

### Bug: "No world loaded" Error When Creating Characters/Locations (2025-12-15)

**Issue**: When creating a new character or location, the form shows "No world loaded" error because it tries to get `world_id` from `game_state.world.read()`, but with routing, the world_id comes from the URL route parameter instead.

**Root Cause**: After implementing routing (Phase 15), the `world_id` is passed as a route parameter to components, but `CharacterForm` and `LocationForm` were still trying to access it from `GameState`, which may not be populated.

**Fix Applied**:
- ✅ Added `world_id: String` prop to `CharacterForm` component
- ✅ Added `world_id: String` prop to `LocationForm` component
- ✅ Updated both forms to use the prop instead of `game_state.world.read()`
- ✅ Updated `CreatorMode` to pass `props.world_id` to both forms
- ✅ Fixed borrow checker issues by cloning `world_id` before `move` closures

**Files Modified**:
- `Player/src/presentation/components/creator/character_form.rs`
- `Player/src/presentation/components/creator/location_form.rs`
- `Player/src/presentation/components/creator/mod.rs`

**Note**: This pattern should be applied to any other forms that need `world_id` (ItemForm, etc.) when they are implemented.

### Refactor: Creator Mode Reactive Architecture (2025-12-15)

**Issue**: Entity browser lists didn't update after creating entities. Initial implementation used a hacky `refresh_counter` signal that triggered re-fetches.

**Root Cause**: Data was stored in child components (`CharacterList`, `LocationList`) with local signals. When data changed, there was no way to notify the lists except by incrementing a counter to trigger `use_effect` re-runs.

**Solution**: Implemented proper reactive architecture following existing codebase patterns (`SessionState`, `GenerationState`):

- ✅ **Moved signal ownership to CreatorMode** - Characters and locations lists stored as `Signal<Vec<...>>` at parent level
- ✅ **Removed refresh_counter hack** - No more counter increments
- ✅ **Updated EntityBrowser** - Receives signals as props instead of refresh trigger
- ✅ **Updated list components** - `CharacterList` and `LocationList` now read from prop signals (no local fetching)
- ✅ **Direct signal updates** - Forms update signals directly when saving:
  - **Create**: Add new item to signal (optimistic update)
  - **Update**: Find and update item in signal in-place
- ✅ **Initial data fetching** - CreatorMode fetches on mount, updates signals once

**Benefits**:
- True reactivity - components automatically update when signals change
- Optimistic updates - items appear immediately
- Single source of truth - data stored once, shared across components
- Better performance - no unnecessary re-fetches
- Cleaner code - follows established patterns

**Files Modified**:
- `Player/src/presentation/components/creator/mod.rs`
- `Player/src/presentation/components/creator/entity_browser.rs`
- `Player/src/presentation/components/creator/character_form.rs`
- `Player/src/presentation/components/creator/location_form.rs`

**Reference**: See `plans/creator_reactive_refactor.md` for detailed architecture analysis.

### Bug: Location Loading 404 Error (2025-12-15)

**Issue**: When clicking a location from the entity browser list, the form shows: `Failed to load location: HTTP 404: GET /api/worlds/{world_id}/locations/{location_id} failed`

**Root Cause**: The Player's `LocationService::get_location()` method was calling `/api/worlds/{world_id}/locations/{location_id}`, but the Engine API endpoint is `/api/locations/{id}` (without the world_id prefix).

**Fix Applied**:
- ✅ Updated `LocationService::get_location()` to use correct endpoint: `/api/locations/{location_id}`
- ✅ Kept `world_id` parameter for API compatibility (marked as unused with `_world_id`)

**Files Modified**:
- `Player/src/application/services/location_service.rs`

**Note**: This matches the pattern used by `update_location()` and `delete_location()` which also use `/api/locations/{id}`.

### Critical Oversight: LLM Suggestions Not Using Queue System (2025-12-15)

**Issue**: LLM suggestion requests are bypassing the queue system entirely, even though the infrastructure exists and Phase 20 explicitly calls for queued suggestions.

**Root Cause**: 
- `LLMReasoningQueue` infrastructure exists and can handle `LLMRequestType::Suggestion`
- But HTTP routes in `suggestion_routes.rs` call `SuggestionService` directly (synchronous)
- `LLMQueueService` has a placeholder handler that just logs and completes (doesn't process)
- No WebSocket events for suggestions
- Suggestions never enter the queue

**Impact**:
- ❌ Suggestions block HTTP requests (bad UX)
- ❌ No visibility into pending suggestions
- ❌ Can't unify with image generation queue (Phase 20 blocked)
- ❌ No progress tracking or error handling via queue

**Required Fixes**:
1. **Engine**: Update suggestion routes to enqueue to `LLMReasoningQueue` instead of direct calls
2. **Engine**: Implement actual suggestion processing in `LLMQueueService` handler
3. **Engine**: Add WebSocket events (`SuggestionQueued`, `SuggestionComplete`, `SuggestionFailed`)
4. **Engine**: Wire WebSocket broadcasting for suggestion events
5. **Player**: Extend `GenerationState` to track suggestion tasks
6. **Player**: Handle suggestion WebSocket events
7. **Player**: Update `SuggestionButton` to enqueue instead of synchronous HTTP
8. **Player**: Update `GenerationQueuePanel` to show suggestions

**Files to Modify**:
- `Engine/src/infrastructure/http/suggestion_routes.rs` - Enqueue instead of sync
- `Engine/src/application/services/llm_queue_service.rs` - Process suggestions
- `Engine/src/domain/events/generation_events.rs` - Add suggestion events
- `Engine/src/infrastructure/websocket.rs` - Broadcast suggestion events
- `Player/src/presentation/state/generation_state.rs` - Add suggestion tasks
- `Player/src/presentation/handlers/session_message_handler.rs` - Handle events
- `Player/src/presentation/components/creator/suggestion_button.rs` - Use queue
- `Player/src/presentation/components/creator/generation_queue.rs` - Show suggestions

**Reference**: See `plans/suggestion_queue_oversight.md` for detailed analysis and implementation plan.

**Priority**: **HIGH** - This is a prerequisite for Phase 20 completion.

---

## Executive Summary

After comprehensive review of the ROADMAP, user stories, and codebase, the following critical issues must be addressed before continuing with feature development:

### Critical Issues (Must Fix First)
1. **GenerationService Orphaned** - 509 lines of fully implemented code, never instantiated
2. **ChallengeSuggestionDecision Handler Incomplete** - Has TODO, no actual logic
3. **NarrativeEventSuggestionDecision Handler Incomplete** - Has TODO, no actual logic

### Recommended Priority Order
1. **Phase 18B** (Generation Event Wiring) - Unblocks Phase 20
2. **Phase 16** (Director Decision Queue) - Can parallel with 18B
3. **Complete Suggestion Decision Handlers** - Required for gameplay

---

## Issue 1: Orphaned GenerationService

### Location
- **File**: `Engine/src/application/services/generation_service.rs` (509 lines)
- **Exports**: `Engine/src/application/services/mod.rs` - exported but unused

### Problem Description
`GenerationService` is a complete, well-designed service for managing ComfyUI asset generation. It includes:
- `GenerationEvent` enum with 4 event types (BatchQueued, BatchProgress, BatchComplete, BatchFailed)
- `GenerationRequest` struct for queuing requests
- Full batch tracking with `BatchTracker` struct
- Event channel using `mpsc::UnboundedSender<GenerationEvent>`
- Methods: `queue_generation()`, `process_queue()`, `cancel_batch()`

**But it is NEVER instantiated in `AppState`.**

### Impact
- Phase 20 (Unified Generation Queue UI) cannot work without this
- Phase 18B (Generation Event Wiring) is blocked
- ComfyUI integration is incomplete - no real-time progress events

### Fix Instructions

**Step 1: Add GenerationService to AppState**

File: `Engine/src/infrastructure/state.rs`

Add after line 67 (after `dm_approval_queue_service`):
```rust
pub generation_service: Arc<GenerationService>,
```

Add import at top:
```rust
use crate::application::services::GenerationService;
```

**Step 2: Instantiate GenerationService in AppState::new()**

File: `Engine/src/infrastructure/state.rs`

After line 179 (after dm_approval_queue_service creation), add:
```rust
// Create event channel for generation service
let (generation_event_tx, generation_event_rx) = tokio::sync::mpsc::unbounded_channel();

// Create generation service
let generation_service = Arc::new(GenerationService::new(
    Arc::new(comfyui_client.clone()) as Arc<dyn crate::application::ports::outbound::ComfyUIPort>,
    asset_repo.clone(),
    std::path::PathBuf::from("./data/assets"),
    std::path::PathBuf::from("./workflows"),
    generation_event_tx,
));
```

Add `generation_service` to the struct initialization.

**Step 3: Wire Event Channel to WebSocket Broadcasting**

File: `Engine/src/main.rs`

Add a background worker to consume `generation_event_rx` and broadcast to sessions:

```rust
// Generation event worker
let generation_event_worker = {
    let sessions = state.sessions.clone();
    let mut rx = generation_event_rx;
    tokio::spawn(async move {
        tracing::info!("Starting generation event broadcaster");
        while let Some(event) = rx.recv().await {
            let sessions = sessions.read().await;
            // Broadcast to all DMs in sessions
            // Convert GenerationEvent to ServerMessage and send
            match event {
                GenerationEvent::BatchQueued { batch_id, position } => {
                    // Send GenerationQueued message
                }
                GenerationEvent::BatchProgress { batch_id, progress } => {
                    // Send GenerationProgress message
                }
                // ... etc
            }
        }
    })
};
```

**Step 4: Add Missing ServerMessage Variants**

File: `Engine/src/infrastructure/websocket.rs`

Verify these variants exist in `ServerMessage` enum:
```rust
GenerationQueued { batch_id: String, entity_type: String, entity_id: String, asset_type: String, position: u32 },
GenerationProgress { batch_id: String, progress: u8 },
GenerationComplete { batch_id: String, asset_count: u32 },
GenerationFailed { batch_id: String, error: String },
```

File: `Player/src/application/dto/websocket_messages.rs`

Add same variants if missing.

### Testing
After implementation:
1. Start Engine, verify no panic on startup
2. Call POST /api/assets/generate
3. Verify WebSocket receives GenerationQueued event
4. Wait for completion, verify GenerationComplete event

---

## Issue 2: Incomplete ChallengeSuggestionDecision Handler

### Location
- **File**: `Engine/src/infrastructure/websocket.rs` lines 665-711

### Problem Description
The handler for `ClientMessage::ChallengeSuggestionDecision` has this code:
```rust
if approved {
    // TODO: Extract challenge_id from request_id (approval item)
    // TODO: Trigger the challenge with modified_difficulty if provided
    tracing::info!("DM approved challenge suggestion for request {}", request_id);
}
```

The DM sees challenge suggestions in the approval popup but cannot actually trigger them.

### Impact
- Challenge suggestions from LLM are displayed but non-functional
- DM cannot approve challenges to trigger them for players

### Fix Instructions

**Step 1: Look up the approval item to get challenge details**

File: `Engine/src/infrastructure/websocket.rs`

Replace the TODO block (lines 695-709) with:
```rust
if approved {
    // Look up the approval item to get challenge details
    let approval_item = state.dm_approval_queue_service.get_by_id(&request_id).await;
    
    if let Ok(Some(item)) = approval_item {
        if let Some(challenge_suggestion) = &item.challenge_suggestion {
            // Parse challenge_id from the suggestion
            let challenge_uuid = match uuid::Uuid::parse_str(&challenge_suggestion.challenge_id) {
                Ok(uuid) => crate::domain::value_objects::ChallengeId::from_uuid(uuid),
                Err(_) => {
                    tracing::error!("Invalid challenge_id in suggestion: {}", challenge_suggestion.challenge_id);
                    return None;
                }
            };
            
            // Load the challenge
            let challenge = match state.challenge_service.get_challenge(challenge_uuid).await {
                Ok(Some(c)) => c,
                Ok(None) => {
                    tracing::error!("Challenge {} not found", challenge_suggestion.challenge_id);
                    return None;
                }
                Err(e) => {
                    tracing::error!("Failed to load challenge: {}", e);
                    return None;
                }
            };
            
            // Apply modified difficulty if provided
            let difficulty = if let Some(mod_diff) = modified_difficulty {
                // Parse the modified difficulty string (e.g., "DC 15" -> Difficulty::DC(15))
                crate::domain::entities::Difficulty::DC(mod_diff.parse().unwrap_or(10))
            } else {
                challenge.difficulty.clone()
            };
            
            // Get the target character (from the player who made the original action)
            let target_character_id = item.source_action_id.to_string(); // Need proper player-to-character mapping
            
            // Broadcast ChallengePrompt to players
            let prompt = ServerMessage::ChallengePrompt {
                challenge_id: challenge_suggestion.challenge_id.clone(),
                challenge_name: challenge.name.clone(),
                skill_name: challenge_suggestion.skill_name.clone(),
                difficulty_display: difficulty.display(),
                character_modifier: 0, // TODO: Get from character sheet when Phase 14C is complete
            };
            
            let sessions = state.sessions.read().await;
            if let Some(session_id) = sessions.get_client_session(client_id) {
                sessions.broadcast_to_session(session_id, &prompt);
            }
            
            tracing::info!(
                "Triggered challenge {} for session via suggestion",
                challenge_suggestion.challenge_id
            );
        } else {
            tracing::warn!("No challenge suggestion found in approval item {}", request_id);
        }
    } else {
        tracing::error!("Approval item {} not found", request_id);
    }
} else {
    tracing::info!("DM rejected challenge suggestion for request {}", request_id);
}
```

**Step 2: Add get_by_id method to DMApprovalQueueService if missing**

File: `Engine/src/application/services/dm_approval_queue_service.rs`

Check if this method exists. If not, add:
```rust
pub async fn get_by_id(&self, id: &str) -> Result<Option<ApprovalItem>, QueueError> {
    // Parse the id as QueueItemId and look up the item
    let uuid = uuid::Uuid::parse_str(id)?;
    let item_id = QueueItemId::from_uuid(uuid);
    self.queue.get(item_id).await.map(|opt| opt.map(|qi| qi.payload))
}
```

### Testing
1. Submit a player action that triggers an LLM response with challenge suggestion
2. DM receives approval popup with challenge suggestion
3. DM clicks "Approve Challenge"
4. Verify players receive ChallengePrompt message
5. Verify challenge roll flow works

---

## Issue 3: Incomplete NarrativeEventSuggestionDecision Handler

### Location
- **File**: `Engine/src/infrastructure/websocket.rs` lines 714-760

### Problem Description
The handler for `ClientMessage::NarrativeEventSuggestionDecision` has:
```rust
if approved {
    // TODO: Implement narrative event triggering
    // 1. Load the narrative event from repository
    // 2. Mark it as triggered (with selected_outcome)
    // 3. Execute any immediate effects
    // 4. Record a StoryEvent for the timeline
    // 5. Broadcast scene direction to DM
    tracing::info!("DM approved narrative event {} trigger", event_id);
}
```

### Impact
- Narrative event suggestions are displayed but cannot be triggered
- Story Arc feature (Phase 17) is incomplete

### Fix Instructions

**Step 1: Implement the narrative event triggering logic**

File: `Engine/src/infrastructure/websocket.rs`

Replace the TODO block (lines 736-746) with:
```rust
if approved {
    // 1. Load the narrative event from repository
    let event_uuid = match uuid::Uuid::parse_str(&event_id) {
        Ok(uuid) => crate::domain::value_objects::NarrativeEventId::from_uuid(uuid),
        Err(_) => {
            tracing::error!("Invalid event_id: {}", event_id);
            return None;
        }
    };
    
    let narrative_event = match state.narrative_event_service.get(event_uuid).await {
        Ok(Some(event)) => event,
        Ok(None) => {
            tracing::error!("Narrative event {} not found", event_id);
            return None;
        }
        Err(e) => {
            tracing::error!("Failed to load narrative event: {}", e);
            return None;
        }
    };
    
    // 2. Find the selected outcome (or default to first)
    let outcome = if let Some(outcome_id) = &selected_outcome {
        narrative_event.outcomes.iter()
            .find(|o| o.id.to_string() == *outcome_id)
            .cloned()
            .unwrap_or_else(|| narrative_event.outcomes.first().cloned().unwrap())
    } else {
        narrative_event.outcomes.first().cloned()
            .ok_or_else(|| {
                tracing::error!("Narrative event {} has no outcomes", event_id);
            }).ok()?
    };
    
    // 3. Mark event as triggered
    // This would update the event's status in the database
    // For now, we'll just record the story event
    
    // 3. Mark event as triggered
    if let Err(e) = state.narrative_event_service.mark_triggered(
        event_uuid, 
        Some(outcome.name.clone())
    ).await {
        tracing::error!("Failed to mark narrative event as triggered: {}", e);
    }
    
    // 4. Record a StoryEvent for the timeline
    // Get session_id for the story event
    let sessions = state.sessions.read().await;
    let session_id = sessions.get_client_session(client_id);
    if let Some(session_id) = session_id {
        if let Err(e) = state.story_event_service.record_narrative_event_triggered(
            narrative_event.world_id,
            crate::domain::value_objects::SessionId::from_uuid(
                uuid::Uuid::parse_str(session_id).unwrap_or_default()
            ),
            None, // scene_id
            None, // location_id
            event_uuid,
            narrative_event.name.clone(),
            Some(outcome.name.clone()),
            outcome.effects.iter().map(|e| format!("{:?}", e)).collect(),
            vec![], // involved_characters
            None, // game_time
        ).await {
            tracing::error!("Failed to record story event: {}", e);
        }
    }
    drop(sessions); // Release read lock before broadcast
    
    // 5. Broadcast scene direction to DM
    let scene_direction = ServerMessage::NarrativeEventTriggered {
        event_id: event_id.clone(),
        event_name: narrative_event.name.clone(),
        outcome_description: outcome.description.clone(),
        scene_direction: narrative_event.scene_direction.clone(),
    };
    
    let sessions = state.sessions.read().await;
    if let Some(session_id) = sessions.get_client_session(client_id) {
        sessions.send_to_dm_in_session(session_id, &scene_direction);
    }
    
    tracing::info!(
        "Triggered narrative event '{}' with outcome '{}'",
        narrative_event.name,
        outcome.description
    );
}
```

**Step 2: Add NarrativeEventTriggered to ServerMessage**

File: `Engine/src/infrastructure/websocket.rs`

Add to ServerMessage enum:
```rust
NarrativeEventTriggered {
    event_id: String,
    event_name: String,
    outcome_description: String,
    scene_direction: String,
},
```

File: `Player/src/application/dto/websocket_messages.rs`

Add same variant.

### Testing
1. Create a narrative event in the world
2. Submit player action that triggers LLM to suggest the event
3. DM receives approval popup with narrative event suggestion
4. DM clicks "Trigger Event"
5. Verify DM receives NarrativeEventTriggered message
6. Verify StoryEvent is recorded in timeline

---

## Issue 4: WebSocket Message Type Synchronization

### Problem Description
Engine and Player have separate message type definitions that must stay synchronized.

### Location
- Engine: `Engine/src/infrastructure/websocket.rs` - `ServerMessage`, `ClientMessage` enums
- Player: `Player/src/application/dto/websocket_messages.rs` - same enums

### Current State
After the code review fixes, the following variants were added to Player but need verification:
- `NarrativeEventSuggestionInfo` - Added to `PendingApproval` struct
- Narrative event UI in approval popup

### Verification Steps
1. Compare `ServerMessage` enum in both files - all variants must match
2. Compare `ClientMessage` enum in both files - all variants must match
3. Run `cargo check` on both projects

### Missing Variants to Add
Based on Phase 18B requirements (not yet implemented):
- `ComfyUIStateChanged { state: String, message: Option<String>, retry_in_seconds: Option<u32> }`
- Verify `GenerationQueued`, `GenerationProgress`, `GenerationComplete`, `GenerationFailed` exist

---

## Issue 5: Character Modifier Hardcoded (Phase 14C Dependency)

### Location
- **File**: `Engine/src/infrastructure/websocket.rs` lines 537 and 633

### Problem Description
Challenge prompts use `character_modifier = 0` instead of actual character skill modifiers:
```rust
// TODO (Phase 14C integration): Get actual character modifier from character sheet
let character_modifier = 0;
```

### Impact
- Challenges don't reflect character abilities
- Players don't get skill bonuses/penalties
- Dice rolls are less meaningful

### Fix (Deferred)
This requires Phase 14C (Character Sheets) which defines how character skill values are stored and retrieved. The fix involves:
1. Loading the character's sheet template
2. Finding the skill value for the challenge's skill
3. Calculating the modifier based on rule system

**This is NOT a blocker** - challenges work, just without modifiers.

---

## Issue 6: DDD Patterns Partially Wired (Phase 3.1)

### Status
Structures exist but are not wired into the application flow.

### Existing Code
- `Engine/src/domain/aggregates/world_aggregate.rs` (216 lines) - WorldAggregate with invariant enforcement
- `Engine/src/domain/events/domain_events.rs` (238 lines) - DomainEvent enum with 17 event types
- `Engine/src/application/ports/inbound/use_cases.rs` (177 lines) - Use case traits

All have `#[allow(dead_code)]` annotations with documentation explaining Phase 3.1 integration.

### Impact
- Low priority - application works without these patterns
- Current services access entities directly (acceptable)
- No event bus for cross-cutting concerns

### When to Address
Phase 3.1 can be done as technical debt cleanup after features are stable. It involves:
1. Wiring WorldAggregate into services (mutations go through aggregate)
2. Creating EventBus for domain events
3. Implementing use case traits on services

**NOT a blocker for Phase 16, 18, or 20.**

---

## Recommended Implementation Order

### Phase A: Foundation Fixes (MUST DO FIRST)
Estimated: 2-3 hours

1. **A.1**: Wire GenerationService to AppState (Issue 1)
2. **A.2**: Add generation event broadcasting worker
3. **A.3**: Verify/add missing WebSocket message types
4. **A.4**: Run cargo check on both projects

### Phase B: Handler Completions (Required for Gameplay)
Estimated: 2 hours

1. **B.1**: Complete ChallengeSuggestionDecision handler (Issue 2)
2. **B.2**: Complete NarrativeEventSuggestionDecision handler (Issue 3)
3. **B.3**: Add any missing service methods (get_by_id, etc.)

### Phase C: Phase 16 Implementation
Estimated: 4-6 hours
See `plans/16-director-decision-queue.md` for detailed tasks

### Phase D: Phase 20 Implementation
Estimated: 4-6 hours
See `plans/20-unified-generation-queue.md` for detailed tasks
NOTE: Blocked until Phase A.1-A.3 complete

---

## Files to Modify Summary

| File | Changes |
|------|---------|
| `Engine/src/infrastructure/state.rs` | Add GenerationService to AppState |
| `Engine/src/main.rs` | Add generation event broadcaster worker |
| `Engine/src/infrastructure/websocket.rs` | Complete ChallengeSuggestionDecision, NarrativeEventSuggestionDecision handlers; Add NarrativeEventTriggered message |
| `Engine/src/application/services/dm_approval_queue_service.rs` | Add get_by_id method if missing |
| `Player/src/application/dto/websocket_messages.rs` | Add NarrativeEventTriggered, verify ComfyUI messages |

---

## Dependencies Between Issues

```
Issue 1 (GenerationService) ──────────────────┐
                                              │
                                              ├──> Phase 20 (Unified Queue UI)
                                              │
Issue 4 (Message Sync) ───────────────────────┘

Issue 2 (ChallengeSuggestionDecision) ────────┐
                                              │
                                              ├──> Full Gameplay Loop
                                              │
Issue 3 (NarrativeEventSuggestionDecision) ───┘
```

---

## Acceptance Criteria

### Before Feature Development Can Proceed:
- [✅] GenerationService instantiated and operational (2025-12-15)
- [✅] Generation events broadcast via WebSocket (2025-12-15)
- [✅] ChallengeSuggestionDecision triggers actual challenges (2025-12-15)
- [✅] NarrativeEventSuggestionDecision triggers actual events (2025-12-15)
- [✅] All message types synchronized between Engine and Player (2025-12-15)
- [✅] `cargo check` passes on both projects (2025-12-15)
- [ ] Manual test: Full approval flow works (action → LLM → approval → execution) - *Ready for testing*

---

## Context for Implementing Agent

This plan was created after:
1. Reading ROADMAP.md completely (1800+ lines)
2. Reading Phase 16, 18, and 20 user story documents
3. Analyzing the codebase for unwired code
4. Running the previous code review that addressed:
   - Removed unused config fields
   - Fixed LLM queue service lookups
   - Removed empty modules
   - Annotated unused DDD structures
   - Added narrative event suggestion to UI

The codebase follows **Hexagonal Architecture**:
- `domain/` - Entities, value objects, aggregates (core business logic)
- `application/` - Services, ports (inbound/outbound), DTOs
- `infrastructure/` - Adapters (Neo4j, WebSocket, HTTP, ComfyUI, Ollama)
- `presentation/` (Player only) - Dioxus UI components

Services take Arc<dyn Port> for dependency injection. All queue services use the pluggable backend system (InMemory or SQLite).

---

## Appendix A: Architecture Notes

### Key Files
| File | Purpose |
|------|---------|
| `Engine/src/infrastructure/state.rs` | AppState struct - holds all services |
| `Engine/src/main.rs` | Entry point, starts workers and Axum server |
| `Engine/src/infrastructure/websocket.rs` | WebSocket handler with ClientMessage/ServerMessage enums |
| `Player/src/application/dto/websocket_messages.rs` | Player-side message enums (must sync with Engine) |
| `Engine/src/application/services/*.rs` | All business logic services |

### Service Pattern
All services follow this pattern:
```rust
pub struct MyService {
    repository: Arc<dyn MyRepositoryPort>,
}

impl MyService {
    pub fn new(repository: Arc<dyn MyRepositoryPort>) -> Self {
        Self { repository }
    }
    
    pub async fn do_something(&self) -> Result<...> {
        // Use self.repository
    }
}
```

### Queue System
Five queues exist (all operational):
- `PlayerActionQueue` - Player actions → LLM processing
- `DMActionQueue` - DM actions (scene transitions, etc.)
- `LLMReasoningQueue` - Ollama requests with semaphore control
- `AssetGenerationQueue` - ComfyUI requests (batch size = 1)
- `DMApprovalQueue` - Pending DM approvals

Queues use pluggable backends:
```rust
pub trait QueuePort<T> {
    async fn enqueue(&self, item: QueueItem<T>) -> Result<QueueItemId>;
    async fn dequeue(&self) -> Result<Option<QueueItem<T>>>;
    async fn get(&self, id: QueueItemId) -> Result<Option<QueueItem<T>>>;
    // ...
}
```

### WebSocket Message Flow
```
Player Client                 Engine                      DM Client
     |                           |                            |
     | PlayerAction             |                            |
     |------------------------->|                            |
     |                          | Queue to PlayerActionQueue |
     |                          | Worker processes           |
     |                          | LLM generates response     |
     |                          | Queue to DMApprovalQueue   |
     |                          |                            |
     |                          | ApprovalRequired           |
     |                          |--------------------------->|
     |                          |                            |
     |                          | ApprovalDecision           |
     |                          |<---------------------------|
     |                          |                            |
     | DialogueResponse         |                            |
     |<-------------------------|                            |
```

---

## Appendix B: Testing Commands

```bash
# Build and check Engine
cd Engine && cargo check

# Build and check Player
cd Player && cargo check

# Run Engine with tracing
cd Engine && RUST_LOG=debug cargo run

# Run Player (desktop)
cd Player && cargo run --features desktop

# Verify WebSocket connection
# Connect to ws://localhost:3001/ws
```

---

## Appendix C: Related Plan Documents

| Document | Contains |
|----------|----------|
| `plans/16-director-decision-queue.md` | Full Phase 16 spec with UI mockups |
| `plans/20-unified-generation-queue.md` | Full Phase 20 spec with architecture |
| `plans/18-comfyui-enhancements.md` | Phases 18A-18E with all tasks |
| `plans/19-queue-system.md` | Queue system architecture (COMPLETE) |
| `plans/ROADMAP.md` | Master tracking document |

---

## Appendix D: Known TODOs in websocket.rs

After fixes in this plan, the following TODOs will remain (not blockers):

1. **Line 149**: `character_name: None, // TODO: Load from character selection`
   - Waiting for character selection UI in Player

2. **Line 411**: `// TODO: Update directorial context and store in session`
   - Non-critical, directorial notes work without persistence

3. **Lines 534, 630**: `character_modifier = 0`
   - Requires Phase 14C character sheet UI integration

4. **Line 866**: `character_name: None`
   - Same as #1
