# Guardsquare (DexGuard) & New Relic Plugin Incompatibility Sample

This repository is a minimal reproducible example (reproducer) demonstrating a build classpath dependency conflict between the **Guardsquare Gradle Plugin** and the **New Relic Android Agent Gradle Plugin**.

---

## 🚨 The Issue

When both plugins are declared in the root `build.gradle.kts` classpath, executing Guardsquare-related tasks (such as `./gradlew guardsquareDownloadCli`) fails with a `NoSuchMethodError` / `Unable to find method` exception:

```text
Unable to find method 'java.nio.file.attribute.FileTime org.apache.commons.io.file.attribute.FileTimes.fromUnixTime(long)'
'java.nio.file.attribute.FileTime org.apache.commons.io.file.attribute.FileTimes.fromUnixTime(long)'
```

### Root Cause Analysis
This is caused by a **classpath collision** on the `commons-io:commons-io` library inside the Gradle buildscript classpath:
* One of the plugins (likely the Guardsquare plugin) expects a newer version of `commons-io` that includes `org.apache.commons.io.file.attribute.FileTimes`.
* The New Relic Gradle plugin (e.g., version `7.7.6`) pulls in an older version of `commons-io` (or transitively resolves to one) that lacks the `FileTimes.fromUnixTime(long)` method, overriding the newer version in Gradle's flat classloader.

---

## 🛠️ How to Reproduce

1. Ensure both plugins are active in the root `build.gradle.kts` file:
   ```kotlin
   // build.gradle.kts (Root)
   plugins {
       // ...
       alias(libs.plugins.guardsquare) apply false
       alias(libs.plugins.newrelic) apply false // <--- KEEP ACTIVE TO TRIGGER THE ERROR
   }
   ```

2. Run the following command in your terminal:
   ```bash
   ./gradlew guardsquareDownloadCli
   ```

3. The build will fail with the `FileTimes.fromUnixTime` missing method error.

---

## 🔍 Workaround / Proof of Concept

To verify that New Relic is introducing the incompatible dependency:

1. Open the root `build.gradle.kts` file.
2. Comment out or remove the New Relic plugin declaration on line 6:
   ```kotlin
   plugins {
       alias(libs.plugins.android.application) apply false
       alias(libs.plugins.kotlin.compose) apply false
       alias(libs.plugins.guardsquare) apply false
       // alias(libs.plugins.newrelic) apply false // Commented out
   }
   ```
3. Run the command again:
   ```bash
   ./gradlew guardsquareDownloadCli
   ```
4. The task will now complete successfully with no errors.
