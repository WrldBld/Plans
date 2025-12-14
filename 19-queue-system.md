# Phase 19: Unified Queue System

## Overview

WrldBldr requires a robust queue system to manage the flow of actions, AI processing, and human approvals. This phase implements a hexagonal queue architecture with pluggable storage backends.

---

## Queue Architecture

### Five Queues

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              QUEUE SYSTEM OVERVIEW                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                         INPUT QUEUES (Actions)                                 │ │
│  │                                                                                │ │
│  │  ┌─────────────────────────────┐    ┌─────────────────────────────────────┐   │ │
│  │  │ PlayerActionQueue           │    │ DMActionQueue                       │   │ │
│  │  │                             │    │                                     │   │ │
│  │  │ • Speak to NPC              │    │ • Approval decisions                │   │ │
│  │  │ • Examine object            │    │ • Direct NPC control                │   │ │
│  │  │ • Travel                    │    │ • Scene transitions                 │   │ │
│  │  │ • Use item                  │    │ • Trigger events                    │   │ │
│  │  │                             │    │                                     │   │ │
│  │  │ Priority: Normal            │    │ Priority: High                      │   │ │
│  │  └───────────┬─────────────────┘    └──────────────────┬──────────────────┘   │ │
│  │              │                                         │                      │ │
│  └──────────────┼─────────────────────────────────────────┼──────────────────────┘ │
│                 │                                         │                        │
│                 ▼                                         │                        │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                      PROCESSING QUEUES (AI Work)                               │ │
│  │                                                                                │ │
│  │  ┌─────────────────────────────┐    ┌─────────────────────────────────────┐   │ │
│  │  │ LLMReasoningQueue           │    │ AssetGenerationQueue                │   │ │
│  │  │                             │    │                                     │   │ │
│  │  │ • NPC response generation   │    │ • Character portraits               │   │ │
│  │  │ • Suggestions (Phase 15)    │    │ • Location backdrops                │   │ │
│  │  │ • Challenge reasoning       │    │ • Item images                       │   │ │
│  │  │                             │    │                                     │   │ │
│  │  │ BATCH_SIZE: configurable    │    │ BATCH_SIZE: 1 (always)              │   │ │
│  │  │ Default: 1 (sequential)     │    │ (ComfyUI queues internally)         │   │ │
│  │  └───────────┬─────────────────┘    └─────────────────────────────────────┘   │ │
│  │              │                                                                 │ │
│  └──────────────┼─────────────────────────────────────────────────────────────────┘ │
│                 │                                                                   │
│                 ▼                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                      APPROVAL QUEUE (Human Decision)                           │ │
│  │                                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐  │ │
│  │  │ DMApprovalQueue                                                         │  │ │
│  │  │                                                                         │  │ │
│  │  │ • NPC responses awaiting approval                                       │  │ │
│  │  │ • Tool calls awaiting approval                                          │  │ │
│  │  │ • Challenge suggestions awaiting approval                               │  │ │
│  │  │                                                                         │  │ │
│  │  │ Status: Pending | Approved | Rejected | Delayed | Expired               │  │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                                │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                 │                                                                   │
│                 ▼                                                                   │
│            Broadcast to Players                                                     │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Queue Purposes

| Queue | Layer | Purpose | Persistence | Priority |
|-------|-------|---------|-------------|----------|
| `PlayerActionQueue` | Input | Player actions awaiting processing | Yes | Normal |
| `DMActionQueue` | Input | DM actions awaiting processing | Yes | High |
| `LLMReasoningQueue` | Processing | LLM requests with concurrency control | Optional | FIFO |
| `AssetGenerationQueue` | Processing | ComfyUI requests with concurrency control | Optional | FIFO |
| `DMApprovalQueue` | Approval | Decisions awaiting human approval | Yes | By urgency |

---

## Hexagonal Architecture

### Port Interfaces

