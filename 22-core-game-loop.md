# Phase 22: Core Game Loop - Skill Challenges with LLM Tool Calls

**Status**: IN PROGRESS
**Priority**: HIGH - This is the main gameplay feature
**Depends On**: Architecture fixes from `code_review_discoveries.md`
**Last Updated**: 2025-12-16

---

## Overview

This phase implements the complete game loop where:
1. PC interacts with NPC → LLM suggests skill challenge with DC
2. DM approves/modifies/rejects the challenge suggestion
3. Player inputs dice roll (formula or manual result)
4. Engine resolves → outcomes include tool calls for state changes
5. DM has final say on all LLM suggestions

### User-Confirmed Design Decisions

| Decision | Choice |
|----------|--------|
| Dice System | Hybrid - suggest default dice from rule system, allow player override |
| Regeneration | DM chooses per request - single outcome or all outcomes |
| Persistence | Immediate - tool effects persist to Neo4j right away |
| Ad-hoc Challenges | Full support - DM can create custom challenges mid-session |

---

## Prerequisites

Before starting Phase 22, complete these fixes from `code_review_discoveries.md`:

### Required Fixes (Do First)

- [ ] **Fix 22-PRE-1**: Remove infrastructure import from `session_join_service.rs`
- [ ] **Fix 22-PRE-2**: Remove infrastructure import from `challenge_resolution_service.rs`
- [ ] **Fix 22-PRE-3**: Remove infrastructure import from `narrative_event_approval_service.rs`
- [ ] **Fix 22-PRE-4**: Fix HttpClient usage in `world_select.rs`
- [ ] **Fix 22-PRE-5**: Fix HttpClient usage in `presentation/services.rs`

---

## Implementation Phases

### Phase 22A: Character Modifier Integration (Engine)

**Status**: [ ] Not Started
**Estimated Effort**: 2-3 hours
**Files to Modify**:
- `Engine/src/application/services/challenge_resolution_service.rs`
- `Engine/src/application/services/player_character_service.rs` (create if needed)
- `Engine/src/infrastructure/persistence/player_character_repository.rs`

#### Task 22A.1: Create PlayerCharacterService.get_skill_modifier()

**File**: `Engine/src/application/services/player_character_service.rs`

```rust
use crate::application::ports::outbound::repository_port::PlayerCharacterRepository;
use crate::domain::value_objects::ids::{PlayerCharacterId, SkillId};

pub struct PlayerCharacterService<R: PlayerCharacterRepository> {
    repository: R,
}

impl<R: PlayerCharacterRepository> PlayerCharacterService<R> {
    pub fn new(repository: R) -> Self {
        Self { repository }
    }

    /// Get the skill modifier for a player character
    ///
    /// Looks up the character's sheet data and finds the skill entry.
    /// Returns: proficient ? (bonus + proficiency_bonus) : bonus
    pub async fn get_skill_modifier(
        &self,
        pc_id: &PlayerCharacterId,
        skill_id: &SkillId,
    ) -> Result<i32, ServiceError> {
        // 1. Load PlayerCharacter entity
        let pc = self.repository
            .get(pc_id)
            .await?
            .ok_or(ServiceError::NotFound("PlayerCharacter not found"))?;

        // 2. Get character sheet data
        let sheet = pc.sheet_data;

        // 3. Find skill entry by skill_id
        for field in &sheet.fields {
            if let FieldValue::SkillEntry { skill_id: sid, proficient, bonus } = &field.value {
                if sid == skill_id {
                    // 4. Calculate modifier
                    let proficiency_bonus = sheet.proficiency_bonus.unwrap_or(2);
                    return Ok(if *proficient {
                        *bonus + proficiency_bonus
                    } else {
                        *bonus
                    });
                }
            }
        }

        // Skill not found, return 0
        Ok(0)
    }
}
```

#### Task 22A.2: Wire into ChallengeResolutionService

**File**: `Engine/src/application/services/challenge_resolution_service.rs`

**Current (around line 108)**:
```rust
// TODO: integrate real character modifiers
let character_modifier = 0;
```

**Replace with**:
```rust
// Look up character's skill modifier
let character_modifier = match self.get_challenge_character(session_id, &challenge) {
    Some(pc_id) => {
        self.player_character_service
            .get_skill_modifier(&pc_id, &challenge.skill_id)
            .await
            .unwrap_or(0)
    }
    None => 0,
};
```

#### Task 22A.3: Map Session Participant to PlayerCharacter

Add helper method:

```rust
fn get_challenge_character(
    &self,
    session_id: &SessionId,
    challenge: &Challenge,
) -> Option<PlayerCharacterId> {
    // 1. Get session participants
    // 2. Find the player who triggered the challenge
    // 3. Return their PlayerCharacterId
    // For now, return the first PC in session
    todo!("Implement participant mapping")
}
```

#### Task 22A.4: Fix character_id in AppEvent

