# DateTime Handling in Android

## Overview

DateTime handling in Android requires careful consideration of SDK versions due to the introduction of `java.time` API in Android 8.0 (API 26).

## The DateTime Evolution

```
API < 26 (Pre-Oreo)
    ↓
java.util.Date (outdated, mutable)
java.util.Calendar (verbose, error-prone)
SimpleDateFormat (not thread-safe)
    ↓
API 26+ (Oreo and later)
    ↓
java.time.* (modern, immutable, thread-safe)
LocalDateTime, ZonedDateTime, Instant
DateTimeFormatter (thread-safe)
```

## Modern Approach (API 26+)

### Core Classes

```kotlin
import java.time.*
import java.time.format.DateTimeFormatter

// Point in time (no timezone)
val instant = Instant.now()
// 2025-10-20T10:30:45Z

// Local date (no time or timezone)
val localDate = LocalDate.now()
// 2025-10-20

// Local time (no date or timezone)
val localTime = LocalTime.now()
// 10:30:45

// Local datetime (no timezone)
val localDateTime = LocalDateTime.now()
// 2025-10-20T10:30:45

// Zoned datetime (with timezone)
val zonedDateTime = ZonedDateTime.now()
// 2025-10-20T10:30:45-07:00[America/Los_Angeles]

// Offset datetime (with UTC offset)
val offsetDateTime = OffsetDateTime.now()
// 2025-10-20T10:30:45-07:00
```

### Creating Dates

```kotlin
// Specific date
val date = LocalDate.of(2025, 10, 20)
val date2 = LocalDate.of(2025, Month.OCTOBER, 20)

// Specific time
val time = LocalTime.of(14, 30)
val time2 = LocalTime.of(14, 30, 45)
val time3 = LocalTime.of(14, 30, 45, 123456789)

// Specific datetime
val dateTime = LocalDateTime.of(2025, 10, 20, 14, 30)
val dateTime2 = LocalDateTime.of(date, time)

// Specific instant (epoch milliseconds)
val instant = Instant.ofEpochMilli(System.currentTimeMillis())

// From string
val parsed = LocalDate.parse("2025-10-20")
val parsed2 = LocalDateTime.parse("2025-10-20T14:30:45")
```

### Formatting

```kotlin
// Built-in formatters
val date = LocalDate.now()
date.format(DateTimeFormatter.ISO_DATE)
// 2025-10-20

date.format(DateTimeFormatter.ISO_LOCAL_DATE)
// 2025-10-20

// Custom formatter
val formatter = DateTimeFormatter.ofPattern("dd MMM yyyy")
val formatted = date.format(formatter)
// 20 Oct 2025

// With locale
val localeFormatter = DateTimeFormatter
    .ofPattern("dd MMMM yyyy", Locale.FRENCH)
val formatted2 = date.format(localeFormatter)
// 20 octobre 2025

// Predefined styles
val formatter3 = DateTimeFormatter.ofLocalizedDate(FormatStyle.FULL)
// Monday, October 20, 2025
```

### Parsing

```kotlin
val formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy")
val date = LocalDate.parse("20/10/2025", formatter)

val dateTime = LocalDateTime.parse(
    "20-10-2025 14:30",
    DateTimeFormatter.ofPattern("dd-MM-yyyy HH:mm")
)
```

### Calculations

```kotlin
val now = LocalDate.now()

// Add/subtract
val tomorrow = now.plusDays(1)
val nextWeek = now.plusWeeks(1)
val nextMonth = now.plusMonths(1)
val lastYear = now.minusYears(1)

// Period (date-based)
val period = Period.between(now, tomorrow)
println("${period.days} days")

// Duration (time-based)
val start = LocalDateTime.now()
val end = start.plusHours(2).plusMinutes(30)
val duration = Duration.between(start, end)
println("${duration.toHours()} hours ${duration.toMinutesPart()} minutes")

// Compare dates
val isBefore = date1.isBefore(date2)
val isAfter = date1.isAfter(date2)
val isEqual = date1.isEqual(date2)

// Check if date is in range
val isInRange = date.isAfter(start) && date.isBefore(end)
```

## Legacy Approach (API < 26)

### Using Date and Calendar