```rust
// application/ports/outbound/queue_port.rs

/// Generic queue item with metadata
#[derive(Debug, Clone)]
pub struct QueueItem<T> {
    pub id: QueueItemId,
    pub payload: T,
    pub status: QueueItemStatus,
    pub priority: u8,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub attempts: u32,
    pub max_attempts: u32,
    pub metadata: HashMap<String, String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum QueueItemStatus {
    Pending,
    Processing,
    Completed,
    Failed,
    Delayed,
    Expired,
}

/// Core queue port - storage-agnostic interface
#[async_trait]
pub trait QueuePort<T>: Send + Sync
where
    T: Send + Sync + Clone + Serialize + DeserializeOwned,
{
    /// Add item to queue
    async fn enqueue(&self, payload: T, priority: u8) -> Result<QueueItemId, QueueError>;

    /// Get next item for processing (marks as Processing)
    async fn dequeue(&self) -> Result<Option<QueueItem<T>>, QueueError>;

    /// Peek at next item without removing
    async fn peek(&self) -> Result<Option<QueueItem<T>>, QueueError>;

    /// Mark item as completed
    async fn complete(&self, id: QueueItemId) -> Result<(), QueueError>;

    /// Mark item as failed (may retry based on attempts)
    async fn fail(&self, id: QueueItemId, error: &str) -> Result<(), QueueError>;

    /// Delay item for later processing
    async fn delay(&self, id: QueueItemId, until: DateTime<Utc>) -> Result<(), QueueError>;

    /// Get item by ID
    async fn get(&self, id: QueueItemId) -> Result<Option<QueueItem<T>>, QueueError>;

    /// Get all items with status
    async fn list_by_status(&self, status: QueueItemStatus) -> Result<Vec<QueueItem<T>>, QueueError>;

    /// Get queue depth (pending items)
    async fn depth(&self) -> Result<usize, QueueError>;

    /// Clear completed/failed items older than duration
    async fn cleanup(&self, older_than: Duration) -> Result<usize, QueueError>;
}

/// Extended port for approval queues with human-facing features
#[async_trait]
pub trait ApprovalQueuePort<T>: QueuePort<T>
where
    T: Send + Sync + Clone + Serialize + DeserializeOwned,
{
    /// Get items by session
    async fn list_by_session(&self, session_id: SessionId) -> Result<Vec<QueueItem<T>>, QueueError>;

    /// Get history (completed/failed/expired items)
    async fn get_history(&self, session_id: SessionId, limit: usize) -> Result<Vec<QueueItem<T>>, QueueError>;

    /// Expire items older than duration
    async fn expire_old(&self, older_than: Duration) -> Result<usize, QueueError>;
}

/// Port for processing queues with concurrency control
#[async_trait]
pub trait ProcessingQueuePort<T>: QueuePort<T>
where
    T: Send + Sync + Clone + Serialize + DeserializeOwned,
{
    /// Get batch size configuration
    fn batch_size(&self) -> usize;

    /// Get number of items currently processing
    async fn processing_count(&self) -> Result<usize, QueueError>;

    /// Check if can accept more work
    async fn has_capacity(&self) -> Result<bool, QueueError>;
}
```

### Queue Item Types

```rust
// domain/value_objects/queue_items.rs

/// Player action waiting to be processed
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PlayerActionItem {
    pub session_id: SessionId,
    pub player_id: String,
    pub action_type: String,
    pub target: Option<String>,
    pub dialogue: Option<String>,
    pub timestamp: DateTime<Utc>,
}

/// DM action waiting to be processed
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DMActionItem {
    pub session_id: SessionId,
    pub dm_id: String,
    pub action: DMAction,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DMAction {
    ApprovalDecision {
        request_id: String,
        decision: ApprovalDecision,
    },
    DirectNPCControl {
        npc_id: String,
        dialogue: String,
    },
    TriggerEvent {
        event_id: String,
    },
    TransitionScene {
        scene_id: SceneId,
    },
}

/// LLM request waiting to be processed
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LLMRequestItem {
    pub request_type: LLMRequestType,
    pub session_id: Option<SessionId>,
    pub prompt: GamePromptRequest,
    pub callback_id: String,  // For routing response back
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum LLMRequestType {
    NPCResponse { action_item_id: QueueItemId },
    Suggestion { field_type: String, entity_id: Option<String> },
    ChallengeReasoning { challenge_id: String },
}

/// Asset generation request
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AssetGenerationItem {
    pub session_id: Option<SessionId>,
    pub entity_type: String,
    pub entity_id: String,
    pub workflow_id: String,
    pub prompt: String,
    pub count: u32,
}

/// Decision awaiting DM approval
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ApprovalItem {
    pub session_id: SessionId,
    pub source_action_id: QueueItemId,  // Links back to PlayerActionItem
    pub decision_type: DecisionType,
    pub urgency: DecisionUrgency,
    pub npc_name: String,
    pub proposed_dialogue: String,
    pub internal_reasoning: String,
    pub proposed_tools: Vec<ProposedToolInfo>,
    pub retry_count: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DecisionType {
    NPCResponse,
    ToolUsage,
    ChallengeSuggestion,
    SceneTransition,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq, PartialOrd, Ord)]
pub enum DecisionUrgency {
    Normal = 0,
    AwaitingPlayer = 1,
    SceneCritical = 2,
}
```

