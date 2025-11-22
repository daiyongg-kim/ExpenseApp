# SecureExpense - Technical Architecture

**Version**: 1.0
**Architecture Style**: Clean Architecture + MVVM
**Language**: Kotlin 100%

---

## 📐 Architecture Overview

SecureExpense follows **Clean Architecture** principles with clear separation of concerns across three main layers:

```
┌─────────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                       │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Composables │◄─│  ViewModels  │◄─│  UI States   │       │
│  └─────────────┘  └──────────────┘  └──────────────┘       │
│                           │                                  │
└───────────────────────────┼──────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      DOMAIN LAYER                            │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Use Cases  │  │  Entities    │  │ Repositories │       │
│  │             │  │  (Models)    │  │ (Interfaces) │       │
│  └─────────────┘  └──────────────┘  └──────────────┘       │
│                           │                                  │
└───────────────────────────┼──────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                       DATA LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Repositories │  │  Data Sources│  │   Mappers    │      │
│  │     (Impl)   │  │ (Room/API)   │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

---

## 🏛️ Layer Details

### 1. Presentation Layer

**Responsibility**: UI and user interaction

**Components**:
- **Composables**: Jetpack Compose UI components
- **ViewModels**: State management and business logic coordination
- **UI States**: Sealed classes representing screen states
- **Navigation**: Navigation graph and routing

**Key Principles**:
- ViewModels expose `StateFlow` for reactive UI
- No direct dependency on data layer (depends on domain)
- Single source of truth for UI state
- Proper lifecycle management

**Example Structure**:
```kotlin
// presentation/screens/dashboard/DashboardViewModel.kt
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val getMonthlySpendingUseCase: GetMonthlySpendingUseCase,
    // Other use cases...
) : ViewModel() {

    // State exposed as StateFlow
    val dashboardState: StateFlow<DashboardState> = combine(
        getMonthlySpendingUseCase(),
        // Other flows...
    ) { spending, ... ->
        DashboardState.Success(spending, ...)
    }.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5000),
        DashboardState.Loading
    )

    // User actions
    fun refresh() { /* ... */ }
}
```

---

### 2. Domain Layer

**Responsibility**: Business logic and rules

**Components**:
- **Entities (Models)**: Core business objects (Transaction, Category, Budget)
- **Use Cases**: Single-purpose business operations
- **Repository Interfaces**: Contracts for data access
- **Business Rules**: Validation, calculations, domain logic

**Key Principles**:
- No Android dependencies (pure Kotlin)
- Single Responsibility Principle for use cases
- Interface segregation for repositories
- Domain models reflect business concepts

**Example Structure**:
```kotlin
// domain/model/Transaction.kt
data class Transaction(
    val id: Long = 0,
    val amount: Double,
    val category: Category,
    val currency: String,
    val description: String,
    val date: LocalDate
) {
    // Business logic
    val displayAmount: String
        get() = formatCurrency(amount, currency)

    fun isExpense(): Boolean = amount > 0
}

// domain/usecase/GetMonthlySpendingUseCase.kt
class GetMonthlySpendingUseCase @Inject constructor(
    private val repository: TransactionRepository
) {
    operator fun invoke(year: Int, month: Int): Flow<Double> {
        return repository.getTransactionsByMonth(year, month)
            .map { transactions ->
                transactions.sumOf { it.amount }
            }
    }
}

