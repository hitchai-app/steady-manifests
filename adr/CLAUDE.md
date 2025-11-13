# Architecture Decision Records (ADR) Guidelines

## Purpose

ADRs document **why** architectural decisions were made, not **how** they are implemented. They capture the context, reasoning, and trade-offs for future reference.

## What Belongs in an ADR

### ✅ DO Include:

- **Context**: The problem, constraints, and forces driving the decision
- **Decision**: The architectural choice made (concise summary)
- **Rationale**: Why this approach was chosen over alternatives
- **Consequences**: Positive, negative, and neutral impacts
- **Alternatives Considered**: Other options and why they were rejected
- **References**: Links to industry standards, documentation, research

### ❌ DO NOT Include:

- **Complete code listings** - Reference file paths instead
- **Full YAML/configuration manifests** - Summarize key configuration points
- **Step-by-step implementation guides** - These belong in task documentation
- **Detailed test procedures** - Reference test strategy, not full commands
- **Low-level implementation details** - Focus on architectural choices

## Structure

```markdown
# ADR-XXX: [Decision Title]

## Status

[Proposed | Accepted | Deprecated | Superseded by ADR-XXX]

## Context

Brief description of the problem, constraints, and forces.

## Decision

The architectural choice made (2-3 sentences).

## Consequences

### Positive

- Benefit 1
- Benefit 2

### Negative

- Trade-off 1
- Risk 1

### Neutral

- Change 1

## Alternatives Considered

### Option 1: [Name]

- Why rejected

### Option 2: [Name]

- Why rejected

## References

- Industry standards
- Documentation links
```

## Examples

### ✅ Good ADR Content:

```markdown
## Decision

Configure CORS at the Traefik ingress layer using Headers Middleware.

## Rationale

- Industry standard (AWS, GCP, Azure)
- Prevents duplicate header issues
- Better performance (edge preflight handling)
- Centralized policy management
```

### ❌ Bad ADR Content:

```markdown
## Implementation

Create this file:

\`\`\`yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
name: cors-headers
namespace: steady-stage
spec:
headers:
accessControlAllowOriginList: - "https://app-stage.steady.ops.last-try.org" # ... 50 more lines ...
\`\`\`

Then run these commands:
\`\`\`bash
kubectl apply -f cors-middleware.yaml
kubectl port-forward ...

# ... 20 more lines ...

\`\`\`
```

## Key Principles

1. **ADRs are about decisions, not instructions**
2. **Focus on the "why", not the "how"**
3. **Keep it concise** - Target 200-400 words for core content
4. **Reference, don't duplicate** - Link to implementation files instead of copying them
5. **Architecture-level** - Decisions that affect multiple components or long-term structure

## When NOT to Write an ADR

- Routine implementation choices (variable naming, minor refactorings)
- Temporary workarounds or quick fixes
- Choices easily reversed without significant impact
- Decisions that don't affect architecture or other developers

## Refinement Process

If an ADR contains too much implementation detail:

1. **Extract** code listings to implementation documentation
2. **Summarize** configuration as key points
3. **Reference** file paths instead of copying content
4. **Focus** on the architectural reasoning

## Related Documentation

- **Concepts**: High-level domain models and principles
- **ADRs**: Architectural decisions and rationale (this directory)
- **Tasks**: Step-by-step implementation guides with code examples

ADRs bridge concepts and tasks - they explain **why we chose this approach** without detailing **how to implement it step-by-step**.
