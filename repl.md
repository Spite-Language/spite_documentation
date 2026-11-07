# REPL and live reload

`spite --repl` runs the program, then answers `attributes`, `functions` and `classes`, `help`, `exit`, paths
such as `monsters[0].health` or `program.player_name`, a singleton by its class's name (`World().entity_count()`,
`Tick().step_milliseconds`), assignment of a number, Boolean, text or enum literal (`monsters[0].health = 5`, which
prints the value read back), and calls with literal arguments that print what they return, chained like any path
(`monsters[0].roar()`, `monsters.count()`, `World().position_of(1).across`), written in Spite by the
compiler into the build that asks for it. `--repl-port` answers the same commands over TCP, one JSON line each,
answered where the program waits or at the end of a loop's pass
([concurrency.md](concurrency.md#the-repl-answers-at-the-waits)), and `spite --connect` is its client. `--hot-reload`
swaps the functions whose code changed into the running program, keeping its state, when a file is saved or when the
REPL is sent `reload` ([Live reload](#live-reload---hot-reload)); a change to a class's attributes is refused, and
takes a restart. `bytes <path>` answers a value's address, size, bytes and layout as JSON
([Native memory](#native-memory-bytes)). `describe Monster`, `enums` and `memory` describe a class, the program's
enums and the heap in use; an assignment takes `null` into a `T?`, a literal into a list or dictionary element, and
another path's instance (`follower = leader`); a path walks into a union's active member. A reload swaps in a change
to an enum's values, each held value keeping its meaning. With `--hot-reload`, `eval <expression>` and
`run <statement>` compile code typed at the prompt into the running program and run it once
([below](#code-typed-at-the-prompt)).

```text
spite> monsters[0]
Monster { name: Goblin, health: 30 }
spite> monsters[0].health = 5
5
spite> monsters[0].roar()
Goblin roars!
spite> monstrs
no attribute 'monstrs' in program: console, player_name, player_age, monsters
```

The REPL inspects and drives the *running* program, local (`--repl`, standard input) and remote
(`--repl-port`, TCP). It is ordinary Spite, `ReadEvaluatePrintLoop` in the standard library, walking the program
through reflection ([reflection.md](reflection.md)). It reads and changes the running state, and with
`--hot-reload` it also swaps in code you edited while the program keeps running
([Live reload](#live-reload---hot-reload)).

A REPL or live-reload build is slower on purpose: it trades speed for what it tells you while the program runs.
Never measure performance with one; benchmarks use a production build
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

What the loop needs to reach a live value (the attributes and functions of every class the loop can reach,
and the member templates that fit each list's elements, so `monsters.sum_health()` works at the prompt) is
compiled only into a `--repl`/`--repl-port` build, never into a normal one, so it costs nothing when you are not
debugging. A REPL build is also an [inspectable build](compiler.md#development-builds-and-tree-shaking):
nothing is tree-shaken, so every internal can be reached, and the executable is larger than a production build of
the same program. For the same reason the REPL's reads count only in that build: an attribute the program's own
code never reads is still an error in a normal one ([style.md](style.md#nothing-unused)), which is why `Monster`
above has an `is_alive()`.

## Local: `--repl`

```
spite --repl
```

Runs the constructor normally; when it returns, instead of exiting, reads commands from stdin with a
`spite> ` prompt until `exit` or end of input, then drops everything and exits cleanly (memory balanced, just
like a normal run).

## Remote: `--repl-port`

```
spite --repl-port=4000
```

Before the constructor runs, the program listens on `127.0.0.1:4000` **only**, and a background thread takes
one client at a time, and there is **no authentication**. Anyone who can reach that port on that machine can read
and mutate the running program. This is a local debugging tool for you and an AI on the same machine, never
something to expose past `127.0.0.1`. The program keeps running while it is served, and each command is answered
on the program's own thread the next time it waits (a `program.sleep`, a `Console.read_line()`, a file or socket
read) or reaches the end of a pass of one of its loops, so a command never sees it halfway through a step
([concurrency.md](concurrency.md#the-repl-answers-at-the-waits) serves a frame loop between frames and a busy loop
between passes). When the constructor returns the process stays alive until a client sends
`exit`, which flushes what the program printed and ends it with exit code 0. The port is part of the build: a
program built with `--repl-port` always listens on that port, and a port another program holds stops it before
its constructor runs, with `error: the REPL could not listen on 127.0.0.1:<port>`. With `--repl` as well, the
console loop runs first, and after its `exit` the process keeps serving the port. What serving costs, and only in
a `--repl-port` build: one thread waiting on the socket, and one load of a flag at the end of every pass of the
program's own loops, which calls into the REPL only when a command is waiting.

Talk to it with the compiler's own client:

```
spite --connect=4000
spite --connect=4000 --command="program.player_name"
```

With no `--command` it is interactive: a `spite> ` prompt that sends each line and prints the answer as
`value (type)` or `error: message`. With `--command` it sends that one command, prints the raw JSON line, and
exits, which is the form tests and AI clients use. `--connect` runs from any folder, since it builds nothing.
Nothing listening on the port is `error: nothing is listening on 127.0.0.1:4000` and exit code 1, and a port
outside 1 to 65535 is refused, for `--connect` as for `--repl-port`: `error: --repl-port takes a port number from
1 to 65535: spite --repl-port=4000`. The port is a `Build` field, so changing it means building again.

## The wire protocol

One command per line in, one line of JSON out: `{"ok":true,"value":"...","type":"Integer"}` on success,
`{"ok":false,"error":"..."}` on failure. A multi-line value (such as the answer to `attributes`) uses `\n`
inside that one JSON string, never a real newline on the wire. `type` is the class of the value; it is empty
for an answer that is not a value (`help`, `attributes`, `functions`, `exit`) and `Nothing` for a call that
returns nothing. The entry instance is always rooted at the fixed name `program`, regardless of what the entry
class is actually called, and a path may leave `program.` out. The wire format is JSON, but it is not a
promise: the format is free to become whatever an AI client reads best, binary included.

The commands are the ones `--repl` answers:

- **paths**: `program`, `program.monsters`, `program.monsters[0].health`. A `T?` that holds a value is walked
  through, and a `Dictionary`'s entries are walked by key: `program.settings.volume`.
- **calls**: `program.monsters.count()`, `program.monsters[0].roar()`, with literal arguments. A call that answers
  an object is walked on like any path: `World().position_of(1).across`. The standard library's functions are
  called the same way, on any of its values a path reaches: `player.name.upper_case()`.
- **singletons**, by their class's name: `World()`, `Tick().step_milliseconds`, `Ui.Panel().title`,
  `Column<Position>().values[0]`, and one made with arguments by its arguments: `Channel(1).number`. The name binds
  the instance the program already has and never makes one.
- **a dictionary's** `count()`, `keys()` and `has(key)`: `scores.has("ann")`.
- **assignment** of a number, `Boolean`, text or enum literal: `program.player_name = "Aria"`, through
  `set_<attribute>` when the class declares one, answering the value read back afterwards. A text literal is
  written as in a program, escapes included (`"a \"quoted\" word"`). The target may be a list or dictionary element
  (`names[1] = "cat"`, `scores["ann"] = 9`), a `T?` takes `null`, and an attribute or element of a class takes
  another path's instance of that class (`follower = leader`), which then holds the same object, or one made at the
  prompt (`follower = Circle(3)`) in a program built with `--hot-reload` and `--repl-port`
  ([Code typed at the prompt](#code-typed-at-the-prompt)).
- `describe Circle` (a class's attributes and functions, with their types and signatures), `enums` (each
  of the program's enums with its values) and `memory` (the live allocations and the bytes they hold).
- `attributes`, `functions` (of the program, or of any path: `functions World()`), `classes` (the singletons the
  prompt can bind, then the program's other classes), `help`, `exit`.
- `reload`, `last_reload` and `wait_reload`, which answer in a `--hot-reload` build
  ([Live reload](#live-reload---hot-reload)) and fail everywhere else with `'reload' answers in a program built with --hot-reload, which swaps its code while it
  runs, and this one was not`.

### What the answers look like

An instance prints one level deep, `Monster { name: Goblin, health: 30 }`: an object inside it prints as
`Monster {...}`, a list or dictionary at any depth as `List<Monster>(2)` or `Dictionary<String, Integer>(2)`, a `T?`
as `null` or the value it holds, a union as its active member, and text without quotes. A call that answers
nothing prints nothing, and nothing may follow it in the same command: `World().spawn() ran, and it answers
nothing, so nothing can follow it`.

Every mistake is one line, never a crash, and says what is there instead:

- an unknown attribute lists the ones that exist: `no attribute 'monstrs' in program: console, player_name,
  player_age, monsters`;
- an unknown function: `no function 'nope' in program: 'functions' lists them`;
- an index past the end: `index 5 is out of range: names holds 2`;
- a literal of the wrong kind: `type mismatch: target.health is an Integer, and "x" is not one`;
- calling an attribute: `target.name is not callable`;
- `null` into a value that is not a `T?`, and another path's instance of a different class, are refused naming the
  types.

The prompt reaches what code outside a class may reach: a function or attribute whose name starts with `_` is
neither listed nor called, and neither is `drop()`. A singleton is named by its qualified name (`Ui.Panel()`), or by
its name alone when one singleton has it; a singleton the program has not made yet is refused, `Tick has not been
made yet: nothing in the program has asked for it, and the prompt never makes one`, and so is a class that is not a
singleton. `classes` lists the singletons the prompt can bind, `not made yet` after one the program has not made,
then the program's other classes; of the standard library's singletons it lists only `Environment` and `Build`.

## A worked debugging session

This session runs against the program above, built with `--repl-port`: every answer below is the exact line the
program sends back.

```wire repl_program
$ spite --connect=4000 --command="program.player_name"
{"ok":true,"value":"Hero","type":"String"}

$ spite --connect=4000 --command="program.monsters.count()"
{"ok":true,"value":"2","type":"Integer"}

$ spite --connect=4000 --command="program.monsters[0]"
{"ok":true,"value":"Monster { name: Goblin, health: 30 }","type":"Monster"}

$ spite --connect=4000 --command="program.monsters[0].roar()"
{"ok":true,"value":"Goblin roars!","type":"String"}

# Debugging a suspiciously low health value: mutate it and confirm.
$ spite --connect=4000 --command="program.monsters[0].health = 5"
{"ok":true,"value":"5","type":"Integer"}

$ spite --connect=4000 --command="program.monsters[0].health"
{"ok":true,"value":"5","type":"Integer"}

# A typo'd path is one line, never a crash.
$ spite --connect=4000 --command="program.monstrs"
{"ok":false,"error":"no attribute 'monstrs' in program: console, player_name, player_age, monsters"}

$ spite --connect=4000 --command="attributes"
{"ok":true,"value":"console: Console = Console {...}\nplayer_name: String = Hero\nplayer_age: Integer = 20\nmonsters: List<Monster> = List<Monster>(2)","type":""}

$ spite --connect=4000 --command="exit"
{"ok":true,"value":"","type":""}
```

For an AI driving this: issue one command per `spite --connect=4000 --command="..."` call (or hold one interactive
session open), read the single JSON line back, and branch on `"ok"`. `attributes`, `functions` and `classes` first, to see
what is there (only functions whose arguments are numbers, Boolean, text or an enum can be called), then walk paths
down to the value in question before mutating anything.

## Singletons: where a game's state lives

A game's state rarely hangs off its entry class: the entry makes an app and runs it, and the world, the clock and
the columns of components are singletons. The prompt binds a singleton by its class's name, the way the program's
own code does (`World()`, `Ui.Panel()`, `Column<Position>()`), and reads, calls and assigns through it. It
binds the one instance the program already made; it never makes one, so reading a singleton the program has not
asked for yet says so instead of creating it.

```gdscript title=repl_world/position.spite
var across = 0
var along = 0

func Position(new_across: Integer, new_along: Integer) {
    across = new_across
    along = new_along
}

func distance_to(other_across: Integer, other_along: Integer): Integer {
    var across_gap = across - other_across
    var along_gap = along - other_along
    return across_gap * across_gap + along_gap * along_gap
}
```
```gdscript title=repl_world/tick.spite
singleton

var step = 0
var step_milliseconds = 16
```
```gdscript title=repl_world/world.spite
singleton

var positions = List<Position>()
var tick = Tick()

func spawn(across: Integer, along: Integer) {
    var made = Position(across, along)
    positions.append(made)
    tick.step = tick.step + 1
}

func entity_count(): Integer {
    return positions.count()
}

func position_of(entity: Integer): Position? {
    return positions[entity]
}

func step_length(): Integer {
    return tick.step_milliseconds
}
```
```gdscript title=repl_world/repl_world.spite entry
var console = Console()
var world = World()

func ReplWorld() {
    load "../spite_standard_library/windows"
    world.spawn(3, 4)
    world.spawn(7, 1)
    var count = world.entity_count()
    console.print("entities", count)
}
```
```output
entities 2
```

This session runs against it, built with `--repl-port`:

```wire repl_world
# The entry class holds a console and the world; the game's state is in its singletons.
$ spite --connect=4000 --command="attributes"
{"ok":true,"value":"console: Console = Console {...}\nworld: World = World {...}","type":""}

$ spite --connect=4000 --command="World().entity_count()"
{"ok":true,"value":"2","type":"Integer"}

$ spite --connect=4000 --command="World().position_of(1)"
{"ok":true,"value":"Position { across: 7, along: 1 }","type":"Position?"}

$ spite --connect=4000 --command="World().position_of(0).distance_to(0, 0)"
{"ok":true,"value":"25","type":"Integer"}

$ spite --connect=4000 --command="Tick().step_milliseconds = 50"
{"ok":true,"value":"50","type":"Integer"}

$ spite --connect=4000 --command="functions World()"
{"ok":true,"value":"spawn(across: Integer, along: Integer): Nothing\nentity_count(): Integer\nposition_of(entity: Integer): Position?\nstep_length(): Integer\nto_debug(): String","type":""}

$ spite --connect=4000 --command="Position()"
{"ok":false,"error":"'Position' is a class, not a singleton: the prompt never makes an object, it reads the ones the program holds, and 'classes' lists the singletons"}

$ spite --connect=4000 --command="exit"
{"ok":true,"value":"","type":""}
```

A singleton made with arguments has one instance for each argument list
([classes_and_files.md](classes_and_files.md#one-instance-per-argument-values)), and the prompt names each by its
arguments, as the program does: `Channel(1)` is the instance the program's `Channel(1)` made, and an argument list
the program never asks for has no instance to read. A dictionary answers `keys()` and `has(key)` beside `count()`,
and a text literal at the prompt is written as in a program, with its escapes:

```gdscript title=repl_radio/channel.spite
singleton

var number = 0
var messages = List<String>()

func Channel(channel_number: Integer) {
    number = channel_number
}
```
```gdscript title=repl_radio/repl_radio.spite entry
var console = Console()
var news = Channel(1)
var music = Channel(2)
var volumes = Dictionary<String, Integer>()
var station = "city"

func ReplRadio() {
    load "../spite_standard_library/windows"
    news.messages.append("storm tonight")
    volumes["news"] = 7
    volumes["music"] = 4
    var tuned = volumes.count()
    console.print(station, "tuned", tuned, "and", music.number)
}
```
```output
city tuned 2 and 2
```

```wire repl_radio
$ spite --connect=4000 --command="classes"
{"ok":true,"value":"Build()\nChannel(1)\nChannel(2)\nReplRadio","type":""}

$ spite --connect=4000 --command="Channel(1).messages.count()"
{"ok":true,"value":"1","type":"Integer"}

$ spite --connect=4000 --command="Channel(3)"
{"ok":false,"error":"the program holds no Channel(3): a singleton made with arguments has one instance for each argument list the program asks for, Channel(1), Channel(2)"}

$ spite --connect=4000 --command="volumes.keys()"
{"ok":true,"value":"news\nmusic","type":"List<String>"}

$ spite --connect=4000 --command="volumes.has(\"talk\")"
{"ok":true,"value":"false","type":"Boolean"}

$ spite --connect=4000 --command="station = \"the \\\"late\\\" show\""
{"ok":true,"value":"the \"late\" show","type":"String"}

$ spite --connect=4000 --command="station = \"the {hour} show\""
{"ok":false,"error":"\"the {hour} show\" has a hole, and the prompt reads a text literal as it is written: \\{ writes a brace, and 'eval' makes text from values"}

$ spite --connect=4000 --command="exit"
{"ok":true,"value":"","type":""}
```

## Native memory: `bytes`

`bytes <path>` answers where a value lives and what its bytes are, as one JSON object an agent can diff between two
processes, a client's player against the server's, while it rubber-bands. For an instance it gives the class,
the address, the size, the bytes in hexadecimal, and each attribute's offset, size and value; for a list, the list
object and its buffer of items; for a number or text held by an attribute or a list, its own slot:

```text
spite> bytes World().positions[1]
{"class":"Position","address":"0x1e8647ec050","bytes":16,"hex":"04000000580000000700000001000000","fields":[{"name":"across","class":"Integer","offset":8,"bytes":4,"value":"7"},{"name":"along","class":"Integer","offset":12,"bytes":4,"value":"1"}]}
spite> bytes World().positions
{"class":"List<Position>","address":"0x1e8647f6100","bytes":40,"hex":"0300...","buffer":{"address":"0x1e8647f5f50","bytes":16,"shown":16,"hex":"f0bf7e64e801000050c07e64e8010000"}}
spite> bytes Tick().step_milliseconds
{"class":"Integer","address":"0x1e8647ebf1c","bytes":4,"hex":"10000000","value":"16"}
spite> bytes 0x1e8647ebf10 16
{"address":"0x1e8647ebf10","bytes":16,"hex":"02000000590000000200000010000000"}
spite> bytes 0x10 8
unreadable: 0x10 for 8 bytes
```

The first eight bytes of every object are its header, the reference count and the class's number. A read only
reads, and it never crashes the program it inspects: the bytes are copied through the operating system's checked
read, and an address that is not readable answers `unreadable`. At most 4096 bytes are shown at a time.

## Breakpoints

A program built with `--hot-reload` and a REPL (`--repl` or `--repl-port`) can be stopped at a line while it runs:

```text
$ spite --connect=4000 --command="break ticker.spite:16"
{"ok":true,"value":"the program stops before the statement on ticker.spite:16","type":""}

$ spite --connect=4000 --command="where"
{"ok":true,"value":"ticker.spite:16","type":""}

$ spite --connect=4000 --command="ticker.total"
{"ok":true,"value":"2","type":"Integer"}

$ spite --connect=4000 --command="continue"
{"ok":true,"value":"continued from ticker.spite:16","type":""}
```

`break <file>:<line>` compiles a stop into the function that line belongs to and swaps it in, the way a reload
swaps in a saved file; a program that never sets one carries nothing for it. The file is one of the program's own,
named by the end of its path, and the line is one a statement starts on; anything else is refused and the program
keeps its code. The next time any code reaches the line, the program stops before it and waits for the prompt:
`where` answers where it stopped, `locals` lists `self` and every local in scope there, a line each (`next: Integer = 3`), every other command answers
as usual, and `continue` lets the program go on. `breaks` lists the breakpoints and `clear` (or
`clear ticker.spite:16`) removes them, compiling the code again without the stop. A call made at the prompt that
reaches a breakpoint answers `stopped at ticker.spite:16` at once, and its own answer is the answer to `continue`.
At the console the program prints `stopped at ticker.spite:16` and reads commands until `continue`. When it is not
stopped, `where`, `locals` and `continue` answer `the program is not stopped at a breakpoint: 'break hero.spite:12'
sets one`. Only the program's own files hold a breakpoint, since only their code is swapped, and in a build without
`--hot-reload` and a REPL every breakpoint command, `eval` and `run` answer that they need one.

## Code typed at the prompt

In a program built with `--hot-reload` and a REPL, `eval` compiles an expression into the running program and
answers its value as text, and `run` compiles a statement and runs it. The code is ordinary Spite, and `program`
names the entry instance:

```text
$ spite --connect=4000 --command="eval program.stock.count() + 40"
{"ok":true,"value":"41","type":""}

$ spite --connect=4000 --command="eval Lookup().of(12)"
{"ok":true,"value":"22","type":""}

$ spite --connect=4000 --command="run program.stock.append(9)"
{"ok":true,"value":"","type":""}
```

The code is compiled as the body of a function of a class of its own, swapped in by a reload and called once, so it
reads and changes what the entry instance reaches, makes objects (`Lookup().of(12)` makes a `Lookup` the program
never held), and calls any function, a generic class's included. `eval`'s value becomes text as it would in
`"{value}"`. A value that does not become text, and code that does not compile, answer the compiler's error, and the
program keeps its code. The code stays loaded until the next `eval` or `run`.

An assignment whose right side makes an object is compiled the same way, so an attribute can be given an object
the program never made:

```text
$ spite --connect=4000 --command="follower = Circle(3)"
{"ok":true,"value":"Circle { radius: 3 }","type":"Circle?"}
```

The assignment is compiled as `run`'s code is, the statement `program.follower = Circle(3)`, run once, and the
answer is the target read back. A class that does not fit answers the compiler's error, and the program keeps its
code. In a program built without `--hot-reload` such an assignment is refused, since nothing there compiles code.

## Live reload: `--hot-reload`

`--hot-reload` builds a program that can take new code while it runs. Save a file of the program, and the program
is compiled again; the functions whose machine code changed are copied into the running program and swapped in
the next time it waits or ends a pass of a loop, the same moments the REPL is answered at. Objects, singletons
and everything the REPL changed stay as they were: only functions change. Every function sits in a slot so it can
be swapped, which makes the build slower; measure performance only with a production build
([Measure only a production build](compiler.md#measure-only-a-production-build)).

```
spite --hot-reload --repl-port=4000
```

It works with or without a REPL. Without one, each swap is reported on the program's error output (`spite: rebuilt
Monster`); with `--repl` or `--repl-port`, `reload` swaps in what changed right away and answers what it rebuilt, and
`last_reload` answers what the last swap did, which is how you learn about one the file watcher made.
`wait_reload` waits for the watcher: it answers once every save of the program's files has been compiled and
swapped in (or refused), with what `last_reload` would then say, so a tool that saves a file never polls. A program
built without `--hot-reload` has none of this: no watcher, no swapping, and `reload` answers that it was not
built for it. `--hot-reload` keeps every function (it implies `--development`), since a new version of a class
may call one nothing called before. It is slower than a production build, since it inlines no call and every call
goes through a slot a reload can re-point: a `--hot-reload` build trades run speed for live information, and speed
is measured on production builds. A reload compiles with exactly the flags the program was built with. Live
reload runs on Windows.

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

This session runs against a copy of the program above, built with `--hot-reload --repl-port`:

```text
$ spite --connect=4000 --command="program.visits = 42"
{"ok":true,"value":"42","type":"Integer"}

# hot_counter.spite: greeting() now returns "welcome back, visit {visits}"
$ spite --connect=4000 --command="reload"
{"ok":true,"value":"rebuilt HotCounter","type":""}

$ spite --connect=4000 --command="greeting()"
{"ok":true,"value":"welcome back, visit 42","type":"String"}

# monster.spite: roar() now returns "{name} roars louder", and nothing is sent: the watcher sees the save
$ spite --connect=4000 --command="wait_reload"
{"ok":true,"value":"rebuilt Monster","type":""}

$ spite --connect=4000 --command="monster.roar()"
{"ok":true,"value":"Goblin roars louder","type":"String"}

$ spite --connect=4000 --command="exit"
{"ok":true,"value":"","type":""}
```

`visits` kept the 42 set before the first reload, and the second reload rebuilt `Monster` alone. When the watcher
has already swapped a save in, `reload` finds nothing left to do and answers `unchanged`. For an AI: edit the file and send `reload`; or save and send `wait_reload`, which answers once the
watcher has swapped the save in.

### What a reload can change

| You change | What happens |
|---|---|
| a function's body | the next call runs the new body; a call already running finishes the old one |
| a new function on an existing class | the new code calls it, and so can the prompt |
| a function you deleted | whatever still holds it (a function value, the REPL) keeps its last code; the answer says `removed Greeter.old` |
| an attribute's default value | instances made after the reload get the new default |
| an enum: a value added after the others | taken; the objects that hold the enum keep their values |
| an enum: a value inserted before the others, removed or reordered | refused: `the reload changes the values of Gauge.Mood other than adding new ones last ... start the program again, or add the new values after the others` |
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
spite> reload
error: the reload was refused, and the program keeps all of its code: the reload changes the attributes of Counter (total is new), and the objects the program holds keep the attributes they were made with: start the program again to take the new attributes
```

Start the program again to take the new attributes. An attribute's *default value* is not a layout change: instances
made after the reload get the new default, and the ones already made keep what they hold. An enum is the same: a value
added after the others is taken, and any other change to an enum's values is refused, since every value the program
holds keeps the number it was made with.

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

- **One slot per function.** In a `--hot-reload` build every function starts with a jump through its slot, eight
  bytes of the executable that hold the address of the function's body: the program's own functions, the standard
  library's, and the helpers the compiler writes. Every call, function value and thread entry names the function's
  start, so all of them follow a swap. A normal build is untouched: direct calls, and tree shaking. The cost is one
  indirect jump per call, only in a `--hot-reload` build.
- **What the build records.** Beside the executable, `game.reload_host` lists what the running program holds: every
  function with its start, its slot and a digest of its machine code, the state its code reaches by name, the system
  functions it imports, every class's layout, every enum's values and a digest of each of the program's files.
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
  (allocating, freeing, stopping) are compiled differently, when the new code calls a system function the running
  program never imports, when it reaches state the compiler did not name, when the room kept for new code is full,
  and when a function writes more or fewer object literals of one set of attributes than the running one. Each
  answer ends with what to do, most often `start the program again`.
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