// domain/repository/TransactionRepository.kt (Interface)
interface TransactionRepository {
    fun getAllTransactions(): Flow<List<Transaction>>
    suspend fun addTransaction(transaction: Transaction): Result<Long>
    // Other methods...
}
```

---

### 3. Data Layer

**Responsibility**: Data access and storage

**Components**:
- **Repository Implementations**: Concrete implementations of domain interfaces
- **Data Sources**: Room DAOs, Retrofit APIs
- **DTOs/Entities**: Database and network models
- **Mappers**: Convert between data and domain models
- **Cache Management**: In-memory and persistent caching

**Key Principles**:
- Repository pattern for data abstraction
- Mapper pattern for model conversion
- Error handling with Result wrapper
- Caching strategy for offline support

**Example Structure**:
```kotlin
// data/repository/TransactionRepositoryImpl.kt
class TransactionRepositoryImpl @Inject constructor(
    private val transactionDao: TransactionDao,
    private val mapper: TransactionMapper,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) : TransactionRepository {

    override fun getAllTransactions(): Flow<List<Transaction>> {
        return transactionDao.getAllTransactions()
            .map { entities ->
                entities.map { mapper.toDomain(it) }
            }
            .flowOn(dispatcher)
    }

    override suspend fun addTransaction(transaction: Transaction): Result<Long> {
        return withContext(dispatcher) {
            try {
                val entity = mapper.toEntity(transaction)
                val id = transactionDao.insertTransaction(entity)
                Result.success(id)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }
}

// data/local/entity/TransactionEntity.kt
@Entity(tableName = "transactions")
data class TransactionEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val amount: Double,
    val categoryId: Long,
    val currency: String,
    val description: String,
    val date: Long // Unix timestamp
)

// data/mapper/TransactionMapper.kt
class TransactionMapper @Inject constructor() {
    fun toDomain(entity: TransactionEntity, category: CategoryEntity): Transaction {
        return Transaction(
            id = entity.id,
            amount = entity.amount,
            category = categoryMapper.toDomain(category),
            currency = entity.currency,
            description = entity.description,
            date = Instant.ofEpochMilli(entity.date).atZone(ZoneId.systemDefault()).toLocalDate()
        )
    }

    fun toEntity(domain: Transaction): TransactionEntity {
        return TransactionEntity(
            id = domain.id,
            amount = domain.amount,
            categoryId = domain.category.id,
            currency = domain.currency,
            description = domain.description,
            date = domain.date.atStartOfDay(ZoneId.systemDefault()).toInstant().toEpochMilli()
        )
    }
}
```

---

## 🔄 Data Flow Patterns

### Pattern 1: User Action → UI Update

```
User Tap Button
       │
       ▼
  Composable calls ViewModel method
       │
       ▼
  ViewModel calls UseCase
       │
       ▼
  UseCase calls Repository
       │
       ▼
  Repository queries Database/API
       │
       ▼
  Data flows back through layers
       │
       ▼
  StateFlow emits new state
       │
       ▼
  Composable recomposes with new state
```

**Example**:
```kotlin
// 1. User taps "Add Transaction" button
@Composable
fun TransactionScreen(viewModel: TransactionViewModel = hiltViewModel()) {
    Button(onClick = {
        viewModel.addTransaction(transaction) // 2. Call ViewModel
    }) {
        Text("Add")
    }
}

// 3. ViewModel calls UseCase
@HiltViewModel
class TransactionViewModel @Inject constructor(
    private val addTransactionUseCase: AddTransactionUseCase
) : ViewModel() {
    fun addTransaction(transaction: Transaction) {
        viewModelScope.launch {
            addTransactionUseCase(transaction) // 4. Call UseCase
        }
    }
}

// 5. UseCase calls Repository
class AddTransactionUseCase @Inject constructor(
    private val repository: TransactionRepository
) {
    suspend operator fun invoke(transaction: Transaction): Result<Long> {
        return repository.addTransaction(transaction) // 6. Call Repository
    }
}

// 7. Repository saves to database
class TransactionRepositoryImpl @Inject constructor(
    private val dao: TransactionDao
) : TransactionRepository {
    override suspend fun addTransaction(transaction: Transaction): Result<Long> {
        return try {
            val id = dao.insertTransaction(mapper.toEntity(transaction))
            Result.success(id)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

---

### Pattern 2: Reactive Data Streams (Flow)

```
Database Insert
       │
       ▼
  Flow emits new data
       │
       ▼
  Repository transforms data
       │
       ▼
  UseCase applies business logic
       │
       ▼
  ViewModel combines multiple Flows
       │
       ▼
  StateFlow updates UI State
       │
       ▼
  Composable automatically recomposes
```

**Example**:
```kotlin
// DAO emits Flow
@Dao
interface TransactionDao {
    @Query("SELECT * FROM transactions ORDER BY date DESC")
    fun getAllTransactions(): Flow<List<TransactionEntity>>
}

// Repository transforms to domain models
class TransactionRepositoryImpl @Inject constructor(
    private val dao: TransactionDao,
    private val mapper: TransactionMapper
) : TransactionRepository {
    override fun getAllTransactions(): Flow<List<Transaction>> {
        return dao.getAllTransactions()
            .map { entities -> entities.map { mapper.toDomain(it) } }
    }
}

// UseCase applies business logic
class GetRecentTransactionsUseCase @Inject constructor(
    private val repository: TransactionRepository
) {
    operator fun invoke(limit: Int): Flow<List<Transaction>> {
        return repository.getAllTransactions()
            .map { it.take(limit) }
    }
}

// ViewModel combines flows
@HiltViewModel
class DashboardViewModel @Inject constructor(
    getRecentTransactionsUseCase: GetRecentTransactionsUseCase
) : ViewModel() {
    val recentTransactions: StateFlow<List<Transaction>> =
        getRecentTransactionsUseCase(5)
            .stateIn(
                viewModelScope,
                SharingStarted.WhileSubscribed(5000),
                emptyList()
            )
}

// Composable collects state
@Composable
fun DashboardScreen(viewModel: DashboardViewModel = hiltViewModel()) {
    val transactions by viewModel.recentTransactions.collectAsState()

    LazyColumn {
        items(transactions) { transaction ->
            TransactionItem(transaction)
        }
    }
}
```

---

## 🔐 Security Architecture

### Security Layers

```
┌─────────────────────────────────────────────────────────┐
│                  Authentication Layer                    │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Biometric + PIN ──► Keystore ──► Session Token │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Encryption Layer                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │  SQLCipher (DB) + EncryptedSharedPreferences    │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    Network Layer                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  TLS 1.3 + Certificate Pinning + No Cleartext   │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Security Implementation

**1. Authentication Flow**:
```kotlin
// SecurityManager coordinates all security
class SecurityManager @Inject constructor(
    private val biometricAuthenticator: BiometricAuthenticator,
    private val pinManager: PINManager,
    private val keyStoreManager: KeyStoreManager
) {
    fun authenticate(
        activity: FragmentActivity,
        onSuccess: () -> Unit,
        onError: (String) -> Unit
    ) {
        if (biometricAuthenticator.isAvailable()) {
            biometricAuthenticator.authenticate(activity, onSuccess, onError)
        } else {
            // Fallback to PIN
            pinManager.requestPIN(onSuccess, onError)
        }
    }

    fun createSession(): SessionToken {
        return keyStoreManager.generateSessionToken()
    }
}
```

**2. Data Encryption**:
```kotlin
// Database encryption with SQLCipher
@Provides
@Singleton
fun provideDatabase(
    @ApplicationContext context: Context,
    securityManager: SecurityManager
): ExpenseDatabase {
    val passphrase = securityManager.getDatabasePassphrase()
    val factory = SupportFactory(passphrase)

    return Room.databaseBuilder(context, ExpenseDatabase::class.java, "secure_expense.db")
        .openHelperFactory(factory)
        .build()
}

// Sensitive preferences encryption
class SecurePreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    private val encryptedPrefs = EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun saveString(key: String, value: String) {
        encryptedPrefs.edit().putString(key, value).apply()
    }

    fun getString(key: String, default: String? = null): String? {
        return encryptedPrefs.getString(key, default)
    }
}
```

**3. Network Security**:
```kotlin
// Certificate pinning with OkHttp
@Provides
@Singleton
fun provideOkHttpClient(): OkHttpClient {
    val certificatePinner = CertificatePinner.Builder()
        .add("api.exchangerate-api.com", "sha256/AAAAAAA...") // Real hash
        .build()

    return OkHttpClient.Builder()
        .certificatePinner(certificatePinner)
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
            else HttpLoggingInterceptor.Level.NONE
        })
        .build()
}
```

---

## 💉 Dependency Injection Architecture

### Hilt Module Structure

```
┌─────────────────────────────────────────────────────────┐
│                    Application Module                    │
│                   @InstallIn(SingletonComponent)         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  - Database (Room)                              │    │
│  │  - Network (Retrofit, OkHttp)                   │    │
│  │  - Security (Keystore, Biometric)               │    │
│  │  - WorkManager                                  │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    Repository Module                     │
│                   @InstallIn(SingletonComponent)         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  @Binds Repository Interfaces → Implementations │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    ViewModel Module                      │
│                   @HiltViewModel (auto-injected)         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  ViewModels with UseCase dependencies          │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Module Examples

**Database Module**:
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(
        @ApplicationContext context: Context,
        securityManager: SecurityManager
    ): ExpenseDatabase {
        val passphrase = securityManager.getDatabasePassphrase()
        val factory = SupportFactory(passphrase)

        return Room.databaseBuilder(context, ExpenseDatabase::class.java, "expense_db")
            .openHelperFactory(factory)
            .build()
    }

    @Provides
    fun provideTransactionDao(database: ExpenseDatabase): TransactionDao {
        return database.transactionDao()
    }

    @Provides
    fun provideCategoryDao(database: ExpenseDatabase): CategoryDao {
        return database.categoryDao()
    }
}
```

**Repository Module**:
```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindTransactionRepository(
        impl: TransactionRepositoryImpl
    ): TransactionRepository

    @Binds
    @Singleton
    abstract fun bindCategoryRepository(
        impl: CategoryRepositoryImpl
    ): CategoryRepository

    @Binds
    @Singleton
    abstract fun bindExchangeRateRepository(
        impl: ExchangeRateRepositoryImpl
    ): ExchangeRateRepository
}
```

**Network Module**:
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        // Certificate pinning, logging, timeouts
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.exchangerate-api.com/v4/")
            .client(okHttpClient)
            .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
            .build()
    }

    @Provides
    @Singleton
    fun provideExchangeRateApi(retrofit: Retrofit): ExchangeRateApi {
        return retrofit.create(ExchangeRateApi::class.java)
    }
}
```

