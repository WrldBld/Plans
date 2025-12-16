# Phase 20: Unified Generation Queue UI (Creator Mode)

> **Note**: This was originally "Phase 15" but was renamed to "Phase 20" to avoid conflict with "Phase 15: Routing & Navigation".
>
> **Clarification**: "Unified" refers to the **UI only** - a single panel showing tasks from both ComfyUI and LLM queues. The Engine queues remain completely separate and independent.

## Overview

WrldBldr uses AI generation for two distinct purposes:
1. **Image Generation** via ComfyUI workflows (character portraits, location backdrops)
2. **LLM Suggestions** via Ollama (names, descriptions, wants, backstories, etc.)

**Status**: [x] **COMPLETE**

This phase unifies both under a single Generation Queue system, providing consistent UX and better visibility into all pending AI work. All features have been implemented including cancel/retry, sorting, navigation, and error handling polish.

---

## Relationship with Decision Queue

WrldBldr has **two distinct queue systems** for different purposes:

| Aspect | Generation Queue (This Phase) | Decision Queue ([Phase 16](./16-director-decision-queue.md)) |
|--------|------------------------------|--------------------------------------------------------------|
| **Location** | Creator Mode | Director Mode |
| **Purpose** | Create world content | Control gameplay decisions |
| **Contents** | Images, text suggestions | NPC responses, tool usage, challenges |
| **Timing** | Asynchronous (seconds to minutes) | Real-time (players waiting) |
| **Urgency** | Low (continue building) | High (gameplay blocked) |
| **Results** | Assets/suggestions for forms | Actions affecting players |

**Key Distinction**:
- **Generation Queue**: "Help me create content for my world" (building phase)
- **Decision Queue**: "Approve what the AI wants to do in the story" (gameplay phase)

---

## User Story

**As a** Dungeon Master using Creator Mode
**I want to** see all AI generation requests (images AND suggestions) in a unified queue
**So that** I have visibility into pending work and a consistent experience for all AI-assisted creation

### Acceptance Criteria

- Clicking "Suggest" on any field adds an entry to the generation queue
- Queue shows suggestion requests alongside image generation requests
- Visual distinction between request types (icons, colors)
- Users can see real-time progress/status of all generation
- Results appear in queue when ready
- Suggestion results can be selected to populate the target field
- Failed requests show error messages in queue
- Queue persists across panel navigation within Creator Mode
- After a page reload, the unified generation queue is **reconstructed from the Engine** via a read-only endpoint (e.g. `/api/generation/queue`), so in-flight and recently-completed items are visible again.
- Clicking **“View”** on a suggestion task does **not** lose context:
  - **Preferred**: Navigates back to the originating form (character/location/item/map/event, etc.) and focuses the target field, showing the normal suggestion UI for that field.
  - **Fallback (if navigation is hard)**: Opens a modal that clearly labels the target entity + field (e.g. “Character Name for Aria Windwalker”) and allows applying a suggestion from there.

---

## Architecture

```
Current State:
==============

Image Generation                    LLM Suggestions
       │                                   │
   GenerationState                    HTTP POST
       │                                   │
   WebSocket events                   Inline spinner
       │                                   │
   Queue Component                    Direct response
       │                                   │
   Select from batch                  Populate field


Target State:
=============

                 All Generation Requests
                          │
                    GenerationState
                          │
              ┌───────────┴───────────┐
              │                       │
         Image Tasks             Suggestion Tasks
              │                       │
         Queue to Engine         Queue to Engine
              │                       │
         ComfyUI Worker          LLM Worker
              │                       │
         WebSocket Events        WebSocket Events
              │                       │
              └───────────┬───────────┘
                          │
                   Unified Queue UI
                          │
                   Select Results
```

---

## Technical Design

### Player: Extended GenerationState

