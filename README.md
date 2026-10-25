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

The state is the focused window and the visible UI tree from
`uiautomator dump`, with layout-only nodes pruned:

```json
{"package": "...", "activity": "...", "focus": "<window title>",
 "lastAction": <the action just applied, or null>,
 "nodes": [{"resourceId": "...", "class": "View", "text": "...", "desc": "...",
            "label": "...", "clickable": true, "scrollable": false, "enabled": true,
            "checkable": false, "checked": false, "selected": false, "focused": false,
            "longClickable": false, "bounds": [left, top, right, bottom], "children": [...]}]}
```

Compose test tags show up as `resourceId` when the app sets
`testTagsAsResourceId`. `label` is the node's text, else its content
description, else its descendants' labels: Compose puts a clickable
row's text on its children. A state is reported once two consecutive
dumps agree (at most four dumps).

Actions select nodes by any of `resourceId`, `text`, `desc`, `label` and
`class`, plus `index` among the matches in depth-first order:

```json
{"Tap": {"label": "Interests"}}              {"Tap": {"x": 540, "y": 1200}}
{"LongPress": {"resourceId": "card", "index": 2}}
{"TypeText": {"text": "hello", "into": {"resourceId": "searchTextField"}}}
{"Scroll": {"direction": "down", "percent": 60}}   (largest scrollable node, or add a selector)
{"Swipe": {"from": [100, 1500], "to": [100, 500]}}
"Back"   "Home"   "Enter"
```

A selector that matches nothing is skipped (logged to stderr), so
replayed and shrunk traces don't fail on nodes that aren't there.

The driver picks the first device in `BOMBADIL_ANDROID_SERIALS`
(comma-separated, default `ANDROID_SERIAL`, then `emulator-5554`) that no
other driver process on the host holds, so `--parallelism N` works with
N devices. With fewer devices than workers, the extra drivers fail with
"all devices are in use".

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
emulators that are already running. Emulators outlive the driver, since
booting one takes longer than Bombadil waits for a driver's `hello`.
`EMULATORS=3 setup.sh` boots three, for parallel fuzzing, and
`setup.sh stop` kills them.

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

`follow.spec.ts` claims no topic is ever followed, which the app
violates as soon as a topic toggle is tapped. Use it to see a violation
found and shrunk:

```sh
$B external test --specification $S/follow.spec.ts --exit-on-violation --output-path /tmp/follow -- $S/driver.sh
$B external shrink --specification $S/follow.spec.ts --trace /tmp/follow -- $S/driver.sh
```

Each action takes about 4 seconds, two `uiautomator dump`s of about 2
seconds each, and a session takes about 7 seconds to start. Keep
`--time-limit-run` well above that when fuzzing: the default of 10s
allows only an action or two.
