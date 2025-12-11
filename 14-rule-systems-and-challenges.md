# Phase 14: Modular Rule Systems & Challenge Mechanics

## Overview

WrldBldr supports multiple TTRPG systems through a modular rule system architecture. This phase introduces:
1. **Dice System Selection** during world creation
2. **Character Sheet Templates** per rule system
3. **Skill Definitions** for challenge resolution
4. **Challenge Mechanics** in Director Mode
5. **LLM + DM Decision Flow** for scene analysis and challenge triggering

**Core Philosophy**: The LLM analyzes scenes and suggests when challenges might be relevant. The DM makes final decisions. This combination drives gameplay forward while keeping the DM in control.

---

## Architecture Overview

```
World Creation
     │
     ├── Select Rule System (D20, D100, Narrative, Custom)
     │         │
     │         └── Loads: Dice mechanics, default skills, sheet template
     │
     └── Customize (optional)
               ├── Add/remove skills
               ├── Modify sheet fields
               └── Define custom dice expressions

Director Mode
     │
     ├── Challenge Library (predefined + custom)
     │         │
     │         └── Skill challenges, ability checks, saves
     │
     ├── Scene Challenge Setup
     │         │
     │         └── Define triggers, conditions, consequences
     │
     └── LLM Analysis Loop
               │
               ├── LLM: "Player examining statue - History check possible?"
               ├── DM: Approve/Reject/Modify
               └── Resolution: Roll → Outcome → Narrative continues
```

---

## User Stories

### Epic: Rule System Selection

#### US-14.1: Select Rule System During World Creation
**As a** Dungeon Master
**I want to** choose a rule system when creating a world
**So that** the game mechanics match my intended TTRPG system

**Acceptance Criteria:**
- Rule system selection appears as step 2 of world creation (after name/description)
- Available systems:
  - D20 System (D&D 5e, Pathfinder)
  - D100 System (Call of Cthulhu, RuneQuest)
  - Narrative (Kids on Bikes, FATE)
  - Custom (build your own)
- Each system shows brief description and example games
- Selection loads appropriate defaults (dice, skills, sheet template)
- Can proceed without customization (use defaults)

**API Changes:**
```rust
// World creation request
POST /api/worlds
{
  "name": "Horror Campaign",
  "description": "...",
  "rule_system": {
    "type": "d100",
    "variant": "call_of_cthulhu_7e"  // optional preset
  }
}
```

---

#### US-14.2: View Rule System Presets
**As a** Dungeon Master
**I want to** see presets for popular game systems
**So that** I can quickly set up a world with familiar rules

**Acceptance Criteria:**
- Each base system has presets:
  - D20: "D&D 5e", "Pathfinder 2e", "Generic D20"
  - D100: "Call of Cthulhu 7e", "RuneQuest", "Generic D100"
  - Narrative: "Kids on Bikes", "FATE Core", "Powered by Apocalypse"
- Selecting a preset auto-fills skills and sheet template
- "Generic" options provide minimal defaults
- Custom system starts blank

**UI: System cards with preset dropdown**

---

#### US-14.3: Preview Rule System Before Selection
**As a** Dungeon Master
**I want to** preview what a rule system includes
**So that** I can make an informed choice

**Acceptance Criteria:**
- Clicking "Preview" on a system shows:
  - Dice mechanics explanation
  - Default skill list
  - Character sheet preview
  - Example challenge resolution
- Preview is read-only
- "Select This System" button in preview

---

### Epic: Character Sheet Templates

#### US-14.4: View Character Sheet Template
**As a** Dungeon Master
**I want to** see the character sheet template for my rule system
**So that** I know what attributes characters will have

**Acceptance Criteria:**
- Character sheet template shown during world creation (optional step)
- Template shows all fields:
  - Attributes/Stats (STR, DEX, etc. or STR, CON, SIZ, etc.)
  - Skills list with categories
  - Derived values (HP, Sanity, AC, etc.)
  - Custom fields section
- Different templates per rule system

---

#### US-14.5: Customize Character Sheet Template
**As a** Dungeon Master
**I want to** customize the character sheet for my world
**So that** I can add house rules or campaign-specific fields

**Acceptance Criteria:**
- Can add custom fields (text, number, checkbox, select)
- Can rename existing fields
- Can hide default fields (not delete - for compatibility)
- Can reorder fields
- Can add skill categories
- Changes apply to all new characters in this world
- Existing characters show new fields with defaults

