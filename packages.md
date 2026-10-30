# Packages, namespaces, and mods

There are no imports. Everything lives in one global namespace, populated by loading folders:

```gdscript
func Game() {
    load "package"
    load "cookie_clicker"
}
```

- The program's own folder is a root like any loaded one: every folder inside it is a namespace, recursively,
  with no `load` needed ([programs.md](programs.md)).
- The loaded folder itself does **not** appear in the namespace; folders *inside* it do, and every folder is
  only a namespace: `package/engine/renderer.spite` is `Engine.Renderer()`, and
  `package/engine/physics/body.spite` is `Engine.Physics.Body()`. No file is special: a file named like its
  folder is an ordinary class inside it, so `package/engine/physics/physics.spite` is `Engine.Physics.Physics()`,
  reopened by another root's `engine/physics/physics.spite` like any class. A file and a folder of one name side
  by side, `engine/renderer.spite` beside `engine/renderer/`, would make `Engine.Renderer` both a class and a
  namespace, so that is an error ([the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/packages.md#packages-namespaces-and-loading)).
- `load` only ever takes a literal string: a variable or expression there is an error, so the compiler
  always knows every bundle statically. A `load` may sit under an `if` on a `Build` field the compiler already knows,
  which is how one entry function loads the standard library folder of the system it is built for
  ([programs.md](programs.md#how-a-program-is-loaded)).
- Folder names are always lowercase `snake_case`, package roots included: `window_plugin`, never
  `window-plugin`, and a folder that is not is an error before anything compiles. Of the folders a `load` names,
  only the last one becomes a namespace, so only it is held to the rule: `load "../../Old Tools/render_vulkan"`
  is fine, and `load "../../plugins/RenderVulkan"` is refused for `RenderVulkan`
  ([the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/packages.md#packages-namespaces-and-loading)).
- Resolving a bare `Name` from inside a class tries, in order: what that class declares (its `enum`, `union`
  and `type`), its folder, each parent folder, then the whole program. A dotted `Component.Requested` is looked up the same way, and so is
  every name wherever it is written: a constructor call, a parameter, a return type, an attribute or local
  annotation, a generic argument (`Remove<Component.Requested>`), a `type` or `union` member, a `==` class test
  and an enum inside a class (`Component.Requested.Size`), so from `window/system/` the name reaches
  `window/component/requested.spite` without writing `Window.` (`conformance/stage6/relative_namespaces`).
  A class's folder is where the walk starts, so `click_test/click_test.spite` (`ClickTest.ClickTest`) reaches
  `click_test/system/verify.spite` as `System.Verify()` (`conformance/stage6/folder_class_namespace`).
- A namespace holds classes and namespaces, nothing else. An `enum`, a `union` and a `type` belong to the class
  that declares them, like its attributes and functions: plain `Job` inside `player.spite`, and `Player.Job` (or
  `Game.Player.Job` from outside `game/`) everywhere else. A plain `Job` in another class is an error naming
  `Player.Job`, and so is `Game.Job`, which looks for it in the namespace `Game` instead of its class
  ([the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/packages.md#packages-namespaces-and-loading)).
- A generic class is found the same way, so it may live in any folder: `Asset.Pack<Asset.Texture>("textures")`
  and, from inside `game/`, `Pack<Rule>("rules")` for `game/pack.spite` both work
  (`conformance/stage6/namespaced_generics`). A generic name that finds nothing is one error, and the lines that
  use what it would have made are not reported again (`diagnostics/failed_constructor`).
- What the using class declares comes first, before every class. So `type Healing` in
  `system/regenerate.spite` is what `Healing` means there, even when the program's entry class is also `Healing`
  (`conformance/stage6/nearest_type`). The library is held to the same rule, so a program's own class `Entry` does
  not hide the union `Entry` that `Directory` declares, which the program names `Directory.Entry`:

```gdscript title=own_entry_class/entry.spite
var name = "mine"
```
```gdscript title=own_entry_class/own_entry_class.spite entry
var console = Console()

func OwnEntryClass() {
    load "../spite_standard_library/windows"
    var entry = Entry()
    console.print(entry.name)
}
```
```output
mine
```

- From outside its class, an enum is named through its owner (`Player.Job`), but a union or a `type` never is: a
  class that takes one declares its own copy. Two classes may each declare a `type Buildable`, and when their
  members are the same they are one type, so a `List<Buildable>` of one goes where the other's is wanted.

## Monkey patching (mods)

Every loaded root merges into the same namespaces. A second root with the same folder structure and file name
**reopens** the class instead of colliding with it: a later `func`/`var` of the same name replaces the
earlier one (in load order), and a name not seen before is simply added. A `union`, `type` or `enum` declared
again replaces the earlier declaration the same way ([below](#reopening-an-enum-replaces-it)). This is how game
mods work.

```gdscript title=package_demo/package/monster.spite
func describe(): String {
    return "a wild monster"
}
```
```gdscript title=package_demo/package/engine/renderer.spite
func render(): String {
    return "rendering the scene"
}
```

`mods` is loaded after `package`, so the `describe` below replaces `package/monster.spite`'s own, and `taunt`
is a name that folder never had. That is the whole of monkey patching.

```gdscript title=package_demo/mods/monster.spite
func describe(): String {
    return "a modded monster"
}

func taunt(): String {
    return "the modded monster taunts you"
}
```
```gdscript title=package_demo/package_demo.spite entry
var console = Console()

func PackageDemo() {
    load "package"
    load "mods"
    var renderer = Engine.Renderer()
    var rendered = renderer.render()
    console.print(rendered)
    var monster = Monster()
    var description = monster.describe()
    console.print(description)
    var taunt = monster.taunt()
    console.print(taunt)
}
```
```output
rendering the scene
a modded monster
the modded monster taunts you
```

A reopening file declares only the names it adds or replaces, and a `var`'s replacement must keep the same type. Within one file each name is declared once; replacing is what a
*later* file does.

The program's own folder is merged first and the folders it loads come after it, so "later" means a loaded
folder, not the program. A file in the program root that reopens a class a loaded folder declares keeps every
name the loaded folder does not have (an added attribute or function works), but where both declare the same
name, the constructor included, the loaded folder's version is the one that stays. To replace something a
loaded folder declares, put the replacement in a folder loaded after it, as `mods` is above. A repository pinned to a commit
([below](#loading-a-repository-pinned-to-a-commit)) is merged before the program instead, so where the program's
own files and a pinned copy declare the same name, the program's declaration stays.

The program's entry class is the one class nothing reopens: a loaded `bundle/potion.spite` in the program
`potion/` would otherwise become part of the entry class `Potion`, so it is an error naming both files
(`diagnostics/entry_class_reopened`).

The standard library is loaded before the program, so a program's own file reopens any class of it the same way:
`environment.spite` adds settings to `Environment` ([programs.md](programs.md#run-time-settings-environment)),
`list.spite` adds a member template to `List` ([collections.md](collections.md#write-your-own-member-template)),
`integer.spite` adds a function to every `Integer` ([values_and_types.md](values_and_types.md#numbers-are-classes)), and
`string.spite` could replace how `String` trims. Your foot to shoot. Each operating system's folder of the library
is built the same way: it reopens the classes that system does differently
([foreign_libraries.md](foreign_libraries.md#each-operating-system-reopens-what-it-changes)).

```gdscript title=reopen_string/string.spite
func shouted(): String {
    var upper = upper_case()
    return "{upper}!"
}
```
```gdscript title=reopen_string/reopen_string.spite entry
var console = Console()

func ReopenString() {
    load "../spite_standard_library/windows"
    var greeting = "hello"
    var shouted = greeting.shouted()
    console.print(shouted)
}
```
```output
HELLO!
```

### Reopening an enum replaces it

An enum a class declares is open the same way, and like a function it is replaced whole: a reopening file
that declares the enum again lists every value the enum has from then on, in its own order. Nothing is
appended and nothing merged, so a later folder (in the order above: the program's own folder first, then each
loaded folder in load order) that wants to add a value restates the list with the new one in it. An engine's
phases are an enum for exactly this: a mod restates the phases with its own among them, and everything that
walks the enum's values, `Phase.values`
([metaprogramming.md](metaprogramming.md#walking-a-programs-structure)), walks the new list.

```gdscript title=phase_mod/engine/schedule.spite
enum Phase {
    'update'
    'render'
}

var console = Console()

func run_every_phase() {
    Phase.values.each(run_phase)
}

func run_phase(phase: Phase) {
    console.print("running", phase)
}
```
```gdscript title=phase_mod/mods/schedule.spite
enum Phase {
    'input'
    'update'
    'render'
}
```
```gdscript title=phase_mod/phase_mod.spite entry
func PhaseMod() {
    load "engine"
    load "mods"
    var schedule = Schedule()
    schedule.run_every_phase()
}
```
```output
running input
running update
running render
```

A value the reopening leaves out is gone: code that still names it is a compile error, the same `'fog' is not
a value of this enum` as for any value the enum never had (`diagnostics/enum_reopening_replaces`). One
declaration is the whole truth about an enum, so a reader never has to add up several files to know its values.
Since the values are walked while compiling, a new one costs nothing at run time beyond what walking it
generates.

## The `Spite` namespace is reserved

`Spite` is the root namespace for reflection (`Spite.Class`, `Spite.Attribute`; see
[metaprogramming.md](metaprogramming.md)). A `spite/` folder of your own **reopens** those classes, which is
how a package is tried out before it is upstreamed. Inside the class, a reopening reads the private fields
(`_name`, `_namespace`) that the read-only `name` and `namespace` answer from outside:

```gdscript title=reopen_spite_class/spite/class.spite
func name_with_namespaces(): String {
    if _namespace {
        return "{_namespace.name_with_namespaces}.{_name}"
    }
    return _name
}
```
```gdscript title=reopen_spite_class/reopen_spite_class.spite entry
var console = Console()

func ReopenSpiteClass() {
    load "../spite_standard_library/windows"
    var console_class_name = console.class.name_with_namespaces()
    console.print(console_class_name)
}
```
```output
Console
```

Every class object in the program answers `name_with_namespaces()` from then on, because there is one `Spite.Class` and
that folder reopened it. A *new* class under `Spite` is a diagnostic instead: that namespace holds the
standard library's own classes, and a class of your own belongs in a namespace of your own. `load "spite"` is
a diagnostic too:

```gdscript title=spite_namespace_error/spite_namespace_error.spite entry error
func SpiteNamespaceError() {
    load "spite"
}
```
```diagnostic
'spite' is reserved for the built-in Spite namespace
```

`Spite.Internal` is the one namespace of the standard library that a program may not name at all: it holds the
machinery under the library (raw memory, threads, locks), and the library offers plain forms instead, such as
`Buffer` and `BinaryReader` for bytes. Naming it from your own code is a compile error that says which plain form
to use.

## `load` is a bundle boundary

`load` marks where a dynamic library or lazy-loaded bundle can split, the way an async `import()` does in
webpack. A `load` line compiles to nothing at run time. So a `load` costs nothing when the program runs; what it loads is
tree-shaken like the rest of the program.
`load` is a keyword written without parentheses, on a line of its own, and nothing else may be named `load`: a
function that loads something says what, `load_texture` ([the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/packages.md#packages-namespaces-and-loading)).

```gdscript title=load_parentheses_error/load_parentheses_error.spite entry error
func LoadParenthesesError() {
    load "../spite_standard_library/windows"
    load("level")
}
```
```diagnostic
'load' is a keyword, not a function: write it without parentheses, 'load "level"'
```

A `load` under an `if` is decided while compiling. The condition may read `Build` fields, text, whole numbers,
`true` and `false`, joined by `==`, `!=`, `and`, `or` and `not`, and only the branch it picks is loaded:

```gdscript
var build = Build()

func Engine() {
    if build.target_operating_system == "windows" {
        load "../plugins/windows_renderer"
    }
}
```

This works in every file of the program's folder and of a loaded package, so a package can pick its own plugins.
Each path is relative to the folder of the file it is written in, and only relative: an absolute path such as
`load "D:/projects/engine/core"` would build only on a machine that has that folder, so it is an error that names
the relative path to write instead, like `load "../../engine/core"`. A package from another repository is loaded
by where it sits beside yours, or pinned to a commit (below). A condition the compiler cannot decide, such as
one that reads `program.arguments()`, is an error rather than a folder that is quietly left out.

Each loaded root can become a separate dynamic library, tree shaking is computed per bundle, and a `load` inside an
`if` is loaded lazily when that line runs ([the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/packages.md#packages-namespaces-and-loading)).

## Why a dependency is a line in `load`

Setting up a project is where many languages become a thousand ways to do one thing: build files, lock files,
manifests, registries and the tools that manage them, each with its own habits. That variety is a cost paid by
everyone who has to read, build or copy the project, and it is why some people pick a language for its tooling
instead of for the language. Spite has one way: a dependency is a git URL with a commit hash in the `load` call,
and the ordinary compile fetches it. Git is the registry, so there is no central index for a competing ecosystem to
copy from or split away from, and no second convention to learn.

## Loading a repository pinned to a commit

A dependency is a repository and a commit, written in the `load` line itself. There is no package
manager, no registry, no lockfile and no fetch step: the ordinary compile fetches what a `load` names, and the pin
lives in the source, which is already versioned.

```gdscript
func Game() {
    load "github.com/example/engine@6c7dca9/core"
    load "../audio@b41e0d2/plugins/ogg_plugin"
}
```

The text is `<repository>@<commit>`, optionally followed by `/<folder inside it>`, which is loaded as a root
exactly as a folder on disk would be; with no folder, the repository's own top folder is the root. The repository
is either a URL, `github.com/example/engine` (fetched over `https://`) or one written with its scheme (`https://`,
`ssh://`, `git://` or `file://`), or a repository on this machine, named by a path that starts with
`./` or `../`, relative to the folder of the file the `load` is written in like any `load`.
Nothing else is a repository, so `load "engine@6c7dca9"` is an error that shows both forms, and a folder
without an `@` is an ordinary folder.

The commit is required, and it is a commit: 7 to 40 lowercase hexadecimal digits. A branch or a tag moves, and a
pin must not, so `@main` is an error, and so is a URL without an `@`. A pinned commit never changes silently: the
compiler checks the commit out once, then compiles from that copy.

**Where the copy lives.** The first compile that meets a pin checks the commit's files out into `.spite/git/` in the
folder `spite` was run from ([compiler.md](compiler.md#where-the-outputs-go)), as
`.spite/git/<repository name>_<number>/<commit>/`, the number worked out from the repository's whole path or URL,
and prints `fetched <repository>@<commit> into <folder>`. Every later compile finds it there and runs no git at all.
The copy is plain files, with no `.git` in it, kept out of the program's folders, so nothing mistakes it for source
to edit. From a repository on this machine the files are written straight from its objects: its working files, its
index, its branches and its worktrees are left exactly as they were, so the repository you are working in can be
the one you pin. From a URL the repository is cloned once, bare, into `.spite/git/<repository name>_<number>/repository`,
and fetched again only when a pin names a commit it does not hold yet. The compiler runs the `git` on the `PATH`.

**The copy is read-only.** Fetching records every file of the commit beside the copy, and each compile checks the
copy against that record before reading it: a file changed or removed there, or a `.spite` file added, is an error
naming it, never code that quietly differs from the commit. To change a package, change its repository, commit, and pin the
new commit; to fetch a copy again, delete its folder. A copy is never formatted, and live reload never watches it
([repl.md](repl.md#live-reload---hot-reload)): what it holds cannot change while the program runs.

**Pins inside a package.** A fetched package may pin repositories of its own. A relative repository path in it is
read from where the file sits in *its* repository, not in the copy, so an engine at `D:/Projects/engine` that
loads `../../../../spite_truetype@41c09e2/truetype` from `plugins/ui_plugin/ui/` reaches
`D:/Projects/spite_truetype` both when you build the engine itself and when a game loads the engine at a pin. A
package fetched from a URL names the repositories it loads by URL, since a path on its author's machine means
nothing on yours. An ordinary `load` in a fetched package stays inside that package's repository: one that leaves
it is an error, since the copy holds only its own commit's files. **Two pins of one repository** at different commits are one library ([below](#two-versions-of-one-repository)).

This program loads a folder of the language's own repository as it was at a commit. The documentation's programs
are written out into `.spite/docs/<name>/` of that repository and compiled from there, so `../../..` is the
repository itself:

```gdscript title=pinned_widgets/pinned_widgets.spite entry
var console = Console()

func PinnedWidgets() {
    load "../../..@6cb29ccf/conformance/stage2/namespaces_load/package"
    var widget = Widgets.Widget("gadget", 3)
    var description = widget.describe()
    console.print(description)
}
```
```output
gadget x3
```

### Two versions of one repository

Two pins of one repository at different commits are one library: every commit's files reopen the same namespaces
and classes, merged the way a [reopening](#monkey-patching-mods) is. A game that pins `engine@6c7dca9` and a
plugin that pins `engine@41c09e2` share one engine, so the physics the plugin runs moves the same bodies the game
draws. A plugin never needs bumping in step with the engine as long as the commits agree.

```gdscript
func Game() {
    load "../engine@6c7dca9/core"
    load "../minimap@b41e0d2/minimap"
}
```
```gdscript
func Minimap() {
    load "../../engine@41c09e2/core"
}
```

What one commit adds, the merged class has; what both declare must be the same in both. A function given two
bodies, or an attribute given two types, by two commits stops the compile with an error naming the declaration
and both commits, such as "'Renderer.draw' is declared one way by 6c7dca9 and another by 41c09e2, two commits of
one repository this program loads as one library (...): pin commits that agree, or declare it in the program's
own files, which win". Blank lines and comments do not count as a difference.

The program's own files win: the files of pinned copies are merged before the program's folder and the folders
it loads without a pin, so a `renderer.spite` of the game's own that declares `draw` settles which `draw` the
game runs, and the two commits may then disagree on it. A class keeps its plain name, `Renderer`, in messages,
reflection and `--final-classes`, since there is only one.

## Files beside a package's source

A package does not know where the program that loads it lives, so a relative path it opens (`File("shaders/
sky.spv")`) is read from the folder the program runs in, not from the package. A class asks where it
comes from instead: `class.source_files` inside it, `Recipe.source_files` for a class by name, and
`$item_type.source_files` for a generic's class answer a `List<File>`, every file that declares or reopens the class
in load order, and a namespace's `source_directories` answers a `List<Directory>` the same way
([reflection.md](reflection.md#the-spite-classes)). The paths are absolute, worked out while compiling, so a plugin
finds the files beside it wherever it is loaded from and whatever folder the program runs in.

Those paths are on the machine that **built** the program. It is the right answer while developing, and the wrong
one for a program shipped to someone else, whose machine has no such folder: a shipped program copies or cooks the
files it needs into its own output and opens them from there. Asking a `Spite.Class` held in a variable
(`kind.source_files`, with `var kind: Spite.Class = Recipe`) answers the same files when the program runs, and only a
program that asks that way carries the paths of its class objects.

## Final classes

Because patching is dangerous to read silently, `--final-classes=folder` writes every class as it ended up, after
every root is merged and every reopening resolved, as a program that runs the same as the one it was printed from
([compiler.md](compiler.md#inspect-merged-classes)). This is the file to read when you are not sure which mod won.

---

Next: [Concurrency: waiting without colouring](concurrency.md), running work in parallel without marking functions.