---

## 🗄️ Database Schema

### Entity Relationship Diagram

```
┌─────────────────┐           ┌─────────────────┐
│  transactions   │           │   categories    │
├─────────────────┤           ├─────────────────┤
│ id (PK)         │◄──────────┤ id (PK)         │
│ amount          │ categoryId│ name            │
│ categoryId (FK) │           │ icon            │
│ currency        │           │ color           │
│ description     │           │ budget          │
│ notes           │           └─────────────────┘
│ date            │
│ createdAt       │
│ updatedAt       │
└─────────────────┘

┌─────────────────┐
│  exchange_rates │
├─────────────────┤
│ id (PK)         │
│ fromCurrency    │
│ toCurrency      │
│ rate            │
│ timestamp       │
└─────────────────┘

┌─────────────────┐
│  budgets        │
├─────────────────┤
│ id (PK)         │
│ categoryId (FK) │
│ amount          │
│ month           │
│ year            │
└─────────────────┘
```

### Database Migrations

```kotlin
// Migration strategy
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            "ALTER TABLE transactions ADD COLUMN notes TEXT"
        )
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            """
            CREATE TABLE IF NOT EXISTS exchange_rates (
                id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
                fromCurrency TEXT NOT NULL,
                toCurrency TEXT NOT NULL,
                rate REAL NOT NULL,
                timestamp INTEGER NOT NULL
            )
            """.trimIndent()
        )
    }
}

@Database(
    entities = [TransactionEntity::class, CategoryEntity::class, ExchangeRateEntity::class],
    version = 3
)
abstract class ExpenseDatabase : RoomDatabase() {
    abstract fun transactionDao(): TransactionDao
    abstract fun categoryDao(): CategoryDao
    abstract fun exchangeRateDao(): ExchangeRateDao
}
```

