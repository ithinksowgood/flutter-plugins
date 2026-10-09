# SowGood's desktop_drop fork

This branch, `sowgood/desktop-drop-0.7.1`, is **desktop_drop 0.7.1 exactly as
published on pub.dev** (upstream tag `desktop_drop-v0.7.1`, commit `6e2bef5`;
every published file compared equal) **plus one fix** to its web
implementation. The SowGood app (`ithinksowgood/sowgood-app`) pins this
branch's head commit in `pubspec.yaml` `dependency_overrides` (git `url`,
`path: packages/desktop_drop`, `ref: <commit>`). Nothing else in this monorepo
is used.

## The change

`packages/desktop_drop/lib/desktop_drop_web.dart`, the web `ondrop` handler.

- **Before:** it called `webkitGetAsEntry()!` on every item of the drag,
  synchronously and outside the chain's `catchError`. The browser returns null
  for string items (a dragged link, selected text, the page markup that comes
  with an image dragged from another web page) and for a file item without a
  FileSystemEntry. Any such drop threw a TypeError out of the handler (an
  uncaught page error) and delivered nothing, not even the file that came with
  it. A file dragged from disk on its own worked.
- **After:**
  - Only items of kind `file` are read.
  - A file item without an entry is read with `getAsFile()`.
  - The handler catches its own errors and prints them with `debugPrint`, as
    the plugin already does for the asynchronous part.
  - `performOperation_web` is sent even when no file was dropped, as an empty
    list. The browser sends no `dragleave` after a drop, so the DropDoneEvent
    is what returns a `DropTarget` to idle. An empty DropDoneEvent already
    exists on Linux (`performOperation_linux` drops unreadable paths).
- `packages/desktop_drop/test/web_drop_test.dart` is a browser test
  (`flutter test --platform chrome`; it is skipped on the VM). It checks that
  a drop of a file with string items delivers the file, and that a drop with
  no file item ends with no files.

The same commit, based on upstream `main`, is on branch
`fix/web-drop-non-file-items` for an upstream pull request. Upstream issue
MixinNetwork/flutter-plugins#456 reports the same crash; open upstream pull
requests #459 and #503 change the same handler.

## Why 0.7.1

It is the version the app already resolved. `stream_chat_flutter` 9.28.0
(whose message input is the app's only `DropTarget`) allows
`desktop_drop >=0.5.0 <0.8.0`. On Android, 0.8.0 and 0.8.1 also required
Android Gradle Plugin 9; 0.8.2 restored older AGP support (with Kotlin
Gradle Plugin 2.0 or later), according to upstream's changelog.

## Moving the fix to a new version

1. `git fetch upstream --tags`
2. `git checkout -b sowgood/desktop-drop-<version> desktop_drop-v<version>`
3. Cherry-pick the fix commit (`desktop_drop: read only the files of a web
   drop ...`) and this README, then push the branch.
4. In the app, set `ref:` to the new head commit, run `pub get`, and check
   the lock.

## Retire this fork when BOTH hold

- an upstream desktop_drop release carries the fix, and
- the app's `stream_chat_flutter` allows that version.

Then delete the `desktop_drop` override from the app's `pubspec.yaml`, run
`pub get`, and check that the lock resolves desktop_drop from pub.dev.