```kotlin
import java.util.Date
import java.util.Calendar
import java.text.SimpleDateFormat
import java.util.Locale

// Current date
val now = Date()

// Current timestamp
val timestamp = System.currentTimeMillis()
val date = Date(timestamp)

// Using Calendar (more features)
val calendar = Calendar.getInstance()

// Set specific date
calendar.set(2025, Calendar.OCTOBER, 20)  // Month is 0-indexed!
val specificDate = calendar.time

// Get components
val year = calendar.get(Calendar.YEAR)
val month = calendar.get(Calendar.MONTH)  // 0-11
val day = calendar.get(Calendar.DAY_OF_MONTH)
val hour = calendar.get(Calendar.HOUR_OF_DAY)
val minute = calendar.get(Calendar.MINUTE)

// Add/subtract
calendar.add(Calendar.DAY_OF_MONTH, 1)  // Add 1 day
calendar.add(Calendar.MONTH, -1)  // Subtract 1 month

// Format date
val sdf = SimpleDateFormat("dd MMM yyyy", Locale.getDefault())
val formatted = sdf.format(date)

// Parse date
val parsed = sdf.parse("20 Oct 2025")
```

### SimpleDateFormat Patterns

```kotlin
// Common patterns
"yyyy-MM-dd"                // 2025-10-20
"dd/MM/yyyy"                // 20/10/2025
"MMM dd, yyyy"              // Oct 20, 2025
"EEEE, MMMM dd, yyyy"       // Monday, October 20, 2025
"HH:mm:ss"                  // 14:30:45
"hh:mm a"                   // 02:30 PM
"dd MMM yyyy HH:mm"         // 20 Oct 2025 14:30
"yyyy-MM-dd'T'HH:mm:ss'Z'"  // 2025-10-20T14:30:45Z (ISO 8601)
```

## Cross-Version Compatibility

### Option 1: ThreeTenABP (Recommended for API < 26)

```toml
[versions]
threetenbp = "1.6.9"

[libraries]
threetenbp = { group = "org.threeten", name = "threetenbp", version.ref = "threetenbp" }
```

```kotlin
dependencies {
    // For API < 26, use ThreeTenABP
    implementation(libs.threetenbp)
}
```

Initialize in Application:
```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        AndroidThreeTen.init(this)
    }
}
```

Usage (same API as java.time):
```kotlin
import org.threeten.bp.LocalDate
import org.threeten.bp.LocalDateTime
import org.threeten.bp.format.DateTimeFormatter

val date = LocalDate.now()
val formatter = DateTimeFormatter.ofPattern("dd MMM yyyy")
val formatted = date.format(formatter)
```

### Option 2: Desugaring (Core Library Desugaring)

Enable `java.time` on older APIs:

```kotlin
android {
    compileOptions {
        // Enable desugaring
        isCoreLibraryDesugaringEnabled = true
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")
}
```

Now use `java.time` on all API levels:
```kotlin
import java.time.LocalDate
import java.time.format.DateTimeFormatter

val date = LocalDate.now()  // Works on API < 26!
```

### Option 3: Version Checking

```kotlin
fun getCurrentDate(): String {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        // API 26+
        val date = LocalDate.now()
        date.format(DateTimeFormatter.ofPattern("dd MMM yyyy"))
    } else {
        // API < 26
        val sdf = SimpleDateFormat("dd MMM yyyy", Locale.getDefault())
        sdf.format(Date())
    }
}
```

## Timezone Handling

### Always Consider Timezones

```kotlin
// ❌ Bad - Assumes local timezone
val now = LocalDateTime.now()
// Problem: Different time in different timezones

// ✅ Good - Explicit timezone
val now = ZonedDateTime.now(ZoneId.of("America/Los_Angeles"))

// ✅ Best - Use UTC for storage
val now = Instant.now()
// Store as: 2025-10-20T17:30:45Z
// Display in user's timezone
```

### Convert Timezones

```kotlin
val utcTime = ZonedDateTime.now(ZoneId.of("UTC"))

// Convert to specific timezone
val laTime = utcTime.withZoneSameInstant(ZoneId.of("America/Los_Angeles"))
val tokyoTime = utcTime.withZoneSameInstant(ZoneId.of("Asia/Tokyo"))

// Get user's timezone
val userZone = ZoneId.systemDefault()
val userTime = utcTime.withZoneSameInstant(userZone)
```

### Display Timezone-Aware Times

