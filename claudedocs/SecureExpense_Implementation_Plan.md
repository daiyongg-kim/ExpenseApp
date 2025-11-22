# SecureExpense - Implementation Plan

**Version**: 1.0
**Timeline**: 4 weeks part-time (20h/week) OR 2 weeks full-time (40h/week)
**Approach**: TDD + Clean Architecture + Iterative MVP

---

## 📅 Timeline Overview

### Option A: Part-Time (4 Weeks, 20h/week = 80 hours total)
- **Week 1**: Foundation & Architecture (20h)
- **Week 2**: Core Features (20h)
- **Week 3**: Security & UI Polish (20h)
- **Week 4**: Testing & Documentation (20h)

### Option B: Full-Time (2 Weeks, 40h/week = 80 hours total)
- **Week 1**: Foundation + Core Features (40h)
- **Week 2**: Security + Testing + Documentation (40h)

---

## 🎯 Phase-Based Implementation

This plan uses an **MVP-first** approach:
- **Phase 1**: Minimum Viable Product (60% of features)
- **Phase 2**: Enhancement & Polish (30% of features)
- **Phase 3**: Advanced Features (10% of features)

---

## Phase 1: Foundation & MVP (Week 1-2)

### Day 1-2: Project Setup & Architecture (8-10h)

#### Hour 1-2: Project Initialization
**Tasks**:
- [ ] Create new Android project in Android Studio
- [ ] Configure `build.gradle.kts` with dependencies
- [ ] Setup project structure (data/domain/presentation layers)
- [ ] Configure Git repository
- [ ] Add `.gitignore` (exclude `local.properties`, `*.apk`, etc.)

**Checklist**:
```bash
# Create project
- Package name: com.secureexpense
- Minimum SDK: 26
- Build configuration language: Kotlin DSL

# Initial commit
git init
git add .
git commit -m "Initial project setup"
git branch -M main
git remote add origin <your-repo-url>
git push -u origin main
```

**Dependencies to Add**:
```kotlin
// build.gradle.kts (Project level)
plugins {
    id("com.android.application") version "8.2.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.22" apply false
    id("com.google.dagger.hilt.android") version "2.50" apply false
}

// build.gradle.kts (App level)
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
    id("com.google.dagger.hilt.android")
    id("kotlinx-serialization")
}

dependencies {
    // Compose BOM
    implementation(platform("androidx.compose:compose-bom:2024.02.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")

    // Lifecycle
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")

    // Navigation
    implementation("androidx.navigation:navigation-compose:2.7.6")

    // Hilt
    implementation("com.google.dagger:hilt-android:2.50")
    kapt("com.google.dagger:hilt-compiler:2.50")
    implementation("androidx.hilt:hilt-navigation-compose:1.1.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")

    // Room
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")

    // Retrofit
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-kotlinx-serialization:2.9.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

    // Serialization
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")

    // Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.mockk:mockk:1.13.9")
    testImplementation("app.cash.turbine:turbine:1.0.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")

    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-tooling")
}
```

---

#### Hour 3-4: Data Layer Foundation
**Tasks**:
- [ ] Create Room entities (`Transaction`, `Category`, `Budget`)
- [ ] Create DAOs with Flow return types
- [ ] Setup Database with migrations strategy
- [ ] Write DAO unit tests

**Entity Definition**:
```kotlin
// data/local/entity/TransactionEntity.kt
@Entity(tableName = "transactions")
data class TransactionEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val amount: Double,
    val categoryId: Long,
    val currency: String,
    val description: String,
    val notes: String?,
    val date: Long, // Unix timestamp
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
)

@Entity(tableName = "categories")
data class CategoryEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val name: String,
    val icon: String, // Material icon name
    val color: String, // Hex color code
    val budget: Double = 0.0
)
```

**DAO with Flow**:
```kotlin
// data/local/dao/TransactionDao.kt
@Dao
interface TransactionDao {
    @Query("SELECT * FROM transactions ORDER BY date DESC")
    fun getAllTransactions(): Flow<List<TransactionEntity>>

    @Query("SELECT * FROM transactions WHERE date >= :startDate AND date <= :endDate")
    fun getTransactionsByDateRange(startDate: Long, endDate: Long): Flow<List<TransactionEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertTransaction(transaction: TransactionEntity): Long

    @Delete
    suspend fun deleteTransaction(transaction: TransactionEntity)

    @Query("SELECT SUM(amount) FROM transactions WHERE date >= :startDate AND date <= :endDate")
    fun getTotalSpending(startDate: Long, endDate: Long): Flow<Double?>
}
```

**Test Example**:
```kotlin
// test/data/local/dao/TransactionDaoTest.kt
@RunWith(AndroidJUnit4::class)
class TransactionDaoTest {
    private lateinit var database: ExpenseDatabase
    private lateinit var dao: TransactionDao

    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            ExpenseDatabase::class.java
        ).allowMainThreadQueries().build()

        dao = database.transactionDao()
    }

    @Test
    fun insertAndRetrieveTransaction() = runTest {
        // Given
        val transaction = TransactionEntity(
            amount = 50.0,
            categoryId = 1,
            currency = "CAD",
            description = "Groceries",
            date = System.currentTimeMillis()
        )

        // When
        dao.insertTransaction(transaction)

        // Then
        dao.getAllTransactions().test {
            val list = awaitItem()
            assertEquals(1, list.size)
            assertEquals("Groceries", list[0].description)
        }
    }
}
```

---

#### Hour 5-6: Domain Layer
**Tasks**:
- [ ] Create domain models (clean entities)
- [ ] Define repository interfaces
- [ ] Create use cases for core operations
- [ ] Write use case unit tests

**Domain Model**:
```kotlin
// domain/model/Transaction.kt
data class Transaction(
    val id: Long = 0,
    val amount: Double,
    val category: Category,
    val currency: String,
    val description: String,
    val notes: String? = null,
    val date: LocalDate,
    val createdAt: LocalDateTime = LocalDateTime.now()
) {
    val displayAmount: String
        get() = NumberFormat.getCurrencyInstance().apply {
            currency = Currency.getInstance(this@Transaction.currency)
        }.format(amount)
}

data class Category(
    val id: Long = 0,
    val name: String,
    val icon: String,
    val color: Color,
    val budget: Double = 0.0
)
```