**Data Model:**
```rust
pub struct CharacterSheetTemplate {
    pub id: TemplateId,
    pub name: String,
    pub rule_system: RuleSystemType,
    pub sections: Vec<SheetSection>,
}

pub struct SheetSection {
    pub name: String,
    pub fields: Vec<SheetField>,
}

pub struct SheetField {
    pub id: String,
    pub label: String,
    pub field_type: FieldType,
    pub default_value: Option<serde_json::Value>,
    pub visible: bool,
    pub order: i32,
}

pub enum FieldType {
    Number { min: Option<i32>, max: Option<i32> },
    Text { max_length: Option<usize> },
    Checkbox,
    Select { options: Vec<String> },
    Skill { base_attribute: Option<String> },
    DerivedValue { formula: String },
}
```

---

#### US-14.6: Create Character with System Sheet
**As a** Dungeon Master
**I want to** create characters using the world's sheet template
**So that** all characters have consistent stats for the rule system

**Acceptance Criteria:**
- Character creation form uses world's template
- All required fields must be filled
- Derived values auto-calculate
- Skills show with their base values
- Can use LLM to suggest stat distributions
- Character saved with all sheet data

---

### Epic: Skill System

#### US-14.7: View World Skills
**As a** Dungeon Master
**I want to** see all skills available in my world
**So that** I know what challenges I can create

**Acceptance Criteria:**
- Skills page in World Settings (Creator Mode > Settings)
- Skills grouped by category:
  - D20: Strength, Dexterity, Intelligence, Wisdom, Charisma, Constitution
  - D100: Interpersonal, Investigation, Academic, Practical, Combat
  - Narrative: Approaches/Aspects
- Each skill shows:
  - Name
  - Associated attribute (if any)
  - Description
  - Example uses

---

#### US-14.8: Add Custom Skill
**As a** Dungeon Master
**I want to** add custom skills to my world
**So that** I can represent unique abilities in my campaign

**Acceptance Criteria:**
- "Add Skill" button in skills management
- Form fields:
  - Skill name (required)
  - Category (select or new)
  - Base attribute (optional)
  - Description
- Skill immediately available for challenges
- Characters can have values for new skill

---

#### US-14.9: Remove or Hide Skill
**As a** Dungeon Master
**I want to** remove skills that don't fit my campaign
**So that** players aren't confused by irrelevant options

**Acceptance Criteria:**
- Can hide default skills (greyed out, not in challenge list)
- Can delete custom skills (with confirmation)
- Hidden skills still exist on characters (not lost)
- Warning if skill is used in active challenges

---

### Epic: Challenge System

#### US-14.10: View Challenge Library
**As a** Dungeon Master
**I want to** see available challenge types
**So that** I can quickly add challenges to scenes

**Acceptance Criteria:**
- Challenge library panel in Director Mode
- Challenges organized by:
  - Skill challenges (one skill roll)
  - Ability checks (raw attribute)
  - Saving throws (reactive checks)
  - Opposed checks (vs NPC or environment)
  - Complex challenges (multiple rolls)
- Quick filter/search
- Favorites/recently used section

---

#### US-14.11: Create Skill Challenge
**As a** Dungeon Master
**I want to** create a skill challenge for the current scene
**So that** players can attempt to overcome obstacles

**Acceptance Criteria:**
- "New Challenge" button in Director panel
- Challenge form:
  - Name/description
  - Skill required (dropdown from world skills)
  - Difficulty (Easy/Medium/Hard/Very Hard or DC number)
  - Success outcome (text or trigger)
  - Failure outcome (text or trigger)
  - Partial success (optional, for narrative systems)
- Challenge attached to current scene
- Can be triggered manually or by condition

