# WrldBldr Code Review Report

**Date:** 2025-12-17
**Reviewer:** Claude Code
**Scope:** Engine and Player codebases - Architecture, Anti-patterns, Dead Code

---

## Executive Summary

Both codebases demonstrate **strong architectural foundations** with hexagonal architecture properly implemented. However, several issues require attention across both projects.

| Category | Engine | Player |
|----------|--------|--------|
| Critical Issues | 2 | 3 |
| High Issues | 5 | 4 |
| Medium Issues | 4 | 6 |
| Low Issues | 3 | 7 |
| **Total** | **14** | **20** |

---

## Part 1: Engine Code Review

### Hexagonal Architecture Compliance

**Overall Status:** Good - No layer violations detected

| Check | Result |
|-------|--------|
| Domain imports application? | No |
| Domain imports infrastructure? | No |
| Application imports infrastructure? | No |
| HTTP handlers call services (not repos)? | Yes |
| Repositories implement port traits? | Yes |
| Services use trait bounds? | Yes |

### Critical Issues

#### 1. Unsafe `.unwrap()` on Mutex Locks

**File:** `src/infrastructure/comfyui.rs`
**Lines:** 55, 71, 88-89, 96-97, 99, 137, 143, 151, 166, 197, 244, 252, 258, 283, 327, 371

```rust
let state = self.state.lock().unwrap();  // CRITICAL: Panics if poisoned
```

**Impact:** If any code panics while holding the mutex, subsequent calls crash the application.

**Fix:** Use proper error handling:
```rust
let state = self.state.lock()
    .map_err(|e| ComfyUIError::Internal(format!("Lock poisoned: {}", e)))?;
```

#### 2. Unused `outcomes` Parameter in Ad-Hoc Challenge

**File:** `src/application/services/challenge_resolution_service.rs:735`

```rust
pub async fn handle_adhoc_challenge(
    ...
    outcomes: serde_json::Value,  // Accepted but NEVER USED
) -> Option<serde_json::Value>
```

**Impact:** Ad-hoc challenges cannot have custom outcomes - silently discarded.

### High Severity Issues

#### 1. Serde on Domain Entities Without ADR

**Files:**
- `src/domain/entities/sheet_template.rs:309,391` - CharacterSheetData, FieldValue
- `src/domain/value_objects/approval.rs:6,15-16` - ProposedToolInfo, ApprovalDecision
- `src/domain/value_objects/comfyui_config.rs:6` - ComfyUIConfig

**Issue:** These types have `#[derive(Serialize, Deserialize)]` without documented architectural justification. Other domain value objects (like IDs) have ADR comments explaining the trade-off.

**Fix:** Add ADR comment or move to DTOs:
```rust
//! # Architectural Note
//!
//! These types intentionally include serde derives because:
//! 1. Serialization is intrinsic to their purpose
//! 2. Creating separate DTOs adds significant boilerplate
```

#### 2. Code Duplication: Roll Handling Logic

**File:** `src/application/services/challenge_resolution_service.rs`
**Lines:** 184-343 vs 348-533

`handle_roll()` and `handle_roll_input()` share ~70% identical code:
- Parse challenge ID
- Load challenge
- Get session and player info
- Look up skill modifier
- Publish AppEvent
- Execute outcome triggers
- Broadcast ChallengeResolved

**Fix:** Extract to `resolve_challenge_internal()`.

#### 3. God Object: AppState

**File:** `src/infrastructure/state.rs:40-108`

The `AppState` struct holds 30+ fields with a ~250 line constructor.

**Fix:** Split into domain-specific state modules:
```rust
pub struct AppState {
    pub persistence: PersistenceState,
    pub services: ServiceState,
    pub queues: QueueState,
}
```

#### 4. Magic Numbers Throughout

**Files/Lines:**
- `session.rs:757` - `Self::new(30)` conversation turns
- `comfyui.rs:101-104` - `>= 5` failures, `60` seconds open
- `comfyui.rs:261` - `30` seconds cache TTL
- `location_service.rs:157-163` - `255` name limit, `10000` description limit

**Fix:** Define constants:
```rust
pub mod validation {
    pub const MAX_NAME_LENGTH: usize = 255;
    pub const MAX_DESCRIPTION_LENGTH: usize = 10000;
}
```

