# Critical Oversight: LLM Suggestions Not Using Queue System

**Date**: 2025-12-15  
**Status**: ✅ **RESOLVED** (as of 2025-12-15, see Engine commit `249ed62` and Player commit `ce6c89f`)  
**Priority**: (Historical) Was **HIGH** – originally blocked Phase 20 until fixed

> This document is now a **postmortem / design record**. The fixes described here have been implemented in Engine and Player; see Phase 20 section in `ROADMAP.md` and `20-unified-generation-queue.md` for current state.

---

## Problem Statement

**LLM suggestion requests are bypassing the queue system entirely**, even though:
1. The `LLMReasoningQueue` infrastructure exists and can handle suggestions
2. `LLMRequestItem` has a `Suggestion` variant
3. The plan (Phase 20) explicitly calls for queued suggestions
4. But suggestions are currently **synchronous HTTP requests**

---

## Current State Analysis

### What EXISTS (Infrastructure Ready)

✅ **LLMReasoningQueue** - Queue system operational  
✅ **LLMRequestItem** - Has `LLMRequestType::Suggestion { field_type, entity_id }` variant  
✅ **LLMQueueService** - Has placeholder for suggestion handling (line 338-342)  
✅ **Queue workers** - Background processing infrastructure ready  
✅ **WebSocket infrastructure** - Event broadcasting ready  

### What's MISSING (The Gap) – Original State (Now Fixed)

