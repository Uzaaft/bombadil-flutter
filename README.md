# bombadil-flutter

Test Flutter apps with [Bombadil](https://github.com/antithesishq/bombadil)
through its external driver protocol (version 0, see
`lib/bombadil-rpc/PROTOCOL.md` in the Bombadil repo). Not a first-party
Bombadil project.

The driver is `bombadil_flutter`, a Dart package that **runs inside the
app**. An app adopts it with a second entrypoint that calls `runBombadil`
with the app's own `main`. The app runs headless on `flutter_tester`, the
engine that `flutter test` uses. States are the app's semantics
(accessibility) tree, read from the framework. Actions are semantics
actions on its nodes.

- `lib/` is the package: `runBombadil` and the driver
  (`lib/src/flutter/`), and the protocol layer (`runDriver`,
  `BombadilDriver`) it is built on.
- `spec/` has TypeScript helpers for specifications: the state type, and
  action generators for the visible nodes.
- `experiments/compass_app/` runs flutter/samples' `compass_app` under
  Bombadil.

Run everything inside the flake's dev shell (`nix develop`, or
`direnv allow`). It provides Flutter (with Dart and `flutter_tester`) and
sets `BOMBADIL_FLUTTER_FONTS` to a font directory.

## Adopting it

Add the package to a package that depends on the app (or to the app's
dev dependencies), and write an entrypoint:

```dart
// bombadil_main.dart
import 'package:bombadil_flutter/bombadil_flutter.dart';
import 'package:my_app/main.dart' as app;

void main() => runBombadil(app.main, name: 'my_app');
```

The app's code stays as it is. Build the entrypoint's debug (JIT) kernel
and the app's assets, then run the kernel on `flutter_tester`:

```sh
flutter build bundle --debug --target-platform linux-x64 -t bombadil_main.dart
engine=$(dirname "$(readlink -f "$(command -v flutter)")")/cache/artifacts/engine/linux-x64
BOMBADIL_FLUTTER_PROTOCOL_FD=3 "$engine/flutter_tester" \
  --icu-data-file-path="$engine/icudtl.dat" --disable-vm-service \
  --enable-checked-mode --verify-entry-points --enable-software-rendering \
  --non-interactive --flutter-assets-dir=build/flutter_assets \
  build/flutter_assets/kernel_blob.bin 3>&1 1>&2
```

`experiments/compass_app/driver.sh` is that command. Use the app's own
`build/flutter_assets` for `--flutter-assets-dir` if the entrypoint lives
in another package: its asset manifest lists the app's assets.

`runBombadil` options: `name`, `size` (logical pixels, default
`BOMBADIL_FLUTTER_SIZE` or `1024x768`), `devicePixelRatio` (1),
`settleCap` (5s), `startupCap` (45s), `stallTimeout` (60s, sent in
`hello`, because a JIT app takes a few seconds to start), `fontsDir`
(default `BOMBADIL_FLUTTER_FONTS`) and `rasterize` (see below).
`BOMBADIL_FLUTTER_TIMING=1` logs where each step's time goes to stderr.

### Why flutter_tester, and this binding

- **Headless.** `flutter_tester` needs no display, no GTK and no GPU. The
  Linux desktop build would need a virtual display, its plugins and a
  window, only for the driver to read the same semantics tree.
- **Not `flutter test`.** It runs `flutter_tester` too, but captures the
  test's stdio and owns the test zone. Running `flutter_tester` directly
  passes Bombadil's stdin through. Its stdout carries the engine's logs and
  `print`s, so the protocol goes out on file descriptor 3, which the
  launcher points at Bombadil's stdout (`3>&1 1>&2`). No relay is needed.
- **Not the test bindings.** `AutomatedTestWidgetsFlutterBinding` runs the
  app in fake async: timers and animations only move when the driver
  pumps, and real I/O (shared preferences, network images) stalls unless
  the driver keeps advancing fake time. `LiveTestWidgetsFlutterBinding`
  runs in real time, so every transition costs its real duration.
  `BombadilBinding` is `WidgetsFlutterBinding` with the driver running
  frames itself on a virtual frame clock. Animations finish in
  milliseconds, while Dart timers and I/O run in real time.
