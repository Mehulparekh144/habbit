# Habbit: plan

An open-source iOS app for tracking fitness habits. It reads Fitbit Air data from Apple
Health, logs strength workouts with progressive overload, and turns hitting your goals
into streaks.

The look follows the Ronas IT "Habit Tracker App" concept on Dribbble (colours, type,
card grid, week strip), with **no mascot**.

Status: the plan is settled after a grilling session (Oct 2026). Next step is building.

---

## 1. Decisions

| Area | Decision |
|---|---|
| Platform | Native SwiftUI, **iOS 27 minimum**, built with Xcode on macOS 27 |
| Distribution | Open source, **MIT**. The author is the first user, but nothing is hard-coded to one person, so App Store release stays possible |
| Backend | None. All data lives on the device |
| Data source | **Apple Health (HealthKit)** is the main source. The Google Health app (formerly Fitbit) writes Fitbit Air data into it |
| Google Health | **Optional "Connect Google Health"** (OAuth). It's a data connection, not an app login. It adds HRV and fills gaps when Apple Health has nothing for a day. Each person who builds Habbit uses their own Google Cloud client ID |
| Steps | Daily goal (default 10,000, editable), hourly, 7- and 30-day charts |
| Calories | Goal = **all active calories** (default 500 kcal, editable). Pie chart: **Workouts vs Steps & movement**. Resting calories shown for context |
| Sleep | Stage bar and plain-language comparisons to your own 14-day average. Default goal 7.5 h |
| Workouts | Your own templates. Pre-loaded common exercises plus custom ones. **Sets × reps × weight × RPE per set** |
| Scheduling | **Flexible** by default (N workouts a week), with an optional weekly schedule |
| Logging | **After** the Fitbit syncs: Habbit detects the new strength workout and asks you to fill in what you did. Calories and heart rate come from the Fitbit. You can also log without a Fitbit workout |
| At the gym | Read-only "Today's workout" card: targets, last time's numbers, suggested weight |
| Progression | Double progression. Charts for estimated 1RM, weekly volume per muscle, PRs |
| Streaks | 🔥 Daily · 💪 Weekly workouts · ⭐ Perfect Day badge · 🧊 Freezes |
| Setup | Asks for units (kg/lb and weight step), body stats and goals. All editable later |
| Version 1 scope | Fitness only. Custom habits come after version 1 |

## 2. Data: Apple Health