```kotlin
fun formatForUser(instant: Instant): String {
    val userZone = ZoneId.systemDefault()
    val zonedDateTime = instant.atZone(userZone)
    
    val formatter = DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm z")
    return zonedDateTime.format(formatter)
    // "20 Oct 2025 14:30 PDT"
}
```

## Storage Recommendations

### Best Practices

```kotlin
// ✅ Store as epoch milliseconds (Long)
val timestamp = Instant.now().toEpochMilli()
// Store: 1729445445000

// ✅ Store as ISO 8601 string (UTC)
val isoString = Instant.now().toString()
// Store: "2025-10-20T17:30:45Z"

// ❌ Don't store as formatted string
val formatted = "Oct 20, 2025 2:30 PM"  // Hard to parse, locale-dependent

// ❌ Don't store with timezone offset only
val withOffset = "2025-10-20T14:30:45-07:00"  // Loses timezone info
```

### Room Storage

```kotlin
// Option 1: Store as Long (epoch millis)
@Entity
data class Event(
    @PrimaryKey val id: String,
    val name: String,
    val timestamp: Long  // Epoch milliseconds
)

// Option 2: Type converter
class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Instant? {
        return value?.let { Instant.ofEpochMilli(it) }
    }
    
    @TypeConverter
    fun toTimestamp(instant: Instant?): Long? {
        return instant?.toEpochMilli()
    }
}

@Database(entities = [Event::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase()

// Usage
@Entity
data class Event(
    @PrimaryKey val id: String,
    val name: String,
    val eventTime: Instant  // Automatically converted
)
```

### API Response Handling

```kotlin
// Backend sends ISO 8601 string
@Serializable
data class EventDto(
    val id: String,
    val name: String,
    @Serializable(with = InstantSerializer::class)
    val eventTime: Instant
)

// Custom serializer for Instant
object InstantSerializer : KSerializer<Instant> {
    override val descriptor = PrimitiveSerialDescriptor("Instant", PrimitiveKind.STRING)
    
    override fun serialize(encoder: Encoder, value: Instant) {
        encoder.encodeString(value.toString())
    }
    
    override fun deserialize(decoder: Decoder): Instant {
        return Instant.parse(decoder.decodeString())
    }
}
```

## Common Use Cases

### Display Relative Time

```kotlin
fun getRelativeTime(instant: Instant): String {
    val now = Instant.now()
    val duration = Duration.between(instant, now)
    
    return when {
        duration.seconds < 60 -> "just now"
        duration.toMinutes() < 60 -> "${duration.toMinutes()}m ago"
        duration.toHours() < 24 -> "${duration.toHours()}h ago"
        duration.toDays() < 7 -> "${duration.toDays()}d ago"
        duration.toDays() < 30 -> "${duration.toDays() / 7}w ago"
        else -> {
            val date = instant.atZone(ZoneId.systemDefault()).toLocalDate()
            date.format(DateTimeFormatter.ofPattern("dd MMM yyyy"))
        }
    }
}

// Using Android's DateUtils (works on all API levels)
import android.text.format.DateUtils

fun getRelativeTimeSpan(timestamp: Long): String {
    return DateUtils.getRelativeTimeSpanString(
        timestamp,
        System.currentTimeMillis(),
        DateUtils.MINUTE_IN_MILLIS
    ).toString()
    // "3 minutes ago", "Yesterday", etc.
}
```

### Format for Display

```kotlin
// User-friendly date
fun formatDate(instant: Instant): String {
    val zonedDateTime = instant.atZone(ZoneId.systemDefault())
    
    val formatter = DateTimeFormatter.ofPattern("MMM dd, yyyy")
    return zonedDateTime.format(formatter)
    // "Oct 20, 2025"
}

// Full datetime
fun formatDateTime(instant: Instant): String {
    val zonedDateTime = instant.atZone(ZoneId.systemDefault())
    
    val formatter = DateTimeFormatter.ofPattern("dd MMM yyyy 'at' hh:mm a")
    return zonedDateTime.format(formatter)
    // "20 Oct 2025 at 02:30 PM"
}

// ISO 8601 for API
fun formatForApi(instant: Instant): String {
    return instant.toString()
    // "2025-10-20T17:30:45Z"
}
```

### Parse from String