---

## 🎨 UI Architecture (Jetpack Compose)

### Component Hierarchy

```
SecureExpenseApp
├── MainScaffold
│   ├── TopBar (per screen)
│   ├── BottomNavigationBar
│   └── NavHost
│       ├── DashboardScreen
│       │   ├── SummaryCard
│       │   ├── BudgetProgressCard
│       │   ├── CategoryPieChart
│       │   └── TransactionList
│       ├── TransactionsScreen
│       │   ├── SearchBar
│       │   └── TransactionList
│       │       └── SwipeableTransactionItem
│       ├── BudgetScreen
│       ├── AnalyticsScreen
│       └── SettingsScreen
```

### State Management Pattern

```kotlin
// Unidirectional Data Flow (UDF)

@Composable
fun TransactionsScreen(
    viewModel: TransactionViewModel = hiltViewModel()
) {
    // 1. Collect state
    val state by viewModel.transactionsState.collectAsState()

    // 2. Render UI based on state
    when (state) {
        is TransactionsState.Loading -> LoadingIndicator()
        is TransactionsState.Success -> TransactionList(
            transactions = state.transactions,
            onTransactionClick = { transaction ->
                // 3. User event triggers action
                viewModel.onTransactionClick(transaction)
            },
            onTransactionDelete = { transaction ->
                viewModel.deleteTransaction(transaction)
            }
        )
        is TransactionsState.Error -> ErrorState(state.message)
    }
}

// ViewModel manages state
@HiltViewModel
class TransactionViewModel @Inject constructor(
    private val getAllTransactionsUseCase: GetAllTransactionsUseCase
) : ViewModel() {

    // State as StateFlow
    val transactionsState: StateFlow<TransactionsState> =
        getAllTransactionsUseCase()
            .map { TransactionsState.Success(it) }
            .catch { emit(TransactionsState.Error(it.message ?: "Unknown error")) }
            .stateIn(
                viewModelScope,
                SharingStarted.WhileSubscribed(5000),
                TransactionsState.Loading
            )

    // Actions
    fun onTransactionClick(transaction: Transaction) {
        // Handle click
    }

    fun deleteTransaction(transaction: Transaction) {
        viewModelScope.launch {
            deleteTransactionUseCase(transaction)
        }
    }
}

// State sealed class
sealed interface TransactionsState {
    data object Loading : TransactionsState
    data class Success(val transactions: List<Transaction>) : TransactionsState
    data class Error(val message: String) : TransactionsState
}
```

