# MonoSphere Testing Strategy

This document describes the testing strategy and patterns used in the MonoSphere application. It covers the various testing approaches, from unit testing to property-based testing, and provides guidance on how to effectively test the application's components.

## Testing Philosophy

MonoSphere follows these testing principles:

1. **Test-Driven Development**: Tests are written before or alongside implementation code.
2. **Functional Testing**: Focus on testing the behavior of pure functions.
3. **Property-Based Testing**: Test properties that should hold for all valid inputs.
4. **Immutability Testing**: Verify that data structures remain immutable.
5. **Isolation**: Tests should be isolated and not depend on each other or external systems.
6. **Comprehensive Coverage**: All critical paths and components are tested.

## Testing Layers

MonoSphere implements the following testing layers:

### 1. Unit Tests

Unit tests verify the behavior of individual functions, types, and components in isolation. They typically follow the Arrange-Act-Assert pattern:

```swift
func testTaskToggle() {
    // Arrange
    let task = Task(
        id: UUID(),
        title: "Test Task",
        description: nil,
        completed: false,
        createdAt: Date(),
        updatedAt: Date()
    )
    
    // Act
    let toggled = task.toggle()
    
    // Assert
    XCTAssertTrue(toggled.completed)
    XCTAssertEqual(task.id, toggled.id)
    XCTAssertEqual(task.title, toggled.title)
    // ... other assertions
}
```

Unit tests focus on:
- Correctness of individual functions
- Edge cases and boundary conditions
- Error handling and validation

### 2. Property-Based Tests

Property-based tests verify that certain properties or invariants hold for all valid inputs. Instead of testing specific examples, they generate many random inputs and check that the properties always hold:

```swift
func testTaskToggleIdempotence() {
    // Property: Toggling a task's completion status twice returns it to the original state
    property("Task.toggle() is idempotent when applied twice") <- forAll(taskGen) { task in
        let toggledOnce = task.toggle()
        let toggledTwice = toggledOnce.toggle()
        return task.completed == toggledTwice.completed
    }
}
```

Property-based tests focus on:
- Invariants that should always be true
- Algebraic properties (idempotence, commutativity, associativity)
- Roundtrip conversions (serialize/deserialize)
- Behavior across a wide range of inputs

### 3. Integration Tests

Integration tests verify that components work together correctly. In MonoSphere, integration tests typically involve:

```swift
func testEventStoreWithMigration() async throws {
    // Set up multiple components
    let registry = EventMigrationRegistry()
    registry.register(migrator: TaskCreatedV1_0_0ToV1_1_0Migrator())
    
    let eventStore = InMemoryEventStore(migrationRegistry: registry)
    
    // Test their interaction
    let oldEvent = Event(
        id: "test-id",
        type: TaskEventType.created,
        timestamp: Date(),
        metadata: EventMetadata(version: "1.0.0"),
        payload: [
            "taskId": AnyCodable("task-1"),
            "title": AnyCodable("Test Task"),
            // ... other fields
        ]
    )
    
    try await eventStore.append(oldEvent)
    
    // Verify the integrated behavior
    let events = try await eventStore.getEvents(startId: nil, limit: nil)
    
    XCTAssertEqual(events.count, 1)
    let migratedEvent = events[0]
    
    XCTAssertEqual(migratedEvent.metadata.version, "1.1.0")
    XCTAssertNotNil(migratedEvent.payload["estimatedHours"])
}
```

Integration tests focus on:
- Component interactions
- End-to-end workflows
- System boundaries and interfaces

## Property-Based Testing

MonoSphere uses SwiftCheck for property-based testing, which is particularly well-suited to test the functional aspects of the codebase.

### Generators

Generators define how to create random values for testing:

```swift
// UUID Generator
static let idGen: Gen<UUID> = Gen<UUID> { _ in
    return UUID()
}

// Date Generator
static let dateGen: Gen<Date> = Gen<Date> { _ in
    let randomSeconds = Double.random(in: 0..<(60 * 60 * 24 * 365))
    return Date().addingTimeInterval(randomSeconds)
}

// Task Generator
static let taskGen: Gen<Task> = Gen<Task>.compose { composer in
    return Task(
        id: composer.generate(using: idGen),
        title: composer.generate(
            using: Gen<String>.string.suchThat { !$0.isEmpty }
        ),
        description: composer.generate(using: Gen<String>.optional),
        completed: composer.generate(using: Gen<Bool>.bool),
        createdAt: composer.generate(using: dateGen),
        updatedAt: composer.generate(using: dateGen)
    )
}
```

### Properties

Properties define invariants that should hold for all valid inputs:

```swift
// Property: Adding a task to a project and then removing it results in the original project
property("Project.addTask() followed by Project.removeTask() is identity") <- forAll(projectGen, taskGen) { project, task in
    let withTask = project.addTask(task)
    let withoutTask = withTask.removeTask(id: task.id)
    
    // Only compare tasks array - we expect other properties like updatedAt to be different
    return project.tasks.count == withoutTask.tasks.count
}
```

### Common Properties to Test

MonoSphere tests these common functional properties:

1. **Idempotence**: Applying an operation twice is equivalent to applying it once
    ```swift
    func f(x) == f(f(x))
    ```

2. **Roundtrip**: Converting a value to another form and back preserves the value
    ```swift
    decode(encode(x)) == x
    ```

3. **Commutativity**: The order of operations doesn't matter
    ```swift
    f(g(x)) == g(f(x))
    ```

4. **Identity**: There exists an operation that leaves a value unchanged
    ```swift
    f(x, identity) == x
    ```

## Testing Components

### Domain Models

Domain models are tested for:

1. **Immutability**: Verify that operations return new instances
2. **Invariants**: Verify that business rules are enforced
3. **Equality**: Verify that equality is defined correctly
4. **Transformations**: Verify that update operations work correctly

```swift
func testTaskEquality() {
    let id = UUID()
    
    let task1 = Task(id: id, title: "Task", /* ... */)
    let task2 = Task(id: id, title: "Task", /* ... */)
    let task3 = Task(id: UUID(), title: "Task", /* ... */)
    
    // Tasks with the same ID should be considered equal
    XCTAssertEqual(task1, task2)
    
    // Tasks with different IDs should be considered different
    XCTAssertNotEqual(task1, task3)
}
```

### Event Sourcing

Event sourcing components are tested for:

1. **Event Creation**: Verify that events are created correctly
2. **Event Storage**: Verify that events are stored and retrieved correctly
3. **Event Projection**: Verify that state is projected correctly from events
4. **Command Handling**: Verify that commands generate appropriate events
5. **Migration**: Verify that event schema migrations work correctly

```swift
func testEventMigration() {
    let registry = EventMigrationRegistry()
    registry.register(migrator: TaskCreatedV1_0_0ToV1_1_0Migrator())
    
    // Create an event with old version
    let oldEvent = Event(
        id: "test-id",
        type: TaskEventType.created,
        timestamp: Date(),
        metadata: EventMetadata(version: "1.0.0"),
        payload: [
            "taskId": AnyCodable("task-1"),
            "title": AnyCodable("Test Task"),
            // ... other fields
        ]
    )
    
    // Migrate the event
    let migratedEvent = registry.migrateEvent(oldEvent)
    
    // Verify the migration
    XCTAssertEqual(migratedEvent.metadata.version, "1.1.0")
    XCTAssertNotNil(migratedEvent.payload["estimatedHours"])
}
```

### Storage

Storage implementations are tested for:

1. **Persistence**: Verify that data is stored and retrieved correctly
2. **Queries**: Verify that queries return the expected results
3. **Transactions**: Verify that transactions are atomic
4. **Error Handling**: Verify that errors are handled correctly
5. **Migration**: Verify that data migrations work correctly