**Data Model:**
```rust
pub struct Challenge {
    pub id: ChallengeId,
    pub name: String,
    pub description: String,
    pub challenge_type: ChallengeType,
    pub skill_id: SkillId,
    pub difficulty: Difficulty,
    pub outcomes: ChallengeOutcomes,
    pub trigger_condition: Option<TriggerCondition>,
    pub scene_id: Option<SceneId>,
    pub active: bool,
}

pub enum ChallengeType {
    SkillCheck,
    AbilityCheck,
    SavingThrow,
    OpposedCheck { opponent_skill: SkillId },
    ComplexChallenge { required_successes: u32 },
}

pub enum Difficulty {
    // D20 style
    DC(u32),
    // D100 style
    Percentage(u32),
    // Narrative style
    Descriptor(String), // "Risky", "Desperate"
    // Opposed
    Opposed,
}

pub struct ChallengeOutcomes {
    pub success: Outcome,
    pub failure: Outcome,
    pub partial: Option<Outcome>,
    pub critical_success: Option<Outcome>,
    pub critical_failure: Option<Outcome>,
}

pub struct Outcome {
    pub description: String,
    pub triggers: Vec<OutcomeTrigger>,
}

pub enum OutcomeTrigger {
    RevealInformation(String),
    EnableChallenge(ChallengeId),
    DisableChallenge(ChallengeId),
    ModifyCharacterStat { stat: String, modifier: i32 },
    TriggerScene(SceneId),
    Custom(String),
}
```

---

#### US-14.12: Set Challenge Trigger Condition
**As a** Dungeon Master
**I want to** define when a challenge becomes available
**So that** the LLM can suggest it at the right moment

**Acceptance Criteria:**
- Trigger condition builder:
  - Object interaction: "When player examines [object]"
  - Location: "When player enters [area]"
  - Dialogue: "When player asks about [topic]"
  - Previous challenge: "After [challenge] succeeds/fails"
  - Custom: Free text description
- Multiple conditions can be AND/OR combined
- LLM uses these to recognize opportunities

**Example Triggers:**
```
Challenge: "Notice Secret Door"
Trigger: "Player succeeds at History check on the statue"
         OR "Player explicitly searches the wall"
```

---

#### US-14.13: Chain Challenges Together
**As a** Dungeon Master
**I want to** create challenge chains
**So that** success in one challenge unlocks another

**Acceptance Criteria:**
- In challenge outcomes, can select "Enable Challenge"
- Visual challenge flow in Director panel
- Chain examples:
  - History → Perception → Investigation
  - Persuasion fails → Intimidation available
- Can create branching paths

---

### Epic: LLM Scene Analysis

#### US-14.14: LLM Suggests Challenge Opportunity
**As a** Dungeon Master
**I want** the LLM to recognize when a challenge might be relevant
**So that** I don't miss opportunities for skill checks

**Acceptance Criteria:**
- LLM monitors player actions and dialogue
- When trigger conditions match, LLM suggests challenge
- Suggestion appears in DM approval queue:
  - "Player is examining the ancient statue. Trigger History check?"
  - Shows challenge details
  - DM can: Approve / Reject / Modify
- Approved challenges prompt the player for a roll

**LLM Integration:**
```rust
// Added to LLM context
pub struct SceneAnalysisContext {
    pub active_challenges: Vec<Challenge>,
    pub player_action: String,
    pub scene_description: String,
    pub npc_present: Vec<CharacterId>,
}

// LLM response includes
pub struct LlmResponse {
    pub dialogue: String,
    pub internal_reasoning: String,
    pub proposed_tools: Vec<ProposedTool>,
    pub challenge_suggestions: Vec<ChallengeSuggestion>, // NEW
}

pub struct ChallengeSuggestion {
    pub challenge_id: ChallengeId,
    pub reasoning: String,
    pub confidence: f32, // 0.0 - 1.0
}
```

---

#### US-14.15: DM Approves Challenge Trigger
**As a** Dungeon Master
**I want to** approve or reject LLM challenge suggestions
**So that** I control the pacing and difficulty

**Acceptance Criteria:**
- Challenge suggestion appears in approval panel
- Shows:
  - Challenge name and skill
  - Why LLM thinks it's relevant
  - Difficulty
  - Potential outcomes
- DM options:
  - **Approve**: Trigger the challenge
  - **Reject**: Continue without check
  - **Modify**: Change difficulty or skill
  - **Delay**: "Not yet, but remember this"
- Rejected challenges don't repeat immediately

---

#### US-14.16: Resolve Challenge Roll
**As a** Dungeon Master
**I want to** resolve a challenge and see the outcome
**So that** the narrative can continue

**Acceptance Criteria:**
- After approval, roll interface appears:
  - Character's skill/modifier shown
  - Difficulty shown
  - "Roll" button (or manual entry)
  - Dice animation
- Result calculated and compared to difficulty
- Outcome displayed:
  - Success/Failure/Partial
  - Outcome text
  - Any triggers activated