---

## Storage Backends

### Backend Options

| Backend | Persistence | Performance | Use Case |
|---------|-------------|-------------|----------|
| **InMemory** | No | Fastest | Development, testing |
| **SQLite** | Yes | Fast | Production (single server) |
| **Redis** | Yes | Fast | Production (multi-server) |
| **Neo4j** | Yes | Slower | If graph relationships needed |

### SQLite Schema

```sql
-- Queue items table (one per queue type, or polymorphic)
CREATE TABLE queue_items (
    id TEXT PRIMARY KEY,
    queue_name TEXT NOT NULL,
    payload_json TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    priority INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    scheduled_at TEXT,  -- For delayed items
    attempts INTEGER NOT NULL DEFAULT 0,
    max_attempts INTEGER NOT NULL DEFAULT 3,
    error_message TEXT,
    metadata_json TEXT
);

CREATE INDEX idx_queue_status ON queue_items(queue_name, status, priority DESC, created_at);
CREATE INDEX idx_queue_scheduled ON queue_items(queue_name, status, scheduled_at) WHERE status = 'delayed';

-- Session-specific index for approval queue
CREATE INDEX idx_queue_session ON queue_items(queue_name, (json_extract(payload_json, '$.session_id')), status);
```

### Infrastructure Implementation

```rust
// infrastructure/queues/sqlite_queue.rs

pub struct SqliteQueue<T> {
    pool: SqlitePool,
    queue_name: String,
    _phantom: PhantomData<T>,
}

impl<T> SqliteQueue<T>
where
    T: Send + Sync + Clone + Serialize + DeserializeOwned,
{
    pub async fn new(pool: SqlitePool, queue_name: &str) -> Result<Self, QueueError> {
        // Ensure table exists
        sqlx::query(QUEUE_TABLE_SCHEMA)
            .execute(&pool)
            .await?;

        Ok(Self {
            pool,
            queue_name: queue_name.to_string(),
            _phantom: PhantomData,
        })
    }
}

#[async_trait]
impl<T> QueuePort<T> for SqliteQueue<T>
where
    T: Send + Sync + Clone + Serialize + DeserializeOwned + 'static,
{
    async fn enqueue(&self, payload: T, priority: u8) -> Result<QueueItemId, QueueError> {
        let id = QueueItemId::new();
        let payload_json = serde_json::to_string(&payload)?;
        let now = Utc::now();

        sqlx::query(
            "INSERT INTO queue_items (id, queue_name, payload_json, status, priority, created_at, updated_at)
             VALUES (?, ?, ?, 'pending', ?, ?, ?)"
        )
        .bind(id.to_string())
        .bind(&self.queue_name)
        .bind(&payload_json)
        .bind(priority as i32)
        .bind(now.to_rfc3339())
        .bind(now.to_rfc3339())
        .execute(&self.pool)
        .await?;

        Ok(id)
    }

    async fn dequeue(&self) -> Result<Option<QueueItem<T>>, QueueError> {
        // Atomic dequeue with status update
        let row = sqlx::query(
            "UPDATE queue_items
             SET status = 'processing', updated_at = ?, attempts = attempts + 1
             WHERE id = (
                 SELECT id FROM queue_items
                 WHERE queue_name = ? AND status = 'pending'
                 ORDER BY priority DESC, created_at ASC
                 LIMIT 1
             )
             RETURNING *"
        )
        .bind(Utc::now().to_rfc3339())
        .bind(&self.queue_name)
        .fetch_optional(&self.pool)
        .await?;

        row.map(|r| Self::row_to_item(r)).transpose()
    }

    // ... other implementations
}
```

### In-Memory Implementation (for testing/development)