**Repository Interface**:
```kotlin
// domain/repository/TransactionRepository.kt
interface TransactionRepository {
    fun getAllTransactions(): Flow<List<Transaction>>
    fun getTransactionsByMonth(year: Int, month: Int): Flow<List<Transaction>>
    suspend fun addTransaction(transaction: Transaction): Result<Long>
    suspend fun updateTransaction(transaction: Transaction): Result<Unit>
    suspend fun deleteTransaction(transaction: Transaction): Result<Unit>
    fun getMonthlySpending(year: Int, month: Int): Flow<Double>
}
```

**Use Case Example**:
```kotlin
// domain/usecase/GetMonthlySpendingUseCase.kt
class GetMonthlySpendingUseCase @Inject constructor(
    private val repository: TransactionRepository
) {
    operator fun invoke(year: Int, month: Int): Flow<MonthlySpending> {
        return repository.getMonthlySpending(year, month)
            .map { total ->
                MonthlySpending(
                    total = total,
                    year = year,
                    month = month
                )
            }
    }
}

// Test
class GetMonthlySpendingUseCaseTest {
    @Test
    fun `returns monthly spending total`() = runTest {
        // Given
        val repository = mockk<TransactionRepository>()
        coEvery { repository.getMonthlySpending(2024, 1) } returns flowOf(1500.0)

        val useCase = GetMonthlySpendingUseCase(repository)

        // When
        val result = useCase(2024, 1).first()

        // Then
        assertEquals(1500.0, result.total)
    }
}
```

---

#### Hour 7-8: Repository Implementation
**Tasks**:
- [ ] Implement repository with mapper pattern
- [ ] Add error handling with Result wrapper
- [ ] Write repository unit tests with MockK

**Repository Implementation**:
```kotlin
// data/repository/TransactionRepositoryImpl.kt
class TransactionRepositoryImpl @Inject constructor(
    private val transactionDao: TransactionDao,
    private val categoryDao: CategoryDao,
    private val transactionMapper: TransactionMapper
) : TransactionRepository {

    override fun getAllTransactions(): Flow<List<Transaction>> {
        return transactionDao.getAllTransactions()
            .map { entities -> entities.map { transactionMapper.toDomain(it) } }
            .catch { emit(emptyList()) }
    }

    override suspend fun addTransaction(transaction: Transaction): Result<Long> {
        return try {
            val entity = transactionMapper.toEntity(transaction)
            val id = transactionDao.insertTransaction(entity)
            Result.success(id)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

**Mapper**:
```kotlin
// data/mapper/TransactionMapper.kt
class TransactionMapper @Inject constructor(
    private val categoryMapper: CategoryMapper
) {
    fun toDomain(entity: TransactionEntity, category: CategoryEntity): Transaction {
        return Transaction(
            id = entity.id,
            amount = entity.amount,
            category = categoryMapper.toDomain(category),
            currency = entity.currency,
            description = entity.description,
            notes = entity.notes,
            date = Instant.ofEpochMilli(entity.date)
                .atZone(ZoneId.systemDefault())
                .toLocalDate()
        )
    }

    fun toEntity(domain: Transaction): TransactionEntity {
        return TransactionEntity(
            id = domain.id,
            amount = domain.amount,
            categoryId = domain.category.id,
            currency = domain.currency,
            description = domain.description,
            notes = domain.notes,
            date = domain.date.atStartOfDay(ZoneId.systemDefault()).toInstant().toEpochMilli()
        )
    }
}
```

---

#### Hour 9-10: Dependency Injection Setup
**Tasks**:
- [ ] Create Hilt Application class
- [ ] Setup DI modules (Database, Network, Repository)
- [ ] Configure navigation module

**Application Class**:
```kotlin
// SecureExpenseApp.kt
@HiltAndroidApp
class SecureExpenseApp : Application()
```

**Database Module**:
```kotlin
// di/DatabaseModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(
        @ApplicationContext context: Context
    ): ExpenseDatabase {
        return Room.databaseBuilder(
            context,
            ExpenseDatabase::class.java,
            "expense_database"
        )
        .addMigrations() // Add migrations as needed
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

// di/RepositoryModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindTransactionRepository(
        impl: TransactionRepositoryImpl
    ): TransactionRepository
}
```

**Checkpoint**: Compile and run tests. You should have ~15 passing unit tests at this point.

---

### Day 3-4: UI Foundation (8-10h)

#### Hour 1-2: Theme Setup
**Tasks**:
- [ ] Create Material 3 theme with RBC colors
- [ ] Define Typography scale
- [ ] Setup dynamic color support
- [ ] Create spacing and dimension tokens

**Theme Implementation**:
```kotlin
// presentation/theme/Color.kt
val md_theme_light_primary = Color(0xFF005EB8) // RBC Blue
val md_theme_light_onPrimary = Color(0xFFFFFFFF)
val md_theme_light_primaryContainer = Color(0xFFD6E3FF)
val md_theme_light_onPrimaryContainer = Color(0xFF001B3D)

val md_theme_light_secondary = Color(0xFFFFD400) // RBC Gold
// ... complete color scheme

// presentation/theme/Theme.kt
@Composable
fun SecureExpenseTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            if (darkTheme) dynamicDarkColorScheme(LocalContext.current)
            else dynamicLightColorScheme(LocalContext.current)
        }
        darkTheme -> darkColorScheme(
            primary = md_theme_dark_primary,
            secondary = md_theme_dark_secondary,
            // ...
        )
        else -> lightColorScheme(
            primary = md_theme_light_primary,
            secondary = md_theme_light_secondary,
            // ...
        )
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

---

#### Hour 3-5: Shared Components
**Tasks**:
- [ ] Create `ExpenseCard` component with variants
- [ ] Create `EmptyState` component
- [ ] Create `LoadingIndicator` component
- [ ] Create `ErrorState` component
- [ ] Write Compose UI tests for components

