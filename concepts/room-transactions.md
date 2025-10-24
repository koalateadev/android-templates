# Room Transactions Deep Dive

## Overview

Understanding Room transactions helps you maintain data integrity and optimize database operations.

## ACID Properties

Room transactions guarantee ACID properties:

### Atomicity
All operations succeed or all fail - no partial updates

```kotlin
@Transaction
suspend fun transferMoney(fromAccount: String, toAccount: String, amount: Double) {
    // Both operations succeed or both fail
    accountDao.debit(fromAccount, amount)   // Step 1
    accountDao.credit(toAccount, amount)    // Step 2
    
    // If Step 2 fails, Step 1 is rolled back
}
```

### Consistency
Database moves from one valid state to another

```kotlin
@Transaction
suspend fun updateUserAndPosts(user: User, posts: List<Post>) {
    userDao.update(user)
    postDao.insertAll(posts)
    // Database constraints maintained throughout
}
```

### Isolation
Concurrent transactions don't interfere

```kotlin
// Transaction 1
@Transaction
suspend fun transaction1() {
    val balance = accountDao.getBalance("account1")
    accountDao.setBalance("account1", balance + 100)
}

// Transaction 2 (concurrent)
@Transaction
suspend fun transaction2() {
    val balance = accountDao.getBalance("account1")
    accountDao.setBalance("account1", balance - 50)
}

// Room ensures correct final balance
```

### Durability
Committed changes persist even after crash

## How @Transaction Works

### Under the Hood

```kotlin
@Transaction
suspend fun complexOperation() {
    // Room executes:
    database.beginTransaction()
    try {
        // Your operations here
        operationA()
        operationB()
        operationC()
        
        database.setTransactionSuccessful()
    } finally {
        database.endTransaction()
    }
}
```

### Automatic Rollback

```kotlin
@Transaction
suspend fun safeUpdate() {
    userDao.insert(user1)  // ✅ Inserted
    userDao.insert(user2)  // ❌ Throws exception (duplicate ID)
    userDao.insert(user3)  // Never executed
    
    // Transaction rolls back - user1 insertion reverted
}
```

## Transaction Scope

### DAO Methods

```kotlin
@Dao
interface UserDao {
    
    // Automatically wrapped in transaction
    @Insert
    suspend fun insert(user: User)
    
    @Insert
    suspend fun insertAll(users: List<User>)  // Single transaction
    
    @Update
    suspend fun update(user: User)
    
    @Delete
    suspend fun delete(user: User)
    
    // Query - read-only, no transaction
    @Query("SELECT * FROM users")
    suspend fun getAllUsers(): List<User>
    
    // Custom transaction
    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserWithPosts(userId: String): UserWithPosts
}
```

### Multiple Operations

```kotlin
@Dao
interface TransactionDao {
    
    @Insert
    suspend fun insertUser(user: User): Long
    
    @Insert
    suspend fun insertPosts(posts: List<Post>)
    
    @Query("DELETE FROM posts WHERE user_id = :userId")
    suspend fun deleteUserPosts(userId: String)
    
    // Wrap multiple operations in transaction
    @Transaction
    suspend fun createUserWithPosts(user: User, posts: List<Post>) {
        val userId = insertUser(user)
        val userPosts = posts.map { it.copy(userId = userId) }
        insertPosts(userPosts)
    }
    
    @Transaction
    suspend fun deleteUserAndPosts(userId: String) {
        deleteUserPosts(userId)
        deleteUser(userId)
    }
    
    @Query("DELETE FROM users WHERE id = :userId")
    suspend fun deleteUser(userId: String)
}
```

## Conflict Strategies

### OnConflictStrategy

```kotlin
@Dao
interface UserDao {
    
    // ABORT - Throw exception (default)
    @Insert(onConflict = OnConflictStrategy.ABORT)
    suspend fun insertAbort(user: User)
    
    // REPLACE - Replace existing row
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertReplace(user: User)  // Acts like UPSERT
    
    // IGNORE - Ignore new row
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertIgnore(user: User)
}
```

### How Conflicts Work

