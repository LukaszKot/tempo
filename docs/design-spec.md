# Tempo — Design Specification

Nov 23, 2026 · @Łukasz Kot

## Overview

Tempo is a minimal Pomodoro timer with two clients: a native iPhone app and a responsive web app. It is completely free, with no paid tier. It has one user today but is built as a multi-user product: everyone signs in, and all data belongs to the signed-in account. The account syncs Sessions, tags, settings and the live timer between the two clients.

**Platforms and architecture:**

- **iPhone:** native SwiftUI app with an iPhone layout only. iPad runs it in compatibility mode. It is not published to the App Store. It is installed onto the user's own phone either from Xcode, with Developer Mode turned on, or through TestFlight internal testing. Signing uses a paid Apple Developer Program membership ($99 a year).
- **Web:** responsive web app for both desktop and mobile browser widths, built with React and MSAL and hosted on Azure Static Web Apps (Free).
- **Backend:** a .NET 11 web API that stores and syncs each user's data. It runs as a container app (scales to zero) in the shared Container Apps environment, with a free-tier database on the shared Azure SQL server. It lives in this `tempo` repo, together with the iPhone and web apps.
- **Infrastructure:** Tempo's own Terraform, including its app registration in the shared tenant, lives in this `tempo` repo and runs only through CI. This follows `platform`'s ADR-0001 and ADR-0002, under which `platform` holds only the shared tenant. Tempo reads the tenant's outputs (tenant ID, authority URL) from GitHub Actions Variables.
- **Identity:** Microsoft Entra External ID, using the shared portfolio tenant provisioned in `../platform` (`https://lukaszkotplatform.ciamlogin.com/`). Tempo registers as its own app inside that tenant. Users sign in with Apple or with email; there are no other sign-in methods. Self-service sign-up is disabled. Only accounts the owner creates can sign in, but the data model is fully multi-user.
- **Not supported:** Mac app, Apple Watch, Vision Pro, Android app.

There is no app or website blocking and no paid tier on any platform.

**Vocabulary used throughout this spec:**

- **Session**: one focus interval.
- **Short Break**: the break after most Sessions.
- **Long Break**: the break after a full set of Sessions.
- **Cycle**: N Sessions with their breaks, ending in a Long Break.
- **Phase**: any one step of the cycle, either a Session, a Short Break or a Long Break.
- **Thread**: one parallel line of work the user is juggling, such as an agent on the work laptop or tidying the room. A Session can move between threads; breaks belong to no thread.

## Feature inventory

Tempo has about 30 user-facing features. All of them are free. Platforms: i = iPhone app, W = web app.

