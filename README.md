# bombadil-compose

Write [Bombadil](https://github.com/antithesishq/bombadil) external drivers
for Jetpack Compose apps in Kotlin. Not a first-party Bombadil project.

A driver launches the app, reports its state as JSON, and performs the
actions Bombadil picks, speaking Bombadil's external driver protocol
(version 0, see `lib/bombadil-rpc/PROTOCOL.md` in the Bombadil repo) on
stdin and stdout. Implement `BombadilDriver` and call `runDriver`; log
to stderr, since stdout carries the protocol.

## Development

```sh
nix develop   # or direnv allow
gradle test installDist
```

The example counter driver has a bug that its specification catches:

```sh
bombadil external test --specification examples/counter.spec.ts -- build/install/bombadil-compose/bin/bombadil-compose
```

## Android apps over adb

`me.uzaaft.bombadil.android.AdbAppDriver` drives an installed Android
app (Compose or not) from the host through `adb`, and
`bin/bombadil-android --package <id> [--apk <file>] [--agent <apk>] [--grant <permission>]...`
runs it as a Bombadil driver. Each session installs the APK (if given),
clears the app's data, grants the permissions, disables animations and
launches the app.

### The on-device helper

The driver reads the UI and acts through a small instrumentation APK,
`android-helper/` (Kotlin against `android.jar`, about 25 KB). The
driver carries it as a resource, installs it on each device the first
time, and starts it with `adb shell am instrument -w`. The helper keeps
one `UiAutomation` connection open, sees every interactive window
(including system dialogs such as crash and ANR dialogs), and answers
line-delimited JSON requests on the abstract socket `bombadil-helper`,
which the driver reaches through `adb forward`. It outlives sessions:
later sessions reuse it as long as it runs the same APK. `AndroidHelper`
documents its commands.

Gradle builds the APK without the Android Gradle plugin: the `helper`
source set compiles the Kotlin against `android.jar` (with `-no-jdk`, so
only Android's Java library is visible), and `android-helper/build.sh`
runs R8, `aapt2`, `zipalign` and `apksigner` from the dev shell's SDK.
R8 keeps only the parts of the Kotlin standard library the helper uses
(`android-helper/rules.pro`); dexing the whole library with `d8` would
add about 2 MB. The signing key is generated per build directory, and
the driver reinstalls the helper when a new key makes the update fail.

While the helper runs, nothing else on the device can use UiAutomation
(`uiautomator dump`, Espresso or other UI tests): Android allows one
connection at a time.

### The in-process agent

With `--agent <apk>`, the driver also runs an agent inside the app's
process, the way Espresso and Compose's UI tests do: `android-agent/` is
an instrumentation whose target is the app, so Android loads it into the
app's process and class loader, and only if it carries the app's
signature. `gradle agentDex` compiles it (Kotlin, against `android.jar`
and the Compose version the app ships) into a dex with only its own
classes, since the app provides Compose and the Kotlin standard library;
`android-agent/package.sh <dex dir> <app package> <keystore> <apk>`
packages and signs it for one app (for debug builds, with
`~/.android/debug.keystore`). Each session starts the agent with
`am instrument`, which starts the app's process with the agent in it,
before the app is launched, so the agent sees every Compose root the app
creates (`ViewRootForTest.onViewCreatedCallback`).

While the app's own windows are in front, the agent:

- reads them directly on the main thread: Compose semantics trees and
  plain Views, in the helper's JSON shape, with ids hashed the same way.
  It follows Compose's accessibility layer, so that states look the same
  as through the helper: unmerged nodes with touch bounds; no unplaced,
  transparent or hidden nodes, nor nodes covered by important nodes in
  front (a screen still composed behind the one on top); test tags as
  resource ids only under `testTagsAsResourceId`; and no windows behind a
  modal one (a dialog). On Now in Android's screens the agent's nodes,
  bounds and flags match the helper's exactly. It takes 5ms instead of
  25ms, and no frame has to reach the screen first.
- acts with semantics actions (`OnClick`, `ScrollBy`, `SetText`, ...) or
  View methods, falling back to touches injected into the app's own
  windows. Back is a key event the app has handled by the time the
  agent replies. A node in a window covered by another of the app's
  windows (under a dialog) gets a real touch instead, which the window
  in front receives, as it would from a user.
- reports the state once the main thread has nothing left to do, from
  Compose's and Android's own signals, as Compose's UI tests decide
  idleness: no snapshot changes waiting to be applied, no `Recomposer`
  with pending work (recompositions, or animations waiting for a frame),
  no Compose root with a pending measure or layout, no message due in the
  main looper's queue (which includes pending View traversals), no work
  in the coroutine scheduler behind the IO and Default dispatchers (tasks
  queued or running, read from its description) and no running Room
  worker: most responses to input reach the main thread through them
  (ViewModel, repository, database, flow). The app's front window must
  also have input focus, which moves a frame or two after a dialog opens
  or closes. Each check runs on the main thread; while the app is busy,
  the next one runs when the queue goes idle or, for work done in frames,
  after the next frame. No sleeps. One cap (2s, `settled: false`) covers
  apps that never go idle, such as an endless animation, and 300ms
  without focus means something outside the app is in front.

The agent also keeps tests inside the app, the way Espresso-Intents
stubs intents: a blocking `ActivityMonitor`, asked about every
`startActivity` before it happens, refuses those that would leave the
app (an explicit intent to another package, or an implicit one that
nothing in the app handles, such as a link for the browser or a share
sheet) with a cancelled result, so the other app never launches. The
app's own activities and permission requests (a system dialog that
belongs to the app's flow) go through. Each state lists the hand-offs
blocked during the action that produced it in `blockedIntents`:
`[{action, data, package}]`, `package` being where it would have gone,
or null when the app can't see that (Android 11's package visibility
hides the browser from apps that don't declare it).
`--allow-external-activities` (`NIA_ALLOW_EXTERNAL=1` in the experiment)
turns this off. The agent's state also includes the keyboard, as an
`inputMethod` window without nodes, when it's shown over the app (API 30
and later), as the helper's does: Back closes it before anything else.

Whatever leaves the app's windows goes to the helper. If an action makes
the app start an activity (an `ActivityMonitor` counts every
`startActivity`) or finish one, the helper waits until the task stack's
top activity has changed and its window is shown and quiet, and then the
agent takes over again if the new activity is the app's. Home, actions in
other apps (the browser), and states where something outside the app
has input focus (a system dialog: the agent waits up to 300ms for focus
to come back to the app after one of its own dialogs closes) use the
helper alone. If the app dies, the session goes on with the helper.

Settling is still best effort, as in Bombadil's browser driver: work the
agent doesn't see (a delayed coroutine, an image being decoded) shows up
in a later state.

### State

The state is the focused window and the pruned accessibility tree of
every interactive window, top window first:

```json
{"package": "<package of the active window>", "activity": "<last activity shown>",
 "focus": "<active window's title>",
 "lastAction": <the action just applied, or null>,
 "windows": [{"type": "application", "package": "...", "title": "...", "active": true,
              "focused": true, "layer": 2, "bounds": [0, 0, 1080, 2400],
              "nodes": [{"id": "5c1f3a0e", "resourceId": "...", "class": "View",
                         "text": "...", "desc": "...", "label": "...",
                         "clickable": true, "longClickable": false, "scrollable": false,
                         "enabled": true, "checkable": false, "checked": false,
                         "selected": false, "focused": false, "editable": false,
                         "bounds": [left, top, right, bottom], "children": [...]}]}]}
```

Window types are `application`, `inputMethod`, `system` and
`accessibilityOverlay`. Compose test tags show up as `resourceId` when
the app sets `testTagsAsResourceId`. `label` is the node's text, else its
content description, else its descendants' labels: Compose puts a
clickable row's text on its children.

A node's `id` hashes its window's type and package, the nearest ancestor
with a resource id, its own resource id, class and label, and the label
of its nearest clickable ancestor (so the identical "Follow" buttons of
different rows differ). It doesn't depend on position or on what else is
on screen, so the same node has the same id after scrolling, in another
session, and in replayed or shrunk traces. Nodes that still look alike
get `-1`, `-2`, ... in depth-first order.

Settling after an action is best effort, as in Bombadil's browser
driver: the driver waits for the app to respond, not for it to stop
changing. If something is still animating, the next state shows the
rest, and temporal properties (`eventually`, `next`) handle states in
between. Before each action the helper clears the frame statistics of
the app's windows and the keyboard (`UiAutomation.clearWindowContentFrameStats`).
After the action it reports the tree once one of these has happened:

- a frame: a watched window has presented a frame after the action, and
  neither a further frame nor an accessibility event followed for 33ms
  (two refresh periods at 60Hz). Frames catch content changes that send
  no accessibility events, which is most of Compose's. One refresh
  period, "the next frame after the last one", is too short on the
  emulator: Compose's frames after a tap come 10-150ms apart, and 16% of
  states were caught mid-transition, against 10% with two periods.
- an activity change: the top activity of the task stack (read with the
  shell's permissions) differs from the one before the action, and once
  a window of its package is active, events and frames paused for 33ms.
  The task stack changes within milliseconds of an activity starting or
  finishing, while the window change is announced up to 200ms later.
- a window change: a dialog or keyboard came or went, and accessibility
  events then paused for 33ms with some window active.

Accessibility events wake the wait up; between them the helper polls the
frame statistics and the top activity once per refresh period. The
remaining timeouts are fallbacks, not the mechanism. After 500ms with no
frame, activity or window change, the action changed nothing visible,
and the tree is reported as it is. A window change that never goes quiet
is reported after 1s, and an endless animation after 2s. Back and Home in
another app (the browser an article opens in) wait for a 250ms pause:
WebView handles Back, presents a frame and finishes its activity some
150ms later, with no signal in between, and reporting the browser still
in front made the spec's next Back leave the app. The first state of a
session is the exception: it waits until two trees in a row agree, so
that every session starts from the same state.

What best effort costs, measured by fetching the tree again a second
after each state: in 3-minute tests about 10% of states had changed, both
with the agent (9.4%) and without (9.8%), most of them right after an
article opened in the browser, whose page was still loading, and a few
dialogs still loading their content. Waiting only for the first frame
after an action left 21% of states changed, and the agent without its
background-worker check 30%.

### Actions

Actions address nodes by `node` id or, for convenience, by any of
`resourceId`, `text`, `desc`, `label` and `class`, plus `index` among the
matches in depth-first order. Other keys are ignored, so specifications
can add a `label` to make traces readable:

```json
{"Tap": {"node": "5c1f3a0e", "label": "Interests"}}   {"Tap": {"x": 540, "y": 1200}}
{"LongPress": {"resourceId": "card", "index": 2}}
{"TypeText": {"text": "hello", "into": {"resourceId": "searchTextField"}}}
{"Scroll": {"node": "9e0b11c2", "direction": "down"}}
{"Scroll": {"direction": "down", "percent": 60}}
{"Swipe": {"from": [100, 1500], "to": [100, 500]}}
"Back"   "Home"   "Enter"
```

`Tap`, `LongPress` and `Scroll` perform the node's accessibility action
(click, long click, scroll up/down/left/right or forward/backward) and
fall back to injected touches at the node's centre. `TypeText` appends to
the focused field (or the `into` node) with the set-text action. `Scroll`
without a node scrolls the app's largest scrollable node. With `percent`,
it drags that share of the node instead. Drags and swipes plan every
event's time and hold still before lifting, so they don't fling and
always scroll the same distance. `Back` and `Home` are global actions.

An action on a node that isn't there is skipped (logged to stderr), so
replayed and shrunk traces don't fail on nodes that are gone.

The driver picks the first device in `BOMBADIL_ANDROID_SERIALS`
(comma-separated, default `ANDROID_SERIAL`, then `emulator-5554`) that no
other driver process on the host holds, so `--parallelism N` works with
N devices. With fewer devices than workers, the extra drivers fail with
"all devices are in use". `BOMBADIL_ANDROID_TIMINGS=<file>` makes the
driver append each step's timings to a file, one JSON line per step.

## Experiment: Now in Android

`experiments/nowinandroid/` tests
[Now in Android](https://github.com/android/nowinandroid) with its demo
data (no backend), pinned to commit `2982e63c` in `env.sh`. By default
it builds the `demoBenchmarkRelease` variant: R8-minified and not
debuggable like a release build, but signed with the debug key, so that
the in-process agent can be signed to match. `NIA_VARIANT=debug` tests
`demoDebug` instead. The agent is dexed without its dependencies and
uses the app's Compose and Kotlin classes, so `agent-keep.pro` (added to
the app's R8 rules through `agent-keep.init.gradle`, without touching
the checkout) keeps the classes it uses under their own names; it costs
the APK 130 KB (4.63 to 4.77 MB). Everything runs in the dev shell, which
provides the Android SDK, platform 37, build tools, the emulator and an
API 35 x86_64 image. The emulator needs KVM.

```sh
nix develop
experiments/nowinandroid/setup.sh    # clone, build, boot an emulator; ~25s once built
```

`setup.sh` clones the app to `~/repositories/github.com/android/nowinandroid`
(`NIA_DIR` overrides), builds it with its own Gradle wrapper (nixpkgs'
Gradle is too old for its AGP), builds this project, creates an AVD under
`~/.local/share/bombadil-compose/avd` and boots it headless, or reuses
emulators that are already running. Emulators outlive the driver:
a cold boot takes about 20 seconds, which every run would pay otherwise;
each session clears the app's data instead.
`EMULATORS=3 setup.sh` boots three, for parallel fuzzing
(`--parallelism 3`), and `setup.sh stop` kills them. All instances run
`-read-only` so they can share the AVD; an emulator of that AVD that you
started without `-read-only` stops others from booting.

The emulators render on the host's GPU when there is one (`NIA_GPU=auto`,
the default; `NIA_GPU=swiftshader` forces software rendering and
`NIA_GPU=host` the GPU). Headless, `emulator -gpu host` fails: its host
renderer opens an X display through GLX ("Failed to open display"), and
its launcher puts the bundled SwiftShader GL libraries first on
`LD_LIBRARY_PATH`, so even with `ANDROID_EGL_ON_EGL=1` it renders on
SwiftShader. `setup.sh` therefore starts the emulator's qemu binary
directly, with `ANDROID_EGL_ON_EGL=1` (EGL needs no display) and a
library path of the emulator's own libraries, libglvnd (`GLVND_LIB`, from
the dev shell) and the host's drivers in `/run/opengl-driver` (NixOS;
`NIA_GL_DRIVERS` overrides). libglvnd then picks NVIDIA's EGL, and the
guest reports `GLES: ... (NVIDIA GeForce RTX 5090/PCIe/SSE2), OpenGL ES
3.1` in `dumpsys SurfaceFlinger`; nvidia-smi lists the qemu processes. If
the GPU emulator exits during boot, `setup.sh` boots it again with
SwiftShader. A GPU boot takes about 14 seconds, against 19.

