---
name: Kotlin Tooling
description: Development tools, linting, and build configuration for Kotlin projects.
metadata:
  labels: [kotlin, tooling, gradle, detekt, ktlint]
  triggers:
    files: ['build.gradle.kts', 'settings.gradle.kts', 'detekt.yml']
    keywords: [gradle, detekt, ktlint, kotlin-gradle-plugin]
---

# Kotlin Tooling

## **Priority: P1 (OPERATIONAL)**

Essential tooling for Kotlin development: build configuration, linting, and code quality.

## Gradle Configuration

### Kotlin DSL (build.gradle.kts)
```kotlin
plugins {
    kotlin("android") version "1.9.20"
    kotlin("kapt") version "1.9.20"
    id("com.google.devtools.ksp") version "1.9.20-1.0.14"
}

android {
    kotlinOptions {
        jvmTarget = "17"
        freeCompilerArgs += listOf(
            "-opt-in=kotlin.RequiresOptIn",
            "-Xcontext-receivers"
        )
    }
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")
}
```

### Version Catalogs (libs.versions.toml)
```toml
[versions]
kotlin = "1.9.20"
coroutines = "1.7.3"

[libraries]
kotlin-stdlib = { module = "org.jetbrains.kotlin:kotlin-stdlib", version.ref = "kotlin" }
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }

[plugins]
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

## Code Quality Tools

### Detekt (Static Analysis)
```yaml
# detekt.yml
build:
  maxIssues: 0

complexity:
  LongMethod:
    threshold: 60
  ComplexMethod:
    threshold: 15

naming:
  FunctionNaming:
    functionPattern: '[a-z][a-zA-Z0-9]*'
  ClassNaming:
    classPattern: '[A-Z][a-zA-Z0-9]*'

style:
  MagicNumber:
    ignoreNumbers: ['-1', '0', '1', '2']
```

**Gradle Integration:**
```kotlin
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.4"
}

detekt {
    buildUponDefaultConfig = true
    config.setFrom("$projectDir/config/detekt.yml")
}
```

### ktlint (Code Formatting)
```kotlin
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "11.6.1"
}

ktlint {
    version.set("1.0.1")
    android.set(true)
    ignoreFailures.set(false)
    
    filter {
        exclude("**/generated/**")
        include("**/kotlin/**")
    }
}
```

## Compiler Options

### Opt-in Requirements
```kotlin
// Enable experimental APIs
kotlinOptions {
    freeCompilerArgs += listOf(
        "-opt-in=kotlinx.coroutines.ExperimentalCoroutinesApi",
        "-opt-in=kotlinx.coroutines.FlowPreview",
        "-opt-in=androidx.compose.material3.ExperimentalMaterial3Api"
    )
}
```

### Progressive Mode
```kotlin
kotlinOptions {
    // Enable stricter checks
    freeCompilerArgs += "-progressive"
}
```

## KSP vs KAPT

### Prefer KSP (Kotlin Symbol Processing)
```kotlin
// ✅ Good - Faster than KAPT
plugins {
    id("com.google.devtools.ksp") version "1.9.20-1.0.14"
}

dependencies {
    ksp("androidx.room:room-compiler:2.6.1")
    ksp("com.google.dagger:hilt-compiler:2.48")
}

// ❌ Avoid KAPT when KSP available
kapt("androidx.room:room-compiler:2.6.1") // Slower
```

## Build Optimization

### Gradle Properties
```properties
# gradle.properties
org.gradle.jvmargs=-Xmx4g -XX:+HeapDumpOnOutOfMemoryError
org.gradle.parallel=true
org.gradle.caching=true

kotlin.code.style=official
kotlin.incremental=true
kotlin.incremental.java=true
```

### Build Cache
```kotlin
// settings.gradle.kts
buildCache {
    local {
        isEnabled = true
        directory = File(rootDir, "build-cache")
    }
}
```

## Testing Tools

### JUnit 5 (Recommended)
```kotlin
dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.1")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("com.google.truth:truth:1.1.5")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### Kotest (Alternative)
```kotlin
dependencies {
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
}
```

## IDE Configuration

### .editorconfig
```ini
[*.{kt,kts}]
indent_size = 4
insert_final_newline = true
max_line_length = 120
ij_kotlin_allow_trailing_comma = true
ij_kotlin_allow_trailing_comma_on_call_site = true
```

### Android Studio Settings
- **Code Style**: Kotlin → Set from → Kotlin style guide
- **Inspections**: Enable all Kotlin inspections
- **File Templates**: Customize for consistency

## Anti-Patterns

- **No Mixed Groovy/Kotlin DSL**: Use Kotlin DSL consistently
- **No KAPT When KSP Available**: KSP is 2x faster
- **No Hardcoded Versions**: Use version catalogs
- **No Disabled Linting**: Fix issues, don't suppress
- **No Ignoring Warnings**: Treat warnings as errors

## Reference & Examples

For detailed configuration examples:
See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

kotlin/best-practices | jetpack-compose/architecture
