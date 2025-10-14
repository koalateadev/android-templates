# Android Setup Templates

A collection of setup instructions and configurations for common Android development libraries and frameworks.

## 📚 Setup Guides

### Core Dependencies
- [Gradle Setup & Version Catalogs](./setup/gradle-version-catalogs.md)
- [Kotlin Configuration](./setup/kotlin-setup.md)

### UI & Design
- [Jetpack Compose](./setup/jetpack-compose.md)
- [Material3](./setup/material3.md)

### Networking
- [OkHttp](./setup/okhttp.md)
- [Retrofit](./setup/retrofit.md)

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

## 🚀 Quick Start

1. Choose the libraries you need for your project
2. Follow the setup instructions in each guide
3. Copy and adapt the configuration snippets to your project

## 📝 Notes

- All examples use Gradle Kotlin DSL (`.kts`)
- Versions are current as of October 2025
- Always check for the latest stable versions on Maven Central

## 🎯 Common Patterns

Practical code templates for everyday scenarios (11 comprehensive guides):

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

## 🤝 Contributing

Feel free to add more setup guides or update existing ones as libraries evolve.

