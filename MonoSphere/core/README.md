# MonoSphere Core

Shared domain models and business logic that can be utilized by both iOS and web clients.

## Overview

The core module contains:

1. Domain models as immutable data structures
2. Pure functions for business logic
3. Type definitions shared across platforms
4. Event sourcing primitives

## Structure

- `models/`: Immutable domain models
- `logic/`: Pure functions implementing business logic
- `events/`: Event definitions for event sourcing
- `types/`: TypeScript type definitions (used by web client and for documentation)

## Design Principles

1. **Immutability**: All data structures are immutable
2. **Pure Functions**: Business logic implemented as pure functions
3. **Type Safety**: Strong typing for all models and operations
4. **Event Sourcing**: State changes modeled as immutable events