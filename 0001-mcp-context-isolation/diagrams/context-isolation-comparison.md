# Context Isolation vs Traditional Lazy Loading

## The Key Difference

Traditional lazy loading still pollutes the main context. Our proposal keeps the main context **permanently clean**.

## Traditional Lazy Loading

```mermaid
sequenceDiagram
    participant User
    participant Main as Main Context
    participant MCP as MCP Server

    User->>Main: Start session
    Note over Main: Context: 0 tokens (clean)

    User->>Main: "Query the database"
    Main->>MCP: Load postgres MCP
    MCP-->>Main: Tool definitions (~12,000 tokens)
    Note over Main: Context: 12,000 tokens (occupied)

    User->>Main: "Check GitHub PR"
    Main->>MCP: Load github MCP
    MCP-->>Main: Tool definitions (~18,000 tokens)
    Note over Main: Context: 30,000 tokens (growing)

    Note over Main: MCPs stay loaded<br/>Context never recovers
```

## Our Proposal: Context Isolation

```mermaid
sequenceDiagram
    participant User
    participant Main as Main Context
    participant Fork as Forked Context
    participant MCP as MCP Server

    User->>Main: Start session
    Note over Main: Context: ~1,300 tokens<br/>(base MCPs only)

    User->>Main: "Query the database"
    Main->>Fork: Spawn database-specialist agent
    Fork->>MCP: Load postgres MCP
    MCP-->>Fork: Tool definitions (isolated)
    Fork-->>Main: Return result only
    Note over Fork: Context released
    Note over Main: Still ~1,300 tokens (clean)

    User->>Main: "Check GitHub PR"
    Main->>Fork: Spawn code-reviewer agent
    Fork->>MCP: Load github MCP
    MCP-->>Fork: Tool definitions (isolated)
    Fork-->>Main: Return result only
    Note over Fork: Context released
    Note over Main: Still ~1,300 tokens (clean)
```

## Side-by-Side Comparison

```mermaid
graph TB
    subgraph Traditional["Traditional Lazy Loading"]
        T1[Session Start] --> T2[Need Database]
        T2 --> T3[Load postgres into Main]
        T3 --> T4[Need GitHub]
        T4 --> T5[Load github into Main]
        T5 --> T6["Main Context: 30,000+ tokens"]
        style T6 fill:#ff6b6b,color:#fff
    end

    subgraph Isolation["Context Isolation (Our Proposal)"]
        I1[Session Start] --> I2[Need Database]
        I2 --> I3[Fork → Load postgres]
        I3 --> I4[Return result, release fork]
        I4 --> I5[Need GitHub]
        I5 --> I6[Fork → Load github]
        I6 --> I7[Return result, release fork]
        I7 --> I8["Main Context: ~1,300 tokens"]
        style I8 fill:#51cf66,color:#fff
    end
```

## Token Impact Over Time

```mermaid
xychart-beta
    title "Context Usage Over Session"
    x-axis ["Start", "1st MCP", "2nd MCP", "3rd MCP", "4th MCP", "5th MCP"]
    y-axis "Tokens" 0 --> 80000
    line "Traditional Lazy Loading" [0, 15000, 30000, 45000, 55000, 67000]
    line "Context Isolation" [1300, 1300, 1300, 1300, 1300, 1300]
```
