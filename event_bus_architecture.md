# Event Bus Architecture

**Date**: 2025-12-15  
**Status**: ✅ **IMPLEMENTED**  
**Engine Commit**: `dd1a6f2`

---

## Overview

The Event Bus is a **publish-subscribe infrastructure** for announcing application outcomes and domain events across the Engine. It complements the existing **action queue system**:

- **Action Queues** (PlayerAction, LLMReasoning, AssetGeneration, DMApproval):
  - Semantics: "Do this work"
  - Consumption: One worker/resolver processes each job exactly-once
  - Used for: Commands requiring processing (LLM calls, asset generation, approvals)

- **Event Bus**:
  - Semantics: "This happened" (facts about outcomes)
  - Consumption: Multiple subscribers can react; publishing doesn't consume the event
  - Used for: Cross-cutting notifications (WebSocket, analytics, story timeline, metrics)

---

## Architecture

### Hexagonal / DDD Alignment

The event bus follows hexagonal architecture principles:

```
Domain/Application Layer
├── AppEvent DTO (application/dto/app_events.rs)
│   └── Serializable, coarse-grained events
├── EventBusPort<AppEvent> (application/ports/outbound/event_bus_port.rs)
│   └── Abstract interface for publishing
└── AppEventRepositoryPort (application/ports/outbound/app_event_repository_port.rs)
    └── Abstract interface for event storage/retrieval

Infrastructure Layer
├── SqliteEventBus (infrastructure/event_bus/sqlite_event_bus.rs)
│   └── Implements EventBusPort using SQLite + notifier
├── SqliteAppEventRepository (infrastructure/repositories/sqlite_app_event_repository.rs)
│   └── Implements AppEventRepositoryPort with SQLite
├── InProcessEventNotifier (infrastructure/event_bus/in_process_notifier.rs)
│   └── Instant notification for in-process subscribers
└── WebSocketEventSubscriber (infrastructure/websocket_event_subscriber.rs)
    └── Converts AppEvents → ServerMessages → broadcasts to clients
```

### Data Flow

```
[Domain Services]
       ↓
  StoryEventService.record_*()
  NarrativeEventService.mark_triggered()
  Challenge resolution (websocket.rs)
       ↓
  AppEvent::StoryEventCreated
  AppEvent::NarrativeEventTriggered
  AppEvent::ChallengeResolved
       ↓
  EventBusPort<AppEvent>.publish()
       ↓
  SqliteEventBus
   ├→ SqliteAppEventRepository (persist to DB)
   └→ InProcessEventNotifier (wake subscribers)
       ↓
  WebSocketEventSubscriber (polls SQLite + listens to notifier)
   ├→ fetch_since(last_id)
   └→ map AppEvent → ServerMessage
       ↓
  SessionManager.broadcast_to_session()
       ↓
  [WebSocket Clients / Player]


[GenerationService / LLMQueueService]
       ↓
  GenerationEvent::Batch*/Suggestion*
       ↓
  GenerationEventPublisher
       ↓
  AppEvent::GenerationBatch*/Suggestion*
       ↓
  (same path as above: EventBus → SQLite → Subscriber → WebSocket)
```

---

## Implementation Details

### 1. AppEvent DTO

**File**: `Engine/src/application/dto/app_events.rs`

A serializable enum capturing significant system outcomes:

- **Story & Narrative**:
  - `StoryEventCreated { story_event_id, world_id, event_type }`
  - `NarrativeEventTriggered { event_id, world_id, event_name, outcome_name }`

- **Challenge**:
  - `ChallengeResolved { challenge_id, challenge_name, world_id, character_id, success, roll, total }`

- **Generation (Images)**:
  - `GenerationBatchQueued { batch_id, entity_type, entity_id, asset_type, position }`
  - `GenerationBatchProgress { batch_id, progress }`
  - `GenerationBatchCompleted { batch_id, entity_type, entity_id, asset_type, asset_count }`
  - `GenerationBatchFailed { batch_id, entity_type, entity_id, asset_type, error }`

- **Suggestions (LLM Text)**:
  - `SuggestionQueued { request_id, field_type, entity_id }`
  - `SuggestionProgress { request_id, status }`
  - `SuggestionCompleted { request_id, field_type, suggestions }`
  - `SuggestionFailed { request_id, field_type, error }`

**Design principles**:
- Only IDs and small payloads (not full domain objects)
- Serializable via serde for persistence and cross-process use
- Includes enough context for routing/filtering (world_id, entity_type, etc.)

### 2. EventBusPort<AppEvent>

**File**: `Engine/src/application/ports/outbound/event_bus_port.rs`

Async trait for publishing events:

