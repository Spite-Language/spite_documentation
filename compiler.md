# Compiler command line

The compiler compiles one program. A program is a folder, and its entry file is the one named after the folder:
compiling `game` compiles `game/game.spite`, whose class `Game` is constructed to start the program. The compiler
builds the folder it is run from: `cd game`, then `spite`. The compiler
is written in Spite, builds itself, and writes machine code directly (Windows x64): there is no C compiler, linker
or other toolchain behind it, and nothing to install beside the compiler and the standard library's folder, which the program loads by its path. The command
below is written `spite`, for the compiler's executable, `spite.exe`.

The compiler first reads the **whole** program (the standard library, the program's folder, its `build.spite`
and every folder it `load`s) and only then decides what to produce, from the program's
[`Build`](programs.md#compile-time-settings-build). Every option is a field of `Build`, given as `--name=value`
(a `Boolean` option may be given bare, `--check`). A flag is written in kebab-case and sets the `snake_case` field
of the same name: `--executable-path=build/game.exe` sets `Build.executable_path`, and a field a program declares in
its own `build.spite`, `worker_stack_size`, is given as `--worker-stack-size=256`. Nothing is written beside the
program's source unless a path says so ([where the outputs go](#where-the-outputs-go)).

```
spite                               in game: build its executable, and run it
spite --check                       format it and check that it compiles, building nothing
spite --build                       build it without running it
spite --executable-path=out/game.exe    build it there
spite --serve=true                  set a field the program's build.spite declares
spite --final-classes=.spite/final  also write the merged classes as Spite into .spite/final
spite --high-ir                     also write the high representation, .spite/build/<name>.high_ir.txt
spite --low-ir                      the same for the low representation, .spite/build/<name>.low_ir.txt
```

Each instruction in the low representation ends with ` | ` and every field it holds as `name=value` (a list as
`[a,b]`, a vector loop's lists as a count), so a field an optimisation forgets to carry shows in the file. Each
statement in the high representation, and each `function` line, ends the same way: a value it holds is its own fields
in braces, `{name=x ...}`, a list of values `[{...}, {...}]`, a block `{statements=3 statements line=12 ...}`, a type
its name, a function its key, and a missing one `null`.

Every flag is a `Build` field, listed under [Build options](#build-options). Flags mix
freely, and the outputs combine: one compile can build the executable and write its final classes. `--optimized`
marks a release build: `Build.optimized` is true, which the program's own code can read. A production build is
optimised the same way with it or without it, so `--optimized` does not make the program faster. `--optimized=false` stays allowed and switches it off even where
the program's `build.spite` says `true`. `--target-operating-system=linux` (or `windows`, or `mac`) sets
`Build.target_operating_system`, which a program reads to choose its own code; the executable is a Windows x64 one
whatever it says. There is no `--mode`, no `--output` and no `--format`: each is an unknown flag.

How the compiler is built from source, and how it is put together, is in
[spite_compiler](https://github.com/Spite-Language/spite_compiler)'s
[README.md](https://github.com/Spite-Language/spite_compiler/blob/master/README.md) and
[ARCHITECTURE.md](https://github.com/Spite-Language/spite_compiler/blob/master/ARCHITECTURE.md); the programs that
define what it must accept and refuse are in
[spite_spec](https://github.com/Spite-Language/spite_spec). [Getting started](getting_started.md) installs it and
runs a first program.

## Name the program

```bash
cd game
spite
```

`spite` builds the folder it is run from, which is how a program is named (D1775): `cd game` and then `spite`.
Nothing on the command line is given by position: every value is a named setting.

The folder supplies the program's other classes, and the entry class's constructor takes no arguments: the
program reads its command line through `Environment()` and what its build decided through `Build()`. A folder
with no entry file named after it is an error naming the file it looked for
(`error: 'game' holds no program: a program is a folder holding an entry file named after it, 'game/game.spite'`). The
compiler compiles itself the same way, `cd spite_compiler` then `spite`, whose entry is
`spite_compiler/spite_compiler.spite`.

## How a program is loaded

The compiler follows the `load` lines of the program's entry file in order, then of each file it reaches.
[programs.md](programs.md#how-a-program-is-loaded) walks through it.

## Where it runs

`spite` runs the program attached to the terminal: its output appears as it is printed, its standard input
is yours, and the compiler exits with the program's exit code, so a script can run `spite` where it would
run the executable. The program is given no flag at all: it reads its settings from the environment and from the
`.env` file of the folder, like any program run by hand ([programs.md](programs.md#run-time-settings-environment)).

The program the compiler builds runs in the folder you ran it from, its own: a relative path the program opens
(`File("save.txt")`, `Directory("levels")`, a cache folder) is relative to that folder, not to the compiler's
repository. A foreign library's header named by a relative path (`DynamicLibrary`'s third argument) is looked for
in the working directory as well.

## Choose the outputs

The compiler reads the program first, and decides what to produce only after, from `Build`, so a program's own
`build.spite` can choose its outputs like any other option ([programs.md](programs.md#compile-time-settings-build)).
`spite` builds the executable and runs it, with its settings. Two flags say to stop sooner:
`--check` (`Build.check`) only checks that the program compiles, and `--build` (`Build.build`) builds the
executable without running it. `final_classes` writes the program back out as Spite into a folder, beside whatever
else the build does ([below](#inspect-merged-classes)), and `optimization_report` writes what the compiler could not
optimise into a file ([below](#read-what-was-not-optimised)); the defaults are in [Build options](#build-options).

`--check` still reads and compiles the whole program and reports [its errors](#how-many-errors-are-listed), and writes nothing but the
formatting, which is not an output: every compile does it first ([below](#formatting)). It writes no executable
and never touches one an earlier build left, so checking while the program runs is safe:

```bash
cd game
spite            # build .spite/build/game.exe and run it
spite --build    # build it, run nothing
spite --check    # only check that it compiles
```

A build never leaves an old executable behind. `spite` and `spite --build` first remove the executable
an earlier build left at their path, so when the build fails, nothing is left there that still runs the old code:
the executable at that path is always the one the last build made, or none. That holds when the program cannot be
read too: a program with a text that never closes removes the old executable, and builds nothing.

Whole-program steps (tree shaking, the constants `Build` folds, which templates are made) run on the complete
program before any output is written, so every output describes the program that was built.

## Where the outputs go

**Nothing is written beside the program's source unless a path says so**. Each output has its own path
option, and without one it goes into `.spite/`, a folder inside the program's own folder. `.spite/` holds only what
the compiler writes, and can be deleted whenever you like; no `.spite/` folder is ever read as part of a program.

| Output | Path option | Default |
|---|---|---|
| the executable | `executable_path` | `<program>/.spite/build/<name>.exe` |
| the high representation | none: `high_ir` is on or off | not written; `--high-ir` writes `<program>/.spite/build/<name>.high_ir.txt` |
| the low representation | none: `low_ir` is on or off | not written; `--low-ir` writes `<program>/.spite/build/<name>.low_ir.txt` |

The executable's exit code is what the entry constructor answers. A path moves the executable:

```bash
spite --build --executable-path=build/game.exe
```

`.spite/git/` holds the checkouts of the repositories a program's `load`s pin to a commit
([packages.md](packages.md#loading-a-repository-pinned-to-a-commit)).

A folder the path names is created for a final-classes folder and a report, but not for the executable: the folder of
`--executable-path` has to exist. `--executable-path` given with `--check`, which builds no executable, is an
error (`'--executable-path' says where the executable goes, and '--check' builds none: leave out --check to build it`),
so a path never passes silently. Every `Build`
field is a constant in the built program, the paths included.

Beside every executable the compiler writes `<name>.functions`: one line per function of the code, with its start
and end as offsets from the start of the code section and its key. Nothing of it is in the executable. When a native
fault halts the program, the crash report reads that file to name the function that faulted and the functions that
called it ([failure.md](failure.md)); without the file it prints only the fault line. Beside it goes
`<name>.crashes` too, one line per `crash` and `fallback` site, which a crash report's identity is looked up in
([failure.md](failure.md#what-a-crash-reports)).

## Build options

Every option is a `Build` field with a literal default, declared by the standard library's `Build`:

| Option | Default | What it does |
|---|---|---|
| `check` | `false` | only checks that the program compiles: no executable, nothing run ([above](#choose-the-outputs)) |
| `build` | `false` | builds the executable without running it |
| `executable_path` | `""` | where the executable goes; `""` is `<program>/.spite/build/<name>.exe` ([above](#where-the-outputs-go)) |
| `final_classes` | `""` | writes the merged classes to this folder |
| `optimization_report` | `""` | writes what could not be optimised, and why, to this file ([below](#read-what-was-not-optimised)) |
| `high_ir` | `false` | writes the program's high representation to `<program>/.spite/build/<name>.high_ir.txt` |
| `low_ir` | `false` | writes the program's low representation to `<program>/.spite/build/<name>.low_ir.txt` |
| `development` | `false` | an [inspectable build](#development-builds-and-tree-shaking): nothing tree-shaken, internals as ordinary objects |
| `repl` | `false` | runs the program, then opens a REPL on it at the terminal ([repl.md](repl.md)) |
| `repl_port` | `0` (off) | serves the remote REPL on `127.0.0.1` at this port while the program runs ([repl.md](repl.md)) |
| `hot_reload` | `false` | swaps changed classes into the running program; implies `development` ([repl.md](repl.md#live-reload---hot-reload)) |
| `debug_memory` | `false` | counts allocations and frees and prints them when the program ends ([below](#counting-memory---debug-memory)) |
| `trace_fallbacks` | `false` | writes each `fallback` taken of the program to the error stream as it fails, not only in a crash's report ([failure.md](failure.md#what-a-crash-reports)) |
| `program` | the folder named | cannot be given: it is the folder on the command line |

```bash
spite --debug-memory
spite --build --executable-path=build/game.exe --debug-memory
```

A program may give an option a different default by reopening `Build` in its own `build.spite`
(`var debug_memory = true` to always count its memory), and add options of its own the same
way. No option is read before the program is, so every one of them, the outputs included, can be given a default
there, written as a literal (`true`, a number, or a text without `{}` in it), since the compiler reads it before
anything runs. A program that loads no standard library has no `Build`, and takes only `--debug-memory` and
`--optimized`. Every `Build` field is
a constant in the built program, so with `var build = Build()` beside the attributes, `if build.debug_memory { }`
keeps one branch.

None of this is in the program that is built: choosing, building and writing outputs is the compiler's own
work, and `Build` is a folded singleton that holds nothing at run time. Only four options add code to the program,
and only to a build that asks for them: `--debug-memory` its allocation table, `--repl` and `--repl-port` the REPL
and the reflection it walks, `--hot-reload` a slot per function and a file watcher, and `--trace-fallbacks` a write
in each `fallback` taken. Each one's cost is on its
page.

### Measure only a production build

REPL and live-reload builds are slower on purpose. They exist to tell you more while the program runs: every
function sits in a slot so it can be swapped, the program can stop at a breakpoint, and the REPL can inspect any
object. None of that is free, and none of it is meant to be fast.

So any benchmark or performance comparison of Spite uses a build with no `--repl`, no `--repl-port`, no
`--hot-reload` and no `--debug-memory` (which counts every allocation). A number measured on any other build
describes the tooling, not the program.

## Pass settings to the program

The compiler is an executable like any other: each of its flags is a setting its own `environment.spite`
declares, read and checked the way a program's are ([programs.md](programs.md#run-time-settings-environment)).
Running it is the compile of the program it builds, so the same command line also sets the fields that
program's `build.spite` declares, kebab-case like the compiler's own:

```bash
spite --debug-memory --player-count=2
```

`--debug-memory` is the compiler's and `--player-count=2` sets `Build.player_count`, written into the build as a
constant. A field of the program's own `build.spite` that no flag sets is read from its variable,
`PLAYER_COUNT=2`, then from a `PLAYER_COUNT=2` line of a `.env` file in the folder built, the variable winning, and
else keeps its default ([programs.md](programs.md#compile-time-settings-build)). A value read there that does not
fit the field's type is an error naming where it was read: `'VOLUME=many' in .env sets 'Build.volume', which is an
Integer: a whole number, like VOLUME=3`. The compiler's own settings are read from the environment too:
`OPTIMIZED=true` is `--optimized`. Any other flag is an error,
a setting of the program's `Environment` too: that one is given to the executable,
`.spite/build/game.exe --volume=7`, or read from the environment by the program `spite` runs, which is given no
flags. Nothing is given by position: a word that is not a flag (`spite ada`) is an error naming the setting form.
There is no `--` separator: a bare `--` is an error showing the form above.

Compiler flags and program settings share one set of names, so an `Environment` field named like a `Build` field
is a compile error naming both. A flag naming no field of either is an error listing the fields `Build` has, a
setting given the snake_case spelling is an error naming the kebab one, and a value that is not of the setting's
type is an error naming the forms it takes, so typos never pass silently.

## Formatting

**The compiler is the formatter, and every compile formats first**. Once the whole program has been read,
each of the program's own files whose formatted text differs from what is on disk is rewritten, printing
`formatted <path>`, and the program is read again, so what is compiled, and every line an error names, is the
formatted file. Nothing turns this off, and a file the formatter refuses stops the compile: a program is never
compiled from text that is not in the one style. A refused file is
`<path>: error: the formatter refuses it, and a program compiles only once formatted: <reason>`, and the compiler
exits with status 1. A file that does not lex or parse is reported by reading the program, before the formatter
sees it. The files formatted are the program's folder with every folder below it, never the standard library. Reading a program, or a folder it loads, skips every folder named `.spite`, `.spite-cache` or `.git`, so what a build or a version control tool keeps there is never read as part of the program.

**A file edited during the compile is never overwritten.** Just before writing a formatted file, the compiler reads it
again; if it no longer holds the text the compile read (another editor or agent changed it meanwhile), it is left
as it is now, `<path> changed while it was being compiled` is printed, and the program is read again from disk. A
file whose formatting does not change is never written, so its modification time never moves.

There is no command that only formats. A linter or a language server runs `spite --check` in the program's folder, which formats the
program and checks it without building, and there is no mode that checks the formatting without writing it: the
fix is always the compiler's own. What the formatter rewrites, and what it refuses to, is [style.md](style.md).

## Development builds and tree shaking

A normal build keeps only what the program uses: a condition on a codegen value or a `Build` field keeps one
branch, a template exists only for the names called, and reflection only where it is read
([metaprogramming.md](metaprogramming.md#tree-shaking)).

An **inspectable build** (`--development`, `--hot-reload`, `--repl` or `--repl-port`) keeps every generated
function instead, every function of your classes compiled even when nothing calls it (one that takes a member a
walk hands it, a `Spite.AttributeDeclaration`, `Spite.Attribute`, `Spite.FunctionDeclaration` or `Spite.Argument`, is
compiled for each member walked), so live reload has all of them to swap and the REPL can reach every internal, and the
singletons that hold nothing, such as `Build`, are ordinary objects. A `false` form (`--development=false`) does not
make a build inspectable, and `--optimized` together with a REPL flag is still an inspectable build. Every other
build is a production build, where unused things are tree-shaken, down to the classes nothing reachable uses. Conditions on
codegen values and `Build` fields fold in both: they are facts of the build, not values the running program could
change. Which optimisation applies in which build is in [optimizations.md](optimizations.md).

### Every function is checked

Every build, `--check` and `--build` alike, compiles every function of your classes outside a generic class, called
or not, so a mistake in one nobody calls is reported like any other:

```gdscript
func never_called(): Integer {
    return missing_function(2)
}
```

```
game.spite:9: error: this class has no function 'missing_function' (in Game.never_called)
```

A production build compiles them after the program's own functions and then sets aside everything they made, so
the executable is exactly the one the reached functions make: a class only they make, a copy function, a shape they
fill or the order objects are numbered in never reaches it. The rules a function is built against refuse in every
function; the proofs that need the whole program (that a value is present, that an attribute is read) prove the
reached functions. An inspectable build keeps the unreached functions in the executable instead.

## Counting memory: `--debug-memory`

`--debug-memory` builds the program with a table of live allocations and prints `allocations: N frees: N` when it ends.
The two numbers differ only when something leaked, and then the report names the classes whose instances are
still alive, which is how a leaked cycle shows up ([memory.md](memory.md#cycles-are-freed)). The table costs a
lookup on every allocation and free, and exists only in a `--debug-memory` build: any other build
keeps no table.

The same build fills every byte of fresh memory the program has not written with `0xA5`, the room of a
`Buffer(size)` and a buffer's grown tail included, so code that reads bytes nobody wrote reads the same wrong
value on every run instead of passing by luck: a `Long` read from such room is `-6510615555426900571`. Any
other build hands out memory as before.

## Inspect merged classes

`--final-classes=folder` writes the discovered, merged classes as readable `.spite` files, one per class, under
the namespace folders it belongs to. It is useful after a `load` or a reopening: the file holds the declarations
that won, so reopening stops being invisible. What it writes is a program, not a report: running the printed
entry file runs the same program. `Build` is printed with the defaults it
was declared with (the program's own `build.spite` included), not the values this build folded, so running the
printed program takes its flags again. A repository loaded at two commits is one library, so each of its classes
is printed once, as the commits merged it ([packages.md](packages.md#two-versions-of-one-repository)).

Each attribute and function is marked with one comment line linking the `.spite` file that supplied it: the
program's own file, a file it loads, or the reopening that replaced it, as a path relative to the printed
file. The folder holds a `.final-classes` file, which is what lets a comment there link a `.spite` file
([style.md](style.md#comments-are-links)), so the printed program compiles with its links checked like any other.
A folder holding that file is never part of the program around it: printed into the program's own folder
(`--final-classes=final`), the classes are not read as the program's source the next time it compiles.

What is written is what the program **ends up with**, not what was written down: the classes come from the
generator after it has run, so a class the generator never made is not there, and a generic template the program
instantiates is printed from its merged file, so the printed program can instantiate it again. Tree shaking of a production build runs after that, so a library class
the program never uses (`FileSystemWatcher`, `Tcp.Server`) may still be printed although it was dropped from the executable.
The classes the compiler writes for itself (the copy table, the REPL's prompt, the fault report) are not printed:
compiling the printed program writes them again.

Every class the program names comes from a file of the program or of the standard library, including `DynamicLibrary`,
`String` and the numbers (`Integer`, `Long`, `Double`, ...), so every one of them is printed like a
class of your own. A function whose body the compiler supplies (such as a `Buffer`'s reads and writes) is added to its
class as the compiler's own reopening, and is printed as a declaration without a body:

```gdscript
func _read_integer_at(at: Integer): Integer
func _write_integer_at(at: Integer, value: Integer)
```

A declaration without a body is what the compiler reads back, so the printed program still compiles: the
printed file reopens the class with the same members, and the compiler supplies the same bodies
again. Anywhere else, a `func` with no body is an error, because only the compiler can supply one.

`instantiated/` holds one file per generic instantiation besides, named for the values it was given
(`weapon_integer_true.spite`, class `WeaponIntegerTrue`), written as a `type` with the members that instantiation has,
for reading what each instantiation ended up with.

The folder has no default: anything inside the program's own folder would be read back as part of the program,
so the folder is always named. It combines with every build; with `--check` it is the only thing written:

```bash
spite --final-classes=.spite/final --check
```

## Read what was not optimised

The compiler optimises on its own and says nothing about what worked ([optimizations.md](optimizations.md)). What it
could **not** do is what costs you, so `--optimization-report=file` writes exactly that: one Markdown file listing
every place an optimisation fell back, each line a link to the source line and the reason, so every line is
something you can act on. Nothing is written without the flag, and the program built is the same either way.

```bash
spite --optimization-report=.spite/optimization.md --check
```

The report has one section per optimisation, with how many places it lists:

- **Lists that hold references**: each `List<T>` whose items are kept as references, at the line that first made
  it, and why `T` cannot be laid inline (an attribute that is text, a list or another object, a `drop()`, a function
  using `this` as a value), or that it would fit inline but the list is given objects that were not made for it or
  its items are used for more than reading.
- **Lists not in the frame**: a local list that could have lived in the function's frame
  ([optimizations.md](optimizations.md#a-local-list-of-known-size-lives-in-the-frame)) and did not: appended to in
  a loop, more items than the frame holds, or the line that does more than read it.
- **Objects not in the frame**: a local made by a constructor that stays on the heap
  ([optimizations.md](optimizations.md#objects-that-never-leave-their-function-live-in-registers-or-the-frame)), with the line
  that lets it go (`kept.append(badge)`), or what about its class keeps every object on the heap.
- **Copies not elided**: a `var twin = original.copy()` that allocates, with the line that lets the copy go, or
  why its class cannot be copied into the frame.
- **Overflow checks kept**: each `+`, `-`, `*` or `-` in front of a whole number that keeps its check
  ([optimizations.md](optimizations.md#arithmetic-a-range-proves-is-not-checked)), with the operand whose range
  nothing proves or the ranges that let the answer pass its type
  (`` `product * 3` keeps its overflow check: 'product' runs from -2147483648 to 2147483647, so the answer may not
  fit in an Integer ``). Hold a total in a wider type, or bound a value where it is made, and the line goes.
- **Waits that hold the frame**: each call a frame loop would have started
  ([concurrency.md](concurrency.md#a-wait-in-a-frame-does-not-hold-the-frame)) that waits in place instead,
  because the call or the frame writes something back from a value it read before a wait
  (`` `tally.add_late()` waits in place: Tally.add_late writes `count` from a value it read before a wait, and the
  frame could write it while the call waits ``). Read the value again after the wait, and the line goes.

A line looks like this, its link relative to the report's folder so it opens from wherever the file is read:

```markdown
- [game/game.spite:23](game/game.spite#L23): `badge` (`Badge`) is made on the heap: line 24 lets it go: `badges.append(badge)`
```

Like `--final-classes`, the report describes what the program ends up with: a function that tree shaking removes
reports nothing, and the library's lines are listed with the program's, sorted by file and line, since they cost
the program the same. An inspectable build (`--development`, `--repl`, `--repl-port`, `--hot-reload`) places
nothing in the frame by design; its report says so and lists only the lists that hold references and the overflow
checks kept.

## Errors and usage

A word given by position (`spite game`, `spite format`), a folder with no entry file, an entry constructor that takes
arguments, an entry class with a `generic` line, `--check` given with `--build` or `--executable-path`, a flag
the compiler no longer has (`--run`, `--executable`, `--tune-for-this-machine`, `--translation-units`), a flag given
twice, a flag with an underscore, a `--name=value` that names no field of `Build` or `Environment`, and a bare `--`
are errors, each naming the fix. The underscore one reads `'--player_name' is written '--player-name': flags and
settings are kebab-case on the command line, and the field they set is 'player_name'`, and it is checked before
anything is read. `--target-operating-system` is read from the flag alone: a `var target_operating_system` in
`build.spite` is an error naming the flag, and it takes only `windows`, `linux` or `mac`. A `var operating_system` or
`var program` in `build.spite` is an error too: the compiler supplies both.

A few more, each with the words it prints:

- `--operating-system` and `--program` cannot be given: the compiler supplies both. `'--program' cannot be given:
  the program is the folder the compiler runs in, 'cd folder' then 'spite', and Build.program is set from it`.
- A value of the wrong type: `the setting Environment.optimized was given 'maybe', which is not a Boolean, 'true'
  or 'false': write --optimized=true on the command line, or OPTIMIZED=true in the environment or a .env file` for
  one of the compiler's own settings, and `'--volume=many' was given to the compiler, but 'Build.volume' is an
  Integer: a whole number, like --volume=3` for a field of the program's `Build`. A text setting given bare
  (`--optimization-report` with no `=file`) is refused the same way.
- `--repl-port` takes a port from 1 to 65535: `--repl-port takes a port number from 1 to 65535: spite
  --repl-port=4000`.
- An unknown flag lists every field the program's `Build` declares and shows both places a setting can belong:
  `'--nothing' is not a compiler option, and this program's Build declares no 'nothing' (it declares check, build,
  ...). To decide it when compiling, reopen Build in the program's build.spite with 'var nothing = ...'; to read it
  when the program runs, declare it in environment.spite with 'var nothing = ...' and give it to the program's
  executable`.

An error is `<path>:<line>: error: <message> (in <Class>.<function>)`, with the function's name even when the error
is in its signature, and `(in <Class>.)` for one among a class's attributes and types.

## How many errors are listed

A compile lists its errors together, so one compile tells you what to fix. A failure stops its own check, and the
checks go on to the rest:

- A file that does not lex or parse stops that file; the other files are still read and checked. If it is the entry
  file, the compile stops there.
- Each rule on a read file reports what it finds and goes on, so two unread locals are two errors. A line is told once:
  a second mistake on it waits for the next compile.
- A function that does not type-check stops that function, and every other reached function is still built. Functions
  reached only through it are not, so errors there wait for the next compile.
- A stage whose input is broken does not run: errors in reading leave out the types, and a type error leaves out the
  proofs that need the whole build (unread names, values that may be null, the rest of the build's facts), which come
  once it builds. An error from an earlier stage never echoes as a false one in a later stage.

The errors in the program's own files come first, in the order found, at most 25, followed by `and N more in the
program` when there were more. Then come the errors in loaded packages, at most 25 together, followed by one
`and N more in <package>` per package that had more. Finding a type error takes one more pass over the program, so
past 25 of those the compile stops looking and the list ends `and possibly more in the program`.

Fixes the compiler made on the way are printed before the errors, and the exit code is 1.

---

Next: [REPL and live reload](repl.md), running a program with a REPL and swapping changed classes into it.
