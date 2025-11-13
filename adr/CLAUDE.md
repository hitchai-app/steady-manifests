# Architecture Decision Records (ADR) Guidelines

## Purpose

ADRs are **decision statements**, not tasks or instructions.

They document:

- **What decision was made** (the architectural choice)
- **Why it was made** (context, forces, rationale)
- **What the trade-offs are** (consequences)

They do NOT document:

- How to implement the decision
- Step-by-step instructions
- Code examples or configurations

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

1. **ADRs are decision statements** - "We decided X" not "Do X"
2. **Focus on reasoning** - Why this choice over alternatives
3. **Keep it concise** - Target 200-400 words for core content
4. **Reference, don't duplicate** - Link to implementation, don't describe it
5. **Architecture-level** - Decisions that affect multiple components or long-term structure
6. **Immutable record** - Capture the decision at the time it was made

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

## Critical Distinction

**ADRs are NOT:**

- ❌ Instructions or task lists
- ❌ Implementation guides
- ❌ Tutorials or how-tos
- ❌ Step-by-step procedures

**ADRs ARE:**

- ✅ Decision statements ("We decided to use X")
- ✅ Rationale explanations ("Because of Y and Z")
- ✅ Trade-off documentation ("This gives us A but costs us B")
- ✅ Historical records ("At the time, we chose X over Y")

## Related Documentation

- **Concepts**: High-level domain models and principles
- **ADRs**: Architectural decisions and rationale (this directory)
- **Tasks**: Step-by-step implementation guides with code examples

ADRs answer "What did we decide and why?" not "How do we implement it?"