❌ **HTTP routes bypass queue** - `suggestion_routes.rs` called `SuggestionService` directly  
❌ **No enqueue logic** - Suggestions never entered `LLMReasoningQueue`  
❌ **LLMQueueService TODO** - Suggestion handler just logged and completed (didn't process)  
❌ **No WebSocket events** - No `SuggestionQueued`, `SuggestionProgress`, `SuggestionComplete` events  
❌ **No unified queue visibility** - Suggestions didn't appear in GenerationQueuePanel  

---

## Evidence from Codebase (Historical)

### Engine: Suggestion Routes (Original - WRONG)

```rust
// Engine/src/infrastructure/http/suggestion_routes.rs

pub async fn suggest_character_names(
    State(state): State<Arc<AppState>>,
    Json(req): Json<SuggestionRequestDto>,
) -> Result<Json<SuggestionResponseDto>, (StatusCode, String)> {
    // ❌ DIRECT CALL - bypasses queue
    let service = SuggestionService::new(state.llm_client.clone());
    let suggestions = service.suggest_character_names(&context).await?;
    Ok(Json(SuggestionResponseDto { suggestions }))
}
```

**Problem**: This is synchronous - blocks HTTP request until LLM responds.

### Engine: LLMQueueService (Original - INCOMPLETE)

```rust
// Engine/src/application/services/llm_queue_service.rs:338-342

LLMRequestType::Suggestion { .. } => {
    // Send suggestion result via WebSocket (Phase 15)
    // No DM approval needed
    tracing::info!("Suggestion request - will be handled in Phase 15");
    let _ = queue_clone.complete(item_id).await;  // ❌ Just completes, doesn't process!
}
```

**Problem**: Handler exists but doesn't actually call `SuggestionService` or send results.

### Player: SuggestionButton (Original - SYNCHRONOUS)

```rust
// Player/src/presentation/components/creator/suggestion_button.rs:58-97

spawn(async move {
    loading.set(true);
    // ❌ SYNCHRONOUS HTTP CALL
    let result = match suggestion_type {
        SuggestionType::CharacterName => service.suggest_character_name(&context).await,
        // ... blocks until response
    };
    // Results appear in inline dropdown, not queue
});
```

**Problem**: Makes blocking HTTP call, shows inline spinner, not queued.

---

## Resolution Summary (Implemented)

The following fixes have been implemented in the codebase:

- **Engine**:
  - Updated `suggestion_routes.rs` so `/api/suggest` enqueues `LLMRequestItem { request_type: LLMRequestType::Suggestion { .. } }` into `LLMReasoningQueue` and returns `{ request_id, status: "queued" }` immediately.
  - Extended `LLMQueueService` to process `LLMRequestType::Suggestion`, call `SuggestionService`, and emit `GenerationEvent::SuggestionQueued`, `SuggestionProgress`, `SuggestionComplete`, and `SuggestionFailed`.
  - Mapped `GenerationEvent::Suggestion*` to `ServerMessage::Suggestion*` in the generation event broadcaster in `main.rs`.

- **Player**:
  - Extended `GenerationState` to track `SuggestionTask` and `SuggestionStatus` alongside image `GenerationBatch`.
  - Added `ServerMessage::Suggestion*` variants and handled them in `session_message_handler.rs` to update `GenerationState`.
  - Updated `SuggestionButton` to call a new `SuggestionService::enqueue_suggestion()` method and rely on queue/WebSocket events rather than synchronous HTTP.
  - Updated `GenerationQueuePanel` to display both image batches and suggestion tasks in a unified queue UI.
  - UX requirement: Clicking **“View”** on a suggestion task must either (preferred) navigate back to the originating form field (character/location/item/map/event, etc.) and show the standard suggestion UI there, or (fallback) open a contextual modal that clearly labels the entity + field being edited and allows applying a suggestion without losing the original options.

The remainder of this document describes the original problem and design; it no longer reflects the current “issue” status but remains accurate as a design rationale.

---

## Required Changes (Historical Plan)

> **Note**: The required changes described in this section have been implemented and integrated. This section remains for historical purposes.

### 1. Engine: Route Suggestions Through Queue

**File**: `Engine/src/infrastructure/http/suggestion_routes.rs`

**Change**: Instead of calling `SuggestionService` directly, enqueue to `LLMReasoningQueue`

```rust
pub async fn suggest_character_names(
    State(state): State<Arc<AppState>>,
    Json(req): Json<SuggestionRequestDto>,
) -> Result<Json<SuggestionQueuedResponse>, (StatusCode, String)> {
    // ✅ ENQUEUE instead of direct call
    let request_id = uuid::Uuid::new_v4().to_string();
    
    let llm_request = LLMRequestItem {
        request_type: LLMRequestType::Suggestion {
            field_type: "character_name".to_string(),
            entity_id: None,
        },
        session_id: None, // Creator mode, no session
        prompt: build_suggestion_prompt(&req.context, "character_name"),
        callback_id: request_id.clone(),
    };
    
    state.llm_queue_service.enqueue(llm_request).await?;
    
    // Return immediately with request_id
    Ok(Json(SuggestionQueuedResponse {
        request_id,
        status: "queued".to_string(),
    }))
}
```

### 2. Engine: Implement Suggestion Processing

**File**: `Engine/src/application/services/llm_queue_service.rs`

**Change**: Actually process suggestions and send WebSocket events

```rust
LLMRequestType::Suggestion { field_type, entity_id } => {
    // ✅ ACTUALLY PROCESS THE SUGGESTION
    let suggestion_service = SuggestionService::new(llm_client.clone());
    
    // Build context from prompt
    let context = extract_context_from_prompt(&item.prompt);
    
    // Call appropriate suggestion method
    let suggestions = match field_type.as_str() {
        "character_name" => suggestion_service.suggest_character_names(&context).await,
        "character_description" => suggestion_service.suggest_character_description(&context).await,
        // ... etc
    };
    
    match suggestions {
        Ok(results) => {
            // ✅ Send WebSocket event
            send_suggestion_complete_event(
                &item.callback_id,
                &field_type,
                results,
            ).await;
            
            queue_clone.complete(item_id).await?;
        }
        Err(e) => {
            send_suggestion_failed_event(&item.callback_id, &e.to_string()).await;
            queue_clone.fail(item_id, &e.to_string()).await?;
        }
    }
}
```

### 3. Engine: Add WebSocket Events

**File**: `Engine/src/domain/events/generation_events.rs` (or create if doesn't exist)

**Add**:
```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum GenerationEvent {
    // Existing image events...
    
    // New suggestion events
    SuggestionQueued {
        request_id: String,
        field_type: String,
        entity_id: Option<String>,
    },
    SuggestionProgress {
        request_id: String,
        status: String,
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

### 4. Player: Update SuggestionButton to Use Queue

**File**: `Player/src/presentation/components/creator/suggestion_button.rs`

**Change**: Enqueue instead of synchronous call

```rust
let fetch_suggestions = move |_| {
    spawn(async move {
        // ✅ ENQUEUE instead of direct call
        let request_id = svc.enqueue_suggestion(suggestion_type, context).await?;
        
        // Add to GenerationState
        generation_state.add_suggestion_task(request_id, suggestion_type, context);
        
        // Show "queued" indicator
        // Wait for WebSocket event with results
    });
};
```

### 5. Player: Handle Suggestion WebSocket Events

**File**: `Player/src/presentation/handlers/session_message_handler.rs`

**Add**:
```rust
ServerMessage::SuggestionComplete { request_id, suggestions } => {
    generation_state.suggestion_complete(&request_id, suggestions);
}
```

### 6. Player: Extend GenerationState for Suggestions

**File**: `Player/src/presentation/state/generation_state.rs`

**Add**:
```rust
pub enum GenerationTask {
    Image { /* existing */ },
    Suggestion {
        request_id: String,
        field_type: SuggestionFieldType,
        status: SuggestionStatus,
        results: Vec<String>,
    },
}
```

---

## Implementation Order

1. **Engine: Add WebSocket events** (GenerationEvent enum)
2. **Engine: Update suggestion routes** (enqueue instead of sync)
3. **Engine: Implement suggestion processing** (LLMQueueService handler)
4. **Engine: Wire WebSocket broadcasting** (send events to sessions)
5. **Player: Extend GenerationState** (add suggestion tasks)
6. **Player: Handle WebSocket events** (update state on events)
7. **Player: Update SuggestionButton** (enqueue instead of sync)
8. **Player: Update GenerationQueuePanel** (show suggestions)

---

## Impact Assessment

### Before (Current State)
- ❌ Suggestions block HTTP requests (bad UX)
- ❌ No visibility into pending suggestions
- ❌ No unified queue (images and suggestions separate)
- ❌ Can't see suggestion progress
- ❌ No retry/cancel capability

### After (Fixed State)
- ✅ Suggestions are async (non-blocking)
- ✅ Unified queue shows all generation work
- ✅ Real-time progress visibility
- ✅ Consistent UX with image generation
- ✅ Foundation for Phase 20 completion

---

## Dependencies

- ✅ LLMReasoningQueue infrastructure (exists)
- ✅ WebSocket infrastructure (exists)
- ✅ GenerationState for images (exists)
- ⚠️ Need to extend GenerationState for suggestions
- ⚠️ Need to add WebSocket event types
- ⚠️ Need to wire suggestion processing

---

## Testing Checklist

- [ ] Suggestion request returns immediately with request_id
- [ ] Request appears in queue with "queued" status
- [ ] WebSocket event sent when processing starts
- [ ] WebSocket event sent when complete with results
- [ ] Results appear in unified queue panel
- [ ] Can select suggestion to populate field
- [ ] Error handling works (failed suggestions show error)
- [ ] Multiple concurrent suggestions work
- [ ] Queue persists across navigation

---

## Related Documents

- `plans/20-unified-generation-queue.md` - Original plan
- `plans/unified_generation_queue_analysis.md` - Analysis showing partial implementation
- `Engine/src/application/services/llm_queue_service.rs` - Queue service with TODO
- `Engine/src/infrastructure/http/suggestion_routes.rs` - Routes that need updating

---

## Conclusion

This is a **critical architectural gap**. The infrastructure exists but isn't being used. Fixing this is a prerequisite for Phase 20 completion and provides immediate UX benefits (non-blocking suggestions, unified visibility).

**Estimated Effort**: 4-6 hours
**Priority**: **HIGH** - Should be fixed before Phase 20 UI work
