# Android Concepts & Deep Dives

In-depth explanations of how Android systems work under the hood.

## 📚 Available Guides

### [Lifecycle Deep Dive](./lifecycle.md)
Understand Android and Compose lifecycles:
- Activity and Fragment lifecycle
- ViewModel lifecycle
- Composable lifecycle
- Lifecycle observers
- Process lifecycle
- When to use each lifecycle method

### [Compose Rendering](./compose-rendering.md)
How Compose draws your UI:
- Composition, Layout, and Drawing phases
- Recomposition triggers
- Smart recomposition
- Skipping and stability
- Performance implications
- Composition vs recomposition

### [Coroutines & Flow Internals](./coroutines-flow-internals.md)
How coroutines and Flow work:
- Coroutine context and dispatchers
- Structured concurrency
- Job hierarchy
- Cancellation propagation
- Flow operators internals
- Cold vs hot flows
- Channels and buffers

### [System Access](./system-access.md)
Understanding Android system services:
- Context types (Application vs Activity)
- System services overview
- Service managers
- Content providers
- Broadcast receivers
- Permissions model

### [Room Transactions](./room-transactions.md)
Database transactions explained:
- ACID properties
- Transaction scope
- @Transaction annotation
- Nested transactions
- Conflict strategies
- Write-ahead logging
- Performance considerations

### [Material3 Design System](./material3-concepts.md)
Material3 design principles:
- Design tokens
- Color schemes (static vs dynamic)
- Typography scale
- Shape system
- Motion and transitions
- Adaptive layouts
- Accessibility in Material3

### [Background Work](./background-work.md)
Job scheduling and alarms:
- WorkManager vs AlarmManager vs JobScheduler
- Doze mode and app standby
- Background execution limits
- Foreground services
- Exact alarms
- When to use each approach

### [OAuth2 & JWT](./oauth-jwt.md)
Authentication concepts:
- OAuth2 flow explained
- Authorization vs Authentication
- JWT structure and validation
- Token refresh strategies
- Secure token storage
- PKCE for mobile apps
- Common security pitfalls

### [DateTime Handling](./datetime.md)
Date and time across Android versions:
- java.time (API 26+) vs legacy Date/Calendar
- Core library desugaring
- ThreeTenABP for older APIs
- Timezone handling
- Storage recommendations
- Formatting and parsing
- Common pitfalls (SimpleDateFormat, Calendar months)
- Version-safe utilities
- Compose DatePicker/TimePicker integration

### [WebView](./webview.md)
Understanding WebView and web content:
- WebView vs Custom Tabs vs Browser
- Security settings and best practices
- JavaScript bridge (Android ↔ JavaScript)
- File upload and camera support
- Cookie management
- Loading local assets
- WebView in Compose
- Memory management and cleanup
- Debugging WebView
- When NOT to use WebView

## 🎯 How This Differs

| Folder | Purpose | Content |
|--------|---------|---------|
| **setup/** | Configuration | "How to set it up" |
| **patterns/** | Implementation | "How to use it" |
| **concepts/** | Understanding | "How it works" |

## 💡 When to Use These Guides

- 📖 **Learning**: Understand fundamentals before implementation
- 🐛 **Debugging**: Know how systems work to fix issues
- 🏗️ **Architecture**: Make informed decisions
- 📚 **Teaching**: Help others understand Android
- 🎓 **Interviews**: Deepen your knowledge
- 🔍 **Troubleshooting**: Diagnose problems effectively

## 🎓 Recommended Reading Order

### For Beginners
1. [Lifecycle Deep Dive](./lifecycle.md)
2. [System Access](./system-access.md)
3. [Compose Rendering](./compose-rendering.md)

### For Intermediate
1. [Coroutines & Flow Internals](./coroutines-flow-internals.md)
2. [Room Transactions](./room-transactions.md)
3. [Material3 Design System](./material3-concepts.md)

### For Advanced
1. [Background Work](./background-work.md)
2. [OAuth2 & JWT](./oauth-jwt.md)

## 🔗 Related Content

- [Setup Guides](../setup/) - Configure these libraries
- [Pattern Guides](../patterns/) - Implement these patterns
- [Quick Start](../QUICKSTART.md) - Get started quickly

## 🤝 Contributing

These guides aim to explain complex topics clearly. If you have:
- Better explanations
- Diagrams or visualizations
- Real-world examples
- Common misconceptions to clarify

Feel free to contribute!

---

**Deep understanding leads to better code.** 💡

