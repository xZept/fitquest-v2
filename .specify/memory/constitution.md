<!--
Sync Impact Report
- Version change: uninitialized template → 1.0.0
- Modified principles:
  - [PRINCIPLE_1_NAME] → I. Established Android Architecture & Manual Wiring (NON-NEGOTIABLE)
  - [PRINCIPLE_2_NAME] → II. Room Offline-First Persistence (NON-NEGOTIABLE)
  - [PRINCIPLE_3_NAME] → III. USDA FoodData Central (FDC) API Integration & Caching
  - [PRINCIPLE_4_NAME] → IV. Secret & API Key Hygiene (NON-NEGOTIABLE)
  - [PRINCIPLE_5_NAME] → V. Gradle Build, Lint & Unit Test Verification Gates (NON-NEGOTIABLE)
- Added sections:
  - Package Structure & Technical Constraints
  - Development Workflow & Quality Gates
- Removed sections: None
- Dependent templates status (read-only per Scope Guard):
  - .specify/templates/plan-template.md: ✅ aligned (evaluates Constitution Check gates at runtime)
  - .specify/templates/spec-template.md: ✅ aligned
  - .specify/templates/tasks-template.md: ✅ aligned
- Deferred TODOs: None
-->

# FitQuest Constitution

## Core Principles

### I. Established Android Architecture & Manual Wiring (NON-NEGOTIABLE)

- FitQuest MUST remain a single-module (`:app`) Android application (`namespace = "com.example.fitquest"`, `minSdk = 27`, `compileSdk = 36`, `targetSdk = 36`, Kotlin `jvmTarget = "17"` with core library desugaring enabled).
- Feature screens MUST follow the established `AppCompatActivity`, `Fragment`/`BottomSheetDialogFragment`, `RecyclerView.Adapter`, and XML layout + ViewBinding UI patterns (`app/src/main/res/layout/`). Jetpack Compose usage MUST remain confined to `com.example.fitquest.ui.theme` or explicitly requested Compose components without migrating existing XML screens unprompted.
- Dependencies MUST be wired manually via `FitQuestApp` (`Application` subclass), `AppDatabase.getInstance(context)`, `DataStoreManager`, and direct repository instantiation (`FitquestRepository`, `FoodRepository`, `ProgressRepository`, `ShopRepository`, `ExerciseHelpRepo`).
- Contributors MUST NOT introduce dependency injection frameworks (Hilt, Dagger, Koin), new MVI/MVVM/Navigation frameworks, multi-module splits, or any third-party libraries not already declared in `gradle/libs.versions.toml` and `app/build.gradle.kts`.
- Asynchronous work MUST use Kotlin Coroutines (`lifecycleScope`, `Dispatchers.IO`), lightweight session/preference state MUST use Preferences DataStore (`DataStoreManager` on `"user_prefs"`), and scheduled background tasks MUST use existing `WorkManager` workers and `AlarmManager`/`BroadcastReceiver` components registered in `AndroidManifest.xml`.

**Rationale**: FitQuest's gamified workout, macro tracking, and shop flows rely on cohesive manual wiring across Activities and repositories; introducing DI containers or competing UI/state frameworks creates architectural fragmentation and build complexity.

### II. Room Offline-First Persistence (NON-NEGOTIABLE)

- Local storage MUST treat the Room database (`AppDatabase`, database name `"fitquestDB"`, KSP-compiled via `androidx.room:room-compiler`) alongside bundled `app/src/main/assets/*.csv` datasets as the authoritative offline-first source of truth for users, profiles, settings, active quests, quest history, workout sessions, set logs, wallets, monsters, items, weight logs, foods, portions, food logs, macro plans, and macro diaries.
- Every Room DAO call and repository data mutation MUST run off the main UI thread using `suspend` functions and `withContext(Dispatchers.IO)`.
- Food logging mutations (`logIntake`, `updateLogServing`, `deleteLog`) and macro target recalculations (`computeAndSaveMacroPlan`) MUST synchronously maintain daily `MacroDiary` rows keyed uniquely by `(userId, dayKey)`, where `dayKey` is an integer `YYYYMMDD` computed in the `Asia/Manila` time zone (`ZoneId.of("Asia/Manila")`).
- Entity changes in `Entities.kt` MUST preserve existing foreign keys (`ForeignKey.CASCADE` where applicable), indices, and `TypeConverters` (`Converters.kt`), and MUST explicitly account for `AppDatabase` versioning and `.fallbackToDestructiveMigration()` impact on local user data.

**Rationale**: Core fitness tracking (quests, workouts, weight history, and daily macro diaries) must operate reliably without network connectivity and keep aggregated daily macro snapshots consistent with individual meal logs.

