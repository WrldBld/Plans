# Unified Generation Queue: Plans vs Implementation Analysis

**Date**: 2025-12-14  
**Status**: **PARTIALLY IMPLEMENTED** (Images only, LLM suggestions not unified)

## Executive Summary

The Unified Generation Queue (Phase 15) is **partially implemented**. Image generation (ComfyUI) uses a queue system with WebSocket events, but LLM suggestions remain **synchronous HTTP requests** with inline loading spinners. The plans call for unifying both into a single queue system, but this unification has not been completed.

**Implementation Coverage**: ~50% - Image generation queued, LLM suggestions not unified

---

## Detailed Comparison

### Planned Features (from plans/15-unified-generation-queue.md)

**Core Goal**: Unify image generation and LLM suggestions into a single queue system

**Acceptance Criteria**:
- ✅ Clicking "Suggest" on any field adds an entry to the generation queue
- ❌ Queue shows suggestion requests alongside image generation requests
- ❌ Visual distinction between request types (icons, colors)
- ✅ Users can see real-time progress/status of all generation (images only)
- ✅ Results appear in queue when ready (images only)
- ❌ Suggestion results can be selected to populate the target field (from queue)
- ❌ Failed requests show error messages in queue
- ✅ Queue persists across panel navigation within Creator Mode (images only)

---

## Current Implementation Status

### Image Generation (ComfyUI) - ✅ IMPLEMENTED

**Status**: ✅ **FULLY IMPLEMENTED**

**Components**:
- ✅ `Player/src/presentation/state/generation_state.rs` - GenerationState for batch tracking
- ✅ `Player/src/presentation/components/creator/generation_queue.rs` - GenerationQueuePanel component
- ✅ WebSocket events for generation progress (GenerationQueued, GenerationProgress, GenerationComplete, GenerationFailed)
- ✅ Queue UI showing batches with status (Queued, Generating, Ready, Failed)
- ✅ Selection modal for completed batches

**Current Behavior**:
- Image generation requests are queued
- WebSocket events provide real-time progress updates
- Queue panel shows all active batches
- Users can select from completed batches

**Evidence**:
- `GenerationState` exists and tracks image batches
- `GenerationQueuePanel` displays image generation queue
- WebSocket handlers process generation events

### LLM Suggestions - ❌ NOT UNIFIED

**Status**: ❌ **NOT UNIFIED WITH QUEUE**

**Current Implementation**:
- ✅ `Player/src/presentation/components/creator/suggestion_button.rs` - SuggestionButton component
- ✅ `Engine/src/infrastructure/http/suggestion_routes.rs` - Synchronous HTTP endpoints
- ✅ `Engine/src/application/services/suggestion_service.rs` - SuggestionService
- ❌ **No queue system** - suggestions are synchronous HTTP requests
- ❌ **No WebSocket events** - no SuggestionQueued, SuggestionProgress, SuggestionComplete events
- ❌ **Inline loading spinners** - not integrated with generation queue

**Current Behavior**:
- Clicking "Suggest" makes **synchronous HTTP request**
- Button shows loading spinner while waiting
- Results appear in **inline dropdown** (not queue)
- No visibility into pending suggestions
- No way to see suggestion progress
- No unified queue showing both images and suggestions

**Evidence**:
- `SuggestionButton` uses `suggestion_service.suggest_*()` which are synchronous HTTP calls
- No `SuggestionQueue` service exists in Engine
- No suggestion-related WebSocket events
- Suggestions don't appear in `GenerationQueuePanel`

---

## Architecture Comparison

### Planned Architecture

```
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

### Current Architecture

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
```

**Gap**: LLM suggestions are **not unified** with the queue system.

---

## Domain Model Comparison

### Planned Domain Model