```swift
func testCoreDataEventStore() async throws {
    // Use in-memory store for testing
    let dataModel = TestEventDataModel() // In-memory Core Data model
    let eventStore = CoreDataEventStore(dataModel: dataModel)
    
    // Test event storage and retrieval
    let event = createTestEvent(id: "test-1", type: "test.event")
    try await eventStore.append(event)
    
    let events = try await eventStore.getEvents()
    XCTAssertEqual(events.count, 1)
    XCTAssertEqual(events[0].id, "test-1")
}
```

## Test Doubles

MonoSphere uses these test doubles:

### 1. In-Memory Event Store

The `InMemoryEventStore` serves as both a production implementation and a test double:

```swift
let eventStore = InMemoryEventStore()
```

### 2. Test-Specific Core Data Model

A test-specific Core Data model uses in-memory storage:

```swift
class TestEventDataModel: EventDataModel {
    override var persistentContainer: NSPersistentContainer {
        let container = super.persistentContainer
        guard let description = container.persistentStoreDescriptions.first else {
            fatalError("No persistent store description found")
        }
        
        description.type = NSInMemoryStoreType
        
        return container
    }
}
```

### 3. Test-Specific Event Migrators

Test-specific event migrators simplify migration testing:

```swift
class TestEventMigrator: EventMigrator {
    let eventType = "test.migrate"
    let sourceVersion = EventVersion(major: 1, minor: 0, patch: 0)
    let targetVersion = EventVersion(major: 1, minor: 1, patch: 0)
    
    func migrate(payload: [String: AnyCodable]) -> [String: AnyCodable] {
        var newPayload = payload
        newPayload["migrated"] = AnyCodable(true)
        return newPayload
    }
}
```

## Best Practices

When writing tests for MonoSphere, follow these best practices:

1. **Test Pure Functions**: Focus on testing the behavior of pure functions with well-defined inputs and outputs.

2. **Use Property-Based Testing**: For functions with algebraic properties, use property-based testing to verify these properties.

3. **Test State Transitions**: Verify that state transitions work correctly, especially for event sourcing.

4. **Isolate Tests**: Tests should not depend on each other or external systems.

5. **Use Test Doubles**: Use test doubles to isolate components from their dependencies.

6. **Test Edge Cases**: Identify and test edge cases and boundary conditions.

7. **Test Error Handling**: Verify that errors are handled correctly.

8. **Keep Tests Fast**: Tests should run quickly to provide fast feedback.

9. **Make Tests Readable**: Tests should serve as documentation for how the code works.

10. **Test Business Rules**: Verify that business rules and invariants are enforced.

## Writing Effective Tests

### Unit Test Template

```swift
func testComponentBehavior() {
    // Arrange: Set up the test environment
    let component = Component(/* ... */)
    
    // Act: Perform the operation
    let result = component.operation(/* ... */)
    
    // Assert: Verify the result
    XCTAssertEqual(result, expectedResult)
}
```

### Property-Based Test Template

```swift
func testPropertyName() {
    property("Description of the property") <- forAll(generator) { input in
        // Verify that the property holds for the input
        let result = function(input)
        return propertyHolds(result)
    }
}
```

### Integration Test Template

```swift
func testIntegratedComponents() async throws {
    // Arrange: Set up the components
    let component1 = Component1(/* ... */)
    let component2 = Component2(component1)
    
    // Act: Perform the integrated operation
    let result = try await component2.operation(/* ... */)
    
    // Assert: Verify the integrated behavior
    XCTAssertEqual(result, expectedResult)
}
```

## Test Organization

Tests are organized by component and test type:

```
Models/
  Tests/
    TaskTests.swift          # Unit tests for Task model
    ProjectTests.swift       # Unit tests for Project model
    PropTests.swift          # Property-based tests
    EventVersioningTests.swift # Tests for event versioning
    CoreDataEventStoreTests.swift # Tests for Core Data event store
```

This organization ensures that tests are easy to find and understand, and that each component is thoroughly tested.