#### 5. 18+ Unused Imports

From `cargo check`:
```
warning: unused imports: `EnhancedChallengeSuggestion`, `EnhancedOutcomes`, `OutcomeDetail`
warning: unused import: `std::sync::Arc`
warning: unused import: `ChallengeRepositoryPort`
```

**Fix:** Run `cargo fix --allow-dirty`.

### Medium Severity Issues

#### 1. Silently Swallowed Errors with `.ok()`

**Files:**
- `challenge_resolution_service.rs:86`
- `narrative_event_approval_service.rs:119,132,141,166`
- Various repositories - UUID parsing

**Fix:** Log warnings:
```rust
.filter_map(|s| {
    uuid::Uuid::parse_str(&s)
        .map(ChallengeId::from_uuid)
        .map_err(|e| tracing::warn!("Invalid UUID {}: {}", s, e))
        .ok()
})
```

#### 2. Dead Code with `#[allow(dead_code)]`

**Files:**
- `session_join_service.rs:238` - `convert_to_internal_snapshot`
- `session.rs:89,113,490,547,737,743` - Multiple fields/methods
- `async_session_port.rs:25,49` - Unused field and variant

**Fix:** Remove or document with tracking issues.

#### 3. Long Files (>500 lines)

| File | Lines |
|------|-------|
| `persistence/story_event_repository.rs` | 1383 |
| `persistence/narrative_event_repository.rs` | 1371 |
| `session.rs` | 1353 |
| `services/llm_service.rs` | 1108 |
| `services/tool_execution_service.rs` | 1023 |
| `services/challenge_resolution_service.rs` | 903 |
| `websocket.rs` | 928 |

**Fix:** Split into modules (e.g., `repository/mod.rs`, `repository/queries.rs`).

#### 4. 13 TODO Comments

```
websocket.rs:207 - character_name: None, // TODO: Load from character selection
player_character_routes.rs:193,275 - // TODO: Get user_id from authenticated session
challenge_resolution_service.rs:91 - /// # TODO: Architecture Violation
```

### Low Severity Issues

1. **Primitive Obsession** - Service methods accept `String` IDs instead of typed IDs
2. **Unused variable** in ComfyUI client (`last_error` set but never read)
3. **Missing unit tests** for critical paths (`challenge_resolution_service.rs`)

---

## Part 2: Player Code Review

### Hexagonal Architecture Compliance

**Overall Status:** Good with notable exceptions

| Check | Result |
|-------|--------|
| Domain imports application? | No |
| Domain imports infrastructure? | No |
| Application imports infrastructure? | Only in tests |
| Presentation imports infrastructure? | **YES - VIOLATION** |

### Critical Issues

#### 1. Presentation Imports Infrastructure (ApiAdapter)

**File:** `src/presentation/services.rs:15`

```rust
use crate::infrastructure::http_client::ApiAdapter;  // VIOLATION
```

**Impact:** Tight coupling between presentation and infrastructure, violates dependency inversion.

**Fix:** Move concrete instantiation to `main.rs` composition root.

#### 2. Routes.rs Imports Infrastructure

**File:** `src/routes.rs:104-105, 760-761, 905`

```rust
use crate::infrastructure::api::{set_engine_url, ws_to_http};
let connection = crate::infrastructure::connection_factory::ConnectionFactory::create_game_connection(&server_url);
```

**Fix:** Create Platform port methods for these operations.

#### 3. Duplicate `FormField` Component

**Files:**
- `components/creator/character_form.rs:498-516`
- `components/creator/location_form.rs:466-484`

Identical component defined in both files.

**Fix:** Extract to `components/shared/form_field.rs`.

### High Severity Issues

#### 1. God Component: DirectorModeContent

**File:** `src/presentation/views/dm_view.rs:388-848` (460+ lines)

Handles: scene preview, conversation log, 8+ modals, skills loading, session info, tone selection, NPC management, quick actions.

**Fix:** Split into:
- `ScenePreviewPanel`
- `ConversationLogPanel`
- `SessionInfoPanel`
- `QuickActionsPanel`

#### 2. DMView File Too Large

**File:** `src/presentation/views/dm_view.rs` - 1193 lines

Contains 10+ components that should be separate files.

