# Phase 18: ComfyUI Integration Enhancements

## Overview

This document addresses gaps in WrldBldr's ComfyUI integration, covering resilience, event wiring, style references, Director Mode generation, and batch management.

**Current State Issues:**
- GenerationService emits events but they're NOT broadcast via WebSocket
- Player's session_service has TODO: "Update GenerationState when passed to this function"
- GenerationQueuePanel only shows "Failed" label, not error message
- No retry logic, no circuit breaker, no health monitoring
- Style references not implemented (deferred from Phase 11)

---

## User Decisions

- **Health Monitoring**: Hybrid - Check before queue + reconnect with backoff on failures
- **Retry Strategy**: Configurable - Default 3 retries (5s, 15s, 45s exponential), user configurable in settings
- **Style References**: Both IPAdapter and prompt injection. User can manually set reference input if auto-detect fails.
- **Director Mode Generation**: Full modal - same generation modal as Creator Mode, accessible from Director view

---

## Phase Dependencies

```
Phase 18A: ComfyUI Resilience (Backend)
    │
    └──> Phase 18B: Generation Event Wiring (Full Stack) [CRITICAL]
              │
              ├──> Phase 18C: Style Reference System (Full Stack)
              │
              ├──> Phase 18D: Director Mode Quick Generate (Frontend)
              │
              └──> Phase 18E: Batch Management UI (Frontend)
```

**18A** is foundational - adds health monitoring and retry logic
**18B** is CRITICAL - wires events from Engine to Player (currently disconnected with TODOs)
**18C-18E** can be done in parallel after 18B

---

## Phase 18A: ComfyUI Resilience

### User Stories

**US-18A.1**: Health check before queuing
**As the** system
**I want to** check ComfyUI health before queuing jobs
**So that** I can fail fast and inform users when ComfyUI is unavailable

**Acceptance Criteria:**
- Health check called before `queue_prompt()`
- If unhealthy, return `ComfyUIError::ServiceUnavailable` immediately
- Health check result cached for 5 seconds to avoid excessive checks
- Log health check failures at WARN level

---

**US-18A.2**: Retry with exponential backoff
**As the** system
**I want to** retry failed ComfyUI requests with exponential backoff
**So that** transient failures don't cause permanent failures

**Acceptance Criteria:**
- Default: 3 retries with 5s, 15s, 45s delays (exponential: base * 3^attempt)
- Each retry attempt logged at INFO level
- Final failure logged at ERROR level
- Retry only on transient errors (HTTP 5xx, timeout, connection refused)
- Do NOT retry on 4xx errors (bad workflow, invalid params)

---

**US-18A.3**: Configurable retry settings
**As a** DM in settings
**I want to** configure ComfyUI retry behavior
**So that** I can tune for my system

**Acceptance Criteria:**
- Settings: max_retries (1-5, default 3), base_delay_seconds (1-30, default 5)
- Settings stored in Engine config
- Changes apply immediately without restart

---

**US-18A.4**: Circuit breaker
**As the** system
**I want** a circuit breaker to stop hammering a dead ComfyUI
**So that** the system remains responsive

**Acceptance Criteria:**
- After 5 consecutive failures, circuit opens
- Open circuit rejects requests immediately for 60 seconds
- After 60 seconds, circuit half-opens (allow 1 probe request)
- On probe success, circuit closes; on probe failure, stays open

---

**US-18A.5**: Configurable timeouts
**As the** system
**I want** configurable timeouts for ComfyUI operations
**So that** slow generations don't block the queue

**Acceptance Criteria:**
- health_check timeout: 5 seconds (non-configurable)
- queue_prompt timeout: 30 seconds (configurable)
- get_history timeout: 10 seconds (configurable)
- get_image timeout: 60 seconds (configurable)

---

### Domain Model

```rust
// Engine/src/domain/value_objects/comfyui_config.rs (NEW)

/// Configuration for ComfyUI retry behavior
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ComfyUIConfig {
    pub max_retries: u8,              // 1-5, default 3
    pub base_delay_seconds: u8,       // 1-30, default 5
    pub queue_timeout_seconds: u16,   // default 30
    pub history_timeout_seconds: u16, // default 10
    pub image_timeout_seconds: u16,   // default 60
}

impl Default for ComfyUIConfig {
    fn default() -> Self {
        Self {
            max_retries: 3,
            base_delay_seconds: 5,
            queue_timeout_seconds: 30,
            history_timeout_seconds: 10,
            image_timeout_seconds: 60,
        }
    }
}
```

