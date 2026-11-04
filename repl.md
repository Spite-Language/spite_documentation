# REPL and live reload

`spite --repl` starts your program with a console attached. You type Spite at the `spite> ` prompt, and the line
runs inside the running program: an expression prints its value, a statement just runs. There are no commands to
learn, because every line is ordinary code: to see what a class holds you ask reflection, to print you call
`console.print`, and to make an object you make one.

```text
spite> monsters[0]
Monster { name: Goblin, health: 30 }
spite> monsters[0].health = 5
spite> monsters[0].health
5
spite> monsters[0].roar()
Goblin roars!
spite> var strongest = monsters[1]
spite> strongest.name
Orc
```

`--hot-reload` swaps the code you save into a running program, keeping its state
([Live reload](#live-reload---hot-reload)). An AI does not type into a console: it drives a running program through
the language's MCP ([mcp.md](mcp.md)), which runs code the same way and makes sure every save is live first.

The REPL, live reload and the MCP all compile new code into a program while it runs, so they run on Windows for now
(Linux and macOS: [spite_compiler#297](https://github.com/Spite-Language/spite_compiler/issues/297)). A REPL or
live-reload build is slower on purpose: it trades speed for what it tells you while the program runs. Never measure
performance with one; benchmarks use a production build
([Measure only a production build](compiler.md#measure-only-a-production-build)).

## The program this page uses

```gdscript title=repl_program/monster.spite
var name = "Monster"
var health = 10

func Monster(new_name: String, new_health: Integer) {
    name = new_name
    health = new_health
}

func roar(): String {
    return "{name} roars!"
}

func is_alive(): Boolean {
    return health > 0
}
```
```gdscript title=repl_program/repl_program.spite entry
var console = Console()
var player_name = "Hero"
var player_age = 20
var monsters = List<Monster>()

func ReplProgram() {
    load "../spite_standard_library/windows"
    var goblin = Monster("Goblin", 30)
    monsters.append(goblin)
    var orc = Monster("Orc", 50)
    monsters.append(orc)
    console.print("player", player_name, "age", player_age)
    var monsters_count = monsters.count()
    console.print("monster count", monsters_count)
}
```
```output
player Hero age 20
monster count 2
```

## Typing Spite into a running program

```
spite --repl
```

The program starts and the prompt appears at once. A line you type runs the next time the program waits (a
`program.sleep`, a file or socket read) or finishes a pass of one of its loops, so it sees the program between two
steps, never halfway through one. Once the constructor has returned, a line runs as soon as you press Enter.

- **An expression prints its value**, as `"{value}"` would make it text. A statement (an assignment, a call that
  answers nothing, an `if`, a `while`) prints only what it prints itself.
- **`program` is the running entry instance**, and its attributes and functions can also be named bare:
  `program.player_name` and `player_name` are the same thing.
- **A line is ordinary code.** It can call any function, `console.print("a", 1, true)` included, make objects,
  name any class, and reach a singleton by its name, `World().entity_count()`, exactly as the program's own code
  does.
- **`var` and `func` stay.** `var strongest = monsters[1]` keeps `strongest` for every line after it, and
  `func healthy(): Integer { ... }` declares a function the next lines can call. Declaring the same name again
  replaces it.
- **A line that opens a block continues** on the next one, under a `...> ` prompt, until its braces close:

```text
spite> for monster in monsters {
...>     console.print(monster.name, monster.health)
...> }
Goblin 5
Orc 50
```

- **A line that does not compile** prints the compiler's error, and nothing changes: the session and the program
  are as they were before the line.

Questions about the program itself are lines of Spite too, answered by reflection
([reflection.md](reflection.md)):

```text
spite> Spite.Class.instances.count()
2
spite> Monster.attributes.count()
2
spite> Monster.functions.has("roar")
true
spite> monsters[0].attributes["health"].value
5
```

`Spite.Class.instances` lists the program's classes, `Monster.attributes` and `Monster.functions` describe one, an
instance's `.attributes` reads its live values, and `Program().live_allocations()` and `live_bytes()` answer what
the heap holds ([memory.md](memory.md)).

### Input and ending

While the REPL is open, everything you type is the REPL's. A `Console.read_line()` in the program answers `null`,
as if the input had ended, instead of waiting for the keyboard.

The end of input (Ctrl+Z then Enter on Windows, Ctrl+D elsewhere), or a line calling `program.exit(0)`, ends the
program the normal way: everything is released and memory is balanced. **Ctrl+C ends it at once**, whatever is
running, your line or the program's own code, with exit code 130.

### A scratchpad

Run `spite --repl` in a folder with no program, and you get an empty one with the standard library loaded: a
place to try an expression, a text function or a class of the standard library before you write it into a program.

## Loading what you changed

The REPL does not watch your files: what you are poking at does not change under you because you saved something.
When you want a change, load it:

```text
spite> load "monster.spite"
rebuilt Monster
spite> monsters[0].roar()
Goblin roars louder!
```

A file the program already loads is updated in its place, so what goes live is what a fresh start would choose
([monkey patching](packages.md#monkey-patching-mods) included). A file the program never loaded is loaded after
everything else, as if you had added a `load` line at the end of the entry's constructor, so its declarations win.
Everything the program held stays: objects, singletons, and the `var`s you typed.

If you would rather have every save swapped in as you make it, start the REPL with the watcher too:

```
spite --repl --hot-reload
```

`load` still compiles a file at once when you do not want to wait for the watcher.

## Live reload: `--hot-reload`

`--hot-reload` builds a program that can take new code while it runs. Save a file of the program, and the classes
that file declares are compiled again, on their own, into a small library the running program loads and swaps in
the next time it waits or ends a pass of a loop, the same moments a REPL line runs at. Objects, singletons
and everything a REPL line changed stay as they were: only functions change. Every function sits in a slot so it can
be swapped, which makes the build slower; measure performance only with a production build
([Measure only a production build](compiler.md#measure-only-a-production-build)).

```
spite --hot-reload
```

Each swap, or each save it refuses, is one line on the program's error output (`spite: rebuilt Monster`). The
REPL carries the same machinery without the watcher, so it changes only when you `load`
([Loading what you changed](#loading-what-you-changed)); `spite --repl --hot-reload` has both. The language's MCP
always runs with the watcher, and its `evaluate` waits until every save is live before it runs anything
([mcp.md](mcp.md)). A development build does not include any of this unless you ask: the slots cost every call a
jump. `--hot-reload` keeps every function (it implies `--development`), since a new version of a class
may call one nothing called before, and it compiles like the default build, so a reload is quick. It is
slower than a release build, since its code is not optimised and every call goes through a slot a reload can
re-point: a `--hot-reload` build trades run speed for live information, and speed is measured on release builds.

```gdscript title=hot_counter/monster.spite
var name = ""

func Monster(starting_name: String) {
    name = starting_name
}

func roar(): String {
    return "{name} roars"
}
```
```gdscript title=hot_counter/hot_counter.spite entry
var console = Console()
var visits = 0
var monster = Monster("Goblin")

func HotCounter() {
    load "../spite_standard_library/windows"
    visits = 1
    var welcome = greeting()
    console.print(welcome)
}

func greeting(): String {
    return "hello, visit {visits}"
}

func monster_roar(): String {
    return monster.roar()
}
```
```output
hello, visit 1
```

This session runs against a copy of the program above, started with `spite --repl --hot-reload`:

```text
spite> visits = 42

# hot_counter.spite: greeting() now returns "welcome back, visit {visits}", and the watcher swaps the save in
spite: rebuilt HotCounter
spite> greeting()
welcome back, visit 42

# monster.spite: roar() now returns "{name} roars louder"
spite: rebuilt Monster
spite> monster.roar()
Goblin roars louder
```

`visits` kept the 42 set before the first reload, and the second reload rebuilt `Monster` alone. Without
`--hot-reload` the same two changes go live with `load "hot_counter.spite"` and `load "monster.spite"`.

### What a reload can change

| You change | What happens |
|---|---|
| a function's body | the next call runs the new body; a call already running finishes the old one |
| a new function on an existing class | the new code calls it, `Monster.functions` lists it and a REPL line can call it |
| a function's parameters or return type | it becomes a new function: the rebuilt class and every class that calls it are rebuilt too |
| a function you deleted | whatever still holds it (a function value, the REPL's session) keeps its last code; the answer says `removed Monster.roar` |
| an attribute's default value | instances made after the reload get the new default |
| a class's attributes: added, removed, renamed or retyped | every live object of the class moves to the new attributes ([below](#changing-a-classs-attributes)); the answer says what each class's objects kept |
| an enum's values: added, removed or reordered | the whole program is compiled again, and every value a running object holds keeps its meaning; a `switch` that meets a value the reload removed halts naming it |
| a file that does not compile | refused with the compiler's error, and the program keeps all of its code |
| a function of a class the program reopens, a class of the standard library or a number (`func doubled(): Integer` in `integer.spite`) | the whole program is compiled again and every function whose code changed is swapped in, the standard library's and the compiler's helpers included |
| `environment.spite`: a setting added, removed or given a new default | the whole program is compiled again, `Environment` moves to its new attributes, and every setting is read again the way it is when the program starts: from the command line it was started with, the environment, or the new default |
| `build.spite`: a field added, removed or given a new default | the whole program is compiled again, and the program's code sees the new value: in a `--hot-reload` build a `Build` field the program declares is read while it runs, and the standard library's reads of its own fields (the compiler's options) stay constants |
| a file deleted | the program is compiled without it, as for any other change |
| a function, attribute or constructor that a later `load` reopens ([monkey patching](packages.md#monkey-patching-mods)) | the reload keeps load order, as a fresh start would: an edit to the earlier declaration changes nothing while the later one replaces it, an edit to the later one goes live, deleting the later one (or its file) brings the earlier one back, and a replacement a later load gains goes live; a `Build` field the program's own `build.spite` declares stays the program's ([Build options](compiler.md#build-options)). When nothing the program runs changed, the answer says `no code the program runs changed, so nothing was rebuilt` |
| a new class | its functions are compiled into the new code, and `Spite.Class.instances` lists it |

A refused reload leaves the program exactly as it was, so a save that caught a file half-written is harmless: the
next save reloads.

### Changing a class's attributes

Add an attribute to a class, remove one, rename one or change its type, and save: the reload moves every object of
the class the running program holds to the new attributes, and the program carries on with the same objects: the
same identity, the same references to them. An attribute that
kept its name and its type keeps its value. A new attribute, or one whose type changed, holds its default, as it
would in an object made now. A removed attribute is released, once every object has moved. The answer names every
class it moved and what its objects kept, so nothing changes silently.

```gdscript title=live_party/hero.spite
var name = ""
var level = 1

func Hero(starting_name: String) {
    name = starting_name
}

func describe(): String {
    return "{name} at level {level}"
}
```
```gdscript title=live_party/step.spite
var distance = 0.0

func Step(new_distance: Float) {
    distance = new_distance
}
```
```gdscript title=live_party/live_party.spite entry
var console = Console()
var hero = Hero("Ann")
var steps = List<Step>()

func LiveParty() {
    load "../spite_standard_library/windows"
    var first = Step(1.5)
    steps.append(first)
    var second = Step(2.5)
    steps.append(second)
    var described = describe()
    console.print(described)
}

func describe(): String {
    return "{hero.describe()}, walked {walked()}"
}

func walked(): Float {
    var total = 0.0
    var index = 0
    while index < steps.count() {
        total = total + steps[index].distance
        index = index + 1
    }
    return total
}
```
```output
Ann at level 1, walked 4
```

This session runs against a copy of the program, started with `spite --repl`. The `Step`s the list holds move too,
wherever the compiler chose to keep them:

```text
spite> hero.level = 7

# hero.spite: a new attribute, 'var health = 100', and describe() says "{name} at level {level} with {health}"
# step.spite: a new attribute, 'var pace = 2.0', before 'distance'
spite> load "hero.spite"
rebuilt Hero, ...
moved every Hero to its new attributes: health is new and holds its default
spite> load "step.spite"
rebuilt Step, ...
moved every Step to its new attributes: pace is new and holds its default
spite> describe()
Ann at level 7 with 100, walked 4
```

**A renamed attribute keeps its value** through a map given with the reload, never in the program's code. A save
where, in one class, an attribute is gone while another is new could be a rename or a removal and an addition,
and a reload never guesses: it holds the change, swaps in nothing, and says what it holds. Rename `level` to
`rank` in `hero.spite` and save:

```text
spite> load "hero.spite"
the reload of Hero is held, since it would lose data: level is gone and rank is new: load "hero.spite", {Hero.attributes["level"]: "rank"} keeps what level holds, and load "hero.spite", {} lets it go
spite> load "hero.spite", {Hero.attributes["level"]: "rank"}
rebuilt Hero, ...
moved every Hero to its new attributes: level is now rank
```

The map pairs an attribute of the running class with the name of the new one, the same shape as a serializer's
map of names ([json.md](json.md#a-key-that-is-not-an-attributes-name)), and applies to that one reload. `rank`
holds the 7 `level` held. An empty map, `{}`, lets the old values go instead. Over the MCP the map is `reload`'s
`renames` ([mcp.md](mcp.md)). An attribute of the map that the running
class lacks, of another type than the new one, or one that the reload does not rename is refused by name.

What it costs: a change to a class's attributes compiles the whole program, since every class that reads them is
compiled again: about 40 seconds for a large game's server, against a few for a change to function bodies alone.
The saves after it are fast again: a reload that compiled the whole program becomes what the next one is compared
with ([How it works](#how-it-works)).

### Knowing what a reload rebuilt

A program that makes things from its own code (an engine that cooks assets with recipes written in Spite) needs
to know when that code changed, so it can make them again with the new code. Watching the files cannot tell it:
the watcher sees the save before the new code is swapped in. The standard library's `Reload` singleton answers from the swaps themselves: `generation()` is how many reloads the program has swapped in, and
`rebuilt_since(generation)` the qualified names of the classes rebuilt after that one, as `$type.name` gives them.

```gdscript title=recooking/recooking.spite entry
var console = Console()
var reload = Reload()
var cooked = 0

func Recooking() {
    load "../spite_standard_library/windows"
    cook_again_if_changed()
    console.print("cooked with generation {cooked}")
}

func cook_again_if_changed() {
    var rebuilt = reload.rebuilt_since(cooked)
    if rebuilt.contains("Recooking") {
        console.print("the recipe changed: cooking again")
    }
    cooked = reload.generation()
}
```
```output
cooked with generation 0
```

A loop that asks every pass costs a comparison of two numbers until a reload happens. In a build without
`--hot-reload` nothing is ever swapped in: `generation()` is `0` and `rebuilt_since` answers an empty list, both
worked out while compiling, so the check folds away with the branch it guards.

### How it works

- **One slot per function.** In a `--hot-reload` build every function is called through a slot the program can
  re-point: the program's own, the standard library's, and the helpers the compiler writes. The function the rest
  of the code calls is a one-line forwarder to its slot. A normal build is untouched: direct calls, and tree
  shaking. The slot costs about a nanosecond per call, one indirect call the C compiler cannot inline
  ([measured](https://github.com/Spite-Language/spite_spec/blob/master/specs/repl.md#live-reload-in-detail)),
  and only in a `--hot-reload` build.
- **Only what changed is compiled again.** The build writes two files beside the executable: `game.reload_host`,
  what the running program holds (every function with its C prototype, its classes and their layouts, and the
  facts its code was compiled against, below) and `game.reload_files`, a hash of each of the program's files. A
  reload runs the compiler that built the program with the same options and `--reload-at`, in the program's folder. It reads the
  program's files and compares their hashes first, so a save that changed nothing answers at once. Otherwise it
  compiles **only the classes of the changed files**: every class is read and checked, as the changed code needs,
  but no other class's functions are compiled: the new code reaches them in the running program. The C it writes
  is only the changed classes' functions and whatever those need that the running program lacks, with the types
  and declarations they use, and it compiles that into `game_reload_1.dll` (`.so` on Linux, `.dylib` on macOS).
  For a large game's server (about 460 files; 880,000 lines of C for the whole program) a changed system is swapped
  in about 6 seconds after the save (5 of them the compiler, 1 the C compiler and starting up), where compiling the
  whole program took 20 to 30 seconds, and a save that changed
  nothing answers in under a second.
- **A reload is compiled against what the running program was compiled with, and compiles the whole program when
  that changes.** Compiling one class depends on the rest: which functions can wait, which classes fit a shape,
  which attributes something reads, the call effects that decide whether a value must be counted, how a
  dictionary is keyed, which functions the program made for a class (a setter, a template instance, a `to_debug`).
  The build records these in the manifest and a reload starts from them. When the changed code changes one of
  them (or a parameter list, or the layout of a class), code in other classes compiled against the old one would
  be stale, so the reload compiles the whole program instead, and says why on the compiler's error output
  (`spite: compiling the whole program, since outside HotCounter changed`). A whole compile compares every function
  it writes with the running program's, and every function whose C changed is swapped in, whatever class it
  belongs to, a helper the compiler wrote included. Only a helper that keeps state of its own (a singleton's
  cache, a table the program fills as it runs) has no slot, since a new copy would start empty: a change that
  reaches one refuses the reload with `the change reaches '<name>', which the running program cannot swap:
  restart the program to take the change, or undo it`. Either way what swaps in is what compiling the whole
  program writes, which compiling a reload of every file both ways and comparing them checks
  (`SPITE_RELOAD_CHECK=<file>`).
- **A reload that compiled the whole program is the next one's baseline.** After it, every function the program
  runs is what that compile wrote, so the facts, the hashes and the class ids it compiled with describe the
  running program better than the build's do. It writes them beside its library (`game_reload_2.baseline`); when
  the program swaps the library in, it keeps them as `game.reload_baseline`, and the next reload compares with them
  instead of the build's manifest; the functions and slots the executable holds stay the build's. A save after a
  change that compiled everything is fast again. Starting the program again starts from the build
  (`game.reload_start`), since a new process runs the build's code.
- **One reload at a time.** A reload compares with the files and the baseline of the code the program will run,
  which swapping a library in writes, so a reload waits to compile until the library before it has been swapped in
  (or refused): the watcher, the REPL and the MCP never compile against the code before a swap that is still on its way,
  and never read those files while a swap writes them.
- **Every object of a program class can move.** In a `--hot-reload` build each object of the program's own classes
  carries two hidden words after its header: where its attributes live when they have moved, and its place in a
  list of the class's live objects, which each allocation adds to and each release takes from. Code reads an
  attribute through `Monster___fields(object)`, which in the build is the object itself. When a reload changes a
  class's attributes, the new code reads through a function instead (the moved attributes when there are some,
  the object otherwise), and the reload lays each live object's attributes out anew in a block of their own and
  points the object at it: the object stays where it was, so every reference to it (a local of a function
  running now, a list, another object) still holds it. The functions that read a class's
  attributes without being its own (its allocation, release and copy, reflection, a union's dispatch,
  a template of the standard library made for it, `List<Step>`) are called through slots too, so they move
  with it. A reload moves nothing until it has installed every slot, and releases nothing a removed attribute held
  until every object has moved. All of it is in a `--hot-reload` build only.
- **The swap happens where the program waits**, or at the end of a pass of one of its
  loops: the program loads the library
  with the operating system's loader, the one `DynamicLibrary` uses, hands it the addresses of the program's
  functions, and re-points the slots. Nothing runs halfway through a step. The library is compiled on a helper
  thread while the program keeps running, so a game keeps drawing frames while its code is rebuilt.
- **The watcher is the standard library's [`FileSystemWatcher`](standard_library.md#watch-files-and-folders)**, the one any
  program can use, started on a thread of its own: `ReadDirectoryChangesW` on Windows, `inotify` on Linux and
  `kqueue` on macOS, with no polling. It is watching before the program's entry runs and before the MCP listens,
  so a save made once the MCP answers is never missed. A save made while the build was still compiling is not
  missed either: when the program starts, each of its files is held against the text the build compiled, and if
  any differs the watcher compiles once before it waits, so the MCP's `evaluate` waits for that compile too. The
  operating system wakes the thread; a burst of changes is waited out until 100 ms pass without one, so a save
  that writes a file in pieces reloads once. Only a change to a `.spite` file, or to a folder, compiles again, and
  never one under a `.spite/` folder: the compiler writes its own output there, so a compile started by its own
  output (which, with an error in the saved code, would refuse the same code again and again) cannot happen. When the program ends, the watcher is stopped before anything it
  uses is let go, after the compile it is running finishes: it looks every quarter of a second whether the program
  is ending, so ending a `--hot-reload` program can take that long. The program's own folder is watched with every folder below it, and so is every folder it `load`s, except
  a repository's checkout under `.spite/git/` ([packages.md](packages.md#loading-a-repository-pinned-to-a-commit)),
  which is read-only: a pinned commit never changes, so there is nothing to reload there, and a change to that
  package is a new commit and a restart. Each reload's library is written beside the executable, into
  `.spite/build/` by default; only when that lies inside a watched folder does it wake the watcher once more, for a check that finds nothing changed.
- The compiler, its options, the program's folder and the executable are recorded in the build, and a reload
  compiles in that folder, so the program can run from any folder.

---

Next: [The language's MCP](mcp.md), how an AI runs code in a running program.
