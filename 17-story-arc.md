# Phase 17: Story Arc Feature

## Overview

The Story Arc feature introduces two complementary systems for comprehensive narrative tracking and design in WrldBldr:

1. **StoryEvent (Past Events Timeline)** - Automatic, immutable logging of ALL gameplay events as they occur
2. **NarrativeEvent (Future Events/Hooks)** - DM-designed events with trigger conditions, branching outcomes, and chain connections

**Key Concept**: Worlds act as game sessions. The Story Arc tracks the complete history of what happened AND allows DMs to design what could happen based on player actions and game state.

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Event Logging | Log everything | Complete history is valuable; DM can hide minor events from view |
| UI Location | New "Story Arc" 4th tab + Director widget | Dedicated space for management, quick access in Director mode |
| Chain Complexity | Full branching from start | Campaign narratives naturally branch; design for real use cases |
| Act Integration | Optional association | Flexibility - events can optionally link to Monomyth stages |

---

## Domain Model

### New ID Types

**File**: `Engine/src/domain/value_objects/ids.rs`

Add to existing ID definitions:
```rust
define_id!(StoryEventId);
define_id!(NarrativeEventId);
define_id!(EventChainId);
```

---

### StoryEvent Entity (Past Events)

**File**: `Engine/src/domain/entities/story_event.rs` (NEW)

This entity represents an immutable record of something that happened during gameplay. Events are automatically created by the system when actions occur.

```rust
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

use crate::domain::value_objects::{
    StoryEventId, WorldId, SessionId, SceneId, LocationId,
    CharacterId, ChallengeId, NarrativeEventId
};

/// A story event - an immutable record of something that happened
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StoryEvent {
    pub id: StoryEventId,
    pub world_id: WorldId,
    pub session_id: SessionId,
    /// Scene where event occurred (if applicable)
    pub scene_id: Option<SceneId>,
    /// Location where event occurred (if applicable)
    pub location_id: Option<LocationId>,
    /// The type and details of the event
    pub event_type: StoryEventType,
    /// When this event occurred (real-world timestamp)
    pub timestamp: DateTime<Utc>,
    /// In-game time context (optional, e.g., "Day 3, Evening")
    pub game_time: Option<String>,
    /// Narrative summary (auto-generated or DM-edited)
    pub summary: String,
    /// Characters involved in this event
    pub involved_characters: Vec<CharacterId>,
    /// Whether this event is hidden from timeline UI (but still tracked)
    pub is_hidden: bool,
    /// Tags for filtering/searching
    pub tags: Vec<String>,
    /// Optional link to causative narrative event
    pub triggered_by: Option<NarrativeEventId>,
}

/// Categories of story events that occurred during gameplay
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum StoryEventType {
    /// Player character moved to a new location
    LocationChange {
        from_location: Option<LocationId>,
        to_location: LocationId,
        character_id: CharacterId,
        travel_method: Option<String>,
    },

    /// Dialogue exchange with an NPC
    DialogueExchange {
        npc_id: CharacterId,
        player_dialogue: String,
        npc_response: String,
        topics_discussed: Vec<String>,
        tone: Option<String>,
    },

    /// Combat encounter started or completed
    CombatEvent {
        combat_type: CombatEventType,
        participants: Vec<CharacterId>,
        enemies: Vec<String>,
        outcome: Option<CombatOutcome>,
        location_id: LocationId,
        rounds: Option<u32>,
    },

    /// Challenge attempted (skill check, saving throw, etc.)
    ChallengeAttempted {
        challenge_id: Option<ChallengeId>,
        challenge_name: String,
        character_id: CharacterId,
        skill_used: Option<String>,
        difficulty: Option<String>,
        roll_result: Option<i32>,
        modifier: Option<i32>,
        outcome: ChallengeEventOutcome,
    },

    /// Item acquired by a character
    ItemAcquired {
        item_name: String,
        item_description: Option<String>,
        character_id: CharacterId,
        source: ItemSource,
        quantity: u32,
    },

    /// Item transferred between characters
    ItemTransferred {
        item_name: String,
        from_character: Option<CharacterId>,
        to_character: CharacterId,
        quantity: u32,
        reason: Option<String>,
    },

    /// Item used or consumed
    ItemUsed {
        item_name: String,
        character_id: CharacterId,
        target: Option<String>,
        effect: String,
        consumed: bool,
    },

    /// Relationship changed between characters
    RelationshipChanged {
        from_character: CharacterId,
        to_character: CharacterId,
        previous_sentiment: Option<f32>,
        new_sentiment: f32,
        sentiment_change: f32,
        reason: String,
    },

    /// Scene transition occurred
    SceneTransition {
        from_scene: Option<SceneId>,
        to_scene: SceneId,
        from_scene_name: Option<String>,
        to_scene_name: String,
        trigger_reason: String,
    },

    /// Information revealed to players
    InformationRevealed {
        info_type: InfoType,
        title: String,
        content: String,
        source: Option<CharacterId>,
        importance: InfoImportance,
        persist_to_journal: bool,
    },

    /// NPC performed an action through LLM tool call
    NpcAction {
        npc_id: CharacterId,
        npc_name: String,
        action_type: String,
        description: String,
        dm_approved: bool,
        dm_modified: bool,
    },

    /// DM manually added narrative marker/note
    DmMarker {
        title: String,
        note: String,
        importance: MarkerImportance,
        marker_type: DmMarkerType,
    },

    /// Narrative event was triggered
    NarrativeEventTriggered {
        narrative_event_id: NarrativeEventId,
        narrative_event_name: String,
        outcome_branch: Option<String>,
        effects_applied: Vec<String>,
    },

    /// Character stat was modified
    StatModified {
        character_id: CharacterId,
        stat_name: String,
        previous_value: i32,
        new_value: i32,
        reason: String,
    },

    /// Flag was set or unset
    FlagChanged {
        flag_name: String,
        new_value: bool,
        reason: String,
    },

    /// Session started
    SessionStarted {
        session_number: u32,
        session_name: Option<String>,
        players_present: Vec<String>,
    },

    /// Session ended
    SessionEnded {
        duration_minutes: u32,
        summary: String,
    },

    /// Custom event type for extensibility
    Custom {
        event_subtype: String,
        title: String,
        description: String,
        data: serde_json::Value,
    },
}

/// Combat event subtypes
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum CombatEventType {
    Started,
    RoundCompleted,
    CharacterDefeated,
    CharacterFled,
    Ended,
}

/// Combat outcome types
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum CombatOutcome {
    Victory,
    Defeat,
    Fled,
    Negotiated,
    Draw,
    Interrupted,
}

/// Challenge event outcome
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum ChallengeEventOutcome {
    CriticalSuccess,
    Success,
    PartialSuccess,
    Failure,
    CriticalFailure,
}

/// Source of an acquired item
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(tag = "source_type", rename_all = "snake_case")]
pub enum ItemSource {
    Found { location: String },
    Purchased { from: String, cost: Option<String> },
    Gifted { from: CharacterId },
    Looted { from: String },
    Crafted,
    Reward { for_what: String },
    Stolen { from: String },
    Custom { description: String },
}

/// Type of revealed information
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum InfoType {
    Lore,
    Quest,
    Character,
    Location,
    Item,
    Secret,
    Rumor,
}

/// Importance level for revealed information
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum InfoImportance {
    Minor,
    Notable,
    Major,
    Critical,
}

/// Importance level for DM markers
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum MarkerImportance {
    Minor,
    Notable,
    Major,
    Critical,
}

/// Types of DM markers
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum DmMarkerType {
    Note,
    PlotPoint,
    CharacterMoment,
    WorldEvent,
    PlayerDecision,
    Foreshadowing,
    Callback,
    Custom,
}

impl StoryEvent {
    pub fn new(
        world_id: WorldId,
        session_id: SessionId,
        event_type: StoryEventType,
    ) -> Self {
        Self {
            id: StoryEventId::new(),
            world_id,
            session_id,
            scene_id: None,
            location_id: None,
            event_type,
            timestamp: Utc::now(),
            game_time: None,
            summary: String::new(),
            involved_characters: Vec::new(),
            is_hidden: false,
            tags: Vec::new(),
            triggered_by: None,
        }
    }

    pub fn with_scene(mut self, scene_id: SceneId) -> Self {
        self.scene_id = Some(scene_id);
        self
    }

    pub fn with_location(mut self, location_id: LocationId) -> Self {
        self.location_id = Some(location_id);
        self
    }

    pub fn with_game_time(mut self, game_time: impl Into<String>) -> Self {
        self.game_time = Some(game_time.into());
        self
    }

    pub fn with_summary(mut self, summary: impl Into<String>) -> Self {
        self.summary = summary.into();
        self
    }

    pub fn with_characters(mut self, characters: Vec<CharacterId>) -> Self {
        self.involved_characters = characters;
        self
    }

    pub fn with_tag(mut self, tag: impl Into<String>) -> Self {
        self.tags.push(tag.into());
        self
    }

    pub fn hidden(mut self) -> Self {
        self.is_hidden = true;
        self
    }

    pub fn triggered_by(mut self, event_id: NarrativeEventId) -> Self {
        self.triggered_by = Some(event_id);
        self
    }

    /// Generate an automatic summary based on event type
    pub fn auto_summarize(&mut self) {
        self.summary = match &self.event_type {
            StoryEventType::LocationChange { .. } => "Traveled to a new location".to_string(),
            StoryEventType::DialogueExchange { npc_name, .. } =>
                format!("Spoke with {}", npc_name.unwrap_or("an NPC".to_string())),
            StoryEventType::CombatEvent { combat_type: CombatEventType::Started, .. } =>
                "Combat began".to_string(),
            StoryEventType::CombatEvent { outcome: Some(CombatOutcome::Victory), .. } =>
                "Won the battle".to_string(),
            StoryEventType::ChallengeAttempted { challenge_name, outcome, .. } =>
                format!("{}: {:?}", challenge_name, outcome),
            StoryEventType::ItemAcquired { item_name, .. } =>
                format!("Acquired {}", item_name),
            StoryEventType::DmMarker { title, .. } => title.clone(),
            _ => "Event occurred".to_string(),
        };
    }
}
```

---

### NarrativeEvent Entity (Future Events/Hooks)

**File**: `Engine/src/domain/entities/narrative_event.rs` (NEW)

This entity represents a DM-designed event that can trigger when conditions are met. It supports complex triggers, branching outcomes, and chaining to other events.