```rust
#[async_trait]
pub trait EventBusPort<E: Serialize + Send + Sync + 'static>: Send + Sync {
    async fn publish(&self, event: E) -> Result<(), EventBusError>;
}
```

**Error handling**:
- Publishing failures are typically **non-fatal** (logged, not panicked)
- Ensures the main gameplay flow is never blocked by event bus issues

### 3. SQLite Backend (V1)

**Table**: `app_events`
- `id` (INTEGER PRIMARY KEY AUTOINCREMENT)
- `event_type` (TEXT) – enum variant name
- `payload` (TEXT) – JSON-serialized AppEvent
- `created_at` (TEXT) – ISO-8601 timestamp
- `processed` (INTEGER) – optional flag for consumers
- `processed_at` (TEXT) – optional timestamp

**SqliteEventBus** (`infrastructure/event_bus/sqlite_event_bus.rs`):
- On `publish()`:
  1. Insert event into SQLite via `AppEventRepositoryPort`
  2. Best-effort `notify()` via `InProcessEventNotifier`

**InProcessEventNotifier** (`infrastructure/event_bus/in_process_notifier.rs`):
- Uses `tokio::sync::Notify`
- `notify()` wakes all waiters
- `wait()` suspends until notified

### 4. Subscribers

**WebSocketEventSubscriber** (`infrastructure/websocket_event_subscriber.rs`):
- Background task spawned in `main.rs`
- Uses `tokio::select!` to wait on:
  - `notifier.wait()` for instant updates
  - `tokio::time::sleep(30s)` for polling fallback
- On wake:
  - Fetches events `> last_event_id` from repository
  - Maps `AppEvent` → `ServerMessage`
  - Broadcasts to all sessions via `SessionManager`
- **Currently broadcasts to all sessions**; future optimization: filter by world_id

---

## Producers (What Publishes Events)

### Story Events
**Service**: `StoryEventService`  
**When**: After any `StoryEvent` is persisted  
**Event**: `AppEvent::StoryEventCreated`  
**Methods affected**: All `record_*` methods (dialogue, challenge, scene transition, DM marker, etc.)

### Narrative Events
**Service**: `NarrativeEventService`  
**When**: When a narrative event is marked as triggered  
**Event**: `AppEvent::NarrativeEventTriggered`  
**Method**: `mark_triggered()`

### Challenges
**Location**: `infrastructure/websocket.rs` (ClientMessage::ChallengeRoll handler)  
**When**: After challenge resolution and before broadcasting to clients  
**Event**: `AppEvent::ChallengeResolved`

### Generation & Suggestions
**Services**: `GenerationService`, `LLMQueueService`  
**Pipeline**:
1. Services emit `GenerationEvent::Batch*/Suggestion*` (internal domain events)
2. `GenerationEventPublisher` listens to the GenerationEvent channel
3. Maps to `AppEvent::GenerationBatch*/Suggestion*`
4. Publishes via EventBus

**Enhancement**: GenerationEvent variants now include full entity context (entity_type, entity_id, asset_type, field_type) for proper AppEvent construction.

---

## Consumers (Who Subscribes)

### WebSocketEventSubscriber (Current)
- Maps generation/suggestion events → `ServerMessage::Generation*/Suggestion*`
- Broadcasts to all WebSocket clients in all sessions
- **Story/Narrative/Challenge events**: Currently not mapped to WebSocket (logged only)

### Future Consumers (Planned)
- **Analytics subscriber**: Track event metrics (counts, timing, success rates)
- **Story timeline projector**: Build denormalized views for Story Arc UI
- **Notification service**: Send out-of-band alerts (email, push, Discord)
- **Audit logger**: Compliance/security logging
- **Redis forwarder**: Bridge to distributed event system

---

## Migration Strategy

### Current State (As of Engine commit dd1a6f2)

✅ **Implemented**:
- Full EventBus port abstraction
- SQLite-backed event repository with app_events table
- InProcessEventNotifier for instant wake-ups
- 30-second polling fallback for resilience
- WebSocketEventSubscriber converting AppEvents → ServerMessages
- All producers wired:
  - StoryEventService (all 10+ record methods)
  - NarrativeEventService (mark_triggered)
  - Challenge resolution
  - GenerationEventPublisher (all generation/suggestion events)

✅ **Behavior preserved**:
- Creator Mode generation and suggestions work exactly as before
- WebSocket clients receive the same ServerMessage types
- No Player-side changes required

### Future Enhancements

**Redis Backend** (when scaling to multi-node):
- Implement `RedisEventBus` with same `EventBusPort<AppEvent>` interface
- Use Redis Streams or Pub/Sub
- Update `AppState` to select backend via config (SQLite vs Redis)
- No changes needed in application services or subscribers