```rust
// Engine/src/infrastructure/comfyui.rs (MODIFY)

/// Extended error types
#[derive(Debug, thiserror::Error)]
pub enum ComfyUIError {
    #[error("HTTP request failed: {0}")]
    HttpError(#[from] reqwest::Error),
    #[error("API error: {0}")]
    ApiError(String),
    #[error("ComfyUI service unavailable")]
    ServiceUnavailable,
    #[error("Circuit breaker open - ComfyUI failing")]
    CircuitOpen,
    #[error("Request timeout after {0} seconds")]
    Timeout(u16),
    #[error("Max retries ({0}) exceeded")]
    MaxRetriesExceeded(u8),
}

/// Connection state for ComfyUI
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize)]
pub enum ComfyUIConnectionState {
    Connected,
    Degraded { consecutive_failures: u8 },
    Disconnected,
    CircuitOpen { until: DateTime<Utc> },
}

/// Circuit breaker state
struct CircuitBreaker {
    state: CircuitBreakerState,
    failure_count: u8,
    last_failure: Option<DateTime<Utc>>,
    open_until: Option<DateTime<Utc>>,
}

enum CircuitBreakerState {
    Closed,
    Open,
    HalfOpen,
}
```

### API Endpoints

```
GET  /api/config/comfyui       # Get current ComfyUI config
PUT  /api/config/comfyui       # Update ComfyUI config
GET  /api/status/comfyui       # Get current connection state
```

### Implementation Tasks

| Task | File | Description |
|------|------|-------------|
| 18A.1 | `Engine/src/domain/value_objects/comfyui_config.rs` (NEW) | Create ComfyUIConfig value object with Default impl |
| 18A.2 | `Engine/src/infrastructure/comfyui.rs` | Extend ComfyUIError enum with ServiceUnavailable, CircuitOpen, Timeout, MaxRetriesExceeded |
| 18A.3 | `Engine/src/infrastructure/comfyui.rs` | Add CircuitBreaker struct with state tracking, check_circuit(), record_success(), record_failure() |
| 18A.4 | `Engine/src/infrastructure/comfyui.rs` | Add cached health check with 5s TTL using last_health_check: Option<(DateTime<Utc>, bool)> |
| 18A.5 | `Engine/src/infrastructure/comfyui.rs` | Add `async fn with_retry<T, F>(&self, operation: F) -> Result<T, ComfyUIError>` with exponential backoff |
| 18A.6 | `Engine/src/infrastructure/comfyui.rs` | Integrate health check into queue_prompt - call before queuing |
| 18A.7 | `Engine/src/infrastructure/comfyui.rs` | Add timeouts to all HTTP calls using `client.post(...).timeout(Duration::from_secs(timeout))` |
| 18A.8 | `Engine/src/infrastructure/http/config_routes.rs` (NEW) | GET/PUT /api/config/comfyui, GET /api/status/comfyui |
| 18A.9 | `Engine/src/infrastructure/comfyui.rs` | Track ComfyUIConnectionState based on recent operations, expose via pub fn connection_state() |

---

## Phase 18B: Generation Event Wiring [CRITICAL]

### Problem Statement

Currently:
- `GenerationService` emits `GenerationEvent` variants but they're NOT broadcast via WebSocket
- Player's `session_service.rs` has TODO at line ~484: "Update GenerationState when passed to this function"
- `GenerationQueuePanel` only shows "Failed" label, not the actual error message
- No ComfyUI connection status visible to users

### User Stories

**US-18B.1**: Real-time progress updates
**As a** user
**I want to** see generation progress updates in real-time
**So that** I know my batch is being processed

**Acceptance Criteria:**
- GenerationQueuePanel updates progress bar as batch generates
- Progress shown as percentage (0-100%)
- Updates received via WebSocket (not polling)

---

**US-18B.2**: Detailed error messages
**As a** user
**I want to** see detailed error messages when generation fails
**So that** I can troubleshoot

**Acceptance Criteria:**
- Failed batches show error message in queue panel
- Error expandable to show full details
- Error messages use user-friendly language (not raw HTTP errors)

---