---

## 🧪 Testing Architecture

### Testing Pyramid

```
              ┌───────────────┐
              │  E2E Tests    │
              │   (10%)       │  ← Compose UI Tests
              └───────────────┘
           ┌──────────────────────┐
           │  Integration Tests   │
           │      (20%)           │  ← Repository + DAO Tests
           └──────────────────────┘
        ┌────────────────────────────┐
        │      Unit Tests            │
        │        (70%)               │  ← ViewModel, UseCase, Mapper
        └────────────────────────────┘
```

### Test Structure

**Unit Tests** (70% of tests):
```kotlin
// ViewModelTest
@OptIn(ExperimentalCoroutinesApi::class)
class DashboardViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var getMonthlySpendingUseCase: GetMonthlySpendingUseCase
    private lateinit var viewModel: DashboardViewModel

    @Before
    fun setup() {
        getMonthlySpendingUseCase = mockk()
    }

    @Test
    fun `dashboard loads successfully`() = runTest {
        // Given
        coEvery { getMonthlySpendingUseCase(any(), any()) } returns flowOf(1000.0)

        // When
        viewModel = DashboardViewModel(getMonthlySpendingUseCase)

        // Then
        viewModel.dashboardState.test {
            assertEquals(DashboardState.Loading, awaitItem())
            val success = awaitItem() as DashboardState.Success
            assertEquals(1000.0, success.monthlySpending)
        }
    }
}

// UseCaseTest
class GetMonthlySpendingUseCaseTest {
    private lateinit var repository: TransactionRepository
    private lateinit var useCase: GetMonthlySpendingUseCase

    @Before
    fun setup() {
        repository = mockk()
        useCase = GetMonthlySpendingUseCase(repository)
    }

    @Test
    fun `returns correct monthly spending`() = runTest {
        // Given
        val transactions = listOf(
            Transaction(amount = 50.0, /* ... */),
            Transaction(amount = 100.0, /* ... */)
        )
        coEvery { repository.getTransactionsByMonth(2024, 1) } returns flowOf(transactions)

        // When
        val result = useCase(2024, 1).first()

        // Then
        assertEquals(150.0, result)
    }
}
```

**Integration Tests** (20% of tests):
```kotlin
// RepositoryTest with real DAO
@RunWith(AndroidJUnit4::class)
class TransactionRepositoryTest {
    private lateinit var database: ExpenseDatabase
    private lateinit var repository: TransactionRepositoryImpl

    @Before
    fun setup() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = Room.inMemoryDatabaseBuilder(context, ExpenseDatabase::class.java)
            .allowMainThreadQueries()
            .build()
        repository = TransactionRepositoryImpl(database.transactionDao(), mapper)
    }

    @After
    fun teardown() {
        database.close()
    }

    @Test
    fun `add and retrieve transaction`() = runTest {
        // Given
        val transaction = Transaction(/* ... */)

        // When
        repository.addTransaction(transaction)

        // Then
        repository.getAllTransactions().test {
            val list = awaitItem()
            assertEquals(1, list.size)
            assertEquals(transaction.description, list[0].description)
        }
    }
}
```

