# Distribution

## Branches

```
main     upstream release commits + VERSION + bump.sh           no CI, no FFI
apple    main + 7zip-apple + release-mac                        mac-v*  →  lib7zip_apple.a (arm64 + x86_64)
```

`VERSION` holds `<7-Zip release>+MacRAR.Patch.<n>`. Its base must equal `MY_VERSION_NUMBERS` in `C/7zVersion.h`; `bump.sh`, `release-mac.sh` and the release workflow all refuse a mismatch.

## Set up the worktrees

```bash
W=/path/to/7zip                           # directory that holds .bare + the checkouts

git clone --bare git@github.com:ARAS-Workspace/7zip.git "$W/.bare"
git --git-dir="$W/.bare" config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
git --git-dir="$W/.bare" remote add upstream https://github.com/ip7z/7zip.git
git --git-dir="$W/.bare" remote set-url --push upstream DISABLED
git --git-dir="$W/.bare" fetch origin
git --git-dir="$W/.bare" fetch upstream --tags

for b in main apple; do
  git --git-dir="$W/.bare" worktree add "$W/$b" "$b"
done
```

## Take an upstream release

7-Zip publishes one commit per release on `ip7z/7zip` `main`, named after the version.

```bash
git --git-dir="$W/.bare" fetch upstream --tags
git -C "$W/main" cherry-pick -x <upstream-sha>          # e.g. the "26.05" commit
(cd "$W/main" && .github/scripts/bump.sh 26.05+MacRAR.Patch.1)
git -C "$W/main" push origin main

git -C "$W/apple" merge main
git -C "$W/apple" push origin apple
```

Then build and test on `apple` (`7zip-apple/build.sh`) before releasing.

## Release

```bash
W=/path/to/7zip
V=26.04+MacRAR.Patch.2

(cd "$W/main" && .github/scripts/bump.sh "$V")
git -C "$W/main" push origin main

git -C "$W/apple" merge main
git -C "$W/apple" push origin apple

(cd "$W/apple" && .github/scripts/release-mac.sh)
```

## Verify

```bash
for b in main apple; do
  printf "%-6s %s\n" "$b" "$(cat "$W/$b/VERSION")"
done

gh run list --repo ARAS-Workspace/7zip --limit 5
gh release list --repo ARAS-Workspace/7zip
```

## Consume

```bash
# app-mac (MacRAR)
gh release download "mac-v$V" --repo ARAS-Workspace/7zip --dir /tmp/sza
tar -xzf /tmp/sza/*.tar.gz -C Libraries/SevenZipKit/
(cd Libraries/SevenZipKit && shasum -a 256 -c CHECKSUM.sha256)
```
