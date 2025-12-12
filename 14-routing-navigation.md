# Phase 14: Routing & Navigation System

## Overview

The Player application currently uses signal-based view switching without proper routing. This causes navigation issues:
- **Web**: Refresh loses location, no browser history integration, no shareable URLs
- **Mobile**: No deep linking support
- **Desktop**: No custom URL scheme handling

This phase introduces proper routing using Dioxus Router to provide URL-based navigation with platform-appropriate behaviors.

---

## Current State

### Signal-Based Navigation (main.rs)

```rust
// Current implementation
let mut current_view = use_signal(|| AppView::MainMenu);

// View switching via signals
match current_view_val {
    AppView::MainMenu => rsx! { MainMenu { ... } },
    AppView::RoleSelect => rsx! { RoleSelect { ... } },
    AppView::WorldSelect => rsx! { WorldSelectView { ... } },
    AppView::PlayerView => rsx! { PCView { ... } },
    AppView::DungeonMasterView => rsx! { DMView { ... } },
    AppView::SpectatorView => rsx! { SpectatorView { ... } },
}
```

### Problems

1. **No URL State**: Navigation state exists only in memory
2. **No History**: Browser back/forward buttons don't work
3. **No Deep Links**: Cannot navigate directly to a world or view
4. **No Bookmarks**: Users can't save/share specific locations
5. **Lost on Refresh**: All navigation resets to MainMenu

---

## Proposed Route Structure

```
/                           → MainMenu (server URL entry)
/roles                      → RoleSelect (DM/Player/Spectator)
/worlds                     → WorldSelect (requires role in state)
/worlds/:world_id/dm        → DMView (Director mode default)
/worlds/:world_id/dm/creator → DMView (Creator mode)
/worlds/:world_id/dm/settings → DMView (Settings mode)
/worlds/:world_id/play      → PCView
/worlds/:world_id/watch     → SpectatorView
```

### URL Parameters

- `:world_id` - UUID of the selected world
- Query params for state preservation:
  - `?server=<url>` - Engine server URL
  - `?mode=creator|director|settings` - DM mode (optional)

### Example URLs

```
http://localhost:8080/                           # Main menu
http://localhost:8080/roles?server=localhost:3000  # Role selection
http://localhost:8080/worlds                     # World selection
http://localhost:8080/worlds/abc-123/dm          # DM view (Director)
http://localhost:8080/worlds/abc-123/dm/creator  # DM view (Creator)
http://localhost:8080/worlds/abc-123/play        # Player view
http://localhost:8080/worlds/abc-123/watch       # Spectator view
```

---

## User Stories

### US-14.1: URL-Based Navigation (Web)
**As a** web user
**I want** the URL to reflect my current location
**So that** I can bookmark, share, and refresh without losing my place

**Acceptance Criteria:**
- URL changes as user navigates through app
- Refresh preserves current view and world
- Browser back/forward buttons work correctly
- Deep links navigate directly to the correct view

---

### US-14.2: Browser History Integration
**As a** web user
**I want** the browser history to track my navigation
**So that** I can use back/forward buttons naturally

**Acceptance Criteria:**
- Each navigation creates a history entry
- Back button returns to previous view
- Forward button after back works correctly
- History shows meaningful page titles

---

### US-14.3: Deep Link to World (Web)
**As a** user
**I want to** share a link that takes someone directly to a world
**So that** my friends can join my game session easily

**Acceptance Criteria:**
- URLs like `/worlds/abc-123/play` work when shared
- If not connected, prompt for server URL
- If world doesn't exist, show helpful error
- Role is inferred from URL path

---

### US-14.4: Mobile Deep Linking
**As a** mobile user
**I want** to open the app via a deep link
**So that** I can join a game from a notification or shared link

**Acceptance Criteria:**
- Support `wrldbldr://` custom URL scheme
- Handle `wrldbldr://worlds/abc-123/play` links
- App opens directly to the correct view
- Show connection prompt if not connected

**URL Scheme:**
```
wrldbldr://                           → MainMenu
wrldbldr://roles                      → RoleSelect
wrldbldr://worlds/abc-123/play        → PCView for world
wrldbldr://worlds/abc-123/dm          → DMView for world
```

---

### US-14.5: Desktop URL Scheme
**As a** desktop user
**I want** the app to handle `wrldbldr://` URLs
**So that** I can click links that open the app directly

**Acceptance Criteria:**
- Register `wrldbldr://` protocol handler on install
- Handle protocol URLs when app is running
- Launch app if not running when URL clicked
- Navigate to correct view based on URL

