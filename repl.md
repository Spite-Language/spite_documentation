# REPL and live reload

`spite --repl` starts your program with a console attached. You type Spite at the `spite> ` prompt, and the line
runs inside the running program: an expression prints its value, a statement just runs. There are no commands to
learn, because every line is ordinary code: to see what a class holds you ask reflection, to print you call
`console.print`, and to make an object you make one.

```text
spite> monsters.count()
2
spite> player_name = "Aria"
spite> player_name
Aria
spite> monsters[1]
Monster { name: "Orc", health: 50 }
spite> var ogre = Monster("Ogre", 70)
spite> ogre.roar()
Ogre roars!
```

`--hot-reload` swaps the code you save into a running program, keeping its state
([Live reload](#live-reload---hot-reload)), and `--repl-port` serves the same lines to a tool or an AI over a local
socket ([Remote](#remote---repl-port)).

The REPL and live reload compile new code into a program while it runs, so they run on Windows for now (Linux and
macOS: [spite_compiler#297](https://github.com/Spite-Language/spite_compiler/issues/297)). A REPL or live-reload
build is slower on purpose: it trades speed for what it tells you while the program runs. Never measure
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

A REPL build is an [inspectable build](compiler.md#development-builds-and-tree-shaking): nothing is tree-shaken,
so every function can be called from a line, and the executable is larger than a production build of the same
program. An attribute the program's own code never reads is still an error
([style.md](style.md#nothing-unused)), which is why `Monster` above has an `is_alive()`.

## Typing Spite into a running program

```
spite --repl
```

The program starts and the prompt appears at once. A line you type runs the next time the program waits (a
`program.sleep`, a file or socket read) or finishes a pass of one of its loops, so it sees the program between two
steps, never halfway through one. Once the constructor has returned, a line runs as soon as you press Enter.

- **An expression prints its value**: text as it is, anything else as `to_debug()` shows it (`3`, `null`,
  `Monster { name: "Goblin", health: 30 }`, `["a", "b"]`). A statement (an assignment, a call that answers
  nothing, an `if`, a `while`) prints only what it prints itself, and so does an expression whose text is empty.
- **`program` is the running entry instance**, and its attributes and functions can also be named bare:
  `program.player_name` and `player_name` are the same thing, an attribute or parameter typed as a union
  included (`spare = Circle(4)`). A name starting with `_`, and a function taking a
  variadic parameter, are reached through their own code, not from a line.
- **A line is ordinary code.** It can call any function, `console.print("a", 1, true)` included, make objects,
  name any class, and reach a singleton by its name, `World().entity_count()`, exactly as the program's own code
  does.
- **`var` and `func` stay.** `var strongest = monsters[1]` keeps `strongest` for every line after it, and
  `func healthy(): Integer { ... }` declares a function the next lines can call. Declaring the same name again
  replaces it. A `var` keeps a number, a `Boolean`, text, an enum value or an object of a class (a `List` and a
  `Dictionary` included); a value of a `union`, a `type` or a function lives only in its own line.
- **A line that opens a block continues** on the next one, under a `...> ` prompt, until its braces close:

```text
spite> func healthy(): Integer {
...>     var total = 0
...>     var index = 0
...>     while index < monsters.count() {
...>         var monster = monsters[index]
...>         total = total + monster.health
...>         index = index + 1
...>     }
...>     return total
...> }
spite> healthy()
80
```

- **A line meets every rule the program's own code meets**: privacy, no call inside an argument, names, nothing
  unused, and the rest. `monsters[0].roar()` is refused, since `monsters[0]` may be missing, and the error's place is
  the line as you typed it, where a file's error names the file and line number (a block or a `func` names its own
  line). A typed line is never rewritten, so a check the compiler would remove from a file (a `crash` on a value that
  cannot be null) is refused, naming the fix. Rules about where things sit in a file do not apply to a line, which
  has no file: a line reaches a singleton by its name, `Program().exit(0)`. The forwarders and storage the compiler
  writes around your line are its own code and are not checked:

```text
spite> console.print(monsters.count())
console.print(monsters.count()): error: 'monsters.count()' is called inside an argument of 'console.print': compute it first into a named 'var' and pass the name
```

- **A line that does not compile** prints the compiler's error, and nothing changes: the session and the program
  are as they were before the line.

Questions about the program itself are lines of Spite too, answered by reflection
([reflection.md](reflection.md)):

```text
spite> Monster.attributes.count()
2
```

`Monster.attributes` and `Monster.functions` describe a class, and `Program().live_allocations()` and
`live_bytes()` answer what the heap holds ([memory.md](memory.md)).

### Input and ending

While the REPL is open, everything you type is the REPL's. A `Console.read_line()` in the program answers `null`,
as if the input had ended, instead of waiting for the keyboard.

The end of input (Ctrl+Z then Enter on Windows), or a line calling `Program().exit(0)`, ends the program: the end of
input the normal way, with everything released and memory balanced. **Ctrl+C ends it at once**, whatever is
running, your line or the program's own code, with exit code 130.

## Loading what you changed

The REPL does not watch your files: what you are poking at does not change under you because you saved something.
A line is compiled against the files as the program loaded them, and a line typed after a save that was not loaded
yet is refused, naming the file. When you want a change, load it:

```text
spite> load "monster.spite"
rebuilt Monster
spite> ogre.roar()
Ogre roars louder!
```

The path is read from the entry's folder, as a program's `load` reads it, and the file keeps its place in the
load order, so what goes live is what a fresh start would choose
([monkey patching](packages.md#monkey-patching-mods) included). A new file of the program's folders is loaded the
same way. Everything the program held stays: objects, singletons, and the `var`s you typed. What a reload can
change, and what it refuses, is the same as for a save ([What a reload can change](#what-a-reload-can-change)).

If you would rather have every save swapped in as you make it, start the REPL with the watcher too:

```
spite --repl --hot-reload
```

`load` still compiles a file at once when you do not want to wait for the watcher.

## Remote: `--repl-port`

```
spite --repl-port=4000
```

The same lines, sent over a socket instead of typed: a tool, a test or an AI on the same machine drives the running
program with them. Before the constructor runs the program listens on `127.0.0.1:4000` **only**, takes one client
at a time, and there is **no authentication**: anyone who can reach that port on that machine can run any code in
the program. This is a local development tool, never something to expose past `127.0.0.1`. Each line is run where
a typed one is ([Typing Spite into a running program](#typing-spite-into-a-running-program)), in the same session:
a `var` declared over the socket is there for the next line, and for the console when the build has both
(`--repl --repl-port=4000`). When the constructor returns the process stays up, serving lines, until a line ends it
(`Program().exit(0)`) or, with `--repl` as well, the console's input ends. The port is a `Build` field, so changing
it means building again, and a port another program holds stops the program before its constructor runs, with
`error: the REPL could not listen on 127.0.0.1:<port>`. `--repl-port` carries the swap machinery as `--repl` does,
and the file watcher only with `--hot-reload`.

Talk to it with the compiler's own client:

```
spite --connect=4000
spite --connect=4000 --command="player_name"
```

With no `--command` it is interactive: a `spite> ` prompt that sends each line and prints the answer as
`value (type)` or `error: message`. With `--command` it sends that line, prints the raw JSON answer, and exits,
which is the form tests and AI clients use. `--connect` runs from any folder, since it builds nothing. Nothing
listening on the port is `error: nothing is listening on 127.0.0.1:4000` and exit code 1, and so is a port outside
1 to 65535: `error: --connect takes a port number from 1 to 65535: spite --connect=4000` (`--repl-port` is refused
the same way, with its own name).

## The wire protocol

One line in, one line of JSON out: `{"ok":true,"value":"...","type":"Integer"}` when the line ran,
`{"ok":false,"error":"..."}` when it did not compile. `value` is what the console would print (empty for a
statement) and `type` the class of an expression's value. A line that opens a block is answered
`{"ok":true,"value":"","type":"","continues":true}`, and the block's last line is answered with what the whole block
did. A multi-line answer uses `\n` inside its one JSON string, never a real newline on the wire. The wire format is
JSON, but it is not a promise: it is free to become whatever an AI client reads best.

In a build with `--hot-reload`, a line first swaps in every saved file the watcher has seen settle, so a tool that
saves a file and then sends a line runs it against the new code:

```text
$ spite --connect=4000 --command="monsters.count()"
{"ok":true,"value":"2","type":"Integer"}

$ spite --connect=4000 --command="player_name = \"Aria\""
{"ok":true,"value":"","type":""}

$ spite --connect=4000 --command="monstrs"
{"ok":false,"error":"monstrs: error: 'monstrs' is not a local, an attribute or a function of 'Spite.Session'"}
```

A line that does not compile answers the compiler's error, and the program and the session stay as they were. For
an AI driving this: send one line per `spite --connect=4000 --command="..."` call, read the JSON back, and branch on
`"ok"`. Ask reflection first (`Monster.attributes.count()`) to see what is there.

## Live reload: `--hot-reload`

`--hot-reload` builds a program that can take new code while it runs. Save a file of the program, and the program
is compiled again; the functions whose machine code changed are copied into the running program and swapped in
the next time it waits or ends a pass of a loop, the same moments a REPL line runs at. Objects, singletons
and everything a line changed stay as they were: only functions change, and `Environment` reads its settings
again when a save gives one a new default. Every function sits in a slot so it can
be swapped, which makes the build slower; measure performance only with a production build
([Measure only a production build](compiler.md#measure-only-a-production-build)).

```
spite --hot-reload --repl-port=4000
```

It works with or without a REPL. Each swap, or each save it refuses, is one line on the program's error output
(`spite: rebuilt Monster`). With `--repl-port` as well, a line sent over the socket first swaps in every save the
watcher has seen, so a tool that saves a file and sends a line never polls. `--repl` and `--repl-port` carry the
same machinery without the watcher, so a file changes there only when a line `load`s it
([Loading what you changed](#loading-what-you-changed)). A development build does not include any of this unless
you ask: the slots cost every call a jump. `--hot-reload` keeps every function (it implies `--development`), since
a new version of a class may call one nothing called before. It is slower than a production build, since it
inlines no call and every call goes through a slot a reload can re-point: a `--hot-reload` build trades run speed
for live information, and speed is measured on production builds. A reload compiles with exactly the flags the
program was built with. Live reload runs on Windows.

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
| a new function on an existing class | the new code calls it, and so can a REPL line |
| a function you deleted | whatever still holds it (a function value, the REPL) keeps its last code; the answer says `removed Greeter.old` |
| an attribute's default value | instances made after the reload get the new default |
| a setting's default in `environment.spite` | every setting is read again the way it is when the program starts: from the command line it was started with, the environment, a `.env` it loaded, or the new default |
| an enum: a value added after the others | taken; the objects that hold the enum keep their values |
| an enum: a value inserted before the others, removed or reordered | refused: `the reload changes the values of Gauge.Mood other than adding new ones last ... start the program again, or add the new values after the others` |
| an enum: a value's number (`'red' = 1` saved as `'red' = 5`) | refused: `the reload changes Palette.Shade's 'red' from 1 to 5 ... start the program again, or give 'red' its number 1 back` |
| a class's attributes: added, removed, renamed or retyped | refused: `the reload changes the attributes of Counter (total is new), and the objects the program holds keep the attributes they were made with: start the program again to take the new attributes` |
| a file that does not compile | refused with the compiler's error, and the program keeps all of its code |
| a function of a class the program reopens, a class of the standard library or a number (`func doubled(): Integer` in `integer.spite`) | the whole program is compiled again and every function whose code changed is swapped in, the standard library's and the compiler's helpers included |
| nothing the program runs | `unchanged`, or, when a file changed but no code the program runs did, `no code the program runs changed, so nothing was rebuilt` |

A refused reload leaves the program exactly as it was, so a save that caught a file half-written is harmless: the
next save reloads.

### Changing a class's attributes

A reload swaps functions. It does not move the objects the running program holds to a new layout, so a save that adds,
removes, renames or retypes an attribute of a class is refused as a whole, the program keeps all of its code, and the
answer names the class and the attributes:

```text
spite> load "counter.spite"
error: the reload was refused, and the program keeps all of its code: the reload changes the attributes of Counter (total is new), and the objects the program holds keep the attributes they were made with: start the program again to take the new attributes
```

Start the program again to take the new attributes. An attribute's *default value* is not a layout change: instances
made after the reload get the new default, and the ones already made keep what they hold. An enum is the same: a value
added after the others is taken, and any other change to an enum's values is refused, a changed number included
(the answer names both numbers), since every value the program holds keeps the number it was made with.

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

- **One slot per function.** In a `--hot-reload` build every function starts with a jump through its slot, eight
  bytes of the executable that hold the address of the function's body: the program's own functions, the standard
  library's, and the helpers the compiler writes. Every call, function value and thread entry names the function's
  start, so all of them follow a swap. A normal build is untouched: direct calls, and tree shaking. The cost is one
  indirect jump per call, only in a `--hot-reload` build.
- **What the build records.** Beside the executable, `game.reload_host` lists what the running program holds: every
  function with its start, its slot and a digest of its machine code, the state its code reaches by name, the system
  functions it imports, every class's layout, every enum's values with their numbers and a digest of each of the program's files.
- **A reload compiles the whole program.** It runs the compiler that built the program, in the program's folder,
  with the same flags and `--reload-at`. It compares the program's files with the manifest first, so a save that
  changed nothing answers `unchanged`. Otherwise it compiles the program and compares every function's digest with
  the manifest's: the functions whose code changed, and the ones the running program lacks, are written into
  `game.reload_<n>` beside the executable, laid out at the address the program keeps for new code, and every
  other function keeps the code the program runs. It answers `rebuilt Monster` (the classes of the rebuilt
  functions), `removed Greeter.old` for a function that is gone, or `no code the program runs changed, so nothing
  was rebuilt`. The next manifest is written as `game.reload_pending`, and takes the place of `game.reload_host`
  once the code is swapped in, so the manifest always describes the code the program runs.
- **What else a reload refuses.** Besides a file that does not compile, a class's attributes and an enum's values
  (above), a reload is refused, and the program keeps all of its code, when the routines every function shares
  (allocating, freeing, stopping) are compiled differently, when it reaches state the compiler did not name, when
  the room kept for new code is full, and when a function writes more or fewer object literals of one set of
  attributes than the running one. Each answer ends with what to do, most often `start the program again`.
- **New code may call a system function the program never called.** A line or a save that needs a lock, say, which
  the running program never imported, gets it when it swaps in: the program looks the function up in its system
  library then (`LoadLibraryA` and `GetProcAddress` on Windows), and a program that never reloads pays nothing for
  it.
- **The swap happens where the program waits**, or at the end of a pass of one of its loops. A call that waits, such
  as `console.read_line()`, a `File` read or write, or a socket's receive, is a place to swap just before and just
  after it, and `Program.sleep` swaps while it sleeps, so a program with no loop picks up a save before its next
  statement. There the program copies the new code in and re-points the slots. Nothing runs halfway through a
  step, a call already running finishes on the old code, and new code is never unloaded, since values it made may
  still be held. The compile runs there too: the program waits for the compiler while it reloads.
- **The watcher is the standard library's [`FileSystemWatcher`](standard_library.md#watch-files-and-folders)**, the
  one any program can use, with no polling. The program's own folder is watched with every folder below it, and so
  is every folder it `load`s. Only a saved `.spite` file reloads, and never one under a `.spite/` folder: the
  compiler writes its own output there, and a repository's checkout under `.spite/git/` is a pinned commit
  ([packages.md](packages.md#loading-a-repository-pinned-to-a-commit)) that never changes. A file still being
  written is waited out until it is quiet, so a save that writes a file in pieces reloads once.
- **A crash inside reloaded code** names its file, line, class and function like any other, and its offset in the
  reload ([failure.md](failure.md#what-a-native-fault-reports)).
- The compiler, its flags, the program's folder and the executable are recorded in the build, and a reload
  compiles in that folder, so the program can run from any folder.

---

Next: [Testing](testing.md), a test is a package that crashes.