- Result logged to conversation
- LLM receives outcome for narrative continuity

---

#### US-14.17: Manual Challenge Trigger
**As a** Dungeon Master
**I want to** manually trigger a challenge at any time
**So that** I can call for checks when I see fit

**Acceptance Criteria:**
- "Trigger Challenge" button in Director panel
- Opens challenge picker (from library or scene)
- Can quick-create ad-hoc challenge
- Bypasses LLM suggestion flow
- Same resolution flow as LLM-triggered

---

### Epic: Player Experience

#### US-14.18: Player Sees Challenge Prompt
**As a** Player
**I want to** see when I need to make a skill check
**So that** I can roll and see the result

**Acceptance Criteria:**
- Challenge notification appears in PC View
- Shows:
  - "The DM calls for a [Skill] check"
  - Difficulty (if DM chooses to reveal)
  - My modifier for this skill
  - "Roll" button
- Roll result shown with drama (suspense)
- Outcome narrated by LLM/DM

---

#### US-14.19: Player Views Character Skills
**As a** Player
**I want to** see my character's skills and modifiers
**So that** I know my chances before rolling

**Acceptance Criteria:**
- Character sheet accessible in PC View
- Skills tab shows all skills with values
- Modifier calculation shown (base + bonuses)
- Recently used skills highlighted
- Can see skill descriptions

---

## UI Mockups

