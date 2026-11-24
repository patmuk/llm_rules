# stacks/apple.md — Xcode builds

For any project that builds for an Apple platform, whatever the UI
toolkit. Read before an Xcode build in a jj workspace and when cleaning
a workspace up.

## Build output

- Prefer a build directory inside the workspace's own build output
  (`-derivedDataPath`), so removing the workspace removes it.
- Never point one workspace's build directory at another's.

## DerivedData survives the workspace

A build without `-derivedDataPath` (Xcode itself, a plain `xcodebuild`,
a toolkit's wrapper) lands in `~/Library/Developer/Xcode/DerivedData`,
keyed by project path, as a folder of up to several GB that survives
the workspace's `rm`. After `jj workspace forget` and the `rm`
(`LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §2.3):

```sh
ws="$(cd .. && pwd)/<repo>-<task-slug>"
for d in ~/Library/Developer/Xcode/DerivedData/*/; do
  p=$(/usr/libexec/PlistBuddy -c "Print :WorkspacePath" "$d/info.plist" 2>/dev/null)
  case "$p" in "$ws"/*) rm -rf "$d" ;; esac
done
```

## Simulator

- `xcrun simctl launch --terminate-running-process …`: a plain relaunch
  can race the old instance and ignore the launch arguments.
- `log show` and `uninstall` fail on a shut-down simulator: boot it
  first.
- An app installed before its database schema changed may fail to
  start its core and run without its data; tests then fail as if the
  app were wrong. Believe a red UI test only once the app's log of the
  run is clean; uninstall and run again otherwise.
- Cross-check that screenshots of different screens differ before
  trusting an "identical" diff.
