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
| Data source | **Apple Health (HealthKit)**. The Google Health app (formerly Fitbit) writes Fitbit Air data into it |
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
- **Known gaps:** Google Health doesn't share HRV or skin temperature. Neither is needed.
- **Backup plan if the Google Health → Apple Health sync breaks:** the Google Health API,
  which Bevel uses. Its access is currently restricted. Everything goes through one
  protocol, so the source can be swapped:

```swift
protocol HealthDataSource {
    func dailySteps(in range: DateInterval) async throws -> [Date: Int]
    func dailyEnergy(in range: DateInterval) async throws -> [Date: EnergySplit] // active, resting
    func sleep(in range: DateInterval) async throws -> [SleepSession]
    func workouts(in range: DateInterval) async throws -> [RecordedWorkout]
}
// HealthKitSource (version 1) · MockSource (previews/tests) · GoogleHealthSource (backup)
```

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
Habbit/                       app target
  App/                        HabbitApp, RootView, AppState
  DesignSystem/               Colors, Typography, Card, WeekStrip, CheckToggle, Confetti
  Features/
    Onboarding/ Home/ Steps/ Calories/ Sleep/ Workouts/ Streaks/ Profile/
  Data/
    Models/                   SwiftData models
    Health/                   HealthDataSource, HealthKitSource, MockSource
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
| 4 | **Workouts**: templates, exercise seed data, log sheet, workout detection and notification, Today card | Log a Fitbit-recorded session and see the next suggestion |
| 5 | **Streaks and celebrations**: week strip, chips, freezes, milestones, confetti | Streaks update correctly from real data |
| 6 | **Detail screens**: calorie pie, sleep insights, progress charts | Every card opens to a useful detail screen |
| 7 | **Setup and polish**: onboarding, widget, evening nudge, dark mode, README | Ready to use daily and to open-source |
| later | Custom habits, Google Health API source, App Store | |

## 7. Getting your Mac ready

1. Install **Xcode** (the version with the iOS 27 SDK) from the Mac App Store. It's
   large, so start the download early.
2. Open Xcode → Settings → Accounts → add your Apple ID. Free signing works, but apps
   expire after 7 days. The $99/yr developer account removes that limit and enables
   TestFlight.
3. iPhone: Settings → Privacy & Security → **Developer Mode** on. Then connect it by
   cable once.
