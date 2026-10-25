# bombadil-flutter

Test Flutter apps with [Bombadil](https://github.com/antithesishq/bombadil)
through its external driver protocol (version 0, see
`lib/bombadil-rpc/PROTOCOL.md` in the Bombadil repo). Not a first-party
Bombadil project.

- `embedder/` is **the Flutter driver**: `bombadil-flutter-driver`, a Rust
  program that hosts any Flutter app's debug bundle through the engine's
  embedder API (`flutter_embedder.h`, `libflutter_engine.so`). It needs no
  window, no display and no changes to the app, and it speaks the protocol
  on stdin and stdout. States are the app's semantics (accessibility)
  tree. Actions are semantics actions on its nodes.
- `spec/` has TypeScript helpers for specifications: the state type, and
  action generators for the visible nodes.
- `experiments/compass_app/` runs the driver against flutter/samples'
  `compass_app`.
- `lib/` is a generic Dart library for writing other external drivers in
  Dart (`BombadilDriver` and `runDriver`). The Flutter driver doesn't use
  it.

Run everything inside the flake's dev shell (`nix develop`, or
`direnv allow`). The shell provides Flutter, Rust and bindgen. It sets
`FLUTTER_EMBEDDER` to the prebuilt linux-x64 embedder engine for the
engine revision of nixpkgs' Flutter (a fixed-output derivation, so update
its revision and hash along with nixpkgs) and `BOMBADIL_FLUTTER_FONTS` to
a font directory. Only x86_64 Linux is supported.

## The driver

```sh
flutter build bundle --debug --target-platform linux-x64   # in the app
cd embedder && cargo build --release
bombadil external test --specification spec/generic.spec.ts -- \
  embedder/target/release/bombadil-flutter-driver \
    --assets <app>/build/flutter_assets \
    --icu-data <flutter>/bin/cache/artifacts/engine/linux-x64/icudtl.dat \
    --fonts "$BOMBADIL_FLUTTER_FONTS" --size 1024x768
```

Other options: `--name`, `--stall-timeout-ms` (sent in `hello`, 60000 by
default, because a JIT app takes a few seconds to start), and engine
switches after `--`. The app runs in debug (JIT) mode from
`kernel_blob.bin`, with the software renderer, at a fixed size and a
device pixel ratio of 1.

**Fonts.** The engine only looks for system fonts in `/usr/share/fonts/`.
Without fonts, all text has zero width, layouts collapse, and taps land on
the wrong widgets. `--fonts <dir>` runs the app from a symlinked copy of its
assets whose `FontManifest.json` adds one font family per subdirectory of
`<dir>`. The flake's directory has Roboto and DejaVu Sans.

**Settling.** The driver reports a state once the app has asked for no
frame and changed no semantics for 300ms (at least 300ms after an action),
or after 5s, with `settled: false`. Text cursors don't blink (the
deterministic-cursor accessibility feature), so they don't keep apps busy.

### State

```jsonc
{
  "route": "/search",          // last routeInformationUpdated on flutter/navigation (Router/go_router apps)
  "size": [1024, 768],
  "tree": {                    // the semantics tree, from node 0
    "id": 4, "label": "Book New Trip", "value": "…", "hint": "…", "tooltip": "…", "identifier": "…",
    "flags": ["enabled", "focusable", "button"],   // also textField, checked/unchecked, selected, focused,
                                                    // header, hidden, obscured, scopesRoute, image, ...
    "actions": ["tap", "focus"],                   // supported semantics actions
    "rect": [939, 696, 69, 56],                    // x, y, width, height on screen
    "visible": true,                               // not hidden and partly on screen
    "scroll": {"position": 0, "min": 0, "max": 812},  // for scrollable nodes
    "children": []
  },
  "errors": [],                // Flutter error reports and unhandled exceptions logged since the last state
  "exitRequested": false,      // the app called SystemNavigator.pop since the last state
  "settled": true,
  "lastAction": {"action": {"Tap": {"node": 4}}, "ok": true}  // or ok: false with an error
}
```

### Actions

Externally tagged, on node IDs from the current state:

