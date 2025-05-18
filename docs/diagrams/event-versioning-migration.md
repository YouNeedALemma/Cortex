# Event Versioning and Migration

This document illustrates the event versioning and migration system in the MonoSphere application, which allows for schema evolution over time while maintaining backward compatibility.

## Event Migration Registry Architecture

```mermaid
classDiagram
    class EventMigrationRegistry {
        -migrators Map~String, Map~EventVersion, EventMigrator~~
        -latestVersions Map~String, EventVersion~
        +shared EventMigrationRegistry
        +register(migrator EventMigrator)
        +migrateEvent(event Event) Event
        +getLatestVersion(eventType String) EventVersion
        +getMigrationPath(eventType String, sourceVersion EventVersion) [EventMigrator]
    }
    
    class EventMigrator {
        <<interface>>
        +eventType String
        +sourceVersion EventVersion
        +targetVersion EventVersion
        +migrate(payload Map~String, AnyCodable~) Map~String, AnyCodable~
    }
    
    class EventVersion {
        +major Int
        +minor Int
        +patch Int
        +compare(other EventVersion) Int
    }
    
    class TaskEventMigrator {
        +eventType String
        +sourceVersion EventVersion
        +targetVersion EventVersion
        +migrate(payload Map~String, AnyCodable~) Map~String, AnyCodable~
    }
    
    class ProjectEventMigrator {
        +eventType String
        +sourceVersion EventVersion
        +targetVersion EventVersion
        +migrate(payload Map~String, AnyCodable~) Map~String, AnyCodable~
    }
    
    EventMigrationRegistry -- EventMigrator : registers
    EventMigrationRegistry -- EventVersion : tracks
    EventMigrator <|.. TaskEventMigrator : implements
    EventMigrator <|.. ProjectEventMigrator : implements
    EventMigrator -- EventVersion : has source/target
```

## Event Versioning Structure

```mermaid
graph TD
    A[Event Version] --> B[Major]
    A --> C[Minor]
    A --> D[Patch]
    
    subgraph VersionTypes["Version Change Types"]
        E["Major: Breaking changes
        (incompatible schema)"]
        F["Minor: Non-breaking additions
        (backward compatible)"]
        G["Patch: Bug fixes, clarifications
        (no schema changes)"]
    end
    
    B -.-> E
    C -.-> F
    D -.-> G
    
    style A fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b
    style B fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c
    style C fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c
    style D fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c
    style VersionTypes fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style E fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style F fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style G fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

## Migration Path Discovery

```mermaid
flowchart TD
    A[Event Retrieval] --> B[Check Event Version]
    B --> C{Needs Migration?}
    C -->|No| D[Use Event As-Is]
    C -->|Yes| E[Get Migration Path]
    E --> F[Apply Migration Steps]
    F --> G[Update Event Version]
    G --> H[Return Migrated Event]
    
    classDef retrievalNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef decisionNode fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#f57f17;
    classDef migrationNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef outputNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    
    class A,B retrievalNode;
    class C decisionNode;
    class E,F,G migrationNode;
    class D,H outputNode;
```

## Migration Chain Example

```mermaid
graph LR
    A["Event v1.0.0"] --> B["Migrator
    v1.0.0 → v1.1.0"]
    B --> C["Event v1.1.0"]
    C --> D["Migrator
    v1.1.0 → v2.0.0"]
    D --> E["Event v2.0.0"]
    E --> F["Migrator
    v2.0.0 → v2.1.0"]
    F --> G["Event v2.1.0"]
    
    classDef eventNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef migratorNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    
    class A,C,E,G eventNode;
    class B,D,F migratorNode;
```

## Migration Sequence

```mermaid
sequenceDiagram
    participant ES as EventStore
    participant Registry as EventMigrationRegistry
    participant Path as Migration Path
    participant Migrator as EventMigrator
    participant Event
    
    ES->>Registry: migrateEvent(event)
    activate Registry
    
    Registry->>Event: get version
    Event-->>Registry: version
    
    Registry->>Registry: getLatestVersion(eventType)
    
    alt Version is Current
        Registry-->>ES: event (unchanged)
    else Version Needs Migration
        Registry->>Registry: getMigrationPath(eventType, version)
        Registry-->>Path: migrationPath
        
        loop For Each Migrator in Path
            Registry->>Migrator: migrate(payload)
            activate Migrator
            Migrator-->>Registry: migratedPayload
            deactivate Migrator
        end
        
        Registry->>Event: create new event with updated payload and version
        Event-->>Registry: migratedEvent
        Registry-->>ES: migratedEvent
    end
    
    deactivate Registry
```

## Example Migration Implementation

```mermaid
classDiagram
    class TaskCreatedEventMigratorV1ToV2 {
        +eventType String = "task.created"
        +sourceVersion EventVersion = "1.0.0"
        +targetVersion EventVersion = "2.0.0"
        +migrate(payload Map~String, AnyCodable~) Map~String, AnyCodable~
    }
    
    class TaskEventMigrationExample {
        <<example>>
        V1.0.0 Payload:
        {
          "id": "task-123",
          "title": "Example Task",
          "description": "Task description",
          "status": "pending"  // Renamed to "isCompleted" in v2
        }
        
        V2.0.0 Payload:
        {
          "id": "task-123",
          "title": "Example Task", 
          "description": "Task description",
          "isCompleted": false,  // Changed from "status"
          "priority": "medium"   // New field
        }
    }
    
    TaskCreatedEventMigratorV1ToV2 -- TaskEventMigrationExample : transforms
```

## Migration Strategy Considerations

```mermaid
mindmap
    root((Event Migration<br>Strategies))
        Continuous Migration
            ::icon(fa fa-sync-alt)
            Migrate on read
            Always latest schema
            Performance cost
        Snapshot Migration
            ::icon(fa fa-camera)
            Periodic migrations
            Reduced runtime costs
            Background process
        Parallel Schemas
            ::icon(fa fa-code-branch)
            Support multiple versions
            Client compatibility
            Complex to maintain
        Versioned Projections
            ::icon(fa fa-project-diagram)
            Version-specific views
            Minimizes migrations
            Higher storage cost
```

## Benefits of Event Versioning in MonoSphere

1. **Schema Evolution** - Allows event schemas to evolve over time
2. **Backward Compatibility** - Older events can still be processed with newer code
3. **Forward Compatibility** - Newer events can be partially understood by older code
4. **Migration Path** - Clear path for migrating between versions
5. **Semantic Versioning** - Clear indication of compatibility changes
6. **Automatic Migration** - Events are automatically migrated when retrieved