```sh
B=bombadil   # a Bombadil build with `bombadil external`
S=experiments/nowinandroid

$B external test --specification $S/spec.ts --time-limit 3m -- $S/driver.sh
$B external fuzz --specification $S/spec.ts --time-limit-fuzz 3m --time-limit-run 30s -- $S/driver.sh
```

`spec.ts` checks that the app stays in front (the agent blocks the
browser an article would open in), that the only hand-offs it attempts
are web links and that tapping a news card attempts one, that it never
shows a crash or ANR dialog, shows the
whole bottom navigation on top-level screens (within 2 seconds, since
states are best effort), and leaves a topic's
screen on Back (unless the keyboard is up). The system's permission
dialog counts as in the app: the driver grants notifications, but without
`--grant` the app asks, and the agent lets that request through. It taps visible clickable nodes, scrolls, types into the
focused search field, and presses Back except where Back would close the
app.

`follow.spec.ts` claims no interest is ever followed on the Interests
tab, which the app violates as soon as one is followed there (at least
two actions: open Interests, tap a follow button). Use it to see a
violation found and shrunk:

```sh
$B external test --specification $S/follow.spec.ts --exit-on-violation --output-path /tmp/follow -- $S/driver.sh
$B external shrink --specification $S/follow.spec.ts --trace /tmp/follow -- $S/driver.sh
```

