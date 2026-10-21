# AltStore source

The source for AltStore Classic and SideStore is published at:

<https://thebcd69.github.io/bcd-games/source.json>

Users can add that URL under Sources. The site root also has links to add the
source in either app. The source distributes the sideloading IPA and does not
support AltStore PAL.

## Enable hosting

A repository administrator must complete these steps before the first deployment:

1. In **Settings → Pages → Build and deployment**, select **GitHub Actions**.
2. If the `github-pages` environment restricts deployment branches and tags,
   allow the `main` branch and the release tags (`v*`). A release event runs
   with a tag reference, even though the generator is checked out from `main`.
3. In **Actions → Publish AltStore source**, select **Run workflow** on `main`.
4. Check that the source URL returns JSON, then add it in AltStore Classic or
   SideStore.

No personal access token or repository secret is required. The workflow uses
GitHub's temporary token. Only the deploy job can write to Pages. Pull requests
run the generator's tests, without downloading an IPA or deploying. Forks do
not publish this source.

## Updates

The workflow checks the latest stable GitHub release when a release is
published, edited or deleted. It also checks hourly, at 17 minutes past the hour,
and supports manual runs. The hourly check covers assets uploaded after a
release is published and release events suppressed when another workflow uses
`GITHUB_TOKEN`. GitHub can delay scheduled runs. It can also disable scheduled
workflows after 60 days without repository activity; re-enable the workflow in
Actions if that happens.

The source contains the latest stable release only. Drafts and prereleases are
excluded. Downloads point directly to the versioned `BCD.ipa` release asset.
The generator verifies its size and SHA-256 digest, then reads the version,
build number, bundle identifier, minimum iOS version, entitlements and privacy
descriptions from the IPA. It includes embedded app extensions' permissions.
The icon uses the upstream image at that release's tag.

The deployed `state.json` records the generator hash and relevant release
metadata. An unchanged release needs no IPA download or Pages deployment.
Changed release notes and replacement assets trigger an update. If validation
fails, the previous Pages deployment stays available.

## Local checks

The tests use Python's standard library and run on macOS or Linux:

```sh
python3 tests/host/check-altsource.py
```

Generating the source requires macOS and its `codesign` command. It reads the
downloaded executables' entitlements without running those executables:

```sh
python3 tools/update-altsource.py --force
```

To use an IPA already downloaded from the latest stable release:

```sh
python3 tools/update-altsource.py --force --ipa /absolute/path/BCD.ipa
```

The file must match the latest release's checksum. Output goes to
`build-out/altsource/`, which is ignored by Git. The workflow uploads only that
directory. It does not commit generated files or IPA files.

Source format: [AltStore's developer documentation](https://faq.altstore.io/developers/make-a-source).