```kotlin
// Database has: User(id=1, name="John", email="john@old.com")

// ABORT
insert(User(id=1, name="John", email="john@new.com"))
// Throws exception, nothing changed

// REPLACE
insertReplace(User(id=1, name="John", email="john@new.com"))
// Replaces: User(id=1, name="John", email="john@new.com")

// IGNORE
insertIgnore(User(id=1, name="John", email="john@new.com"))
// Ignores insert, keeps: User(id=1, name="John", email="john@old.com")
```

## Write-Ahead Logging (WAL)

### What is WAL?

```
Without WAL:
    Write → Database file locked → Readers blocked

With WAL (default in Room):
    Write → WAL file → Readers can still read
    Background checkpoint → Merge to database
```

### Enable/Disable WAL

```kotlin
Room.databaseBuilder(context, AppDatabase::class.java, "db")
    .setJournalMode(RoomDatabase.JournalMode.WRITE_AHEAD_LOGGING)  // Default
    .build()

// Disable WAL (not recommended)
Room.databaseBuilder(context, AppDatabase::class.java, "db")
    .setJournalMode(RoomDatabase.JournalMode.TRUNCATE)
    .build()
```

### WAL Benefits

✅ **Concurrent reads and writes**  
✅ **Better performance** for most cases  
✅ **Reduced lock contention**

### WAL Drawbacks

⚠️ **Larger database size** (temporary)  
⚠️ **More complex** recovery  

## Nested Transactions

### How Nesting Works

```kotlin
@Dao
interface NestedDao {
    
    @Transaction
    suspend fun outerTransaction() {
        operationA()
        innerTransaction()  // Nested
        operationB()
    }
    
    @Transaction
    suspend fun innerTransaction() {
        operationC()
        operationD()
    }
}

// Room handles nesting:
// - Single SQLite transaction
// - Inner @Transaction is no-op
// - All succeed or all fail
```

## Performance Considerations

### Batch Operations

```kotlin
// ❌ Slow - Individual transactions
suspend fun insertOneByOne(users: List<User>) {
    users.forEach { user ->
        userDao.insert(user)  // New transaction each time
    }
}

// ✅ Fast - Single transaction
suspend fun insertBatch(users: List<User>) {
    userDao.insertAll(users)  // One transaction for all
}
```

### Transaction Size

```kotlin
// ❌ Bad - Huge transaction
@Transaction
suspend fun processMillionRecords() {
    // Locks database for extended time
    // High memory usage
    millionRecords.forEach { insert(it) }
}

// ✅ Good - Chunked transactions
suspend fun processInChunks(records: List<Record>) {
    records.chunked(1000).forEach { chunk ->
        processChunk(chunk)  // Separate transaction per 1000 items
    }
}

@Transaction
suspend fun processChunk(chunk: List<Record>) {
    recordDao.insertAll(chunk)
}
```

## Observing Changes

### Flow Updates During Transactions

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun observeUsers(): Flow<List<User>>
    
    @Transaction
    suspend fun updateMultiple(users: List<User>) {
        users.forEach { update(it) }
        // Flow emits ONCE after transaction completes
        // Not after each individual update
    }
}
```

## Common Patterns

### Upsert (Update or Insert)

```kotlin
@Dao
interface UserDao {
    @Upsert
    suspend fun upsert(user: User)
    
    // Equivalent to:
    // @Insert(onConflict = OnConflictStrategy.REPLACE)
    // suspend fun upsert(user: User)
}
```

### Delete and Insert

```kotlin
@Dao
interface DataDao {
    
    @Query("DELETE FROM users WHERE category = :category")
    suspend fun deleteByCategory(category: String)
    
    @Insert
    suspend fun insertAll(users: List<User>)
    
    @Transaction
    suspend fun replaceCategory(category: String, users: List<User>) {
        deleteByCategory(category)
        insertAll(users)
    }
}
```

### Conditional Updates

```kotlin
@Dao
interface AccountDao {
    
    @Query("UPDATE accounts SET balance = balance - :amount WHERE id = :accountId AND balance >= :amount")
    suspend fun debitIfSufficient(accountId: String, amount: Double): Int
    