**Current (around line 120)**:
```rust
let character_id = "unknown".to_string();
```

**Replace with**:
```rust
let character_id = self.get_challenge_character(session_id, &challenge)
    .map(|id| id.0.to_string())
    .unwrap_or_else(|| "unknown".to_string());
```

#### Acceptance Criteria 22A

- [ ] `get_skill_modifier()` correctly looks up skill from character sheet
- [ ] Challenge resolution uses actual character modifier
- [ ] AppEvent includes correct character_id
- [ ] Engine compiles with no new warnings

---

### Phase 22B: Extended Tool System (Engine)

**Status**: [ ] Not Started
**Estimated Effort**: 3-4 hours
**Files to Modify**:
- `Engine/src/domain/value_objects/game_tools.rs`
- `Engine/src/application/services/tool_execution_service.rs`
- `Engine/src/infrastructure/persistence/character_repository.rs`

#### Task 22B.1: Add New GameTool Variants

**File**: `Engine/src/domain/value_objects/game_tools.rs`

Add these variants to the `GameTool` enum:

```rust
pub enum GameTool {
    // Existing
    GiveItem { item_name: String, description: String },
    RevealInfo { info_type: String, content: String, importance: String },
    ChangeRelationship { target_id: String, change: String, amount: i32, reason: String },
    TriggerEvent { event_type: String, description: String },

    // NEW - Phase 22B
    ModifyNpcMotivation {
        npc_id: String,
        motivation_type: String,  // "want", "fear", "goal"
        old_value: Option<String>,
        new_value: String,
        reason: String,
    },
    ModifyCharacterDescription {
        character_id: String,
        change_type: String,  // "appearance", "personality", "backstory"
        description: String,
    },
    ModifyNpcOpinion {
        npc_id: String,
        target_pc_id: String,
        opinion_change: i32,  // -100 to +100
        reason: String,
    },
    TransferItem {
        from_id: String,  // Character ID or "world" for loot
        to_id: String,
        item_name: String,
    },
    AddCondition {
        character_id: String,
        condition_name: String,
        description: String,
        duration: Option<String>,  // "until rest", "3 rounds", "permanent"
    },
    RemoveCondition {
        character_id: String,
        condition_name: String,
    },
    UpdateCharacterStat {
        character_id: String,
        stat_name: String,  // "hp", "mp", "gold", etc.
        delta: i32,  // positive or negative change
    },
}
```

#### Task 22B.2: Add Tool Descriptions for LLM

Add method for LLM function calling:

```rust
impl GameTool {
    /// Get the tool definition for LLM function calling
    pub fn get_llm_tool_definitions() -> Vec<LlmToolDefinition> {
        vec![
            LlmToolDefinition {
                name: "modify_npc_motivation".to_string(),
                description: "Change an NPC's wants, fears, or goals".to_string(),
                parameters: json!({
                    "type": "object",
                    "properties": {
                        "npc_id": { "type": "string", "description": "The NPC's ID" },
                        "motivation_type": { "type": "string", "enum": ["want", "fear", "goal"] },
                        "new_value": { "type": "string", "description": "The new motivation" },
                        "reason": { "type": "string", "description": "Why this changed" }
                    },
                    "required": ["npc_id", "motivation_type", "new_value", "reason"]
                }),
            },
            // ... add all other tools
        ]
    }
}
```

#### Task 22B.3: Implement Tool Execution

**File**: `Engine/src/application/services/tool_execution_service.rs`

Add match arms for new tools:

```rust
pub async fn execute_tool(
    &self,
    tool: &GameTool,
    session_id: &SessionId,
) -> Result<ToolExecutionResult, ServiceError> {
    match tool {
        // ... existing tools ...

        GameTool::ModifyNpcMotivation { npc_id, motivation_type, new_value, reason, .. } => {
            // 1. Load character
            let character = self.character_service.get_character(npc_id).await?;

            // 2. Update motivation based on type
            match motivation_type.as_str() {
                "want" => {
                    // Add or update want
                    self.character_service.update_want(npc_id, new_value).await?;
                }
                "fear" => {
                    // Update fear (stored in wants with negative intensity?)
                    todo!("Implement fear modification");
                }
                "goal" => {
                    // Update immediate goal
                    todo!("Implement goal modification");
                }
                _ => return Err(ServiceError::InvalidInput("Unknown motivation type")),
            }

            // 3. Return result with state change
            Ok(ToolExecutionResult {
                tool_name: "modify_npc_motivation".to_string(),
                success: true,
                state_changes: vec![
                    StateChange::NpcMotivationChanged {
                        npc_id: npc_id.clone(),
                        motivation_type: motivation_type.clone(),
                        new_value: new_value.clone(),
                    }
                ],
                message: format!("{}'s {} is now: {}", character.name, motivation_type, new_value),
            })
        }

        GameTool::ModifyNpcOpinion { npc_id, target_pc_id, opinion_change, reason } => {
            // 1. Load or create relationship
            // 2. Modify sentiment by opinion_change
            // 3. Persist to Neo4j
            // 4. Return state change
            todo!("Implement opinion modification");
        }

        GameTool::AddCondition { character_id, condition_name, description, duration } => {
            // 1. Load character
            // 2. Add condition to character's conditions list
            // 3. Persist
            // 4. Return state change
            todo!("Implement condition addition");
        }

        // ... implement other tools similarly
    }
}
```