```rust
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

use crate::domain::value_objects::{
    NarrativeEventId, EventChainId, WorldId, ActId, SceneId,
    LocationId, CharacterId, ChallengeId
};

/// A narrative event that can be triggered when conditions are met
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NarrativeEvent {
    pub id: NarrativeEventId,
    pub world_id: WorldId,

    // Basic Info
    /// Name of the event (for DM reference)
    pub name: String,
    /// Detailed description of what this event represents
    pub description: String,
    /// Tags for organization and filtering
    pub tags: Vec<String>,

    // Trigger Configuration
    /// Conditions that must be met to trigger this event
    pub trigger_conditions: Vec<NarrativeTrigger>,
    /// How multiple conditions are evaluated
    pub trigger_logic: TriggerLogic,

    // Scene Direction
    /// Narrative text shown to DM when event triggers (sets the scene)
    pub scene_direction: String,
    /// Suggested opening dialogue or action
    pub suggested_opening: Option<String>,
    /// NPCs that should be featured in this event
    pub featured_npcs: Vec<CharacterId>,

    // Outcomes
    /// Possible outcomes with their effects and chains
    pub outcomes: Vec<EventOutcome>,
    /// Default outcome if DM doesn't select one
    pub default_outcome: Option<String>,

    // Status
    /// Whether this event is currently active (can be triggered)
    pub is_active: bool,
    /// Whether this event has already been triggered
    pub is_triggered: bool,
    /// Timestamp when triggered (if triggered)
    pub triggered_at: Option<DateTime<Utc>>,
    /// Which outcome was selected (if triggered)
    pub selected_outcome: Option<String>,
    /// Whether this event can repeat (trigger multiple times)
    pub is_repeatable: bool,
    /// Times this event has been triggered (for repeatable events)
    pub trigger_count: u32,

    // Timing
    /// Optional delay before event actually fires (in turns/exchanges)
    pub delay_turns: u32,
    /// Expiration - event becomes inactive after this (optional)
    pub expires_after_turns: Option<u32>,

    // Associations
    /// Scene this event is tied to (if any)
    pub scene_id: Option<SceneId>,
    /// Location this event is tied to (if any)
    pub location_id: Option<LocationId>,
    /// Act this event belongs to (optional Monomyth integration)
    pub act_id: Option<ActId>,

    // Organization
    /// Priority for ordering multiple triggered events (higher = first)
    pub priority: i32,
    /// Is this a favorite for quick access
    pub is_favorite: bool,

    // Chain Membership
    /// Part of an event chain
    pub chain_id: Option<EventChainId>,
    /// Position in chain (if part of one)
    pub chain_position: Option<u32>,

    // Metadata
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

/// How multiple trigger conditions are evaluated
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq, Default)]
pub enum TriggerLogic {
    /// All conditions must be met (AND)
    #[default]
    All,
    /// Any single condition can trigger (OR)
    Any,
    /// At least N conditions must be met
    AtLeast(u32),
}

/// A single trigger condition
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NarrativeTrigger {
    /// The type and parameters of this trigger
    pub trigger_type: NarrativeTriggerType,
    /// Human-readable description for DM
    pub description: String,
    /// Whether this specific condition must be met (for AtLeast logic)
    pub is_required: bool,
    /// Unique identifier for this trigger within the event
    pub trigger_id: String,
}

/// Types of triggers for narrative events
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum NarrativeTriggerType {
    /// NPC performs a specific action or completes dialogue
    NpcAction {
        npc_id: CharacterId,
        npc_name: String,
        action_keywords: Vec<String>,
        action_description: String,
    },

    /// Player enters a specific location
    PlayerEntersLocation {
        location_id: LocationId,
        location_name: String,
    },

    /// Player is at location during specific time
    TimeAtLocation {
        location_id: LocationId,
        location_name: String,
        time_context: String,
    },

    /// Specific dialogue topic is discussed
    DialogueTopic {
        keywords: Vec<String>,
        with_npc: Option<CharacterId>,
        npc_name: Option<String>,
    },

    /// Challenge is completed
    ChallengeCompleted {
        challenge_id: ChallengeId,
        challenge_name: String,
        requires_success: Option<bool>,
    },

    /// Relationship reaches a threshold
    RelationshipThreshold {
        character_id: CharacterId,
        character_name: String,
        with_character: CharacterId,
        with_character_name: String,
        min_sentiment: Option<f32>,
        max_sentiment: Option<f32>,
    },

    /// Player has specific item
    HasItem {
        item_name: String,
        quantity: Option<u32>,
    },

    /// Player does NOT have specific item
    MissingItem {
        item_name: String,
    },

    /// Another narrative event was completed
    EventCompleted {
        event_id: NarrativeEventId,
        event_name: String,
        outcome_name: Option<String>,
    },

    /// Turn count reached (since session start or since another event)
    TurnCount {
        turns: u32,
        since_event: Option<NarrativeEventId>,
    },

    /// Game flag is set to true
    FlagSet {
        flag_name: String,
    },

    /// Game flag is not set (or false)
    FlagNotSet {
        flag_name: String,
    },

    /// Character stat meets threshold
    StatThreshold {
        character_id: CharacterId,
        stat_name: String,
        min_value: Option<i32>,
        max_value: Option<i32>,
    },

    /// Combat ended with specific result
    CombatResult {
        victory: Option<bool>,
        involved_npc: Option<CharacterId>,
    },

    /// Custom condition (LLM evaluates based on description)
    Custom {
        description: String,
        /// If true, LLM will evaluate this condition against current context
        llm_evaluation: bool,
    },
}

/// An outcome branch for a narrative event
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventOutcome {
    /// Unique identifier for this outcome within the event
    pub name: String,
    /// Display label for DM
    pub label: String,
    /// Description of what happens in this outcome
    pub description: String,
    /// Conditions for this outcome (how does player reach this?)
    pub condition: Option<OutcomeCondition>,
    /// Effects that occur when this outcome happens
    pub effects: Vec<EventEffect>,
    /// Narrative events to chain to after this outcome
    pub chain_events: Vec<ChainedEvent>,
    /// Narrative summary to add to timeline
    pub timeline_summary: Option<String>,
}

/// Condition for an outcome branch
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum OutcomeCondition {
    /// DM selects this outcome manually
    DmChoice,

    /// Challenge result determines outcome
    ChallengeResult {
        challenge_id: Option<ChallengeId>,
        success_required: bool,
    },

    /// Combat result determines outcome
    CombatResult {
        victory_required: bool,
    },

    /// Specific dialogue choice made
    DialogueChoice {
        keywords: Vec<String>,
    },

    /// Player takes specific action
    PlayerAction {
        action_keywords: Vec<String>,
    },

    /// Player has item
    HasItem {
        item_name: String,
    },

    /// Custom condition (LLM evaluates)
    Custom {
        description: String,
    },
}

/// Effects that occur as part of an event outcome
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "effect_type", rename_all = "snake_case")]
pub enum EventEffect {
    /// Change relationship between characters
    ModifyRelationship {
        from_character: CharacterId,
        from_name: String,
        to_character: CharacterId,
        to_name: String,
        sentiment_change: f32,
        reason: String,
    },

    /// Give item to player
    GiveItem {
        item_name: String,
        item_description: Option<String>,
        quantity: u32,
    },

    /// Take item from player
    TakeItem {
        item_name: String,
        quantity: u32,
    },

    /// Reveal information to players
    RevealInformation {
        info_type: String,
        title: String,
        content: String,
        persist_to_journal: bool,
    },

    /// Set a game flag
    SetFlag {
        flag_name: String,
        value: bool,
    },

    /// Enable a challenge
    EnableChallenge {
        challenge_id: ChallengeId,
        challenge_name: String,
    },

    /// Disable a challenge
    DisableChallenge {
        challenge_id: ChallengeId,
        challenge_name: String,
    },

    /// Enable another narrative event
    EnableEvent {
        event_id: NarrativeEventId,
        event_name: String,
    },

    /// Disable another narrative event
    DisableEvent {
        event_id: NarrativeEventId,
        event_name: String,
    },

    /// Trigger scene transition
    TriggerScene {
        scene_id: SceneId,
        scene_name: String,
    },

    /// Start combat encounter
    StartCombat {
        participants: Vec<CharacterId>,
        participant_names: Vec<String>,
        combat_description: String,
    },

    /// Modify character stat
    ModifyStat {
        character_id: CharacterId,
        character_name: String,
        stat_name: String,
        modifier: i32,
    },

    /// Add experience/reward
    AddReward {
        reward_type: String,
        amount: i32,
        description: String,
    },

    /// Custom effect (description for DM/LLM)
    Custom {
        description: String,
        requires_dm_action: bool,
    },
}

/// Reference to a chained event
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChainedEvent {
    /// Event to chain to
    pub event_id: NarrativeEventId,
    /// Name for display
    pub event_name: String,
    /// Delay before chain activates (turns)
    pub delay_turns: u32,
    /// Additional trigger condition for chain (beyond just completing parent)
    pub additional_trigger: Option<NarrativeTriggerType>,
    /// Description of why this chains
    pub chain_reason: Option<String>,
}

impl NarrativeEvent {
    pub fn new(world_id: WorldId, name: impl Into<String>) -> Self {
        let now = Utc::now();
        Self {
            id: NarrativeEventId::new(),
            world_id,
            name: name.into(),
            description: String::new(),
            tags: Vec::new(),
            trigger_conditions: Vec::new(),
            trigger_logic: TriggerLogic::All,
            scene_direction: String::new(),
            suggested_opening: None,
            featured_npcs: Vec::new(),
            outcomes: Vec::new(),
            default_outcome: None,
            is_active: true,
            is_triggered: false,
            triggered_at: None,
            selected_outcome: None,
            is_repeatable: false,
            trigger_count: 0,
            delay_turns: 0,
            expires_after_turns: None,
            scene_id: None,
            location_id: None,
            act_id: None,
            priority: 0,
            is_favorite: false,
            chain_id: None,
            chain_position: None,
            created_at: now,
            updated_at: now,
        }
    }

    /// Check if this event's triggers match the current game context
    pub fn evaluate_triggers(&self, context: &TriggerContext) -> TriggerEvaluation {
        let mut matched = Vec::new();
        let mut unmatched = Vec::new();

        for trigger in &self.trigger_conditions {
            if self.trigger_matches(&trigger.trigger_type, context) {
                matched.push(trigger.trigger_id.clone());
            } else {
                unmatched.push(trigger.trigger_id.clone());
            }
        }

        let total = self.trigger_conditions.len();
        let matched_count = matched.len();

        let is_triggered = match self.trigger_logic {
            TriggerLogic::All => matched_count == total,
            TriggerLogic::Any => matched_count > 0,
            TriggerLogic::AtLeast(n) => matched_count >= n as usize,
        };

        // Check required triggers
        let required_met = self.trigger_conditions
            .iter()
            .filter(|t| t.is_required)
            .all(|t| matched.contains(&t.trigger_id));

        TriggerEvaluation {
            is_triggered: is_triggered && required_met,
            matched_triggers: matched,
            unmatched_triggers: unmatched,
            total_triggers: total,
            confidence: matched_count as f32 / total as f32,
        }
    }

    fn trigger_matches(&self, trigger: &NarrativeTriggerType, context: &TriggerContext) -> bool {
        // Implementation would check each trigger type against context
        // This is a simplified placeholder
        match trigger {
            NarrativeTriggerType::FlagSet { flag_name } => {
                context.flags.get(flag_name).copied().unwrap_or(false)
            }
            NarrativeTriggerType::FlagNotSet { flag_name } => {
                !context.flags.get(flag_name).copied().unwrap_or(false)
            }
            NarrativeTriggerType::PlayerEntersLocation { location_id, .. } => {
                context.current_location.as_ref() == Some(location_id)
            }
            // ... other trigger type checks
            _ => false,
        }
    }
}

/// Context for evaluating triggers
#[derive(Debug, Clone, Default)]
pub struct TriggerContext {
    pub current_location: Option<LocationId>,
    pub current_scene: Option<SceneId>,
    pub time_context: Option<String>,
    pub flags: std::collections::HashMap<String, bool>,
    pub inventory: Vec<String>,
    pub completed_events: Vec<NarrativeEventId>,
    pub completed_challenges: Vec<ChallengeId>,
    pub turn_count: u32,
    pub recent_dialogue_topics: Vec<String>,
    pub recent_player_action: Option<String>,
}

/// Result of trigger evaluation
#[derive(Debug, Clone)]
pub struct TriggerEvaluation {
    pub is_triggered: bool,
    pub matched_triggers: Vec<String>,
    pub unmatched_triggers: Vec<String>,
    pub total_triggers: usize,
    pub confidence: f32,
}
```