**US-18B.3**: ComfyUI disconnected banner
**As a** user
**I want to** see a banner when ComfyUI is disconnected
**So that** I know why generation isn't working

**Acceptance Criteria:**
- Banner appears at top of Creator Mode when ComfyUI disconnected
- Banner shows "ComfyUI Disconnected - Reconnecting..." or "ComfyUI Unavailable"
- Banner includes recovery suggestion
- Banner auto-dismisses when connection restored

---

**US-18B.4**: Global state updates
**As a** user
**I want** the generation queue to update across all my open panels
**So that** I don't miss batch completion

**Acceptance Criteria:**
- GenerationState is global, not per-panel
- Batch completion shows notification badge
- Navigation to queue panel shows updated state

---

**US-18B.5**: Session-scoped events
**As the** system
**I want** generation events broadcast to the correct session
**So that** multi-user scenarios work correctly

**Acceptance Criteria:**
- Generation events sent only to session that requested them
- DM and Players in same session both receive events
- Events include batch_id for correlation

---

### UI Mockups

**Error State in Queue Panel:**
```
+----------------------------------------------------------+
|  GENERATION QUEUE                                         |
+----------------------------------------------------------+
|                                                           |
|  [x] Grizzled Veteran (Portrait)        FAILED            |
|      ComfyUI Error: Workflow node "IPAdapter" not found   |
|      [ Show Details v ]                                   |
|      [ Retry ]                                            |
|                                                           |
|  [*] Elven Archer (Portrait)            45%               |
|      |==================........|                         |
|                                                           |
|  [...] Town Square (Backdrop)           #2 in queue       |
|                                                           |
+----------------------------------------------------------+

Legend: [x] = red error icon, [*] = yellow generating icon, [...] = gray queued icon
```

**Expanded Error Details:**
```
+----------------------------------------------------------+
|  [x] Grizzled Veteran (Portrait)        FAILED            |
|      +--------------------------------------------+       |
|      | Full Error:                                |       |
|      | HTTP 400 - Node "IPAdapter" (id: 15) not  |       |
|      | found. Check ComfyUI custom nodes are     |       |
|      | installed.                                 |       |
|      |                                            |       |
|      | Batch ID: 8a7f3e21-...                    |       |
|      | Workflow: character-portrait-generation   |       |
|      | Attempted: 3 times                        |       |
|      +--------------------------------------------+       |
|      [ Retry ] [ Dismiss ]                                |
+----------------------------------------------------------+
```

**ComfyUI Disconnected Banner:**
```
+===========================================================+
| [!] ComfyUI Disconnected - Generation unavailable         |
|     Reconnecting in 15s... [ Retry Now ]                  |
+===========================================================+
```

### WebSocket Messages

**New Message Type:**
```rust
// Add to ServerMessage enum (both Engine and Player)
ServerMessage::ComfyUIStateChanged {
    state: String,           // "connected", "degraded", "disconnected", "circuit_open"
    message: Option<String>, // Human-readable status
    retry_in_seconds: Option<u32>, // Countdown for reconnect
}
```

**Existing Messages (already defined, need wiring):**
```rust
ServerMessage::GenerationQueued { batch_id, entity_type, entity_id, asset_type, position }
ServerMessage::GenerationProgress { batch_id, progress }
ServerMessage::GenerationComplete { batch_id, asset_count }
ServerMessage::GenerationFailed { batch_id, error }
```

### Implementation Tasks

| Task | File | Description |
|------|------|-------------|
| 18B.1 | `Engine/src/infrastructure/websocket.rs` | Wire GenerationService event channel to WebSocket broadcaster - subscribe to event_sender, map GenerationEvent to ServerMessage, send to session |
| 18B.2 | `Engine/src/infrastructure/websocket.rs` + `Player/src/infrastructure/websocket/messages.rs` | Add ComfyUIStateChanged message variant to both |
| 18B.3 | `Engine/src/infrastructure/comfyui.rs` + `websocket.rs` | Emit event when ComfyUI connection state changes, broadcast to all sessions |
| 18B.4 | `Player/src/application/services/session_service.rs` | Handle generation events in handle_server_message(), call generation_state methods (fix TODO at line ~484) |
| 18B.5 | `Player/src/presentation/state/generation_state.rs` | Add `comfyui_state: Signal<ComfyUIConnectionState>` and `set_comfyui_state()` method |
| 18B.6 | `Player/src/presentation/components/creator/generation_queue.rs` | Show error message for Failed batches, add expandable details section, add Retry button |
| 18B.7 | `Player/src/presentation/components/creator/comfyui_banner.rs` (NEW) | Create disconnected banner component with countdown and retry button |
| 18B.8 | `Player/src/presentation/views/dm_view.rs` or Creator layout | Render ComfyUIBanner at top when state != Connected |

