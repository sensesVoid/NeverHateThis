# NeverHateThis

[![CI](https://github.com/sensesVoid/NeverHateThis/actions/workflows/ci.yml/badge.svg)](https://github.com/sensesVoid/NeverHateThis/actions/workflows/ci.yml)

An offline-first Android app that turns a phone into a playable guitar, bass and vocal rig — built for cheap inputs, speaker output, and honest latency.

Status: design phase. No code yet. The app will ship as a signed APK on GitHub Releases.

## Documents

- [Product requirements](docs/PRD.md)
- [Design](docs/design/2026-11-13-nht-design.md)
- [Amp voicing references](docs/research/amp-voicing-references.md)
- [Releasing](docs/RELEASING.md)

## Repository layout

| Path | Purpose |
|---|---|
| `android/` | Kotlin + Jetpack Compose app |
| `core/` | Rust DSP engine, chain, content and audio I/O |
| `content/` | Starter content and the content build/sign pipeline |
| `tools/` | Build, signing and release scripts |
| `docs/` | Product, design and research documents |

## Distribution

Signed arm64 APKs are published on [GitHub Releases](https://github.com/sensesVoid/NeverHateThis/releases). There is no Play Store listing.

## License

Code: [Apache-2.0](LICENSE). Content is licensed per asset.
