# Habbit: plan for a personal iOS habit and fitness tracker

A personal iOS app that pulls steps, calories and sleep from a Fitbit Air, tracks a
workout plan with progressive overload, and keeps the streaks going. The visual style
follows the Ronas IT "Habit Tracker App" concept on Dribbble: chunky type, pastel cards,
a lavender and indigo palette and a mascot.

---

## 1. Getting data from the Fitbit Air (the riskiest part, so it comes first)

**The situation as of Oct 2026:**
- The legacy **Fitbit Web API** goes away on **Oct 30, 2026**, three weeks from now.
  Don't build on it.
- Its replacement is the **Google Health API** (`health.googleapis.com/v4`). It has
  steps, active energy burned, sleep with stages, and exercise sessions. Its scopes are
  "restricted", and Google says it is limiting onboarding of new projects for now.
- Fitbit does **not** write to Apple Health on iOS, so HealthKit can't be the main source.

**Plan:**
1. **Week 0 spike:** create a Google Cloud project, enable the Google Health API, set
   up an OAuth consent screen in *Testing* mode with your own Google account as the
   only test user, and use an iOS OAuth client with PKCE. Then try to read today's
   steps from the CLI or curl.
   - If that works, it's enough for a personal app. Testing mode needs no app
     verification, but refresh tokens may expire every 7 days, which means
     re-authenticating weekly. Confirm this during the spike.
   - If Google blocks the project, fall back to option 2.
2. **Fallback:** a third-party Fitbit→Apple Health sync app, with Habbit reading
   **HealthKit**. iPhone steps are a last resort.
3. **Architecture keeps this swappable.** All health data goes through one protocol,
   so the UI never knows the source:

```swift
protocol HealthDataSource {
    func steps(for day: Date) async throws -> Int
    func activeCalories(for day: Date) async throws -> Double   // exercise/movement only
    func sleep(for night: Date) async throws -> SleepSession?    // stages + timings
    func exerciseSessions(for day: Date) async throws -> [ExerciseSession]
}
// GoogleHealthSource (primary) · HealthKitSource (fallback) · MockSource (dev/previews)
```

## 2. Features

### 2.1 Steps
- Daily step goal, e.g. 10,000, shown as the lime card on Home with a progress ring.
- Hourly bar chart for today, plus 7- and 30-day trends.
- When the goal is hit: check mark, haptic, and the day counts toward the streak.

### 2.2 Calories ("burn X through exercise")
Fitbit's headline number is **total = BMR (maintenance) + active**. You want a goal you
reach through movement, so the app tracks two numbers:

| Metric | Meaning | Use |
|---|---|---|
| **Active burn**, the goal metric | Calories from movement and workouts only | Goal, e.g. 600 kcal. Fills the ring. |
| Total burn, for context | BMR + active, matching the Fitbit app | Shown smaller beneath |

- Google Health API returns *active energy burned*. The app calculates BMR on the device
  (Mifflin-St Jeor, from your age, height, weight and sex, entered in onboarding), so the
  total roughly matches what the Fitbit app shows.
- Breakdown: "Workouts 380 kcal · Walking/other 220 kcal". It uses exercise sessions
  from Fitbit, with logged lifting sessions matched by time.

### 2.3 Workout plan and progressive overload
**Data model (SwiftData):**
```
WorkoutPlan ─┬─ PlanDay (e.g. "Push", weekday Mon/Thu)
             └── PlannedExercise (exercise, target sets, rep range 8–12, target RPE)
WorkoutSession ── LoggedExercise ── LoggedSet (weight, reps, RPE 1–10, isWarmup)
Exercise (name, muscle group, equipment, icon)
```
- **Plan builder:** build your split, and each exercise gets sets × rep range plus a
  target effort (RPE).
- **Live session screen,** styled like the green "Meditation" screen: big session
  timer, current exercise, last session's numbers shown as ghost text ("Last: 60 kg × 10,
  10, 9 @ RPE 8"), quick +/- for weight and reps, an RPE picker, a rest timer, and a
  **Finish** button.
- **Overload engine** (double progression, all rules on the device):
  - All sets reach the **top of the rep range** at **RPE ≤ 8**: suggest **+2.5 kg**
    (or +5 lb) next time.
  - Reps drop below the range or RPE ≥ 9.5 twice in a row: suggest holding or dropping 5–10%.
  - Otherwise: "add a rep".
- **Progress views:** estimated 1RM per lift (Epley: `w × (1 + reps/30)`), weekly
  volume (sets × reps × weight) per muscle group, and PR badges with a mascot celebration.

### 2.4 Sleep you can actually understand
- Last night as a **simple stage bar** (Awake / REM / Light / Deep) with plain-language
  lines instead of jargon:
  - "You slept **7 h 12 m**, 18 m short of your 7.5 h goal."
  - "Deep + REM: **2 h 05 m**, about normal for you."
  - "Bedtime was 48 m later than your usual."
- **Your own baselines:** comparisons are against your 14-day average, not population
  norms.
- Weekly view: duration bars plus a bedtime-consistency band.
- Sleep goal met counts as an optional streak goal.
- These are informational insights only, with no medical claims.

### 2.5 Streaks (the motivating part)
- **Daily goals** are configurable. Default: *Workout done (on scheduled days)* +
  *Steps ≥ goal*. Active calories and sleep can be added.
- **Perfect Day** = all daily goals met. The main streak counts consecutive Perfect Days.
- **Rest days:** a planned rest day only needs the step goal, so rest never breaks
  the streak.
- **Streak freezes:** earn 1 per 7-day streak, up to 2 banked, and use them
  automatically on a missed day.