```rust
// Player/src/presentation/state/generation_state.rs

#[derive(Clone, Debug)]
pub enum GenerationType {
    Image {
        batch_id: String,
        entity_type: String,
        entity_id: String,
        asset_type: String,
    },
    Suggestion {
        request_id: String,
        field_type: String,
        entity_id: Option<String>,
        suggestion_type: SuggestionType,
    },
}

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

### Current Implementation

**Status**: ❌ **PARTIALLY IMPLEMENTED**

**What Exists**:
- ✅ `GenerationState` with `GenerationBatch` for images
- ✅ `BatchStatus` enum (Queued, Generating, Ready, Failed)
- ❌ **No `GenerationType` enum** - only tracks images
- ❌ **No suggestion events** in GenerationState
- ❌ **No `SuggestionRequest` type** in GenerationState

**What's Missing**:
- `GenerationType` enum to distinguish images vs suggestions
- Suggestion request tracking in GenerationState
- Suggestion-related WebSocket event handlers

---

## Backend Comparison

### Planned Backend

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

### Current Backend

**Status**: ❌ **NOT IMPLEMENTED**

**What Exists**:
- ✅ `Engine/src/application/services/suggestion_service.rs` - Synchronous suggestion service
- ✅ `Engine/src/infrastructure/http/suggestion_routes.rs` - HTTP endpoints for suggestions
- ❌ **No `SuggestionQueue` service** - suggestions are processed synchronously
- ❌ **No async queue processing** - each request blocks until complete
- ❌ **No WebSocket events** for suggestions

**What's Missing**:
- `SuggestionQueue` service for async processing
- WebSocket event broadcasting for suggestions
- Queue management (enqueue, process, retry)

---

## API Contract Comparison

### Planned API

**WebSocket Events**:
```rust
// Engine → Player
SuggestionQueued {
    request_id: String,
    field_type: String,
    entity_id: Option<String>,
}
SuggestionProgress {
    request_id: String,
    status: String,
}
SuggestionComplete {
    request_id: String,
    suggestions: Vec<String>,
}
SuggestionFailed {
    request_id: String,
    error: String,
}
```

**HTTP Endpoints** (Optional, for backward compatibility):
```
POST /api/suggestions/queue
GET /api/suggestions/queue/{request_id}
```

### Current API

**Status**: ❌ **SYNCHRONOUS HTTP ONLY**

**What Exists**:
- ✅ HTTP endpoints: `POST /api/suggestions/character-name`, etc.
- ✅ Synchronous responses with suggestions array
- ❌ **No WebSocket events** for suggestions
- ❌ **No queue endpoints**

**Current Flow**:
1. Frontend makes HTTP POST request
2. Backend processes synchronously (blocks)
3. Backend returns suggestions array
4. Frontend shows dropdown with results

**Planned Flow** (not implemented):
1. Frontend makes HTTP POST request (or WebSocket message)
2. Backend queues request, returns `request_id` immediately
3. Backend processes asynchronously
4. Backend sends WebSocket events (Queued, Progress, Complete)
5. Frontend shows request in unified queue
6. Frontend allows selection from queue when ready

---

## UI Comparison

### Planned UI

**Unified Generation Queue Panel**:
- Shows **both** image generation and LLM suggestions
- Visual distinction: icons/colors for different types
- Filter tabs: All, Images, Suggestions
- Expandable cards showing results
- Progress indicators per task
- Selection from queue when ready

### Current UI

**Status**: ❌ **PARTIALLY IMPLEMENTED**

**What Exists**:
- ✅ `GenerationQueuePanel` component
- ✅ Shows image generation batches
- ✅ Status indicators (Queued, Generating, Ready, Failed)
- ✅ Selection button for ready batches
- ❌ **Does NOT show LLM suggestions**
- ❌ **No filter tabs** (All, Images, Suggestions)
- ❌ **No visual distinction** for different types (only images)

**SuggestionButton Current Behavior**:
- Shows inline loading spinner
- Displays dropdown with results directly
- **Not integrated with queue**
- No visibility in unified queue panel

---

## Impact Assessment

### High Priority Gaps

1. **LLM Suggestions Not Queued**
   - **Impact**: Inconsistent UX between image generation and suggestions
   - **Severity**: HIGH
   - **User Experience**: Users can't see pending suggestions, no progress visibility

2. **No Unified Queue UI**
   - **Impact**: Two separate systems (queue for images, inline for suggestions)
   - **Severity**: HIGH
   - **User Experience**: Confusing, inconsistent workflow

3. **Synchronous Blocking**
   - **Impact**: UI freezes while waiting for suggestions
   - **Severity**: MEDIUM
   - **User Experience**: Poor responsiveness, can't continue working while suggestions process

### Medium Priority Gaps

4. **No Suggestion Progress Events**
   - **Impact**: No visibility into suggestion processing status
   - **Severity**: MEDIUM

5. **No Error Handling in Queue**
   - **Impact**: Failed suggestions don't appear in queue
   - **Severity**: MEDIUM

---

## Recommendations

### Immediate Actions (High Priority)

1. **Implement Backend Suggestion Queue**
   - Create `SuggestionQueue` service
   - Modify suggestion routes to queue instead of sync
   - Add WebSocket event broadcasting for suggestions
   - Return `request_id` immediately, process async

2. **Extend GenerationState for Suggestions**
   - Add `GenerationType` enum (Image, Suggestion)
   - Add suggestion request tracking
   - Handle suggestion WebSocket events
   - Update `GenerationBatch` or create `SuggestionRequest` type

3. **Update SuggestionButton to Use Queue**
   - Remove synchronous HTTP call
   - Add request to GenerationState queue
   - Show "queued" indicator instead of loading spinner
   - Disable button while request pending

4. **Unify Queue UI**
   - Update `GenerationQueuePanel` to show both types
   - Add filter tabs (All, Images, Suggestions)
   - Add visual distinction (icons, colors)
   - Allow selection of suggestion results from queue

### Short-term (Medium Priority)

5. **Add Progress Events**
   - Send `SuggestionProgress` events during processing
   - Update queue UI to show progress

6. **Error Handling**
   - Show failed suggestions in queue
   - Allow retry from queue

7. **Queue Persistence**
   - Ensure queue persists across navigation
   - Handle reconnection scenarios

---

## Conclusion

The Unified Generation Queue is **partially implemented**. Image generation uses a proper queue system with WebSocket events, but LLM suggestions remain synchronous HTTP requests with inline loading spinners. The unification called for in Phase 15 has not been completed.

**Estimated Effort to Complete**: 1-2 weeks
- Backend queue: 3-4 days
- Frontend integration: 3-4 days
- UI unification: 2-3 days

**Priority**: **HIGH** - This is a UX consistency issue that affects the core Creator Mode workflow.