---

### EventChain Entity

**File**: `Engine/src/domain/entities/event_chain.rs` (NEW)

```rust
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

use crate::domain::value_objects::{EventChainId, WorldId, NarrativeEventId, ActId};

/// A chain of connected narrative events forming a story arc
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventChain {
    pub id: EventChainId,
    pub world_id: WorldId,

    /// Name of this story arc/chain
    pub name: String,
    /// Description of what this chain represents narratively
    pub description: String,

    /// IDs of events in this chain (ordered by chain_position)
    pub events: Vec<NarrativeEventId>,

    /// Whether this chain is currently active
    pub is_active: bool,
    /// Current position in the chain (index of next event)
    pub current_position: u32,
    /// Events that have been completed in this chain
    pub completed_events: Vec<NarrativeEventId>,

    /// Optional: Act this chain belongs to
    pub act_id: Option<ActId>,

    /// Tags for organization
    pub tags: Vec<String>,

    /// Color for visualization (hex)
    pub color: Option<String>,

    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

impl EventChain {
    pub fn new(world_id: WorldId, name: impl Into<String>) -> Self {
        let now = Utc::now();
        Self {
            id: EventChainId::new(),
            world_id,
            name: name.into(),
            description: String::new(),
            events: Vec::new(),
            is_active: true,
            current_position: 0,
            completed_events: Vec::new(),
            act_id: None,
            tags: Vec::new(),
            color: None,
            created_at: now,
            updated_at: now,
        }
    }

    /// Get the current (next) event in the chain
    pub fn current_event(&self) -> Option<&NarrativeEventId> {
        self.events.get(self.current_position as usize)
    }

    /// Mark an event as completed and advance if it's current
    pub fn complete_event(&mut self, event_id: &NarrativeEventId) {
        self.completed_events.push(event_id.clone());

        if let Some(current) = self.current_event() {
            if current == event_id {
                self.current_position += 1;
            }
        }
    }

    /// Check if chain is complete
    pub fn is_complete(&self) -> bool {
        self.current_position as usize >= self.events.len()
    }

    /// Get progress as percentage
    pub fn progress(&self) -> f32 {
        if self.events.is_empty() {
            return 0.0;
        }
        self.completed_events.len() as f32 / self.events.len() as f32
    }
}
```

---

## User Stories

### Epic: Past Events Timeline

#### US-17.1: View Story Timeline
**As a** Dungeon Master
**I want to** see a chronological timeline of everything that has happened in my session
**So that** I can quickly recall past events and maintain narrative consistency

**Acceptance Criteria:**
- Timeline displays all StoryEvents in reverse chronological order (newest first)
- Events can be optionally grouped by session, day, or scene
- Each event shows: timestamp, type icon, summary, involved characters
- Events are color-coded by type:
  - Blue: Dialogue
  - Red: Combat
  - Green: Challenge
  - Gold: Items
  - Purple: Narrative Events
  - Gray: Scene Transitions
  - Orange: DM Markers
- Timeline loads with pagination (50 events per page)
- Infinite scroll or "Load More" button
- Total event count shown

---

#### US-17.2: Add DM Marker
**As a** Dungeon Master
**I want to** add custom markers and notes to the timeline
**So that** I can highlight important narrative moments the system didn't automatically capture

**Acceptance Criteria:**
- "Add Marker" button opens inline form or modal
- Form fields:
  - Title (required, max 100 chars)
  - Note (optional, max 2000 chars, supports markdown)
  - Importance (Minor, Notable, Major, Critical)
  - Marker Type (Note, Plot Point, Character Moment, World Event, etc.)
  - Tags (optional, comma-separated)
- Markers appear immediately in timeline with distinctive styling
- Markers can be edited or deleted by DM
- Markers have special icon (flag or star)

---

#### US-17.3: Filter Timeline Events
**As a** Dungeon Master
**I want to** filter the timeline by event type, character, location, and date
**So that** I can quickly find relevant past events

**Acceptance Criteria:**
- Filter bar with following options:
  - Event Type: Multi-select checkboxes for all StoryEventTypes
  - Characters: Dropdown with all characters (NPCs and PCs)
  - Locations: Dropdown with all locations
  - Sessions: Dropdown with session numbers
  - Date Range: From/To date pickers
  - Tags: Text input with autocomplete
- Filters combine with AND logic
- "Clear All Filters" button
- Filter state persisted during session
- URL query params for shareable filtered views
- Event count updates to show "Showing X of Y events"

---

#### US-17.4: Hide/Show Minor Events
**As a** Dungeon Master
**I want to** hide trivial events from the timeline view
**So that** I can focus on significant narrative moments without clutter

**Acceptance Criteria:**
- Each event card has "Hide" action (eye icon)
- Hidden events removed from view but stored in database
- Toggle button: "Show Hidden Events" reveals all
- Bulk hide option: "Hide all [event type]" in filter dropdown
- Auto-hide suggestions for events marked as minor
- Hidden count shown: "X hidden events"

---

#### US-17.5: View Event Details
**As a** Dungeon Master
**I want to** click on a timeline event to see its full details
**So that** I can review exact dialogue, outcomes, and consequences

**Acceptance Criteria:**
- Click event card opens detail modal/panel
- Detail view shows all event metadata:
  - Full timestamp (date, time, game time)
  - Event type with icon
  - Complete description/content
  - All involved characters with portraits and links
  - Location with link
  - Scene context
  - Related narrative event (if triggered by one)
  - All tags
  - Effects applied (for NarrativeEventTriggered)
- For DialogueExchange: Full conversation transcript
- For CombatEvent: Participant list, round count, outcome details
- For ChallengeAttempted: Roll details, modifiers, DC
- Edit button for summary and tags
- "Hide Event" button
- "Copy to Clipboard" for summary

---

### Epic: Narrative Events (Future Hooks)

#### US-17.6: Create Narrative Event
**As a** Dungeon Master
**I want to** create a narrative event that will trigger when specific conditions are met
**So that** I can prepare story beats in advance and have the system alert me when they're relevant

**Acceptance Criteria:**
- "Create Event" button opens Narrative Event Designer modal
- Basic Info section:
  - Name (required)
  - Description (optional, markdown supported)
  - Tags (optional)
- Event saved as draft until explicitly activated
- Validation: Must have at least one trigger condition
- Validation: Must have at least one outcome
- Success feedback with option to create another

---

#### US-17.7: Define Event Triggers
**As a** Dungeon Master
**I want to** define various trigger conditions for my narrative events
**So that** they activate at the right narrative moment

**Acceptance Criteria:**
- Trigger Conditions section in designer
- "Add Condition" button with trigger type dropdown
- Available trigger types (all NarrativeTriggerType variants):
  - NPC Action (select NPC, enter keywords)
  - Player Enters Location (select location)
  - Time at Location (select location + time context)
  - Dialogue Topic (enter keywords, optionally select NPC)
  - Challenge Completed (select challenge, success/failure/any)
  - Relationship Threshold (select characters, min/max sentiment)
  - Has Item (enter item name)
  - Missing Item (enter item name)
  - Event Completed (select another event, optionally outcome)
  - Turn Count (enter number)
  - Flag Set (enter flag name)
  - Flag Not Set (enter flag name)
  - Stat Threshold (select character, stat, min/max)
  - Combat Result (victory/defeat/any)
  - Custom (enter description, toggle LLM evaluation)
- Trigger Logic selector: All (AND) / Any (OR) / At Least N
- Each condition shows human-readable preview
- "Required" toggle for individual conditions (for AtLeast logic)
- Reorder conditions via drag-and-drop
- Delete condition button

---

#### US-17.8: Define Event Outcomes
**As a** Dungeon Master
**I want to** define multiple possible outcomes for an event based on how players respond
**So that** the story can branch naturally

**Acceptance Criteria:**
- Outcomes section in designer
- "Add Outcome" button
- Each outcome has:
  - Name/Identifier (e.g., "victory", "defeat", "negotiated")
  - Display Label (e.g., "Players defeat the goblins")
  - Description of what happens
  - Condition (how players reach this outcome):
    - DM Choice (default)
    - Challenge Result
    - Combat Result
    - Dialogue Choice (keywords)
    - Player Action (keywords)
    - Has Item
    - Custom
  - Effects list (see US-17.8a)
  - Chain Events list (see US-17.9)
  - Timeline Summary (what shows in timeline)
- At least one outcome required
- Default outcome selector (if no explicit choice made)
- Reorder outcomes via drag-and-drop
- Delete outcome button

---

#### US-17.8a: Define Outcome Effects
**As a** Dungeon Master
**I want to** specify effects that occur when an outcome is selected
**So that** the game state updates automatically

**Acceptance Criteria:**
- "Add Effect" button within each outcome
- Available effect types (all EventEffect variants):
  - Modify Relationship (select characters, change amount, reason)
  - Give Item (name, description, quantity)
  - Take Item (name, quantity)
  - Reveal Information (type, title, content, persist toggle)
  - Set Flag (name, true/false)
  - Enable Challenge (select challenge)
  - Disable Challenge (select challenge)
  - Enable Event (select another event)
  - Disable Event (select another event)
  - Trigger Scene (select scene)
  - Start Combat (select participants)
  - Modify Stat (select character, stat, modifier)
  - Add Reward (type, amount, description)
  - Custom (description, requires DM action toggle)
- Each effect shows preview of what will happen
- Effects execute in order when outcome is applied
- Delete effect button

---

#### US-17.9: Create Event Chains
**As a** Dungeon Master
**I want to** link narrative events into chains
**So that** completing one event can set up the next story beat

**Acceptance Criteria:**
- Event Chains view in Story Arc tab
- "Create Chain" button opens chain editor
- Chain has:
  - Name
  - Description
  - Color (for visualization)
  - Tags
  - Act association (optional)
