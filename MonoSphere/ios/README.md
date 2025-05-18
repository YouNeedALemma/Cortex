# MonoSphere iOS Client

Native iOS application for MonoSphere built with SwiftUI and The Composable Architecture.

## Architecture

The application follows a functional architecture based on The Composable Architecture (TCA):

- **State**: Immutable app state
- **Actions**: Events that can change state
- **Reducers**: Pure functions that compute new state based on current state and actions
- **Effects**: Side effects handled as Combine publishers
- **View**: SwiftUI views that derive from state

## Project Structure

- `Models/`: Swift implementations of core domain models
- `Features/`: Feature modules built with TCA
- `Common/`: Shared UI components and utilities
- `Services/`: Services for external interactions (local storage, networking)

## Dependencies

- [The Composable Architecture](https://github.com/pointfreeco/swift-composable-architecture): Functional architecture pattern
- [SwiftUI](https://developer.apple.com/xcode/swiftui/): Declarative UI framework
- [Combine](https://developer.apple.com/documentation/combine): Functional reactive programming framework

## Getting Started

1. Open `MonoSphere.xcodeproj`
2. Wait for Swift Package Manager to resolve dependencies
3. Build and run the project