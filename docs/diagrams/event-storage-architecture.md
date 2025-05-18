# Event Storage Architecture

This diagram illustrates the event storage architecture of the MonoSphere application, showing how events are stored and retrieved, and the relationship between different storage implementations.

## Event Store Protocol Hierarchy

```mermaid
classDiagram
    class EventStore {
        <<interface>>
        +append(Event) throws
        +getEvents(startId, limit) throws~[Event]~
        +getEventsByType(type, startId, limit) throws~[Event]~
        +getEventsByCorrelationId(correlationId) throws~[Event]~
        +migrationRegistry EventMigrationRegistry
    }
    
    class InMemoryEventStore {
        -events [Event]
        -migrationRegistry EventMigrationRegistry
        +InMemoryEventStore(migrationRegistry)
        +append(Event) throws
        +getEvents(startId, limit) throws~[Event]~
        +getEventsByType(type, startId, limit) throws~[Event]~
        +getEventsByCorrelationId(correlationId) throws~[Event]~
    }
    
    class CoreDataEventStore {
        -dataModel EventDataModel
        -migrationRegistry EventMigrationRegistry
        +CoreDataEventStore(dataModel, migrationRegistry)
        +append(Event) throws
        +getEvents(startId, limit) throws~[Event]~
        +getEventsByType(type, startId, limit) throws~[Event]~
        +getEventsByCorrelationId(correlationId) throws~[Event]~
    }
    
    EventStore <|.. InMemoryEventStore : implements
    EventStore <|.. CoreDataEventStore : implements
```

## Core Data Storage Components

```mermaid
classDiagram
    class CoreDataEventStore {
        -dataModel EventDataModel
        -migrationRegistry EventMigrationRegistry
        +append(Event) throws
        +getEvents(startId, limit) throws~[Event]~
        +getEventsByType(type, startId, limit) throws~[Event]~
        +getEventsByCorrelationId(correlationId) throws~[Event]~
    }
    
    class EventDataModel {
        -modelName String
        -persistentContainer NSPersistentContainer
        +shared EventDataModel
        -createEventModel() NSManagedObjectModel
        +viewContext NSManagedObjectContext
        +newBackgroundContext() NSManagedObjectContext
        +performBackgroundTask(task) async
    }
    
    class CoreDataSerialization {
        +serializePayload(payload) throws~Data~
        +deserializePayload(data) throws~[String: AnyCodable]~
        +eventToEntity(event, context) throws~EventEntity~
        +entityToEvent(entity) throws~Event~
    }
    
    class EventEntity {
        +id String
        +type String
        +timestamp Date
        +payloadData Data
        +metadata EventMetadataEntity
    }
    
    class EventMetadataEntity {
        +userId String?
        +deviceId String?
        +version String
        +correlationId String?
        +causationId String?
        +event EventEntity
    }

    CoreDataEventStore --> EventDataModel : uses
    CoreDataEventStore --> CoreDataSerialization : uses
    EventDataModel --> EventEntity : manages
    EventDataModel --> EventMetadataEntity : manages
    EventEntity --> EventMetadataEntity : has one
    EventMetadataEntity --> EventEntity : belongs to
```

## Storage Implementation Factory

```mermaid
graph TD
    A[Client Code] --> B{EventStoreType}
    B -->|inMemory| C[InMemoryEventStore]
    B -->|coreData| D[CoreDataEventStore]
    B -->|custom| E[Custom EventStore]
    
    C --> F[EventStore Protocol]
    D --> F
    E --> F
    
    D --> G[EventDataModel]
    D --> H[CoreDataSerialization]
    G --> I[Core Data Stack]
    
    style A fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style B fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style C fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    style D fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    style E fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    style F fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    style G fill:#e0f7fa,stroke:#006064,stroke-width:2px
    style H fill:#e0f7fa,stroke:#006064,stroke-width:2px
    style I fill:#e0f7fa,stroke:#006064,stroke-width:2px
```

## Event Storage and Retrieval Sequence

```mermaid
sequenceDiagram
    participant Client
    participant Store as EventStore
    participant Migration as EventMigrationRegistry
    participant Context as NSManagedObjectContext
    participant Serialization as CoreDataSerialization
    
    %% Store Event Sequence
    Client->>Store: append(event)
    activate Store
    
    Store->>Context: newBackgroundContext()
    activate Context
    Context-->>Store: context
    deactivate Context
    
    Store->>Context: performBackgroundTask(task)
    activate Context
    
    Context->>Serialization: eventToEntity(event, context)
    activate Serialization
    Serialization->>Serialization: serializePayload(event.payload)
    Serialization-->>Context: entity
    deactivate Serialization
    
    Context->>Context: save()
    Context-->>Store: success
    deactivate Context
    
    Store-->>Client: success
    deactivate Store
    
    %% Retrieve Events Sequence
    Client->>Store: getEvents(startId, limit)
    activate Store
    
    Store->>Context: newBackgroundContext()
    activate Context
    Context-->>Store: context
    deactivate Context
    
    Store->>Context: performBackgroundTask(task)
    activate Context
    
    Context->>Context: fetchRequest()
    Context->>Context: execute(fetchRequest)
    Context-->>Store: entities
    deactivate Context
    
    loop For each entity
        Store->>Serialization: entityToEvent(entity)
        activate Serialization
        Serialization->>Serialization: deserializePayload(entity.payloadData)
        Serialization-->>Store: event
        deactivate Serialization
        
        Store->>Migration: migrateEvent(event)
        activate Migration
        Migration-->>Store: migratedEvent
        deactivate Migration
    end
    
    Store-->>Client: events
    deactivate Store
```

## Storage Implementation Considerations

In the MonoSphere application, event storage follows these key design principles:

1. **Abstraction** - The EventStore protocol abstracts away storage implementation details
2. **Pluggability** - Multiple storage implementations can be swapped based on needs
3. **Migration Support** - All storage implementations support event migration
4. **Serialization** - Complex event payloads are serialized to binary data for storage
5. **Asynchronous Operations** - Storage operations are performed asynchronously
6. **Background Processing** - Core Data operations run in background contexts
7. **Transactional Safety** - Core Data provides transaction safety for event storage