**ExpenseCard Component**:
```kotlin
// presentation/components/ExpenseCard.kt
@Composable
fun ExpenseCard(
    transaction: Transaction,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        onClick = onClick,
        modifier = modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surface
        )
    ) {
        Row(
            modifier = Modifier
                .padding(16.dp)
                .fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Row(
                horizontalArrangement = Arrangement.spacedBy(12.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                // Category Icon
                Surface(
                    shape = CircleShape,
                    color = transaction.category.color.copy(alpha = 0.1f),
                    modifier = Modifier.size(40.dp)
                ) {
                    Icon(
                        imageVector = Icons.Default.ShoppingCart, // Dynamic based on category
                        contentDescription = null,
                        tint = transaction.category.color,
                        modifier = Modifier.padding(8.dp)
                    )
                }

                // Description & Category
                Column {
                    Text(
                        text = transaction.description,
                        style = MaterialTheme.typography.bodyLarge,
                        fontWeight = FontWeight.Medium
                    )
                    Text(
                        text = transaction.category.name,
                        style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.onSurfaceVariant
                    )
                }
            }

            // Amount
            Text(
                text = transaction.displayAmount,
                style = MaterialTheme.typography.titleMedium,
                fontWeight = FontWeight.Bold,
                color = MaterialTheme.colorScheme.error
            )
        }
    }
}

// Preview
@Preview(showBackground = true)
@Composable
fun ExpenseCardPreview() {
    SecureExpenseTheme {
        ExpenseCard(
            transaction = Transaction(
                id = 1,
                amount = 50.0,
                category = Category(1, "Groceries", "shopping_cart", Color.Green),
                currency = "CAD",
                description = "Weekly groceries",
                date = LocalDate.now()
            ),
            onClick = {}
        )
    }
}
```

**Component Test**:
```kotlin
// androidTest/presentation/components/ExpenseCardTest.kt
@Test
fun expenseCard_displaysCorrectInformation() {
    val transaction = Transaction(/* test data */)

    composeTestRule.setContent {
        ExpenseCard(transaction = transaction, onClick = {})
    }

    composeTestRule.onNodeWithText("Groceries").assertIsDisplayed()
    composeTestRule.onNodeWithText("$50.00").assertIsDisplayed()
}
```

---

#### Hour 6-8: Navigation Setup
**Tasks**:
- [ ] Create navigation graph
- [ ] Setup bottom navigation
- [ ] Implement screen scaffolds
- [ ] Add navigation tests

**Navigation Setup**:
```kotlin
// presentation/navigation/NavGraph.kt
@Composable
fun SecureExpenseNavHost(
    navController: NavHostController,
    modifier: Modifier = Modifier
) {
    NavHost(
        navController = navController,
        startDestination = Screen.Dashboard.route,
        modifier = modifier
    ) {
        composable(Screen.Dashboard.route) {
            DashboardScreen(
                onNavigateToTransactions = {
                    navController.navigate(Screen.Transactions.route)
                }
            )
        }

        composable(Screen.Transactions.route) {
            TransactionsScreen(
                onTransactionClick = { transactionId ->
                    navController.navigate(Screen.TransactionDetail.createRoute(transactionId))
                }
            )
        }

        // ... other screens
    }
}

// Bottom Navigation
@Composable
fun MainScaffold() {
    val navController = rememberNavController()

    Scaffold(
        bottomBar = {
            NavigationBar {
                val items = listOf(
                    Screen.Dashboard,
                    Screen.Transactions,
                    Screen.Budget,
                    Screen.Analytics,
                    Screen.Settings
                )

                items.forEach { screen ->
                    NavigationBarItem(
                        icon = { Icon(screen.icon, contentDescription = screen.title) },
                        label = { Text(screen.title) },
                        selected = false, // TODO: track current route
                        onClick = { navController.navigate(screen.route) }
                    )
                }
            }
        }
    ) { paddingValues ->
        SecureExpenseNavHost(
            navController = navController,
            modifier = Modifier.padding(paddingValues)
        )
    }
}
```

---

### Day 5-6: Dashboard Screen (8-10h)

#### Hour 1-3: Dashboard ViewModel
**Tasks**:
- [ ] Create `DashboardState` sealed class
- [ ] Implement `DashboardViewModel` with Flow
- [ ] Write ViewModel tests with Turbine
- [ ] Test error scenarios

**Dashboard State**:
```kotlin
// presentation/screens/dashboard/DashboardState.kt
sealed interface DashboardState {
    data object Loading : DashboardState
    data class Success(
        val monthlySpending: Double,
        val budget: Double,
        val categoryBreakdown: List<CategorySpending>,
        val recentTransactions: List<Transaction>
    ) : DashboardState
    data class Error(val message: String) : DashboardState
}

data class CategorySpending(
    val category: Category,
    val amount: Double,
    val percentage: Float
)
```

**ViewModel Implementation**:
```kotlin
// presentation/screens/dashboard/DashboardViewModel.kt
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val getMonthlySpendingUseCase: GetMonthlySpendingUseCase,
    private val getCategoryBreakdownUseCase: GetCategoryBreakdownUseCase,
    private val getRecentTransactionsUseCase: GetRecentTransactionsUseCase,
    private val getBudgetUseCase: GetBudgetUseCase
) : ViewModel() {

    private val currentMonth = YearMonth.now()

    val dashboardState: StateFlow<DashboardState> = combine(
        getMonthlySpendingUseCase(currentMonth.year, currentMonth.monthValue),
        getBudgetUseCase(),
        getCategoryBreakdownUseCase(currentMonth.year, currentMonth.monthValue),
        getRecentTransactionsUseCase(limit = 5)
    ) { spending, budget, breakdown, transactions ->
        DashboardState.Success(
            monthlySpending = spending,
            budget = budget,
            categoryBreakdown = breakdown,
            recentTransactions = transactions
        )
    }.catch { error ->
        emit(DashboardState.Error(error.message ?: "Unknown error"))
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = DashboardState.Loading
    )

    fun refresh() {
        // Trigger data refresh
    }
}
```