- **No rasterization.** By default frames build, lay out, paint and
  update semantics, but aren't sent to the engine. `flutter_tester` would
  rasterize each one in software on the UI thread, and the driver reads
  semantics, not pixels. `rasterize: true` (or
  `BOMBADIL_FLUTTER_RASTERIZE=1`) sends them, for apps that wait for
  frame timings.
- **The view's metrics.** `flutter_tester` reports an 800x600 view at a
  device pixel ratio of 3. The binding overrides them through flutter_test's
  `TestPlatformDispatcher`, which is why the package depends on
  `flutter_test`.

**Fonts.** The engine only looks for system fonts in `/usr/share/fonts/`,
which NixOS doesn't have. Without fonts, text has zero width: in
compass_app, "Book New Trip" is 69px wide (just its icon) instead of 185px,
and layouts and taps go wrong. `runBombadil` loads one font family per
subdirectory of `BOMBADIL_FLUTTER_FONTS` with `FontLoader` before the app
starts. The flake's directory has Roboto and DejaVu Sans.

**Plugins.** No plugin is implemented. Method calls that the platform
doesn't answer get a successful null reply (JSON and standard method
codecs, and Pigeon host APIs), so plugin calls neither hang nor throw
`MissingPluginException`. The driver keeps the route
(`flutter/navigation`), the clipboard and `SystemNavigator.pop`
(`flutter/platform`). Blinking cursors are off
(`EditableText.debugDeterministicCursor`), so they don't keep apps busy.

## Settling (best effort)

After each action, the driver waits until the app is idle, the way
Bombadil's browser driver drains the event queue and waits for the next
frame. Each round yields to the event loop, so work that is already queued
runs, with its microtasks. Then the driver checks the app's signals:

- a frame is scheduled (`SchedulerBinding.hasScheduledFrame`) or running
  (`schedulerPhase`). Running animations keep scheduling frames, so they
  are waited for.
- a platform message hasn't been answered yet;
- a zero-delay timer (`Timer.run`, `Future(...)`) in the app's zone
  hasn't run yet.

If the app is busy, the driver waits for one of these to change (a frame
requested or finished, a reply, a timer run), not for a duration. When
nothing is pending, it captures the state.

The driver doesn't wait for what the app itself waits on: timers with a
delay (`Future.delayed`), network and file I/O, and image decoding. If the
screen changes when such work finishes, the change shows up in a later
state. Temporal properties (`eventually`, `next`) handle the states in
between. If the app isn't idle within `settleCap` (5s), for example a
spinner that animates forever, the state says `settled: false`. That cap is
the only timeout in settling.

**Frame clock.** Frames run as soon as the app asks for one, each 16.67ms
after the previous one in frame time, so animations run in milliseconds of
wall time. Frame time never falls behind wall time. After 120 frames in one
step (two seconds of animation), frames are paced in real time, so a
spinner that waits for I/O sees the I/O finish at a realistic point.

## State

```jsonc
{
  "route": "/search",          // the route the router last reported (routeInformationUpdated)
  "size": [1024, 768],
  "tree": {                    // the semantics tree, from the root
    "id": "ab3accf1",          // derived from what the node is: the same across runs
    "flutterId": 41,           // Flutter's id: differs across runs, tells look-alikes apart within one
    "label": "Book New Trip", "value": "…", "hint": "…", "tooltip": "…", "identifier": "…",
    "flags": ["enabled", "focusable", "button"],   // also textField, checked/unchecked, selected, focused,
                                                    // header, hidden, obscured, scopesRoute, image, ...
    "actions": ["tap", "focus"],                   // supported semantics actions
    "customActions": [{"id": 3, "label": "Archive"}],
    "rect": [823, 696, 185, 56],                   // x, y, width, height in logical pixels on screen
    "visible": true,                               // not hidden and partly on screen
    "scroll": {"position": 0, "min": 0, "max": 812},  // for scrollable nodes
    "children": []
  },
  "errors": [],                // FlutterError reports and uncaught errors since the last state
  "exitRequested": false,      // the app called SystemNavigator.pop since the last state
  "settled": true,
  "lastAction": {"action": {"Tap": {"node": "ab3accf1"}}, "ok": true}  // or ok: false with an error
}
```