### World Creation - Rule System Selection

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Create New World                                                     [×]   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Step 2 of 3: Choose Rule System                                           │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                  │
│  │  🎲 D20 SYSTEM          │  │  📊 D100 SYSTEM         │                  │
│  │                         │  │                         │                  │
│  │  Roll d20 + modifier    │  │  Roll percentile dice   │                  │
│  │  vs Difficulty Class    │  │  under skill value      │                  │
│  │                         │  │                         │                  │
│  │  ┌─────────────────┐    │  │  ┌─────────────────┐    │                  │
│  │  │ D&D 5e        ▼ │    │  │  │ CoC 7e        ▼ │    │                  │
│  │  └─────────────────┘    │  │  └─────────────────┘    │                  │
│  │                         │  │                         │                  │
│  │  [Preview] [Select ✓]   │  │  [Preview] [Select]     │                  │
│  └─────────────────────────┘  └─────────────────────────┘                  │
│                                                                             │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                  │
│  │  📖 NARRATIVE           │  │  🔧 CUSTOM              │                  │
│  │                         │  │                         │                  │
│  │  Fiction-first with     │  │  Build your own system  │                  │
│  │  descriptive outcomes   │  │  from scratch           │                  │
│  │                         │  │                         │                  │
│  │  ┌─────────────────┐    │  │                         │                  │
│  │  │ Kids on Bikes ▼ │    │  │  Dice, skills, and     │                  │
│  │  └─────────────────┘    │  │  sheet fully custom     │                  │
│  │                         │  │                         │                  │
│  │  [Preview] [Select]     │  │  [Preview] [Select]     │                  │
│  └─────────────────────────┘  └─────────────────────────┘                  │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  [← Back]                                              [Next: Skills →]    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Rule System Preview Modal

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  D100 System: Call of Cthulhu 7th Edition                            [×]   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DICE MECHANICS                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│  Roll d100 (two d10s). Success if roll ≤ skill value.                      │
│                                                                             │
│  • Regular Success: Roll ≤ skill                                           │
│  • Hard Success: Roll ≤ skill/2                                            │
│  • Extreme Success: Roll ≤ skill/5                                         │
│  • Fumble: Roll 96-100 (or 100 if skill > 50)                             │
│                                                                             │
│  ATTRIBUTES                          SAMPLE SKILLS                         │
│  ─────────────────────────────────   ─────────────────────────────────────  │
│  • STR (Strength)                    Interpersonal:                        │
│  • CON (Constitution)                  Charm, Fast Talk, Intimidate,       │
│  • SIZ (Size)                          Persuade                            │
│  • DEX (Dexterity)                                                         │
│  • APP (Appearance)                  Investigation:                        │
│  • INT (Intelligence)                  Library Use, Spot Hidden,           │
│  • POW (Power)                         Listen, Psychology                  │
│  • EDU (Education)                                                         │
│  • Luck                              Academic:                             │
│                                        Accounting, History, Law,           │
│  DERIVED VALUES                        Occult, Science (various)           │
│  ─────────────────────────────────                                         │
│  • HP = (CON + SIZ) / 10             Combat:                               │
│  • Sanity = POW                        Fighting (Brawl), Firearms,         │
│  • Magic Points = POW / 5              Dodge                               │
│  • Move Rate = based on STR/DEX/SIZ                                        │
│                                                                             │
│  EXAMPLE CHALLENGE                                                          │
│  ─────────────────────────────────────────────────────────────────────────  │
│  "Investigate the strange tome"                                            │
│  Skill: Library Use (base 20%)                                             │
│  Player has 45% → Rolls 32 → Regular Success!                              │
│  Outcome: Discovers the book is a copy of the Necronomicon                 │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│                                              [Cancel]  [Select This System] │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Director Mode - Challenge Panel

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DIRECTOR MODE                                           [Director] [Creator] [Settings]
├────────────────────────────────────────────────┬────────────────────────────┤
│                                                │                            │
│  SCENE: The Dusty Library                      │  CHALLENGES                │
│  ────────────────────────────────────────────  │  ────────────────────────  │
│                                                │                            │
│  [Scene preview area]                          │  Scene Challenges:         │
│                                                │  ┌──────────────────────┐  │
│                                                │  │ 📚 Research the Tome │  │
│  CONVERSATION LOG                              │  │ Library Use • Hard   │  │
│  ────────────────────────────────────────────  │  │ [Trigger] [Edit] [×] │  │
│                                                │  └──────────────────────┘  │
│  Player: "I want to examine the old books     │  ┌──────────────────────┐  │
│  on that shelf."                               │  │ 🔍 Notice Hidden Door│  │
│                                                │  │ Spot Hidden • Medium │  │
│  ┌────────────────────────────────────────┐   │  │ Requires: Research ✓ │  │
│  │ 🤖 LLM SUGGESTION                      │   │  │ [Trigger] [Edit] [×] │  │
│  │                                        │   │  └──────────────────────┘  │
│  │ Player examining books - Library Use   │   │                            │
│  │ check to find useful information?      │   │  [+ Add Challenge]         │
│  │                                        │   │                            │
│  │ Challenge: "Research the Tome"         │   │  ────────────────────────  │
│  │ Difficulty: Hard (skill/2)             │   │  QUICK CHALLENGES          │
│  │                                        │   │                            │
│  │ [Approve ✓] [Modify ✏️] [Reject ✗]     │   │  [Spot Hidden]             │
│  └────────────────────────────────────────┘   │  [Listen]                  │
│                                                │  [Psychology]              │
│  NPC Responses awaiting approval: 0            │  [Fast Talk]               │
│                                                │  [+ More...]               │
│                                                │                            │
└────────────────────────────────────────────────┴────────────────────────────┘
```

### Challenge Creation Modal

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Create Challenge                                                    [×]   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Name *                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ Research the Ancient Tome                                             │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  Description                                                                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ Carefully examining the tome to understand its contents and origin    │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  Skill *                           Difficulty *                            │
│  ┌─────────────────────────┐      ┌─────────────────────────┐             │
│  │ Library Use           ▼ │      │ Hard (skill/2)        ▼ │             │
│  └─────────────────────────┘      └─────────────────────────┘             │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  OUTCOMES                                                                   │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│  On Success:                                                                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ You discover this is a rare copy of the Necronomicon. The binding    │ │
│  │ contains a hidden map showing a location in the Miskatonic Valley.    │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│  Triggers: [+ Enable "Notice Hidden Door"] [+ Reveal Information]          │
│                                                                             │
│  On Failure:                                                                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ The text is in a language you don't recognize. You feel uneasy       │ │
│  │ looking at the strange symbols.                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│  Triggers: [+ Sanity Check (0/1)]                                          │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  TRIGGER CONDITIONS (when should LLM suggest this?)                        │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ ○ When player examines [the old tome / books / shelf]                │ │
│  │ ○ When player asks about [the strange symbols / the book]            │ │
│  │ ● Custom: Player shows interest in researching the library           │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  [Cancel]                                              [Create Challenge]   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Challenge Resolution Interface

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SKILL CHECK                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         📚 Library Use                                      │
│                                                                             │
│                    ┌─────────────────────────┐                              │
│                    │                         │                              │
│                    │     Harvey Walters      │                              │
│                    │                         │                              │
│                    │     Skill: 45%          │                              │
│                    │     Difficulty: Hard    │                              │
│                    │     Target: ≤ 22        │                              │
│                    │                         │                              │
│                    └─────────────────────────┘                              │
│                                                                             │
│                         ┌─────────────┐                                     │
│                         │             │                                     │
│                         │    🎲 32    │                                     │
│                         │             │                                     │
│                         │   FAILURE   │                                     │
│                         │             │                                     │
│                         └─────────────┘                                     │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                             │
│  The text is in a language you don't recognize. You feel uneasy looking    │
│  at the strange symbols. Something about them seems... wrong.              │
│                                                                             │
│  [Sanity Check triggered: 0/1]                                             │
│                                                                             │
│                                                        [Continue →]         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### PC View - Challenge Notification

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  THE DUSTY LIBRARY                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [Visual novel scene with library background and characters]                │
│                                                                             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  The DM calls for a Library Use check.                               │   │
│  │                                                                       │   │
│  │  ┌───────────────────────────────────────────┐                       │   │
│  │  │  Your Library Use: 45%                    │                       │   │
│  │  │  Difficulty: Hard (need ≤ 22)             │                       │   │
│  │  └───────────────────────────────────────────┘                       │   │
│  │                                                                       │   │
│  │                        [🎲 Roll]                                      │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────   │
│  [Inventory]  [Character Sheet]  [Journal]                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Skills Management (Creator Mode > Settings)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  World Settings > Skills                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Rule System: Call of Cthulhu 7e (D100)                    [Change System]  │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│  INTERPERSONAL                                           [+ Add Skill]      │
│  ─────────────────────────────────────────────────────────────────────────  │
│  ┌────────────────┬─────────┬────────────────────────────────┬──────────┐  │
│  │ Skill          │ Base    │ Description                    │ Actions  │  │
│  ├────────────────┼─────────┼────────────────────────────────┼──────────┤  │
│  │ Charm          │ 15%     │ Physical appeal and attraction │ [✏️] [👁️] │  │
│  │ Fast Talk      │ 05%     │ Quickly convince or confuse    │ [✏️] [👁️] │  │
│  │ Intimidate     │ 15%     │ Frighten or bully others       │ [✏️] [👁️] │  │
│  │ Persuade       │ 10%     │ Change someone's mind          │ [✏️] [👁️] │  │
│  └────────────────┴─────────┴────────────────────────────────┴──────────┘  │
│                                                                             │
│  INVESTIGATION                                           [+ Add Skill]      │
│  ─────────────────────────────────────────────────────────────────────────  │
│  ┌────────────────┬─────────┬────────────────────────────────┬──────────┐  │
│  │ Library Use    │ 20%     │ Find information in books      │ [✏️] [👁️] │  │
│  │ Listen         │ 20%     │ Hear sounds and eavesdrop      │ [✏️] [👁️] │  │
│  │ Psychology     │ 10%     │ Read people's intentions       │ [✏️] [👁️] │  │
│  │ Spot Hidden    │ 25%     │ Notice concealed things        │ [✏️] [👁️] │  │
│  └────────────────┴─────────┴────────────────────────────────┴──────────┘  │
│                                                                             │
│  CUSTOM SKILLS                                           [+ Add Skill]      │
│  ─────────────────────────────────────────────────────────────────────────  │
│  ┌────────────────┬─────────┬────────────────────────────────┬──────────┐  │
│  │ Mythos Lore    │ 00%     │ Knowledge of eldritch beings   │ [✏️] [🗑️] │  │
│  └────────────────┴─────────┴────────────────────────────────┴──────────┘  │
│                                                                             │
│  👁️ = Toggle visibility   ✏️ = Edit   🗑️ = Delete (custom only)            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Model Summary

```rust
// Rule System
pub enum RuleSystemType {
    D20,
    D100,
    Narrative,
    Custom,
}

