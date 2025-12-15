# Creator Mode Reactive Architecture Refactor

**Created**: 2025-12-15
**Purpose**: Replace refresh counter hack with proper reactive signal patterns

---

## Current Architecture Problems

### Issue 1: Refresh Counter Hack
- `CreatorMode` uses `refresh_counter: Signal<u32>` that gets incremented
- `CharacterList` and `LocationList` watch this counter in `use_effect`
- When counter changes, they re-fetch from API
- **Problem**: This is a workaround, not proper reactivity

### Issue 2: Data Duplication
- Each list component (`CharacterList`, `LocationList`) maintains its own `Signal<Vec<...>>`
- Data is fetched independently in each component
- No shared state - if two components need the same data, they both fetch it

### Issue 3: Unnecessary Re-renders
- Counter increment triggers full re-fetch even if data hasn't changed
- No way to do optimistic updates (add item immediately, sync later)
- No way to update a single item without re-fetching entire list

---

## Proper Reactive Pattern (Following Existing Codebase Patterns)

Looking at `SessionState` and `GenerationState`, the pattern is clear:

1. **Store data in Signals at parent level**
   - `SessionState` has `Signal<Vec<PendingApproval>>`
   - `GenerationState` has `Signal<Vec<GenerationBatch>>`
   - Methods like `add_pending_approval()` use `.write()` to mutate in place

2. **Pass signals down as props**
   - Child components receive the signal
   - They read from it for rendering
   - Components automatically re-render when signal changes

3. **Update signals directly**
   - Use `.set()` for full replacement
   - Use `.write()` for in-place mutation (push, remove, update item)
   - No need for refresh triggers

---

## Proposed Architecture

### CreatorMode State Structure

```rust
pub struct CreatorModeState {
    // Entity lists - stored as signals
    pub characters: Signal<Vec<CharacterSummary>>,
    pub locations: Signal<Vec<LocationSummary>>,
    pub items: Signal<Vec<ItemSummary>>,  // Future
    pub maps: Signal<Vec<MapSummary>>,    // Future
    
    // Loading states
    pub characters_loading: Signal<bool>,
    pub locations_loading: Signal<bool>,
    
    // Error states
    pub characters_error: Signal<Option<String>>,
    pub locations_error: Signal<Option<String>>,
    
    // Selected entity for editing
    pub selected_entity_id: Signal<Option<String>>,
}
```

### Data Flow

```
CreatorMode (owns signals)
    │
    ├──> EntityBrowser (receives signals as props)
    │       │
    │       ├──> CharacterList (reads from characters signal)
    │       └──> LocationList (reads from locations signal)
    │
    └──> CharacterForm / LocationForm
            │
            └──> on_saved callback
                    │
                    └──> Updates signal directly
                            │
                            └──> List automatically re-renders
```

### Implementation Steps

1. **Move signal ownership to CreatorMode**
   - Create signals for characters and locations lists
   - Create signals for loading/error states
   - Remove refresh_counter entirely

2. **Update EntityBrowser props**
   - Remove `refresh_trigger` prop
   - Add `characters: Signal<Vec<CharacterSummary>>` prop
   - Add `locations: Signal<Vec<LocationSummary>>` prop
   - Add loading/error signals as props

3. **Update CharacterList/LocationList**
   - Remove local `use_signal` for data
   - Remove `use_effect` that watches refresh_trigger
   - Receive data signal as prop
   - Only fetch on mount (initial load)
   - Render directly from prop signal

4. **Update CharacterForm/LocationForm**
   - Remove `on_saved` callback that increments counter
   - Add callback that receives the signal
   - On successful save:
     - If creating: Add new item to signal (optimistic update)
     - If updating: Update item in signal
     - Optionally: Re-fetch to sync with server (background)

5. **Initial Data Loading**
   - CreatorMode fetches data on mount
   - Updates signals with fetched data
   - Lists render from signals (no fetching in child components)

---

## Benefits

