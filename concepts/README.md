# Concepts

How Android systems work.

## Available Guides

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

## Organization

| Folder | Purpose |
|--------|---------|
| setup/ | Library configuration |
| patterns/ | Code implementation |
| concepts/ | System internals |

## Reading Order

**Beginner:**
1. Lifecycle
2. System Access
3. Compose Rendering

**Intermediate:**
1. Coroutines & Flow Internals
2. Room Transactions
3. Material3 Concepts

**Advanced:**
1. Background Work
2. OAuth2 & JWT
3. DateTime Handling
4. WebView

## Related

- [Setup Guides](../setup/)
- [Pattern Guides](../patterns/)
- [Quick Start](../QUICKSTART.md)

