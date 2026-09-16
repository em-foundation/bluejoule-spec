# BlueJoule

BlueJoule is a specification for measuring the energy cost of Bluetooth Low Energy activity, not a
tool. Any instrument and any software that satisfy the specification can produce a conformant
capture. EM•Scope is the reference implementation; it is not the definition.

- **[SPEC.md](SPEC.md)** is the specification.
- **[CHANGELOG.md](CHANGELOG.md)** records the decisions behind each version.

This repository holds only what is invariant across every BlueJoule benchmark. A benchmark
repository holds one prescribed activity and its captures:

- [bluejoule-adv](https://github.com/em-foundation/bluejoule-adv), advertising
- [bluejoule-gatt](https://github.com/em-foundation/bluejoule-gatt), GATT

Declaration vocabulary for platforms, power sources, analyzers and activities lives in
[PEDS](https://github.com/em-foundation/PEDS).

## Branches

`main` holds the latest stable version. An `RC-<release>` branch holds the working draft for an
upcoming release, and the pull request from that branch to `main` is where the version is discussed.