```rust
// Player/src/presentation/state/generation_state.rs

#[derive(Clone, Debug)]
pub enum GenerationType {
    Image {
        entity_type: String,  // "character", "location", "item"
        entity_id: String,
        workflow_slot: String,
    },
    Suggestion {
        field_type: SuggestionFieldType,
        target_field: String,  // Field to populate when selected
        context: SuggestionContext,
    },
}

#[derive(Clone, Debug)]
pub enum SuggestionFieldType {
    CharacterName,
    CharacterDescription,
    CharacterWants,
    CharacterFears,
    CharacterBackstory,
    LocationName,
    LocationDescription,
    LocationAtmosphere,
    LocationFeatures,
    LocationSecrets,
}

#[derive(Clone, Debug)]
pub struct GenerationTask {
    pub id: String,           // UUID for tracking
    pub generation_type: GenerationType,
    pub status: GenerationStatus,
    pub results: Vec<GenerationResult>,
    pub error: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Clone, Debug)]
pub enum GenerationResult {
    Image { url: String, thumbnail_url: Option<String> },
    Suggestion { text: String },
}
```

### Engine: WebSocket Events

```rust
// Engine/src/domain/events/generation_events.rs

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum GenerationEvent {
    // Existing image events
    GenerationQueued { batch_id: String, entity_type: String, entity_id: String },
    GenerationProgress { batch_id: String, current: u32, total: u32 },
    GenerationComplete { batch_id: String, asset_ids: Vec<String> },
    GenerationFailed { batch_id: String, error: String },

    // New suggestion events
    SuggestionQueued {
        request_id: String,
        field_type: String,
        entity_id: Option<String>,
    },
    SuggestionProgress {
        request_id: String,
        status: String,  // "processing", "generating", etc.
    },
    SuggestionComplete {
        request_id: String,
        suggestions: Vec<String>,
    },
    SuggestionFailed {
        request_id: String,
        error: String,
    },
}
```

### Engine: Async Suggestion Queue

```rust
// Engine/src/application/services/suggestion_queue.rs

pub struct SuggestionQueue {
    pending: VecDeque<SuggestionRequest>,
    processing: Option<SuggestionRequest>,
    ollama_client: Arc<OllamaClient>,
    event_sender: broadcast::Sender<GenerationEvent>,
}

pub struct SuggestionRequest {
    pub id: String,
    pub session_id: String,
    pub field_type: SuggestionFieldType,
    pub context: SuggestionContext,
    pub created_at: DateTime<Utc>,
}

impl SuggestionQueue {
    pub async fn enqueue(&self, request: SuggestionRequest) -> String {
        // Send SuggestionQueued event
        // Add to queue
        // Return request_id
    }

    pub async fn process_next(&mut self) {
        // Take next from queue
        // Send SuggestionProgress
        // Call Ollama
        // Send SuggestionComplete or SuggestionFailed
    }
}
```

---

## Implementation Tasks

### Phase 20A: Engine Changes

- [x] **20A.1** Create GenerationEvent enum ✅
  - Implemented via GenerationEvent in generation_service.rs
  - Includes SuggestionQueued, SuggestionProgress, SuggestionComplete, SuggestionFailed

- [x] **20A.2** Queue suggestions via LLMQueueService ✅
  - Suggestions routed through LLMReasoningQueue
  - Background worker processes with concurrency control
  - Events broadcast via GenerationEvent channel

- [x] **20A.3** Update suggestion routes to queue ✅
  - `/api/suggest` endpoint queues requests
  - Returns `{ request_id, status }` immediately
  - Async processing via LLMQueueService

- [x] **20A.4** Broadcast generation events via WebSocket ✅
  - GenerationEvent channel integrated
  - WebSocketEventSubscriber forwards to clients
  - Real-time updates for all generation tasks

- [x] **20A.5** Cancel/Retry endpoints ✅
  - `/api/suggest/{request_id}/cancel` - Cancel pending suggestions
  - `/api/assets/batch/{batch_id}/retry` - Retry failed batches
  - Cancel method in LLMQueueService

### Phase 20B: Player Changes

