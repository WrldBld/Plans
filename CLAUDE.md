# Plans - Claude Code Instructions

This directory contains project planning documents, roadmaps, and design specs for WrldBldr.

## Key Files

- `ROADMAP.md` - Main project roadmap with phases and implementation status
- Phase-specific plans in subdirectories

## Conventions

### ROADMAP.md Updates

When completing work on Engine or Player:

1. **Always update ROADMAP.md** with completion status
2. Use format: `- [x] Task name (Date completed: YYYY-MM-DD)`
3. Move completed phases to appropriate status section
4. Add notes about deferred items or known issues

### Phase Planning

- Each major feature should have a phase number (e.g., Phase 17: Story Arc)
- Break phases into sub-phases: A, B, C for backend; D, E, F, G for frontend
- Document dependencies between phases

### Version Control

- Plans are tracked in git alongside code
- Commit plan updates with related code changes
- Use descriptive commit messages for plan changes

## Project Structure Reference

```
WrldBldr/
├── Engine/    # Rust backend (Axum + Neo4j)
├── Player/    # Rust frontend (Dioxus)
└── plans/     # This directory
```

## Related Documentation

- Engine API routes: `Engine/src/infrastructure/http/`
- Player routes: `Player/src/routes.rs`
- Domain entities: `Engine/src/domain/entities/`