- Add events to chain:
  - Search/select from existing events
  - Or create new event inline
- Visual representation of chain as vertical list
- Drag-and-drop reordering
- Within outcome configuration, "Chain To" section:
  - Select event(s) to chain to
  - Set delay (turns)
  - Add additional trigger condition
  - Add chain reason description
- Chain shows progress (X of Y events completed)
- Chains can branch (multiple events chain from one outcome)

---

#### US-17.10: LLM Event Trigger Detection
**As a** Dungeon Master
**I want** the LLM to notify me when trigger conditions for a narrative event are met
**So that** I can decide whether to activate it at the right moment

**Acceptance Criteria:**
- Engine includes active narrative events in LLM context
- LLM prompt includes:
  - List of active events with their triggers
  - Instructions to detect when triggers match
  - Format for suggesting event triggers
- LLM can include `NARRATIVE_EVENT_TRIGGER: <event_name>` in response
- Engine parses trigger suggestions from LLM response
- Engine validates suggested triggers against actual conditions
- If valid, sends EventTriggerSuggestion to DM via WebSocket
- Suggestion appears in Director view widget
- DM can: Approve / Delay (set turns) / Reject
- Approved events execute and log to timeline
- Confidence score shown (based on trigger match percentage)

---

#### US-17.11: Manual Event Trigger
**As a** Dungeon Master
**I want to** manually trigger any narrative event regardless of its conditions
**So that** I have full creative control over the narrative

**Acceptance Criteria:**
- "Trigger Now" button on each event in library
- Confirmation modal shows:
  - Event name and description
  - Current trigger status (X of Y conditions met)
  - Warning if conditions not met
  - Outcome selector (if multiple outcomes)
- Manual triggers bypass condition checks
- Event marked as triggered with "manual" flag
- StoryEvent logged with manual trigger note
- Effects execute immediately
- Works even for inactive events

---

#### US-17.12: Event Library Management
**As a** Dungeon Master
**I want to** browse, search, and organize my narrative events
**So that** I can quickly find and manage them

**Acceptance Criteria:**
- Narrative Events view in Story Arc tab
- List/Grid view toggle
- Search bar (searches name, description, tags)
- Filter options:
  - Status: Active / Inactive / Triggered / Pending
  - Favorites only
  - By chain
  - By act
  - By tags
- Sort options:
  - Priority (default)
  - Name (A-Z)
  - Created date
  - Last modified
  - Trigger progress
- Each event card shows:
  - Name
  - Status badge (Active/Triggered/Inactive)
  - Trigger progress (e.g., "2/3 conditions")
  - Priority number
  - Tags
  - Quick actions: Edit, Trigger, Enable/Disable, Favorite, Delete
- Bulk actions: Enable all, Disable all, Delete selected
- "Duplicate Event" action
- Event count shown

---

### Epic: UI & Visualization

#### US-17.13: Visualize Event Chains
**As a** Dungeon Master
**I want to** see a visual flowchart of my event chains
**So that** I can understand the narrative structure at a glance

**Acceptance Criteria:**
- Event Chains view shows list of chains
- Click chain to open visualizer
- Flowchart visualization:
  - Events as nodes (boxes)
  - Chains/connections as arrows
  - Branching paths shown clearly
- Node appearance:
  - Color: Chain color or status-based
  - Border: Solid (triggered), Dashed (pending), Gray (inactive)
  - Icon: Checkmark (completed), Clock (pending), X (failed)
- Node content:
  - Event name
  - Status indicator
  - Trigger progress
- Click node to view/edit event
- Hover shows event summary
- Zoom and pan controls
- Fit-to-screen button
- Export as image option
- Legend explaining colors/symbols

---

#### US-17.14: Director View Pending Events Widget
**As a** Dungeon Master
**I want to** see relevant pending events in my Director view
**So that** I don't miss narrative opportunities while directing

**Acceptance Criteria:**
- Widget in Director view right panel (below Quick Actions)
- Title: "Pending Events" with count badge
- Shows 3-5 most relevant events:
  - Events with highest trigger match %
  - Events matching current scene/location
  - Events with LLM suggestions pending
- Each item shows:
  - Event name
  - Trigger progress (e.g., "2/3", "Ready!")
  - Quick actions: Trigger, View, Dismiss
- LLM suggestions highlighted with amber border
- "View in Story Arc" link to full library
- Collapsible to save space
- Empty state: "No pending events"

---

#### US-17.15: Export/Import Events
**As a** Dungeon Master
**I want to** export my narrative events and import them into other worlds
**So that** I can reuse story structures and share with others

**Acceptance Criteria:**
- Export options:
  - Single event as JSON
  - Single chain (with all events) as JSON
  - All events for world as JSON
  - All chains for world as JSON
- Export includes:
  - All event data
  - All chain data
  - Referenced IDs (challenges, scenes, etc.) as names for portability
- Import options:
  - Upload JSON file
  - Paste JSON text
- Import process:
  - Validate JSON structure
  - Show preview of what will be imported
  - Handle ID conflicts (rename or skip)
  - Map referenced entities by name where possible
  - Report unmapped references for manual resolution
- Success report: "Imported X events, Y chains"

---

## UI Mockups

### Story Arc Tab Layout

```
+==============================================================================+
| [WrldBldr]  [Director] [Creator] [Story Arc] [Settings]     [Connected ●]   |
+==============================================================================+
|                                                                              |
| +-- Sub-Navigation -------------------------------------------------------+ |
| | [Timeline]  [Narrative Events]  [Event Chains]                          | |
| +------------------------------------------------------------------------+ |
|                                                                              |
| (Content area changes based on selected sub-nav)                            |
|                                                                              |
+==============================================================================+
```

### Timeline View

```
+==============================================================================+
|                              STORY TIMELINE                                  |
+==============================================================================+
| Filters:                                                                     |
| [All Types ▼] [All Characters ▼] [All Locations ▼] [Session: Current ▼]    |
| [Date Range: _____ to _____]  [Tags: __________]  [Clear Filters]           |
| [Show Hidden: OFF]                              Showing 47 of 156 events    |
+------------------------------------------------------------------------------+
|                                                                              |
| +-- SESSION 3: The Festival Aftermath (Dec 15, 2025) ---------------------+ |
|                                                                              |
| 15:42  ◆ SCENE TRANSITION                                                   |
|        Entered: The Rusty Tankard                                           |
|        From: Town Square                                                    |
|        [View Details]                                            [Hide ◎]  |
|                                                                              |
| 15:38  💬 DIALOGUE                                                          |
|        Marcus spoke with Innkeeper Greta                                    |
|        "We need rooms for the night, and information about the forest..."   |
|        Topics: lodging, forest, rumors                                      |
|        [View Full Dialogue]                                      [Hide ◎]  |
|                                                                              |
| 15:32  ⭐ NARRATIVE EVENT                                                   |
|        "Hunt Invitation" triggered                                          |
|        Sir Aldric invited the party on a boar hunt                         |
|        Outcome: Accepted - The Great Hunt enabled                          |
|        [View Event Details]                                      [Hide ◎]  |
|                                                                              |
| 15:28  🎯 CHALLENGE                                                         |
|        Elara: Perception Check (DC 15)                                      |
|        Roll: 18 (+3 WIS) = 21 ✓ Success                                    |
|        "Noticed Sir Aldric watching the group intently"                    |
|        [View Details]                                            [Hide ◎]  |
|                                                                              |
| 15:20  ⚔️ COMBAT ENDED                                                      |
|        Goblin Raid - Victory                                               |
|        Participants: Elara, Marcus, Throk vs 6 Goblins                    |
|        Duration: 4 rounds                                                  |
|        [View Combat Log]                                         [Hide ◎]  |
|                                                                              |
| 15:05  🚩 DM MARKER                                            [Critical]  |
|        "The Turning Point"                                                  |
|        Players chose to defend the townsfolk instead of pursuing the       |
|        goblin leader. This changes everything...                           |
|        [Edit] [Delete]                                           [Hide ◎]  |
|                                                                              |
| +-- SESSION 2: The Festival of Lights (Dec 12, 2025) ---------------------+ |
| ...                                                                          |
+------------------------------------------------------------------------------+
|                                          [+ Add DM Marker]  [Load More ↓]   |
+==============================================================================+
```

### Timeline Event Detail Modal

```
+=========================================================================+
|                         EVENT DETAILS                              [×]  |
+=========================================================================+
| Type: ⭐ NARRATIVE EVENT TRIGGERED                                      |
| Time: Dec 15, 2025 @ 15:32:18                                          |
| Game Time: Day 3, Late Afternoon                                       |
| Session: #3 - The Festival Aftermath                                   |
| Location: The Rusty Tankard                                            |
+-------------------------------------------------------------------------+
|                                                                         |
| EVENT: Hunt Invitation                                                  |
|                                                                         |
| ─── Trigger Conditions ─────────────────────────────────────────────── |
|                                                                         |
|   ✓ Flag "goblin_raid_complete" is set                                 |
|   ✓ Players are at "The Rusty Tankard"                                 |
|   ✓ Time context is "Afternoon" or later                               |
|                                                                         |
| ─── Scene Direction ────────────────────────────────────────────────── |
|                                                                         |
| The door of the tavern swings open dramatically. Sir Aldric            |
| Thornwood, renowned hunter of the Sandpoint region, strides in         |
| with mud-splattered boots and a gleam in his eye. He scans the         |
| room and his gaze settles on your party. A smile spreads across        |
| his weathered face.                                                    |
|                                                                         |
| "So YOU'RE the ones who drove off those goblins! I've been looking    |
| for capable sorts. I have a proposition that might interest you..."    |
|                                                                         |
| ─── Outcome Selected: "Accept Invitation" ──────────────────────────── |
|                                                                         |
| The party agreed to join Sir Aldric on his boar hunt.                  |
|                                                                         |
| Effects Applied:                                                        |
|   • Flag Set: hunt_accepted = true                                     |
|   • Event Enabled: "The Great Hunt"                                    |
|   • Item Given: Hunter's Map                                           |
|   • Relationship: Sir Aldric → Party +0.3                             |
|                                                                         |
| ─── Involved Characters ────────────────────────────────────────────── |
|                                                                         |
|   [👤] Elara (PC)      [👤] Marcus (PC)     [👤] Throk (PC)           |
|   [👤] Sir Aldric Thornwood (NPC)                                      |
|                                                                         |
| ─── Chained Events ─────────────────────────────────────────────────── |
|                                                                         |
|   → "The Great Hunt" (enabled, no delay)                               |
|                                                                         |
| Tags: #main-quest #social #aldric #hunt                                |
|                                                                         |
+-------------------------------------------------------------------------+
|                        [Edit Tags]  [Copy Summary]  [Hide Event]        |
+=========================================================================+
```

### Add DM Marker Modal