#### Task 22B.4: Add StateChange Variants

**File**: `Engine/src/application/dto/app_events.rs` or create new file

```rust
pub enum StateChange {
    // Existing
    ItemGiven { item_name: String, to_character: String },
    RelationshipChanged { from: String, to: String, sentiment: f32 },

    // NEW
    NpcMotivationChanged { npc_id: String, motivation_type: String, new_value: String },
    NpcOpinionChanged { npc_id: String, target_pc_id: String, new_opinion: i32 },
    CharacterDescriptionChanged { character_id: String, change_type: String },
    ItemTransferred { from_id: String, to_id: String, item_name: String },
    ConditionAdded { character_id: String, condition: String },
    ConditionRemoved { character_id: String, condition: String },
    StatChanged { character_id: String, stat: String, new_value: i32 },
}
```

#### Acceptance Criteria 22B

- [ ] All 7 new GameTool variants defined
- [ ] LLM tool definitions generated for all tools
- [ ] At least `ModifyNpcMotivation`, `ModifyNpcOpinion`, `AddCondition` implemented
- [ ] State changes persisted to Neo4j
- [ ] Engine compiles with no new warnings

---

### Phase 22C: Outcome Trigger Execution (Engine)

**Status**: [ ] Not Started
**Estimated Effort**: 2-3 hours
**Files to Modify**:
- `Engine/src/application/services/challenge_resolution_service.rs`
- `Engine/src/application/services/outcome_trigger_service.rs` (create)

#### Task 22C.1: Create OutcomeTriggerService

**File**: `Engine/src/application/services/outcome_trigger_service.rs`

```rust
use crate::domain::entities::challenge::{Outcome, OutcomeTrigger};
use crate::domain::value_objects::ids::SessionId;

pub struct OutcomeTriggerService<C, S, I>
where
    C: ChallengeService,
    S: SceneService,
    I: InventoryService,
{
    challenge_service: C,
    scene_service: S,
    inventory_service: I,
}

impl<C, S, I> OutcomeTriggerService<C, S, I>
where
    C: ChallengeService,
    S: SceneService,
    I: InventoryService,
{
    /// Execute all triggers from a challenge outcome
    pub async fn execute_triggers(
        &self,
        outcome: &Outcome,
        session_id: &SessionId,
        character_id: &str,
    ) -> Result<Vec<StateChange>, ServiceError> {
        let mut state_changes = Vec::new();

        for trigger in &outcome.triggers {
            match trigger {
                OutcomeTrigger::RevealInformation { info, persist } => {
                    // Add to player's notes/journal
                    if *persist {
                        // Store in session state
                    }
                    state_changes.push(StateChange::InformationRevealed {
                        info: info.clone(),
                        persistent: *persist,
                    });
                }

                OutcomeTrigger::EnableChallenge { challenge_id } => {
                    self.challenge_service.set_active(challenge_id, true).await?;
                    state_changes.push(StateChange::ChallengeEnabled {
                        challenge_id: challenge_id.0.to_string(),
                    });
                }

                OutcomeTrigger::DisableChallenge { challenge_id } => {
                    self.challenge_service.set_active(challenge_id, false).await?;
                    state_changes.push(StateChange::ChallengeDisabled {
                        challenge_id: challenge_id.0.to_string(),
                    });
                }

                OutcomeTrigger::ModifyCharacterStat { stat, modifier } => {
                    // Update character's stat
                    // This needs CharacterService integration
                    state_changes.push(StateChange::StatChanged {
                        character_id: character_id.to_string(),
                        stat: stat.clone(),
                        delta: *modifier,
                    });
                }

                OutcomeTrigger::TriggerScene { scene_id } => {
                    // Queue scene transition
                    state_changes.push(StateChange::SceneTransitionQueued {
                        scene_id: scene_id.0.to_string(),
                    });
                }

                OutcomeTrigger::GiveItem { item_name, item_description } => {
                    // Add to character's inventory
                    self.inventory_service.give_item(
                        character_id,
                        item_name,
                        item_description,
                    ).await?;
                    state_changes.push(StateChange::ItemGiven {
                        item_name: item_name.clone(),
                        to_character: character_id.to_string(),
                    });
                }

                OutcomeTrigger::Custom { description } => {
                    // Log for DM manual handling
                    tracing::info!("Custom trigger: {}", description);
                    state_changes.push(StateChange::CustomTrigger {
                        description: description.clone(),
                    });
                }
            }
        }

        Ok(state_changes)
    }
}
```