**UI Tests** (10% of tests):
```kotlin
// Compose UI Test
@HiltAndroidTest
class TransactionScreenTest {
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun `transaction list displays transactions`() {
        // Setup test data
        composeRule.setContent {
            SecureExpenseTheme {
                TransactionsScreen()
            }
        }

        // Verify
        composeRule.onNodeWithText("Groceries").assertIsDisplayed()
        composeRule.onNodeWithText("$50.00").assertIsDisplayed()
    }

    @Test
    fun `swipe to delete removes transaction`() {
        composeRule.setContent {
            SecureExpenseTheme {
                TransactionsScreen()
            }
        }

        // Perform swipe
        composeRule.onNodeWithText("Groceries").performTouchInput {
            swipeLeft()
        }

        // Verify deletion
        composeRule.waitUntil(5000) {
            composeRule.onAllNodesWithText("Groceries").fetchSemanticsNodes().isEmpty()
        }
    }
}
```

---

## 📊 Performance Optimization Strategies

### 1. Database Optimization

**Indexes**:
```kotlin
@Entity(
    tableName = "transactions",
    indices = [
        Index(value = ["date"], name = "idx_transaction_date"),
        Index(value = ["categoryId"], name = "idx_transaction_category")
    ]
)
data class TransactionEntity(/* ... */)
```

**Query Optimization**:
```kotlin
// Use Flow for reactive queries (Room handles optimization)
@Query("SELECT * FROM transactions WHERE date >= :startDate AND date <= :endDate ORDER BY date DESC")
fun getTransactionsByDateRange(startDate: Long, endDate: Long): Flow<List<TransactionEntity>>

// Use LIMIT for pagination
@Query("SELECT * FROM transactions ORDER BY date DESC LIMIT :limit OFFSET :offset")
suspend fun getTransactionsPaginated(limit: Int, offset: Int): List<TransactionEntity>
```

### 2. UI Performance

**LazyColumn Optimization**:
```kotlin
@Composable
fun TransactionList(transactions: List<Transaction>) {
    LazyColumn {
        items(
            items = transactions,
            key = { it.id } // Stable keys for recomposition optimization
        ) { transaction ->
            TransactionItem(transaction)
        }
    }
}
```

**Remember & DerivedStateOf**:
```kotlin
@Composable
fun ExpensiveCalculation(data: List<Transaction>) {
    // Only recalculates when data changes
    val total by remember(data) {
        derivedStateOf { data.sumOf { it.amount } }
    }

    Text("Total: $$total")
}
```

### 3. Memory Management

**ViewModel Lifecycle**:
```kotlin
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val useCase: GetMonthlySpendingUseCase
) : ViewModel() {

    // StateFlow with WhileSubscribed(5s) stops flow when no collectors
    val state: StateFlow<DashboardState> = useCase()
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5000), // Stop after 5s of no collectors
            DashboardState.Loading
        )

    override fun onCleared() {
        super.onCleared()
        // Clean up resources
    }
}
```

**Image Loading (if used)**:
```kotlin
// Use Coil with caching
Image(
    painter = rememberAsyncImagePainter(
        ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .memoryCacheKey(imageUrl)
            .diskCacheKey(imageUrl)
            .build()
    ),
    contentDescription = null
)
```

---

## 🔄 Background Work Architecture

### WorkManager Integration