```rust
// infrastructure/queues/memory_queue.rs

pub struct InMemoryQueue<T> {
    items: RwLock<BTreeMap<QueueItemId, QueueItem<T>>>,
    pending: RwLock<BinaryHeap<PriorityItem>>,
}

#[derive(Eq, PartialEq)]
struct PriorityItem {
    id: QueueItemId,
    priority: u8,
    created_at: DateTime<Utc>,
}

impl Ord for PriorityItem {
    fn cmp(&self, other: &Self) -> Ordering {
        // Higher priority first, then older items first
        self.priority.cmp(&other.priority)
            .then_with(|| other.created_at.cmp(&self.created_at))
    }
}

#[async_trait]
impl<T> QueuePort<T> for InMemoryQueue<T>
where
    T: Send + Sync + Clone + 'static,
{
    async fn enqueue(&self, payload: T, priority: u8) -> Result<QueueItemId, QueueError> {
        let id = QueueItemId::new();
        let now = Utc::now();

        let item = QueueItem {
            id,
            payload,
            status: QueueItemStatus::Pending,
            priority,
            created_at: now,
            updated_at: now,
            attempts: 0,
            max_attempts: 3,
            metadata: HashMap::new(),
        };

        self.items.write().await.insert(id, item);
        self.pending.write().await.push(PriorityItem { id, priority, created_at: now });

        Ok(id)
    }

    // ... other implementations
}
```

---

## Queue Services (Application Layer)

### PlayerActionQueueService

```rust
// application/services/player_action_queue_service.rs

pub struct PlayerActionQueueService<Q: QueuePort<PlayerActionItem>> {
    queue: Q,
    llm_queue_service: Arc<LLMQueueService>,
}

impl<Q: QueuePort<PlayerActionItem>> PlayerActionQueueService<Q> {
    /// Enqueue a player action for processing
    pub async fn enqueue_action(
        &self,
        session_id: SessionId,
        player_id: String,
        action_type: String,
        target: Option<String>,
        dialogue: Option<String>,
    ) -> Result<QueueItemId, QueueError> {
        let item = PlayerActionItem {
            session_id,
            player_id,
            action_type,
            target,
            dialogue,
            timestamp: Utc::now(),
        };

        self.queue.enqueue(item, PRIORITY_NORMAL).await
    }

    /// Process next action in queue
    pub async fn process_next<S: SessionQueryPort>(
        &self,
        session: &S,
    ) -> Result<Option<QueueItemId>, QueueError> {
        let Some(item) = self.queue.dequeue().await? else {
            return Ok(None);
        };

        // Gather context
        let context = self.gather_context(session, &item.payload)?;

        // Submit to LLM queue
        let llm_request = LLMRequestItem {
            request_type: LLMRequestType::NPCResponse { action_item_id: item.id },
            session_id: Some(item.payload.session_id),
            prompt: self.build_prompt(&context),
            callback_id: item.id.to_string(),
        };

        self.llm_queue_service.enqueue(llm_request).await?;

        // Mark action as completed (LLM queue handles the rest)
        self.queue.complete(item.id).await?;

        Ok(Some(item.id))
    }
}
```

### LLMQueueService (with concurrency control)