#### Task 22C.2: Wire into Challenge Resolution

**File**: `Engine/src/application/services/challenge_resolution_service.rs`

After determining the outcome, add:

```rust
// After evaluate_challenge_result()
let (outcome_type, outcome) = evaluate_challenge_result(&challenge, raw_roll, character_modifier);

// Execute outcome triggers
let trigger_state_changes = self.outcome_trigger_service
    .execute_triggers(&outcome, session_id, &character_id)
    .await?;

// Combine with tool call state changes
let mut all_state_changes = trigger_state_changes;
all_state_changes.extend(tool_state_changes);

// Include in broadcast
let result_msg = ServerMessage::ChallengeResolved {
    // ... existing fields ...
    state_changes: all_state_changes,  // Add this field
};
```

#### Acceptance Criteria 22C

- [ ] OutcomeTriggerService created with all trigger types
- [ ] Triggers execute after challenge resolution
- [ ] State changes are collected and broadcast
- [ ] EnableChallenge/DisableChallenge work correctly
- [ ] GiveItem adds to inventory

---

### Phase 22D: Regeneration & Discard Handlers (Engine)

**Status**: [ ] Not Started
**Estimated Effort**: 2-3 hours
**Files to Modify**:
- `Engine/src/infrastructure/websocket.rs`
- `Engine/src/application/services/dm_approval_queue_service.rs`
- `Engine/src/application/services/llm_service.rs`

#### Task 22D.1: Implement RegenerateOutcome Handler

**File**: `Engine/src/infrastructure/websocket.rs`

Find the TODO for RegenerateOutcome and implement:

```rust
ClientMessage::RegenerateOutcome { request_id, outcome_type, guidance } => {
    // 1. Verify sender is DM
    if !is_dm(&client_id, &sessions) {
        return send_error(client_id, "Only DM can regenerate outcomes");
    }

    // 2. Get pending approval item
    let approval_item = dm_approval_queue_service
        .get_pending(&request_id)
        .await?
        .ok_or("Approval not found")?;

    // 3. Get challenge suggestion
    let suggestion = approval_item.challenge_suggestion
        .ok_or("No challenge suggestion to regenerate")?;

    // 4. Call LLM to regenerate specific outcome(s)
    let regenerated = llm_service
        .regenerate_outcome(
            &suggestion,
            outcome_type.as_deref(),  // None = all outcomes
            guidance.as_deref(),
        )
        .await?;

    // 5. Update approval item with regenerated outcomes
    dm_approval_queue_service
        .update_challenge_suggestion(&request_id, regenerated.clone())
        .await?;

    // 6. Send regenerated outcome to DM
    let response = ServerMessage::OutcomeRegenerated {
        request_id: request_id.clone(),
        outcome_type: outcome_type.unwrap_or_else(|| "all".to_string()),
        new_outcome: regenerated,
    };
    send_to_client(&client_id, &response);
}
```

#### Task 22D.2: Implement DiscardChallenge Handler

```rust
ClientMessage::DiscardChallenge { request_id, feedback } => {
    // 1. Verify sender is DM
    if !is_dm(&client_id, &sessions) {
        return send_error(client_id, "Only DM can discard challenges");
    }

    // 2. Get and update approval item
    let mut approval_item = dm_approval_queue_service
        .get_pending(&request_id)
        .await?
        .ok_or("Approval not found")?;

    // 3. Remove challenge suggestion
    approval_item.challenge_suggestion = None;

    // 4. Optionally re-queue for LLM with feedback
    if let Some(fb) = feedback {
        // Add feedback to prompt context
        approval_item.dm_feedback = Some(fb);

        // Re-queue for LLM processing
        llm_queue_service.enqueue_regeneration(approval_item).await?;
    } else {
        // Just remove the challenge, keep dialogue
        dm_approval_queue_service.update(&request_id, approval_item).await?;
    }

    // 5. Confirm to DM
    send_to_client(&client_id, &ServerMessage::ChallengeDiscarded {
        request_id: request_id.clone(),
    });
}
```

#### Task 22D.3: Add LLM Regeneration Method

**File**: `Engine/src/application/services/llm_service.rs`

```rust
/// Regenerate specific outcome(s) for a challenge suggestion
pub async fn regenerate_outcome(
    &self,
    current_suggestion: &ChallengeSuggestionInfo,
    outcome_type: Option<&str>,  // "success", "failure", etc. or None for all
    guidance: Option<&str>,
) -> Result<OutcomeDetailData, ServiceError> {
    let prompt = format!(
        r#"You previously suggested a "{}" challenge.

Current outcomes:
- Success: {}
- Failure: {}

{}

Please regenerate {} with fresh, creative text.
{}

Respond in JSON format:
{{
    "flavor_text": "narrative description",
    "scene_direction": "what happens in the scene",
    "proposed_tools": [...]
}}"#,
        current_suggestion.challenge_name,
        current_suggestion.outcomes.success.flavor_text,
        current_suggestion.outcomes.failure.flavor_text,
        guidance.map(|g| format!("DM guidance: {}", g)).unwrap_or_default(),
        outcome_type.unwrap_or("all outcomes"),
        if outcome_type.is_some() {
            "Only regenerate the specified outcome."
        } else {
            "Regenerate all outcomes (success, failure, critical_success, critical_failure)."
        }
    );

    let response = self.ollama_client.generate(&prompt).await?;

    // Parse JSON response
    let outcome: OutcomeDetailData = serde_json::from_str(&response)?;

    Ok(outcome)
}
```

