# SecureExpense - Quick Setup Guide

**Get started in 30 minutes!** ⚡

---

## ✅ Prerequisites Checklist

Before starting, ensure you have:

- [ ] **Android Studio** (Hedgehog 2023.1.1 or newer)
- [ ] **JDK 17** or higher
- [ ] **Git** installed
- [ ] **Android SDK** installed (API 26-34)
- [ ] **Physical Android device** (recommended for biometric testing)
- [ ] **GitHub account** (for repository hosting)

---

## 🚀 Step 1: Create New Project (5 minutes)

### 1.1 Open Android Studio
```
File → New → New Project
```

### 1.2 Project Configuration
```
Template: Empty Activity
Name: SecureExpense
Package name: com.secureexpense
Save location: [Your preferred location]
Language: Kotlin
Minimum SDK: API 26 (Android 8.0)
Build configuration language: Kotlin DSL (build.gradle.kts)
```

### 1.3 Initial Setup
Click "Finish" and wait for Gradle sync to complete.

---

## 📦 Step 2: Configure Dependencies (10 minutes)

### 2.1 Update `build.gradle.kts` (Project Level)

Replace the content with:

```kotlin
// Top-level build file
plugins {
    id("com.android.application") version "8.2.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.22" apply false
    id("com.google.dagger.hilt.android") version "2.50" apply false
    id("org.jetbrains.kotlin.plugin.serialization") version "1.9.22" apply false
}
```

### 2.2 Update `build.gradle.kts` (App Level)

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
    id("com.google.dagger.hilt.android")
    id("org.jetbrains.kotlin.plugin.serialization")
}

