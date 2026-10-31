# Getting started

## Install the compiler

The compiler is [spite_compiler](https://github.com/Spite-Language/spite_compiler), a program written in Spite that
builds itself and writes Windows x64 machine code directly. It needs no C compiler, linker or other toolchain. Its
standard library is [spite_standard_library](https://github.com/Spite-Language/spite_standard_library). Clone both
side by side in one folder, under their own names, and build the compiler:

```bash
gh repo clone Spite-Language/spite_compiler
gh repo clone Spite-Language/spite_standard_library
bash spite_compiler/scripts/bootstrap.sh --install
```

The script downloads the last compiler the source built itself, builds the source with it until two builds write
the same bytes, and leaves that compiler in `spite_compiler/.spite/spite.exe`; put that folder on your `PATH`.
[ARCHITECTURE.md](https://github.com/Spite-Language/spite_compiler/blob/master/ARCHITECTURE.md) says how it is built.
Only Windows x64 is written for now.

A program loads the standard library by its path, in the first lines of its entry function, so a program in a
folder beside the two clones starts with `load "../spite_standard_library/windows"`.

Then `cd hello_world` and `spite` compile the program of the [next section](#hello-world): the compiler builds the
folder it is run from.

## Build the compiler from source

The compiler builds itself: given a `spite.exe`, it compiles its own folder into the next one.

```bash
cd spite_compiler
.spite/spite.exe --build --executable-path=.spite/spite_next.exe
```

It runs in the compiler's folder, whose entry file loads `../spite_standard_library/windows`, the clone beside it. The next compiler is kept when it rebuilds itself to the same bytes. A build
from nothing, when no `spite.exe` exists yet, is the chain
[ARCHITECTURE.md](https://github.com/Spite-Language/spite_compiler/blob/master/ARCHITECTURE.md#how-the-compiler-is-built)
describes.

## Run a program

```bash
cd hello_world
spite                  # build it and run it
spite --check          # check that it compiles, build nothing
spite --build          # build it without running it
spite --high-ir        # also write the high representation into .spite/build/
spite --low-ir         # the same for the low representation
VOLUME=3 spite         # build it and run it with its setting volume
```

The executable is `hello_world/.spite/build/hello_world.exe`, and its exit code is what the entry constructor
answers. `spite` runs it attached to your terminal: what it prints appears as it prints it, what you
type is its standard input, and the compiler exits with the program's exit code. The program is given no flags:
it reads the settings its `Environment` declares ([programs.md](programs.md)) from the environment, and its
executable takes them as flags, `.spite/build/hello_world.exe --volume=3`. Nothing is given by position. [compiler.md](compiler.md) lists every option. The programs that every compiler of Spite must pass, one
for each rule, are in [spite_spec](https://github.com/Spite-Language/spite_spec).

## Hello world

A program is a folder, and its entry file is the file named after the folder. Make `hello_world/hello_world.spite`:

```gdscript title=hello_world/hello_world.spite entry
var console = Console()

func HelloWorld() {
    load "../spite_standard_library/windows"
    console.print("Hello, Spite!")
}
```
```output
Hello, Spite!
```

and run it:

```bash
cd hello_world
spite
```

A file is a class, named by its file name: `hello_world.spite` is the class `HelloWorld`. There is no `main`: the
program runs by constructing the entry class, so its constructor, the function named like the class, is the
whole program. Building and running are the same command; there is no separate compile step to remember. The
executable is built into the program's own `.spite/` folder, `hello_world/.spite/build/hello_world.exe`, never beside the source,
so it can be run again without the compiler ([compiler.md](compiler.md#choose-the-outputs) has the other outputs). It carries
only what the program uses: the standard library classes hello world never reaches are not in it, and there is
no runtime beneath it ([optimizations.md](optimizations.md#tree-shaking)).

## A second class

Every other `.spite` file in the folder is another class. Classes hold `var` fields (their state, each with a
default) and `func`s (their behaviour); an `enum` declared in a file belongs to its class (`Person.Job` from any other class, though
`'knight'` alone is usually enough). A `List<T>` holds them:

```gdscript title=tour_classes/person.spite
enum Job {
    'knight'
    'mage'
}

var age = 0
var job: Job = 'knight'

func Person(new_age: Integer, new_job: Job) {
    age = new_age
    job = new_job
}
```
```gdscript title=tour_classes/tour_classes.spite entry
var console = Console()

func TourClasses() {
    load "../spite_standard_library/windows"
    var party = List<Person>()
    var knight = Person(22, 'knight')
    party.append(knight)
    var mage = Person(19, 'mage')
    party.append(mage)
    party.each(introduce)
    var sum_age = party.sum_age()
    console.print("total age", sum_age)
}

func introduce(person: Person) {
    console.print("member", person.age, person.job)
}
```
```output
member 22 knight
member 19 mage
total age 41
```

Neither walk over the party is a loop anyone wrote. `each(introduce)` calls a function of yours on every member
([collections.md](collections.md#passing-a-function-for-each-element)), and `sum_age()` exists because `Person`
has an `age` ([collections.md](collections.md#member-templates-loops-you-do-not-write)). This is the whole idea
of Spite: reach for a function with the member's name in it before reaching for a loop. Both are written out by
the compiler as plain loops where they are used, so they cost exactly what the loop would.

## What the compiler will do to your file

Every compile formats the program's own files first, and may rewrite them: the compiler is the formatter, there
is one style, and nothing turns it off ([style.md](style.md)). `spite --check` formats and checks the program
without building it ([compiler.md](compiler.md#formatting)). The compiler will also refuse things other languages accept: an
abbreviated name, a variable nobody reads, a blank line inside a function, a call passed straight into another
call, an object constructed and thrown away. Each comes with an error that says exactly what to write instead. There
are no warnings: read the error and do what it says; there is no flag to silence it.

## Where to go next

Read [classes_and_files.md](classes_and_files.md), [programs.md](programs.md) and
[values_and_types.md](values_and_types.md) next, then [failure.md](failure.md): together they are most of the
language. Keep [compiler.md](compiler.md) at hand for every command-line option.

---

Next: [Write it plainly](write_it_plainly.md), why you write the plain program and the compiler decides how it runs.
