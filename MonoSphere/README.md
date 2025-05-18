# MonoSphere

A unified personal productivity ecosystem following functional programming principles with event sourcing architecture.

## Project Structure

```
MonoSphere/
├── README.md                       # Project overview (you are here)
├── IMPLEMENTATION_SUMMARY.md       # Implementation overview
├── core/                           # Core domain models and logic (TypeScript)
│   ├── events/                     # Event sourcing implementation
│   │   ├── Event.ts                # Event interface
│   │   ├── InMemoryEventStore.ts   # In-memory event store
│   │   ├── ProjectEvents.ts        # Project-related events
│   │   ├── TaskEvents.ts           # Task-related events
│   │   ├── Projections.ts          # State projection from events
│   │   └── index.ts                # Event type exports
│   ├── models/                     # Domain models
│   │   ├── CommandHandler.ts       # Command handling
│   │   └── Storage.ts              # Storage interface
│   ├── types/                      # Type definitions
│   │   ├── Project.ts              # Project domain model
│   │   ├── Task.ts                 # Task domain model
│   │   └── index.ts                # Type exports
│   ├── utils/                      # Utility functions
│   │   └── immutable.ts            # Immutable data helpers
│   └── tests/                      # Unit and property-based tests
├── ios/                            # iOS client (SwiftUI)
│   ├── MonoSphere/                 # Swift package
│   │   ├── Sources/                # Source code
│   │   │   ├── App/                # App target
│   │   │   ├── Models/             # Domain models and event sourcing
│   │   │   │   ├── Events/         # Event implementations
│   │   │   │   └── Storage/        # Storage implementations
│   │   │   ├── Features/           # Feature modules
│   │   │   └── Common/             # Common utilities and components
│   └── MonoSphereApp/              # iOS app target
└── web/                            # Web client (React/TypeScript)
    ├── src/                        # Source code
    │   ├── components/             # UI components
    │   ├── hooks/                  # Custom React hooks
    │   ├── models/                 # Domain models
    │   ├── pages/                  # Page components
    │   ├── state/                  # State management
    │   ├── utils/                  # Utility functions
    │   └── tests/                  # Test suites
    └── public/                     # Static assets
```

## Architecture

MonoSphere follows an event-sourcing architecture with these key characteristics:

- **Event Sourcing**: All changes to application state are recorded as a sequence of immutable events
- **Domain-Driven Design**: Clear bounded contexts with rich domain models
- **Functional Programming**: Pure functions, immutable data structures, and declarative patterns
- **Type Safety**: Strong typing across the entire stack to prevent runtime errors
- **Local-First**: Full functionality offline with optional synchronization

## Core Components

### Event Sourcing System

The core of MonoSphere is built around event sourcing:

- **Events**: Immutable records of all state changes
- **Event Store**: Repository for storing and retrieving events
- **Projections**: Transform event streams into application state
- **Command Handler**: Process user commands and emit events
- **Validation**: Ensure domain rules and constraints are enforced

### Domain Models

Key domain models include:

- **Project**: Group related tasks, track progress and manage deliverables
- **Task**: Individual work items with priority, status and metadata
- **Event**: The core event model with metadata and payload
- **CommandHandler**: Process commands and apply business rules

### Storage Implementations

Multiple storage options:

- **InMemoryEventStore**: Volatile storage for development and testing
- **CoreDataEventStore**: Persistent storage for iOS using CoreData
- **IndexedDBStorage**: Persistent storage for web using IndexedDB
- **LocalStorageStorage**: Fallback storage for web using LocalStorage

## Core Principles

1. **Functional Purity**: All operations modeled as pure functions
2. **Immutable Data**: All data structures are immutable 
3. **Local-First**: Complete data ownership with optional synchronization
4. **Type Safety**: Strong typing across the entire stack

## Advanced Features

- **Event Versioning**: Schema evolution with migration path
- **Error Handling**: Comprehensive domain-specific error types
- **Validation**: Input validation for all command operations
- **Type-Safe Projections**: Strongly typed state projection from events

## Getting Started

### iOS Application

Requirements:
- Xcode 14.0+
- Swift 5.7+
- iOS 16.0+

To run:
1. Open the `ios/MonoSphere.xcodeproj` in Xcode
2. Build and run the project

### Web Application

Requirements:
- Node.js 18+
- Yarn 1.22+
- TypeScript 4.9+

To run:
1. Navigate to the `web` directory
2. Run `yarn install` to install dependencies
3. Run `yarn dev` to start development server

## Testing

The project uses both unit tests and property-based tests:

- **Swift Tests**: Unit tests for Swift domain models and event sourcing
- **TypeScript Tests**: Unit and property-based tests for core logic

## Development

See [development-workflow.md](../development-workflow.md) for detailed development practices.

## Current Status

This project is in active development. The core event sourcing system and domain models are functional, with ongoing work on the UI and feature implementations.