```rust
// application/services/llm_queue_service.rs

pub struct LLMQueueService<Q: ProcessingQueuePort<LLMRequestItem>> {
    queue: Q,
    semaphore: Arc<Semaphore>,
    ollama_client: Arc<OllamaClient>,
    approval_queue: Arc<dyn ApprovalQueuePort<ApprovalItem>>,
}

impl<Q: ProcessingQueuePort<LLMRequestItem>> LLMQueueService<Q> {
    pub fn new(
        queue: Q,
        batch_size: usize,
        ollama_client: Arc<OllamaClient>,
        approval_queue: Arc<dyn ApprovalQueuePort<ApprovalItem>>,
    ) -> Self {
        Self {
            queue,
            semaphore: Arc::new(Semaphore::new(batch_size.max(1))),
            ollama_client,
            approval_queue,
        }
    }

    /// Enqueue LLM request
    pub async fn enqueue(&self, request: LLMRequestItem) -> Result<QueueItemId, QueueError> {
        self.queue.enqueue(request, PRIORITY_NORMAL).await
    }

    /// Background worker that processes LLM requests
    pub async fn run_worker(&self) {
        loop {
            // Wait for capacity
            let permit = self.semaphore.acquire().await.unwrap();

            // Try to get next item
            let item = match self.queue.dequeue().await {
                Ok(Some(item)) => item,
                Ok(None) => {
                    drop(permit);
                    tokio::time::sleep(Duration::from_millis(100)).await;
                    continue;
                }
                Err(e) => {
                    tracing::error!("Failed to dequeue LLM request: {}", e);
                    drop(permit);
                    tokio::time::sleep(Duration::from_secs(1)).await;
                    continue;
                }
            };

            // Process in spawned task (permit moves into task)
            let client = self.ollama_client.clone();
            let queue = self.queue.clone();
            let approval_queue = self.approval_queue.clone();

            tokio::spawn(async move {
                let _permit = permit; // Keep permit alive during processing

                let result = client.generate(&item.payload.prompt).await;

                match result {
                    Ok(response) => {
                        // Route response based on request type
                        if let Err(e) = Self::handle_response(&item.payload, response, &approval_queue).await {
                            tracing::error!("Failed to handle LLM response: {}", e);
                            let _ = queue.fail(item.id, &e.to_string()).await;
                        } else {
                            let _ = queue.complete(item.id).await;
                        }
                    }
                    Err(e) => {
                        tracing::error!("LLM generation failed: {}", e);
                        let _ = queue.fail(item.id, &e.to_string()).await;
                    }
                }
            });
        }
    }

    async fn handle_response(
        request: &LLMRequestItem,
        response: LLMResponse,
        approval_queue: &Arc<dyn ApprovalQueuePort<ApprovalItem>>,
    ) -> Result<(), QueueError> {
        match &request.request_type {
            LLMRequestType::NPCResponse { action_item_id } => {
                // Create approval item for DM
                let approval = ApprovalItem {
                    session_id: request.session_id.unwrap(),
                    source_action_id: *action_item_id,
                    decision_type: DecisionType::NPCResponse,
                    urgency: DecisionUrgency::AwaitingPlayer,
                    npc_name: response.npc_name,
                    proposed_dialogue: response.dialogue,
                    internal_reasoning: response.reasoning,
                    proposed_tools: response.tool_calls,
                    retry_count: 0,
                };

                approval_queue.enqueue(approval, DecisionUrgency::AwaitingPlayer as u8).await?;
            }
            LLMRequestType::Suggestion { .. } => {
                // Send suggestion result via WebSocket (Phase 15)
                // No DM approval needed
            }
            LLMRequestType::ChallengeReasoning { .. } => {
                // Add to approval queue with challenge type
            }
        }

        Ok(())
    }
}
```

### DMApprovalQueueService

```rust
// application/services/dm_approval_queue_service.rs

pub struct DMApprovalQueueService<Q: ApprovalQueuePort<ApprovalItem>> {
    queue: Q,
    event_broadcaster: Arc<dyn EventBroadcaster>,
}

impl<Q: ApprovalQueuePort<ApprovalItem>> DMApprovalQueueService<Q> {
    /// Get all pending approvals for a session (for DM UI)
    pub async fn get_pending(&self, session_id: SessionId) -> Result<Vec<QueueItem<ApprovalItem>>, QueueError> {
        self.queue.list_by_session(session_id).await
    }

    /// Process DM approval decision
    pub async fn process_decision<S: SessionCommandPort>(
        &self,
        session: &mut S,
        item_id: QueueItemId,
        decision: ApprovalDecision,
    ) -> Result<ApprovalOutcome, QueueError> {
        let item = self.queue.get(item_id).await?
            .ok_or(QueueError::NotFound)?;

        let outcome = match decision {
            ApprovalDecision::Accept => {
                self.handle_accept(session, &item.payload).await?
            }
            ApprovalDecision::AcceptWithModification { modified_dialogue, .. } => {
                self.handle_accept_modified(session, &item.payload, &modified_dialogue).await?
            }
            ApprovalDecision::Reject { feedback } => {
                self.handle_reject(&item.payload, &feedback).await?
            }
            ApprovalDecision::TakeOver { dm_response } => {
                self.handle_takeover(session, &item.payload, &dm_response).await?
            }
        };

        // Mark item based on outcome
        match &outcome {
            ApprovalOutcome::Broadcast { .. } => {
                self.queue.complete(item_id).await?;
            }
            ApprovalOutcome::Rejected { needs_reprocessing: true, .. } => {
                // Item stays in queue, will be reprocessed
                self.queue.delay(item_id, Utc::now() + Duration::from_secs(1)).await?;
            }
            ApprovalOutcome::Rejected { needs_reprocessing: false, .. } |
            ApprovalOutcome::MaxRetriesExceeded { .. } => {
                self.queue.fail(item_id, "Rejected by DM").await?;
            }
        }

        Ok(outcome)
    }

    /// Delay a decision for later
    pub async fn delay_decision(
        &self,
        item_id: QueueItemId,
        duration: Duration,
    ) -> Result<(), QueueError> {
        self.queue.delay(item_id, Utc::now() + duration).await
    }

    /// Get decision history for session
    pub async fn get_history(
        &self,
        session_id: SessionId,
        limit: usize,
    ) -> Result<Vec<QueueItem<ApprovalItem>>, QueueError> {
        self.queue.get_history(session_id, limit).await
    }
}
```

