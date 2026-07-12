# Qt & QML Code Review Guide

> Reviewer reminder for Qt C++ (`QObject`, signals/slots, models, threading, containers) and QML/Qt Quick diffs (bindings,
> delegates, layouts, rendering, C++ integration). Qt 6 baseline; Qt 5→6 migration traps flagged inline. Terse by design —
> signal in the diff → what to flag. Language-level C++ concerns → [C++ Guide](cpp.md).
> Distilled from The Qt Company's agent-skills review checklists (BSD-3-Clause, © The Qt Company Ltd.), the Qt 6 docs
> (Best Practices, Performance, Threads and QObjects), and KDAB's "Eight Rules of Multithreaded Qt".

## Contents

- [QObject Ownership and Lifetime](#qobject-ownership-and-lifetime)
- [Signals and Slots](#signals-and-slots)
- [Threading](#threading)
- [Qt Containers and COW](#qt-containers-and-cow)
- [Model/View Contract (QAbstractItemModel)](#modelview-contract-qabstractitemmodel)
- [Qt Error Handling](#qt-error-handling)
- [QML: Bindings and Properties](#qml-bindings-and-properties)
- [QML: Delegates and Dynamic Creation](#qml-delegates-and-dynamic-creation)
- [QML: Layouts and Anchors](#qml-layouts-and-anchors)
- [QML: Rendering and Performance](#qml-rendering-and-performance)
- [QML ↔ C++ Integration](#qml--c-integration)
- [Qt 5 → 6 Migration Traps](#qt-5--6-migration-traps)
- [Review Checklist](#review-checklist)

## QObject Ownership and Lifetime

- `new QObject`-derived without a parent and without explicit lifecycle management → leak; either parent it or own it (`std::unique_ptr` at the composition root). Never **both** — parent + smart pointer double-deletes.
- Deleting a QObject from a slot connected to it (or mid-signal-emission) → use `deleteLater()`, never `delete this`/direct delete in the call chain.
- Caching a `QObject*` that another owner may delete → dangling; use `QPointer<T>` (nulls itself) and null-check on use.
- `QNetworkReply` in a `finished` handler without `deleteLater()` → leaks one reply per request.
- Destructor deleting children manually with `qDeleteAll(children())` while parent-child is in play → double delete or grandchildren leaks; the QObject parent already deletes children.
- Copying a QObject subclass (missing `Q_DISABLE_COPY(_MOVE)`) → identity types must not be copyable.
- `Q_OBJECT` macro missing on a QObject subclass with signals/slots/properties → moc silently degrades (`qobject_cast`, signal lookups fail at runtime).
- Side-effectful expressions inside `Q_ASSERT(...)` → compiled out in release builds; `Q_ASSERT(ptr)` as the only null guard crashes release builds anyway.

## Signals and Slots

- Lambda `connect` without a **context object** → the connection outlives objects the lambda captures:

```cpp
connect(sender, &S::done, [this] { refresh(); });        // ❌ fires after `this` is destroyed
connect(sender, &S::done, this, [this] { refresh(); });  // ✅ auto-disconnects when `this` dies
```

- String-based `SIGNAL()`/`SLOT()` connects in new code → pointer-to-member syntax (compile-time checked). `Qt::UniqueConnection` does not work with lambdas — duplicates silently accumulate.
- Slot that re-emits or mutates state feeding its own trigger → re-entrancy loop; needs a guard or state check.
- Cross-thread `connect` forced to `Qt::DirectConnection` → slot runs in the **emitter's** thread; on a GUI receiver that's a crash. Default `AutoConnection` already queues cross-thread — flag any explicit `DirectConnection` across threads.
- Custom types in queued connections (cross-thread signals) must be registered metatypes → missing `Q_DECLARE_METATYPE` + `qRegisterMetaType<T>()` fails at runtime with only a console warning.
- Blocking wait for a signal (`QEventLoop::exec` in a slot, `waitForFinished` on GUI thread) → re-entrancy and frozen UI; restructure into async continuation.

## Threading

KDAB's eight rules, condensed to review signals:

- GUI classes (`QWidget`, Qt Quick items) touched off the main thread → flag, always. Worker threads compute (`QImage`/`QPainter` are fine) and hand results over via queued signals.
- Blocking the main thread (`QThread::wait`, `sleep`, synchronous network/disk waits) → frozen UI; event-driven design or worker.
- Subclassing `QThread` and adding slots → slots run in the thread that **created** the QThread, not in `run()`; the worker-object pattern (`moveToThread`) is almost always what's meant:

```cpp
// ✅ worker-object pattern
auto* worker = new Worker;                 // no parent — parented objects can't moveToThread
worker->moveToThread(&thread);
connect(&thread, &QThread::finished, worker, &QObject::deleteLater);
connect(this, &Ctrl::doWork, worker, &Worker::process);   // queued into the worker thread
```

- QObject accessed from a thread other than its affinity thread → treat QObject as non-reentrant; cross-thread calls go through queued signals or `QMetaObject::invokeMethod(obj, ..., Qt::QueuedConnection)`.
- Objects living in a worker thread must be destroyed before/with that thread → missing `finished → deleteLater` wiring leaks or destroys from the wrong thread.
- `QtConcurrent::run` writing to members without synchronization → data race; results come back via `QFutureWatcher` signals, not shared members.
- `QAbstractItemModel` mutated from a background thread → flag, always; models belong to the GUI thread — marshal changes over queued signals.
- A raw `bool`/counter flag shared across threads without `std::atomic`/mutex → same UB as plain C++ — Qt does not make it safe.

## Qt Containers and COW

Qt containers are implicitly shared (copy-on-write); accidental *detach* deep-copies the data:

- Non-const range-for over a shared Qt container → detaches; iterate `std::as_const(container)` or use `const auto&` on a const ref.
- `operator[]` on `QMap`/`QHash` for **reads** → non-const `[]` detaches *and* default-inserts missing keys; use `.value(key)` / `.contains()`.
- Returning Qt containers by value is cheap (COW) — flag out-param patterns built to "avoid copies" of Qt containers.
- Mixing Qt and STL algorithms is fine, but `detach()`-heavy hot loops (repeated non-const `[]`, `data()`, `begin()` on shared containers) deserve a question.

## Model/View Contract (QAbstractItemModel)

- Structural changes without the paired signals → views desync/crash: insertions inside `beginInsertRows`/`endInsertRows`, removals inside `beginRemoveRows`/`endRemoveRows` (moves/layout likewise). `layoutChanged` is not a substitute for row add/remove. Begin/end must balance on **every** code path, and `beginRemoveRows(parent, 0, count-1)` with `count == 0` violates the contract (first > last).
- `setData` returning `true` without emitting `dataChanged` → stale views. `dataChanged` should carry the specific changed roles — an empty roles vector means "everything changed" and forces full redraws.
- `data()` must handle every role it advertises: `roleNames()` keys with no matching `case` silently return empty `QVariant`s to QML. Cache the `roleNames()` hash (static/member) instead of rebuilding per call.
- Proxy models reaching into source internals (raw struct pointers) instead of `sourceModel()->data()/index()` → breaks with any other proxy in the chain.
- `flags()` returning `ItemIsEditable`/checkable on non-editable nodes → phantom edit affordances in views.

## Qt Error Handling

- `QFile::open()`, `QSaveFile::commit()` return values ignored → silent data loss.
- `QJsonDocument::fromJson` used without checking `isNull()`/type before drilling in → default-constructed values masquerade as data.
- `QNetworkReply`: read without checking `error()`, no `setTransferTimeout` on the request, `sslErrors` unhandled → hangs and garbage data on flaky networks.
- `QString::arg()`: placeholder count vs `.arg()` chain mismatch renders literal `%3` into user-visible text; prefer the multi-arg overload `str.arg(a, b, c)` — chained `.arg()` substitutes into earlier results.
- Mixed error-reporting styles in one class (return-bool + error-signal + status-getter) → pick one, flag drift.

## QML: Bindings and Properties

- **Imperative assignment destroys the binding** — the single most common QML bug:

```qml
width: parent.width / 2          // declarative binding
Component.onCompleted: width = 300   // ❌ permanently replaces the binding above
// ✅ if reactivity must be restored: width = Qt.binding(() => parent.width / 2)
```

  Runtime-only diagnosis (`qt.qml.binding.removal` logging category); qmllint does not catch it.
- Binding loops: the runtime reports single-cycle loops, but **multi-cycle** loops (A's handler writes B, B's binding reads A) are silent perf sinks — common with `implicitWidth`/`implicitHeight` in layouts.
- `property var` where a concrete type works (`int`, `string`, `list<Item>`) → blocks qmlsc compilation and type checking.
- Unqualified names resolving through the dynamic scope chain (`someProp` instead of `root.someProp`) → fragile and uncompilable; top-level component conventionally `id: root`.
- Properties never written after init → `readonly property`.
- Expensive function calls inside hot bindings → re-run on every dependency change; hoist into a `readonly property` or compute in C++.
- Alias chains (alias → alias) → `undefined` during construction ordering; flag chains deeper than one hop.

## QML: Delegates and Dynamic Creation

- Model roles used bare in delegates (`text: name`) → declare `required property string name`; enables compilation and survives `pragma ComponentBehavior: Bound`. Once *any* required property exists, `index`/`modelData` are no longer auto-injected — declare them too.
- `ListView { reuseItems: true }` with stateful delegates → pooled delegates keep non-model state across reuse (checkbox state bleeding between rows); reset in `ListView.onReused` — `Component.onCompleted` runs only once per pooled instance.
- Imperative `connect()` in `Component.onCompleted` of a delegate → connection survives delegate destruction, `TypeError` when it next fires; use a declarative `Connections { target: ... }` (auto-cleaned).
- `Connections` without an explicit `target` → silently targets `parent`; always set `target` (or `target: null` until assigned).
- `Loader`: `item.foo` without a readiness guard (`asynchronous: true` → `item` is null until `Loader.Ready`) → `TypeError`; use `loader.item?.foo` or gate on status. `source` and `sourceComponent` are mutually exclusive.
- `Component.createObject()` result neither parented nor referenced → GC collects it mid-life; `Qt.createQmlObject(string)` → re-parses per call, flag.
- `parent` inside delegates/Loaders/Popups is **not** what it looks like (delegate container, the Loader itself, the overlay) → use `ListView.view`, ids, and null-check `parent` during create/destroy windows.
- Delegate complexity multiplies by row count: nested Repeaters/Loaders, `clip: true`, shader effects inside a delegate → scrolling jank; keep delegates flat, anchors over binding-positioning.

## QML: Layouts and Anchors

- `anchors.*` and `Layout.*` on the same item → they fight; inside a `RowLayout`/`ColumnLayout`/`GridLayout` use only `Layout.preferredWidth`/`fillWidth` etc. Bare `width:`/`height:`/`x:`/`y:` on a layout child is silently overridden.
- Anchoring to items that aren't parent or sibling → runtime error/undefined layout; four separate edge anchors to parent → `anchors.fill: parent`.
- Reusable components setting explicit `width`/`height` internally → callers can't size them; provide `implicitWidth`/`implicitHeight` from content instead.
- Top-level `states` in a reusable component → `states` is a list property, external assignment **appends** rather than replaces; wrap internal states in a `StateGroup`. `Transition` without `from`/`to` fires on every state change.

## QML: Rendering and Performance

- `Rectangle { color: "transparent" }` as a grouping node → still a geometry node in the scene graph; use `Item`. Compounds inside delegates.
- `opacity: 0` to hide → still rendered, still in input/focus chain; `visible: false` skips both. `opacity` between 0 and 1 forces blending — expensive over large areas.
- `clip: true` → "a visual effect, NOT an optimization" (Qt docs): breaks scene-graph batching; acceptable on a ListView viewport, costly sprinkled on small items.
- `layer.enabled: true` left on permanently → offscreen FBO per frame; enable only around the effect/animation that needs it.
- `Image` without `sourceSize` for large sources → full-resolution decode into GPU memory (a 4000×3000 photo shown at 100×75 still uploads ~48 MB); network/disk images without `asynchronous: true` decode on the GUI thread; check `Image.status` for failure.
- `Text` with `textFormat: Text.RichText` for plain strings → full HTML parser; `PlainText`/`StyledText`. Animating `font.pixelSize` → full relayout per frame; animate `scale` instead.
- Manual `gc()` calls → block the GUI thread mid-animation.
- Heavy JavaScript in QML (data transforms, business logic) → belongs in C++; QML JS blocks compilation and runs interpreted. Per-frame JS in animations → skipped frames.
- Incremental property writes in a JS loop → each write re-triggers bindings; accumulate locally, assign once.

## QML ↔ C++ Integration

- `rootContext()->setContextProperty(...)` in new code → deprecated pattern: unscoped, invisible to tooling, blocks compilation. Use `QML_ELEMENT`/`QML_SINGLETON` registered types, and `createWithInitialProperties`/required properties for per-instance state.
- Ownership across the boundary: a parentless `QObject*` returned from a `Q_INVOKABLE`/slot call gets **JavaScriptOwnership** — the QML GC may delete it under C++'s feet. Give it a parent or `QJSEngine::setObjectOwnership(obj, QJSEngine::CppOwnership)`. Objects read from **properties** keep C++ ownership (opposite default).
- Singletons for shared *behavior*/enums are fine; singletons as shared mutable *data* stores accessed by reusable components → untestable coupling; prefer instantiated types passed in.
- Direction rule: **signals communicate up, functions/properties communicate down.** QML emitting a C++ object's signals, or a signal handler mutating the emitter → inverted flow, flag.

## Qt 5 → 6 Migration Traps

- `Connections { onFoo: ... }` → deprecated; `Connections { function onFoo() {...} }`. Mixing both styles in one block silently drops the function-based handlers.
- `PropertyChanges { target: x; prop: v }` → Qt 6 syntax `PropertyChanges { x.prop: v }` (old form only for Design Studio `.ui.qml`).
- `Binding.restoreMode` default changed (Qt 5 `RestoreNone` → Qt 6 `RestoreBindingOrValue`): Qt 5 code relying on a disabled `Binding` "sticking" silently reverts.
- `QtGraphicalEffects` → `MultiEffect` (Qt 6.5+); `Qt5Compat.GraphicalEffects` is a bridge, not a destination.
- Versioned imports (`import QtQuick 2.15`) → drop versions in Qt 6; they cap the visible API.
- `MouseArea` in new code → pointer handlers (`TapHandler`, `DragHandler`, `HoverHandler`): non-visual, composable, multi-touch; mixing MouseArea with handlers causes grab conflicts.

## Review Checklist

### QObject Lifetime

- [ ] Every `new` QObject has a parent or an explicit owner — never both
- [ ] `deleteLater()` from slots/signal chains; `QPointer` for cached cross-owner pointers
- [ ] `QNetworkReply::deleteLater()` in every finished path
- [ ] `Q_OBJECT` present; copy disabled on identity types; no side effects in `Q_ASSERT`

### Signals and Slots

- [ ] Every lambda connect has a context object; pointer-to-member syntax
- [ ] No cross-thread `DirectConnection`; queued-connection payload types registered
- [ ] No nested event loops / blocking waits on the GUI thread; re-entrant emit paths guarded

### Threading

- [ ] No GUI objects touched off the main thread; no blocking on the main thread
- [ ] Worker-object + `moveToThread`, not QThread-with-slots; workers `deleteLater` on `finished`
- [ ] Cross-thread access only via queued signals/`invokeMethod`; shared flags atomic/mutexed
- [ ] Models never mutated from background threads

### Containers and Models

- [ ] `std::as_const` in range-for over Qt containers; `.value()` not `[]` for map/hash reads
- [ ] Structural model changes wrapped in balanced begin/end pairs; `dataChanged` with roles
- [ ] `roleNames()` cached and consistent with `data()` cases; proxies use source model API

### Qt Error Handling

- [ ] `QFile::open`/JSON parse/`QNetworkReply::error` checked; transfer timeouts set
- [ ] `arg()` placeholders match; one error-reporting style per class

### QML Bindings

- [ ] No imperative writes to bound properties (binding destruction); `Qt.binding` where needed
- [ ] Typed + `readonly` properties; qualified access via ids; no multi-cycle binding loops
- [ ] No expensive calls inside hot bindings

### QML Delegates and Loading

- [ ] `required property` per model role (incl. explicit `index`); `reuseItems` state reset in `onReused`
- [ ] `Connections` with explicit target, function-syntax handlers; no imperative `connect()` in delegates
- [ ] `Loader.item` guarded; `createObject` results parented/referenced; delegates kept flat

### QML Layout and Rendering

- [ ] No anchors/`Layout.*` mixing; reusable components expose implicit sizes; internal states in `StateGroup`
- [ ] `Item` not transparent `Rectangle`; `visible: false` not `opacity: 0`; `clip`/`layer.enabled` justified
- [ ] Images: `sourceSize` + `asynchronous` for large/network sources; `PlainText` for plain strings

### C++ Integration and Migration

- [ ] No new context properties; ownership explicit at the QML/C++ boundary
- [ ] Signals up, functions down; singletons not used as data stores
- [ ] Qt 6 syntax for `Connections`/`PropertyChanges`; pointer handlers over `MouseArea`
