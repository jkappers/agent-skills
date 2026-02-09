# Technical Overview: {Feature Name}

## Summary

[2-3 sentences describing the feature and its purpose]

## Architecture

### System Context

[How this feature fits into the broader system. What role does it play?]

### Component Boundaries

[What components are involved and their responsibilities]

| Component | Responsibility |
|-----------|----------------|
| {Component A} | [What it handles] |
| {Component B} | [What it handles] |

### Data Flow

[How data moves through the system]

```
[Simple diagram showing component relationships]

Example:
User -> API Gateway -> Auth Service -> Database
                    -> Feature Service -> Cache
```

## Integration Points

| System | Integration Type | Notes |
|--------|-----------------|-------|
| {Existing System A} | [API/Event/Direct] | [How it connects] |
| {Existing System B} | [API/Event/Direct] | [How it connects] |

## Decisions

| Decision | ADR |
|----------|-----|
| {Choice made} | [ADR-NNN](../../docs/adr/NNNN-title.md) |

## Security Considerations

- **Authentication**: [How users are authenticated]
- **Authorization**: [How access is controlled]
- **Data Protection**: [Sensitive data handling]

## Risks and Constraints

| Risk/Constraint | Impact | Mitigation |
|-----------------|--------|------------|
| {Risk 1} | [What could go wrong] | [How to address it] |
| {Constraint 1} | [How it limits design] | [How to work within it] |

## Open Questions

- [ ] [Unresolved question for stakeholders]
- [ ] [Decision that needs input]