**Device setup** (covered in the app's setup screens):
1. Google Health → Connections → Apps and services → Apple Health → Turn On All.
2. Apple Health → for each of Steps, Active Energy, Resting Energy, Walking + Running
   Distance and Sleep: Data Sources & Access → Edit → drag **Google Health to the top**.

**Reading the data:**
- Totals use `HKStatisticsCollectionQuery` with `.cumulativeSum`. It applies the user's
  source order and removes duplicates, so the iPhone and Fitbit are never counted twice.
- **Priority check:** if the iPhone is supplying samples for a metric but Google Health
  isn't, Habbit shows a "Your iPhone is overriding your Fitbit" tip with the fix steps.
- **Data read:** `stepCount`, `activeEnergyBurned`, `basalEnergyBurned`,
  `distanceWalkingRunning`, `sleepAnalysis` (stages), `HKWorkout` and `heartRate`.
- Habbit **writes nothing** to Apple Health in version 1.
- **Background updates:** `HKObserverQuery` with background delivery, so Habbit learns
  about new workouts and sleep, plus `BGAppRefreshTask` as a backup.
- **Known gaps:** Google Health doesn't share HRV or skin temperature with Apple Health.
  The optional Google Health connection (section 2b) covers HRV.
- Everything goes through one protocol, so sources can be combined or swapped:

```swift
protocol HealthDataSource {
    func dailySteps(in range: DateInterval) async throws -> [Date: Int]
    func dailyEnergy(in range: DateInterval) async throws -> [Date: EnergySplit] // active, resting
    func sleep(in range: DateInterval) async throws -> [SleepSession]
    func workouts(in range: DateInterval) async throws -> [RecordedWorkout]
}
// HealthKitSource (main) · GoogleHealthSource (optional) · MockSource (previews/tests)
// CompositeSource merges them using the rules in 2b
```

## 2b. Optional: Google Health API connection

**What it is:** a "Connect Google Health" button in setup and in Settings. Habbit has
no accounts and no server, so this is a **data connection, not a login**. The app works
fully without it.

**Which source wins, per metric:**

| Metric | Source |
|---|---|
| Steps, active and resting kcal, sleep, workouts | Apple Health. Use Google only for a day where Apple Health has **no data at all** (e.g. the sync was turned off) |
| HRV | Google Health only (Apple Health never gets it) |
| Resting heart rate | Apple Health |

Never add the two sources together, so nothing is counted twice.

**How sign-in works (no server):**
- OAuth 2.0 with an **iOS client** (no client secret) and **PKCE**, using
  `ASWebAuthenticationSession`. The redirect is the reversed client ID scheme
  (`com.googleusercontent.apps.<id>:/oauth2redirect`).
- **Read-only** scopes only: `googlehealth.activity_and_fitness.readonly`,
  `googlehealth.sleep.readonly` and `googlehealth.health_metrics_and_measurements.readonly`.
  Google renamed scopes in May 2026, so confirm the exact names during the test.
- The refresh token goes in the **Keychain**. The access token is refreshed on demand.
- **When the token expires:** if Google revokes or expires it, Habbit shows a
  "Reconnect Google Health" chip and keeps running on Apple Health data. Nothing breaks.
- **Fetching:** `GET health.googleapis.com/v4/users/me/dataTypes/{type}/dataPoints`,
  called when the app opens and from `BGAppRefreshTask`. Results are cached in SwiftData.
  Webhooks need a server, so they're not used.

**Open source and Google's limits:**
- Google Health scopes are **restricted**. An app Google hasn't verified is limited
  to **100 users**, and going past that needs verification plus a yearly security
  assessment. A shared client ID would hit that limit.
- So **each person who builds Habbit brings their own client ID**:
  `Config/Secrets.xcconfig` (in .gitignore) sets `GOOGLE_CLIENT_ID`, and
  `Secrets.example.xcconfig` is committed as a template. With no ID set, the Google
  button is hidden.
- README steps: create a Google Cloud project → enable the Google Health API →
  set up the OAuth consent screen and add yourself as a test user → create an iOS
  OAuth client with the app's bundle ID → paste the ID into `Secrets.xcconfig`.

**Test first (before building screens on it):**
1. Confirm Google lets the project enable and call the Health API. New projects were
   restricted earlier in 2026.
2. Confirm an **iOS** client type works with these scopes.
3. Read today's steps, last night's sleep and HRV from the phone.
4. **Check how long the sign-in lasts.** An app in "Testing" mode may have to sign in
   again every 7 days. Try "In production" without verification (you'll see Google's
   "unverified app" warning, 100-user limit) to see if that keeps the sign-in alive.
5. **If the test fails:** keep Apple Health only. HRV is a nice-to-have, so nothing in
   version 1 depends on it.

## 3. Features

### 3.1 Calories
- **Ring:** active kcal compared with the goal.
- **Pie chart:**
  - **Workouts:** the summed `totalEnergyBurned` of the day's `HKWorkout`s. If a
    workout has no energy value, use the active energy recorded during its time window.
  - **Steps & movement:** total active calories minus workout calories, never below 0.
- **Below the chart:** "Resting 1,149 kcal · Total 1,299 kcal", matching the Fitbit
  headline number.

### 3.2 Sleep
- A night is the main sleep session ending between 3 pm the previous day and 3 pm on
  the day it belongs to. Naps are shown separately.
- **Stage bar** (Awake / REM / Light, which HealthKit calls Core / Deep) with plain lines:
  - "You slept **7 h 12 m**, 18 m short of your goal."
  - "Deep + REM **2 h 05 m**, about normal for you." (Normal means within ±15% of
    your 14-day median.)
  - "Bedtime was 48 m later than usual."
- **Weekly view:** duration bars and a bedtime-consistency band.
- Informational only, with no medical claims.

### 3.3 Workouts and progressive overload

**Domain model (SwiftData):**
```
Exercise         name, muscleGroups[], equipment, isCustom
WorkoutTemplate  name ("Push A"), [TemplateExercise]
TemplateExercise exercise, order, targetSets, repRange (min...max), targetRPE
Schedule         mode: .flexible(perWeek: Int) | .weekly([Weekday: WorkoutTemplate?])
WorkoutLog       template?, start, end, healthKitWorkoutID?, energyKcal?, avgHR?
LoggedExercise   exercise, order, [LoggedSet]
LoggedSet        weight, reps, rpe (6.0–10.0 in 0.5 steps), isWarmup
UserSettings     unit (kg/lb), weightStep, goals, bodyStats
```

**Logging flow:**
1. The Fitbit records a strength workout (`traditionalStrengthTraining` /
   `functionalStrengthTraining`, among others) and it reaches Apple Health.
2. Habbit's observer fires. It finds an `HKWorkout` with no linked `WorkoutLog` and
   sends a notification: "52-min strength session at 6:10 pm. What did you train?"
3. The log sheet, styled like the green session screen, opens with the template's
   exercises already filled in from targets or last time. You adjust weight, reps and
   RPE per set and save. The log is linked by `healthKitWorkoutID`.
4. **No Fitbit workout:** use "Log workout" and enter start and end times by hand.
   Calories are recorded as "not recorded".

**"Today's workout" card** (read-only, for the gym): the next template in the rotation,
or the scheduled one. Each exercise shows target sets × rep range @ RPE, last time's
sets, and the **suggested weight**.

**Overload engine** (pure Swift, unit-tested). Rules for each exercise, based on the
last session's working sets:
- Every set reached `repRange.max` with RPE ≤ 8: **weight + weightStep**.
- Any set fell below `repRange.min`, or RPE ≥ 9.5, in **two sessions in a row**:
  **deload ~10%**, rounded to the weight step.
- Otherwise: **same weight, aim for +1 rep**.
- **Progress metrics:**
  - Estimated 1RM (Epley: `w × (1 + reps/30)`, sets of 12 reps or fewer)
  - Weekly sets and volume per muscle group
  - PRs: best estimated 1RM, and best weight at a given rep count

### 3.4 Streaks
- **🔥 Daily streak:** a day counts when all *enabled* daily goals are met. Steps is on
  by default; active kcal and sleep can be switched on. Today stays "in progress" until
  midnight and never breaks the streak early.
- **💪 Weekly workout streak:** a week (Mon–Sun, configurable) counts when the workout
  target is met: N logs in flexible mode, or every scheduled workout in weekly mode.
- **⭐ Perfect Day:** all daily goals met **and** a workout logged. It earns a badge and
  a bigger celebration, but it isn't its own streak.
- **🧊 Freezes:** earn 1 per 7 consecutive daily-streak days, bank up to 2, and they're
  used automatically to cover one missed day. They apply to the daily streak only.
- **Milestones:** 3, 7, 14, 30, 50, 100, 200 and 365 days, celebrated with confetti
  and haptics.
- **Streaks are recalculated, never stored as counters.** They're derived from daily
  data, so late syncs and edits always give the right answer.

## 4. Design system

| Token | Value | Use |
|---|---|---|
| `indigo` | `#4B4FE0` | Setup background, + button, focus |
| `lavender` | `#B8C2F2` | Hero backgrounds |
| `lime` | `#C3EA7C` | Completed days, steps, workout log sheet |
| `butter` | `#F3D06E` | Workout card |
| `bubblegum` | `#EE8FC6` | Calories card |
| `sky` | `#A8C5F7` | Sleep card |
| `tangerine` | `#F28A45` | Streak flame, PR accents |
| `mint` | `#A7E4DF` | Secondary accents |
| `ink` | `#111111` | Text, "today" pill, check mark |
| `paper` | `#FFFFFF` / `#F5F5F7` | Background / inputs |

- **Mockups:** `docs/design/mockups.html` (also published as a private Artifact) is the visual reference for every screen.
- **Type:** a heavy rounded display face for headings and big numbers (Archivo Black or
  Rubik Bold, both under the Open Font License and bundled). SF Pro or Inter for body.
- **Shapes:** 20–24 pt rounded cards in a 2-column grid, circular check toggles, pill
  day chips.
- **No mascot:** celebrations use confetti, haptics, bold numbers and an animated flame.
- **Dark mode:** pastel cards stay, and the background switches to near-black.

**Screens:**
1. **Setup** (indigo):
   - Welcome
   - Connect Apple Health, with the source-order walkthrough
   - Units and weight step
   - Body stats
   - Goals
   - Workout plan: pick from presets or build your own, then flexible or weekly mode
2. **Home:**
   - "Good morning, {name}"
   - Week strip: lime = daily goals met, ink = today
   - 🔥/💪 streak chips
   - Cards: Steps · Workout (today's) · Active kcal · Sleep
   - Tab bar: Home · **+** (log workout) · Profile
3. **Detail screens:** Steps, Calories (pie chart), Sleep, Workout history and progress.
4. **Sheets:** log a workout, edit a template, add an exercise.
5. **Profile:** streak calendar, badges, goals, units, data-source health check.

## 5. Project layout

```
Habbit.xcodeproj
Config/                       Secrets.example.xcconfig (Secrets.xcconfig is gitignored)
Habbit/                       app target
  App/                        HabbitApp, RootView, AppState
  DesignSystem/               Colors, Typography, Card, WeekStrip, CheckToggle, Confetti
  Features/
    Onboarding/ Home/ Steps/ Calories/ Sleep/ Workouts/ Streaks/ Profile/
  Data/
    Models/                   SwiftData models
    Health/                   HealthDataSource, HealthKitSource, GoogleHealthSource,
                              CompositeSource, MockSource
    Auth/                     GoogleOAuth (PKCE), KeychainStore
    Sync/                     HealthObserver, BackgroundRefresh, WorkoutMatcher
  Resources/                  Fonts, exercise seed JSON
HabbitWidgets/                streak + steps widget
Packages/HabbitCore/          Swift package, pure logic, no UIKit or HealthKit
  Sources/HabbitCore/         OverloadEngine, StreakEngine, SleepInsights,
                              CalorieBreakdown, WorkoutMatcher, Units
  Tests/HabbitCoreTests/
LICENSE (MIT) · README.md · PLAN.md
```

All the decision-making logic lives in **`HabbitCore`**, a standalone Swift package
built test-first. `swift test` runs it on the Mac without a simulator.

## 6. Build phases

| # | Phase | Done when |
|---|---|---|
| 1 | **HabbitCore** (test-first): overload, streaks, calorie breakdown, sleep insights, workout matching | `swift test` passes |
| 2 | **Xcode project and design system**: tokens, fonts, components, Home with `MockSource` | Home looks like the reference design with fake data |
| 3 | **Apple Health**: permissions, `HealthKitSource`, priority check, observers | Real Fitbit steps, kcal and sleep on Home |
| 3b | **Google Health (optional)**: test, OAuth with PKCE, Keychain, `GoogleHealthSource`, HRV on the Sleep screen | HRV shows up, and losing the sign-in falls back to Apple Health without errors |
| 4 | **Workouts**: templates, exercise seed data, log sheet, workout detection and notification, Today card | Log a Fitbit-recorded session and see the next suggestion |
| 5 | **Streaks and celebrations**: week strip, chips, freezes, milestones, confetti | Streaks update correctly from real data |
| 6 | **Detail screens**: calorie pie, sleep insights, progress charts | Every card opens to a useful detail screen |
| 7 | **Setup and polish**: onboarding, widget, evening nudge, dark mode, README | Ready to use daily and to open-source |
| later | Custom habits, App Store, readiness score built from HRV | |

## 7. Getting your Mac ready

1. Install **Xcode** (the version with the iOS 27 SDK) from the Mac App Store. It's
   large, so start the download early.
2. Open Xcode → Settings → Accounts → add your Apple ID. Free signing works, but apps
   expire after 7 days. The $99/yr developer account removes that limit and enables
   TestFlight.
3. iPhone: Settings → Privacy & Security → **Developer Mode** on. Then connect it by
   cable once.
4. *(For the optional Google Health connection)* At console.cloud.google.com: create a
   project → enable **Google Health API** → **OAuth consent screen**, adding your Gmail
   as a test user → **Credentials → Create OAuth client ID → iOS**, with the bundle ID
   (e.g. `com.mehulparekh.habbit`). Keep the client ID handy.