- [x] **20B.1** Extend GenerationState for suggestions ✅
  - SuggestionTask struct with context storage
  - SuggestionStatus enum (Queued, Processing, Ready, Failed)
  - Context stored for retry functionality

- [x] **20B.2** Handle suggestion WebSocket events ✅
  - SessionMessageHandler processes Suggestion* events
  - Updates GenerationState accordingly
  - Real-time status updates

- [x] **20B.3** Create unified GenerationQueue component ✅
  - GenerationQueuePanel shows all tasks
  - Filter tabs: All, Images, Suggestions
  - Status indicators and progress bars
  - Expandable error details

- [x] **20B.4** Update SuggestionButton to use queue ✅
  - Enqueues via SuggestionService.enqueue_suggestion()
  - Shows "Queued..." indicator
  - Stores context for retry

- [x] **20B.5** Add suggestion result selection ✅
  - SuggestionViewModal for viewing results
  - Click to view and apply suggestions
  - Navigation to originating form

- [x] **20B.6** Cancel/Retry functionality ✅
  - Cancel buttons for batches and suggestions
  - Retry buttons for failed items
  - Context stored for suggestion retry

### Phase 20C: Polish & Integration

- [x] **20C.1** Sorting options ✅
  - Sort dropdown: Newest First, Oldest First, By Status, By Type
  - Status priority sorting (active items first)
  - Type-based sorting

- [x] **20C.2** Navigation to originating form ✅
  - "Select" button navigates to entity
  - "View" button for suggestions navigates to entity
  - Entity selection via callback

- [x] **20C.3** Error handling polish ✅
  - Enhanced error display with icons
  - Expandable error details
  - Monospace font for error messages
  - Better visual hierarchy

- [x] **20C.4** Persist queue state across navigation and reloads ✅
  - Queue survives panel switches within Creator Mode
  - Creator Mode hydrates `GenerationState` from Engine snapshot (`GET /api/generation/queue`)
  - Local client storage keeps state between navigations

### Phase 15D: Read-State Persistence (Per-User, Cross-Device)

- [x] **15D.1** Engine-side read-state repository
  - `GenerationReadStatePort` with SQLite-backed implementation
  - Table `generation_read_state(user_id, entity_type, item_id, read_at)`
- [x] **15D.2** Read-state HTTP API
  - Extend `GET /api/generation/queue?user_id={id}` to include `is_read` flags on batches and suggestions
  - Add `POST /api/generation/read-state` to mark batches and suggestions as read for a given `user_id`
- [x] **15D.3** Player integration
  - `hydrate_generation_queue` passes `SessionState.user_id` to `/api/generation/queue`
  - `GenerationQueuePanel` posts to `/api/generation/read-state` when items are viewed/selected
  - Local storage remains a secondary cache; Engine is source of truth for cross-device behavior

---

## API Changes

### New Endpoints

```
POST /api/suggestions/queue
Request: { field_type: string, context: object }
Response: { request_id: string }

GET /api/suggestions/queue/{request_id}
Response: { status: string, suggestions?: string[], error?: string }
```

### Modified Endpoints

Existing suggestion endpoints (`/api/suggest/character/name`, etc.) will:
1. Accept optional `?async=true` query param
2. If async: return `{ request_id }` and queue for processing
3. If sync (default for backwards compat): behave as before

---

## Dependencies

- WebSocket infrastructure (complete)
- OllamaClient (complete)
- GenerationState for images (complete)
- SuggestionButton component (complete)

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Ollama queue backup | High latency for users | Rate limiting, queue depth limits, status visibility |
| WebSocket disconnection | Lost events | Reconnect logic, fetch queue state on reconnect |
| UI complexity | Confusing UX | Clear visual hierarchy, good defaults, collapsible |

---

## Future Enhancements

- **Batch suggestions**: Generate multiple field suggestions in one request
- **Priority queue**: Allow users to prioritize certain requests
- **Suggestion history**: Remember past suggestions for re-use
- **Entity pre-selection**: Automatically select entity when navigating from queue