**ViewModel Test**:
```kotlin
// test/presentation/screens/dashboard/DashboardViewModelTest.kt
@OptIn(ExperimentalCoroutinesApi::class)
class DashboardViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var getMonthlySpendingUseCase: GetMonthlySpendingUseCase
    private lateinit var getCategoryBreakdownUseCase: GetCategoryBreakdownUseCase
    private lateinit var getRecentTransactionsUseCase: GetRecentTransactionsUseCase
    private lateinit var getBudgetUseCase: GetBudgetUseCase

    private lateinit var viewModel: DashboardViewModel

    @Before
    fun setup() {
        getMonthlySpendingUseCase = mockk()
        getCategoryBreakdownUseCase = mockk()
        getRecentTransactionsUseCase = mockk()
        getBudgetUseCase = mockk()
    }

    @Test
    fun `dashboard loads successfully`() = runTest {
        // Given
        coEvery { getMonthlySpendingUseCase(any(), any()) } returns flowOf(1000.0)
        coEvery { getBudgetUseCase() } returns flowOf(2000.0)
        coEvery { getCategoryBreakdownUseCase(any(), any()) } returns flowOf(emptyList())
        coEvery { getRecentTransactionsUseCase(any()) } returns flowOf(emptyList())

        // When
        viewModel = DashboardViewModel(
            getMonthlySpendingUseCase,
            getCategoryBreakdownUseCase,
            getRecentTransactionsUseCase,
            getBudgetUseCase
        )

        // Then
        viewModel.dashboardState.test {
            assertEquals(DashboardState.Loading, awaitItem())
            val successState = awaitItem() as DashboardState.Success
            assertEquals(1000.0, successState.monthlySpending)
            assertEquals(2000.0, successState.budget)
        }
    }
}
```

---

#### Hour 4-6: Dashboard UI
**Tasks**:
- [ ] Create dashboard screen composable
- [ ] Implement summary cards
- [ ] Add pull-to-refresh
- [ ] Create chart placeholder (simple version)
- [ ] Write UI tests

**Dashboard Screen**:
```kotlin
// presentation/screens/dashboard/DashboardScreen.kt
@Composable
fun DashboardScreen(
    viewModel: DashboardViewModel = hiltViewModel(),
    onNavigateToTransactions: () -> Unit
) {
    val state by viewModel.dashboardState.collectAsState()

    val pullRefreshState = rememberPullRefreshState(
        refreshing = state is DashboardState.Loading,
        onRefresh = { viewModel.refresh() }
    )

    Box(
        modifier = Modifier
            .fillMaxSize()
            .pullRefresh(pullRefreshState)
    ) {
        when (val currentState = state) {
            is DashboardState.Loading -> LoadingIndicator()
            is DashboardState.Error -> ErrorState(currentState.message)
            is DashboardState.Success -> DashboardContent(
                state = currentState,
                onNavigateToTransactions = onNavigateToTransactions
            )
        }

        PullRefreshIndicator(
            refreshing = state is DashboardState.Loading,
            state = pullRefreshState,
            modifier = Modifier.align(Alignment.TopCenter)
        )
    }
}

@Composable
fun DashboardContent(
    state: DashboardState.Success,
    onNavigateToTransactions: () -> Unit
) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // Summary Card
        item {
            SummaryCard(
                spending = state.monthlySpending,
                budget = state.budget
            )
        }

        // Budget Progress
        item {
            BudgetProgressCard(
                spent = state.monthlySpending,
                budget = state.budget
            )
        }

        // Category Breakdown
        item {
            Text(
                text = "Spending by Category",
                style = MaterialTheme.typography.titleMedium,
                fontWeight = FontWeight.Bold
            )
        }

        items(state.categoryBreakdown) { categorySpending ->
            CategorySpendingItem(categorySpending)
        }

        // Recent Transactions
        item {
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(
                    text = "Recent Transactions",
                    style = MaterialTheme.typography.titleMedium,
                    fontWeight = FontWeight.Bold
                )
                TextButton(onClick = onNavigateToTransactions) {
                    Text("See All")
                }
            }
        }

        items(state.recentTransactions) { transaction ->
            ExpenseCard(
                transaction = transaction,
                onClick = { /* Navigate to detail */ }
            )
        }
    }
}
```

---

### Day 7-8: Transaction List & CRUD (10-12h)

#### Hour 1-3: Transaction List Screen
**Tasks**:
- [ ] Create `TransactionViewModel` with Flow & Paging
- [ ] Implement transaction list with lazy loading
- [ ] Add search functionality with debounce
- [ ] Add filter functionality
- [ ] Write tests

**Transaction ViewModel**:
```kotlin
// presentation/screens/transactions/TransactionViewModel.kt
@HiltViewModel
class TransactionViewModel @Inject constructor(
    private val getAllTransactionsUseCase: GetAllTransactionsUseCase,
    private val deleteTransactionUseCase: DeleteTransactionUseCase,
    private val searchTransactionsUseCase: SearchTransactionsUseCase
) : ViewModel() {

    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()

    val transactions: StateFlow<List<Transaction>> = searchQuery
        .debounce(300)
        .flatMapLatest { query ->
            if (query.isEmpty()) {
                getAllTransactionsUseCase()
            } else {
                searchTransactionsUseCase(query)
            }
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    fun onSearchQueryChange(query: String) {
        _searchQuery.value = query
    }

    fun deleteTransaction(transaction: Transaction) {
        viewModelScope.launch {
            deleteTransactionUseCase(transaction)
        }
    }
}
```

**Transaction List UI**:
```kotlin
// presentation/screens/transactions/TransactionsScreen.kt
@Composable
fun TransactionsScreen(
    viewModel: TransactionViewModel = hiltViewModel(),
    onTransactionClick: (Long) -> Unit
) {
    val transactions by viewModel.transactions.collectAsState()
    val searchQuery by viewModel.searchQuery.collectAsState()

    Column(modifier = Modifier.fillMaxSize()) {
        // Search Bar
        SearchBar(
            query = searchQuery,
            onQueryChange = { viewModel.onSearchQueryChange(it) },
            modifier = Modifier.fillMaxWidth()
        )

        // Transaction List
        LazyColumn(
            modifier = Modifier.fillMaxSize(),
            contentPadding = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            // Group by date
            val groupedTransactions = transactions.groupBy { it.date }

            groupedTransactions.forEach { (date, transactionsForDate) ->
                stickyHeader {
                    DateHeader(date = date)
                }

                items(
                    items = transactionsForDate,
                    key = { it.id }
                ) { transaction ->
                    val dismissState = rememberDismissState(
                        confirmValueChange = {
                            if (it == DismissValue.DismissedToStart) {
                                viewModel.deleteTransaction(transaction)
                                true
                            } else false
                        }
                    )

                    SwipeToDismiss(
                        state = dismissState,
                        background = {
                            DismissBackground(dismissState.dismissDirection)
                        },
                        dismissContent = {
                            ExpenseCard(
                                transaction = transaction,
                                onClick = { onTransactionClick(transaction.id) }
                            )
                        }
                    )
                }
            }
        }
    }
}
```

