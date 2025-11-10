# Android Development Templates

Setup instructions and code patterns for modern Android development.

## Setup Guides

### Core Dependencies
- [Gradle Setup & Version Catalogs](./setup/gradle-version-catalogs.md)
- [Kotlin Configuration](./setup/kotlin-setup.md)

### UI & Design
- [Jetpack Compose](./setup/jetpack-compose.md)
- [Material3](./setup/material3.md)

### Networking
- [OkHttp](./setup/okhttp.md)
- [Retrofit](./setup/retrofit.md)
- [Moshi](./setup/moshi.md)

### Database & Storage
- [Room](./setup/room.md)
- [DataStore](./setup/datastore.md)

### Architecture & DI
- [Hilt (Dependency Injection)](./setup/hilt.md)
- [Navigation Component](./setup/navigation.md)
- [ViewModel & Lifecycle](./setup/viewmodel-lifecycle.md)

### Asynchronous Programming
- [Kotlin Coroutines](./setup/coroutines.md)
- [Flow](./setup/flow.md)

### Testing
- [Unit Testing (JUnit, Mockk)](./setup/unit-testing.md)
- [Turbine (Flow Testing)](./setup/turbine.md)
- [UI Testing (Compose)](./setup/compose-ui-testing.md)

### Build & Configuration
- [Build Variants & Flavors](./setup/build-variants.md)
- [ProGuard/R8](./setup/proguard.md)

### Additional Tools
- [Coil (Image Loading)](./setup/coil.md)
- [Timber (Logging)](./setup/timber.md)
- [LeakCanary](./setup/leakcanary.md)
- [Lottie (Animations)](./setup/lottie.md)
- [Koin (DI Alternative)](./setup/koin.md)

### Advanced Features
- [Paging3](./setup/paging3.md)
- [WorkManager](./setup/workmanager.md)
- [Maps (Google Maps & Mapbox)](./setup/maps.md)
- [Apollo GraphQL](./setup/apollo.md)

### Code Quality
- [Lint](./setup/lint.md)
- [Detekt](./setup/detekt.md)

### Navigation
- [Navigation Component](./setup/navigation.md)
- [Navigation Compose (Type-Safe)](./setup/navigation-compose.md)
- [Navigation 3 (Experimental)](./setup/navigation3.md)

## Usage

1. Browse the setup guides for libraries you need
2. Copy configuration from the guide
3. Adapt to your project

## Notes

- Examples use Gradle Kotlin DSL
- Check Maven Central for latest versions
- Versions current as of October 2025

## Concepts

Understanding how Android systems work:

- [Lifecycle](./concepts/lifecycle.md) - Activity, Fragment, ViewModel, and Composable lifecycles
- [Compose Rendering](./concepts/compose-rendering.md) - Composition, Layout, Drawing phases explained
- [Coroutines & Flow Internals](./concepts/coroutines-flow-internals.md) - How coroutines and Flow work under the hood
- [System Access](./concepts/system-access.md) - Context types, system services, content providers
- [Room Transactions](./concepts/room-transactions.md) - ACID properties, transactions, conflict strategies
- [Material3 Concepts](./concepts/material3-concepts.md) - Design tokens, dynamic color, typography, elevation
- [Background Work](./concepts/background-work.md) - WorkManager vs AlarmManager, Doze mode, foreground services
- [OAuth2 & JWT](./concepts/oauth-jwt.md) - OAuth2 flow with PKCE, JWT structure, token management
- [DateTime Handling](./concepts/datetime.md) - Date/time across SDK versions, timezones, formatting
- [WebView](./concepts/webview.md) - WebView vs alternatives, security, JavaScript bridge, Compose integration

## Common Patterns

Code templates for common scenarios:

### Core Patterns
- [Compose UI Patterns](./patterns/compose-ui-patterns.md) - 60+ UI patterns: loading, errors, dialogs, forms, toasts, snackbars, filters
- [State Management](./patterns/state-management.md) - State hoisting, side effects, MVI, unidirectional data flow
- [Flow & Coroutine Patterns](./patterns/flow-coroutine-patterns.md) - StateFlow, combining flows, retry logic, caching

### Advanced Patterns
- [Animation & Transitions](./patterns/animations.md) - Screen transitions, list animations, gestures, custom animations
- [Security Patterns](./patterns/security.md) - Encryption, token management, biometric auth, certificate pinning
- [Performance Optimization](./patterns/performance.md) - Recomposition, LazyColumn, memory management

### App Features
- [Offline-First Patterns](./patterns/offline-first.md) - Network + database, sync strategies, conflict resolution
- [Accessibility](./patterns/accessibility.md) - Screen readers, touch targets, color contrast, TalkBack
- [Architecture Patterns](./patterns/architecture.md) - Repository, UseCase, mappers, Clean Architecture
- [Testing Patterns](./patterns/testing.md) - Robot pattern, fakes, ViewModel tests, UI tests

### Utilities
- [Permissions in Compose](./patterns/permissions-compose.md) - Runtime permissions, image picker
- [Common Intents](./patterns/intents.md) - Share, email, phone, maps, camera

## Contributing

Contributions welcome. Update guides as libraries evolve.

