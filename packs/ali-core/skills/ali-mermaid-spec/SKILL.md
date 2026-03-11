---
name: ali-mermaid-spec
description: |
  Mermaid diagram syntax and rendering patterns. Use when:

  PLANNING: Designing documentation with diagrams, planning flowcharts,
  architecting sequence diagrams, considering diagram types for communication

  IMPLEMENTATION: Writing Mermaid syntax, creating flowcharts, sequence diagrams,
  class diagrams, ERD, Gantt charts, embedding in markdown or web apps

  GUIDANCE: Asking about Mermaid syntax, diagram types, styling options,
  rendering libraries, or best practices for technical diagrams

  REVIEW: Checking Mermaid diagram syntax, validating rendering,
  reviewing diagram clarity and accuracy
---

# Mermaid Diagrams

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing documentation with embedded diagrams
- Choosing diagram types for different use cases
- Planning visual representations of systems or processes
- Architecting diagram-heavy documentation

**Implementation:**
- Writing Mermaid syntax for various diagram types
- Embedding diagrams in markdown files
- Rendering Mermaid in web applications
- Styling and theming diagrams

**Guidance/Best Practices:**
- Mermaid syntax for specific diagram types
- When to use which diagram type
- Rendering options and libraries
- Accessibility and clarity

**Review/Validation:**
- Syntax validation for Mermaid code
- Diagram clarity and completeness
- Rendering compatibility

---

## Key Principles

- **Text-based**: Diagrams defined in plain text, version-controllable
- **Markdown-friendly**: Native support in GitHub, GitLab, many tools
- **Type-specific syntax**: Each diagram type has unique keywords
- **Direction matters**: LR, TB, RL, BT control flow direction
- **Keep it simple**: Overly complex diagrams lose clarity

---

## Patterns We Use

### Flowchart

Basic flowchart with different node shapes:

```mermaid
flowchart TD
    A[Start] --> B{Is it working?}
    B -->|Yes| C[Great!]
    B -->|No| D[Debug]
    D --> B
    C --> E([End])
```

Node shapes:
- `[text]` - Rectangle
- `(text)` - Rounded rectangle
- `([text])` - Stadium/pill shape
- `{text}` - Diamond (decision)
- `[[text]]` - Subroutine
- `[(text)]` - Cylinder (database)
- `((text))` - Circle
- `>text]` - Flag/asymmetric

Direction options:
- `TD` / `TB` - Top to bottom
- `LR` - Left to right
- `RL` - Right to left
- `BT` - Bottom to top

### Sequence Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant A as App
    participant D as Database

    U->>A: Request data
    activate A
    A->>D: Query
    activate D
    D-->>A: Results
    deactivate D
    A-->>U: Response
    deactivate A

    Note over U,A: User sees data

    alt Success
        U->>A: Process
    else Failure
        U->>A: Retry
    end
```

Arrow types:
- `->` Solid line without arrow
- `-->` Dotted line without arrow
- `->>` Solid line with arrow
- `-->>` Dotted line with arrow
- `-x` Solid line with X (async)
- `--x` Dotted line with X

### Class Diagram

```mermaid
classDiagram
    class User {
        +String name
        +String email
        -String password
        +login() bool
        +logout() void
    }

    class Order {
        +int id
        +Date created
        +List~Item~ items
        +calculateTotal() float
    }

    User "1" --> "*" Order : places
    Order *-- Item : contains

    class Item {
        +String name
        +float price
    }
```

Relationships:
- `<|--` Inheritance
- `*--` Composition
- `o--` Aggregation
- `-->` Association
- `--` Link (solid)
- `..>` Dependency
- `..|>` Realization
- `..` Link (dashed)

### Entity Relationship Diagram

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--|{ LINE_ITEM : contains
    PRODUCT ||--o{ LINE_ITEM : "ordered in"

    CUSTOMER {
        int id PK
        string name
        string email UK
    }

    ORDER {
        int id PK
        int customer_id FK
        date created
    }

    LINE_ITEM {
        int order_id FK
        int product_id FK
        int quantity
    }

    PRODUCT {
        int id PK
        string name
        float price
    }
```

