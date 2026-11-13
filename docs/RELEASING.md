# Releasing

The app ships through **GitHub Releases only**. There is no Play account, no AAB, and no F-Droid in v1.

## Versioning

- SemVer tags: `vMAJOR.MINOR.PATCH` (for example `v0.1.0`).
- `versionName` is the tag without the `v`.
- `versionCode` is derived from the tag: `MAJOR * 10000 + MINOR * 100 + PATCH` (for example `0.1.0` → `100`).
- Release notes are generated from the tag and edited if needed; they are the changelog.

## Signing key

The release keystore is generated once and kept offline by the owner. It is never committed.

```powershell
keytool -genkeypair -v -keystore nht-release.jks -alias nht `
  -keyalg RSA -keysize 4096 -validity 10000
```

Add these repository secrets in GitHub → Settings → Secrets and variables → Actions:

| Secret | Value |
|---|---|
| `ANDROID_KEYSTORE_BASE64` | `base64 -w 0 nht-release.jks` |
| `ANDROID_KEYSTORE_PASSWORD` | keystore password |
| `ANDROID_KEY_ALIAS` | `nht` |
| `ANDROID_KEY_PASSWORD` | key password |

The Gradle release config reads `NHT_KEYSTORE_PATH`, `NHT_KEYSTORE_PASSWORD`, `NHT_KEY_ALIAS` and `NHT_KEY_PASSWORD`, which the release job sets from those secrets.

## Release steps

1. Update `versionName`/`versionCode` in the Android module (or the shared version file).
2. Commit, then tag:
   ```bash
   git tag v0.1.0
   git push origin main --tags
   ```
3. CI runs the Rust tests and the Android build, then the release job produces a signed arm64 APK plus `SHA256SUMS` and creates the GitHub Release.
4. Download the APK from the release, verify the checksum, and install it on a reference phone before announcing.

## Installing without Play

- Enable "install unknown apps" for the browser or file manager.
- Verify `sha256sum -c SHA256SUMS` (or `Get-FileHash` on Windows).
- [Obtainium](https://github.com/ImranR98/Obtainium) can track the repository's Releases and offer updates.

## Content packs

Content ships on the same Releases page under a `content-v*` tag. The pack contains the files plus a `manifest.json` signed with the offline Ed25519 key; the app verifies the signature and every SHA-256 before installing. The build/sign/upload script lives in `tools/` once the content pipeline lands.

## Branch protection

After the first push, enable a branch protection rule on `main` requiring the CI workflow to pass. That is a repository setting, not a file.