- **Celebrations:** confetti, an animated mascot, haptics, and milestone badges at
  3/7/14/30/50/100 days. The week strip on Home fills **lime** for completed days,
  matching the design.
- **Widget and notifications:** a Home/Lock Screen widget ("🔥 12 days · 6,240 / 10k
  steps") and an evening nudge if a goal is still open at 8 pm.

### 2.6 Custom habits (from the design's "Let's start a new habit" screen)
Reading, journaling and the like use name, description, interval (every day or specific
days) and icon/colour. They appear as cards next to the fitness cards and can optionally
count toward the Perfect Day.

## 3. Design system (from the reference shots)

| Token | Approx. value | Used for |
|---|---|---|
| `indigo` | `#4B4FE0` | Onboarding background, + button, focus ring |
| `lavender` | `#B8C2F2` | Hero/marketing backgrounds |
| `lime` | `#C3EA7C` | Completed days, steps card, workout session screen |
| `butter` | `#F3D06E` | Workout card |
| `bubblegum` | `#EE8FC6` | Calories card |
| `sky` | `#A8C5F7` | Sleep card |
| `tangerine` | `#F28A45` | Mascot, flame/streak accents |
| `mint` | `#A7E4DF` | Custom-habit icons |
| `ink` | `#111111` | Text, selected "today" pill, done checkmark |
| `paper` | `#FFFFFF` / `#F5F5F7` | Background / inputs |

- **Type:** a heavy rounded display face for headings and numbers ("Good morning,
  **Mehul**", "34 min 46 s"), for example *Archivo Black* or *Rubik Bold*. *Inter* for body.
- **Shapes:** 20–24 pt rounded cards in a 2-column grid, circular check toggles in the
  top-right, pill day chips.
- **Mascot:** the orange character belongs to Ronas IT. Use a similar *original*
  character (commissioned, or drawn and animated with Rive/Lottie) with states: idle,
  cheering (goal hit), flexing (PR), sleepy (sleep screen).

**Screens:**
1. **Onboarding** (indigo): "Build healthy habits with us" → Connect Fitbit → body stats
   for BMR → set goals.
2. **Home:** greeting, scrollable week strip, card grid (Steps · Workout · Active kcal ·
   Sleep · custom habits), tab bar (Home · **+** · Profile).
3. **Workout session** (lime): timer, set logger, rest timer, Finish.
4. **Detail screens:** Steps, Calories, Sleep, and Progress (lift charts).
5. **Sheets:** New habit / New exercise / Edit plan, styled like the design's form sheet.
6. **Profile:** streak calendar, badges, goals, data source settings.

## 4. Tech stack

- **SwiftUI, iOS 17+**, **SwiftData** for local storage, **Swift Charts**.
- **OAuth:** `ASWebAuthenticationSession` with PKCE. Tokens go in the **Keychain**.
- **Sync:** fetch on app open and pull-to-refresh, plus `BGAppRefreshTask` every ~1–2 h,
  with results cached in SwiftData so the app works offline.
- **WidgetKit** for the streak widget, **UserNotifications** for nudges.
- **No backend.** It's a single-user app, so everything stays on the phone.
- **What you'll need:** a Mac with Xcode 16+ to build and run it (this cloud session can
  write the code but can't compile iOS). A free Apple ID works for side-loading but
  apps expire after 7 days. The $99/yr developer account removes that limit and enables
  TestFlight.

```
Habbit/
  App/              HabbitApp.swift, RootView
  DesignSystem/     Colors, Typography, Card, DayStrip, CheckToggle, Mascot
  Features/
    Home/  Steps/  Calories/  Sleep/  Workout/  Streaks/  Habits/  Onboarding/  Profile/
  Data/
    Models/         SwiftData models
    Health/         HealthDataSource, GoogleHealthSource, HealthKitSource, MockSource
    Auth/           OAuthManager, KeychainStore
    Sync/           SyncService, BackgroundRefresh
  Domain/           OverloadEngine, StreakEngine, BMRCalculator, SleepInsights
  Widgets/
  Tests/            OverloadEngine, StreakEngine, BMR, sleep-insight tests
```

## 5. Build phases

| Phase | Scope | Done when |
|---|---|---|
| **0. Spike** (1–2 days) | Google Health API access test with your account | Today's steps, sleep and active kcal print from a real API call |
| **1. Skeleton and design system** | Xcode project, tokens, components, Home with `MockSource` | Home looks like the reference design with fake data |
| **2. Fitbit connection** | OAuth, `GoogleHealthSource`, sync and cache | Real steps, calories and sleep on Home |
| **3. Workouts** | Plan builder, live session, overload engine and charts | Can run a full session and get next-time suggestions |
| **4. Streaks** | Goal rules, Perfect Day, freezes, celebrations, week strip | Streak updates live, with confetti and the mascot |
| **5. Sleep and calorie detail** | Insights, charts, BMR | Plain-language sleep summary and an active vs total breakdown |
| **6. Polish** | Widget, notifications, custom habits, mascot animations | Daily-driver ready |

The domain logic (overload, streaks, BMR, sleep insights) is pure Swift with unit tests,
so it can be written and tested before the UI exists.

## 6. Open questions

1. Can your Google account get Google Health API access? Phase 0 answers this.
2. kg or lb, and your smallest plate increment (2.5 kg / 5 lb / 1.25 kg)?
3. Your current split (PPL, Upper/Lower, full-body) and how many days a week?
4. Default goals: steps, active kcal, sleep hours?
5. Should the streak need *both* a workout and steps, or is either one enough?
6. Do you have a Mac with Xcode, and an Apple Developer account?