```
+=========================================================================+
|                         ADD DM MARKER                              [×]  |
+=========================================================================+
|                                                                         |
| Title *                                                                 |
| +-------------------------------------------------------------------+  |
| | The Turning Point                                                  |  |
| +-------------------------------------------------------------------+  |
|                                                                         |
| Note                                                                    |
| +-------------------------------------------------------------------+  |
| | Players chose to defend the townsfolk instead of pursuing the     |  |
| | goblin leader. This fundamentally changes their reputation in     |  |
| | Sandpoint and closes off the "Chase the Leader" story branch.     |  |
| |                                                                   |  |
| | This is a pivotal moment - they're now seen as heroes, not        |  |
| | just mercenaries.                                                 |  |
| +-------------------------------------------------------------------+  |
|                                                              500/2000   |
|                                                                         |
| Importance                             Marker Type                      |
| +------------------------+             +----------------------------+   |
| | ○ Minor               |             | ○ Note                     |   |
| | ○ Notable             |             | ○ Plot Point               |   |
| | ● Major               |             | ● Player Decision          |   |
| | ○ Critical            |             | ○ Character Moment         |   |
| +------------------------+             | ○ World Event              |   |
|                                        | ○ Foreshadowing            |   |
|                                        +----------------------------+   |
|                                                                         |
| Tags (comma-separated)                                                  |
| +-------------------------------------------------------------------+  |
| | main-quest, reputation, goblins                                    |  |
| +-------------------------------------------------------------------+  |
|                                                                         |
+-------------------------------------------------------------------------+
|                                              [Cancel]  [Add Marker]     |
+=========================================================================+
```

### Narrative Event Library

```
+==============================================================================+
|                        NARRATIVE EVENT LIBRARY                               |
+==============================================================================+
| [🔍 Search events...]                                                        |
| [Status: All ▼] [Chain: All ▼] [Act: All ▼] [Tags ▼] [Favorites Only ☆]    |
| [Sort: Priority ▼]                                     [Grid ⊞] [List ☰]   |
+------------------------------------------------------------------------------+
|                                                                              |
| QUICK ACCESS                                                                 |
| +------------------------------------------------------------------------+  |
| | ⭐ Favorites (3)    |    🔔 Pending (5)    |    ⛓️ In Chains (8)       |  |
| +------------------------------------------------------------------------+  |
|                                                                              |
| ALL EVENTS (24)                                                              |
| +------------------------------------------------------------------------+  |
| |                                                                        |  |
| | +------------------------------------------------------------------+  |  |
| | | ⭐ Goblin Raid                                    [TRIGGERED ✓]  |  |  |
| | |    Main Square attack after Mayor's speech                       |  |  |
| | |    Priority: 10  •  Chain: The Goblin Crisis                    |  |  |
| | |    Tags: combat, main-quest, goblins                            |  |  |
| | |    [Edit] [View] [Duplicate]                                     |  |  |
| | +------------------------------------------------------------------+  |  |
| |                                                                        |  |
| | +------------------------------------------------------------------+  |  |
| | | ☆ Hunt Invitation                                [TRIGGERED ✓]  |  |  |
| | |    Sir Aldric invites party on boar hunt                        |  |  |
| | |    Priority: 5   •  Chain: The Goblin Crisis                    |  |  |
| | |    Tags: social, main-quest, aldric                             |  |  |
| | |    [Edit] [View] [Duplicate]                                     |  |  |
| | +------------------------------------------------------------------+  |  |
| |                                                                        |  |
| | +------------------------------------------------------------------+  |  |
| | | ☆ The Great Hunt                                  [PENDING 1/2]  |  |  |
| | |    The boar hunt with Sir Aldric                                |  |  |
| | |    Priority: 5   •  Chain: The Goblin Crisis                    |  |  |
| | |    ⚡ Triggers: hunt_accepted ✓, at_tickwood_forest ✗           |  |  |
| | |    [Edit] [View] [Trigger Now] [Duplicate]                       |  |  |
| | +------------------------------------------------------------------+  |  |
| |                                                                        |  |
| | +------------------------------------------------------------------+  |  |
| | | ☆ Secret of the Forest                             [ACTIVE]      |  |  |
| | |    Discovery of ancient ruins in Tickwood                       |  |  |
| | |    Priority: 3   •  No Chain                                    |  |  |
| | |    Tags: side-quest, mystery, ruins                             |  |  |
| | |    [Edit] [View] [Trigger Now] [Disable] [Duplicate]             |  |  |
| | +------------------------------------------------------------------+  |  |
| |                                                                        |  |
| | +------------------------------------------------------------------+  |  |
| | | ☆ Aldric's Disappointment                         [INACTIVE]     |  |  |
| | |    (Only if players declined hunt)                              |  |  |
| | |    Priority: 3   •  Chain: The Goblin Crisis (branch)           |  |  |
| | |    [Edit] [View] [Enable] [Duplicate]                            |  |  |
| | +------------------------------------------------------------------+  |  |
| |                                                                        |  |
| +------------------------------------------------------------------------+  |
|                                                                              |
+------------------------------------------------------------------------------+
|                                    [+ Create New Event]  [+ Create Chain]   |
+==============================================================================+
```

### Narrative Event Designer

```
+==============================================================================+
|                      NARRATIVE EVENT DESIGNER                           [×]  |
+==============================================================================+
|                                                                              |
| ─── BASIC INFO ──────────────────────────────────────────────────────────── |
|                                                                              |
| Name *                                                                       |
| +------------------------------------------------------------------------+  |
| | Hunt Invitation                                                        |  |
| +------------------------------------------------------------------------+  |
|                                                                              |
| Description                                                                  |
| +------------------------------------------------------------------------+  |
| | After the party proves themselves by defending Sandpoint from the      |  |
| | goblin raid, the renowned hunter Sir Aldric Thornwood approaches       |  |
| | them with an invitation to join him on a dangerous boar hunt in        |  |
| | the Tickwood Forest.                                                   |  |
| +------------------------------------------------------------------------+  |
|                                                                              |
| Tags (comma-separated)                                                       |
| +------------------------------------------------------------------------+  |
| | main-quest, social, aldric, hunt                                       |  |
| +------------------------------------------------------------------------+  |
|                                                                              |
| ─── TRIGGER CONDITIONS ─────────────────────────────────── Logic: [All ▼] ── |
|                                                                              |
| +------------------------------------------------------------------------+  |
| | [+ Add Condition]                                                      |  |
| +------------------------------------------------------------------------+  |
| | ✓ Flag Set: "goblin_raid_complete"                            [🗑️] [≡] |  |
| |   When the goblin raid has been resolved                               |  |
| |   [ ] Required                                                         |  |
| +------------------------------------------------------------------------+  |
| | ✓ Player at Location: "The Rusty Tankard"                     [🗑️] [≡] |  |
| |   Players must be at the tavern                                        |  |
| |   [ ] Required                                                         |  |
| +------------------------------------------------------------------------+  |
| | ○ Time Context: "Afternoon" or "Evening"                      [🗑️] [≡] |  |
| |   Sir Aldric arrives after midday                                      |  |
| |   [ ] Required                                                         |  |
| +------------------------------------------------------------------------+  |
|                                                                              |
| Preview: "Triggers when: goblin_raid_complete is set AND player is at        |
|           The Rusty Tankard AND time is Afternoon or Evening"               |
|                                                                              |
| ─── SCENE DIRECTION ─────────────────────────────────────────────────────── |
|                                                                              |
| +------------------------------------------------------------------------+  |
| | The door of the tavern swings open dramatically. Sir Aldric            |  |
| | Thornwood, renowned hunter of the Sandpoint region, strides in         |  |
| | with mud-splattered boots and a gleam in his eye. He scans the         |  |
| | room and his gaze settles on your party. A smile spreads across        |  |
| | his weathered face.                                                    |  |
| |                                                                        |  |
| | "So YOU'RE the ones who drove off those goblins! I've been looking    |  |
| | for capable sorts. I have a proposition that might interest you..."    |  |
| +------------------------------------------------------------------------+  |
|                                                                              |
| Featured NPCs: [Sir Aldric Thornwood ×] [+ Add NPC]                         |
|                                                                              |
| ─── OUTCOMES ────────────────────────────────────────────────────────────── |
|                                                                              |
| [+ Add Outcome]                                   Default: [Accept ▼]       |
|                                                                              |
| +------------------------------------------------------------------------+  |
| | ▼ Outcome: "accept" - Accept Invitation                                |  |
| |   ─────────────────────────────────────────────────────────────────   |  |
| |   Description: The party agrees to join Sir Aldric on the hunt        |  |
| |                                                                        |  |
| |   Condition: [Dialogue Choice ▼] Keywords: agree, accept, yes, hunt   |  |
| |                                                                        |  |
| |   Effects:                                                             |  |
| |   +----------------------------------------------------------------+  |  |
| |   | • Set Flag: hunt_accepted = true                        [🗑️]  |  |  |
| |   | • Enable Event: "The Great Hunt"                        [🗑️]  |  |  |
| |   | • Give Item: "Hunter's Map"                             [🗑️]  |  |  |
| |   | • Modify Relationship: Sir Aldric → Party +0.3          [🗑️]  |  |  |
| |   | [+ Add Effect]                                                 |  |  |
| |   +----------------------------------------------------------------+  |  |
| |                                                                        |  |
| |   Chain To:                                                            |  |
| |   +----------------------------------------------------------------+  |  |
| |   | → "The Great Hunt" (delay: 0 turns)                     [🗑️]  |  |  |
| |   | [+ Chain Event]                                                |  |  |
| |   +----------------------------------------------------------------+  |  |
| |                                                                        |  |
| |   Timeline Summary: "Accepted Sir Aldric's invitation to hunt"        |  |
| +------------------------------------------------------------------------+  |
|                                                                              |
| +------------------------------------------------------------------------+  |
| | ▶ Outcome: "decline" - Decline Invitation                    [Expand]  |  |
| +------------------------------------------------------------------------+  |
|                                                                              |
| ─── OPTIONS ─────────────────────────────────────────────────────────────── |
|                                                                              |
| [✓] Active    [ ] Repeatable    Priority: [5    ]    [☆ Favorite]          |
|                                                                              |
| Location: [The Rusty Tankard ▼]    Scene: [None ▼]    Act: [None ▼]        |
|                                                                              |
| Chain: [The Goblin Crisis ▼]    Position: [2]                               |
|                                                                              |
+------------------------------------------------------------------------------+
|                              [Cancel]  [Save Draft]  [Save & Activate]      |
+==============================================================================+
```

### Add Trigger Condition Modal