#### 3. Duplicate `ApiError` Type

**Files:**
- `src/infrastructure/http_client.rs:30-40`
- `src/application/ports/outbound/api_port.rs:11-23`

**Fix:** Define only in port, use from infrastructure.

#### 4. Dead Code Components in dm_view.rs

Unused components:
- `LogEntry` (superseded by `DynamicLogEntry`)
- `ProposedAction`
- `NPCMotivationCard`
- `NPCToggle`

### Medium Severity Issues

#### 1. 1526 Inline Style Occurrences

```rust
style: "display: flex; flex-direction: column; height: 100%; background: #1a1a2e;",
```

**Fix:** Use Tailwind classes (already configured) or CSS files.

#### 2. Hard-coded Color Values

Colors repeated throughout:
- `#0f0f23`, `#1a1a2e`, `#374151`, `#9ca3af`, `#ef4444`, `#22c55e`

**Fix:** CSS custom properties or Tailwind theme.

#### 3. Missing Memoization

**File:** `dm_view.rs:791-794`

```rust
let active_challenges: Vec<ChallengeData> = challenges.read().iter()
    .filter(|c| c.active).cloned().collect();  // Runs every render
```

**Fix:** Use `use_memo` for derived state.

#### 4. Incomplete TODO Comments

6 unimplemented TODOs in dm_view.rs:
- Line 118: `create_adhoc_challenge`
- Line 370: Navigate to event details
- Line 699, 778: View as character
- Line 740: Location preview

#### 5. Inconsistent Service Patterns

Some services use generics, some use `dyn`:
```rust
pub struct ChallengeService<A: ApiPort> { api: A }  // Generic
pub struct SessionService { connection: Arc<dyn GameConnectionPort> }  // Dynamic
```

#### 6. GenerationState References Application Type

**File:** `src/presentation/state/generation_state.rs:54`

```rust
pub context: Option<crate::application::services::suggestion_service::SuggestionContext>,
```

**Fix:** Create presentation-layer copy if needed.

### Low Severity Issues

1. **Large files:** routes.rs (948), http_client.rs (740), character_form.rs (516)
2. **Placeholder components** without implementation roadmap (Items, Maps)
3. **Modal pattern duplication** - Extract reusable Modal wrapper
4. **Form state duplication** between CharacterForm and LocationForm
5. **Unused `to_route_str()` method** in StoryArcSubTab
6. **Duplicate service type re-exports** (CharacterSheetDataApi)
7. **Test imports from infrastructure** (acceptable pattern)

---

## Recommendations Priority Matrix

### Must Fix (This Sprint)

| Issue | Project | Impact |
|-------|---------|--------|
| Mutex unwrap() in ComfyUI | Engine | Crash risk |
| Presentation imports infrastructure | Player | Architecture violation |
| Routes.rs infrastructure imports | Player | Architecture violation |
| Unused outcomes parameter | Engine | Feature broken |

### Should Fix (Next Sprint)

| Issue | Project | Impact |
|-------|---------|--------|
| Duplicate FormField | Player | Maintenance |
| Roll handling duplication | Engine | Maintainability |
| AppState god object | Engine | Testability |
| DMView splitting | Player | Maintainability |
| Dead code removal | Both | Code hygiene |
| Unused imports | Engine | Warnings |

### Consider Improving (Technical Debt)

| Issue | Project | Impact |
|-------|---------|--------|
| Magic numbers | Engine | Readability |
| Inline styles | Player | Performance |
| Serde ADR documentation | Engine | Clarity |
| Memoization | Player | Performance |
| Service pattern consistency | Player | Consistency |

---

## Compiler Warning Summary

### Engine
- 18 unused imports
- 5 unused variables
- 2 unused assignments
- 3 dead code warnings

### Player
- Architecture boundary violations (critical)
- Dead code warnings for unused components
- Clippy warnings for inline styles

---

## Conclusion

Both codebases are **production-capable** with the core hexagonal architecture properly implemented. The critical issues identified should be addressed before the next release, particularly:

1. **Engine:** Fix mutex unwrap patterns to prevent crashes
2. **Player:** Fix architecture violations in presentation layer
3. **Both:** Remove dead code and unused imports

The remaining issues are technical debt that should be addressed iteratively without blocking feature development.
