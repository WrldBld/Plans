# Narrative Event Creation Form: Plans vs Implementation Analysis

**Date**: 2025-12-14  
**Status**: Significant gaps identified

## Executive Summary

The current implementation of the narrative event creation form (`NarrativeEventFormModal`) is **minimal** compared to the comprehensive design specified in the plans. Only **3 basic fields** are implemented out of a full-featured form that should include triggers, outcomes, effects, and extensive configuration options.

**Implementation Coverage**: ~15% of planned functionality

---

## Detailed Comparison

### ✅ Implemented Fields

| Field | Status | Notes |
|-------|--------|-------|
| Name | ✅ Implemented | Required field, validated |
| Description | ✅ Implemented | Optional textarea |
| Scene Direction | ✅ Implemented | Optional textarea |

### ❌ Missing from Basic Info Section

| Field | Required in Plans | Status |
|-------|------------------|--------|
| Tags | Optional | ❌ **Missing** - Should support comma-separated tags for organization |

### ❌ Missing Entire Sections

#### 1. Trigger Conditions Section (COMPLETELY MISSING)

**Planned Features** (from US-17.7):
- "Add Condition" button with trigger type dropdown
- Support for **15+ trigger types**:
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

**Current State**: 
- ❌ No trigger conditions UI
- ❌ No trigger logic selector
- ❌ Backend supports triggers (domain model exists), but no way to create them via UI

**Impact**: **CRITICAL** - Events cannot be configured to trigger automatically. This is a core feature of narrative events.

#### 2. Outcomes Section (COMPLETELY MISSING)

**Planned Features** (from US-17.8):
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
  - Effects list (see below)
  - Chain Events list
  - Timeline Summary (what shows in timeline)
- At least one outcome required
- Default outcome selector (if no explicit choice made)
- Reorder outcomes via drag-and-drop
- Delete outcome button

**Current State**:
- ❌ No outcomes UI
- ❌ Backend supports outcomes (domain model exists), but no way to create them via UI

**Impact**: **CRITICAL** - Events cannot have branching outcomes. This defeats the purpose of narrative events.

#### 3. Outcome Effects Section (COMPLETELY MISSING)

**Planned Features** (from US-17.8a):
- "Add Effect" button within each outcome
- Support for **14+ effect types**:
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

**Current State**:
- ❌ No effects UI
- ❌ Backend supports effects (domain model exists), but no way to create them via UI

**Impact**: **HIGH** - Events cannot modify game state when triggered.

#### 4. Options/Configuration Section (COMPLETELY MISSING)

**Planned Features**:
- Active toggle (whether event can be triggered)
- Repeatable toggle (can trigger multiple times)
- Priority (numeric, for ordering multiple triggered events)
- Favorite toggle (for quick access)
- Location selector (optional association)
- Scene selector (optional association)
- Act selector (optional Monomyth integration)
- Chain selector (link to event chain)
- Position in chain (if part of chain)
- Delay turns (optional delay before event fires)
- Expires after turns (optional expiration)

