# bombadil-compose

Third-party Kotlin / Jetpack Compose driver for [Bombadil](https://github.com/antithesishq/bombadil).

![Now in Android being fuzzed (9-second looping demo)](docs/demo.gif)

Nine seconds of real fuzzing in Now in Android.

Requires x86_64 Linux, KVM, Nix flakes, and a Bombadil build with `external` support.

```sh
nix develop
experiments/nowinandroid/setup.sh
bombadil external test --time-limit 30s \
  --specification experiments/nowinandroid/spec.ts \
  -- experiments/nowinandroid/driver.sh
```

Setup builds the app and driver and boots an emulator. [Specification](experiments/nowinandroid/spec.ts).
Stop emulators with `experiments/nowinandroid/setup.sh stop`.