---

#### Hour 4-6: Add Transaction Bottom Sheet
**Tasks**:
- [ ] Create `AddTransactionViewModel`
- [ ] Implement bottom sheet with form
- [ ] Add validation logic
- [ ] Add amount formatter
- [ ] Write UI tests

**Add Transaction ViewModel**:
```kotlin
// presentation/screens/transactions/AddTransactionViewModel.kt
@HiltViewModel
class AddTransactionViewModel @Inject constructor(
    private val addTransactionUseCase: AddTransactionUseCase,
    private val getCategoriesUseCase: GetCategoriesUseCase
) : ViewModel() {

    var amount by mutableStateOf("")
        private set
    var selectedCategory by mutableStateOf<Category?>(null)
        private set
    var description by mutableStateOf("")
        private set
    var selectedDate by mutableStateOf(LocalDate.now())
        private set
    var notes by mutableStateOf("")
        private set

    val categories: StateFlow<List<Category>> = getCategoriesUseCase()
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5000),
            emptyList()
        )

    val isValid: Boolean
        get() = amount.toDoubleOrNull() != null
            && amount.toDouble() > 0
            && selectedCategory != null
            && description.isNotBlank()

    fun onAmountChange(newAmount: String) {
        // Format and validate
        amount = newAmount.filter { it.isDigit() || it == '.' }
    }

    fun onCategorySelect(category: Category) {
        selectedCategory = category
    }

    fun onDescriptionChange(newDescription: String) {
        description = newDescription
    }

    fun onDateSelect(date: LocalDate) {
        selectedDate = date
    }

    fun onNotesChange(newNotes: String) {
        if (newNotes.length <= 200) {
            notes = newNotes
        }
    }

    fun saveTransaction(onSuccess: () -> Unit) {
        viewModelScope.launch {
            val transaction = Transaction(
                amount = amount.toDouble(),
                category = selectedCategory!!,
                currency = "CAD",
                description = description,
                notes = notes.ifBlank { null },
                date = selectedDate
            )

            addTransactionUseCase(transaction).onSuccess {
                onSuccess()
            }
        }
    }
}
```

**Bottom Sheet UI**:
```kotlin
// presentation/screens/transactions/AddTransactionBottomSheet.kt
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AddTransactionBottomSheet(
    viewModel: AddTransactionViewModel = hiltViewModel(),
    onDismiss: () -> Unit
) {
    val sheetState = rememberModalBottomSheetState()
    val categories by viewModel.categories.collectAsState()

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp)
                .navigationBarsPadding(),
            verticalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            Text(
                text = "Add Transaction",
                style = MaterialTheme.typography.titleLarge,
                fontWeight = FontWeight.Bold
            )

            // Amount Input
            OutlinedTextField(
                value = viewModel.amount,
                onValueChange = { viewModel.onAmountChange(it) },
                label = { Text("Amount") },
                leadingIcon = { Text("$") },
                keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Decimal),
                modifier = Modifier.fillMaxWidth()
            )

            // Category Selection
            Text("Category", style = MaterialTheme.typography.labelLarge)
            LazyRow(
                horizontalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                items(categories) { category ->
                    CategoryChip(
                        category = category,
                        selected = viewModel.selectedCategory == category,
                        onClick = { viewModel.onCategorySelect(category) }
                    )
                }
            }

            // Description Input
            OutlinedTextField(
                value = viewModel.description,
                onValueChange = { viewModel.onDescriptionChange(it) },
                label = { Text("Description") },
                modifier = Modifier.fillMaxWidth()
            )

            // Date Picker (simplified)
            OutlinedTextField(
                value = viewModel.selectedDate.toString(),
                onValueChange = {},
                label = { Text("Date") },
                readOnly = true,
                trailingIcon = {
                    IconButton(onClick = { /* Show date picker */ }) {
                        Icon(Icons.Default.CalendarToday, "Select date")
                    }
                },
                modifier = Modifier.fillMaxWidth()
            )

            // Notes (Optional)
            OutlinedTextField(
                value = viewModel.notes,
                onValueChange = { viewModel.onNotesChange(it) },
                label = { Text("Notes (optional)") },
                supportingText = { Text("${viewModel.notes.length}/200") },
                maxLines = 3,
                modifier = Modifier.fillMaxWidth()
            )

            // Save Button
            Button(
                onClick = {
                    viewModel.saveTransaction(onSuccess = onDismiss)
                },
                enabled = viewModel.isValid,
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Save Transaction")
            }
        }
    }
}
```

**Checkpoint**: At this point, you should have a working MVP with:
- ✅ Add transactions
- ✅ View transactions (with search)
- ✅ Delete transactions (swipe)
- ✅ Dashboard overview
- ✅ ~30 passing tests

---

## Phase 2: Security & Enhancement (Week 3)

### Day 9-10: Security Implementation (10-12h)

#### Hour 1-4: Biometric Authentication
**Tasks**:
- [ ] Implement `BiometricAuthenticator` class
- [ ] Create biometric prompt screen
- [ ] Add fallback PIN system
- [ ] Implement auto-lock mechanism
- [ ] Write security tests

**Biometric Authenticator**:
```kotlin
// data/security/BiometricAuthenticator.kt
class BiometricAuthenticator @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun authenticate(
        activity: FragmentActivity,
        onSuccess: () -> Unit,
        onError: (String) -> Unit
    ) {
        val executor = ContextCompat.getMainExecutor(context)

        val biometricPrompt = BiometricPrompt(
            activity,
            executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    onError(errString.toString())
                }

                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    onSuccess()
                }

                override fun onAuthenticationFailed() {
                    onError("Authentication failed")
                }
            }
        )

        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Unlock SecureExpense")
            .setSubtitle("Use biometric to access your financial data")
            .setAllowedAuthenticators(BIOMETRIC_STRONG or DEVICE_CREDENTIAL)
            .build()

        biometricPrompt.authenticate(promptInfo)
    }
}
```

