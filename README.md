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