pub struct RuleSystem {
    pub system_type: RuleSystemType,
    pub variant: Option<String>,  // "dnd_5e", "coc_7e", etc.
    pub dice_expression: String,  // "d20", "d100", "2d6", etc.
    pub success_comparison: SuccessComparison,
}

pub enum SuccessComparison {
    GreaterOrEqual,  // D20: roll >= DC
    LessOrEqual,     // D100: roll <= skill
    Matches,         // Narrative: specific dice faces
}

// Skills
pub struct Skill {
    pub id: SkillId,
    pub name: String,
    pub category: String,
    pub base_value: Option<i32>,
    pub base_attribute: Option<String>,
    pub description: String,
    pub is_custom: bool,
    pub visible: bool,
}

// Challenge
pub struct Challenge {
    pub id: ChallengeId,
    pub name: String,
    pub description: String,
    pub challenge_type: ChallengeType,
    pub skill_id: SkillId,
    pub difficulty: Difficulty,
    pub outcomes: ChallengeOutcomes,
    pub trigger_conditions: Vec<TriggerCondition>,
    pub scene_id: Option<SceneId>,
    pub active: bool,
    pub prerequisite_challenges: Vec<ChallengeId>,
}

pub struct TriggerCondition {
    pub condition_type: TriggerType,
    pub description: String,
}