```kotlin
// Parse ISO 8601
fun parseIso8601(dateString: String): Instant {
    return Instant.parse(dateString)
    // "2025-10-20T17:30:45Z"
}

// Parse custom format
fun parseCustom(dateString: String): LocalDate {
    val formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy")
    return LocalDate.parse(dateString, formatter)
    // "20/10/2025"
}

// Safe parsing
fun safeParse(dateString: String): Instant? {
    return try {
        Instant.parse(dateString)
    } catch (e: Exception) {
        null
    }
}
```

### Calculate Age

```kotlin
fun calculateAge(birthDate: LocalDate): Int {
    val today = LocalDate.now()
    return Period.between(birthDate, today).years
}

// Usage
val birthDate = LocalDate.of(1990, 5, 15)
val age = calculateAge(birthDate)
// 35 (in 2025)
```

### Days Until Event

```kotlin
fun daysUntil(eventDate: LocalDate): Long {
    val today = LocalDate.now()
    return ChronoUnit.DAYS.between(today, eventDate)
}

// Usage
val christmas = LocalDate.of(2025, 12, 25)
val daysLeft = daysUntil(christmas)
// 66 days
```

### Time Ago

```kotlin
fun timeAgo(instant: Instant): String {
    val now = Instant.now()
    val duration = Duration.between(instant, now)
    
    return when {
        duration.seconds < 60 -> "just now"
        duration.toMinutes() == 1L -> "1 minute ago"
        duration.toMinutes() < 60 -> "${duration.toMinutes()} minutes ago"
        duration.toHours() == 1L -> "1 hour ago"
        duration.toHours() < 24 -> "${duration.toHours()} hours ago"
        duration.toDays() == 1L -> "yesterday"
        duration.toDays() < 7 -> "${duration.toDays()} days ago"
        duration.toDays() < 30 -> "${duration.toDays() / 7} weeks ago"
        duration.toDays() < 365 -> "${duration.toDays() / 30} months ago"
        else -> "${duration.toDays() / 365} years ago"
    }
}
```

## Android-Specific Utilities

### DateUtils (All API Levels)

```kotlin
import android.text.format.DateUtils

// Relative time
val relative = DateUtils.getRelativeTimeSpanString(
    timestamp,
    System.currentTimeMillis(),
    DateUtils.MINUTE_IN_MILLIS,
    DateUtils.FORMAT_ABBREV_RELATIVE
)
// "3 min ago"

// Format date range
val range = DateUtils.formatDateRange(
    context,
    startMillis,
    endMillis,
    DateUtils.FORMAT_SHOW_DATE or DateUtils.FORMAT_SHOW_YEAR
)
// "Oct 20 - Oct 27, 2025"

// Is today/yesterday
val isToday = DateUtils.isToday(timestamp)
val isYesterday = DateUtils.isToday(timestamp + DateUtils.DAY_IN_MILLIS)
```

### DateFormat (System Locale)

```kotlin
import android.text.format.DateFormat

val context: Context = ...

// Get user's preferred date format
val dateFormat = DateFormat.getDateFormat(context)
val formatted = dateFormat.format(Date())
// Respects user's system settings

// Get time format (12h vs 24h)
val timeFormat = DateFormat.getTimeFormat(context)
val formattedTime = timeFormat.format(Date())
// "2:30 PM" or "14:30" based on user preference

// Check if 24-hour format
val is24Hour = DateFormat.is24HourFormat(context)
```

## Version-Safe DateTime Utilities

### Utility Class