### III. USDA FoodData Central (FDC) API Integration & Caching

- Remote food search and nutrient retrieval MUST go exclusively through `FdcService` (`https://api.nal.usda.gov/fdc/`, `v1/foods/search` and `v1/food/{id}`) constructed via `FdcApi` using Retrofit 2, OkHttp 4, and Moshi (`KotlinJsonAdapterFactory`).
- Raw FDC payloads (`FdcModels`) MUST be mapped through `FdcMappers` (`toFoodEntity()` and `toPortions()`) into normalized per-100g macro values (`kcalPer100g`, `proteinPer100g`, `carbPer100g`, `fatPer100g`, clamped to non-negative doubles) and `MeasurementType`-backed `Portion` records with a `100g` `GRAM` baseline.
- `FoodRepository` MUST persist fetched FDC foods (`source = "USDA FDC"`, `sourceRef = fdcId.toString()`) and derived `Portion` entries into Room (`FoodDao.upsertKeepId`, `PortionDao`) so subsequent serving updates, unit conversions (`gramsFor`), and recent food lookups resolve locally and respect FDC rate limits (`X-RateLimit-Remaining`).
- Network failures or missing household portion labels MUST degrade gracefully (falling back to gram-based portions or cached Room entries rather than crashing the caller Activity).

**Rationale**: Normalizing external USDA FDC data at the mapper boundary and caching foods/portions in Room decouples local meal diary math from API schema quirks, latency, and rate limits.

### IV. Secret & API Key Hygiene (NON-NEGOTIABLE)

- All secrets and API keys—including `FDC_API_KEY`—MUST be stored exclusively in git-ignored `local.properties` (or external user/CI Gradle properties) and read in `app/build.gradle.kts` via `providers.gradleProperty("FDC_API_KEY").orNull ?: ""` into `BuildConfig.FDC_API_KEY`.
- Runtime consumers MUST access the FDC credential solely via `BuildConfig.FDC_API_KEY` passed to `FdcApi.create { BuildConfig.FDC_API_KEY }`, which attaches it via the `X-Api-Key` HTTP request header.
- Source files, XML resources, assets, tests, Gradle scripts, and documentation MUST NEVER hardcode or commit real API keys, tokens, or credentials, and `.gitignore` MUST retain `local.properties`.
- Logcat output (`Log.d`, OkHttp `HttpLoggingInterceptor`) MUST NOT log raw secret values or full `X-Api-Key` header strings (`HttpLoggingInterceptor.Level.BASIC` or metadata-only logs such as key length and `X-RateLimit-Remaining` are permitted).

**Rationale**: Preventing credential leakage in Git history and runtime logs protects project API quotas and satisfies Android security hygiene without requiring external secret-management SDKs.

### V. Gradle Build, Lint & Unit Test Verification Gates (NON-NEGOTIABLE)

- Every implementation change MUST verify project health using the three standard Gradle commands executed from the repository root (`.\gradlew.bat` on Windows PowerShell/CMD, `./gradlew` on POSIX shells):
  1. `./gradlew testDebugUnitTest` — executes JVM unit tests in `app/src/test/`.
  2. `./gradlew lint` — runs Android Lint static checks across `:app`.
  3. `./gradlew assembleDebug` — validates KSP Room codegen, KAPT Glide codegen, ViewBinding generation, BuildConfig generation, and debug APK packaging.
- Build configuration files (`build.gradle.kts`, `app/build.gradle.kts`, `gradle/libs.versions.toml`) MUST NOT introduce new dependencies, Gradle plugins, or repositories unless explicitly requested by the user and ratified via constitution amendment.

**Rationale**: Enforcing deterministic local Gradle gates catches Room KSP schema errors, ViewBinding mismatches, resource/manifest defects, and logic regressions before changes land.

## Package Structure & Technical Constraints

Code under `app/src/main/` MUST respect the existing package and asset organization:

- `com.example.fitquest` (`app/src/main/java/com/example/fitquest/`): Application entry point (`FitQuestApp`), screen `Activity` classes (`LoginActivity`, `RegisterActivity`, `DashboardActivity`, `WorkoutActivity`, `WorkoutGeneratorActivity`, `WorkoutSessionActivity`, `QuestGeneratorActivity`, `QuestPreviewActivity`, `ExerciseActivity`, `MacroActivity`, `FoodHistory`, `DiaryActivity`, `DiaryHistoryActivity`, `SessionDetailActivity`, `WeeklyReviewActivity`, `ProfileActivity`, `ShopActivity`, `WeightPromptActivity`), history `Fragment`s, `FoodSearchBottomSheet`, adapters, `WorkManager` workers/schedulers (`MidnightSnapshotWorker`, `MidnightMacroSnapshotWorker`, `MidnightSnapshotScheduler`, `ReminderScheduler`), and `BroadcastReceiver`s (`BootCompletedReceiver`, `WeightReminderReceiver`, `MacroSnapshotReceiver`).
- `com.example.fitquest.database`: `AppDatabase`, Room `@Entity` data classes (`Entities.kt`), `@Dao` interfaces (`DAO.kt`), `Converters.kt`, and `MeasurementType.kt`.
- `com.example.fitquest.fdc`: `FdcApi`, `FdcService`, `FdcModels`, and `FdcMappers`.
- `app/src/main/java/com/example/fitquest/repository/` & `shop/`: Repository and preference coordinators (`FitquestRepository`, `ProgressRepository`, `ExerciseHelpRepo`, `FoodRepository` declared under package `com.example.fitquest.data.repository`, `DataStoreManager` declared under package `com.example.fitquest.datastore`, and `ShopRepository` under `com.example.fitquest.shop`). Contributors MUST preserve existing package declarations on established files unless performing an explicitly approved refactor.
- `com.example.fitquest.models`: Domain and UI transfer models (`Exercise`, `ExerciseVideo`, `Food`, `ProgressModels`, `QuestExercise`, `Tips`).
- `com.example.fitquest.workout`: Workout plan and slot-selection engine (`WorkoutEngine.kt`).
- `com.example.fitquest.utils`, `com.example.fitquest.tips`, `com.example.fitquest.cosmetics`, `com.example.fitquest.ui`: Pure calculation utilities (`MacroCalculator`, `MeasureUtils`, `DayKey`), CSV/asset loaders (`CSVParser`, `ExerciseRepository`, `TipsLoader`, `TipsHelper`, `TipManager`), warmup dialog classes, background cosmetics, sprite-sheet drawables, and Compose theme files.
- `app/src/main/assets/`: Bundled offline CSV catalogs (`exercises.csv`, `exercises_info.csv`, `exercise_video_dataset.csv`, `meal_dataset.csv`, `tips_dataset.csv`).
- `app/src/main/AndroidManifest.xml`: All new Android components (`<activity>`, `<receiver>`, `<service>`) MUST declare explicit `android:exported` values (`false` unless required by launcher or system broadcast intent-filters) and adhere to minimum required permissions.

## Development Workflow & Quality Gates

- **Specification & Planning Alignment**: Every `spec.md`, `plan.md`, and `tasks.md` produced through Spec Kit MUST pass a Constitution Check confirming:
  - Zero new libraries, plugins, or DI frameworks introduced.
  - Preservation of Activity/Fragment + XML ViewBinding UI and manual singleton/repository wiring.
  - Room offline-first persistence and `Asia/Manila` `YYYYMMDD` `dayKey` consistency where data is read or mutated.
  - FDC API calls routed exclusively through `FdcService` -> `FdcMappers` -> `FoodRepository` Room cache, authenticated only via `BuildConfig.FDC_API_KEY` from `local.properties`.
- **Unit Testing Discipline**: Pure domain logic—such as `MacroCalculator`, `FdcMappers`, unit/portion conversion math, `DayKey` formatting, and `WorkoutEngine` slot selection—MUST be covered or validated with JUnit 4 tests under `app/src/test/` when modified.
- **Pre-Completion Verification**: No implementation task is complete until `./gradlew testDebugUnitTest`, `./gradlew lint`, and `./gradlew assembleDebug` succeed cleanly.

## Governance

- This Constitution supersedes conflicting generic Android templates, external style guides, or ad-hoc shortcuts within the FitQuest repository.
- **Amendment Procedure**: Modifying any Core Principle, adding a library or Gradle plugin, introducing a DI or UI/state framework, or altering Room migration rules requires:
  1. Explicit user approval documenting the technical justification.
  2. An amendment to `.specify/memory/constitution.md` with an updated Sync Impact Report and version bump.
  3. Verification that `./gradlew testDebugUnitTest`, `./gradlew lint`, and `./gradlew assembleDebug` pass after any accompanying migration.
- **Versioning Policy**: Constitution versions follow Semantic Versioning (`MAJOR.MINOR.PATCH`):
  - `MAJOR`: Removal or backward-incompatible redefinition of an existing principle or architectural guardrail.
  - `MINOR`: Addition of a new principle/section or material expansion of mandatory governance rules.
  - `PATCH`: Wording clarifications, formatting improvements, or typo corrections with no semantic rule change.
- **Compliance Review**: All plans, task lists, code reviews, and bug remediations MUST verify adherence to Principles I–V prior to merge or commit.

**Version**: 1.0.0 | **Ratified**: 2026-11-26 | **Last Amended**: 2026-11-26