A step (action to state) takes about 70ms (median; p90 82ms) with the
agent blocking hand-offs, on a GPU emulator with the benchmarkRelease
build: a 3-minute test made 2656 states, or 14.8 per second, all of them
in the app. Each emulator then uses about 0.8 host cores. 98% of steps
stay inside the agent: the action takes about 1ms, the app is idle about
60ms later (Compose's Recomposer has work for a number of 60Hz frames
after a tap, however fast each frame renders), and the tree takes 1ms to
build. Before the agent blocked hand-offs, a tenth of the states were in
the browser an article opened in: 9.1 states per second, with the
browser (loading real pages) using half of the emulator's CPU and each
emulator about 2.4 host cores. Earlier 3-minute tests, letting the
browser open, states per second (step median, p90):

| | demoDebug | demoBenchmarkRelease |
|---|---|---|
| SwiftShader | 5.8 (105ms, 488ms) | 6.7 (86ms, 471ms) |
| GPU (RTX 5090) | 8.2 (82ms, 223ms) | 9.1 (78ms, 219ms) |

Fuzzing scales with emulators until the host's CPUs run out (3 minutes
each, `--time-limit-run 30s`):

| Emulators | States/s total | Per worker | Host CPU busy (median, p90) | GPU busy (median) |
|---|---|---|---|---|
| 4 | 57 | 14.3 | 15%, 28% | 8% |
| 8 | 105 | 13.1 | 35%, 64% | 20% |
| 12 | 127 | 10.6 | 74%, 97% | 29% |

Each emulator takes 0.9 to 1.2 host cores while fuzzing, 4GB of guest
RAM and 0.6GB of GPU memory. States that are still changing (a tree
read again a second later differs) were under 1% with every combination
measured. A session takes about 3.6 seconds to start, with or without
the agent.

Without the agent (`NIA_NO_AGENT=1`), steps take 0.17 seconds (4.3 states
per second): the first frame reaches the screen about 50ms after the
action and the helper waits for a 33ms pause in frames, then builds the
tree in about 25ms. Earlier versions managed 3.4 states per second with a
fixed pause after Back, 1.5 when waiting for quiet periods and 0.22 with
`uiautomator dump`. Bombadil's default `--time-limit-run` of 10s allows
about 60 actions per fuzz run, so raise it for longer runs.