```kotlin
object DateTimeUtils {
    
    // Get current timestamp
    fun now(): Long {
        return System.currentTimeMillis()
    }
    
    // Format timestamp for display
    fun format(timestamp: Long, pattern: String = "dd MMM yyyy"): String {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val instant = Instant.ofEpochMilli(timestamp)
            val zonedDateTime = instant.atZone(ZoneId.systemDefault())
            val formatter = DateTimeFormatter.ofPattern(pattern)
            zonedDateTime.format(formatter)
        } else {
            val sdf = SimpleDateFormat(pattern, Locale.getDefault())
            sdf.format(Date(timestamp))
        }
    }
    
    // Parse string to timestamp
    fun parse(dateString: String, pattern: String = "yyyy-MM-dd"): Long? {
        return try {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                val formatter = DateTimeFormatter.ofPattern(pattern)
                val date = LocalDate.parse(dateString, formatter)
                date.atStartOfDay(ZoneId.systemDefault()).toInstant().toEpochMilli()
            } else {
                val sdf = SimpleDateFormat(pattern, Locale.getDefault())
                sdf.parse(dateString)?.time
            }
        } catch (e: Exception) {
            null
        }
    }
    
    // Add days to timestamp
    fun addDays(timestamp: Long, days: Int): Long {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val instant = Instant.ofEpochMilli(timestamp)
            val zonedDateTime = instant.atZone(ZoneId.systemDefault())
            zonedDateTime.plusDays(days.toLong()).toInstant().toEpochMilli()
        } else {
            val calendar = Calendar.getInstance()
            calendar.timeInMillis = timestamp
            calendar.add(Calendar.DAY_OF_MONTH, days)
            calendar.timeInMillis
        }
    }
    
    // Check if same day
    fun isSameDay(timestamp1: Long, timestamp2: Long): Boolean {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val instant1 = Instant.ofEpochMilli(timestamp1)
            val instant2 = Instant.ofEpochMilli(timestamp2)
            val date1 = instant1.atZone(ZoneId.systemDefault()).toLocalDate()
            val date2 = instant2.atZone(ZoneId.systemDefault()).toLocalDate()
            date1 == date2
        } else {
            DateUtils.isToday(timestamp1) && DateUtils.isToday(timestamp2)
        }
    }
}
```

## Compose Integration

### Date Picker

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DatePickerExample() {
    var showDialog by remember { mutableStateOf(false) }
    var selectedDate by remember { mutableStateOf<Long?>(null) }
    
    Button(onClick = { showDialog = true }) {
        Text(
            if (selectedDate != null) {
                DateTimeUtils.format(selectedDate!!)
            } else {
                "Select Date"
            }
        )
    }
    
    if (showDialog) {
        val datePickerState = rememberDatePickerState(
            initialSelectedDateMillis = selectedDate
        )
        
        DatePickerDialog(
            onDismissRequest = { showDialog = false },
            confirmButton = {
                TextButton(onClick = {
                    selectedDate = datePickerState.selectedDateMillis
                    showDialog = false
                }) {
                    Text("OK")
                }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false }) {
                    Text("Cancel")
                }
            }
        ) {
            DatePicker(state = datePickerState)
        }
    }
}
```

### Time Picker

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TimePickerExample() {
    var showDialog by remember { mutableStateOf(false) }
    var selectedHour by remember { mutableStateOf(12) }
    var selectedMinute by remember { mutableStateOf(0) }
    
    Button(onClick = { showDialog = true }) {
        Text("${selectedHour.toString().padStart(2, '0')}:${selectedMinute.toString().padStart(2, '0')}")
    }
    
    if (showDialog) {
        val timePickerState = rememberTimePickerState(
            initialHour = selectedHour,
            initialMinute = selectedMinute
        )
        
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("Select Time") },
            text = { TimePicker(state = timePickerState) },
            confirmButton = {
                TextButton(onClick = {
                    selectedHour = timePickerState.hour
                    selectedMinute = timePickerState.minute
                    showDialog = false
                }) {
                    Text("OK")
                }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false }) {
                    Text("Cancel")
                }
            }
        )
    }
}
```

## Common Patterns

### Countdown Timer

```kotlin
@Composable
fun CountdownTimer(targetTime: Instant) {
    var timeRemaining by remember { mutableStateOf("") }
    
    LaunchedEffect(targetTime) {
        while (true) {
            val now = Instant.now()
            val duration = Duration.between(now, targetTime)
            
            if (duration.isNegative) {
                timeRemaining = "Expired"
                break
            }
            
            val hours = duration.toHours()
            val minutes = duration.toMinutesPart()
            val seconds = duration.toSecondsPart()
            
            timeRemaining = String.format("%02d:%02d:%02d", hours, minutes, seconds)
            
            delay(1000)
        }
    }
    
    Text(timeRemaining, style = MaterialTheme.typography.displayMedium)
}
```

### Clock Display

```kotlin
@Composable
fun LiveClock() {
    var currentTime by remember { mutableStateOf(LocalTime.now()) }
    
    LaunchedEffect(Unit) {
        while (true) {
            currentTime = LocalTime.now()
            delay(1000)
        }
    }
    
    val formatter = DateTimeFormatter.ofPattern("HH:mm:ss")
    Text(currentTime.format(formatter))
}
```