---

## Phase 18C: Style Reference System

### User Stories

**US-18C.1**: Select style reference from gallery
**As a** DM
**I want to** select an existing asset as a style reference
**So that** new generations match my established visual style

**Acceptance Criteria:**
- Gallery shows "Use as Reference" option on asset context menu
- Selected reference shown in generation modal
- Reference cleared with X button

---

**US-18C.2**: Auto-detect IPAdapter inputs
**As the** system
**I want to** auto-detect IPAdapter inputs in workflows
**So that** style references work without manual config

**Acceptance Criteria:**
- Workflow analysis detects nodes with class_type containing "IPAdapter"
- Auto-map first detected IPAdapter image input
- Fall back to prompt injection if no IPAdapter found

---

**US-18C.3**: Manual reference input configuration
**As a** DM
**I want to** manually configure which input receives the style reference
**So that** custom workflows work correctly

**Acceptance Criteria:**
- Workflow config editor shows "Style Reference Input" dropdown
- Dropdown lists all image inputs in workflow
- Option for "None (use prompt injection)" and "Auto-detect"

---

**US-18C.4**: Prompt injection fallback
**As the** system
**I want to** inject style description into prompts when IPAdapter unavailable
**So that** style reference still helps

**Acceptance Criteria:**
- When no IPAdapter mapped, append style keywords to prompt
- Style keywords extracted from reference asset's generation metadata
- Format: `{user_prompt}, in the style of: {style_keywords}`

---

**US-18C.5**: Style lineage tracking
**As a** DM
**I want to** see which assets were generated with a style reference
**So that** I can track style lineage

**Acceptance Criteria:**
- Asset info panel shows "Style Reference: [asset name]" if applicable
- Clicking reference navigates to source asset
- Gallery can filter by "Generated from reference"

---

### UI Mockups

**Style Reference in Generation Modal:**
```
+----------------------------------------------------------+
|  Generate Portrait                                   [X]  |
+----------------------------------------------------------+
|                                                           |
|  Style Reference:                                         |
|  +------+                                                 |
|  |      |  Elven_Archer_Portrait_001                     |
|  | [img]|  Using: IPAdapter                              |
|  |      |  [ Clear ]                                      |
|  +------+                                                 |
|  [ Select from Gallery... ]                               |
|                                                           |
|  Prompt:                                                  |
|  +----------------------------------------------------+  |
|  | A grizzled human veteran with a scar across...    |  |
|  +----------------------------------------------------+  |
|                                                           |
|  Negative Prompt:                                         |
|  +----------------------------------------------------+  |
|  | blurry, low quality, cartoon                       |  |
|  +----------------------------------------------------+  |
|                                                           |
|  Variations: [====4====]                                  |
|                                                           |
|                              [ Cancel ]  [ Generate ]     |
+----------------------------------------------------------+
```

**Asset Context Menu with Reference Option:**
```
+---------------------------+
| Elven_Archer_Portrait_001 |
+---------------------------+
| > Set as Active           |
| > Use as Style Reference  |
| > View Generation Info    |
| > Download                |
| ---                       |
| > Delete                  |
+---------------------------+
```

**Workflow Config - Style Reference Input:**
```
+----------------------------------------------------------+
|  Workflow Configuration: character-portrait-generation    |
+----------------------------------------------------------+
|                                                           |
|  Prompt Mapping:                                          |
|  Primary: [CLIP Text Encode (6) -> text       v]          |
|  Negative: [CLIP Text Encode (7) -> text      v]          |
|                                                           |
|  Style Reference Input:                                   |
|  [ Auto-detect (IPAdapter) v ]                            |
|                                                           |
|    o Auto-detect (IPAdapter)                              |
|    o Load Image (12) -> image                             |
|    o IPAdapter (15) -> image                              |
|    o None (inject into prompt)                            |
|                                                           |
+----------------------------------------------------------+
```

### Domain Model