```
+=========================================================================+
|                     ADD TRIGGER CONDITION                          [×]  |
+=========================================================================+
|                                                                         |
| Trigger Type                                                            |
| +-------------------------------------------------------------------+  |
| | ○ NPC Action           ○ Player Enters Location                   |  |
| | ○ Time at Location     ○ Dialogue Topic                           |  |
| | ○ Challenge Completed  ○ Relationship Threshold                   |  |
| | ● Flag Set             ○ Flag Not Set                             |  |
| | ○ Has Item             ○ Missing Item                             |  |
| | ○ Event Completed      ○ Turn Count                               |  |
| | ○ Stat Threshold       ○ Combat Result                            |  |
| | ○ Custom                                                          |  |
| +-------------------------------------------------------------------+  |
|                                                                         |
| ─── Flag Set Configuration ─────────────────────────────────────────── |
|                                                                         |
| Flag Name *                                                             |
| +-------------------------------------------------------------------+  |
| | goblin_raid_complete                                               |  |
| +-------------------------------------------------------------------+  |
| Suggestions: goblin_raid_complete, hunt_accepted, aldric_met          |
|                                                                         |
| Description (for DM reference)                                          |
| +-------------------------------------------------------------------+  |
| | When the goblin raid has been resolved                             |  |
| +-------------------------------------------------------------------+  |
|                                                                         |
| [ ] This condition is required (must match for AtLeast logic)          |
|                                                                         |
| Preview: "Triggers when flag 'goblin_raid_complete' is set to true"    |
|                                                                         |
+-------------------------------------------------------------------------+
|                                              [Cancel]  [Add Condition]  |
+=========================================================================+
```

### Add Effect Modal

```
+=========================================================================+
|                          ADD EFFECT                                [×]  |
+=========================================================================+
|                                                                         |
| Effect Type                                                             |
| +-------------------------------------------------------------------+  |
| | ○ Modify Relationship  ○ Give Item          ○ Take Item           |  |
| | ○ Reveal Information   ● Set Flag           ○ Enable Challenge    |  |
| | ○ Disable Challenge    ○ Enable Event       ○ Disable Event       |  |
| | ○ Trigger Scene        ○ Start Combat       ○ Modify Stat         |  |
| | ○ Add Reward           ○ Custom                                   |  |
| +-------------------------------------------------------------------+  |
|                                                                         |
| ─── Set Flag Configuration ─────────────────────────────────────────── |
|                                                                         |
| Flag Name *                                                             |
| +-------------------------------------------------------------------+  |
| | hunt_accepted                                                      |  |
| +-------------------------------------------------------------------+  |
|                                                                         |
| Value: (●) True   ( ) False                                            |
|                                                                         |
| Preview: "Sets flag 'hunt_accepted' to true"                           |
|                                                                         |
+-------------------------------------------------------------------------+
|                                                  [Cancel]  [Add Effect] |
+=========================================================================+
```

### Event Chain Visualizer

```
+==============================================================================+
|                    EVENT CHAIN: The Goblin Crisis                       [×]  |
+==============================================================================+
| [Edit Chain Info]  [Add Event]  [Rearrange]                [Export JSON]    |
+------------------------------------------------------------------------------+
|                                                                              |
|                    ┌─────────────────────┐                                   |
|                    │   🟢 Goblin Raid     │                                   |
|                    │   ══════════════    │                                   |
|                    │   @ Town Square     │                                   |
|                    │   TRIGGERED ✓       │                                   |
|                    │   Outcome: Victory  │                                   |
|                    └─────────┬───────────┘                                   |
|                              │                                               |
|                              ▼                                               |
|                    ┌─────────────────────┐                                   |
|                    │  🟢 Hunt Invitation  │                                   |
|                    │  ══════════════════ │                                   |
|                    │  @ Rusty Tankard    │                                   |
|                    │  TRIGGERED ✓        │                                   |
|                    │  Outcome: Accepted  │                                   |
|                    └─────────┬───────────┘                                   |
|                              │                                               |
|                    ┌─────────┴─────────┐                                     |
|                    │                   │                                     |
|                    ▼                   ▼                                     |
|         ┌─────────────────┐   ┌─────────────────────┐                       |
|         │ 🟡 The Great    │   │ ⚫ Aldric's         │                       |
|         │    Hunt         │   │    Disappointment   │                       |
|         │ ═══════════════ │   │ ═══════════════════ │                       |
|         │ @ Tickwood      │   │ (if declined)       │                       |
|         │ PENDING (1/2)   │   │ INACTIVE            │                       |
|         │ ⚡ hunt_accepted│   │                     │                       |
|         └────────┬────────┘   └─────────────────────┘                       |
|                  │                                                           |
|         ┌────────┴────────┐                                                  |
|         │                 │                                                  |
|         ▼                 ▼                                                  |
| ┌───────────────┐ ┌───────────────┐                                         |
| │ 🔵 Hunt       │ │ 🔵 Hunt       │                                         |
| │    Victory    │ │    Tragedy    │                                         |
| │ ═════════════ │ │ ═════════════ │                                         |
| │ (if success)  │ │ (if failure)  │                                         |
| │ INACTIVE      │ │ INACTIVE      │                                         |
| └───────────────┘ └───────────────┘                                         |
|                                                                              |
| ─── LEGEND ──────────────────────────────────────────────────────────────── |
| 🟢 Triggered    🟡 Pending (conditions partially met)    🔵 Active          |
| ⚫ Inactive     ═══ Double border = current chain position                  |
|                                                                              |
| Chain Progress: ████████░░░░░░░░ 2/6 events (33%)                           |
|                                                                              |
+------------------------------------------------------------------------------+
|                                          [Fit to Screen]  [Delete Chain]    |
+==============================================================================+
```

### Director View Pending Events Widget

```
+-------------------------------------------------------------------+
| PENDING EVENTS                                              [3] ▼ |
+-------------------------------------------------------------------+
|                                                                   |
| ┌───────────────────────────────────────────────────────────────┐ |
| │ ⚡ The Great Hunt                                [READY! 2/2] │ |
| │    LLM suggests triggering this event                        │ |
| │    [Trigger ▶] [Delay ⏸] [Dismiss ×]                        │ |
| └───────────────────────────────────────────────────────────────┘ |
|                                                                   |
| ┌───────────────────────────────────────────────────────────────┐ |
| │    Secret of the Forest                              [1/3]   │ |
| │    Waiting: player at ruins, perception check                │ |
| │    [View Details →]                                          │ |
| └───────────────────────────────────────────────────────────────┘ |
|                                                                   |
| ┌───────────────────────────────────────────────────────────────┐ |
| │    Midnight Visitor                                  [0/2]   │ |
| │    Waiting: midnight, at inn                                 │ |
| │    [View Details →]                                          │ |
| └───────────────────────────────────────────────────────────────┘ |
|                                                                   |
| [View All in Story Arc →]                                         |
+-------------------------------------------------------------------+
```

### Event Trigger Suggestion Popup

```
+==============================================================================+
|                      ⚡ EVENT TRIGGER SUGGESTION                              |
+==============================================================================+
|                                                                              |
| LLM detected that conditions are met for:                                    |
|                                                                              |
| ┌────────────────────────────────────────────────────────────────────────┐  |
| │  "The Great Hunt"                                              [★ Fav] │  |
| │                                                                        │  |
| │  The boar hunt with Sir Aldric begins. The party ventures into        │  |
| │  Tickwood Forest, unaware of the dangers that await them.             │  |
| └────────────────────────────────────────────────────────────────────────┘  |
|                                                                              |
| MATCHING TRIGGERS:                                                           |
| +------------------------------------------------------------------------+  |
| |   ✓ Flag "hunt_accepted" is set                                        |  |
| |   ✓ Players are at "Tickwood Forest"                                   |  |
| +------------------------------------------------------------------------+  |
| All conditions met (2/2) • Confidence: 95%                                  |
|                                                                              |
| SCENE DIRECTION:                                                             |
| +------------------------------------------------------------------------+  |
| │ The forest path narrows as ancient oaks tower overhead. Sir Aldric    │  |
| │ raises a hand, signaling silence. "We're entering boar territory      │  |
| │ now," he whispers, nocking an arrow. "Stay alert—these beasts are     │  |
| │ fiercer than any goblin."                                             │  |
| │                                                                        │  |
| │ A distant snort echoes through the trees...                           │  |
| +------------------------------------------------------------------------+  |
|                                                                              |
| FEATURED NPCs: Sir Aldric Thornwood                                         |
|                                                                              |
| OUTCOMES:                                                                    |
|   → "hunt_success" - Successfully hunt the boar                            |
|   → "hunt_failure" - The hunt goes wrong                                   |
|   → "hunt_discovery" - Discover something unexpected                       |
|                                                                              |
+------------------------------------------------------------------------------+
|                                                                              |
| [Approve - Trigger Now]  [Delay: 1 turn ▼]  [Delay Until...]  [Reject]      |
|                                                                              |
+==============================================================================+
```

---

## API Endpoints

### Story Events

```
# List story events for a session (with pagination and filters)
GET /api/sessions/{session_id}/story-events
Query params:
  - type: comma-separated StoryEventType names
  - character_id: filter by involved character
  - location_id: filter by location
  - from_date: ISO datetime
  - to_date: ISO datetime
  - include_hidden: boolean (default false)
  - tags: comma-separated
  - limit: number (default 50, max 200)
  - offset: number
Response: 200 OK
{
  "events": [StoryEventResponse],
  "total": number,
  "has_more": boolean
}

# Get single story event
GET /api/story-events/{event_id}
Response: 200 OK - StoryEventResponse

# Create story event (DM markers and custom events)
POST /api/sessions/{session_id}/story-events
Body: CreateStoryEventRequest
{
  "event_type": StoryEventType,
  "summary": string,
  "game_time": string | null,
  "tags": string[]
}
Response: 201 Created - StoryEventResponse

# Update story event (summary, hidden, tags)
PUT /api/story-events/{event_id}
Body: UpdateStoryEventRequest
{
  "summary": string | null,
  "is_hidden": boolean | null,
  "tags": string[] | null
}
Response: 200 OK - StoryEventResponse

# Delete story event (DM markers only)
DELETE /api/story-events/{event_id}
Response: 204 No Content
Error: 400 if not a DM marker

# Search story events across sessions
GET /api/worlds/{world_id}/story-events/search
Query params:
  - q: search query
  - from_date: ISO datetime
  - to_date: ISO datetime
  - session_id: filter by session
  - limit: number
  - offset: number
Response: 200 OK - { events: StoryEventResponse[], total: number }

# Get story event statistics
GET /api/worlds/{world_id}/story-events/stats
Response: 200 OK
{
  "total_events": number,
  "by_type": { [type: string]: number },
  "by_session": { [session_id: string]: number }
}
```

### Narrative Events