### Schedule Notification

```kotlin
fun scheduleNotification(context: Context, dateTime: LocalDateTime) {
    val zonedDateTime = dateTime.atZone(ZoneId.systemDefault())
    val triggerTime = zonedDateTime.toInstant().toEpochMilli()
    
    val alarmManager = context.getSystemService<AlarmManager>()!!
    
    val intent = Intent(context, NotificationReceiver::class.java)
    val pendingIntent = PendingIntent.getBroadcast(
        context,
        0,
        intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )
    
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        if (alarmManager.canScheduleExactAlarms()) {
            alarmManager.setExact(
                AlarmManager.RTC_WAKEUP,
                triggerTime,
                pendingIntent
            )
        }
    } else {
        alarmManager.setExact(
            AlarmManager.RTC_WAKEUP,
            triggerTime,
            pendingIntent
        )
    }
}
```

### Date Range Filter

```kotlin
fun filterByDateRange(
    items: List<Item>,
    startDate: LocalDate,
    endDate: LocalDate
): List<Item> {
    return items.filter { item ->
        val itemDate = Instant.ofEpochMilli(item.timestamp)
            .atZone(ZoneId.systemDefault())
            .toLocalDate()
        
        !itemDate.isBefore(startDate) && !itemDate.isAfter(endDate)
    }
}
```

## Localization

### Locale-Aware Formatting

```kotlin
fun formatForLocale(instant: Instant, locale: Locale): String {
    val zonedDateTime = instant.atZone(ZoneId.systemDefault())
    
    val formatter = DateTimeFormatter
        .ofPattern("dd MMMM yyyy", locale)
    
    return zonedDateTime.format(formatter)
}

// Usage
formatForLocale(instant, Locale.US)        // "20 October 2025"
formatForLocale(instant, Locale.FRENCH)    // "20 octobre 2025"
formatForLocale(instant, Locale.JAPANESE)  // "20 10月 2025"
```

### Localized Formats

```kotlin
import java.time.format.FormatStyle

fun localizedFormat(instant: Instant): String {
    val zonedDateTime = instant.atZone(ZoneId.systemDefault())
    
    val formatter = DateTimeFormatter.ofLocalizedDateTime(
        FormatStyle.MEDIUM
    ).withLocale(Locale.getDefault())
    
    return zonedDateTime.format(formatter)
    // Automatically adapts to user's locale
}
```

## Common Pitfalls

### ❌ Using SimpleDateFormat Without Locale

```kotlin
// Bad - Uses default locale (can change)
val sdf = SimpleDateFormat("dd/MM/yyyy")

// Good - Explicit locale
val sdf = SimpleDateFormat("dd/MM/yyyy", Locale.US)
```

### ❌ SimpleDateFormat Thread Safety

```kotlin
// Bad - SimpleDateFormat is not thread-safe
class BadRepository {
    private val dateFormat = SimpleDateFormat("yyyy-MM-dd", Locale.US)
    
    suspend fun formatDate(date: Date): String = withContext(Dispatchers.IO) {
        dateFormat.format(date)  // Can crash with concurrent access!
    }
}

// Good - Create new instance or synchronize
class GoodRepository {
    suspend fun formatDate(date: Date): String = withContext(Dispatchers.IO) {
        val sdf = SimpleDateFormat("yyyy-MM-dd", Locale.US)
        sdf.format(date)  // Safe
    }
}

// Better - Use java.time (thread-safe)
class BestRepository {
    suspend fun formatDate(instant: Instant): String {
        val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd")
        val date = instant.atZone(ZoneId.systemDefault()).toLocalDate()
        return date.format(formatter)  // Thread-safe
    }
}
```

### ❌ Storing Formatted Dates

```kotlin
// Bad - Hard to parse, locale-dependent
@Entity
data class Event(
    val formattedDate: String  // "Oct 20, 2025"
)

// Good - Store as timestamp
@Entity
data class Event(
    val timestamp: Long  // 1729445445000
)
```

### ❌ Ignoring Timezones

```kotlin
// Bad - Assumes local timezone
val dateTime = LocalDateTime.now()
sendToServer(dateTime.toString())
// Server in different timezone gets confused

// Good - Use UTC
val instant = Instant.now()
sendToServer(instant.toString())
// "2025-10-20T17:30:45Z" - unambiguous
```

