# Phase 12: ComfyUI Workflow Settings & Configuration

## Background Research

### ComfyUI API Workflow Format

ComfyUI uses two JSON formats:
1. **Regular Workflow** - Contains visual information (node positions, UI settings)
2. **API Format** - Stripped-down version for programmatic execution

The API format structure:
```json
{
  "3": {
    "class_type": "KSampler",
    "inputs": {
      "seed": 156680208700286,
      "steps": 20,
      "cfg": 8,
      "sampler_name": "euler",
      "scheduler": "normal",
      "denoise": 1,
      "model": ["4", 0],
      "positive": ["6", 0],
      "negative": ["7", 0],
      "latent_image": ["5", 0]
    }
  },
  "6": {
    "class_type": "CLIPTextEncode",
    "inputs": {
      "text": "a beautiful landscape",
      "clip": ["4", 1]
    }
  }
}
```

**Key insight**: Inputs that are arrays like `["4", 0]` are **node connections** (node_id, output_index). Inputs that are primitive values (strings, numbers, booleans) are **user-configurable parameters**.

### Sources
- [ComfyUI Workflow JSON Spec](https://docs.comfy.org/specs/workflow_json)
- [9elements: Hosting ComfyUI via API](https://9elements.com/blog/hosting-a-comfyui-workflow-via-api/)
- [ViewComfy: Production-Ready ComfyUI API](https://www.viewcomfy.com/blog/building-a-production-ready-comfyui-api)

---

## User Stories

### Epic: Workflow Management

#### US-12.1: View Available Workflow Slots
**As a** Director/Creator
**I want to** see which workflow slots are configured and which are empty
**So that** I know what asset types I can generate

**Acceptance Criteria:**
- Display a list of all workflow slots (Character Portrait, Character Sprite, Scene Backdrop, etc.)
- Show status for each: "Configured" (green), "Not Configured" (gray)
- Show the workflow name/file for configured slots
- Quick access to configure each slot

---

#### US-12.2: Upload Workflow JSON File
**As a** Director/Creator
**I want to** upload a ComfyUI workflow JSON file (API format)
**So that** I can use my custom workflows for asset generation

**Acceptance Criteria:**
- File picker that accepts `.json` files
- Validate that the file is valid ComfyUI API format
- Display parsing errors if invalid
- Show preview of detected nodes before confirming
- Store the workflow for the selected slot

---

#### US-12.3: Paste Workflow JSON
**As a** Director/Creator
**I want to** paste workflow JSON directly from clipboard
**So that** I can quickly import workflows without saving to file first

**Acceptance Criteria:**
- Large textarea for pasting JSON
- "Paste from Clipboard" button for convenience
- Same validation as file upload
- Format/prettify button to clean up pasted JSON

---

#### US-12.4: View Workflow Input Parameters
**As a** Director/Creator
**I want to** see all configurable inputs in my workflow
**So that** I understand what can be customized

**Acceptance Criteria:**
- Parse workflow JSON and extract all non-connection inputs
- Display inputs grouped by node (with node class_type as header)
- Show input name, current value, and detected type
- Distinguish between: text fields, numbers, selects (if enum-like), checkboxes (booleans)

---

#### US-12.5: Set Default Values for Workflow Inputs
**As a** Director/Creator
**I want to** configure default values for workflow inputs
**So that** I don't have to set common values every time I generate

**Acceptance Criteria:**
- Form fields for each detected input parameter
- Appropriate input types based on detected value type
- "Reset to Original" button per field
- "Save Defaults" to persist configuration
- Mark which inputs should be "locked" (always use default, never prompt)

---

#### US-12.6: Mark Inputs as Prompt Fields
**As a** Director/Creator
**I want to** specify which text inputs should receive the generation prompt
**So that** the system knows where to inject character/location descriptions

**Acceptance Criteria:**
- Identify all text inputs in the workflow
- Allow marking one or more as "Primary Prompt" field
- Allow marking inputs as "Negative Prompt" field
- Show indicator on prompt-mapped fields
- Validation: at least one prompt field required for usable workflow

---

#### US-12.7: Override Defaults at Generation Time
**As a** Director/Creator
**I want to** adjust parameters before generating
**So that** I can fine-tune specific generations without changing global defaults

**Acceptance Criteria:**
- "Advanced Options" expandable section in generation modal
- Show all non-locked inputs with current default values
- Allow editing any value for this generation only
- Changes don't persist to defaults

---

#### US-12.8: Test Workflow Configuration
**As a** Director/Creator
**I want to** test my workflow configuration
**So that** I can verify it works before using it for actual assets

> **Queue Integration**: Test generation requests appear in the [Unified Generation Queue](./15-unified-generation-queue.md), providing consistent progress visibility.

**Acceptance Criteria:**
- "Test Workflow" button in settings
- Opens modal with test prompt input
- Runs generation with current defaults
- Test request appears in unified generation queue with progress
- Shows progress and result
- Option to save test output or discard

---

#### US-12.9: Export/Import Workflow Configurations
**As a** Director/Creator
**I want to** export my workflow configurations
**So that** I can share them or back them up

**Acceptance Criteria:**
- Export all workflow configurations as single JSON file
- Import configurations from exported file
- Merge or replace options on import
- Include: workflow JSON, defaults, prompt mappings

---

### Epic: Generation UI Integration

> **Note**: All generation requests flow through the [Unified Generation Queue](./15-unified-generation-queue.md), which handles both image generation (ComfyUI) and LLM suggestions (Ollama) in a single queue panel.

#### US-12.10: Dynamic Generation Form
**As a** Director/Creator
**I want to** see a generation form tailored to my workflow
**So that** I can control all available parameters

**Acceptance Criteria:**
- Generation modal shows workflow-specific inputs
- Locked inputs are hidden
- Prompt field is prominent at top
- Advanced inputs in collapsible section
- Seed field with "Random" and "Reuse Last" options
- Submitted requests appear in unified generation queue

---

#### US-12.11: Workflow Selection
**As a** Director/Creator
**I want to** choose which workflow to use when generating
**So that** I can use different workflows for different styles

**Acceptance Criteria:**
- If multiple workflows configured for same type, show selector
- Preview workflow name and last-used date
- Default to most recently used

---

## UI Mockups

### Settings Page - Workflow Configuration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Settings                                                        [× Close]  │
├─────────────────────────────────────────────────────────────────────────────┤
│  [Connection]  [Workflows]  [Defaults]  [About]                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WORKFLOW CONFIGURATION                                                     │
│                                                                             │
│  Configure ComfyUI workflows for each asset type. Upload or paste           │
│  workflow JSON files exported from ComfyUI in API format.                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  CHARACTER ASSETS                                                   │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  Portrait (256×256)                                                 │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │  ✓ Configured: sdxl_portrait_v2.json                        │   │    │
│  │  │  Nodes: 8 | Inputs: 12 | Prompt field: CLIPTextEncode.text  │   │    │
│  │  │  [Configure] [Test] [Remove]                                │   │    │
│  │  └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                     │    │
│  │  Sprite (512×512)                                                   │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │  ○ Not Configured                                           │   │    │
│  │  │  [Upload Workflow] [Paste JSON]                             │   │    │
│  │  └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                     │    │
│  │  Expression Sheet (768×768)                                         │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │  ○ Not Configured                                           │   │    │
│  │  │  [Upload Workflow] [Paste JSON]                             │   │    │
│  │  └─────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  LOCATION ASSETS                                                    │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  Backdrop (1920×1080)                                               │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │  ✓ Configured: flux_landscape.json                          │   │    │
│  │  │  Nodes: 12 | Inputs: 18 | Prompt field: positive_prompt     │   │    │
│  │  │  [Configure] [Test] [Remove]                                │   │    │
│  │  └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                     │    │
│  │  Tilesheet (512×512)                                                │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │  ○ Not Configured                                           │   │    │
│  │  │  [Upload Workflow] [Paste JSON]                             │   │    │
│  │  └─────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  ITEM ASSETS                                                        │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  Icon (64×64)                                                       │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │  ○ Not Configured                                           │   │    │
│  │  │  [Upload Workflow] [Paste JSON]                             │   │    │
│  │  └─────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  [Import All] [Export All]                                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Workflow Upload/Paste Modal

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Configure Workflow: Character Portrait                          [× Close]  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [Upload File]  [Paste JSON]                                    ← Tabs      │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  Paste your ComfyUI workflow JSON (API format) below:                       │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ {                                                                   │    │
│  │   "3": {                                                            │    │
│  │     "class_type": "KSampler",                                       │    │
│  │     "inputs": {                                                     │    │
│  │       "seed": 156680208700286,                                      │    │
│  │       "steps": 20,                                                  │    │
│  │       "cfg": 8,                                                     │    │
│  │       ...                                                           │    │
│  │                                                                     │    │
│  │                                                                     │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  [📋 Paste from Clipboard]  [Format JSON]               300 lines, valid ✓  │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  DETECTED STRUCTURE:                                                        │
│  • 8 nodes detected                                                         │
│  • 14 configurable inputs found                                             │
│  • Text inputs: 2 (likely prompt fields)                                    │
│                                                                             │
│  [Cancel]                                              [Continue Setup →]   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Workflow Configuration - Input Mapping

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Configure: Character Portrait                           [← Back] [× Close] │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROMPT FIELD MAPPING                                                       │
│  Select which text fields should receive the generation prompt              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  ○ Node 6: CLIPTextEncode → "text"                                  │    │
│  │    Current: "a beautiful portrait"                                  │    │
│  │    [Set as Primary Prompt ✓]  [Set as Negative Prompt]              │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  ○ Node 7: CLIPTextEncode → "text"                                  │    │
│  │    Current: "ugly, blurry, low quality"                             │    │
│  │    [Set as Primary Prompt]  [Set as Negative Prompt ✓]              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  DEFAULT VALUES                                                             │
│  Configure defaults for other parameters                                    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  KSampler (Node 3)                                                  │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │                                                                     │    │
│  │  steps          [20        ]  ☐ Lock                                │    │
│  │  cfg            [7.5       ]  ☐ Lock                                │    │
│  │  sampler_name   [euler_a  ▼]  ☐ Lock                                │    │
│  │  scheduler      [normal   ▼]  ☐ Lock                                │    │
│  │  denoise        [1.0       ]  ☑ Lock                                │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  EmptyLatentImage (Node 5)                                          │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │                                                                     │    │
│  │  width          [256       ]  ☑ Lock                                │    │
│  │  height         [256       ]  ☑ Lock                                │    │
│  │  batch_size     [4         ]  ☐ Lock                                │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  [Cancel]                                                    [Save Config]  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Generation Modal with Workflow Options

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Generate Character Portrait                                     [× Close]  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROMPT                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ A portrait of Eldric, an elderly wizard with a long silver beard,  │    │
│  │ piercing blue eyes, wearing a deep purple robe with golden         │    │
│  │ embroidery. Wise and weathered expression.                         │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│  [✨ Suggest from Character]                                                │
│                                                                             │
│  NEGATIVE PROMPT (optional)                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ ugly, deformed, blurry, low quality, watermark                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Variations:  [1] [2] [4] [8]              ← How many to generate           │
│                                                                             │
│  ▼ Advanced Options                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                                                                     │    │
│  │  Seed:        [Random ▼]  [_____________]                          │    │
│  │                                                                     │    │
│  │  Steps:       [20        ]  (default: 20)                          │    │
│  │  CFG Scale:   [7.5       ]  (default: 7.5)                         │    │
│  │  Sampler:     [euler_a  ▼]  (default: euler_a)                     │    │
│  │  Batch Size:  [4         ]  (default: 4)                           │    │
│  │                                                                     │    │
│  │  [Reset to Defaults]                                               │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Using workflow: sdxl_portrait_v2.json                                      │
│                                                                             │
│  [Cancel]                                                      [Generate]   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Test Workflow Modal

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Test Workflow: Character Portrait                               [× Close]  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Enter a test prompt to verify your workflow configuration works:           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ A test portrait of a fantasy character                             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  RESULT                                                                     │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                                                                     │    │
│  │                    ┌──────────┐                                     │    │
│  │                    │          │                                     │    │
│  │                    │  [img]   │                                     │    │
│  │                    │          │                                     │    │
│  │                    └──────────┘                                     │    │
│  │                                                                     │    │
│  │                  Generation successful!                             │    │
│  │                  Time: 4.2s | Size: 256×256                        │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  [Run Another Test]                           [Save as Default] [Discard]   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Model

### WorkflowConfiguration Entity

```rust
pub struct WorkflowConfiguration {
    pub id: WorkflowConfigId,
    pub slot: WorkflowSlot,                    // CharacterPortrait, Backdrop, etc.
    pub name: String,                          // User-friendly name
    pub workflow_json: serde_json::Value,      // The raw ComfyUI API workflow
    pub prompt_mappings: Vec<PromptMapping>,   // Which inputs receive prompts
    pub input_defaults: Vec<InputDefault>,     // Default values
    pub locked_inputs: Vec<String>,            // Input paths that are locked
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

pub struct PromptMapping {
    pub node_id: String,
    pub input_name: String,
    pub mapping_type: PromptMappingType,       // Primary, Negative
}

pub enum PromptMappingType {
    Primary,
    Negative,
}

pub struct InputDefault {
    pub node_id: String,
    pub input_name: String,
    pub default_value: serde_json::Value,
}

pub enum WorkflowSlot {
    CharacterPortrait,
    CharacterSprite,
    CharacterExpressionSheet,
    LocationBackdrop,
    LocationTilesheet,
    ItemIcon,
    // Future: Map, Effect, etc.
}

/// Parsed input from workflow for UI display
pub struct WorkflowInput {
    pub node_id: String,
    pub node_type: String,
    pub input_name: String,
    pub input_type: InputType,
    pub current_value: serde_json::Value,
    pub is_connection: bool,
}

pub enum InputType {
    Text,
    Integer,
    Float,
    Boolean,
    Select(Vec<String>),  // If we can detect enum options
    Unknown,
}
```

---

## API Endpoints

```
# Workflow Configuration
GET    /api/workflows                           # List all configurations
GET    /api/workflows/{slot}                    # Get config for slot
POST   /api/workflows/{slot}                    # Create/update config
DELETE /api/workflows/{slot}                    # Remove config

# Workflow Analysis
POST   /api/workflows/analyze                   # Parse JSON, return inputs
POST   /api/workflows/{slot}/test               # Test workflow with prompt

# Import/Export
GET    /api/workflows/export                    # Export all configs
POST   /api/workflows/import                    # Import configs
```

---

## Implementation Phases

### Phase 12A: Workflow Storage & Analysis ✅ COMPLETE
- [x] Create `WorkflowConfiguration` entity and repository
- [x] Implement workflow JSON parser to extract inputs
- [x] Distinguish connection inputs from value inputs
- [x] Create analysis endpoint (`POST /api/workflows/analyze`)

### Phase 12B: Settings UI - Workflow List ✅ COMPLETE
- [x] Add Settings page/modal to Player (Settings tab in DM View)
- [x] Create workflow slot list component (`workflow_slot_list.rs`)
- [x] Show configured vs unconfigured slots
- [x] Add upload and paste entry points

### Phase 12C: Workflow Configuration UI ✅ COMPLETE
- [x] Create workflow paste/upload modal (`workflow_upload_modal.rs`)
- [x] Build input mapping interface
- [x] Implement defaults configuration form (`workflow_config_editor.rs`)
- [x] Add lock/unlock functionality

### Phase 12D: Generation Integration ✅ COMPLETE
- [x] Update generation modal to read workflow config
- [x] Build dynamic form from workflow inputs (GenerateAssetModal)
- [x] Implement override UI for non-locked inputs
- [x] Add seed handling (random seed by default)

### Phase 12E: Testing & Polish ✅ COMPLETE
- [x] Add workflow test functionality (Test modal with image preview)
- [ ] Implement import/export (deferred)
- [x] Add validation and error handling
- [x] Create helpful error messages for common issues

---

## Technical Notes

### Parsing Workflow Inputs

```rust
fn extract_inputs(workflow: &serde_json::Value) -> Vec<WorkflowInput> {
    let mut inputs = Vec::new();

    if let Some(nodes) = workflow.as_object() {
        for (node_id, node) in nodes {
            let class_type = node.get("class_type")
                .and_then(|v| v.as_str())
                .unwrap_or("Unknown");

            if let Some(node_inputs) = node.get("inputs").and_then(|v| v.as_object()) {
                for (input_name, value) in node_inputs {
                    // Connection inputs are arrays: ["node_id", output_index]
                    let is_connection = value.is_array();

                    if !is_connection {
                        inputs.push(WorkflowInput {
                            node_id: node_id.clone(),
                            node_type: class_type.to_string(),
                            input_name: input_name.clone(),
                            input_type: detect_type(value),
                            current_value: value.clone(),
                            is_connection: false,
                        });
                    }
                }
            }
        }
    }

    inputs
}

fn detect_type(value: &serde_json::Value) -> InputType {
    match value {
        serde_json::Value::String(_) => InputType::Text,
        serde_json::Value::Number(n) => {
            if n.is_i64() { InputType::Integer } else { InputType::Float }
        }
        serde_json::Value::Bool(_) => InputType::Boolean,
        _ => InputType::Unknown,
    }
}
```

### Applying Prompt at Generation Time

```rust
fn apply_generation_params(
    workflow: &mut serde_json::Value,
    config: &WorkflowConfiguration,
    prompt: &str,
    negative_prompt: Option<&str>,
    overrides: &HashMap<String, serde_json::Value>,
) {
    // Apply prompt mappings
    for mapping in &config.prompt_mappings {
        let text = match mapping.mapping_type {
            PromptMappingType::Primary => prompt,
            PromptMappingType::Negative => negative_prompt.unwrap_or(""),
        };

        if let Some(node) = workflow.get_mut(&mapping.node_id) {
            if let Some(inputs) = node.get_mut("inputs") {
                inputs[&mapping.input_name] = serde_json::Value::String(text.to_string());
            }
        }
    }

    // Apply defaults (unless overridden)
    for default in &config.input_defaults {
        let key = format!("{}.{}", default.node_id, default.input_name);
        if !overrides.contains_key(&key) {
            // Apply default...
        }
    }

    // Apply overrides
    for (key, value) in overrides {
        // Parse "node_id.input_name" and apply...
    }

    // Randomize seed if needed
    randomize_seed(workflow);
}
```
