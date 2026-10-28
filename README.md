# Spite

> **Spite: you write your intention, the compiler reminds you that you are a moron, and emits the fastest
> possible code.**

Spite is a small, opinionated language meant to be written mostly by AI and skimmed by humans: one way to do each
thing, no macros, and metaprogramming and a real standard library instead of loops. Its compiler is written in Spite,
builds itself and writes machine code directly.

A file is a class named after it, and a program is a folder: `battle/battle.spite` is the entry, and its constructor
runs the program.

```gdscript title=battle/monster.spite
var name = ""
var health = 0

func Monster(new_name: String, new_health: Integer) {
    name = new_name
    health = new_health
}

func alive(): Boolean {
    return health > 0
}
```
```gdscript title=battle/battle.spite entry
var console = Console()

func Battle() {
    load "../spite_standard_library/windows"
    var monsters = [Monster("slime", 12), Monster("ghost", 0), Monster("troll", 30)]
    var standing = monsters.filter_alive().sum_health()
    console.print("health still standing:", standing)
    var troll = monsters.last()
    crash troll
    show_name(troll)
    show_health(troll)
}

func show_attribute(attribute: Spite.AttributeDeclaration, monster: Monster) {
    console.print(attribute.name, "=", monster[attribute])
}
```
```output
health still standing: 42
name = troll
health = 30
```

Nobody wrote `filter_alive` or `sum_health`. `List` has templates, `filter_<member>()` and `sum_<member>()`, and the
compiler writes the two this program calls for `Monster`, then fuses the chain into one loop with no list in between.
`show_name` and `show_health` are the same idea turned on your own code: `show_attribute` takes a
`Spite.AttributeDeclaration`, so naming an attribute in its place makes the compiler write a typed function for it,
one for `name` and one for `health`. `monsters.last()` may find nothing, so it is a `Monster?`, and `crash troll`
says an empty list here is a bug: after it, `troll` is a `Monster`. Nothing is looked up while the program runs, and
whatever it does not call is not in the executable ([collections.md](collections.md),
[metaprogramming.md](metaprogramming.md)).

Rust has Rustaceans; Spite has morons. A moron is anyone who writes Spite (people, AI agents and the language's own
author), said with affection, and the language is built so a moron's code still comes out right and fast:

- **You write the plain program; the compiler decides how it runs.** Threads, waiting, memory layout and where every
  value lives are the compiler's choice, proven while compiling, with no runtime shipped beside the program. A plain
  program slower than the same program tuned by hand is a compiler bug: [Write it plainly](write_it_plainly.md).
- **The compiler refuses mistakes instead of guessing.** An error names the problem and the fix (a race, a value
  that may be null, a misspelt or abbreviated name), and there is no warning to ignore. The compiler is also the
  formatter and the linter: it rewrites your file to the one style.
