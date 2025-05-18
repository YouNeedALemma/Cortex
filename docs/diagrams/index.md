# MonoSphere Diagrams Index

This index provides an overview of all the architectural and component diagrams available for the MonoSphere application.

## Architecture Diagrams

These diagrams provide a high-level view of the system architecture:

1. [**Event Sourcing Architecture**](./event-sourcing-architecture.md)
   - Core event sourcing pattern and flow
   - Event structure and types
   - Event flow through the system
   
2. [**Application Layered Architecture**](./application-layered-architecture.md)
   - Clean architecture layers
   - Layered architecture with concerns
   - Data flow through layers
   - Clean architecture onion model
   
## Component Diagrams

These diagrams illustrate specific components and their relationships:

3. [**Event Storage Architecture**](./event-storage-architecture.md)
   - Event store protocol hierarchy
   - Core Data storage components
   - Storage implementation factory
   - Event storage and retrieval sequence
   
4. [**Command Processing Flow**](./command-processing-flow.md) 
   - Command flow
   - Command processing sequence
   - Command structure
   - Validation process
   
5. [**Event Versioning and Migration**](./event-versioning-migration.md)
   - Event migration registry
   - Versioning structure
   - Migration path discovery
   - Migration sequence
   
## Domain Model Diagrams

These diagrams show the domain model structure and relationships:

6. [**Domain Model Relationships**](./domain-model-relationships.md)
   - Core domain models
   - Entity relationships
   - Model creation and mutation flows
   - Immutable update patterns
   - State container composition
   - Domain event types
   
## Module Diagrams

These diagrams illustrate the module structure and dependencies:

7. [**Module Dependencies**](./module-dependencies.md)
   - Swift package module structure
   - Module dependencies
   - Models module decomposition
   - Features module composition
   - Common module components
   - Dependency flow between modules
   
## Integration Diagrams

These diagrams show integration with external frameworks:

8. [**TCA Integration**](./tca-integration.md)
   - TCA component structure
   - TCA and event sourcing integration
   - TCA action to command flow
   - Projection to TCA state flow
   - TCA environment integration
   - Example TCA reducer

## Diagram Reading Guide

When interpreting these diagrams, note that:

- **Color Scheme**:
  - Blue: UI components, user interactions, and TCA components
  - Purple: Domain models, commands, and business logic
  - Green: Event sourcing, events, and projections
  - Orange: Storage components and persistence
  - Yellow: Validation, migration, and cross-cutting concerns

- **Arrow Directions**:
  - Arrows generally indicate the flow of data or control
  - Solid lines indicate direct dependencies
  - Dotted lines indicate conceptual relationships

- **Component Types**:
  - Rectangles with solid borders represent concrete components
  - Rectangles with dashed borders represent interfaces or protocols
  - Rounded rectangles often represent processes or actions
  - Diamonds typically represent decision points