#### Acceptance Criteria 22D

- [ ] RegenerateOutcome handler processes request and calls LLM
- [ ] DiscardChallenge handler removes challenge from approval
- [ ] DM receives OutcomeRegenerated/ChallengeDiscarded confirmation
- [ ] Guidance text is included in LLM prompt
- [ ] Can regenerate single outcome or all outcomes

---

### Phase 22E: Ad-hoc Challenge Backend (Engine)

**Status**: [ ] Not Started
**Estimated Effort**: 2-3 hours
**Files to Modify**:
- `Engine/src/infrastructure/websocket.rs`
- `Engine/src/application/services/challenge_resolution_service.rs`

#### Task 22E.1: Implement CreateAdHocChallenge Handler

**File**: `Engine/src/infrastructure/websocket.rs`

```rust
ClientMessage::CreateAdHocChallenge {
    challenge_name,
    skill_name,
    difficulty,
    target_pc_id,
    outcomes,
} => {
    // 1. Verify sender is DM
    if !is_dm(&client_id, &sessions) {
        return send_error(client_id, "Only DM can create ad-hoc challenges");
    }

    // 2. Get session info
    let session_id = get_client_session(&client_id, &sessions)?;
    let world_id = get_session_world(&session_id, &sessions)?;

    // 3. Look up skill by name
    let skill = skill_service
        .find_by_name(&world_id, &skill_name)
        .await?
        .ok_or("Skill not found")?;

    // 4. Parse difficulty
    let parsed_difficulty = parse_difficulty_string(&difficulty)?;

    // 5. Create transient challenge (not persisted)
    let challenge_id = ChallengeId::new();
    let challenge = Challenge {
        id: challenge_id.clone(),
        world_id,
        scene_id: None,
        name: challenge_name.clone(),
        description: format!("Ad-hoc challenge: {}", challenge_name),
        challenge_type: ChallengeType::SkillCheck,
        skill_id: skill.id.clone(),
        difficulty: parsed_difficulty,
        outcomes: ChallengeOutcomes {
            success: Outcome {
                description: outcomes.success.clone(),
                triggers: vec![],
            },
            failure: Outcome {
                description: outcomes.failure.clone(),
                triggers: vec![],
            },
            partial: None,
            critical_success: outcomes.critical_success.as_ref().map(|s| Outcome {
                description: s.clone(),
                triggers: vec![],
            }),
            critical_failure: outcomes.critical_failure.as_ref().map(|s| Outcome {
                description: s.clone(),
                triggers: vec![],
            }),
        },
        trigger_conditions: vec![],
        active: true,
        prerequisite_challenges: vec![],
        order: 0,
        is_favorite: false,
        tags: vec!["ad-hoc".to_string()],
    };

    // 6. Store in session's active challenges
    challenge_resolution_service
        .register_adhoc_challenge(&session_id, challenge.clone())
        .await?;

    // 7. Send challenge prompt to target PC
    let prompt = ServerMessage::ChallengePrompt {
        challenge_id: challenge_id.0.to_string(),
        challenge_name: challenge_name.clone(),
        skill_name: skill_name.clone(),
        difficulty_display: difficulty.clone(),
        description: format!("Ad-hoc challenge: {}", challenge_name),
        character_modifier: 0,  // Will be looked up when roll comes in
        suggested_dice: Some("1d20".to_string()),  // Default
        rule_system_hint: Some("Roll and add your modifier".to_string()),
    };

    // Send to specific PC or broadcast
    send_to_participant(&target_pc_id, &session_id, &prompt, &sessions);

    // 8. Confirm to DM
    send_to_client(&client_id, &ServerMessage::AdHocChallengeCreated {
        challenge_id: challenge_id.0.to_string(),
        challenge_name,
        target_pc_id,
    });
}
```

#### Task 22E.2: Add Session Challenge Storage

**File**: `Engine/src/application/services/challenge_resolution_service.rs`