**Auth Screen**:
```kotlin
// presentation/screens/auth/BiometricAuthScreen.kt
@Composable
fun BiometricAuthScreen(
    viewModel: AuthViewModel = hiltViewModel(),
    onAuthSuccess: () -> Unit
) {
    val context = LocalContext.current
    val activity = context.findActivity()

    LaunchedEffect(Unit) {
        viewModel.authenticate(
            activity = activity,
            onSuccess = onAuthSuccess
        )
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Default.Fingerprint,
            contentDescription = "Biometric",
            modifier = Modifier.size(80.dp),
            tint = MaterialTheme.colorScheme.primary
        )

        Spacer(modifier = Modifier.height(24.dp))

        Text(
            text = "Unlock with Biometric",
            style = MaterialTheme.typography.headlineMedium
        )

        Spacer(modifier = Modifier.height(8.dp))

        Text(
            text = "Touch the fingerprint sensor",
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )

        Spacer(modifier = Modifier.height(32.dp))

        TextButton(onClick = { viewModel.showPinFallback() }) {
            Text("Use PIN Instead")
        }
    }
}
```

---

#### Hour 5-8: Database Encryption
**Tasks**:
- [ ] Add SQLCipher dependency
- [ ] Implement encrypted database
- [ ] Add EncryptedSharedPreferences
- [ ] Migrate existing data (if any)
- [ ] Test encryption

**Encrypted Database**:
```kotlin
// build.gradle.kts
dependencies {
    implementation("net.zetetic:android-database-sqlcipher:4.5.4")
    implementation("androidx.sqlite:sqlite:2.4.0")
    implementation("androidx.security:security-crypto:1.1.0-alpha06")
}

// di/DatabaseModule.kt
@Provides
@Singleton
fun provideDatabase(
    @ApplicationContext context: Context,
    securityManager: SecurityManager
): ExpenseDatabase {
    val passphrase = securityManager.getDatabasePassphrase()
    val factory = SupportFactory(passphrase)

    return Room.databaseBuilder(
        context,
        ExpenseDatabase::class.java,
        "secure_expense.db"
    )
    .openHelperFactory(factory)
    .build()
}

// data/security/SecurityManager.kt
class SecurityManager @Inject constructor(
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

    fun getDatabasePassphrase(): ByteArray {
        val storedPassphrase = encryptedPrefs.getString(KEY_DB_PASSPHRASE, null)

        return if (storedPassphrase != null) {
            Base64.decode(storedPassphrase, Base64.DEFAULT)
        } else {
            // Generate new passphrase
            val newPassphrase = ByteArray(32).apply {
                SecureRandom().nextBytes(this)
            }
            encryptedPrefs.edit()
                .putString(KEY_DB_PASSPHRASE, Base64.encodeToString(newPassphrase, Base64.DEFAULT))
                .apply()
            newPassphrase
        }
    }

    companion object {
        private const val KEY_DB_PASSPHRASE = "db_passphrase"
    }
}
```

---

#### Hour 9-10: Certificate Pinning & ProGuard
**Tasks**:
- [ ] Implement certificate pinning for API
- [ ] Configure ProGuard rules
- [ ] Add root detection (basic)
- [ ] Disable screenshots for sensitive screens

**Certificate Pinning**:
```kotlin
// di/NetworkModule.kt
@Provides
@Singleton
fun provideOkHttpClient(): OkHttpClient {
    val certificatePinner = CertificatePinner.Builder()
        .add("api.exchangerate-api.com", "sha256/AAAAAAAAAAAAA...") // Replace with actual hash
        .build()

    return OkHttpClient.Builder()
        .certificatePinner(certificatePinner)
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
            else HttpLoggingInterceptor.Level.NONE
        })
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build()
}
```

**ProGuard Rules** (app/proguard-rules.pro):
```proguard
# Keep data models
-keep class com.secureexpense.data.model.** { *; }
-keep class com.secureexpense.domain.model.** { *; }

# Keep Retrofit interfaces
-keep interface com.secureexpense.data.remote.** { *; }

# Kotlinx Serialization
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.**
-keep,includedescriptorclasses class com.secureexpense.**$$serializer { *; }

# Room
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class *

# Obfuscate
-repackageclasses 'o'
-allowaccessmodification
```

**Security Utils**:
```kotlin
// util/SecurityUtils.kt
object SecurityUtils {
    fun isDeviceRooted(): Boolean {
        // Check for common root indicators
        val paths = arrayOf(
            "/system/app/Superuser.apk",
            "/sbin/su",
            "/system/bin/su",
            "/system/xbin/su"
        )
        return paths.any { File(it).exists() }
    }

    @Composable
    fun SecureScreen(content: @Composable () -> Unit) {
        val context = LocalContext.current
        val activity = context.findActivity()

        DisposableEffect(Unit) {
            // Disable screenshots
            activity.window.setFlags(
                WindowManager.LayoutParams.FLAG_SECURE,
                WindowManager.LayoutParams.FLAG_SECURE
            )

            onDispose {
                activity.window.clearFlags(WindowManager.LayoutParams.FLAG_SECURE)
            }
        }

        content()
    }
}
```

---

### Day 11-12: Currency Exchange Integration (8-10h)

#### Hour 1-3: Exchange Rate API
**Tasks**:
- [ ] Create API interface with Retrofit
- [ ] Implement repository with caching
- [ ] Add WorkManager for background updates
- [ ] Write integration tests

**API Implementation**:
```kotlin
// data/remote/api/ExchangeRateApi.kt
interface ExchangeRateApi {
    @GET("latest/{base}")
    suspend fun getLatestRates(
        @Path("base") baseCurrency: String
    ): Response<ExchangeRateResponse>
}

@Serializable
data class ExchangeRateResponse(
    val base: String,
    val date: String,
    @SerialName("time_last_updated")
    val timeLastUpdated: Long,
    val rates: Map<String, Double>
)

// data/repository/ExchangeRateRepositoryImpl.kt
class ExchangeRateRepositoryImpl @Inject constructor(
    private val api: ExchangeRateApi,
    private val rateDao: ExchangeRateDao,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) : ExchangeRateRepository {

    override suspend fun getExchangeRate(from: String, to: String): Result<Double> {
        return withContext(dispatcher) {
            try {
                // Check cache first
                val cachedRate = rateDao.getRate(from, to)
                if (cachedRate != null && !cachedRate.isExpired()) {
                    return@withContext Result.success(cachedRate.rate)
                }

                // Fetch from API
                val response = api.getLatestRates(from)
                if (response.isSuccessful) {
                    val rates = response.body()?.rates ?: emptyMap()
                    val rate = rates[to] ?: return@withContext Result.failure(
                        Exception("Rate not found")
                    )

                    // Cache the rate
                    rateDao.insertRate(
                        ExchangeRateEntity(
                            fromCurrency = from,
                            toCurrency = to,
                            rate = rate,
                            timestamp = System.currentTimeMillis()
                        )
                    )

                    Result.success(rate)
                } else {
                    Result.failure(Exception("API error: ${response.code()}"))
                }
            } catch (e: Exception) {
                // Fallback to cached rate even if expired
                val cachedRate = rateDao.getRate(from, to)
                if (cachedRate != null) {
                    Result.success(cachedRate.rate)
                } else {
                    Result.failure(e)
                }
            }
        }
    }
}
```