1. **True Reactivity**: Components automatically update when signals change
2. **Optimistic Updates**: Can add items immediately, sync with server in background
3. **Single Source of Truth**: Data stored once, shared across components
4. **Better Performance**: No unnecessary re-fetches, only update what changed
5. **Cleaner Code**: No hacky counters, follows established patterns
6. **Easier Testing**: Signals can be mocked/stubbed easily

---

## Migration Strategy

1. Keep existing code working (don't break current functionality)
2. Add new signal-based architecture alongside old code
3. Migrate one component at a time (Characters first, then Locations)
4. Remove old refresh counter code once migration complete
5. Test thoroughly to ensure no regressions

---

## Code Examples

### Before (Current - Hacky)
```rust
// CreatorMode
let mut refresh_counter: Signal<u32> = use_signal(|| 0);

// CharacterList
let mut characters: Signal<Vec<CharacterSummary>> = use_signal(Vec::new);
use_effect(move || {
    let _ = refresh_trigger.read(); // Watch counter
    spawn(async move {
        // Re-fetch entire list
    });
});

// CharacterForm on_saved
on_saved: move |_| {
    refresh_counter.set(*refresh_counter.read() + 1); // Increment hack
}
```

### After (Proper Reactive)
```rust
// CreatorMode
let mut characters: Signal<Vec<CharacterSummary>> = use_signal(Vec::new);
let mut characters_loading = use_signal(|| true);

// Initial fetch on mount
use_effect(move || {
    spawn(async move {
        match svc.list_characters(&world_id).await {
            Ok(fetched) => characters.set(fetched),
            Err(e) => error.set(Some(e.to_string())),
        }
        characters_loading.set(false);
    });
});

// CharacterList (receives signal as prop)
#[component]
fn CharacterList(
    characters: Signal<Vec<CharacterSummary>>,  // Read-only signal
    // ...
) -> Element {
    rsx! {
        for character in characters.read().iter() {
            // Render from signal
        }
    }
}

// CharacterForm on_saved
on_saved: move |new_character| {
    // Optimistic update - add immediately
    characters.write().push(new_character);
    // Optionally: Re-fetch in background to sync
}
```

---

## Additional Optimizations

### 1. Optimistic Updates
When creating a character:
- Immediately add to `characters` signal
- Show in list right away
- Re-fetch in background to get server-assigned ID
- Update the item in signal with real ID

### 2. Partial Updates
When updating a character:
- Find item in signal by ID
- Update it in place using `.write()`
- No need to re-fetch entire list

### 3. Delete Handling
When deleting:
- Remove from signal using `.write().retain()`
- Show confirmation, then remove
- Optionally sync with server

### 4. Search/Filter
- Add search query as signal
- Filter the signal data in render
- No need to re-fetch when search changes

---

## Files to Modify

1. `Player/src/presentation/components/creator/mod.rs`
   - Move signal ownership here
   - Remove refresh_counter
   - Add initial data fetching

2. `Player/src/presentation/components/creator/entity_browser.rs`
   - Update props to receive signals
   - Remove refresh_trigger prop
   - Pass signals to list components

3. `Player/src/presentation/components/creator/character_form.rs`
   - Update on_saved to receive signal
   - Add item to signal on create
   - Update item in signal on edit

4. `Player/src/presentation/components/creator/location_form.rs`
   - Same as character_form

---

## Testing Checklist

- [ ] Create character → appears in list immediately
- [ ] Edit character → changes reflect in list
- [ ] Delete character → removed from list
- [ ] Switch tabs → correct list shows
- [ ] Refresh page → data loads correctly
- [ ] Error handling → errors display properly
- [ ] Loading states → loading indicators show

---

## Future Enhancements

1. **Pagination**: For large lists, add pagination signals
2. **Sorting**: Add sort order signal, sort in render
3. **Caching**: Cache fetched data, only re-fetch when needed
4. **WebSocket Updates**: Listen for server updates, update signals
5. **Undo/Redo**: Track signal history for undo functionality