```rust
use std::collections::HashMap;
use tokio::sync::RwLock;

pub struct ChallengeResolutionService {
    // ... existing fields ...

    /// Transient storage for ad-hoc challenges (not persisted to DB)
    adhoc_challenges: RwLock<HashMap<SessionId, HashMap<ChallengeId, Challenge>>>,
}

impl ChallengeResolutionService {
    /// Register an ad-hoc challenge for the session
    pub async fn register_adhoc_challenge(
        &self,
        session_id: &SessionId,
        challenge: Challenge,
    ) -> Result<(), ServiceError> {
        let mut challenges = self.adhoc_challenges.write().await;
        challenges
            .entry(session_id.clone())
            .or_default()
            .insert(challenge.id.clone(), challenge);
        Ok(())
    }

    /// Get challenge by ID - checks both DB and ad-hoc storage
    pub async fn get_challenge(
        &self,
        challenge_id: &ChallengeId,
        session_id: &SessionId,
    ) -> Result<Option<Challenge>, ServiceError> {
        // First check ad-hoc storage
        let adhoc = self.adhoc_challenges.read().await;
        if let Some(session_challenges) = adhoc.get(session_id) {
            if let Some(challenge) = session_challenges.get(challenge_id) {
                return Ok(Some(challenge.clone()));
            }
        }
        drop(adhoc);

        // Fall back to database
        self.challenge_service.get_challenge(challenge_id).await
    }
}
```

#### Task 22E.3: Helper for Difficulty Parsing

```rust
fn parse_difficulty_string(s: &str) -> Result<Difficulty, ServiceError> {
    // Try DC format: "DC 15", "dc15", "DC:15"
    if let Some(dc_str) = s.to_uppercase()
        .strip_prefix("DC")
        .map(|s| s.trim_start_matches(|c| c == ' ' || c == ':'))
    {
        if let Ok(dc) = dc_str.trim().parse::<u32>() {
            return Ok(Difficulty::DC(dc));
        }
    }

    // Try percentage format: "75%", "75"
    if let Some(pct_str) = s.strip_suffix('%') {
        if let Ok(pct) = pct_str.trim().parse::<u32>() {
            return Ok(Difficulty::Percentage(pct));
        }
    }

    // Try descriptor
    let lower = s.to_lowercase();
    match lower.as_str() {
        "trivial" | "easy" | "routine" | "moderate" | "challenging" |
        "hard" | "very hard" | "extreme" | "impossible" | "risky" | "desperate" => {
            Ok(Difficulty::Descriptor(DifficultyDescriptor::from_str(&lower)?))
        }
        _ => Ok(Difficulty::Custom(s.to_string())),
    }
}
```

#### Acceptance Criteria 22E

- [ ] CreateAdHocChallenge creates transient challenge
- [ ] Challenge prompt sent to target PC
- [ ] AdHocChallengeCreated confirmation sent to DM
- [ ] Roll resolution works for ad-hoc challenges
- [ ] Ad-hoc challenges use DM-provided outcomes

---

### Phase 22F: Enhanced Approval UI (Player)

**Status**: [ ] Not Started
**Estimated Effort**: 3-4 hours
**Files to Modify**:
- `Player/src/presentation/components/dm_panel/approval_popup.rs`
- `Player/src/presentation/handlers/session_message_handler.rs`

#### Task 22F.1: Add Outcome Display with Tabs

**File**: `Player/src/presentation/components/dm_panel/approval_popup.rs`

The component already has `OutcomeDetailsSection` and `OutcomeTab`. Enhance:

```rust
/// Individual outcome tab with editing capability
#[component]
fn OutcomeTab(
    label: String,
    label_color: String,
    outcome: OutcomeDetailData,
    request_id: String,
    outcome_type: String,
    on_regenerate: EventHandler<RegenerateOutcomeData>,
    on_text_change: Option<EventHandler<(String, String)>>,  // NEW: (field, value)
) -> Element {
    let mut is_expanded = use_signal(|| false);
    let mut is_editing = use_signal(|| false);
    let mut edited_flavor = use_signal(|| outcome.flavor_text.clone());
    let mut edited_direction = use_signal(|| outcome.scene_direction.clone());

    rsx! {
        div {
            class: "outcome-tab",

            // Header with expand/collapse
            button {
                onclick: move |_| {
                    let current = *is_expanded.read();
                    is_expanded.set(!current);
                },
                class: "outcome-header",

                span { style: "color: {label_color};", "{label}" }
                span { class: "expand-icon", if *is_expanded.read() { "▼" } else { "▶" } }
            }

            // Content (when expanded)
            if *is_expanded.read() {
                div {
                    class: "outcome-content",

                    // Flavor text
                    div {
                        class: "outcome-field",
                        label { "Flavor Text" }
                        if *is_editing.read() {
                            textarea {
                                value: "{edited_flavor}",
                                oninput: move |e| edited_flavor.set(e.value()),
                            }
                        } else {
                            p { "{outcome.flavor_text}" }
                        }
                    }

                    // Scene direction
                    div {
                        class: "outcome-field",
                        label { "Scene Direction" }
                        if *is_editing.read() {
                            textarea {
                                value: "{edited_direction}",
                                oninput: move |e| edited_direction.set(e.value()),
                            }
                        } else {
                            p { "{outcome.scene_direction}" }
                        }
                    }

                    // Tool receipts (read-only)
                    if !outcome.proposed_tools.is_empty() {
                        div {
                            class: "tool-receipts",
                            h4 { "Will Execute:" }
                            for tool in outcome.proposed_tools.iter() {
                                div {
                                    class: "tool-receipt",
                                    span { class: "tool-name", "{tool.name}" }
                                    span { class: "tool-desc", "{tool.description}" }
                                }
                            }
                        }
                    }

                    // Action buttons
                    div {
                        class: "outcome-actions",

                        // Edit/Save toggle
                        button {
                            onclick: move |_| {
                                let editing = *is_editing.read();
                                if editing {
                                    // Save changes
                                    if let Some(handler) = &on_text_change {
                                        handler.call((
                                            format!("{}_flavor", outcome_type),
                                            edited_flavor.read().clone()
                                        ));
                                        handler.call((
                                            format!("{}_direction", outcome_type),
                                            edited_direction.read().clone()
                                        ));
                                    }
                                }
                                is_editing.set(!editing);
                            },
                            if *is_editing.read() { "Save" } else { "Edit" }
                        }

                        // Regenerate button
                        button {
                            onclick: move |_| {
                                on_regenerate.call(RegenerateOutcomeData {
                                    request_id: request_id.clone(),
                                    outcome_type: Some(outcome_type.clone()),
                                    guidance: None,
                                });
                            },
                            "Regenerate"
                        }
                    }
                }
            }
        }
    }
}
```