Relationship notation:
- `||` Exactly one
- `o|` Zero or one
- `}|` One or more
- `o{` Zero or more

### State Diagram

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Submitted : submit()
    Submitted --> Approved : approve()
    Submitted --> Rejected : reject()
    Rejected --> Draft : revise()
    Approved --> [*]

    state Submitted {
        [*] --> Pending
        Pending --> UnderReview
        UnderReview --> [*]
    }
```

### Gantt Chart

```mermaid
gantt
    title Project Timeline
    dateFormat YYYY-MM-DD

    section Planning
    Requirements     :a1, 2024-01-01, 7d
    Design          :a2, after a1, 14d

    section Development
    Backend         :b1, after a2, 21d
    Frontend        :b2, after a2, 21d

    section Testing
    Integration     :c1, after b1, 7d
    UAT            :c2, after c1, 7d

    section Deployment
    Release        :milestone, after c2, 0d
```

### Pie Chart

```mermaid
pie title Technology Usage
    "Python" : 40
    "JavaScript" : 30
    "Java" : 20
    "Other" : 10
```

### Git Graph

```mermaid
gitGraph
    commit
    commit
    branch develop
    checkout develop
    commit
    commit
    checkout main
    merge develop
    commit
    branch feature
    checkout feature
    commit
    checkout main
    merge feature
```

---

## Styling

### Theme Configuration

```mermaid
%%{init: {'theme': 'forest'}}%%
flowchart LR
    A --> B
```

Available themes: `default`, `forest`, `dark`, `neutral`, `base`

### Custom Styles

```mermaid
flowchart LR
    A:::highlight --> B
    B --> C:::warning

    classDef highlight fill:#f9f,stroke:#333,stroke-width:2px
    classDef warning fill:#ff0,stroke:#f00
```

### Inline Styling

```mermaid
flowchart LR
    A --> B
    style A fill:#bbf,stroke:#333
    style B fill:#f9f,stroke:#f00,stroke-width:2px
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Too many nodes | Diagram becomes unreadable | Split into multiple diagrams |
| Crossing lines everywhere | Hard to follow flow | Reorganize or use subgraphs |
| Inconsistent direction | Confusing flow | Stick to one direction |
| No labels on edges | Unclear relationships | Add descriptive labels |
| Tiny text in large diagrams | Unreadable when scaled | Keep diagrams focused |
| Wrong diagram type | Misrepresents relationships | Match diagram to data structure |

---

## Quick Reference

### Embedding in Markdown

````markdown
```mermaid
flowchart LR
    A --> B
```
````

### Embedding in HTML

```html
<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>mermaid.initialize({startOnLoad: true});</script>

<pre class="mermaid">
flowchart LR
    A --> B
</pre>
```

### JavaScript Rendering

```javascript
import mermaid from 'mermaid';

mermaid.initialize({
  startOnLoad: true,
  theme: 'default',
  securityLevel: 'loose'
});

// Render specific element
const element = document.querySelector('.mermaid');
const graphDefinition = 'flowchart LR\n  A --> B';
const { svg } = await mermaid.render('graph-id', graphDefinition);
element.innerHTML = svg;
```

### Diagram Type Summary

| Type | Use Case | Keyword |
|------|----------|---------|
| Flowchart | Process flows, decision trees | `flowchart` |
| Sequence | API calls, interactions | `sequenceDiagram` |
| Class | Object relationships | `classDiagram` |
| ERD | Database schema | `erDiagram` |
| State | State machines | `stateDiagram-v2` |
| Gantt | Project timelines | `gantt` |
| Pie | Proportions | `pie` |
| Git | Branch history | `gitGraph` |
| Mindmap | Hierarchical ideas | `mindmap` |
| Timeline | Chronological events | `timeline` |

### Special Characters

```mermaid
flowchart LR
    A["Text with (parentheses)"]
    B["Text with [brackets]"]
    C["Text with {braces}"]
    D["Unicode: &#9733; star"]
```

---

## References

- [Mermaid Documentation](https://mermaid.js.org/intro/)
- [Mermaid Live Editor](https://mermaid.live/)
- [GitHub Mermaid Support](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/creating-diagrams)
- [Mermaid CLI](https://github.com/mermaid-js/mermaid-cli)