```rust
// Engine/src/domain/entities/workflow_config.rs (MODIFY)

/// Mapping for style reference image input
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum StyleReferenceMapping {
    /// Auto-detect IPAdapter node in workflow
    AutoDetect,
    /// Specific node/input mapping
    Specific { node_id: String, input_name: String },
    /// No IPAdapter, inject style keywords into prompt
    PromptInjection,
    /// Disabled - don't use style reference at all
    Disabled,
}

impl Default for StyleReferenceMapping {
    fn default() -> Self {
        Self::AutoDetect
    }
}

// Add to WorkflowConfiguration struct:
pub style_reference_mapping: StyleReferenceMapping,
```

### Implementation Tasks

| Task | File | Description |
|------|------|-------------|
| 18C.1 | `Engine/src/domain/entities/workflow_config.rs` | Add StyleReferenceMapping enum and field to WorkflowConfiguration |
| 18C.2 | `Engine/src/application/services/workflow_service.rs` | Add detect_ipadapter_inputs() - scan workflow JSON for nodes with "IPAdapter" in class_type |
| 18C.3 | `Engine/src/application/services/generation_service.rs` | In prepare_workflow: if style_reference_id set, load image path, inject via IPAdapter or prompt |
| 18C.4 | `Player/src/presentation/components/creator/asset_gallery.rs` | Add style_reference_id to GenerateRequest, add UI section for selecting/clearing reference |
| 18C.5 | `Player/src/presentation/components/creator/asset_gallery.rs` | Add "Use as Reference" to AssetThumbnail context menu |
| 18C.6 | `Player/src/presentation/components/settings/workflow_config_editor.rs` | Add dropdown for StyleReferenceMapping with detected image inputs |
| 18C.7 | `Player/src/presentation/components/creator/asset_gallery.rs` | Show "Style Reference: [name]" in asset info panel when applicable |

---

## Phase 18D: Director Mode Quick Generate

### User Stories

**US-18D.1**: Generate from NPC panel
**As a** DM in Director Mode
**I want to** generate assets for an NPC directly from their panel
**So that** I don't have to switch to Creator Mode

**Acceptance Criteria:**
- NPC panel shows "Generate Portrait" button
- Opens same generation modal as Creator Mode
- Batch queued via same API

---

**US-18D.2**: Pre-populate prompt
**As a** DM
**I want** the prompt pre-populated from the character's description
**So that** I don't have to retype it

**Acceptance Criteria:**
- Generation modal prompt field pre-filled with character description
- User can edit before submitting
- Negative prompt uses workflow defaults

---

**US-18D.3**: Queue indicator
**As a** DM
**I want to** see generation progress in Director Mode
**So that** I know when assets are ready

**Acceptance Criteria:**
- Small queue indicator in Director Mode header
- Shows count of pending generations
- Clicking opens minimal queue panel

---

**US-18D.4**: Auto-associate assets
**As a** DM
**I want** generated assets automatically associated with the NPC
**So that** I don't have to manually link them

**Acceptance Criteria:**
- Generation request includes entity_type="character" and entity_id
- Completed assets appear in NPC's asset gallery
- Notification when complete: "Portrait ready for [NPC Name]"

---

### UI Mockups

**NPC Panel with Generate Button:**
```
+--------------------------------------+
|  Grizzled Veteran                    |
+--------------------------------------+
|  +------+                            |
|  |      |  [ Generate Portrait ]     |
|  | [?]  |  [ Generate Sprite ]       |
|  |      |                            |
|  +------+                            |
|                                      |
|  Mood: [Suspicious v]                |
|                                      |
|  Immediate Goal:                     |
|  [ Protect the artifact from       ]|
|  [ strangers                       ]|
+--------------------------------------+

Note: [?] = placeholder when no portrait exists
```

**Director Mode Header with Queue Badge:**
```
+================================================================+
|  DIRECTOR MODE                            [Queue: 2] [Exit]    |
+================================================================+
```

**Minimal Queue Panel (Director Mode):**
```
+-------------------------------------+
|  Generation Queue               [X] |
+-------------------------------------+
|  [*] Grizzled Veteran (Portrait)    |
|      Generating... 67%              |
|                                     |
|  [...] Town Guard (Sprite)          |
|      #2 in queue                    |
+-------------------------------------+
```

### Implementation Tasks