- `{"Tap":{"node":4}}` sends the tap semantics action, or a pointer tap at
  the node's centre if the node doesn't support tap.
- `LongPress`, `ScrollUp`, `ScrollDown`, `ScrollLeft`, `ScrollRight`,
  `Increase`, `Decrease`, `Dismiss`, `Focus`, `Expand`, `Collapse`,
  `ShowOnScreen`, `Copy` and `Paste` all take `{"node": id}`.
- `{"SetText":{"node":15,"text":"abc"}}` focuses the field if needed, then
  sends `setText`. Without `setText` support, it falls back to
  `TextInputClient.updateEditingState` on `flutter/textinput`.
- `{"CustomAction":{"node":3,"action":7}}` runs a custom semantics action.
- `{"PointerTap":{"x":10,"y":20}}` is a raw touch tap.
- `"Back"` sends `popRoute` on `flutter/navigation`.

An action that doesn't fit the state (an unknown node or an unsupported
action) doesn't end the run: the next state reports `lastAction.ok: false`.
Only malformed actions are driver errors.

### Platform channels

The driver implements no plugins. Every method call gets a successful null
reply, so plugin calls neither hang nor throw `MissingPluginException`.
That covers JSON and standard method codecs, and Pigeon host APIs
(`dev.flutter.pigeon.*`). The driver keeps state from a few channels:

- `flutter/navigation`: the route, from `routeInformationUpdated`.
- `flutter/textinput`: the connected client, for the SetText fallback.
- `flutter/platform`: the clipboard, and `SystemNavigator.pop` as
  `exitRequested`.

The app answers back on `flutter/accessibility` (announcements),
`flutter/keyboard`, `flutter/mousecursor`, `flutter/processtext` and
`flutter/isolate`. Those messages are acknowledged and otherwise ignored.
On startup, the driver sends `flutter/settings` (light theme, text scale
1), `flutter/lifecycle` (resumed) and the `en_US` locale.

## The compass_app experiment

[`compass_app`](https://github.com/flutter/samples/tree/main/compass_app/app)
is Flutter's app-architecture case study. It runs on local data, and the
user is always logged in. `setup.sh` pins flutter/samples at
`21c71902bb2c7da3cce328ba5946cba2f35a660a`, cloning it to
`$FLUTTER_SAMPLES` (default `~/repositories/github.com/flutter/samples`).
It builds the unchanged app's debug bundle and the driver, and links them
into `experiments/compass_app/out/`. `driver.sh` is the launcher.

```sh
nix develop -c experiments/compass_app/setup.sh

# Test for two minutes.
bombadil external test --specification experiments/compass_app/spec.ts \
  --time-limit 2m --output-path /tmp/compass-test -- experiments/compass_app/driver.sh

# Fuzz with two app instances at a time.
bombadil external fuzz --specification experiments/compass_app/spec.ts \
  --parallelism 2 --time-limit-fuzz 2m --time-limit-run 20s \
  --output-path /tmp/compass-fuzz -- experiments/compass_app/driver.sh

# Find a violation on purpose, then shrink it.
bombadil external test --specification experiments/compass_app/violated.spec.ts \
  --exit-on-violation --time-limit 2m --output-path /tmp/compass-violation \
  -- experiments/compass_app/driver.sh
bombadil external shrink --trace /tmp/compass-violation \
  --specification experiments/compass_app/violated.spec.ts \
  --output-path /tmp/compass-shrunk -- experiments/compass_app/driver.sh
```

`spec.ts` explores with taps, text, scrolls, adjustments and back. It
checks these properties:

- no Flutter errors are logged;
- the route is always one the router defines;
- something can always be tapped or scrolled;
- generated actions always apply;
- away from home, back always changes the route or the topmost route
  scope.

`violated.spec.ts` adds `homeAlwaysListsTrips`, which the app violates on
purpose because a booked trip can be swiped away (`Dismissible`).
Bombadil finds a violation, and shrink reduces it to that one swipe.

## Development

```sh
nix develop
(cd embedder && cargo test && cargo clippy --all-targets)
dart pub get && dart test && dart analyze
```