pub enum TriggerType {
    ObjectInteraction(String),
    EnterArea(String),
    DialogueTopic(String),
    ChallengeComplete(ChallengeId),
    Custom(String),
}
```

---

## Implementation Phases

### Phase 14A: Rule System Selection
- [ ] Add RuleSystem to World entity
- [ ] Create rule system presets (D20, D100, Narrative)
- [ ] Update world creation UI with system selection
- [ ] Fix DiceSystem deserialization error

### Phase 14B: Skill System
- [ ] Create Skill entity and repository
- [ ] Populate default skills per rule system
- [ ] Skills management UI in Creator Mode
- [ ] Add/edit/hide skills

### Phase 14C: Character Sheet Templates
- [ ] Create CharacterSheetTemplate entity
- [ ] Default templates per rule system
- [ ] Update character creation to use template
- [ ] Character sheet viewer in PC View

### Phase 14D: Challenge System Core
- [ ] Create Challenge entity and repository
- [ ] Challenge CRUD API endpoints
- [ ] Challenge library UI in Director Mode
- [ ] Manual challenge triggering

### Phase 14E: LLM Challenge Integration
- [ ] Add challenges to LLM context
- [ ] LLM challenge suggestion in response
- [ ] Challenge suggestion approval UI
- [ ] Challenge resolution flow

### Phase 14F: Challenge Chaining
- [ ] Challenge prerequisites and outcomes
- [ ] Trigger condition builder UI
- [ ] Visual challenge flow editor
- [ ] Challenge state tracking per session

---

## API Endpoints

```
# Rule Systems
GET  /api/rule-systems                    # List available rule systems
GET  /api/rule-systems/{type}/presets     # Get presets for a system
GET  /api/rule-systems/{type}/skills      # Get default skills

# World Rule System
GET  /api/worlds/{id}/rule-system         # Get world's rule system
PUT  /api/worlds/{id}/rule-system         # Update world's rule system

# Skills
GET  /api/worlds/{id}/skills              # List world skills
POST /api/worlds/{id}/skills              # Add custom skill
PUT  /api/worlds/{id}/skills/{skill_id}   # Update skill
DELETE /api/worlds/{id}/skills/{skill_id} # Delete custom skill

# Character Sheets
GET  /api/worlds/{id}/sheet-template      # Get world's sheet template
PUT  /api/worlds/{id}/sheet-template      # Update sheet template

# Challenges
GET  /api/worlds/{id}/challenges          # List all challenges
GET  /api/scenes/{id}/challenges          # List scene challenges
POST /api/challenges                      # Create challenge
PUT  /api/challenges/{id}                 # Update challenge
DELETE /api/challenges/{id}               # Delete challenge
POST /api/challenges/{id}/trigger         # Manually trigger challenge
POST /api/challenges/{id}/resolve         # Submit roll result
```

---

## Questions for Clarification

1. **Dice Rolling**: Should dice rolls happen client-side (player rolls) or server-side (for fairness)?

2. **Skill Inheritance**: Should characters inherit base skills from the world template, or copy them (allowing per-character modification)?

3. **Challenge Visibility**: Should players see upcoming challenges, or only when triggered?

4. **Partial Successes**: For D20/D100, should we support "degrees of success" beyond pass/fail?

5. **Opposed Checks**: How should NPC vs Player checks work? Roll both, or use static NPC values?

---

## Dependencies

- **Requires**: World creation (Phase 13) - for rule system selection
- **Requires**: LLM integration (existing) - for challenge suggestions
- **Enhances**: Director Mode - with challenge panel
- **Enhances**: PC View - with roll interface