| Task | File | Description |
|------|------|-------------|
| 18D.1 | `Player/src/presentation/components/dm_panel/npc_motivation.rs` | Add "Generate Portrait" and "Generate Sprite" buttons to NPC panel |
| 18D.2 | `Player/src/presentation/components/dm_panel/director_generate_modal.rs` (NEW) | Create modal component, reuse generation logic from asset_gallery, pre-populate prompt |
| 18D.3 | `Player/src/presentation/views/dm_view.rs` | Add queue badge to Director Mode header showing generation_state.active_count() |
| 18D.4 | `Player/src/presentation/components/dm_panel/director_queue_panel.rs` (NEW) | Create minimal queue panel as slide-in or modal |
| 18D.5 | Component state/API | Extract character description for prompt, truncate if > 500 chars |

---

## Phase 18E: Batch Management UI

### User Stories

**US-18E.1**: Cancel pending batches
**As a** DM
**I want to** cancel a pending batch
**So that** I can free up the queue for more important generations

**Acceptance Criteria:**
- Cancel button on queued batches
- Confirmation dialog: "Cancel generation for [name]?"
- Canceled batches removed from queue immediately
- ComfyUI job aborted if possible

---

**US-18E.2**: Retry failed batches
**As a** DM
**I want to** retry a failed batch
**So that** transient errors don't require re-entering all parameters

**Acceptance Criteria:**
- Retry button on failed batches
- Retry uses same parameters as original
- Retry count displayed (e.g., "Attempt 2 of 3")
- After retry, batch shows "Queued" status

---

**US-18E.3**: Clear from queue
**As a** DM
**I want to** clear completed/failed batches from the queue
**So that** the list stays manageable

**Acceptance Criteria:**
- "Clear" button on completed and failed batches
- "Clear All Completed" bulk action
- Cleared batches removed from GenerationState
- Does NOT delete the generated assets

---

**US-18E.4**: View batch details
**As a** DM
**I want to** see batch details
**So that** I can understand what was generated

**Acceptance Criteria:**
- Expand batch to see full prompt text
- Show workflow used
- Show timestamp (requested at, completed at)
- Show entity name (character/location)

---

**US-18E.5**: Reorder queue (Optional)
**As a** DM
**I want to** reorder queued batches
**So that** urgent generations run first

**Acceptance Criteria:**
- Drag-and-drop reordering in queue
- Move to top / Move to bottom buttons
- Server respects new order

---

### UI Mockup

**Queue with Management Controls:**
```
+----------------------------------------------------------+
|  GENERATION QUEUE                    [ Clear Completed ]  |
+----------------------------------------------------------+
|                                                           |
|  [+] Elven Archer (Portrait)         READY - 4 images    |
|      Completed 2 min ago                                  |
|      [ Select ] [ Clear ]                                 |
|                                                           |
|  [x] Town Guard (Sprite)             FAILED              |
|      Error: ComfyUI timeout                               |
|      [ Retry ] [ Clear ] [ Show Details v ]               |
|                                                           |
|  [*] Merchant (Portrait)             GENERATING 45%       |
|      |==================........|                         |
|                                                           |
|  [...] Castle Entrance (Backdrop)    #2 in queue         |
|      [ Cancel ]                                           |
|                                                           |
|  [...] Tavern Interior (Backdrop)    #3 in queue         |
|      [ Cancel ]                                           |
|                                                           |
+----------------------------------------------------------+

Legend: [+] = green complete, [x] = red failed, [*] = yellow generating, [...] = gray queued
```

**Batch Details Expanded:**
```
+----------------------------------------------------------+
|  [+] Elven Archer (Portrait)         READY - 4 images    |
|      +--------------------------------------------+       |
|      | Prompt: A graceful elven archer with long |       |
|      | silver hair, piercing blue eyes, wearing  |       |
|      | forest green leather armor...             |       |
|      |                                            |       |
|      | Workflow: character-portrait-generation   |       |
|      | Requested: 5 min ago                      |       |
|      | Completed: 2 min ago                      |       |
|      | Style Reference: None                     |       |
|      +--------------------------------------------+       |
|      [ Select ] [ Clear ]                                 |
+----------------------------------------------------------+
```

### API Endpoints

