# Phase 2: Core Domain (Engine)

## Objective
Implement full CRUD operations for all domain entities using Neo4j, making the Engine capable of persisting and retrieving world data.

## Status: ✅ COMPLETE

## Tasks

### 2.1 Neo4j Repository Implementation
**Status:** ✅ COMPLETE
**Files:**
- `Engine/src/infrastructure/persistence/mod.rs`
- `Engine/src/infrastructure/persistence/connection.rs`
- `Engine/src/infrastructure/persistence/world_repository.rs`
- `Engine/src/infrastructure/persistence/character_repository.rs`
- `Engine/src/infrastructure/persistence/location_repository.rs`
- `Engine/src/infrastructure/persistence/scene_repository.rs`
- `Engine/src/infrastructure/persistence/relationship_repository.rs`

**Subtasks:**
- [x] 2.1.1 Refactor persistence.rs into module with separate files
- [x] 2.1.2 Implement WorldRepository with Act support
- [x] 2.1.3 Implement Character CRUD with archetype storage
- [x] 2.1.4 Implement Location CRUD with connections (graph edges)
- [x] 2.1.5 Implement Scene CRUD linked to Acts and Locations
- [x] 2.1.6 Implement Relationship storage (character↔character edges)
- [ ] 2.1.7 Implement GridMap storage (deferred to Phase 7)

### 2.2 REST API Endpoints
**Status:** ✅ COMPLETE
**Files:**
- `Engine/src/infrastructure/http/mod.rs`
- `Engine/src/infrastructure/http/world_routes.rs`
- `Engine/src/infrastructure/http/character_routes.rs`
- `Engine/src/infrastructure/http/location_routes.rs`
- `Engine/src/infrastructure/http/scene_routes.rs`

**Subtasks:**
- [x] 2.2.1 Add Axum routes for World CRUD (including Acts)
- [x] 2.2.2 Add Axum routes for Character CRUD (including archetype changes)
- [x] 2.2.3 Add Axum routes for Location CRUD (including connections)
- [x] 2.2.4 Add Axum routes for Scene CRUD (including directorial notes)
- [x] 2.2.5 Add route to fetch social network graph
- [x] 2.2.6 Add routes for Relationship management

### 2.3 Schema Initialization
**Status:** ✅ COMPLETE (basic)
**Files:**
- `Engine/src/infrastructure/persistence/connection.rs`

**Subtasks:**
- [x] 2.3.1 Create Neo4j constraints on startup (in Neo4jConnection::initialize_schema)
- [x] 2.3.2 Create Neo4j indexes for common queries
- [ ] 2.3.3 Add migration/versioning support (deferred to Phase 8)

### 2.4 Testing
**Status:** ⏳ PENDING
**Files:**
- `Engine/tests/repository_tests.rs`

**Subtasks:**
- [ ] 2.4.1 Integration tests for World repository
- [ ] 2.4.2 Integration tests for Character repository
- [ ] 2.4.3 Integration tests for Relationship queries

## API Routes Summary

> **Note:** Axum 0.8 uses `{param}` syntax for route parameters.

### World Routes
- `GET /api/worlds` - List all worlds
- `POST /api/worlds` - Create a world
- `GET /api/worlds/{id}` - Get world by ID
- `PUT /api/worlds/{id}` - Update world
- `DELETE /api/worlds/{id}` - Delete world
- `GET /api/worlds/{id}/acts` - List acts in world
- `POST /api/worlds/{id}/acts` - Create act in world

### Character Routes
- `GET /api/worlds/{world_id}/characters` - List characters in world
- `POST /api/worlds/{world_id}/characters` - Create character
- `GET /api/characters/{id}` - Get character by ID
- `PUT /api/characters/{id}` - Update character
- `DELETE /api/characters/{id}` - Delete character
- `PUT /api/characters/{id}/archetype` - Change character archetype

### Location Routes
- `GET /api/worlds/{world_id}/locations` - List locations in world
- `POST /api/worlds/{world_id}/locations` - Create location
- `GET /api/locations/{id}` - Get location by ID
- `PUT /api/locations/{id}` - Update location
- `DELETE /api/locations/{id}` - Delete location
- `GET /api/locations/{id}/connections` - Get location connections
- `POST /api/locations/connections` - Create location connection

### Scene Routes
- `GET /api/acts/{act_id}/scenes` - List scenes in act
- `POST /api/acts/{act_id}/scenes` - Create scene
- `GET /api/scenes/{id}` - Get scene by ID
- `PUT /api/scenes/{id}` - Update scene
- `DELETE /api/scenes/{id}` - Delete scene
- `PUT /api/scenes/{id}/notes` - Update directorial notes

### Relationship Routes
- `GET /api/worlds/{world_id}/social-network` - Get social network graph
- `POST /api/relationships` - Create relationship
- `DELETE /api/relationships/{id}` - Delete relationship

## Acceptance Criteria
- [x] Can create, read, update, delete Worlds via API
- [x] Can create Characters with Campbell archetypes
- [x] Can create Relationships between Characters
- [x] Can query the social network graph
- [x] Can create Locations with connections
- [x] Can create Scenes linked to Locations and Acts
- [x] All data persists across Engine restarts
- [ ] Integration tests pass (deferred)

## Dependencies
- Phase 1 (Foundation) ✅

## Notes
- Using neo4rs crate for async Neo4j operations
- Complex fields (stats, wants, etc.) stored as JSON strings
- Archetype history stored as JSON array
- Schema initialization runs on startup in connection.rs
- GridMap storage deferred to Phase 7 (Tactical Combat)
- Axum 0.8 route syntax uses `{param}` instead of `:param`

## Domain Entities Reference
See `plans/00-master-plan.md` PART 1 for complete domain model specifications:
- World, Act, Scene, Location, Character, Relationship, GridMap
- Campbell archetypes, actantial model (Wants), SpatialRelationship enum
- Neo4j graph schema with node types and relationship edges
