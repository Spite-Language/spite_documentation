# Programs: the entry and settings

A program is a folder. Its **entry file** is the file named after the folder (`game/game.spite`, class `Game`),
and the program runs by constructing that class. There is no `main` and no entry point declaration: the
entry class's constructor *is* the program, and when it returns the program ends with exit code `0`.

```bash
cd game
spite
```

The folder is the only way to name a program: the compiler builds the folder it is run from, and a folder with no
file named after it is an error naming the file it looked for (`'game' holds no program: a program is a folder
holding an entry file named after it, 'game/game.spite'`). By
default the build writes the executable into `.spite/` in that folder, `.spite/build/game.exe`, and runs it; nothing is written beside the program's source
([compiler.md](compiler.md#where-the-outputs-go)).

Every other `.spite` file in the folder is another class of the program, and every folder inside it is a
namespace ([packages.md](packages.md)): `game/engine/renderer.spite` is `Engine.Renderer`, and
`game/engine/physics/physics.spite` is `Engine.Physics.Physics`.

## The entry constructor takes no arguments

The command line is not handed to the entry class. It is read where it is needed, through two singletons that
work anywhere:

| Read it with | What it is |
|---|---|
| `Environment()` | the program's run-time settings, which it declares ([below](#run-time-settings-environment)) |
| `Build()` | what was decided when the program was compiled ([below](#compile-time-settings-build)) |

So an entry constructor with parameters is an error that names both:

```gdscript title=entry_arguments_error/entry_arguments_error.spite entry error
var console = Console()

func EntryArgumentsError(name: String) {
    console.print(name)
}
```
```diagnostic
is the entry constructor, and it takes no arguments
```

The entry class is the one that runs, so it cannot be generic: a `generic` line in it is an error, and what needs
codegen values goes in a class the entry class makes.

A constructor is setup, not logic, and the entry constructor is no exception: `fallback` is not allowed in any
constructor ([failure.md](failure.md#fallback-is-not-allowed-in-a-constructor)). A program whose constructor grows
complicated splits it into functions it calls, and then reads as a table of contents.

Nothing reaches the program by position, and a named value is never read by its text (D1775): `--level=3` and a bare `--verbose` reach the program only as the
`Environment` attributes they set, `var level = 0` and `var verbose = false`, typed, with a default and a name the
compiler checks ([below](#run-time-settings-environment)). `Program` has no `arguments()`, no `argument_named` and no
`has_flag` to ask the command line anything else, `Environment` has no `unread_settings()`, and a bare `--` is not a
separator: the built program refuses it, since every value on its command line is a named setting.

## Run-time settings: `Environment`

A program's settings are the fields of `Environment`, a singleton in the standard library. The program reopens
it with a file of its own named `environment.spite`, holding one `var` per setting with a literal default, and
reads it with `Environment()` anywhere:

```gdscript title=program_settings/environment.spite
var endpoint = "local"
var worker_count = 1
var verbose = false
```
```gdscript title=program_settings/program_settings.spite entry vars=endpoint:production,worker-count:4,verbose:true
var console = Console()
var environment = Environment()

func ProgramSettings() {
    if environment.endpoint == "production" {
        console.print("using the production endpoint")
    } else {
        console.print("using the local endpoint")
    }
    console.print("workers", environment.worker_count, "verbose", environment.verbose)
}
```
```output
using the production endpoint
workers 4 verbose true
```

That run is `.spite/build/program_settings.exe --endpoint=production --worker-count=4 --verbose`, after `spite
--build` in `program_settings`: a setting is written like a compiler flag, kebab-case on the command line and
snake_case in code, and a `Boolean` may be given bare. `spite` itself refuses it, since it is no flag of the
compiler and no field of the program's `Build`, and runs the program with no flags (D1812, D1813). Each field is read once, when `Environment()` is first made (and again in a `--hot-reload` build when a save gives a
setting a new default, [repl.md](repl.md#what-a-reload-can-change)), from the first of:

1. the program's own command line, `--endpoint=production`, given to the built program;
2. the process environment variable named by the field in upper case, `ENDPOINT`;
3. a `.env` file the program loaded with `environment.load_from_file(".env")`;
4. the declared default.

Only declared fields are read: nothing else is loaded, so a process variable that names no setting changes nothing,
and the command line wins over the process environment, which wins over a `.env` file, as it does in dotenv. Every
value on the command line must be a setting: a flag no field declares, or a word given by position, stops the
program with exit code 1 and an error saying how to declare it. A flag written in snake_case (`--worker_count=4`
for `var worker_count = 1`) is refused the same way, with the kebab-case spelling to write. The
default's literal is the setting's type: `false` is a `Boolean`, a whole number an `Integer`, `""` a `String`.
A setting may also be a value of an enum the `Environment` declares, its type written, `var mood: Mood =
'calm'`, and then only that enum's value names are accepted, the error listing them. Anything else is an error
naming the forms. A value that is not of the setting's type
(`--verbose=maybe`, or `VERBOSE=maybe` in the environment or a `.env` file) stops the program with exit code 1
and an error naming the setting, the value and the form it takes, since the program was started wrong and there
is no line of it to point to.

A `.env` file is read the way dotenv reads one, out of the box: `environment.load_from_file(".env")` takes its
`NAME=value` lines (spaces around the name and the value are trimmed, blank lines and `#` comments skipped), reads
every setting again, and answers `false` if the file cannot be read, changing nothing. A setting the command line or
the process environment already gives keeps that value, and a name in the file that no setting declares is reported
on the error stream (`warning: 'UNKNOWN_THING' in .env names no setting of Environment, so it was not read`) and
skipped.

```gdscript
var console = Console()
var environment = Environment()

func Server() {
    var loaded = environment.load_from_file(".env")
    if not loaded {
        console.print("no .env here, so the command line, the environment and the defaults decide")
    }
    console.print(environment.verbose)
}
```

The values are read when the program runs, so `if environment.verbose` is an ordinary `if`, with both branches
compiled in.

That reading is the whole run-time cost: one pass over the command line and one environment lookup per declared
field, once, when `Environment()` is first made. A program given any argument makes `Environment()` before its
entry starts, so one that never makes it still refuses a flag it does not declare, and a program given none pays
nothing.

A setting may be named like one of the compiler's own options: `var check = "none"` in `environment.spite` is read from
`--check=yes` on the built program's command line, and the compiler's `--check` is a different flag, given to `spite`.
A setting named like a field the program's `build.spite` declares is the error below, since a program's `Build` and
`Environment` names share one set.

```gdscript title=setting_type_error/environment.spite
var workers: Integer = 1
```
```gdscript title=setting_type_error/setting_type_error.spite entry error
var console = Console()
var environment = Environment()

func SettingTypeError() {
    console.print(environment.workers)
}
```
```diagnostic
so it is declared with only a literal default, and the literal is its type
```

## Compile-time settings: `Build`

`Environment` is read when the program **runs**. `Build` is its twin for when the program is **compiled**: a
singleton in the standard library whose fields are decided by the compiler and written
into the program as constants. Every compiler option is one of them ([compiler.md](compiler.md#build-options)),
and a program adds its own by reopening `Build` in a file named `build.spite`:

```gdscript title=build_settings/build.spite
var serve = false
```
```gdscript title=build_settings/environment.spite
var name = "client"
```
```gdscript title=build_settings/build_settings.spite entry build=serve:true vars=name:tester
var console = Console()
var build = Build()
var environment = Environment()

func BuildSettings() {
    if build.serve {
        console.print("serving as", environment.name)
    } else {
        console.print("this branch is not in the build")
    }
}
```
```output
serving as tester
```

That is `spite --serve=true` in `build_settings`, with `NAME=tester` in the environment. A `--name=value` naming a
`Build` field sets that field while compiling; a field no flag sets is read from its variable (`SERVE=true`), then
from a `.env` file in the folder built (`SERVE=true` as a line of it), and else keeps the default it was declared
with. Either way `build.serve` is the literal `true` in the built program: `if build.serve { }` is decided while
compiling, and the branch it does not take is never generated. `name` belongs to `Environment`, so `spite` refuses
`--name=tester`: the program it runs is given no flags and reads `name` from the environment, and the built
executable is given `--name=tester` itself.

- A `Boolean` field may be given bare: `--optimized` is `--optimized=true`. Any other bare flag is an error naming
  the value it needs.
- The value has to be of the field's type (`--workers=many` for an `Integer` is a compile error, for a `Build`
  field and an `Environment` one alike), a flag naming no field of either is a compile error listing the fields
  `Build` has, and an `Environment` field named like a `Build` field is a compile error naming both, since the two
  share one set of names on the command line. A typo never passes silently.

```gdscript title=unknown_flag_error/unknown_flag_error.spite entry error build=verbos:true
var console = Console()

func UnknownFlagError() {
    console.print("never built")
}
```
```diagnostic
'--verbos' is not a compiler option, and this program's Build declares no 'verbos'
```

- A field's default may be a literal, arithmetic on literals (`var seconds: Integer = 2 * 60`, `var offset = -3`), or
  another field of the same type (`var attempts: Integer = retries`, which takes that field's value, a flag's included): all of these
  are folded into constants. A default that reads anything else is worked out once, when `Build` is first made.
- A program's own `build.spite` may give a compiler option a different default: `var optimized = true` builds it
  optimized unless `--optimized=false` is given. The compiler reads its options before anything runs, so an option
  is declared with only a literal (`true` or `false`, a whole number, or a text without holes), and anything else is
  an error at the declaration that says so. The compiler reads the whole program before it decides
  anything, so what it produces is a default like the rest: `var build = true` only builds it, and
  `var final_classes = "..."` writes its final classes on every build ([compiler.md](compiler.md#choose-the-outputs)).
  Formatting is not an option: every compile formats first, and `format` is not a field a program may declare.
  `target_operating_system` is the one field only a flag sets: declaring it in `build.spite` is a compile error naming
  the flag, even when it names the system being compiled for. `operating_system` and `program` are supplied by the
  compiler, and declaring either in `build.spite` is a compile error too (`'Build.operating_system' cannot be
  declared by a program: it is the system doing the compiling, which the compiler supplies`). The fields that say what one compile writes (`check`,
  `build`, `executable_path`, `final_classes`, `optimization_report`, `high_ir`, `low_ir`) keep their declared value
  inside the program: a flag sets what the compile does, not what `build.low_ir` reads, which is `false` unless
  `build.spite` says otherwise. `check` and `build` both on, from the flags or from `build.spite`, is an error.
- A loaded package may reopen `Build` in a `build.spite` of its own and declare fields, which the program and the
  package read alike and a flag sets like the program's own ([packages.md](packages.md#a-package-declares-its-own-build-fields)).
  `build.spite` files are reopenings like any other and follow load order: where the program's `build.spite` and a
  package loaded after it declare one field, the package's declaration stays, and a flag sets it either way. So a
  package names its fields after itself, never `production` or `debug`:

```gdscript title=build_over_package/engine/build.spite
var engine_plugins_folder = "plugins"
var engine_name = "forge"
```
```gdscript title=build_over_package/build_over_package.spite entry build=engine_name:ember
var build = Build()
var console = Console()

func BuildOverPackage() {
    load "engine"
    console.print("plugins from", build.engine_plugins_folder)
    console.print("engine", build.engine_name)
}
```
```output
plugins from plugins
engine ember
```

`Build` costs nothing when the program runs: every field is a constant written into the program, and nothing is
read or decided at run time. The one exception is a `--hot-reload` build, where a field the program or a package
declares is read from the `Build` singleton while the program runs and both branches of a condition on it are
compiled, so a reload that changes `build.spite` takes effect; the standard library's reads of its own fields stay
constants. `--final-classes` prints `Build` with its declared defaults, not the values the compile folded, so that
running the printed program takes its flags again.

### The operating system: `operating_system` and `target_operating_system`

`build.operating_system` is the system doing the compiling, and `build.target_operating_system` the one the
program is compiled for: `"windows"`, `"linux"` or `"mac"`. The target defaults to the compiling system, and
`--target-operating-system=linux` sets it to Linux; the executable itself is a Windows x64 one whatever it says. Both are constants, so a condition on
them keeps one branch:

```gdscript title=target_system/target_system.spite entry
var console = Console()
var build = Build()

func TargetSystem() {
    var target = build.target_operating_system
    var known = target == "windows" or target == "linux" or target == "mac"
    console.print("known system", known)
    console.print("compiled for this machine", target == build.operating_system)
}
```
```output
known system true
compiled for this machine true
```

Neither `operating_system` nor `program` can be given as a flag or declared in `build.spite`: the compiler supplies both. Which operating
system a program targets is `Build.target_operating_system`, and the compiler reads that system's folder of the
standard library, which is the next section.

## How a program is loaded

Nothing about starting a program is hidden in the compiler. The program is the entry point: the compiler constructs
its entry class, and there is no launcher around it (D628). What it reads first is the standard library, which every
program has without a line asking for it, then the program's own folder and the folders its `load` lines name.

### The standard library needs no load

Every program has the standard library for the system it is compiled for, with no `load` line (D1872). The library
holds `common`, what every system shares, and one folder for each operating system (`windows`, `linux`, `mac`) whose
files reopen the classes that system does differently
([foreign_libraries.md](foreign_libraries.md#each-operating-system-reopens-what-it-changes)). The compiler reads the
folder of the system the program is compiled for, `windows` or the one `--target-operating-system` names, and that
folder loads `common` itself. Where the library comes from, the one inside the compiler or the folder
`--standard-library-path=<folder>` names, is in [compiler.md](compiler.md#where-the-standard-library-comes-from).

```gdscript
func Game() {
    ...
}
```

A `load` of the standard library the program is compiled with, its system's folder or `common` (`load
"../spite_standard_library/windows"`, from before D1872), is removed from the program's file with a `fixed:` line,
so an old program converts itself the first time it is compiled (a constructor that held nothing else goes with
it, since a class without one is made from its defaults):

```
game/game.spite:4: fixed: 'load "../spite_standard_library/windows"' loaded the standard library, which every program has without a load; removed it (in Game.Game)
```

A `load` of another standard library, a folder holding `common` and the system's folder somewhere else, is an error
that names the flag: a program is compiled with one library, and the command line says which.

```
game/game.spite:4: error: 'load "../other/spite_standard_library/windows"' loads the standard library at 'D:/other/spite_standard_library', and a program does not load the standard library: every program has one. To compile with that one, remove this load and give --standard-library-path=D:/other/spite_standard_library (in Game.Game)
```

`load` stays for the program's own folders and its packages ([packages.md](packages.md#load-is-a-bundle-boundary)),
and it may sit under an `if` on a `Build` field. The program's own folder needs no `load`: it is the root the entry
file sits in. The entry class is constructed when the executable starts, and its exit code is `0`
once the constructor returns, or the code a `Program().exit(code)` gives.

```gdscript title=loaded_program/loaded_program.spite entry
var console = Console()
var build = Build()

func LoadedProgram() {
    var named_folder = build.program.ends_with("loaded_program")
    console.print("loaded from", named_folder)
    console.print("for this machine", build.target_operating_system == build.operating_system)
}
```
```output
loaded from true
for this machine true
```

What the executable does around the entry constructor (handing over the command line, and the teardown after it
returns) is small. The executable leaves the command line to `Environment()`, constructs the entry class, releases the
singletons and class objects, and, built with `--debug-memory`, prints the allocation balance.

## Preloaded files

`preload` carries files in the executable. It is written like `load`: literal text on a line of its own in a
function, relative to the file that writes it, compiled to nothing at run time, and it may sit under an `if` on a
`Build` field. It takes a folder or one file, on this machine or in a git repository pinned to a commit (the same
`.spite/git` copy a `load` of that text uses, so the program needs no git when it runs). The program reads what it
preloaded through `Volume`, a singleton named by the volume, which is the last folder of what was preloaded, exactly
as it is spelled on disk:

```gdscript
var console = Console()
var assets = Volume("assets")

func Game() {
    preload "assets"
    var level = assets.file("levels/first.txt")
    var text = level.read_as_string()
    crash text
    console.print(text)
}
```

| Written | Its volume | What the volume holds |
|---|---|---|
| `preload "bunda"` | `Volume("bunda")` | the whole tree of `bunda` |
| `preload "bunda/cu.spite"` | `Volume("bunda")` | only `cu.spite` |
| `preload "../dependency/bin/tool.exe"` | `Volume("bin")` | only `tool.exe` |
| `preload "github.com/x/art@a3f2c91/textures"` | `Volume("textures")` | the tree of `textures` at that commit |

`volume.file(path)` and `volume.directory(path)` answer the same `File` and `Directory` as the disk, reading another
disk. A file on a volume says so in its path, `assets://levels/first.txt`, the volume's name then the path on it, and
`File(level.path)` is the same file; a plain path always means the system's disk. `Path` knows the form: such a path
is rooted, its `parent` stops at `assets://`, and `is_on_a_volume` answers it. `entries()` walks a volume's folders
as it walks the disk's, sorted, `read_as_buffer` reads a binary file, and a path naming nothing on the volume answers
as a missing file does. A volume is read only: the writers (`write_string`, `remove`, `move_to`, `create`, ...)
answer their failure, `last_written` and `lock` answer `null`, and what needs a real file (`Process`, its
`working_directory`, `FileSystemWatcher`) answers its failure on such a path: write the file to the disk first to run
it.

The files are read-only data in the executable, as text literals are, stored by their path on the volume and sorted,
so the same source gives the same executable on any machine; nothing is read or allocated when the program starts. A
preloaded folder leaves out what starts with `.` (`.git`, `.spite`), as the compiler does with folders it reads. A
program with no `preload` compiles the volume checks of `File`, `Directory` and `Path` away. The compiler refuses:

- a `preload` of a path that does not exist, or a bad git path, in the words of `load`;
- two different folders that make one volume name (names compare as exact text; two files preloaded from one folder
  are one volume);
- a volume named by a single letter (`c://` is a drive on Windows) or by `https`, `http`, `ssh`, `git` or `file`,
  which open a URL: rename the folder;
- `Volume("x")` when nothing preloads `x`, a literal `File("x://...")` naming no preloaded volume, and a `preload`
  that no `Volume` reads.

## Configuration is a class

There is no configuration file format. A configuration is an ordinary class whose constructor sets its state,
or, for anything that should differ between runs or builds, a field of `Environment` or `Build`, reopened in
the program's own folder. Both are Spite, so a setting can be computed, reflected on, and read in the REPL like
anything else.

---

Next: [Values and types](values_and_types.md), the types, variables and literals a program is built from.