```
# Cancel a batch (abort if in progress)
DELETE /api/assets/batch/{batch_id}

# Retry a failed batch
POST /api/assets/batch/{batch_id}/retry

# Clear a batch from queue (doesn't delete assets)
DELETE /api/assets/batch/{batch_id}/clear

# Reorder queue (optional)
POST /api/assets/batch/reorder
Body: { batch_ids: ["uuid1", "uuid2", ...] }
```

### Implementation Tasks

| Task | File | Description |
|------|------|-------------|
| 18E.1 | `Engine/src/infrastructure/http/asset_routes.rs` | DELETE /api/assets/batch/{batch_id} - call generation_service.cancel_batch(), attempt ComfyUI cancel |
| 18E.2 | `Engine/src/infrastructure/http/asset_routes.rs` | POST /api/assets/batch/{batch_id}/retry - load original batch, reset status to Queued, re-queue |
| 18E.3 | `Engine/src/infrastructure/http/asset_routes.rs` | DELETE /api/assets/batch/{batch_id}/clear - remove batch record (not assets) |
| 18E.4 | `Player/src/presentation/components/creator/generation_queue.rs` | Add Cancel button for Queued, Retry/Clear for Failed, Select/Clear for Ready |
| 18E.5 | `Player/src/presentation/components/creator/generation_queue.rs` | Add confirmation dialogs using existing patterns |
| 18E.6 | `Player/src/presentation/components/creator/generation_queue.rs` | Add expand/collapse toggle to queue items, show prompt/workflow/timestamps |
| 18E.7 | `Player/src/presentation/components/creator/generation_queue.rs` | Add "Clear All Completed" button in header, batch API calls |
| 18E.8 | (Optional) Both | POST /api/assets/batch/reorder + drag-and-drop UI |

---

## Critical Files Summary

### Phase 18A (Resilience)
- `Engine/src/infrastructure/comfyui.rs` - Core retry/circuit breaker logic
- `Engine/src/domain/value_objects/comfyui_config.rs` (NEW) - Configuration types
- `Engine/src/infrastructure/http/config_routes.rs` (NEW) - Config API

### Phase 18B (Event Wiring) - CRITICAL
- `Engine/src/infrastructure/websocket.rs` - Wire GenerationEvent to broadcast
- `Player/src/application/services/session_service.rs` - Fix TODO at line ~484
- `Player/src/presentation/state/generation_state.rs` - Add comfyui_state signal
- `Player/src/presentation/components/creator/generation_queue.rs` - Error display UI
- `Player/src/presentation/components/creator/comfyui_banner.rs` (NEW) - Disconnected banner

### Phase 18C (Style References)
- `Engine/src/domain/entities/workflow_config.rs` - Add StyleReferenceMapping
- `Engine/src/application/services/workflow_service.rs` - IPAdapter detection
- `Engine/src/application/services/generation_service.rs` - Inject reference
- `Player/src/presentation/components/creator/asset_gallery.rs` - Reference selection UI
- `Player/src/presentation/components/settings/workflow_config_editor.rs` - Reference input config

### Phase 18D (Director Generate)
- `Player/src/presentation/components/dm_panel/npc_motivation.rs` - Add generate buttons
- `Player/src/presentation/components/dm_panel/director_generate_modal.rs` (NEW) - Modal component
- `Player/src/presentation/components/dm_panel/director_queue_panel.rs` (NEW) - Minimal queue
- `Player/src/presentation/views/dm_view.rs` - Queue badge in header

### Phase 18E (Batch Management)
- `Engine/src/infrastructure/http/asset_routes.rs` - Cancel/retry/clear endpoints
- `Player/src/presentation/components/creator/generation_queue.rs` - Management UI with buttons

---

## Styling Reference

From existing codebase patterns:
- Error background: `rgba(239, 68, 68, 0.1)` (light red)
- Error text: `#ef4444` (red)
- Warning icon/text: `#facc15` (yellow)
- Success icon: `#4ade80` (green)
- Panel background: `#0f0f23` (dark blue)
- Border: `#374151` (gray)
- Info text: `#6b7280` (gray)

---

## Dependencies

- **Phase 18A → 18B**: Resilience provides health states that 18B broadcasts
- **Phase 18B → 18C/D/E**: Event wiring enables real-time updates for all subsequent phases
- **Phase 11/12**: Existing asset gallery and workflow config infrastructure
- **Existing WebSocket**: Message types already defined, need wiring