| Area | Feature | Behaviour | Platforms |
| --- | --- | --- | --- |
| Account | Sign-in | Required, through Microsoft Entra External ID (the shared portfolio tenant), with Sign in with Apple or email. Each user sees only their own data. Data lives locally on each client and syncs through the Tempo backend. | i W |
| Timer | Interval timer | Session 25 min, Short Break 5, Long Break 30, Long Break after 4 Sessions | i W |
| Timer | Duration presets | Session: 15/20/25/30/45/50/60/90. Short: 5/10. Long: 15/20/30. | i W |
| Timer | Custom durations | Any value from 1 to 90 min. Change the current phase by pressing and holding the time on iPhone, or clicking it on the web. | i W |
| Timer | Custom cycle | 1 to 10 Sessions before a Long Break (default 4) | i W |
| Timer | Controls | Start, Stop (pause, keeps remaining time), Skip break, Restart cycle | i W |
| Timer | Previous / Next | Move through the cycle; the chosen phase stays pending | i W |
| Timer | Auto-start | Separate toggles for Start Breaks and Start Sessions automatically (both off by default) | i W |
| Timer | Session title | Names the current Session (breaks have no title); empty resets it to "Session"; shown in stats | i W |
| Timer | Timer Sync | Live timer state, durations, cycle and auto-start shared between iPhone and web through the account | i W |
| Focus | Metronome / ticking | Ticking during a Session with its own volume control | i W |
| Focus | Fullscreen break | Break fills the screen using browser fullscreen, with countdown, quote, editable title, Skip and Close *(proposed web version of Flow's Mac feature)* | W |
| Focus | Motivational quotes | Shown after each Session and in notifications | i W |
| Feedback | Notifications | End-of-Session and end-of-break alerts with separate sounds and volume; breaks can be skipped from the notification. On the web, browser notifications arrive only while a Tempo tab is open; there is no web push. | i W |
| Feedback | Celebration | Animation when a cycle step completes | i W |
| Tracking | Statistics chart | Count or Time by Day/Week/Month/Year; colour-coded by tag or by thread; time per thread; trend badge; Today button | i W |
| Tracking | Session list | Grouped by date; view detail, add manually, edit, delete; filter incomplete; shows interruptions and time per thread | i W |
| Tracking | Streaks | Days with at least 1 completed Session; rest days keep the streak but don't add to it; at least 1 Session every 3 days | i W |
| Tracking | Tags + presets | One tag per session, with title, colour and order. Defaults: Work, Personal, Study. A tag's preset overrides durations, cycle and auto-start. | i W |
| Tracking | Export | Completed Sessions as CSV or TXT, including tags, total time, interruptions and time per thread | i W |
| Tracking | Stats sync | Session history synced across iPhone and web through the account, separate from Timer Sync | i W |
| Integrations | Shortcuts / Siri | 17 actions: control, get/set phase, state, duration, title, tag | i |
| Surfaces | Browser tab | Remaining time and phase in the tab title; favicon tinted during breaks *(proposed, replaces Flow's menu bar)* | W |
| Surfaces | Widgets | Daily Quote, Daily Statistics, Tempo Icon (Lock Screen) | i |
| Surfaces | Live Activity | Lock Screen and Dynamic Island countdown with Start/Stop | i |
| Input | Keyboard | Space to start/stop, T for the next thread, 1–9 to pick a thread, Esc to close dialogs, ⌘, for settings. Works while the Tempo tab is focused; there are no global hotkeys. | W |
| Settings | Appearance | System / Light / Dark; dark and tinted app icons on iPhone | i W |
| Threads | Threads | 1 to 9 named parallel lines of work. Next thread cycles 1 → 2 → 3, or the user picks one directly. Switching doesn't affect the timer. No thread is active during breaks; the previous thread resumes after. Time per thread is tracked in stats. | i W |
| Timer | Progressive cycle | Starts at 15/5/15/1 (Session / Short Break / Long Break / cycle length). After each completed cycle, one setting that isn't at its maximum is picked at random and raised to its next preset value. Reset progression starts over. | i W |

## Design specification

The spec adapts the documented behaviour of Flow, the app Tempo replaces. Requirements use MUST/SHOULD. Visual details that the sources don't state are marked *(proposed)*.

### Design principles

1. **Unobtrusive.** On iPhone the timer stays visible on the Lock Screen and in the Dynamic Island. On the web it shows in the browser tab title.
2. **One primary action.** Start/Stop is the only prominent control. Advanced controls (Prev, Next, Skip, Restart) appear on hover on the web, and as secondary buttons on iPhone.
3. **Local first, synced through an account.** Each client works offline on its own local copy. A required account in the shared Entra External ID tenant syncs Sessions, tags, settings and the live timer through the Tempo backend. The backend scopes every record to the signed-in user.
4. **Native where it counts.** On iPhone, use platform surfaces (Lock Screen, widgets, Live Activity, Shortcuts) instead of custom chrome. On the web, use the tab title and browser notifications.

### Information architecture

- **Timer** (home)
  - Phase label and session title (edit it by tapping or clicking the title above the timer)
  - Countdown (press and hold on iPhone, or click on the web, to change the duration)
  - Cycle indicator: one dot per Session in the cycle, with the current position highlighted
  - Primary Start/Stop. Secondary Skip, Restart cycle, Previous/Next.
  - Tag selector, and a thread chip showing the active thread's name, with a Next thread button and a thread picker (hidden during breaks)
- **Statistics**: chart, totals, time per thread, streak, trend badge, Today button, Show All Sessions (list, detail, add, edit, delete), export, reset
- **Settings**
  - Sessions: durations, cycle, auto-start, presets, threads (count and names), progressive cycle (on/off, Reset progression)
  - General: notifications and sounds, metronome, appearance, Timer Sync, Manage Tags, account and sign-out
  - Web only: fullscreen break, tab title, keyboard shortcuts
- **Onboarding**: sign in through Entra External ID, then a "How it works" explainer. Permissions are requested in context: notifications on the first Start (iPhone and browser).

The iPhone app uses a tab bar with Timer, Statistics and Settings. The web app shows the same three sections as top navigation on wide screens and as a bottom tab bar on narrow screens *(proposed)*.

### Screens and surfaces

| Surface | Platform | Content | Key interactions |
| --- | --- | --- | --- |
| Sign-in | iPhone, web | The Entra External ID hosted sign-in page, offering Sign in with Apple and email | Shown on first launch and after sign-out; returns to the Timer when done |
| Timer screen | iPhone | Title, large countdown, cycle dots, Start/Stop, tag chip, thread chip; portrait and landscape | Tap Start; press and hold the time to change it; tag menu |
| Timer page | Web | Same structure as iPhone. Wide screens show the full layout with top navigation; narrow screens use one column with a bottom tab bar. | Space starts/stops, T moves to the next thread, 1–9 picks a thread, Esc closes dialogs; with a mouse, secondary controls appear on hover; on touch screens they are always shown |
| Browser tab | Web | Remaining time and phase in the tab title; favicon tinted during breaks *(proposed)* | Clicking the tab returns to Tempo |
| Fullscreen break | Web | Break countdown, quote, editable title, Skip, Close | Fills the screen using browser fullscreen; Esc exits *(proposed)* |
| Statistics | iPhone, web | Bar chart, colour-coded by tag or by thread; time-per-thread totals; Count/Time toggle; D/W/M/Y; streak card; sessions grouped by date | Filter by tag or thread; Today button; open a session to edit it and see its thread breakdown |
| Live Activity | iPhone | Phase, remaining time, progress, active thread; Dynamic Island compact and expanded | Start/Stop buttons |
| Widgets | iPhone | Daily Quote, Daily Statistics, Tempo Icon (Lock Screen) | Tap opens the app |

### Interaction rules

*(Diagram not exported: timer model · phase cycle and Pending/Running/Paused states. See the Claude Doc.)*

- **Phase end:** play the phase's sound and post a notification. Show a quote after a Session if enabled. Then either auto-start the next phase or leave it pending.
- **Long Break placement:** after Session number N, where N is the cycle length; otherwise a Short Break. Restart cycle goes back to a pending Session 1.
- **Skip:** during a break, skip starts the next Session (subject to auto-start). During a Session, skip records it as incomplete (inferred from the stats' incomplete-session filter).
- **Pause:** Stop keeps the remaining time. Each pause SHOULD be counted as an interruption on the session.
- **Presets:** choosing a tag that has a preset swaps in its durations, cycle and auto-start settings. Settings shows "Preset active". Deselecting the tag restores the previous settings.
- **Sleep and quit:** state persists when the app or tab closes. A running timer is computed from wall-clock time, so it stays correct after the phone locks or the tab goes to the background (`willCompleteAt`) *(proposed)*.
- **Sync:** with Timer Sync on, every state change is sent to the Tempo backend and the other client follows it. Stats sync independently.

### Threads

Threads let one person track several parallel lines of work under the same Session timer. Switching threads never touches the timer; it only shows what the user is working on right now.

- **Count:** the user sets how many threads they have, from 1 to 9 *(proposed limit)*. The default is 1, which hides the thread controls *(proposed)*. Lowering the count hides the highest-numbered threads. They are archived, not deleted, so their names stay in stats (proposed).
- **Names:** each thread has a name the user can edit at any time. An unnamed thread shows "Thread 1", "Thread 2" and so on.
- **Switching:** "Next thread" moves in order, 1 → 2 → 3, and wraps from the last thread back to 1 *(wrap is proposed)*. The user can also pick any thread directly.
- **Timer independence:** switching never starts, pauses, resets or skips the timer.
- **When switching works:** during a Session, whether it is pending, running or paused *(proposed)*.
- **Breaks:** during a Short or Long Break no thread is active and the thread controls are hidden. When the next Session starts, the thread that was active before the break becomes active again.
- **Sync:** the thread count, names and active thread sync between iPhone and web together with the live timer.
- **Statistics:** each Session records thread segments: which thread was active, from when to when. Only Running time counts, so paused time and breaks never go to a thread. Stats show time per thread for the chosen period, can colour the chart by thread instead of by tag, and can filter by thread. A Session's detail view lists its thread breakdown.

Example: a user has 3 threads, "Work laptop agent", "Private laptop agent" and "Spare time". They start a Session on thread 1. When the work agent is busy, they tap Next and move to thread 2. When both agents are busy, they tap Next again and tidy the room on thread 3. During the break no thread is active; after it, the Session continues on thread 3.

### Progressive cycle

Progressive cycle starts the timer at its lowest values and lengthens one setting at random after every completed cycle, until all four reach their maximum. It uses only the preset values below and ignores custom values.

| Setting | Ladder | Steps to max |
| --- | --- | --- |
| Session duration | 15 → 20 → 25 → 30 → 45 → 50 → 60 → 90 min | 7 |
| Short Break | 5 → 10 min | 1 |
| Long Break | 15 → 20 → 30 min | 2 |
| Cycle length | 1 → 2 → … → 10 Sessions | 9 |

1. **Start:** turning progressive cycle on, or pressing Reset progression, sets 15 / 5 / 15 / 1 (Session / Short Break / Long Break / cycle length).
2. **Step:** when a cycle completes, collect the N settings that are not yet at their maximum. Pick a random whole number from 1 to N, uniformly, and raise that setting to its next ladder value.
3. **Max:** a setting at its maximum drops out of the draw. When N is 0, progression stops and every setting stays at its maximum.
4. **Timing:** the new value applies from the next cycle, never mid-cycle.

All four ladders reach their maximum after 19 completed cycles.

**Rules:**

- **Completed cycle:** a cycle counts when its Long Break ends or is skipped. Restart cycle doesn't count.
- **Locked settings:** while progressive cycle is on, the four settings can't be edited by hand, and tag presets don't change them.
- **Turning it off:** the current values stay as the user's manual settings. For example, 30 / 5 / 20 / 6 remains after turning it off.
- **Turning it on again:** it always starts from 15 / 5 / 15 / 1.
- **Sync:** the current ladder positions sync between iPhone and web like other settings.

### Visual language

From the sources: minimal, "unobtrusive" UI. System, Light and Dark appearances; dark and tinted app icons. Liquid Glass styling on iOS 26. Tag colours drive the stats chart. A celebration animation marks completion.

*(Proposed for Tempo, not taken from Flow):*

- The countdown is the hero, in monospaced digits, about 64 to 96 pt.
- One accent for Sessions and a second hue for breaks; everything else is neutral.
- On iPhone, use SF Pro and the system materials. On the web, use the system font stack with the same colour tokens.
- Keep motion under 300 ms, except the completion celebration.

## Data model & settings reference

Six entities cover everything Tempo persists. TimerStatus is the live timer shape that both clients and the Shortcuts actions share. Everything else is *(proposed)*.

| Entity | Fields | Notes |
| --- | --- | --- |
| Account | id (the Entra External ID user object ID), email, displayName, createdAt | Created on first sign-in. Every other record carries this account id, and the API returns only the caller's own records. |
| TimerStatus (live, synced with Timer Sync) | `phase` (session, shortBreak, longBreak), `state` (pending, running, paused), `remainingSeconds`, `title`, `tag`, `cycleProgress`, `cycleLength`, activeThread, `willCompleteAt` (ISO 8601, only while running) | Tempo's status snapshot; one per account |
| Session (synced through the account) | id, title, tagId, startAt, endAt, plannedSeconds, focusedSeconds, interruptions, completed, source (timer or manual), threadSegments \[{threadId, startAt, endAt}\] | Records focus Sessions only, not breaks. Thread segments cover Running time only. Only completed Sessions go to export. |
| Tag | id, title, color, order, preset {sessionMin, shortMin, longMin, cycleLength, autoStartBreaks, autoStartSessions} or null | Defaults: Work, Personal, Study, created for each new account |
| Settings | the table below | One set per account. Keys for one platform are ignored on the other. |
| Thread | id, position (1 to 9), name | One row per thread, per account, plus an archived flag. Threads removed by lowering the count are archived so stats keep their names. activeThread in TimerStatus points at one of them and stays set through breaks, so the same thread resumes afterwards. |

| Setting | Default | Range / options |
| --- | --- | --- |
| Session duration | 25 min | 15/20/30/45/50/60/90; 1 to 90 custom |
| Short Break | 5 min | 5/10; custom |
| Long Break | 30 min | 15/20/30; custom |
| Cycle length | 4 | 1 to 10 |
| Start Breaks automatically | off | bool |
| Start Sessions automatically | off | bool |
| Session Complete / Break Complete sound | not stated | sound list + volume |
| Motivational quotes | not stated | bool |
| Metronome | not stated | bool + volume |
| Appearance | not stated | System / Light / Dark |
| Timer Sync | off | bool |
| Tab title countdown (web) | on *(proposed)* | bool |
| Fullscreen at start of break (web) | off | bool |
| Keyboard shortcuts (web) | on | Space start/stop, T next thread, 1–9 pick thread, Esc close, ⌘, settings |
| Thread count | 1 (proposed) | 1 to 9 (proposed) |
| Thread names | Thread 1, Thread 2, … | free text per thread |
| Progressive cycle | off | bool; when on, also stores the ladder position of each of the 4 settings |

**Automation (iPhone only):**

- **Shortcuts / Siri:** Start/Resume, Pause, Toggle, Skip, Previous, Reset Cycle, Get Phase, Get Remaining Seconds, Get/Set Title, Get/Set Duration, Get/Set Tag, Get All Tags, Get State, Open App, Next Thread, Set Thread, Get Thread (thread actions proposed).
- The web app has no scripting API. Flow's AppleScript, CLI, Alfred and Raycast integrations were Mac-only and are dropped.

Source: [docs.flow.app automation](https://docs.flow.app/automation/).

## Gaps, open questions and recommendations for Tempo

Flow's weak spots are a narrow ecosystem and shallow insights. Those are where Tempo can differ. Tempo already differs by running on the web and dropping blocking. Its core loop, an always-visible timer driving a strict Session and break cycle, is worth copying almost exactly.

**What Flow's sources don't say (check the app or screenshots before treating these as fact):**

- Exact layout, typography, colours and spacing of each screen. No design source was available; the Medium review returned 403.
- How a Mac app or site block is enforced, and what the blocked UI looks like.
- How Timer Sync resolves conflicts, and how presets interact with Timer Sync.
- The calendar event title and fields; how mindful minutes are calculated; the CSV column layout.
- Defaults for quotes, metronome, appearance and Mac window options; whether Health logging is Pro.

**Where Flow is weak (Tempo opportunities):**

- Apple-only, with no web, Windows, Android or team features.
- One tag per session; no tasks or projects, and no planning against an estimate.
- Limited insights: count and time only, with no focus-quality or time-of-day analysis.
- Reviewers call it "more basic than other Mac options".

**Open questions for you:**

- [x] Platforms: decided as a native iPhone app plus a responsive web app for desktop and mobile widths.

  Sync is through a required account and backend. There is no blocking.
- [x] Pricing: decided as completely free with no paid tier, but built as a multi-user product.
- [x] Identity: decided as Entra External ID from the shared portfolio tenant, with Sign in with Apple and email only.
- [x] Code and infrastructure: the apps, the .NET backend and Tempo's Terraform all live in the `tempo` repo, following `platform`'s ADRs. `platform` provides only the shared Entra tenant.
- [x] Distribution: the iPhone app is installed from Xcode with Developer Mode on, or through TestFlight internal testing. It is not on the App Store.
- [x] Web notifications: decided that they arrive only while a Tempo tab is open, with no web push.
- [x] Google Calendar: not needed. It was only suggested as a web stand-in for Apple Calendar logging.
- [x] Apple Developer membership: decided to use the paid Apple Developer Program ($99 a year). It covers Sign in with Apple (the app's capability, plus the Service ID and key Entra needs) and installs that don't expire after 7 days.
- [x] Apple Calendar and Apple Health: dropped, because they add nothing for this app.
- [ ] Is this a clone at feature parity, or a differentiated product? If differentiated, which of the weak spots above? (Deferred: you'll answer later.)
- [x] Scope: no MVP split. Every feature in this spec is in scope.
- [x] Flow screenshots: not wanted. Tempo's visual language comes from the proposed guidance in this spec, not from copying Flow.
- [x] Threads in statistics: decided. Sessions record time per thread, and stats show it.
- [x] Threads details: should these proposed values stand? Up to 9 threads, a default of 1, Next wrapping from the last thread back to 1, and switching allowed while a Session is paused or pending.
- [x] Progressive cycle length: decided. The ladder goes from 1 to 10 in steps of 1, as in the ladder table.
- [x] Progressive cycle rules: decided. They are listed under Progressive cycle > Rules.
- [x] Web stack: decided as React and MSAL on Azure Static Web Apps (Free). The API is a .NET 11 container app in the shared environment, with a free-tier database on the shared SQL server.
- [x] Sign-up: decided as disabled. The data model is still multi-user.
