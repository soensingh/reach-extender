# Publishing a release

These are the maintainer's steps for publishing a new Reach Extender build on this repository's [Releases](https://github.com/soensingh/reach-extender/releases) page. Commands assume the repo is `soensingh/reach-extender` and that you run them from this repository's folder with the GitHub CLI (`gh`) logged in.

> [!IMPORTANT]
> Only the finished DMG goes up here, attached to a GitHub Release. Never commit binaries, build folders, source code, keys or `.env` files to this repository.

## 1. Before you start

- [ ] The Release build is signed with your Developer ID, notarized and stapled, and the DMG itself is signed and notarized too. This happens in the private app repo.
- [ ] The version number in the app matches the tag you're about to create, e.g. `0.1.0` → `v0.1.0`.
- [ ] `CHANGELOG.md` in this repo has a section for the new version with today's date. Commit and push that first.

Check the DMG before uploading:

```sh
VERSION=0.1.0
DMG=ReachExtender-$VERSION.dmg

xcrun stapler validate "$DMG"                                       # notarization ticket is stapled
spctl -a -t open --context context:primary-signature -v "$DMG"      # Gatekeeper accepts it
```

## 2. Name the files

Every release carries **two copies of the same DMG**:

| Asset name | Why |
|---|---|
| `ReachExtender-X.Y.Z.dmg` | The versioned file people can keep and compare. |
| `ReachExtender.dmg` | A **stable name** that never changes. The website's download link depends on it (see step 6). |

```sh
cp "$DMG" ReachExtender.dmg
```

The stable name must be exactly `ReachExtender.dmg`, with the same spelling and capitalization in every release. If one release uses a different name, the website's download link breaks for that release.

## 3. Get the checksum

```sh
shasum -a 256 "$DMG"
```

Both copies have the same checksum. Paste it into the release notes so people can verify their download.

## 4. Write the release notes

Copy the new version's section from `CHANGELOG.md` into a notes file outside the repo, and add the install and checksum details:

```sh
NOTES="/tmp/reach-extender-$VERSION-notes.md"
SHA=$(shasum -a 256 "$DMG" | cut -d' ' -f1)
cat > "$NOTES" <<EOF
## What's new

- (paste the bullets from CHANGELOG.md for $VERSION)

## Install

1. Download **ReachExtender-$VERSION.dmg** below.
2. Open it and drag **Reach Extender** into **Applications**.
3. Open the app and look for the hand in your menu bar.

Requires macOS 14 Sonoma or later. Free for your first 10 minutes of use; then a one-time lifetime license (₹199 in India, \$2 elsewhere) from https://reachextender.soensingh.com.

## Verify your download

\`\`\`
shasum -a 256 ReachExtender-$VERSION.dmg
$SHA
\`\`\`

Full changelog: https://github.com/soensingh/reach-extender/blob/main/CHANGELOG.md
EOF
open -e "$NOTES"   # fill in the bullets, then save
```

## 5. Create the release

Create it as a draft first, check it on GitHub, then publish:

```sh
gh release create "v$VERSION" "$DMG" ReachExtender.dmg \
  --repo soensingh/reach-extender \
  --title "Reach Extender $VERSION" \
  --notes-file "$NOTES" \
  --draft

# Look it over at https://github.com/soensingh/reach-extender/releases, then:
gh release edit "v$VERSION" --repo soensingh/reach-extender --draft=false --latest
```

The short version, which publishes immediately:

```sh
gh release create vX.Y.Z ReachExtender-X.Y.Z.dmg ReachExtender.dmg --title "Reach Extender X.Y.Z" --notes-file notes.md --latest
```

Optional: add `--discussion-category "Announcements"` to also open an announcement discussion for the release.

**Betas:** add `--prerelease`. Pre-releases and drafts are never "latest", so the website keeps serving the last stable version.

## 6. The website's download link (one-time setup)

GitHub always serves the newest stable release's asset at this URL:

```
https://github.com/soensingh/reach-extender/releases/latest/download/ReachExtender.dmg
```

Set the website's `DOWNLOAD_MAC_URL` to that URL **once**. After that you never need to change it for a new version, as long as every release:

1. includes an asset named exactly `ReachExtender.dmg`, and
2. is published (not a draft) and marked **Latest** (not a pre-release).

If you rename the repository, GitHub redirects the old URL, but update `DOWNLOAD_MAC_URL` anyway.

When the Windows build exists, use the same pattern with a stable asset name such as `ReachExtender-Setup.exe` and `…/releases/latest/download/ReachExtender-Setup.exe`.

## 7. Check that it worked

```sh
# The stable link should redirect to the new version's asset:
curl -sIL https://github.com/soensingh/reach-extender/releases/latest/download/ReachExtender.dmg | grep -iE '^(HTTP|location)'

# Download it the way a user would and compare checksums:
curl -sL -o /tmp/rx-check.dmg https://github.com/soensingh/reach-extender/releases/latest/download/ReachExtender.dmg
shasum -a 256 /tmp/rx-check.dmg
```

Then click **Download** on https://reachextender.soensingh.com/download and make sure the new version installs and opens.

## 8. Afterwards

- [ ] Add an empty `## [Unreleased]` section back at the top of `CHANGELOG.md` and update the comparison links at the bottom.
- [ ] Post the release in Discussions → Announcements (if you didn't use `--discussion-category`).
- [ ] Share a short clip of what's new.

## Fixing a bad release

- Wrong file attached: `gh release upload vX.Y.Z ReachExtender.dmg --clobber` replaces an asset in place.
- Bad build: publish a fixed `vX.Y.(Z+1)` and it becomes Latest. Then mark the bad one as a pre-release (`gh release edit vX.Y.Z --prerelease`) or delete it (`gh release delete vX.Y.Z --cleanup-tag`).