---

### US-14.6: Preserve Connection State
**As a** user returning to the app
**I want** my connection to be restored
**So that** I don't have to reconfigure the server every time

**Acceptance Criteria:**
- Server URL persisted in localStorage (web) or app settings (desktop)
- WebSocket reconnects automatically on route change
- Connection status shown during reconnection
- Manual disconnect clears saved state

---

### US-14.7: Route Guards
**As a** developer
**I want** routes to be protected based on state
**So that** users can't access invalid states

**Acceptance Criteria:**
- Cannot access `/worlds` without selecting a role
- Cannot access game views without selecting a world
- Redirect to appropriate step if state is missing
- Show helpful message about what's needed

---

### US-14.8: 404 Handling
**As a** user
**I want** to see a helpful error for invalid URLs
**So that** I understand what went wrong

**Acceptance Criteria:**
- Invalid routes show "Page Not Found" view
- Suggest valid destinations
- Provide link back to main menu
- Log invalid routes for debugging

---

## Technical Implementation

### Router Setup (Dioxus 0.7)

```rust
// Cargo.toml additions
dioxus = { version = "0.7.2", features = ["router"] }

// For web
[target.'cfg(target_arch = "wasm32")'.dependencies]
dioxus = { version = "0.7.2", features = ["web", "router"] }

// For desktop
[target.'cfg(not(target_arch = "wasm32"))'.dependencies]
dioxus = { version = "0.7.2", features = ["desktop", "router"] }
```

### Route Enum Definition

```rust
use dioxus::prelude::*;
use dioxus_router::prelude::*;

#[derive(Clone, Routable, Debug, PartialEq)]
#[rustfmt::skip]
enum Route {
    // Main entry point
    #[route("/")]
    MainMenu {},

    // Role selection
    #[route("/roles")]
    RoleSelect {},

    // World selection (requires role in state)
    #[route("/worlds")]
    WorldSelect {},

    // Game views with world ID
    #[route("/worlds/:world_id/dm")]
    DMView { world_id: String },

    #[route("/worlds/:world_id/dm/:mode")]
    DMViewMode { world_id: String, mode: String },

    #[route("/worlds/:world_id/play")]
    PlayerView { world_id: String },

    #[route("/worlds/:world_id/watch")]
    SpectatorView { world_id: String },

    // 404 fallback
    #[route("/:..route")]
    NotFound { route: Vec<String> },
}
```

### Router Component

```rust
fn App() -> Element {
    // Global state providers
    use_context_provider(GameState::new);
    use_context_provider(SessionState::new);
    use_context_provider(DialogueState::new);
    use_context_provider(GenerationState::new);

    rsx! {
        Router::<Route> {}
    }
}

// Each route component receives params
#[component]
fn DMView(world_id: String) -> Element {
    // Load world if not already loaded
    let game_state = use_context::<GameState>();

    // Check if world is loaded, if not fetch it
    use_effect(move || {
        if game_state.world_id() != Some(&world_id) {
            spawn(async move {
                // Fetch and load world
            });
        }
    });

    rsx! {
        // DM view content
    }
}
```

### History Mode Configuration

```rust
// Web: Use browser history
#[cfg(target_arch = "wasm32")]
fn main() {
    dioxus::launch(App);
    // Dioxus web automatically uses browser history
}

// Desktop: Use memory history with persistence
#[cfg(not(target_arch = "wasm32"))]
fn main() {
    dioxus::launch(App);
    // Desktop uses memory-based routing
}
```

### State Persistence (Web)

```rust
// Use localStorage to persist server URL and role
#[cfg(target_arch = "wasm32")]
fn persist_navigation_state(server_url: &str, role: UserRole) {
    use web_sys::window;
    if let Some(storage) = window()
        .and_then(|w| w.local_storage().ok())
        .flatten()
    {
        let _ = storage.set_item("wrldbldr_server", server_url);
        let _ = storage.set_item("wrldbldr_role", &format!("{:?}", role));
    }
}

#[cfg(target_arch = "wasm32")]
fn restore_navigation_state() -> Option<(String, UserRole)> {
    use web_sys::window;
    let storage = window()?.local_storage().ok()??;
    let server = storage.get_item("wrldbldr_server").ok()??;
    let role_str = storage.get_item("wrldbldr_role").ok()??;
    let role = match role_str.as_str() {
        "DungeonMaster" => UserRole::DungeonMaster,
        "Player" => UserRole::Player,
        "Spectator" => UserRole::Spectator,
        _ => return None,
    };
    Some((server, role))
}
```