- **Nothing fails silently.** Every failure is a compile error or a crash that names its cause, never a wrong value,
  a leak or a hang ([failure.md](failure.md#nothing-fails-silently)).
- **Built mostly by AI, decided by a person.** Every mistake an agent makes is a guardrail the language is missing,
  and it becomes a rule the compiler enforces ([WHY.md](WHY.md#written-by-ai-decided-by-a-person)).

[WHY.md](WHY.md) says more: why to use Spite, what it refuses, and how to help build it.

## Try it

Clone the compiler and the standard library side by side and build the compiler (Windows x64 for now, no C
toolchain needed):

```bash
gh repo clone Spite-Language/spite_compiler
gh repo clone Spite-Language/spite_standard_library
bash spite_compiler/scripts/bootstrap.sh --install
```

Then put the two files above in a `battle` folder beside the clones and run `spite battle`.
[Getting started](getting_started.md) walks through it.

Writing Spite with an AI? Give it the skill in [skills/spite/](skills/spite/SKILL.md), with the whole language on one
page in [reference.md](skills/spite/reference.md).

## Reading order

These pages are the tutorial. Read them in this order the first time; each page assumes the ones before it and ends
with a link to the next.

**Getting started**

1. [getting_started.md](getting_started.md): build the compiler, run hello world, a program of two classes.
2. [write_it_plainly.md](write_it_plainly.md): why you write the plain program and the compiler decides how it
   runs, what it does with it, and how to write code it can make fast.

**The language**

3. [classes_and_files.md](classes_and_files.md): the lexical rules; a file is a class; what a file holds and in
   what order; constructors, singletons, `this`, references.
4. [programs.md](programs.md): the entry file, how a program loads the standard library, `Arguments`, run-time settings
   (`Environment`) and compile-time settings (`Build`, `target_operating_system`).
5. [values_and_types.md](values_and_types.md): variables, numbers and the casting rule, numbers as classes,
   `String`, enums, unions, `type` shapes and duck typing.
6. [failure.md](failure.md): `T?` and narrowing, reading with `[]`, the three failure outcomes, `assert`
   guards, `crash` and its report.
7. [functions_and_operators.md](functions_and_operators.md): functions, function values, variadic arguments,
   operators as functions, attribute interception.
8. [control_flow.md](control_flow.md): `if`, `while` (the only loop), `switch` with `_:`, and `value == Class`.
9. [style.md](style.md): the formatter and the lints: names, comments, blank lines, nothing unused, one call
   per line, and the short forms the compiler insists on.
10. [memory.md](memory.md): reference counting, `copy`, `drop`, cycles, borrowed items, placement, and
   why a program cannot observe memory.
11. [metaprogramming.md](metaprogramming.md): walks over a program's classes and functions, templates whose name
    carries the member, member templates, generics, codegen values and compile-time type tests, tree shaking.
12. [reflection.md](reflection.md): every class an instance of `Spite.Class`, a class against an instance, the
    `Spite` classes and their members, what folds and what is known only at run time.
13. [packages.md](packages.md): `load`, namespaces, reopening classes (mods), the reserved `Spite` namespace.
14. [concurrency.md](concurrency.md): `Concurrent` and `Parallel`, and waiting without `async`/`await`.

**The standard library**

15. [standard_library.md](standard_library.md): every class at a glance; `String`, files, folders, processes,
    the console, the program, sockets, and the floor the library is built on.
16. [collections.md](collections.md): `List` and `Dictionary` as library code, member templates, chains that
    run as one loop, templates of your own.
17. [json.md](json.md): `JsonWriter`/`JsonReader` and `BinaryWriter`/`BinaryReader`: any value to JSON text or
    compact bytes and back.
18. [time.md](time.md): instants, durations, the calendar, time zones as presentation, and ISO 8601 text.
19. [game_maths.md](game_maths.md): vectors, matrices and quaternions for games: `Vector3`, `Matrix4`,
    `Quaternion`, boxes, frustums and rays, column-major, Vulkan clip space, and what each operation costs.
20. [foreign_libraries.md](foreign_libraries.md): `DynamicLibrary`, libraries written in C, C++, Rust, Zig and Go,
    bindings that speak Spite, and how each operating system's folder reopens the classes it changes.
21. [targets.md](targets.md): other targets, the web, isomorphic classes and the wire format.

**Tooling**

22. [compiler.md](compiler.md): every command and flag, the outputs and where they go, `--final-classes`,
    `--development`, and which build to measure.
23. [repl.md](repl.md): the local and remote REPL, `spite connect`, live reload, and a replayed debugging
    session.
24. [testing.md](testing.md): a test is a function that crashes; the test package finds them itself, runs one by name, and why crashing tests suit an AI.
25. [optimizations.md](optimizations.md): everything the compiler optimises without being asked, and what (if
    anything) you could notice.
26. [proofs.md](proofs.md): every fact the compiler proves while compiling, what each one buys, and when it does
    not apply, so you know when to write the check yourself.

## About these pages

Every titled Spite program on these pages is compiled and run, what it prints is compared with the output written
under it (or it must fail with the error written under it), and its memory must balance. The formal definition of the
language is [the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/README.md), one page
for each page here with every rule, edge case and exact error text; where the two disagree, the specification decides
and the page is the bug. The code blocks are fenced as `gdscript` because GitHub cannot highlight Spite yet; the code
is Spite throughout.

For whoever writes or changes these pages:

- **The pages talk to people learning Spite.** No notes to agents, no implementation status ("not built", "planned"),
  no decision numbers or open-question bookkeeping in the middle of a page. Each page ends with a link to the next
  thing to learn.
- **Never name a third-party package.** Packages built on Spite are third-party, even the ones the author maintains.
  Write examples in Spite's own terms, and record a need found through a package as the language need it is.
- **No em dashes**, and no two hyphens between spaces standing in for one. A table has one set of columns, so write
  one row per item rather than two tables side by side.

## The repositories

Spite lives in the [Spite-Language](https://github.com/Spite-Language) organisation:

| Repository | What it holds |
| --- | --- |
| [spite_documentation](https://github.com/Spite-Language/spite_documentation) | this tutorial, [WHY.md](WHY.md) and the skill for language models |
| [spite_compiler](https://github.com/Spite-Language/spite_compiler) | the compiler, written in Spite and built by itself; how it works inside is in its [ARCHITECTURE.md](https://github.com/Spite-Language/spite_compiler/blob/master/ARCHITECTURE.md) and [OPTIMIZATIONS.md](https://github.com/Spite-Language/spite_compiler/blob/master/OPTIMIZATIONS.md) |
| [spite_standard_library](https://github.com/Spite-Language/spite_standard_library) | the standard library |
| [spite_spec](https://github.com/Spite-Language/spite_spec) | the specification, as programs every compiler must pass |
| [spite_benchmarks](https://github.com/Spite-Language/spite_benchmarks) | plain Spite programs measured against hand-tuned C |
| [mortaros_spite](https://github.com/Spite-Language/mortaros_spite) | every decision and why |

---

Next: [Getting started](getting_started.md), building the compiler and running a first program.