The route is what the app reports to the platform. With go_router,
`context.push` doesn't change it (unless the app sets
`GoRouter.optionURLReflectsImperativeAPIs`), so a pushed page reports the
route below it. A node that merges its descendants (`MergeSemantics`, most
buttons and list tiles) carries their labels and has no children, as on a
device.

**Node ids.** Flutter's semantics ids depend on the order in which the app
built its widgets, so they drift across runs, and `--reproduce` and shrinking
would aim at the wrong nodes. A node's `id` is instead an FNV-1a hash of:
the nearest ancestor's `identifier`, its own identifier, role, role flags
(button, text field, checkable, ...) and label, and the label of its
nearest tappable ancestor, which tells apart the identical "Delete"
buttons of different rows. Nodes without a label or identifier also hash
their parent's id. The id leaves out position, value and state, so a node
keeps its id as it scrolls, changes or is replayed. Nodes that still look
alike get `-1`, `-2`, ... in traversal order.

## Actions

Externally tagged, on node ids from the current state:

- `{"Tap":{"node":"ab3accf1"}}` performs the tap semantics action, or a
  pointer tap at the node's centre (through `GestureBinding`) if it
  doesn't support tap.
- `LongPress`, `ScrollUp`, `ScrollDown`, `ScrollLeft`, `ScrollRight`,
  `Increase`, `Decrease`, `Dismiss`, `Focus`, `Expand`, `Collapse`,
  `ShowOnScreen`, `Copy` and `Paste` take `{"node": id}`, through
  `SemanticsOwner.performAction`.
- `{"SetText":{"node":"…","text":"abc"}}` taps the field to focus it if
  needed, then performs `setText`. Without `setText`, it sets the focused
  `EditableText`'s value.
- `{"CustomAction":{"node":"…","action":7}}` runs a custom semantics action.
- `{"PointerTap":{"x":10,"y":20}}` is a raw touch tap.
- `"Back"` is the system back button (`WidgetsBinding.handlePopRoute`):
  the navigators pop, or the app asks to close.

An action that doesn't fit the state (an unknown node or an unsupported
action) doesn't end the run: the next state reports `lastAction.ok: false`.
Only malformed actions are driver errors.

## The compass_app experiment

[`compass_app`](https://github.com/flutter/samples/tree/main/compass_app/app)
is Flutter's app-architecture case study. It runs on local data, and the
user is always logged in. `setup.sh` pins flutter/samples at
`21c71902bb2c7da3cce328ba5946cba2f35a660a`, cloning it to
`$FLUTTER_SAMPLES` (default `~/repositories/github.com/flutter/samples`),
and links the app to `out/app`. It builds the app's assets and the kernel
of `bombadil_main.dart`, a package next to it that depends on the
unchanged app. `driver.sh` is the launcher.

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
purpose, because a booked trip can be swiped away (`Dismissible`). Bombadil
finds a violation, and shrink reduces it to that one swipe
(`{"ScrollLeft":{"node":"f2100e50"}}`). The violation reproduced in 3 of 3
calibration runs.

Two-minute `bombadil external test` runs, measured between the protocol
messages ("driver step" is from `apply` to `state`):

| Driver | states/s | driver step median | p90 |
| --- | --- | --- | --- |
| Rust embedder (before) | 26.8 | 27ms | |
| in-app, rasterizing every frame | 42.9 | 12ms | 46ms |
| in-app (default) | 60.3 | 7ms | 30ms |

Per step, by default (`BOMBADIL_FLUTTER_TIMING=1`, medians): the action
takes 0.3ms, settling 5.9ms (24 frames, all of it waiting for frames),
capturing the tree 0.1ms and encoding it under 0.1ms (3KB; p90 26KB).

## Development

```sh
nix develop
flutter pub get && flutter test && dart analyze
```

The tests are widget tests of the semantics serialisation and node ids,
action dispatch, and settling, plus tests of the protocol layer. They need
`flutter test`: `dart test` can't load `dart:ui`.