### ❌ Calendar Month Confusion

```kotlin
// Bad - Calendar months are 0-indexed!
val calendar = Calendar.getInstance()
calendar.set(2025, 10, 20)  // This is November 20th, not October!

// Good - Use constant
calendar.set(2025, Calendar.OCTOBER, 20)  // October 20th

// Better - Use java.time (1-indexed)
val date = LocalDate.of(2025, 10, 20)  // October 20th
```

## Best Practices

### 1. Choose the Right Approach

```kotlin
// For new apps (minSdk >= 26)
// ✅ Use java.time directly

// For apps supporting older devices
// ✅ Use desugaring (recommended)
// ✅ Use ThreeTenABP (alternative)
// ⚠️ Version check (last resort)
```

### 2. Storage Format

```kotlin
// ✅ Store as UTC timestamp (Long)
val timestamp = Instant.now().toEpochMilli()

// ✅ Or ISO 8601 string
val isoString = Instant.now().toString()

// ❌ Never store formatted dates
```

### 3. Display Format

```kotlin
// ✅ Use user's locale
val formatter = DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM)
    .withLocale(Locale.getDefault())

// ✅ Respect system preferences (12h/24h)
val is24Hour = DateFormat.is24HourFormat(context)
```

### 4. Timezone Handling

```kotlin
// ✅ Store in UTC
val utc = Instant.now()

// ✅ Display in user's timezone
val userZone = ZoneId.systemDefault()
val userTime = utc.atZone(userZone)
```

### 5. Thread Safety

```kotlin
// ✅ java.time is thread-safe
val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd")  // Reusable

// ❌ SimpleDateFormat is NOT thread-safe
val sdf = SimpleDateFormat("yyyy-MM-dd", Locale.US)  // Create new each time
```

## Recommended Approach

### For All New Projects

```kotlin
// 1. Enable desugaring
android {
    compileOptions {
        isCoreLibraryDesugaringEnabled = true
    }
}

dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")
}

// 2. Use java.time everywhere
import java.time.*
import java.time.format.DateTimeFormatter

// 3. Store as Instant or epoch millis
@Entity
data class Event(
    val id: String,
    val timestamp: Long  // or Instant with TypeConverter
)

// 4. Format for display
fun formatForUser(timestamp: Long): String {
    val instant = Instant.ofEpochMilli(timestamp)
    val zoned = instant.atZone(ZoneId.systemDefault())
    val formatter = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM)
    return zoned.format(formatter)
}
```

## Quick Reference

### Recommended Classes

| Use Case | Class | Example |
|----------|-------|---------|
| Timestamp | `Instant` | `Instant.now()` |
| Date only | `LocalDate` | `LocalDate.of(2025, 10, 20)` |
| Time only | `LocalTime` | `LocalTime.of(14, 30)` |
| Date + Time | `LocalDateTime` | `LocalDateTime.now()` |
| With timezone | `ZonedDateTime` | `ZonedDateTime.now(ZoneId.of("UTC"))` |
| Duration | `Duration` | `Duration.ofHours(2)` |
| Period | `Period` | `Period.ofDays(7)` |

### Common Operations

```kotlin
// Current time
Instant.now()
LocalDate.now()
LocalDateTime.now()

// Create specific date
LocalDate.of(2025, 10, 20)
LocalDateTime.of(2025, 10, 20, 14, 30)

// Add/subtract
date.plusDays(7)
date.minusMonths(1)

// Format
date.format(DateTimeFormatter.ofPattern("dd MMM yyyy"))

// Parse
LocalDate.parse("2025-10-20")
Instant.parse("2025-10-20T17:30:45Z")

// Convert
instant.toEpochMilli()  // To timestamp
Instant.ofEpochMilli(timestamp)  // From timestamp
```

## Resources

- [java.time Package](https://developer.android.com/reference/java/time/package-summary)
- [Core Library Desugaring](https://developer.android.com/studio/write/java8-support#library-desugaring)
- [ThreeTenABP](https://github.com/JakeWharton/ThreeTenABP)
- [DateUtils](https://developer.android.com/reference/android/text/format/DateUtils)
- [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601)
- [Time Zone Database](https://www.iana.org/time-zones)

