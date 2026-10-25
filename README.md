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
`bin/bombadil-android --package <id> [--apk <file>] [--grant <permission>]...`
runs it as a Bombadil driver. Each session installs the APK (if given),
clears the app's data, grants the permissions, disables animations and
launches the app.

### The on-device helper

The driver reads the UI and acts through a small instrumentation APK,
`android-helper/` (plain Java against `android.jar`, about 20 KB). The
driver carries it as a resource, installs it on each device the first
time, and starts it with `adb shell am instrument -w`. The helper keeps
one `UiAutomation` connection open, sees every interactive window
(including system dialogs such as crash and ANR dialogs), and answers
line-delimited JSON requests on the abstract socket `bombadil-helper`,
which the driver reaches through `adb forward`. It outlives sessions:
later sessions reuse it as long as it runs the same APK. `AndroidHelper`
documents its commands.

Gradle builds the APK without the Android Gradle plugin: `compileHelper`
compiles the Java, and `android-helper/build.sh` runs `d8`, `aapt2`,
`zipalign` and `apksigner` from the dev shell's SDK. The signing key is
generated per build directory, and the driver reinstalls the helper when
a new key makes the update fail.

While the helper runs, nothing else on the device can use UiAutomation
(`uiautomator dump`, Espresso or other UI tests): Android allows one
connection at a time.

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

A state is reported once two trees in a row agree, each fetched after
200ms without accessibility events. Quiet alone isn't enough, because
Compose doesn't send events for most changes.

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
[Now in Android](https://github.com/android/nowinandroid)'s `demoDebug`
build (local demo data, no backend), pinned to commit `2982e63c` in
`env.sh`. Everything runs in the dev shell, which provides the Android
SDK, platform 37, build tools, the emulator and an API 35 x86_64 image.
The emulator needs KVM.

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

```sh
B=bombadil   # a Bombadil build with `bombadil external`
S=experiments/nowinandroid

$B external test --specification $S/spec.ts --time-limit 3m -- $S/driver.sh
$B external fuzz --specification $S/spec.ts --time-limit-fuzz 3m --time-limit-run 40s -- $S/driver.sh
```

`spec.ts` checks that the app stays in front (articles open in the
browser, which counts), never shows a crash or ANR dialog, shows the
whole bottom navigation on top-level screens, and leaves a topic's
screen on Back. It taps visible clickable nodes, scrolls, types into the
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

Each action takes about 4 seconds, two `uiautomator dump`s of about 2
seconds each, and a session takes about 7 seconds to start. Keep
`--time-limit-run` well above that when fuzzing: the default of 10s
allows only an action or two.