```
# List narrative events for a world
GET /api/worlds/{world_id}/narrative-events
Query params:
  - active: boolean
  - triggered: boolean
  - favorite: boolean
  - chain_id: filter by chain
  - act_id: filter by act
  - tags: comma-separated
  - search: search query
  - sort: priority | name | created | updated | trigger_progress
  - limit: number
  - offset: number
Response: 200 OK
{
  "events": [NarrativeEventResponse],
  "total": number
}

# Get single narrative event with full details
GET /api/narrative-events/{event_id}
Response: 200 OK - NarrativeEventResponse (includes all triggers, outcomes, chains)

# Create narrative event
POST /api/worlds/{world_id}/narrative-events
Body: CreateNarrativeEventRequest
{
  "name": string,
  "description": string | null,
  "tags": string[],
  "trigger_conditions": NarrativeTrigger[],
  "trigger_logic": TriggerLogic,
  "scene_direction": string,
  "suggested_opening": string | null,
  "featured_npcs": CharacterId[],
  "outcomes": EventOutcome[],
  "default_outcome": string | null,
  "is_active": boolean,
  "is_repeatable": boolean,
  "delay_turns": number,
  "expires_after_turns": number | null,
  "scene_id": SceneId | null,
  "location_id": LocationId | null,
  "act_id": ActId | null,
  "priority": number,
  "chain_id": EventChainId | null,
  "chain_position": number | null
}
Response: 201 Created - NarrativeEventResponse

# Update narrative event
PUT /api/narrative-events/{event_id}
Body: UpdateNarrativeEventRequest (same fields as create, all optional)
Response: 200 OK - NarrativeEventResponse

# Delete narrative event
DELETE /api/narrative-events/{event_id}
Response: 204 No Content

# Toggle active status
PUT /api/narrative-events/{event_id}/active
Body: { "active": boolean }
Response: 200 OK - { "is_active": boolean }

# Toggle favorite
PUT /api/narrative-events/{event_id}/favorite
Response: 200 OK - { "is_favorite": boolean }

# Check trigger conditions against current context
POST /api/narrative-events/{event_id}/check-triggers
Body: { "session_id": string }
Response: 200 OK
{
  "is_triggered": boolean,
  "matched_triggers": string[],
  "unmatched_triggers": string[],
  "confidence": number
}

# Manually trigger event
POST /api/narrative-events/{event_id}/trigger
Body: {
  "session_id": string,
  "outcome_name": string | null,  // If specified, skip to this outcome
  "bypass_conditions": boolean    // If true, ignore trigger conditions
}
Response: 200 OK
{
  "story_event_id": StoryEventId,
  "effects_applied": string[],
  "chained_events_enabled": NarrativeEventId[]
}

# Duplicate event
POST /api/narrative-events/{event_id}/duplicate
Body: { "new_name": string | null }
Response: 201 Created - NarrativeEventResponse
```

### Event Chains

```
# List event chains for a world
GET /api/worlds/{world_id}/event-chains
Query params:
  - active: boolean
  - act_id: filter by act
  - tags: comma-separated
Response: 200 OK - EventChainResponse[]

# Get single chain with all event details
GET /api/event-chains/{chain_id}
Query params:
  - include_events: boolean (default true)
Response: 200 OK - EventChainResponse (with events array if include_events)

# Create event chain
POST /api/worlds/{world_id}/event-chains
Body: {
  "name": string,
  "description": string | null,
  "events": NarrativeEventId[],
  "act_id": ActId | null,
  "tags": string[],
  "color": string | null
}
Response: 201 Created - EventChainResponse

# Update event chain
PUT /api/event-chains/{chain_id}
Body: UpdateEventChainRequest (fields optional)
Response: 200 OK - EventChainResponse

# Delete event chain
DELETE /api/event-chains/{chain_id}
Query params:
  - delete_events: boolean (default false, if true also deletes member events)
Response: 204 No Content

# Add event to chain
POST /api/event-chains/{chain_id}/events
Body: {
  "event_id": NarrativeEventId,
  "position": number | null  // If null, adds at end
}
Response: 200 OK - EventChainResponse

# Remove event from chain
DELETE /api/event-chains/{chain_id}/events/{event_id}
Response: 200 OK - EventChainResponse

# Reorder events in chain
PUT /api/event-chains/{chain_id}/events/reorder
Body: {
  "event_ids": NarrativeEventId[]  // New order
}
Response: 200 OK - EventChainResponse
```

### WebSocket Messages

```typescript
// Server → Client: LLM suggests triggering an event
interface EventTriggerSuggestion {
  type: "EventTriggerSuggestion";
  event_id: string;
  event_name: string;
  event_description: string;
  scene_direction: string;
  featured_npcs: { id: string; name: string }[];
  matching_triggers: {
    trigger_id: string;
    description: string;
    matched: boolean;
  }[];
  outcomes: {
    name: string;
    label: string;
    description: string;
  }[];
  confidence: number;  // 0.0-1.0
  suggested_by: "llm" | "system";  // LLM detected vs condition-based
}

// Client → Server: DM decision on event trigger
interface EventTriggerDecision {
  type: "EventTriggerDecision";
  event_id: string;
  decision: "approve" | "delay" | "reject";
  delay_turns?: number;           // Required if decision is "delay"
  selected_outcome?: string;      // Pre-select outcome
  reason?: string;                // Optional, for logging
}

// Server → All: Event was triggered
interface NarrativeEventTriggered {
  type: "NarrativeEventTriggered";
  event_id: string;
  event_name: string;
  scene_direction: string;
  triggered_by: "dm" | "llm" | "system" | "manual";
  story_event_id: string;
}

// Server → All: New story event created
interface StoryEventCreated {
  type: "StoryEventCreated";
  story_event: StoryEventResponse;
}

// Server → DM: Event status changed
interface NarrativeEventStatusChanged {
  type: "NarrativeEventStatusChanged";
  event_id: string;
  event_name: string;
  new_status: "active" | "inactive" | "triggered";
  trigger_progress?: {
    matched: number;
    total: number;
  };
}
```

---

## LLM Integration

### Adding Narrative Events to LLM Context

**File**: `Engine/src/application/services/llm_service.rs`

In the `build_system_prompt_with_notes()` function, add a new section after directorial notes:

```rust
fn build_narrative_event_context(
    active_events: &[NarrativeEvent],
    current_context: &TriggerContext,
) -> String {
    let mut context = String::new();

    // Only include events that are:
    // 1. Active and not yet triggered
    // 2. Relevant to current scene/location (or have no scene/location restriction)
    // 3. Have at least some matching triggers
    let relevant_events: Vec<_> = active_events
        .iter()
        .filter(|e| e.is_active && !e.is_triggered)
        .filter(|e| e.is_relevant_to_context(current_context))
        .filter(|e| e.evaluate_triggers(current_context).matched_triggers.len() > 0
                    || e.trigger_conditions.is_empty())
        .collect();

    if relevant_events.is_empty() {
        return String::new();
    }

    context.push_str("\n\n=== NARRATIVE EVENTS (DM-PREPARED STORY HOOKS) ===\n");
    context.push_str("The following events may trigger based on current circumstances.\n");
    context.push_str("If you detect trigger conditions are FULLY met, flag the event.\n\n");

    for event in relevant_events.iter().take(5) {  // Limit to top 5 most relevant
        let eval = event.evaluate_triggers(current_context);

        context.push_str(&format!("EVENT: {}\n", event.name));
        context.push_str(&format!("Description: {}\n", event.description));
        context.push_str("Triggers when:\n");

        for trigger in &event.trigger_conditions {
            let status = if eval.matched_triggers.contains(&trigger.trigger_id) {
                "✓ MET"
            } else {
                "✗ NOT MET"
            };
            context.push_str(&format!("  - {} [{}]\n", trigger.description, status));
        }

        context.push_str(&format!("Current progress: {}/{} conditions\n",
            eval.matched_triggers.len(),
            event.trigger_conditions.len()));

        if eval.is_triggered {
            context.push_str(">>> ALL CONDITIONS MET - Consider triggering this event <<<\n");
        }

        context.push_str("\n");
    }

    context.push_str("─────────────────────────────────────────────────────────\n");
    context.push_str("INSTRUCTIONS FOR NARRATIVE EVENTS:\n");
    context.push_str("- If ALL trigger conditions for an event are met, include:\n");
    context.push_str("  NARRATIVE_EVENT_TRIGGER: <event_name>\n");
    context.push_str("  TRIGGER_CONFIDENCE: <0.0-1.0>\n");
    context.push_str("  TRIGGER_REASONING: <brief explanation>\n");
    context.push_str("- Do NOT trigger events yourself - just flag them for DM approval\n");
    context.push_str("- Continue normal NPC dialogue even when flagging events\n");
    context.push_str("─────────────────────────────────────────────────────────\n");

    context
}
```

### LLM Response Parsing

```rust
/// Parsed event trigger suggestion from LLM response
#[derive(Debug, Clone)]
pub struct LlmEventTriggerSuggestion {
    pub event_name: String,
    pub confidence: f32,
    pub reasoning: String,
}

/// Parse NARRATIVE_EVENT_TRIGGER from LLM response
fn parse_event_trigger_suggestions(response_text: &str) -> Vec<LlmEventTriggerSuggestion> {
    let mut suggestions = Vec::new();
    let mut current_event: Option<String> = None;
    let mut current_confidence: f32 = 0.8;
    let mut current_reasoning = String::new();

    for line in response_text.lines() {
        let line = line.trim();

        if line.starts_with("NARRATIVE_EVENT_TRIGGER:") {
            // Save previous if exists
            if let Some(event_name) = current_event.take() {
                suggestions.push(LlmEventTriggerSuggestion {
                    event_name,
                    confidence: current_confidence,
                    reasoning: std::mem::take(&mut current_reasoning),
                });
            }

            current_event = Some(
                line.trim_start_matches("NARRATIVE_EVENT_TRIGGER:")
                    .trim()
                    .to_string()
            );
            current_confidence = 0.8;  // Default
        } else if line.starts_with("TRIGGER_CONFIDENCE:") {
            if let Ok(conf) = line
                .trim_start_matches("TRIGGER_CONFIDENCE:")
                .trim()
                .parse::<f32>()
            {
                current_confidence = conf.clamp(0.0, 1.0);
            }
        } else if line.starts_with("TRIGGER_REASONING:") {
            current_reasoning = line
                .trim_start_matches("TRIGGER_REASONING:")
                .trim()
                .to_string();
        }
    }

    // Don't forget last one
    if let Some(event_name) = current_event {
        suggestions.push(LlmEventTriggerSuggestion {
            event_name,
            confidence: current_confidence,
            reasoning: current_reasoning,
        });
    }

    suggestions
}
```

### Custom Trigger LLM Evaluation

For triggers with `llm_evaluation: true`:

```rust
async fn evaluate_custom_trigger_with_llm(
    llm_service: &LlmService,
    trigger: &NarrativeTrigger,
    context: &TriggerContext,
) -> Result<bool, anyhow::Error> {
    let NarrativeTriggerType::Custom { description, llm_evaluation: true } = &trigger.trigger_type
    else {
        return Ok(false);
    };

    let prompt = format!(
        r#"You are evaluating whether a narrative trigger condition is met.

CURRENT GAME STATE:
- Location: {}
- Scene: {}
- Time: {}
- Recent player action: {}
- Recent dialogue topics: {}

TRIGGER CONDITION TO EVALUATE:
"{}"

Based on the current game state, is this condition met?
Answer with only "YES" or "NO" followed by a brief reason.
"#,
        context.current_location.map(|l| l.to_string()).unwrap_or("Unknown".to_string()),
        context.current_scene.map(|s| s.to_string()).unwrap_or("None".to_string()),
        context.time_context.as_deref().unwrap_or("Unknown"),
        context.recent_player_action.as_deref().unwrap_or("None"),
        context.recent_dialogue_topics.join(", "),
        description
    );

    let response = llm_service.simple_completion(&prompt).await?;
    let response_upper = response.trim().to_uppercase();

    Ok(response_upper.starts_with("YES"))
}
```