**Background Worker**:
```kotlin
// data/worker/ExchangeRateUpdateWorker.kt
@HiltWorker
class ExchangeRateUpdateWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: ExchangeRateRepository
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            // Update rates for common currencies
            val currencies = listOf("USD", "EUR", "GBP", "JPY")
            currencies.forEach { currency ->
                repository.getExchangeRate("CAD", currency)
            }
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }

    companion object {
        const val WORK_NAME = "exchange_rate_update"

        fun schedule(context: Context) {
            val constraints = Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()

            val request = PeriodicWorkRequestBuilder<ExchangeRateUpdateWorker>(
                repeatInterval = 24,
                repeatIntervalTimeUnit = TimeUnit.HOURS
            )
                .setConstraints(constraints)
                .build()

            WorkManager.getInstance(context)
                .enqueueUniquePeriodicWork(
                    WORK_NAME,
                    ExistingPeriodicWorkPolicy.KEEP,
                    request
                )
        }
    }
}
```

---

#### Hour 4-6: Multi-Currency UI
**Tasks**:
- [ ] Add currency selector to transaction form
- [ ] Display converted amounts
- [ ] Add currency settings screen
- [ ] Show exchange rate disclaimer

---

### Day 13-14: UI Polish & Charts (8-10h)

#### Hour 1-4: Chart Components
**Tasks**:
- [ ] Create custom pie chart with Canvas
- [ ] Create line chart for trends
- [ ] Add animations to charts
- [ ] Write chart UI tests

**Pie Chart**:
```kotlin
// presentation/components/charts/PieChart.kt
@Composable
fun CategoryPieChart(
    data: List<CategorySpending>,
    modifier: Modifier = Modifier
) {
    var animationProgress by remember { mutableStateOf(0f) }

    LaunchedEffect(data) {
        animate(
            initialValue = 0f,
            targetValue = 1f,
            animationSpec = tween(800, easing = FastOutSlowInEasing)
        ) { value, _ ->
            animationProgress = value
        }
    }

    Canvas(modifier = modifier.size(200.dp)) {
        val totalAmount = data.sumOf { it.amount }
        var startAngle = -90f

        data.forEach { categorySpending ->
            val sweepAngle = (categorySpending.amount / totalAmount * 360f * animationProgress).toFloat()

            drawArc(
                color = categorySpending.category.color,
                startAngle = startAngle,
                sweepAngle = sweepAngle,
                useCenter = true,
                size = size
            )

            startAngle += sweepAngle
        }
    }
}
```

---

#### Hour 5-8: Analytics Screen
**Tasks**:
- [ ] Create analytics screen layout
- [ ] Add spending trends chart
- [ ] Add category comparison
- [ ] Add insights cards

---

## Phase 3: Testing & Documentation (Week 4)

### Day 15-16: Comprehensive Testing (10-12h)

#### Hour 1-3: Unit Test Coverage
**Tasks**:
- [ ] Achieve 80%+ coverage for ViewModels
- [ ] 80%+ coverage for UseCases
- [ ] 80%+ coverage for Repositories
- [ ] Run coverage report

```bash
# Generate coverage report
./gradlew testDebugUnitTestCoverage
open app/build/reports/coverage/test/debug/index.html
```

---

#### Hour 4-6: UI Test Coverage
**Tasks**:
- [ ] E2E test: Add transaction flow
- [ ] E2E test: Search and delete
- [ ] E2E test: Navigation flow
- [ ] E2E test: Biometric authentication (mocked)

**E2E Test Example**:
```kotlin
// androidTest/e2e/AddTransactionFlowTest.kt
@HiltAndroidTest
class AddTransactionFlowTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun completeAddTransactionFlow() {
        // Given - User on dashboard
        composeRule.onNodeWithText("Dashboard").assertIsDisplayed()

        // When - User taps FAB
        composeRule.onNodeWithContentDescription("Add transaction").performClick()

        // Then - Bottom sheet appears
        composeRule.onNodeWithText("Add Transaction").assertIsDisplayed()

        // When - User fills form
        composeRule.onNodeWithText("Amount").performTextInput("50.00")
        composeRule.onNodeWithText("Groceries").performClick()
        composeRule.onNodeWithText("Description").performTextInput("Weekly shopping")

        // And - User saves
        composeRule.onNodeWithText("Save Transaction").performClick()

        // Then - Transaction appears in list
        composeRule.waitUntil(5000) {
            composeRule.onAllNodesWithText("Weekly shopping").fetchSemanticsNodes().isNotEmpty()
        }
        composeRule.onNodeWithText("Weekly shopping").assertIsDisplayed()
        composeRule.onNodeWithText("$50.00").assertIsDisplayed()
    }
}
```

---

#### Hour 7-10: Performance Testing
**Tasks**:
- [ ] Profile app with Android Profiler
- [ ] Check for memory leaks (LeakCanary)
- [ ] Test with 1000+ transactions
- [ ] Measure startup time
- [ ] Optimize identified bottlenecks

---

### Day 17-18: Documentation & Polish (8-10h)

#### Hour 1-3: Documentation
**Tasks**:
- [ ] Write comprehensive README.md
- [ ] Create SETUP.md with instructions
- [ ] Write SECURITY.md documenting security measures
- [ ] Add KDoc comments to public APIs
- [ ] Create architecture diagram