    @Transaction
    suspend fun transferIfPossible(
        fromAccount: String,
        toAccount: String,
        amount: Double
    ): Boolean {
        val debited = debitIfSufficient(fromAccount, amount)
        
        return if (debited > 0) {
            credit(toAccount, amount)
            true
        } else {
            false  // Transaction rolled back
        }
    }
    
    @Query("UPDATE accounts SET balance = balance + :amount WHERE id = :accountId")
    suspend fun credit(accountId: String, amount: Double)
}
```

## Error Handling

### Transaction Failures

```kotlin
suspend fun safeTransaction() {
    try {
        database.withTransaction {
            userDao.insert(user)
            postDao.insertAll(posts)
        }
    } catch (e: SQLiteException) {
        // Transaction rolled back
        Timber.e(e, "Transaction failed")
    }
}
```

## Repository with Transactions

```kotlin
class UserRepository(
    private val database: AppDatabase,
    private val userDao: UserDao,
    private val postDao: PostDao
) {
    
    suspend fun createUserWithPosts(
        user: User,
        posts: List<Post>
    ): Result<Unit> {
        return try {
            database.withTransaction {
                val userId = userDao.insert(user)
                val userPosts = posts.map { it.copy(userId = userId) }
                postDao.insertAll(userPosts)
            }
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    suspend fun deleteUserCompletely(userId: String): Result<Unit> {
        return try {
            database.withTransaction {
                postDao.deleteByUserId(userId)
                userDao.deleteById(userId)
            }
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

## Best Practices

1. ✅ Use `@Transaction` for multi-step operations
2. ✅ Keep transactions short
3. ✅ Batch inserts when possible
4. ✅ Use appropriate conflict strategies
5. ✅ Handle exceptions properly
6. ✅ Don't nest transactions unnecessarily
7. ✅ Use WAL mode (default)
8. ✅ Chunk large operations
9. ✅ Test transaction rollback scenarios
10. ✅ Monitor transaction performance

## Common Pitfalls

### ❌ Forgetting @Transaction

```kotlin
// Bad - Two separate transactions
suspend fun updateUserAndPosts(user: User, posts: List<Post>) {
    userDao.update(user)  // Transaction 1
    postDao.updateAll(posts)  // Transaction 2
    // If second fails, first still committed!
}

// Good - Single transaction
@Transaction
suspend fun updateUserAndPosts(user: User, posts: List<Post>) {
    userDao.update(user)
    postDao.updateAll(posts)
    // Both succeed or both fail
}
```

### ❌ Long-Running Transactions

```kotlin
// Bad - Locks database too long
@Transaction
suspend fun processAndUpdate(users: List<User>) {
    users.forEach { user ->
        val processed = heavyProcessing(user)  // Expensive!
        userDao.update(processed)
    }
}

// Good - Process outside transaction
suspend fun processAndUpdate(users: List<User>) {
    val processed = users.map { heavyProcessing(it) }
    
    @Transaction
    userDao.updateAll(processed)  // Quick transaction
}
```

## Summary

### Key Takeaways

1. **Transactions Ensure Integrity**
   - All or nothing execution
   - Automatic rollback on failure

2. **@Transaction Annotation**
   - Wraps operations in database transaction
   - Works with suspend functions
   - Handles nesting automatically

3. **Conflict Strategies**
   - ABORT: Throw exception (default)
   - REPLACE: Overwrite existing
   - IGNORE: Keep existing

4. **WAL Mode**
   - Allows concurrent reads/writes
   - Default in Room
   - Better performance

5. **Performance**
   - Batch operations when possible
   - Keep transactions short
   - Chunk large datasets

## Resources

- [Room Transactions](https://developer.android.com/training/data-storage/room/accessing-data#query-multiple-tables)
- [SQLite Transactions](https://www.sqlite.org/lang_transaction.html)
- [Write-Ahead Logging](https://www.sqlite.org/wal.html)
- [Room Performance](https://developer.android.com/topic/performance/sqlite-performance-best-practices)