android {
    namespace = "com.secureexpense"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.secureexpense"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        vectorDrawables {
            useSupportLibrary = true
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
        debug {
            applicationIdSuffix = ".debug"
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.8"
    }

    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    // Compose BOM (Bill of Materials)
    val composeBom = platform("androidx.compose:compose-bom:2024.02.00")
    implementation(composeBom)
    androidTestImplementation(composeBom)

    // Compose
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.material:material-icons-extended")

    // Compose Debugging
    debugImplementation("androidx.compose.ui:ui-tooling")
    debugImplementation("androidx.compose.ui:ui-test-manifest")

    // Core Android
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
    implementation("androidx.activity:activity-compose:1.8.2")

    // ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")

    // Navigation
    implementation("androidx.navigation:navigation-compose:2.7.6")

    // Hilt
    implementation("com.google.dagger:hilt-android:2.50")
    kapt("com.google.dagger:hilt-compiler:2.50")
    implementation("androidx.hilt:hilt-navigation-compose:1.1.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")

    // Room
    val roomVersion = "2.6.1"
    implementation("androidx.room:room-runtime:$roomVersion")
    implementation("androidx.room:room-ktx:$roomVersion")
    kapt("androidx.room:room-compiler:$roomVersion")

    // SQLCipher (for encrypted database)
    implementation("net.zetetic:android-database-sqlcipher:4.5.4")
    implementation("androidx.sqlite:sqlite:2.4.0")

    // Retrofit & OkHttp
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

    // Kotlinx Serialization
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
    implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")

    // Security
    implementation("androidx.security:security-crypto:1.1.0-alpha06")
    implementation("androidx.biometric:biometric:1.2.0-alpha05")

    // DataStore
    implementation("androidx.datastore:datastore-preferences:1.0.0")

    // WorkManager
    implementation("androidx.work:work-runtime-ktx:2.9.0")

    // Testing - Unit Tests
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.mockk:mockk:1.13.9")
    testImplementation("app.cash.turbine:turbine:1.0.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    testImplementation("androidx.arch.core:core-testing:2.2.0")

    // Testing - Android Tests
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    androidTestImplementation("com.google.dagger:hilt-android-testing:2.50")
    kaptAndroidTest("com.google.dagger:hilt-compiler:2.50")

    // Debug Tools
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.13")
}

// Allow references to generated code (Hilt)
kapt {
    correctErrorTypes = true
}
```

### 2.3 Update `gradle.properties`

Add these lines:

```properties
# Enable Jetpack Compose
android.useAndroidX=true
android.enableJetifier=true

# Kotlin
kotlin.code.style=official

# Performance
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true

# AndroidX
android.nonTransitiveRClass=true
```

### 2.4 Sync Gradle

Click "Sync Now" in Android Studio. This may take 5-10 minutes for first sync.

---

## 🏗️ Step 3: Create Project Structure (5 minutes)

### 3.1 Create Package Structure

Right-click on `com.secureexpense` → New → Package

Create these packages:
```
com.secureexpense
├── data
│   ├── local
│   │   ├── dao
│   │   ├── entity
│   │   └── database
│   ├── remote
│   │   ├── api
│   │   └── dto
│   ├── repository
│   └── mapper
├── domain
│   ├── model
│   ├── repository
│   └── usecase
├── presentation
│   ├── navigation
│   ├── components
│   ├── theme
│   ├── screens
│   │   ├── auth
│   │   ├── dashboard
│   │   ├── transactions
│   │   ├── budget
│   │   ├── analytics
│   │   └── settings
│   └── util
├── di
└── util
```

### 3.2 Create Application Class

Create `SecureExpenseApp.kt` in `com.secureexpense`:

```kotlin
package com.secureexpense

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class SecureExpenseApp : Application()
```

### 3.3 Update AndroidManifest.xml

Add the application class:

```xml
<application
    android:name=".SecureExpenseApp"
    android:allowBackup="false"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:supportsRtl="true"
    android:theme="@style/Theme.SecureExpense"
    android:usesCleartextTraffic="false"
    tools:targetApi="31">

    <!-- Add biometric permission -->
    <uses-permission android:name="android.permission.USE_BIOMETRIC" />
    <uses-permission android:name="android.permission.INTERNET" />

    <activity
        android:name=".MainActivity"
        android:exported="true"
        android:theme="@style/Theme.SecureExpense">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
</application>
```

---

## 🎨 Step 4: Setup Theme (5 minutes)

### 4.1 Create Theme Files

Create these files in `presentation/theme/`:

**Color.kt**:
```kotlin
package com.secureexpense.presentation.theme

import androidx.compose.ui.graphics.Color

// Light Theme Colors (RBC Branding)
val md_theme_light_primary = Color(0xFF005EB8) // RBC Blue
val md_theme_light_onPrimary = Color(0xFFFFFFFF)
val md_theme_light_primaryContainer = Color(0xFFD6E3FF)
val md_theme_light_onPrimaryContainer = Color(0xFF001B3D)

val md_theme_light_secondary = Color(0xFFFFD400) // RBC Gold
val md_theme_light_onSecondary = Color(0xFF000000)
val md_theme_light_secondaryContainer = Color(0xFFFFE566)
val md_theme_light_onSecondaryContainer = Color(0xFF3D3000)

val md_theme_light_error = Color(0xFFBA1A1A)
val md_theme_light_onError = Color(0xFFFFFFFF)

// Dark Theme Colors
val md_theme_dark_primary = Color(0xFF99CCFF)
val md_theme_dark_onPrimary = Color(0xFF003258)
val md_theme_dark_secondary = Color(0xFFFFE566)
val md_theme_dark_onSecondary = Color(0xFF3D3000)
val md_theme_dark_error = Color(0xFFFFB4AB)
```

**Typography.kt**:
```kotlin
package com.secureexpense.presentation.theme

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

val Typography = Typography(
    displayLarge = TextStyle(
        fontWeight = FontWeight.Bold,
        fontSize = 57.sp,
        lineHeight = 64.sp
    ),
    headlineMedium = TextStyle(
        fontWeight = FontWeight.SemiBold,
        fontSize = 28.sp,
        lineHeight = 36.sp
    ),
    titleLarge = TextStyle(
        fontWeight = FontWeight.SemiBold,
        fontSize = 22.sp,
        lineHeight = 28.sp
    ),
    bodyLarge = TextStyle(
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp
    ),
    labelMedium = TextStyle(
        fontWeight = FontWeight.Medium,
        fontSize = 12.sp,
        lineHeight = 16.sp
    )
)
```

**Theme.kt**:
```kotlin
package com.secureexpense.presentation.theme

import android.os.Build
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.dynamicDarkColorScheme
import androidx.compose.material3.dynamicLightColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.ui.platform.LocalContext

private val LightColorScheme = lightColorScheme(
    primary = md_theme_light_primary,
    onPrimary = md_theme_light_onPrimary,
    primaryContainer = md_theme_light_primaryContainer,
    onPrimaryContainer = md_theme_light_onPrimaryContainer,
    secondary = md_theme_light_secondary,
    onSecondary = md_theme_light_onSecondary,
    error = md_theme_light_error,
    onError = md_theme_light_onError
)

private val DarkColorScheme = darkColorScheme(
    primary = md_theme_dark_primary,
    onPrimary = md_theme_dark_onPrimary,
    secondary = md_theme_dark_secondary,
    onSecondary = md_theme_dark_onSecondary,
    error = md_theme_dark_error
)

@Composable
fun SecureExpenseTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

---

## 🔧 Step 5: Update MainActivity (2 minutes)

Replace `MainActivity.kt` content:

```kotlin
package com.secureexpense

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import com.secureexpense.presentation.theme.SecureExpenseTheme
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            SecureExpenseTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    Greeting("SecureExpense")
                }
            }
        }
    }
}

@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "Hello $name!",
        modifier = modifier
    )
}

@Preview(showBackground = true)
@Composable
fun GreetingPreview() {
    SecureExpenseTheme {
        Greeting("SecureExpense")
    }
}
```

---

## ✅ Step 6: Verify Setup (3 minutes)

### 6.1 Build the Project

Click: **Build → Make Project** (or Ctrl+F9 / Cmd+F9)

Wait for build to complete. Should see "Build Successful" message.

### 6.2 Run the App

1. Connect Android device or start emulator
2. Click "Run" button (green triangle) or press Shift+F10
3. You should see "Hello SecureExpense!" on screen

### 6.3 Run Tests

Click: **Run → Run 'All Tests'**

Initial test should pass (example test from template).

---

## 🎯 Step 7: Initialize Git Repository (2 minutes)

### 7.1 Create .gitignore

Create `.gitignore` in project root:

```gitignore
# Built application files
*.apk
*.aab
*.ap_
*.aab

# Files for the ART/Dalvik VM
*.dex

# Java class files
*.class

# Generated files
bin/
gen/
out/
release/

# Gradle files
.gradle/
build/

# Local configuration file (sdk path, etc)
local.properties

# IntelliJ
*.iml
.idea/
misc.xml
deploymentTargetDropDown.xml
render.experimental.xml

# Keystore files
*.jks
*.keystore

# External native build folder generated in Android Studio 2.2 and later
.externalNativeBuild
.cxx/

# Version control
.svn/

# Android Studio captures folder
captures/

# Lint
lint/

# OS-specific files
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db
```

### 7.2 Initialize Git

```bash
git init
git add .
git commit -m "Initial project setup with dependencies and theme"
```

### 7.3 Create GitHub Repository

1. Go to GitHub.com
2. Click "New Repository"
3. Name: `SecureExpense`
4. Description: "Bank-grade expense tracking Android app"
5. Public repository
6. Don't initialize with README (we already have code)
7. Click "Create repository"

### 7.4 Push to GitHub

```bash
git remote add origin https://github.com/YOUR_USERNAME/SecureExpense.git
git branch -M main
git push -u origin main
```

---

## 🎉 Setup Complete!

You now have:
- ✅ Android project with all dependencies
- ✅ Clean Architecture structure
- ✅ Material 3 theme (RBC colors)
- ✅ Hilt dependency injection setup
- ✅ Git repository with GitHub remote

---

## 🚀 Next Steps

### Immediate Next Steps (Today)
1. ✅ **Verify everything runs**: Build and run app on device/emulator
2. ✅ **Read Day 1 in Implementation Plan**: Understand what to build next
3. ✅ **Start implementing**: Follow Day 1 tasks (Database setup)

### This Week
- Implement data layer (Room database)
- Create domain models and use cases
- Build dashboard screen
- Add transaction CRUD

---

## 🆘 Troubleshooting

### Problem: Gradle sync fails
**Solution**:
1. Check internet connection
2. Invalidate Caches: File → Invalidate Caches and Restart
3. Update Gradle wrapper: `./gradlew wrapper --gradle-version=8.2`

### Problem: Hilt errors ("@HiltAndroidApp not found")
**Solution**:
1. Ensure kapt plugin is applied: `id("kotlin-kapt")` in build.gradle.kts
2. Clean and rebuild: Build → Clean Project, then Build → Rebuild Project

### Problem: Can't run on device
**Solution**:
1. Enable Developer Options on device (tap Build Number 7 times)
2. Enable USB Debugging in Developer Options
3. Install device drivers (Windows) or allow connection (Mac/Linux)

### Problem: App crashes on startup
**Solution**:
1. Check Logcat for error message
2. Ensure `@HiltAndroidApp` is on Application class
3. Ensure `@AndroidEntryPoint` is on MainActivity
4. Verify AndroidManifest.xml has correct application name

---

## 📚 Useful Commands

```bash
# Build project
./gradlew build

# Run tests
./gradlew test

# Run specific test
./gradlew test --tests "DashboardViewModelTest"

# Generate test coverage report
./gradlew jacocoTestReport

# Check for lint issues
./gradlew lint

# Clean build
./gradlew clean

# Assemble release APK
./gradlew assembleRelease
```

---

## 🎓 Resources

- **Android Studio Shortcuts**: [Keyboard Shortcuts](https://developer.android.com/studio/intro/keyboard-shortcuts)
- **Compose Documentation**: [Jetpack Compose](https://developer.android.com/jetpack/compose)
- **Hilt Documentation**: [Hilt Dependency Injection](https://dagger.dev/hilt/)

---

**Setup complete! Time to start building! 🚀**

Proceed to **SecureExpense_Implementation_Plan.md** → **Day 1** to begin development.