---

## Implementation Phases

### Phase 17A: Domain Foundation (Engine)
**Estimated: 1 day**

**Tasks:**
1. Add ID types to `Engine/src/domain/value_objects/ids.rs`:
   - `StoryEventId`
   - `NarrativeEventId`
   - `EventChainId`

2. Create `Engine/src/domain/entities/story_event.rs`:
   - Full `StoryEvent` struct
   - All `StoryEventType` variants
   - Supporting enums (CombatOutcome, ItemSource, etc.)

3. Create `Engine/src/domain/entities/narrative_event.rs`:
   - Full `NarrativeEvent` struct
   - `NarrativeTrigger` and `NarrativeTriggerType`
   - `EventOutcome` and `OutcomeCondition`
   - `EventEffect` and `ChainedEvent`
   - `TriggerContext` and `TriggerEvaluation`

4. Create `Engine/src/domain/entities/event_chain.rs`:
   - `EventChain` struct with methods

5. Update `Engine/src/domain/entities/mod.rs`:
   - Export new modules

**Files to create:**
- `Engine/src/domain/entities/story_event.rs`
- `Engine/src/domain/entities/narrative_event.rs`
- `Engine/src/domain/entities/event_chain.rs`

**Files to modify:**
- `Engine/src/domain/value_objects/ids.rs`
- `Engine/src/domain/entities/mod.rs`

---

### Phase 17B: Story Events Backend (Engine)
**Estimated: 2 days**

**Tasks:**
1. Create `Neo4jStoryEventRepository`:
   - CRUD operations
   - Search with filters
   - Pagination support
   - Cypher queries for timeline

2. Create `story_event_routes.rs`:
   - All REST endpoints
   - Request/response DTOs
   - Validation

3. Wire StoryEvent creation into existing flows:
   - `websocket.rs`: Create StoryEvent for DialogueResponse
   - `websocket.rs`: Create StoryEvent for tool execution results
   - Challenge completion → StoryEvent
   - Scene transition → StoryEvent

4. Add StoryEvent to session state:
   - Recent events for timeline updates
   - WebSocket broadcast on new events

**Files to create:**
- `Engine/src/infrastructure/persistence/story_event_repository.rs`
- `Engine/src/infrastructure/http/story_event_routes.rs`

**Files to modify:**
- `Engine/src/infrastructure/persistence/mod.rs`
- `Engine/src/infrastructure/http/mod.rs`
- `Engine/src/infrastructure/websocket.rs`
- `Engine/src/infrastructure/session.rs`
- `Engine/src/main.rs` (add routes)

---

### Phase 17C: Narrative Events Backend (Engine)
**Estimated: 2 days**

**Tasks:**
1. Create `Neo4jNarrativeEventRepository`:
   - CRUD operations
   - Trigger evaluation helpers
   - Chain membership management

2. Create `narrative_event_routes.rs`:
   - All REST endpoints
   - Trigger checking endpoint
   - Manual trigger endpoint

3. Create `event_chain_routes.rs`:
   - All chain CRUD endpoints
   - Event membership management

4. Create `NarrativeEventService`:
   - Trigger evaluation logic
   - Effect execution
   - Chain progression

5. Add WebSocket messages:
   - `EventTriggerSuggestion`
   - `EventTriggerDecision`
   - `NarrativeEventTriggered`
   - `StoryEventCreated`

**Files to create:**
- `Engine/src/infrastructure/persistence/narrative_event_repository.rs`
- `Engine/src/infrastructure/persistence/event_chain_repository.rs`
- `Engine/src/infrastructure/http/narrative_event_routes.rs`
- `Engine/src/infrastructure/http/event_chain_routes.rs`
- `Engine/src/application/services/narrative_event_service.rs`

**Files to modify:**
- `Engine/src/infrastructure/websocket.rs`
- `Engine/src/infrastructure/websocket/messages.rs`
- `Engine/src/main.rs`

---

### Phase 17D: LLM Integration (Engine)
**Estimated: 2 days**

**Tasks:**
1. Update LlmService prompt building:
   - Add `build_narrative_event_context()` function
   - Include in system prompt

2. Parse event triggers from LLM response:
   - `parse_event_trigger_suggestions()`
   - Validate against actual events

3. Implement custom trigger LLM evaluation:
   - `evaluate_custom_trigger_with_llm()`

4. Wire event suggestions to approval flow:
   - Create `EventTriggerSuggestion` messages
   - Handle `EventTriggerDecision` responses
   - Execute effects on approval

**Files to modify:**
- `Engine/src/application/services/llm_service.rs`
- `Engine/src/infrastructure/websocket.rs`

---

### Phase 17E: Story Arc Tab UI (Player)
**Estimated: 2-3 days**

**Tasks:**
1. Add `StoryArc` to `DMMode` enum in `main.rs`

2. Create `StoryArcView` component:
   - Sub-navigation tabs
   - Route to Timeline/Events/Chains views

3. Create `TimelineView`:
   - Event list with infinite scroll
   - Filter bar component
   - Event card component

4. Create `TimelineEventCard`:
   - Type-specific styling
   - Summary display
   - Actions (hide, view)

5. Create `TimelineEventDetail` modal:
   - Full event information
   - Edit capabilities for editable fields

6. Create `AddDmMarker` form:
   - Modal or inline form
   - Validation

7. Wire to story-events API:
   - Fetch events
   - Create markers
   - Update events

**Files to create:**
- `Player/src/presentation/views/story_arc_view.rs`
- `Player/src/presentation/components/story_arc/mod.rs`
- `Player/src/presentation/components/story_arc/timeline_view.rs`
- `Player/src/presentation/components/story_arc/timeline_event_card.rs`
- `Player/src/presentation/components/story_arc/timeline_event_detail.rs`
- `Player/src/presentation/components/story_arc/timeline_filters.rs`
- `Player/src/presentation/components/story_arc/add_dm_marker.rs`

**Files to modify:**
- `Player/src/main.rs`
- `Player/src/presentation/views/mod.rs`
- `Player/src/presentation/components/mod.rs`

---

### Phase 17F: Narrative Event UI (Player)
**Estimated: 3-4 days**

**Tasks:**
1. Create `NarrativeEventLibrary`:
   - List/grid view
   - Search and filters
   - Event cards with status

2. Create `NarrativeEventDesigner` modal:
   - Basic info section
   - Trigger conditions builder
   - Outcomes builder
   - Options section

3. Create `TriggerConditionBuilder`:
   - Trigger type selector
   - Type-specific configuration forms
   - Condition preview

4. Create `OutcomeBuilder`:
   - Outcome card component
   - Effects list
   - Chain events list

5. Create `AddTriggerModal`:
   - All trigger types
   - Configuration forms

6. Create `AddEffectModal`:
   - All effect types
   - Configuration forms

7. Create `EventTriggerApproval` popup:
   - Show suggestion details
   - Approve/delay/reject actions

**Files to create:**
- `Player/src/presentation/components/story_arc/narrative_event_library.rs`
- `Player/src/presentation/components/story_arc/narrative_event_designer.rs`
- `Player/src/presentation/components/story_arc/trigger_condition_builder.rs`
- `Player/src/presentation/components/story_arc/outcome_builder.rs`
- `Player/src/presentation/components/story_arc/add_trigger_modal.rs`
- `Player/src/presentation/components/story_arc/add_effect_modal.rs`
- `Player/src/presentation/components/story_arc/event_trigger_approval.rs`

---

### Phase 17G: Event Chains & Polish (Player)
**Estimated: 2-3 days**

**Tasks:**
1. Create `EventChainList`:
   - List of chains
   - Chain cards with progress

2. Create `EventChainVisualizer`:
   - Flowchart rendering (CSS-based or canvas)
   - Node components
   - Connection lines
   - Zoom/pan controls

3. Create `EventChainEditor`:
   - Edit chain info
   - Add/remove/reorder events

4. Create `PendingEventsWidget`:
   - Small panel for Director view
   - Relevant events list
   - Quick actions

5. Add Story Arc tab to DM View:
   - Update header tabs
   - Route handling

6. Export/Import functionality:
   - JSON export
   - JSON import with validation

7. Testing and polish:
   - Error handling
   - Loading states
   - Empty states

**Files to create:**
- `Player/src/presentation/components/story_arc/event_chain_list.rs`
- `Player/src/presentation/components/story_arc/event_chain_visualizer.rs`
- `Player/src/presentation/components/story_arc/event_chain_editor.rs`
- `Player/src/presentation/components/dm_panel/pending_events_widget.rs`

**Files to modify:**
- `Player/src/presentation/views/dm_view.rs`

---

## Dependencies

### Internal Dependencies
- **Phase 14D (Challenge System)**: Reuse `TriggerCondition` and `TriggerType` patterns
- **Existing Challenge Library modal**: Pattern for library UI
- **Existing LLM approval flow**: Pattern for event trigger approval
- **Existing session management**: For session_id tracking

### External Dependencies
- None beyond existing dependencies

---

## Testing Checklist

### Backend (Engine)
- [ ] StoryEvent CRUD operations
- [ ] StoryEvent filtering and pagination
- [ ] NarrativeEvent CRUD operations
- [ ] Trigger evaluation logic (all trigger types)
- [ ] Effect execution (all effect types)
- [ ] EventChain management
- [ ] WebSocket message handling
- [ ] LLM context building
- [ ] LLM response parsing

### Frontend (Player)
- [ ] Timeline displays events correctly
- [ ] Timeline filtering works
- [ ] DM markers can be added/edited/deleted
- [ ] Event detail modal shows all information
- [ ] Narrative event designer saves correctly
- [ ] All trigger types can be configured
- [ ] All effect types can be configured
- [ ] Event chains display correctly
- [ ] Pending events widget updates in real-time
- [ ] Export/import works correctly

### Integration
- [ ] Story events created automatically during gameplay
- [ ] LLM suggests event triggers appropriately
- [ ] DM approval flow works end-to-end
- [ ] Effects apply correctly when events trigger
- [ ] Event chains progress correctly

---

## Estimated Total Effort

| Phase | Duration |
|-------|----------|
| 17A: Domain Foundation | 1 day |
| 17B: Story Events Backend | 2 days |
| 17C: Narrative Events Backend | 2 days |
| 17D: LLM Integration | 2 days |
| 17E: Story Arc Tab UI | 2-3 days |
| 17F: Narrative Event UI | 3-4 days |
| 17G: Event Chains & Polish | 2-3 days |
| **Total** | **14-17 days** |