#### Task 22F.2: Handle OutcomeRegenerated Message

**File**: `Player/src/presentation/handlers/session_message_handler.rs`

Replace the TODO:

```rust
ServerMessage::OutcomeRegenerated {
    request_id,
    outcome_type,
    new_outcome,
} => {
    tracing::info!(
        "Outcome '{}' regenerated for request {}: {}",
        outcome_type,
        request_id,
        new_outcome.flavor_text
    );

    // Update the pending approval with regenerated outcome
    let mut approvals = session_state.pending_approvals.write();
    if let Some(approval) = approvals.iter_mut().find(|a| a.request_id == request_id) {
        if let Some(ref mut outcomes) = approval.challenge_outcomes {
            match outcome_type.as_str() {
                "success" => outcomes.success = Some(new_outcome),
                "failure" => outcomes.failure = Some(new_outcome),
                "critical_success" => outcomes.critical_success = Some(new_outcome),
                "critical_failure" => outcomes.critical_failure = Some(new_outcome),
                "all" => {
                    // If all were regenerated, new_outcome contains all
                    // This would need a different response structure
                }
                _ => {}
            }
        }
    }
}
```

#### Task 22F.3: Handle ChallengeDiscarded Message

```rust
ServerMessage::ChallengeDiscarded { request_id } => {
    tracing::info!("Challenge discarded for request {}", request_id);

    // Remove challenge suggestion from pending approval
    let mut approvals = session_state.pending_approvals.write();
    if let Some(approval) = approvals.iter_mut().find(|a| a.request_id == request_id) {
        approval.challenge_suggestion = None;
        approval.challenge_outcomes = None;
    }
}
```

#### Acceptance Criteria 22F

- [ ] Outcome tabs display flavor text and scene direction
- [ ] Tool receipts shown for each outcome
- [ ] Edit button enables inline editing
- [ ] Regenerate button sends RegenerateOutcome message
- [ ] OutcomeRegenerated updates UI in place
- [ ] ChallengeDiscarded removes challenge from approval

---

### Phase 22G: Wire Ad-hoc Challenge Modal (Player)

**Status**: [ ] Not Started
**Estimated Effort**: 1-2 hours
**Files to Modify**:
- `Player/src/presentation/views/dm_view.rs`
- `Player/src/presentation/components/dm_panel/adhoc_challenge_modal.rs`

#### Task 22G.1: Add Button to Director Mode

**File**: `Player/src/presentation/views/dm_view.rs`

Find the appropriate location in Director tab and add:

```rust
// In Director mode toolbar or challenge section
button {
    onclick: move |_| show_adhoc_modal.set(true),
    class: "btn-adhoc-challenge",
    "Create Ad-hoc Challenge"
}

// Modal (conditional render)
if *show_adhoc_modal.read() {
    AdHocChallengeModal {
        player_characters: player_characters.clone(),
        on_create: move |data: AdHocChallengeData| {
            // Send WebSocket message
            let msg = ClientMessage::CreateAdHocChallenge {
                challenge_name: data.challenge_name,
                skill_name: data.skill_name,
                difficulty: data.difficulty,
                target_pc_id: data.target_pc_id,
                outcomes: data.outcomes,
            };
            session_service.send_message(msg);
            show_adhoc_modal.set(false);
        },
        on_close: move |_| show_adhoc_modal.set(false),
    }
}
```

#### Task 22G.2: Handle AdHocChallengeCreated Confirmation

**File**: `Player/src/presentation/handlers/session_message_handler.rs`