**Current State**:
- ❌ No options UI
- ✅ Backend supports these fields (they're in the DTO and domain model)
- ✅ Some fields are set via defaults (e.g., `is_active: true`)

**Impact**: **MEDIUM** - Events can be created but lack fine-grained control.

#### 5. Scene Direction Enhancements (PARTIALLY MISSING)

**Planned Features**:
- Scene Direction textarea ✅ (implemented)
- Suggested Opening (optional dialogue/action) ❌ **Missing**
- Featured NPCs selector ❌ **Missing** (multi-select NPCs that should be featured)

**Current State**:
- ✅ Scene Direction field exists
- ❌ Suggested Opening field missing
- ❌ Featured NPCs selector missing

**Impact**: **LOW-MEDIUM** - Basic scene direction works, but advanced features missing.

---

## Validation Gaps

**Planned Validation** (from US-17.6):
- ✅ Name is required (implemented)
- ❌ **Must have at least one trigger condition** (not validated)
- ❌ **Must have at least one outcome** (not validated)
- ❌ Event saved as draft until explicitly activated (not implemented - events are created as active by default)

**Current State**:
- Only name validation exists
- No validation for triggers/outcomes
- Events are created as active immediately (no draft state)

**Impact**: **HIGH** - Events can be created in invalid states that won't function properly.

---

## API Contract Comparison

### Planned API (from plans/17-story-arc.md)

```json
POST /api/worlds/{world_id}/narrative-events
{
  "name": string,
  "description": string | null,
  "tags": string[],
  "trigger_conditions": NarrativeTrigger[],  // ❌ Not sent
  "trigger_logic": TriggerLogic,              // ❌ Not sent
  "scene_direction": string,
  "suggested_opening": string | null,        // ❌ Not sent
  "featured_npcs": CharacterId[],             // ❌ Not sent
  "outcomes": EventOutcome[],                 // ❌ Not sent
  "default_outcome": string | null,           // ❌ Not sent
  "is_active": boolean,                       // ✅ Sent (default true)
  "is_repeatable": boolean,                   // ✅ Sent (default false)
  "delay_turns": number,                      // ✅ Sent (default 0)
  "expires_after_turns": number | null,       // ✅ Sent (default null)
  "scene_id": SceneId | null,                // ❌ Not sent
  "location_id": LocationId | null,          // ❌ Not sent
  "act_id": ActId | null,                    // ❌ Not sent
  "priority": number,                         // ✅ Sent (default 0)
  "chain_id": EventChainId | null,           // ❌ Not sent
  "chain_position": number | null            // ❌ Not sent
}
```

### Current Implementation

**Frontend Request** (`CreateNarrativeEventRequest`):
```rust
pub struct CreateNarrativeEventRequest {
    pub name: String,                    // ✅
    pub description: String,            // ✅
    pub scene_direction: String,         // ✅
    pub suggested_opening: Option<String>, // ✅ (in DTO, but not in form)
    pub is_repeatable: bool,             // ✅ (default false)
    pub delay_turns: u32,                // ✅ (default 0)
    pub expires_after_turns: Option<u32>, // ✅ (default None)
    pub priority: i32,                   // ✅ (default 0)
    pub is_active: bool,                 // ✅ (default true)
    pub tags: Vec<String>,               // ✅ (in DTO, but not in form)
}
```

**Backend DTO** (`CreateNarrativeEventRequestDto`):
- Matches frontend DTO (same fields)
- Backend accepts these fields but **does not accept**:
  - `trigger_conditions`
  - `trigger_logic`
  - `featured_npcs`
  - `outcomes`
  - `default_outcome`
  - `scene_id`
  - `location_id`
  - `act_id`
  - `chain_id`
  - `chain_position`

**Backend Processing** (`narrative_event_routes.rs`):
- Creates event with `NarrativeEvent::new()`
- Sets basic fields from request
- **Does NOT set**:
  - Trigger conditions (stays empty)
  - Outcomes (stays empty)
  - Featured NPCs (stays empty)
  - Associations (scene, location, act, chain)

---

## UI Mockup Comparison

### Planned UI (from plans/17-story-arc.md, lines 1679-1786)

The planned "Narrative Event Designer" is a comprehensive modal with:
- **Basic Info section**: Name, Description, Tags ✅ (partially - missing tags)
- **Trigger Conditions section**: Full trigger builder with 15+ types ❌ **MISSING**
- **Scene Direction section**: Scene direction, suggested opening, featured NPCs ✅ (partially - only scene direction)
- **Outcomes section**: Full outcome builder with effects and chains ❌ **MISSING**
- **Options section**: All configuration toggles and selectors ❌ **MISSING**

### Current UI (`NarrativeEventFormModal`)

A minimal modal with:
- Header: "New Narrative Event" ✅
- Name field (required) ✅
- Description field (optional) ✅
- Scene Direction field (optional) ✅
- Cancel and Create buttons ✅
- **That's it** - only 3 fields total

---

## Backend Support Status

### Domain Model Support

✅ **Full Support**: The domain model (`Engine/src/domain/entities/narrative_event.rs`) has complete support for:
- All trigger types (`NarrativeTriggerType` enum with 15 variants)
- All outcome types (`EventOutcome` struct)
- All effect types (`EventEffect` enum with 14 variants)
- All configuration options (priority, repeatable, delays, associations, etc.)

### API Support

⚠️ **Partial Support**: 
- Backend DTOs exist but don't accept trigger/outcome data
- Backend routes accept minimal data
- Backend service creates events but with empty triggers/outcomes

### Repository Support

✅ **Full Support**: Repository can persist all fields (domain model is complete)

---

## Impact Assessment

### Critical Gaps (Blocking Core Functionality)

1. **No Trigger Conditions UI** 
   - **Impact**: Events cannot be configured to trigger automatically
   - **Severity**: CRITICAL
   - **User Story**: US-17.7 completely unimplemented

2. **No Outcomes UI**
   - **Impact**: Events cannot have branching outcomes
   - **Severity**: CRITICAL
   - **User Story**: US-17.8 completely unimplemented

3. **No Effects UI**
   - **Impact**: Events cannot modify game state
   - **Severity**: HIGH
   - **User Story**: US-17.8a completely unimplemented

### High Priority Gaps

4. **No Validation for Triggers/Outcomes**
   - **Impact**: Events can be created in invalid states
   - **Severity**: HIGH

5. **No Draft State**
   - **Impact**: Events are immediately active even if incomplete
   - **Severity**: MEDIUM

### Medium Priority Gaps

6. **Missing Configuration Options**
   - **Impact**: Limited control over event behavior
   - **Severity**: MEDIUM

7. **Missing Tags Field**
   - **Impact**: Cannot organize events
   - **Severity**: MEDIUM

8. **Missing Associations** (scene, location, act, chain)
   - **Impact**: Cannot link events to game structure
   - **Severity**: MEDIUM

### Low Priority Gaps

9. **Missing Suggested Opening**
   - **Impact**: Less guidance for DM when event triggers
   - **Severity**: LOW

10. **Missing Featured NPCs**
    - **Impact**: Less context for LLM when event triggers
    - **Severity**: LOW

---

## Recommendations

### Immediate Actions (Critical)

1. **Implement Trigger Conditions Section**
   - Add "Add Condition" button
   - Implement trigger type selector
   - Build UI for each trigger type (start with Flag Set, Player Enters Location, Has Item)
   - Add trigger logic selector (All/Any/AtLeast)
   - Add condition preview and delete buttons

2. **Implement Outcomes Section**
   - Add "Add Outcome" button
   - Implement outcome form (name, label, description)
   - Add outcome condition selector
   - Add effects list (start with Set Flag, Give Item)
   - Add chain events list
   - Add default outcome selector

3. **Add Validation**
   - Require at least one trigger condition
   - Require at least one outcome
   - Show validation errors in UI

### Short-term (High Priority)

4. **Add Tags Field**
   - Simple comma-separated input
   - Parse into Vec<String>

5. **Add Configuration Options**
   - Active/Repeatable toggles
   - Priority input
   - Favorite toggle
   - Location/Scene/Act selectors

6. **Implement Draft State**
   - Add "Save Draft" button
   - Add "Save & Activate" button
   - Default to inactive when created

### Medium-term

7. **Complete Trigger Types**
   - Implement UI for all 15 trigger types
   - Add drag-and-drop reordering

8. **Complete Effect Types**
   - Implement UI for all 14 effect types
   - Add effect preview

9. **Add Chain Integration**
   - Chain selector in form
   - Position in chain input

---

## Conclusion

The current narrative event creation form is a **minimal MVP** that allows creating events with basic information only. It lacks the core functionality (triggers and outcomes) that makes narrative events useful. The backend domain model fully supports all features, but the UI and API contracts need significant expansion to match the planned design.

**Estimated Effort to Complete**:
- Critical gaps: 2-3 weeks
- High priority gaps: 1-2 weeks
- Medium priority gaps: 1-2 weeks
- **Total**: 4-7 weeks of development

**Priority**: **HIGH** - This is a core feature that is currently non-functional for its intended purpose.