---

## Configuration

```rust
// infrastructure/config.rs

#[derive(Debug, Clone, Deserialize)]
pub struct QueueConfig {
    /// Queue storage backend: "memory", "sqlite", "redis"
    #[serde(default = "default_queue_backend")]
    pub backend: String,

    /// SQLite database path (if using sqlite backend)
    #[serde(default = "default_queue_db_path")]
    pub sqlite_path: String,

    /// Redis URL (if using redis backend)
    pub redis_url: Option<String>,

    /// Max concurrent LLM requests
    #[serde(default = "default_llm_batch_size")]
    pub llm_batch_size: usize,

    /// Max concurrent ComfyUI requests (always 1 recommended)
    #[serde(default = "default_asset_batch_size")]
    pub asset_batch_size: usize,

    /// How long to keep completed items before cleanup
    #[serde(default = "default_history_retention")]
    pub history_retention_hours: u64,

    /// How long before pending approvals expire
    #[serde(default = "default_approval_timeout")]
    pub approval_timeout_minutes: u64,
}

fn default_queue_backend() -> String { "sqlite".to_string() }
fn default_queue_db_path() -> String { "./data/queues.db".to_string() }
fn default_llm_batch_size() -> usize { 1 }
fn default_asset_batch_size() -> usize { 1 }
fn default_history_retention() -> u64 { 24 }
fn default_approval_timeout() -> u64 { 30 }
```

Environment variables:
```bash
QUEUE_BACKEND=sqlite          # or "memory" or "redis"
QUEUE_SQLITE_PATH=./data/queues.db
QUEUE_REDIS_URL=redis://localhost:6379
QUEUE_LLM_BATCH_SIZE=1
QUEUE_ASSET_BATCH_SIZE=1
QUEUE_HISTORY_RETENTION_HOURS=24
QUEUE_APPROVAL_TIMEOUT_MINUTES=30
```

---

## Data Flow Examples

### Player Action → NPC Response Flow

```
1. Player sends "Talk to Jasper" via WebSocket

2. WebSocket handler:
   player_action_queue.enqueue(action)
   → Returns immediately, sends "Processing" to player

3. PlayerActionQueue worker:
   - Dequeues action
   - Gathers context (read lock)
   - Creates LLMRequestItem
   - Enqueues to LLMReasoningQueue
   - Marks action complete

4. LLMReasoningQueue worker:
   - Acquires semaphore permit
   - Dequeues request
   - Calls Ollama
   - Creates ApprovalItem
   - Enqueues to DMApprovalQueue
   - Sends "ApprovalRequired" to DM via WebSocket

5. DM sees approval request in UI

6. DM clicks "Approve"
   → WebSocket handler enqueues to DMActionQueue

7. DMActionQueue worker:
   - Dequeues action
   - Processes approval
   - Broadcasts dialogue to players
   - Marks approval complete
```

### Crash Recovery

```
Server crashes after step 4 (LLM completed but before DM notified)

On restart:
1. SQLite queue persists all items
2. DMApprovalQueue has pending item
3. Server re-sends "ApprovalRequired" to DM
4. No work is lost
```

### Audit Trail

