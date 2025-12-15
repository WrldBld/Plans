# Phase 20: Unified Generation Queue UI (Creator Mode)

> **Note**: This was originally "Phase 15" but was renamed to "Phase 20" to avoid conflict with "Phase 15: Routing & Navigation".
>
> **Clarification**: "Unified" refers to the **UI only** - a single panel showing tasks from both ComfyUI and LLM queues. The Engine queues remain completely separate and independent.

## Overview

WrldBldr uses AI generation for two distinct purposes:
1. **Image Generation** via ComfyUI workflows (character portraits, location backdrops)
2. **LLM Suggestions** via Ollama (names, descriptions, wants, backstories, etc.)

Currently these systems operate independently:
- Image generation uses a queue with WebSocket events for progress
- LLM suggestions are synchronous HTTP requests with inline loading spinners

This phase unifies both under a single Generation Queue system, providing consistent UX and better visibility into all pending AI work.

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

### Phase 15A: Engine Changes

- [ ] **15A.1** Create GenerationEvent enum
  - File: `Engine/src/domain/events/generation_events.rs` (new)
  - Add image and suggestion event variants
  - Serialize with `#[serde(tag = "type")]` for WebSocket

- [ ] **15A.2** Create SuggestionQueue service
  - File: `Engine/src/application/services/suggestion_queue.rs` (new)
  - Background worker for processing queued suggestions
  - Rate limiting to prevent Ollama overload
  - Retry logic for transient failures

- [ ] **15A.3** Update suggestion routes to queue instead of sync
  - File: `Engine/src/infrastructure/http/suggestion_routes.rs`
  - Return `{ request_id: String }` immediately
  - Enqueue request for async processing
  - Keep existing sync behavior as fallback option

- [ ] **15A.4** Broadcast generation events via WebSocket
  - File: `Engine/src/infrastructure/websocket.rs`
  - Subscribe to GenerationEvent channel
  - Forward events to appropriate session clients

### Phase 15B: Player Changes

- [ ] **15B.1** Extend GenerationState for suggestions
  - File: `Player/src/presentation/state/generation_state.rs`
  - Add `GenerationType` enum with Image/Suggestion variants
  - Add `SuggestionFieldType` for categorizing requests
  - Add `GenerationResult` enum for results

- [ ] **15B.2** Handle suggestion WebSocket events
  - File: `Player/src/infrastructure/websocket/handlers.rs`
  - Parse SuggestionQueued, SuggestionProgress, SuggestionComplete, SuggestionFailed
  - Update GenerationState accordingly

- [ ] **15B.3** Create unified GenerationQueue component
  - File: `Player/src/presentation/components/creator/generation_queue.rs` (new)
  - Shows all generation tasks in a single list
  - Filter tabs: All, Images, Suggestions
  - Expandable cards showing results
  - Progress indicators per task

- [ ] **15B.4** Update SuggestionButton to use queue
  - File: `Player/src/presentation/components/creator/suggestion_button.rs`
  - Add task to GenerationState when clicked
  - Show "queued" indicator instead of loading spinner
  - Disable button while task pending for same field

- [ ] **15B.5** Add suggestion result selection
  - File: `Player/src/presentation/components/creator/generation_queue.rs`
  - Clicking a suggestion result populates target field
  - Visual feedback on selection
  - Auto-collapse card after selection

### Phase 15C: Polish & Integration

- [ ] **15C.1** Queue visibility in Creator Mode header
  - File: `Player/src/presentation/views/creator_view.rs`
  - Show badge with pending task count
  - Click to expand queue panel

- [ ] **15C.2** Toast notifications for completion
  - Show toast when suggestion completes
  - "3 name suggestions ready" with link to queue

- [ ] **15C.3** Persist queue state across navigation
  - Queue survives panel switches within Creator Mode
  - Clear queue on exit from Creator Mode

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
- **Cancel requests**: Allow users to cancel pending suggestions
- **Suggestion history**: Remember past suggestions for re-use