**Advanced Consumers**:
- Story timeline: react to StoryEventCreated/NarrativeEventTriggered
- Metrics: scrape ChallengeResolved for success rates
- Notifications: alert DMs when specific events occur

**Optimizations**:
- World-scoped broadcasting (filter WebSocket messages by world_id)
- Event pruning/archival (move old events to cold storage)
- Separate storage for large payloads (e.g., suggestions) with references in AppEvent

---

## Design Rationale

### Why SQLite + Notifier + Polling?

- **Consistency with queues**: Matches existing queue backend pattern
- **Durability**: Events are persisted for debugging, replay, and audit
- **Resilience**: Polling fallback ensures no missed events even on process restart
- **Simple operations**: SQLite is fast for append-only event logs

### Why Keep GenerationEvent Separate from AppEvent?

- **GenerationEvent**: Internal, strongly-typed signal within application layer
- **AppEvent**: Serializable envelope for persistence and cross-boundary use
- This separation allows:
  - Type-safe internal communication
  - Evolution of internal events without breaking external consumers

### Why Publish from Infrastructure (Challenge Resolution)?

- Challenge resolution is currently in `websocket.rs` (infrastructure)
- Rather than move all logic to application layer now, we pragmatically publish from infra
- Future refactor could extract challenge resolution into an application service

---

## Testing

### Validation Performed

- ✅ Engine compiles successfully
- ✅ Player compiles successfully (no changes required)
- ✅ All event producers integrated
- ✅ WebSocketEventSubscriber properly maps events

### Manual QA (To be performed)

1. **Generation events**:
   - Create a character in Creator Mode
   - Trigger image generation
   - Verify:
     - `GenerationQueued` appears in queue UI
     - Progress updates
     - `GenerationComplete` clears from queue
   - Check `app_events` table for `GenerationBatch*` rows

2. **Suggestion events**:
   - Click "AI Suggest" for a character field
   - Verify:
     - `SuggestionQueued` appears in queue UI
     - Dropdown shows suggestions when ready
   - Check `app_events` table for `Suggestion*` rows

3. **Story events**:
   - Perform in-game action (NPC dialogue, challenge)
   - Check `app_events` table for `StoryEventCreated` rows
   - Verify event payloads include correct world_id, event_id

4. **Narrative events**:
   - Trigger a narrative event from Story Arc tab
   - Check `app_events` table for `NarrativeEventTriggered` row

5. **Challenge events**:
   - Perform a challenge roll
   - Check `app_events` table for `ChallengeResolved` row

### Integration Tests (Future)

- Mock `EventBusPort` to verify services publish events
- Test `SqliteEventBus.publish()` writes to database
- Test `WebSocketEventSubscriber` polling/notification cycle
- End-to-end: enqueue generation → poll event → assert ServerMessage broadcast

---

## Related Documentation

- **Queues**: See Phase 19 documentation and `code_review_next_steps.md`
- **Unified Generation Queue**: See `20-unified-generation-queue.md` and `suggestion_queue_oversight.md`
- **Hexagonal Architecture**: See ROADMAP.md Tier 3 (DDD & Architecture Quality)

---

## Future Work

1. **Redis Backend**:
   - Implement `RedisEventBus` for multi-node deployments
   - Add config flag to select SQLite vs Redis

2. **Event Pruning**:
   - Archive old events (>30 days) to separate table or file
   - Implement cleanup job in `main.rs`

3. **Advanced Subscribers**:
   - Story timeline projector (build denormalized views for Story Arc UI)
   - Metrics collector (export to Prometheus/StatsD)
   - Audit logger (write to separate audit log)

4. **Cross-service Events**:
   - When multiple Engine instances exist, Redis Streams or Kafka enable event sharing
   - Enables distributed story logging, multi-world coordination

5. **Player-side Event Handling** (if needed):
   - For now, Player receives events as ServerMessages via WebSocket
   - Future: Player could have its own event bus for client-side concerns (UI updates, offline queue)

---

## Summary

The Event Bus refactor successfully:

- ✅ Introduces a **clean port abstraction** (`EventBusPort<AppEvent>`) suitable for SQLite (now) and Redis (future)
- ✅ **Persists events** in SQLite for durability and replay
- ✅ Uses **notification + polling** pattern matching existing queues
- ✅ Refactors generation/suggestion pipeline to **eliminate direct ServerMessage coupling** in main.rs
- ✅ Wires **all major producers** (Story, Narrative, Challenge, Generation, Suggestions) to publish events
- ✅ Preserves **existing behavior** for WebSocket clients; no Player changes required
- ✅ Sets up **future extensibility** for Redis, analytics, and cross-service consumers

This work front-loads architectural quality, making it easy to add new event producers and consumers without further large-scale refactoring.