```rust
ServerMessage::AdHocChallengeCreated {
    challenge_id,
    challenge_name,
    target_pc_id,
} => {
    tracing::info!(
        "Ad-hoc challenge '{}' (ID: {}) created for PC {}",
        challenge_name,
        challenge_id,
        target_pc_id
    );

    // Show confirmation toast/notification
    session_state.add_log_entry(
        "System".to_string(),
        format!("Created ad-hoc challenge: {} for {}", challenge_name, target_pc_id),
        true,
        platform,
    );

    // Could also add to a "recent challenges" list in UI
}
```

#### Acceptance Criteria 22G

- [ ] "Create Ad-hoc Challenge" button visible in Director mode
- [ ] AdHocChallengeModal opens when clicked
- [ ] Form submits CreateAdHocChallenge message
- [ ] Modal closes after submission
- [ ] Confirmation appears in log/toast

---

## Testing Checklist

### End-to-End Flow Test

1. [ ] **Setup**: Start Engine + Player, connect as DM and Player
2. [ ] **PC Action**: Player talks to NPC
3. [ ] **LLM Suggestion**: Engine suggests challenge (verify in DM approval)
4. [ ] **DM Approval**: Accept challenge with modification
5. [ ] **Dice Roll**: Player enters "1d20+3", verify modifier applied
6. [ ] **Resolution**: Verify correct outcome selected (DC comparison)
7. [ ] **Outcome Triggers**: Verify state changes execute
8. [ ] **Broadcast**: All clients receive ChallengeResolved

### Ad-hoc Challenge Test

1. [ ] **Create**: DM creates ad-hoc challenge via modal
2. [ ] **Prompt**: Player receives ChallengePrompt
3. [ ] **Roll**: Player rolls dice
4. [ ] **Outcome**: DM-provided outcome text displays

### Regeneration Test

1. [ ] **Trigger**: DM clicks "Regenerate" on outcome
2. [ ] **LLM Call**: New outcome generated
3. [ ] **UI Update**: Approval popup updates in place
4. [ ] **Discard**: DM discards challenge, verify removed

---

## File Summary

### Engine Files to Create

| File | Purpose |
|------|---------|
| `src/application/services/outcome_trigger_service.rs` | Execute OutcomeTriggers |
| `src/application/services/player_character_service.rs` | Skill modifier lookup (if not exists) |

### Engine Files to Modify

| File | Changes |
|------|---------|
| `src/domain/value_objects/game_tools.rs` | Add 7 new GameTool variants |
| `src/application/services/tool_execution_service.rs` | Implement new tool executions |
| `src/application/services/challenge_resolution_service.rs` | Character modifiers, outcome triggers |
| `src/application/services/llm_service.rs` | Regeneration method |
| `src/infrastructure/websocket.rs` | RegenerateOutcome, DiscardChallenge, CreateAdHocChallenge handlers |

### Player Files to Modify

| File | Changes |
|------|---------|
| `src/presentation/components/dm_panel/approval_popup.rs` | Editing capability |
| `src/presentation/handlers/session_message_handler.rs` | Handle regeneration messages |
| `src/presentation/views/dm_view.rs` | Ad-hoc challenge button |

---

## Success Criteria

When Phase 22 is complete:

1. [ ] Player can enter "1d20+5" or manual result "18"
2. [ ] Character sheet skill bonuses applied automatically
3. [ ] DM can edit outcome text inline
4. [ ] DM can regenerate specific outcome or all outcomes
5. [ ] DM can discard challenge and request non-challenge response
6. [ ] Tool effects persist to Neo4j immediately
7. [ ] DM can create challenges without LLM (ad-hoc)
8. [ ] Outcome triggers execute after resolution
9. [ ] All state changes broadcast to clients
10. [ ] Engine and Player compile with no new warnings

---

## Appendix: WebSocket Message Reference

### Client → Server

```rust
// Existing (already working)
ChallengeSuggestionDecision { request_id, approved, modified_difficulty }
ChallengeRollInput { challenge_id, input_type: DiceInputType }
TriggerChallenge { challenge_id, target_character_id }

// Phase 22 (need implementation)
RegenerateOutcome { request_id, outcome_type, guidance }
DiscardChallenge { request_id, feedback }
CreateAdHocChallenge { challenge_name, skill_name, difficulty, target_pc_id, outcomes }
```

### Server → Client

```rust
// Existing (already working)
ApprovalRequired { request_id, npc_name, ..., challenge_suggestion }
ChallengePrompt { challenge_id, challenge_name, skill_name, difficulty_display, ... }
ChallengeResolved { challenge_id, challenge_name, character_name, roll, modifier, total, outcome, ... }

// Phase 22 (need implementation)
OutcomeRegenerated { request_id, outcome_type, new_outcome }
ChallengeDiscarded { request_id }
AdHocChallengeCreated { challenge_id, challenge_name, target_pc_id }
```