**README Template**:
```markdown
# SecureExpense 🔐💰

A modern, secure Android expense tracking app showcasing advanced Android development practices.

## 🎯 Highlights

- **Security-First**: Bank-grade encryption, biometric authentication
- **Modern Architecture**: Clean Architecture + MVVM with Jetpack Compose
- **Reactive**: Kotlin Coroutines & Flow for async operations
- **Tested**: 80%+ code coverage with comprehensive test suite
- **Performance**: 60fps UI, optimized for large datasets

## 📱 Screenshots

[Add 5-8 screenshots]

## 🏗️ Architecture

[Add architecture diagram]

### Tech Stack

- **Language**: Kotlin 100%
- **UI**: Jetpack Compose + Material 3
- **Architecture**: Clean Architecture + MVVM
- **DI**: Hilt
- **Database**: Room (encrypted with SQLCipher)
- **Networking**: Retrofit + OkHttp
- **Async**: Coroutines + Flow
- **Testing**: JUnit, MockK, Turbine, Compose Testing

## 🚀 Getting Started

See [SETUP.md](SETUP.md) for detailed setup instructions.

## 🔒 Security Features

See [SECURITY.md](SECURITY.md) for security implementation details.

## 📊 Test Coverage

- Unit Tests: 85%
- UI Tests: 75%
- Overall: 82%

## 📄 License

MIT License - see [LICENSE](LICENSE)
```

---

#### Hour 4-6: Demo Materials
**Tasks**:
- [ ] Record 2-minute walkthrough video
- [ ] Take high-quality screenshots (5-8 screens)
- [ ] Create performance benchmark document
- [ ] Export test coverage reports

---

#### Hour 7-10: Final Polish
**Tasks**:
- [ ] Fix all lint warnings
- [ ] Run detekt and fix issues
- [ ] Optimize imports
- [ ] Format code consistently
- [ ] Create signed release APK
- [ ] Tag release v1.0.0 in Git

```bash
# Lint check
./gradlew lint

# Detekt (static analysis)
./gradlew detekt

# Build release APK
./gradlew assembleRelease

# Sign APK (requires keystore)
jarsigner -verbose -sigalg SHA256withRSA \
  -digestalg SHA-256 \
  -keystore my-release-key.jks \
  app/build/outputs/apk/release/app-release-unsigned.apk \
  alias_name
```

---

## 📦 Final Deliverables Checklist

### Code Repository
- [ ] GitHub repository (public)
- [ ] README.md with screenshots
- [ ] SETUP.md with instructions
- [ ] SECURITY.md with security details
- [ ] LICENSE file
- [ ] Clean commit history
- [ ] Tagged release (v1.0.0)

### Builds
- [ ] Signed release APK
- [ ] ProGuard mapping files
- [ ] App tested on physical device

### Documentation
- [ ] Architecture diagram (PNG/SVG)
- [ ] API documentation
- [ ] Test coverage report (HTML)
- [ ] Performance benchmarks

### Demo Materials
- [ ] 2-minute walkthrough video (YouTube/Loom)
- [ ] Screenshot gallery (8-10 high-quality screens)
- [ ] Feature highlight document

---

## 🎯 Success Metrics

### Technical Excellence
- ✅ 100% Kotlin (0 Java files)
- ✅ >80% test coverage
- ✅ 0 critical security vulnerabilities (MobSF scan)
- ✅ Lint score >95
- ✅ 0 memory leaks

### Performance
- ✅ <2s cold start time
- ✅ 60fps UI (no jank)
- ✅ <100MB memory usage
- ✅ <15MB APK size

### RBC Requirements Coverage
- ✅ Demonstrates 5+ years Kotlin expertise
- ✅ Advanced Jetpack Compose with shared components
- ✅ Mastery of Coroutines & Flows
- ✅ RESTful API integration
- ✅ Hilt dependency injection
- ✅ Test-driven development
- ✅ Application security expertise
- ✅ Exquisite UI with excellent performance

---

## 🔄 Optional Enhancements (If Time Permits)

### Bonus Features
- [ ] RxJava alternative implementation (separate branch)
- [ ] OpenAPI spec for future backend
- [ ] Export transactions as CSV
- [ ] Dark mode support
- [ ] Widget for quick expense entry
- [ ] Backup/restore functionality

### Open Source Contributions
- [ ] Extract shared Compose components as library
- [ ] Publish library to Maven Central
- [ ] Write Medium article on security implementation
- [ ] Create YouTube tutorial on Clean Architecture

---

## 📝 Daily Progress Tracking

Use this template to track your progress:

```markdown
## Day [X] - [Date]

### Completed
- [ ] Task 1
- [ ] Task 2

### In Progress
- [ ] Task 3

### Blockers
- None / [Describe blocker]

### Tomorrow's Goals
- [ ] Goal 1
- [ ] Goal 2

### Notes
[Any important learnings or decisions]
```

---

## 🆘 Troubleshooting Common Issues

### Issue 1: SQLCipher Build Errors
**Solution**: Ensure correct SQLCipher version and exclude default SQLite:
```kotlin
configurations.all {
    exclude(group = "androidx.sqlite", module = "sqlite")
}
```

### Issue 2: Biometric Prompt Not Showing
**Solution**: Check device capabilities and permissions:
```kotlin
val biometricManager = BiometricManager.from(context)
when (biometricManager.canAuthenticate(BIOMETRIC_STRONG)) {
    BiometricManager.BIOMETRIC_SUCCESS -> // Show prompt
    else -> // Show error or fallback
}
```

### Issue 3: Room Migration Errors
**Solution**: Use fallback migration strategy during development:
```kotlin
.fallbackToDestructiveMigration() // Remove for production!
```

---

## 📚 Additional Resources

### Documentation
- [Jetpack Compose](https://developer.android.com/jetpack/compose)
- [Kotlin Coroutines](https://kotlinlang.org/docs/coroutines-overview.html)
- [Room Database](https://developer.android.com/training/data-storage/room)
- [Hilt](https://dagger.dev/hilt/)

### Security
- [Android Security Best Practices](https://developer.android.com/topic/security/best-practices)
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)

### Testing
- [Compose Testing](https://developer.android.com/jetpack/compose/testing)
- [Testing Coroutines](https://kotlinlang.org/docs/coroutines-guide.html#testing-coroutines)

---

**Good luck with your implementation! 🚀**

Remember: **Start with MVP, test continuously, commit often, and polish at the end.**