```sql
-- Query to see all actions for a session
SELECT * FROM queue_items
WHERE queue_name = 'player_actions'
  AND json_extract(payload_json, '$.session_id') = 'session-123'
ORDER BY created_at;

-- Query to see approval history
SELECT * FROM queue_items
WHERE queue_name = 'dm_approvals'
  AND status IN ('completed', 'failed')
  AND json_extract(payload_json, '$.session_id') = 'session-123'
ORDER BY updated_at DESC
LIMIT 50;
```

---

## WebSocket Events

### Queue Status Events (to DM)

```rust
#[derive(Serialize)]
#[serde(tag = "type")]
pub enum QueueEvent {
    // Player action queued
    ActionQueued {
        action_id: String,
        player_name: String,
        action_type: String,
        queue_depth: usize,
    },

    // LLM processing started
    LLMProcessing {
        action_id: String,
        queue_position: usize,
    },

    // Approval ready for DM
    ApprovalRequired {
        approval_id: String,
        // ... full approval details
    },

    // Queue depth update
    QueueStatus {
        player_actions_pending: usize,
        llm_requests_pending: usize,
        llm_requests_processing: usize,
        approvals_pending: usize,
    },
}
```

---

## Implementation Plan

### Phase 19A: Core Queue Infrastructure

- [ ] **19A.1** Create queue port interfaces
  - File: `Engine/src/application/ports/outbound/queue_port.rs`
  - `QueuePort<T>`, `ApprovalQueuePort<T>`, `ProcessingQueuePort<T>`
  - `QueueItem<T>`, `QueueItemStatus`, `QueueError`

- [ ] **19A.2** Create queue item types
  - File: `Engine/src/domain/value_objects/queue_items.rs`
  - `PlayerActionItem`, `DMActionItem`, `LLMRequestItem`, `AssetGenerationItem`, `ApprovalItem`

- [ ] **19A.3** Implement InMemoryQueue
  - File: `Engine/src/infrastructure/queues/memory_queue.rs`
  - For development and testing

- [ ] **19A.4** Implement SqliteQueue
  - File: `Engine/src/infrastructure/queues/sqlite_queue.rs`
  - Production persistence
  - Add `sqlx` with SQLite feature to Cargo.toml

### Phase 19B: Queue Services

- [ ] **19B.1** Create PlayerActionQueueService
  - File: `Engine/src/application/services/player_action_queue_service.rs`
  - Enqueue and process player actions

- [ ] **19B.2** Create DMActionQueueService
  - File: `Engine/src/application/services/dm_action_queue_service.rs`
  - Enqueue and process DM actions

- [ ] **19B.3** Create LLMQueueService
  - File: `Engine/src/application/services/llm_queue_service.rs`
  - Concurrency-controlled LLM processing
  - Background worker task

- [ ] **19B.4** Create AssetGenerationQueueService
  - File: `Engine/src/application/services/asset_generation_queue_service.rs`
  - ComfyUI request processing

- [ ] **19B.5** Create DMApprovalQueueService
  - File: `Engine/src/application/services/dm_approval_queue_service.rs`
  - Migrate from current ApprovalService
  - Add history, delay, expiration

### Phase 19C: WebSocket Integration

- [ ] **19C.1** Update WebSocket handler to use queues
  - PlayerAction → PlayerActionQueue
  - ApprovalDecision → DMActionQueue

- [ ] **19C.2** Add queue status WebSocket events
  - QueueStatus updates to DM
  - ActionQueued, LLMProcessing events

- [ ] **19C.3** Start background workers
  - In main.rs, spawn worker tasks for each queue

### Phase 19D: Configuration & Polish

- [ ] **19D.1** Add QueueConfig to AppConfig
- [ ] **19D.2** Queue factory based on backend config
- [ ] **19D.3** Cleanup task for old items
- [ ] **19D.4** Health check for queue status
- [ ] **19D.5** Metrics/logging for queue operations

---

## Dependencies

- Phase 15 (Generation Queue) - Will use LLMQueueService + AssetGenerationQueueService
- Phase 16 (Decision Queue) - Will use DMApprovalQueueService
- Phase 18 (Lock Refactoring) - Queue services use phased operations pattern

---

## Future Enhancements

- **Redis backend** for multi-server deployments
- **Priority lanes** for different action types
- **Rate limiting** per player/session
- **Dead letter queue** for permanently failed items
- **Queue metrics dashboard** in UI
- **Replay capability** for debugging sessions
