# NeverHateThis

An offline-first Android app that turns a phone into a playable guitar, bass and vocal rig — built for cheap inputs, speaker output, and honest latency.

Status: design phase. No code yet.

## Documents

- [Product requirements](docs/PRD.md)
- [Design](docs/design/2026-11-13-nht-design.md)

## Repository layout

| Path | Purpose |
|---|---|
| `android/` | Kotlin + Jetpack Compose app |
| `core/` | Rust DSP engine, chain, content and audio I/O |
| `content/` | Starter content and the content build/sign pipeline |
| `tools/` | Build, signing and release scripts |
| `docs/` | Product and design documents |

## License

Code: Apache-2.0 (planned, see design decision 1). Content is licensed per asset.