### Desktop URL Scheme Registration

```rust
// For desktop, register URL scheme on first run
#[cfg(not(target_arch = "wasm32"))]
fn register_url_scheme() {
    // Platform-specific URL scheme registration
    #[cfg(target_os = "macos")]
    {
        // macOS: Add to Info.plist during build
        // CFBundleURLTypes with wrldbldr scheme
    }

    #[cfg(target_os = "windows")]
    {
        // Windows: Registry entry for wrldbldr://
        // HKEY_CLASSES_ROOT\wrldbldr
    }

    #[cfg(target_os = "linux")]
    {
        // Linux: .desktop file with MimeType
        // x-scheme-handler/wrldbldr
    }
}
```

---

## Implementation Phases

### Phase 14A: Basic Router Setup
- [ ] Add `router` feature to Dioxus dependencies
- [ ] Define `Route` enum with all routes
- [ ] Update `main.rs` to use `Router::<Route>`
- [ ] Convert each view to accept route parameters
- [ ] Remove signal-based view switching

### Phase 14B: Route Guards & Navigation
- [ ] Implement route guards for protected routes
- [ ] Add redirect logic for missing state
- [ ] Create `NotFound` component
- [ ] Update header back button to use router navigation
- [ ] Ensure all navigation uses `Link` or `navigator()`

### Phase 14C: Web History Integration
- [ ] Verify browser history works correctly
- [ ] Add `<title>` updates for each route
- [ ] Implement state persistence to localStorage
- [ ] Add state restoration on page load
- [ ] Test deep links and refresh behavior

### Phase 14D: Desktop URL Scheme
- [ ] Create platform-specific URL scheme handlers
- [ ] Add registration logic for each platform
- [ ] Handle incoming URLs when app is running
- [ ] Test URL scheme on macOS, Windows, Linux

### Phase 14E: Mobile Deep Linking (Future)
- [ ] Configure Android intent filters
- [ ] Configure iOS URL types
- [ ] Handle app launch from deep links
- [ ] Test on Android and iOS devices

---

## Migration Strategy

### Step 1: Parallel Implementation
Keep existing signal-based navigation while building router alongside:

```rust
// Temporary: Both systems active
fn App() -> Element {
    // Old signal-based approach (keep for now)
    let current_view = use_signal(|| AppView::MainMenu);

    // New router (add alongside)
    // Router::<Route> {}
}
```

### Step 2: Gradual Route Migration
Migrate one route at a time:
1. MainMenu → `/`
2. RoleSelect → `/roles`
3. WorldSelect → `/worlds`
4. Game views → `/worlds/:id/*`

### Step 3: State Management Updates
Update state management to work with routes:
- Remove `current_view` signal
- Use route parameters for world_id
- Keep context providers for reactive state

### Step 4: Navigation Updates
Replace all navigation calls:
```rust
// Old
current_view.set(AppView::PlayerView);

// New
navigator().push(Route::PlayerView { world_id: id.clone() });
```

---

## Testing Checklist

### Web
- [ ] URL updates on navigation
- [ ] Refresh preserves location
- [ ] Back/forward buttons work
- [ ] Deep links load correct view
- [ ] Invalid URLs show 404
- [ ] Protected routes redirect properly

### Desktop
- [ ] Navigation works (memory-based)
- [ ] URL scheme registered
- [ ] External links open app
- [ ] App handles URLs when already running

### Mobile (Future)
- [ ] Deep links open app
- [ ] Intent filters configured
- [ ] Universal links work

---

## Dependencies

- **Requires**: Dioxus 0.7+ with router feature
- **Requires**: Phase 13 (World Selection) complete
- **Optional**: Platform-specific build configuration for URL schemes

---

## Out of Scope (Future)

- Server-side rendering (SSR)
- Route-based code splitting
- Analytics integration
- A/B testing by route
- Route-based permissions system

---

## Questions for Clarification

1. **State on Deep Link**: If user opens a deep link to a world they're not connected to, should we:
   - Prompt for server URL?
   - Use last saved server?
   - Show "connect first" message?

2. **Desktop Persistence**: Should desktop app persist navigation state between sessions?

3. **URL Scheme Name**: Is `wrldbldr://` the desired scheme, or should it be `wrldbuilder://` or something else?

4. **Mobile Priority**: Is mobile deep linking needed now, or can it wait for the mobile-specific phase?