```kotlin
// ExchangeRateUpdateWorker
@HiltWorker
class ExchangeRateUpdateWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: ExchangeRateRepository
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            // Fetch latest rates
            val currencies = listOf("USD", "EUR", "GBP")
            currencies.forEach { currency ->
                repository.getExchangeRate("CAD", currency)
            }
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }
}

// Schedule periodic work
fun scheduleExchangeRateUpdate(context: Context) {
    val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .setRequiresBatteryNotLow(true)
        .build()

    val request = PeriodicWorkRequestBuilder<ExchangeRateUpdateWorker>(
        repeatInterval = 24,
        repeatIntervalTimeUnit = TimeUnit.HOURS
    )
        .setConstraints(constraints)
        .setBackoffCriteria(
            BackoffPolicy.EXPONENTIAL,
            PeriodicWorkRequest.MIN_BACKOFF_MILLIS,
            TimeUnit.MILLISECONDS
        )
        .build()

    WorkManager.getInstance(context)
        .enqueueUniquePeriodicWork(
            "exchange_rate_update",
            ExistingPeriodicWorkPolicy.KEEP,
            request
        )
}
```

---

## 📐 Design Patterns Used

### 1. Repository Pattern
**Purpose**: Abstract data sources
**Implementation**: Repository interfaces in domain, implementations in data layer

### 2. Mapper Pattern
**Purpose**: Convert between data and domain models
**Implementation**: Dedicated mapper classes with `toDomain()` and `toEntity()` methods

### 3. Use Case Pattern
**Purpose**: Single-purpose business operations
**Implementation**: One class per business operation (e.g., `GetMonthlySpendingUseCase`)

### 4. Observer Pattern
**Purpose**: Reactive UI updates
**Implementation**: Kotlin Flow and StateFlow

### 5. Factory Pattern
**Purpose**: Object creation
**Implementation**: Hilt dependency injection modules

### 6. Strategy Pattern
**Purpose**: Flexible authentication methods
**Implementation**: `BiometricAuthenticator` and `PINManager` with common interface

### 7. Singleton Pattern
**Purpose**: Single instance of expensive objects
**Implementation**: Hilt `@Singleton` scope for database, network clients

---

## 🚀 Deployment Architecture

### Build Variants

```kotlin
android {
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            isDebuggable = true
            isMinifyEnabled = false
        }

        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
    }

    flavorDimensions += "environment"
    productFlavors {
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
            versionNameSuffix = "-dev"
        }

        create("prod") {
            dimension = "environment"
        }
    }
}
```

### CI/CD Pipeline (GitHub Actions)

```yaml
name: Android CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
      - name: Run unit tests
        run: ./gradlew testDebugUnitTest
      - name: Generate coverage report
        run: ./gradlew jacocoTestReport
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v3
      - name: Build release APK
        run: ./gradlew assembleRelease
      - name: Upload APK
        uses: actions/upload-artifact@v3
        with:
          name: app-release.apk
          path: app/build/outputs/apk/release/app-release.apk
```

---

## 📚 Key Architectural Decisions

### 1. Why Clean Architecture?
- ✅ Clear separation of concerns
- ✅ Testability (each layer can be tested independently)
- ✅ Scalability (easy to add features without breaking existing code)
- ✅ Technology independence (easy to swap Room for SQL Delight, etc.)

### 2. Why Jetpack Compose?
- ✅ Modern declarative UI
- ✅ Less boilerplate than XML
- ✅ Better performance with recomposition
- ✅ Industry standard for new Android development

### 3. Why Kotlin Flow?
- ✅ Native Kotlin coroutines support
- ✅ Better type safety than LiveData
- ✅ Lifecycle-aware with `stateIn` and `collectAsState`
- ✅ Composable operators (map, filter, combine, etc.)

### 4. Why Hilt over Koin?
- ✅ Compile-time dependency injection (catches errors early)
- ✅ Better performance (no reflection)
- ✅ Official Google recommendation
- ✅ Better integration with Jetpack libraries

### 5. Why Room over Realm?
- ✅ Official Jetpack library
- ✅ Better Kotlin support (Flow, coroutines)
- ✅ SQLCipher integration for encryption
- ✅ More mature and stable

---

## 🎓 Learning Resources

### Architecture
- [Guide to app architecture (Google)](https://developer.android.com/topic/architecture)
- [Clean Architecture (Robert C. Martin)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

### Jetpack Compose
- [Compose Tutorial](https://developer.android.com/jetpack/compose/tutorial)
- [Thinking in Compose](https://developer.android.com/jetpack/compose/mental-model)

### Security
- [Android Security Best Practices](https://developer.android.com/topic/security/best-practices)
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)

---

**This architecture is designed to showcase senior-level Android development skills for RBC's requirements while maintaining production-ready quality and best practices.**
