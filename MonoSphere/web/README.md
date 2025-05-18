# MonoSphere Web Client

Web application for MonoSphere built with TypeScript, React, and functional programming libraries.

## Architecture

The application follows a functional architecture inspired by Elm:

- **State**: Immutable application state
- **Messages**: Events that can change state
- **Update**: Pure functions that compute new state based on current state and messages
- **Effects**: Side effects handled outside the core update loop
- **View**: React components that derive from state

## Project Structure

- `src/`
  - `models/`: TypeScript implementations of core domain models
  - `state/`: Application state management
  - `components/`: React components
  - `hooks/`: Custom React hooks
  - `utils/`: Utility functions
  - `services/`: Services for external interactions

## Dependencies

- [React](https://reactjs.org/): UI library
- [TypeScript](https://www.typescriptlang.org/): Type-safe JavaScript
- [fp-ts](https://github.com/gcanti/fp-ts): Functional programming utilities
- [io-ts](https://github.com/gcanti/io-ts): Runtime type checking
- [Immer](https://github.com/immerjs/immer): Immutable state updates with normal JS syntax

## Getting Started

1. Install dependencies: `yarn install`
2. Start development server: `yarn start`
3. Build for production: `yarn build`