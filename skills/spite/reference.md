# Spite: the whole language on one page

Everything here is enforced by the compiler. When it rejects something it says what to write instead, and it
never warns: it errors or it is fine. A file that does not read or parse lists every such mistake in it as
`path:line: error: message` (its spelling first; once it spells right, every syntax mistake, one for each broken
statement or declaration); the other errors come as `path:line: error: message (in Class.function)`. A later stage
waits for the earlier one to be clean, so fix the list and compile again. Every compile first rewrites your files into the one
style (`formatted path` is printed), so what you read back may differ from what you wrote. Where a mistake has
exactly one fix the compiler makes it and prints `fixed: ...`; the list is under
[the compiler fixes](#the-compiler-fixes-what-has-one-fix).
The [documentation](https://github.com/Spite-Language/spite_documentation/blob/master/README.md) has a page for
each part of the language; this page is the working set.

Code blocks are fenced as `gdscript` only so GitHub colours them: GitHub has no Spite highlighter yet. Every
one of them is Spite, not GDScript: write `.spite` files.

## Run it

Run every command in the program's folder (`cd program`): the compiler builds the folder it runs in, and nothing
is given by position.

```
spite                                   build .spite/build/program.exe and run it
spite --debug-memory                    print the allocation balance at the end
spite --repl-port=4000                  serve the REPL; spite --connect=4000 --command="..." asks it
spite --check                           format and compile: the errors, if any (builds nothing, keeps an old executable)
spite --build                           build .spite/build/program.exe without running it
spite --check --optimization-report=report.md   list what was not optimised (heap objects, lists, copies), each with its line and why
spite --player-count=2                  a field the program's build.spite declares (kebab-case)
```

**Iterate on the running program, not by rebuilding**. Start it once with
`spite --hot-reload --repl-port=4000` and keep it running (it stays up after its constructor returns, until
a client sends `exit`). Save a file, then `spite --connect=4000 --command="wait_reload"`: it answers once the save is
compiled and swapped in, with what was rebuilt or the compiler's errors, and objects keep their state. Ask instead
of printing: a path or call (`World().player.health`, `Channel(1).number`, `monsters[0].roar()`,
`scores.contains("ann")`), an assignment (`player.health = 5`, `follower = Circle(3)`),
`describe Monster`, `classes`, `enums`, `memory`, and `eval <expression>` / `run <statement>` to compile code into
the running program. `break monster.spite:42` stops before that line; `where` and `locals` (`self` and every local in scope, one
`name: Type = value` per line) answer there, `continue` goes on, `breaks` lists and `clear` removes. A reload that
adds, removes, renames or retypes an attribute, or changes an enum's values other than by adding new ones last (a value's number included), is
refused, the program keeping all of its old code: send `exit` and start it again. Use `--check` only for the errors
or for a program that cannot stay running, and run the tests before calling the work done. Measure speed only on a
production build (no `--repl`, `--repl-port`, `--hot-reload` or `--debug-memory`): `--optimized` only sets
`Build.optimized` and does not make the program faster. A field of the program's `build.spite` that no flag sets is
read from its variable (`PLAYER_COUNT=2`), then from a `.env` line in the folder. `--target-operating-system` only
sets `Build.target_operating_system`: the executable is always Windows x64. Every answer is one JSON line. The
commands are in [repl.md](https://github.com/Spite-Language/spite_documentation/blob/master/repl.md).

A program lives in its own folder, and `spite` run in it runs it: the first line of its entry function loads the standard
library by its relative path (`load "../spite_standard_library/windows"`, which loads `common` itself; never `common` alone), and `game/`'s every sub folder is a
namespace and nothing else (`game/engine/renderer/shader.spite` is `Engine.Renderer.Shader`, and
`game/engine/physics/physics.spite` is `Engine.Physics.Physics`: no file is special). A file and a folder of one
name side by side (`engine/renderer.spite` beside `engine/renderer/`) is an error, since a dotted name means one
thing. A class that hides another class visible from an enclosing namespace (`Physics.Plugin` beside a root
`Plugin`) is an error naming both, and so is a class named like a standard-library class (`Geometry.Vector3` for `Vector3`; a namespaced library
class such as `Spite.Function` is always written with its namespace, so a `Function` of your own is fine): use the
library's class, reopening it to add what it lacks. Three or more classes in one folder whose names start with the
same whole word (`HighBlock`, `HighFunction`, `HighValue`) are an error: move them into the folder `high/` and name
them `Block`, `Function`, `Value` (only the program's own classes count, so `spite/function.spite` reopening
`Spite.Function` is fine). `load "folder"` inside a function loads another package (it takes literal text: a
folder path relative to the file, never absolute, or `repository@commit/folder` for a pinned repository; only its last
folder, the namespace, must be snake_case); a file at the same namespace path
reopens the class: same-named functions, attributes and enums replace (an enum declared again is its whole new
list of values), the rest are added; a reopening may not name a local, parameter or attribute like any function of the class. A reopening of a reflection class (`Spite.Class`, `Spite.Function`, `Spite.Namespace`, the other `Spite` classes describing the program) adds functions but cannot declare one the class has, by its library name or the name read (`get_is_singleton`, `is_singleton`, `name`): `'is_singleton' is a reflection function of Spite.Class, and reflection functions cannot be replaced`. A file in the program's own folder that reopens a class a loaded folder
declares loses wherever both declare a name, the constructor included, since loaded folders merge later: put a
replacement in a folder loaded after the one it replaces. Two `load`s of folders one inside the other are an error
(load only the outer one). A `load` under an `if` must be decidable from `Build` fields and literals, and a `load`
inside a `while` or `switch` is an error. A folder some file names in a `load` is a package of its own, never also a
namespace. `Build` is the exception: a field the program's own `build.spite` declares keeps the
program's value over a loaded package's (a flag, then the program, then the package). A repository pinned with
`load "../engine@6c7dca9/core"` is merged before the program, so the program's own declaration of a name wins over
it, and two commits of one repository are one library: what both declare must agree, or the compile stops naming
the declaration and both commits. A package opens the files
beside its own source through `class.source_files` (or `$item_type.source_files`, or `Recipe.source_files` for a
class by name), a `List<File>` of every file that declares or reopens the class, and a namespace's
`.source_directories` a `List<Directory>`: absolute paths on the machine that built it, never a path relative to where
the program runs.
A name that resolves to no class is `unknown type 'X'`, with `did you mean
'Component.Eye'?` when a class ends the same way: an environment's folder (`server/`) is not part of the name.
A program's settings and arguments reach it as typed, from bash, PowerShell or `cmd` alike:
`/Game/Legacy/` stays `/Game/Legacy/`, so do not work around Git for Windows' path rewriting.

## A file is a class

```gdscript
var name = ""
var health = 10
var target: Monster? = null
var friends = List<Monster>()

func Monster(starting_name: String) {
    name = starting_name
}

func hurt(amount: Integer) {
    health = health - amount
}

func is_alive(): Boolean {
    return health > 0
}
```

- The file name is the class name (`monster.spite` is `Monster`). A file holds declarations only, in this
  order: a `singleton` line, `generic $name` lines, enums, unions, types, variables with a default, variables without one, the constructor, conversions, presets, then
  functions (`a file holds declarations and nothing else` for a statement at file level). A file in
  the wrong order is put back in order by the compiler (`fixed: ... moved it up, since a file is ordered`), except the library's `Buffer`, `ForeignCallback` and `Spite.*` classes, whose attributes keep their written order because the compiler lays them out as written. Every `var` has a default value (`expected '=' (every variable needs a default value)`). A
  default may only read or write attributes declared above it (directly or through a function of its object, so a
  function a default calls that sets another attribute counts as a write). One reading or writing an attribute
  declared below it in the same file is moved for you (`fixed: '_hundred' was declared below 'tripled', whose
  default reads it; moved it above, ...`, or `whose default writes it`); a circle, a default reading or writing its
  own attribute, or a use the compiler cannot move (another file of a reopened class, a library file) is an error
  (`the default of '_first' reads '_second', whose default reads '_first': ...`, `writes` where it only writes). A
  name starting with `_` is private to its class, for reads and writes alike (`'_hidden' is private to 'Secret'`).
- The function named like the class is the constructor. Do not write an empty one: a class without a constructor
  is made from its defaults, and `func Monster() { }` is an error (`has an empty constructor: delete it`).
- A constructed object is kept and used: `Report(text)` written as a statement of its own is an error. A class
  whose construction is the whole point is a function instead (`report(text)` on the class that needs it).
- A program is a folder, and it starts by constructing the class of the file named after the folder
  (`game/game.spite` is `Game`). That constructor takes no arguments: settings come from `Environment()` and
  `Build()` below. Every value on the command line is named (`--player=value`, a bare `--verbose`) and is read
  only as the `Environment` or `Build` attribute it sets, never by its text or its position.
- `return` is always written. The return type is written `(): Type`. No return type means it returns `Nothing`,
  so `return 1` there is `'one' answers nothing, so 'return' takes no value`.
- Only `while` exists. There is no `for`, `break` or `continue`: an early exit is part of the loop's condition.
  Prefer the member templates below.
- There is no `self.`: members are read by name and functions called by name. `this` is the instance itself,
  only for handing it to something (`registry.record(this)`); `this.name` and `this.helper()` are errors.
- One function per name, no overloading (`'Shop' declares 'price' twice`): an argument is cast to the
  parameter's type. The last parameter may be
  `...values: List<Type>`, and the caller writes the values one by one (passing a whole list there is an error).
- A call is never passed straight into another call: compute it first into a named `var` and pass the name,
  `var token_text = source.slice(start, end)`. The error: `'text.slice(0, 1)' is called inside an argument of
  'console.print': compute it first into a named 'var' and pass the name`. The holes of a text are not
  arguments: `"{names.count()} names"` is fine.
- A constructor is not an argument either, on any line: make the object on a line of its own and pass its name,
  `var token = Token("number", token_text)`, then `tokens.append(token)`. The error:
  `'Bundle.CounterButton(screen.id)' is constructed inside an argument of 'world.create_entity_from_bundle': a
  constructor call is never an argument, so make it first on a line of its own, 'var counter_button =
  Bundle.CounterButton(screen.id)', and pass 'counter_button'`. That goes for `Label(Font())` and
  `buffers.set(List<String>())` too. A constructor read at once is fine (`JsonWriter<Order>().write(order)`).
- The two branches of an `if`/`else` never compute the same call (`'measure(2)' is computed in both branches`):
  compute it once before the `if`.
- An `if`/`else` never sits directly inside a branch of another `if`/`else`: move the inner decision into a
  function named for what it decides, or use one `switch` when both test which member of a union or which value
  of an enum a value is. A
  flat `else if` chain, and an `if` with no `else`, are fine. The error: `this 'if'/'else' is inside a branch of
  another 'if'/'else': move it into a function named for what it decides`.
- An `if` whose body is only another `if`, neither with an `else`, is an error: write one condition,
  `if modified and size { ... }`.

## Names, comments, unused things

- A variable, parameter or attribute never has the name of a function of its class: `var stem = file_stem(path)`,
  never `var file_stem = file_stem(path)` (`the variable 'file_stem' has the name of a function of this class`).
- `snake_case` for variables, attributes, parameters, functions and enum values; `PascalCase` for classes,
  enums, unions and types; never a single letter except the axis names `x`, `y`, `z`, `w`; never an abbreviation
  (`message` not `msg`, `index` not `idx`, `value` not `val`). A local or parameter with one right spelling is renamed in the file (`fixed: 'msg' abbreviated, ...; renamed it 'message'`, `myValue` to `my_value`); the rest are errors: `the variable 'n' is a single letter`,
  `'temp_file' abbreviates 'temp', which could mean 'temporary' or 'temperature': write the word you mean`. The language's own
  type names follow the rule: `Integer`, `Boolean`, never `Int`, `Bool`. File and folder names are snake_case too: one of the program's own with one right spelling is renamed on the disk with every `load` naming it (`fixed: the folder 'BadFolder' was not named in snake_case, ...; renamed it 'bad_folder'`, `ToolHelper.spite` to `tool_helper.spite`); one with two capitals in a row (`HTTPHelper`), a taken new name (compared without case), a library or pinned file, or a `load` that only runs through the folder stays the error.
- A function whose whole body answers a same-named member of an attribute a caller can reach is an error (`'get_name' only answers 'owner.name': remove it, and write 'token.owner.name' where it is called`), and so is a `get_` function reading another object's attribute under any name. An attribute private with no getter (`_connection`), a differently named question (`is_atomic`) and a call passing anything but its own parameters are allowed.
- A local that is never read is an error (`'total' is never read: remove it`); one never assigned whose value is a
  literal or a name is removed by the compiler (`fixed: ... removed it`). Assigning is not reading, and no
  spelling silences it. A parameter the signature needs but the body ignores is named `_name` (`the parameter
  'amount' is never read: remove it, or name it '_amount' if the signature needs it`); a `_name` that is read is
  renamed `name` in the file, with a `fixed:` line. The parameters of an operator function, of a one-parameter
  `set_<attribute>` and of a template over members need no `_`. A function nobody calls is never reported as unused, yet every build (`--check` too) compiles and checks it, outside a generic class, so a mistake in it is still an error. A use inside a branch a codegen test rules out (`if $slot_type == Entity { ... }`) still counts, so
  a parameter only that branch reads is not unused in the other instantiations.
- An attribute nothing reads is an error too (`the attribute 'world' is never read: remove it`). A private `_name` attribute is checked too.
  Assigning it is not reading it, and a read counts only for its own class: `tag.label` reads `Tag.label`, not
  the entry's `label`. A public attribute of a folder the program `load`s is not checked, since
  code the program does not load may read it, except a singleton binding (`var world = World()`), which is an
  error unread anywhere: a class that needs the singleton binds it itself. A template over a `Spite.Attribute` counts only when it reads the value
  (`x.attributes[attribute]`). An attribute kept as a marker that a walk inspects through `attribute.name` or
  `attribute.class` is unused, so say what the marker means some other way. A function nobody calls still
  reads what it names.
- The words other languages use for things Spite writes differently (`none`, `nil`, `undefined`, `self`,
  `new`, `import`, `require`, `elif`) are errors wherever they appear, naming the Spite form, so none of them
  can name a variable or a parameter either: `var none: Long = 0` says to write `null`. Pick another name
  (`no_handle`, `empty`). `load` is reserved too: it always loads a package, so a function that loads something
  says what (`load_texture`). So are `type` and `singleton`, never a name even after a `.`: a kind of thing
  is `kind`, the one object is `instance`.
- A number is plain digits with `_` between digits for reading (`1_000_000`, `0.000_01`); there is no exponent
  (`2.5e3` is an error that writes `2_500`).
- `class`, `attributes`, `functions`, `instances` and `memory` never name an attribute or a function (nor a
  `get_`/`set_` one): reflection gives every object those members, and `JsonReader` and every attribute walk read
  through `value.attributes`. The error suggests the class's name in front (`table_attributes`). A local may use
  them.
- No name is taken by a language beneath Spite: `short`, `default`, `register`, `static`, `unsigned`,
  `stdout`, `near`, `far` and `pascal` are ordinary names for a variable, attribute, parameter or function, and
  so are `allocate`, `make`, `retain` and `release`. Do not rename around C. (`int`, `char`, `bool`, `min` and
  `max` are still errors, as abbreviations.)
- A function body holds no empty lines: the blank line is where a second function wants to be, so name the part
  below it and call it. The formatter (and so every compile) deletes one you write.
- A file holds at most 1200 statements, its file content budget: more is an error (`this file holds 1201 statements, over its file content budget of 1200: ...`), with no way to lift it. Every statement counts as one at any depth, and so does a class-level `var`; a statement spread over many lines is one. Comments, blank lines, closing braces, function headers, the entries of an enum, union or type, and the cases of a `switch` (the `switch` itself is one) count nothing, so an enum or an exhaustive switch over it can be any size. Get back within it by cleaning up, not by cutting the file at the number: group state into an object, and give each concern a file of its own.
- A comment is one line, right above a function or at the top of the file (the class's heading; one anywhere else
  between declarations is moved there), and nothing but a link to a markdown heading:
  `# notes.md#why-this-exists` (relative to the folder of the file it is in, like `load`; the file and the heading must exist; anchors as GitHub makes them, a repeated heading's second one `#details-1`). One inside a function or inside an `enum`, `union` or `type` body is an error. Only the program's own files are checked: the comments of the standard library and of a package are dropped when they are read, so their links never stop a build.
  Anything else, including `//` and `/* */`, is an error. If the code already says it, do not write it.

## The compiler fixes: what has one fix

A mistake with exactly one fix is not an error: the compiler writes the fix into your file, prints
`path:line: fixed: ...` and compiles the fixed file, so read the file back before editing it again. It fixes:

- declarations out of the file's order, and a union or `type` that two classes of a namespace declare alike (moved to
  the namespace's entry class, see Values);
- a local or parameter name with one right spelling (camelCase, a doubled `_`, an abbreviation with one expansion, a
  read `_name`), and a keyword one letter off at the start of a line (`retrun`);
- a local or attribute nothing reads whose value and every write do nothing, and statements after a `return`;
- a load of the library's `common` folder alone, which becomes a load of `windows` beside it (which loads `common`);
- a line ending in `\r\n` or a lone `\r`: a Spite file ends every line with `\n`, and the whole file is written
  back that way;
- a check on a proven value (`fallback`, `crash` or `if`), a local copied only to narrow (every use names the source, a
  copy of a copy included: `var receiver = call.inner` then `var source = receiver.inner` leaves `call.inner.inner`), an `if` that only returns
  what `fallback` answers, and an `if` or `while` the compiler decides while compiling (an `if` always true becomes its
  body; a `while` always false goes). A local taken from a narrowed path is proven too (`crash thing.store`,
  `var held = thing.store`: `if held` loses its `if`). A fixed check or decided `if` rests on what was narrowed:
  when a call or a loop's next pass takes that narrowing back, the compile fails on it and leaves those lines as
  written.

A mistake whose fix is a choice stays an error, and a file of the standard library or of a package is never
rewritten.

## Values

- Numbers: `Integer` (32 bit, the default), `Long`, `Tiny`, `Short`, `Byte`, `UnsignedShort`, `UnsignedInteger`,
  `UnsignedLong`, `Float` (the default for decimals), `Double`. `Boolean`. `String` (double quotes only; escapes
  `\n`, `\t`, `\r`, `\\`, `\"` and `\{` for a literal brace, in any order: `"\\\{"` is a backslash and a brace; a
  text inside a hole keeps its own escapes and holes).
- No cast syntax: the right side is cast toward the left. `"age {3}"` is `"age 3"`; `var total: Integer = "12"` reads
  it while compiling. A value meeting another class is cast through that class's conversion, `func Color(from: String): Color?`
  declared in `Color`, wherever it meets it (argument, `return`, assignment): `var paint: Color? = text` calls
  it, and the cast answers what the conversion answers (a `T?` is narrowed before a plain `T`). A literal at that place
  (text with no hole, a number with its sign) is worked out while compiling in every build: `var paint: Color = "#ff8000"`
  needs no narrowing, one that answers `null` is a compile error (`the literal "x" does not convert: 'Color(from: String)'
  answers null for it`), a `T?` holding one is proven present so a check of it is refused, and a conversion that needs a
  run (asks the system, prints, a singleton) stays a `T?`; a local holding a literal is not one. A conversion is never
  called by name (`Color("#ff8000")`, `Integer(text)` and `Float(count)` are errors: write the value where the class is
  wanted), and a class with no such conversion is an error naming it (`IpAddress declares no
  'IpAddress(from: Point)'`). A class is built three ways: its constructor, a conversion (`func X(from: Y): X?`, always its class's `?`: `: X` is an error, `the conversion 'X' answers 'X', but a conversion always answers its class's '?'`; one with no `null` path needs no narrowing) and a
  preset (`func Pi(): Float`, called `Float.Pi()`). Anything with a `to_string()` (numbers, `Boolean`, enums, a class that declares one) casts to text wherever a
  `String` is wanted: `label.text = clicks`, `show(clicks)`, `return clicks`, `names[badge]` on a `Dictionary<String, V>` (a whole number given as its key is cast to its text: `scores[7]` is
  `scores["7"]`). So a
  text of one hole and nothing else, `"{clicks}"`, is an error naming the direct form; elsewhere write
  `clicks.to_string()`. Arithmetic is done in the left side's type, so write the wider operand first: `total * count` with a `Long`
  `total`, never `count * total`, which is an error (so is an `Integer` plus a `Float`); a literal on the right that
  fits is fine. A constant that overflows `Integer` (`65536 * 65536`) is an error: write the number. Comparisons follow
  the same rule: `count < total` with a `Long` `total` is an error, write `total > count`. Nothing wraps silently: `+ - *`, unary
  `-` and the smallest signed value `/ -1` that do not fit their type halt naming the line in every build, signed
  and unsigned alike (the check is left out where the ranges of local numbers prove the answer fits: a `Long` total
  of `Integer` terms over a list, a remainder, a clamped value, a value bounded by a named number nothing writes
  such as `floor.minimum(_hundred)` or `fallback amount < _thousand`), and so does a value assigned, passed or returned into a narrower type it does not fit (a
  number written there that does not fit is a compile error). Where wrapping is the point (a hash), call
  `a.wrapping_sum(b)`, `a.wrapping_subtract(b)` or `a.wrapping_multiply(b)`, which keep the low bits. A whole number divided by zero (`/` or `%`) halts naming the line, and a divisor written as zero is an error; after `fallback divisor != 0` the check is gone. Floats keep infinity and not-a-number.
  An integer literal is an `Integer`: `var big = 3000000000` is an error (`does not fit in an Integer ... store it
  in a wider type`); write `var big: Long = 3000000000`. `%` on a `Float`/`Double` keeps the left side's sign
  (`-7.5 % 2.0` is `-1.5`). A decimal prints its shortest round-trip digits (`0.1`, `1e+21`), and `inf` / `nan`.
- A `var` may redeclare a name in the same scope, with a new type: `fallback content` then
  `var content = content.trim()` turns a `String?` into a `String`. The right side reads the old binding, then the old
  value is dropped. A shadowed name never read is removed with `fixed:`. A `var` never shadows a function it can see.
- Bits are functions on the whole numbers, never symbols: `value.shifted_left(count)`, `shifted_right(count)`
  (arithmetic on a signed type, logical on an unsigned one), `bits_and(mask)`, `bits_or(mask)`,
  `bits_exclusive_or(mask)`, `bits_inverted()`, `set_bit_count()`, `leading_zero_count()`, `trailing_zero_count()`.
  The mask is cast to the receiver's type; a count of the width or more shifts everything out, a negative one
  halts. Do not fake them with `/` and `%` by powers of two.
- A decimal literal beside a `Float` is a `Float` (`tenth == 0.1` is `true` after `var tenth: Float = 0.1`). The
  last bits of a decimal result are never a promise: loops may fuse multiply-adds and reorder sums, except a loop
  that compares decimals with `==` or `!=`.
- Maths is functions on the numbers too, every name in full, never a `Math` class: on a `Float` or `Double`
  `square_root()`, `sine()`, `cosine()`, `tangent()`, `arc_sine()`, `arc_cosine()`, `arc_tangent()`,
  `rise.arc_tangent_over(run)` (C's `atan2(rise, run)`), `power(exponent)`, `exponential()`, `logarithm()`
  (natural), `logarithm_base_2()`, `logarithm_base_10()`, `floor()`, `ceiling()`, `round()`, `truncate()`,
  `absolute()`, `minimum(other)`, `maximum(other)`, `clamp(low, high)`, `is_finite()`, `is_infinite()`,
  `is_not_a_number()`; on a whole number `absolute()`, `minimum`, `maximum`, `clamp`. Each is the C library's function,
  called inline; do not write your own `sine` or square root from a series. Constants are presets, called on the
  class: `Float.Pi()`, `Float.Tau()`, `EulerNumber()`, `Infinity()`, `NotANumber()`, `Largest()`, `Smallest()` (the most
  negative), and `Integer.Largest()`, `Long.Smallest()` and so on (`angle.pi` and `Float.pi()` are errors naming `Float.Pi()`,
  and a value's function called on the class, `Float.square_root()`, is an error too). Nothing halts: `(-1.0).square_root()` is
  `nan`, and `nan` passes on through `minimum`, `maximum` and `clamp` (`nan.minimum(0.0)` is `nan`).
- Everything that is not a number, a `Boolean` or an enum value is a reference: passing, assigning and storing share
  the same object. `copy()` copies one level, `deep_copy()` all the way down (through unions and `type` shapes by
  the class each value holds; a union value has `deep_copy()` too; an object reached twice through a cycle is copied once, back references included; on a `String` both answer a new text; a class that declares its own `deep_copy()` is copied through it wherever a
  deep copy reaches it); inside a class, a bare `copy()` or `deep_copy()` copies the object itself. `drop()` runs when the last reference goes. Two objects that refer to each other (`parent.adopt(child)` where `Child(name, this)` stores the parent, and the parent keeps the child) are plain references: the compiler picks the back reference from how the program links them, reads it as `null` once its object is freed, and frees the cycle; a cycle it cannot read this way leaks, so clear one side. A back reference holds no count, so after a call that may free its object (`child.parent.leave()`), prove it again (`crash child.parent`) before reading it; an old proof halts at run time naming the attribute.
- `var item = list[index]` names the stored item: `item.x = 1` writes it, but `item = made` only renames the local,
  and an assignment nothing reads after is an error; replace an item with `list[index] = made`. Writing back
  `list[index] = item` after changing it costs nothing when nothing between can change the slot.
- An assignment runs its value before it reads its target: in `items[position] = moved()` a `moved()` that sets
  `position` moves where the value lands, and `boxes[next].amount = refill()` reads `boxes[next]` after `refill()`, in
  a state machine too. A proof of the target that the value's call may take back is refused there: name the object
  first (`var target = keeper.target`, then `target.body = made()`).
- `List<T>`: `[1, 2, 3]`, `append`, `prepend`, `insert`, `remove_at`, `remove_last`, `remove_first`, `first`,
  `last` (both a `T?`, `null` on an empty list, like `[]`: `var first = names.first()` then `crash first`), `count`, `contains` (elements that are numbers, `Boolean`, `String`
  or an enum only: on a list of a class use `any(f)` or `find_by_<member>`; a list of texts or whole numbers only
  appended to, searched and read gets a hash index from the compiler, so `if not seen.contains(name) {
  seen.append(name) }` is how to write a set, never a `Dictionary` kept beside it), `index_of` (an `Integer?`, the same
  elements as `contains`), `is_empty`, `clear`, `reverse`,
  `join` (text, numbers, `Boolean` and enum values all join), never `add` or `pop`. `list[index]` is a `T?` (out of range gives nothing), and `crash names[index]` narrows it like a path,
  and so does `crash glyphs[code - 32]`, or any index with no call in it, with no copy into a local first;
  `crash names.count() == 3` (or `>= 3`, or `> 2`) proves `names[0]` to `names[2]`, so does
  `var names = ["a", "b", "c"]`, `crash not names.is_empty()` proves `names[0]` (a list's, never a dictionary's),
  and `while index < names.count()` proves `names[index]` in the loop body, so a `crash names[index]` inside that
  loop is an error saying so: delete it). `[]` calls `get_at(index)`, which is only ever called through `[]`:
  `names.get_at(0)` is an error naming `names[0]` (passing `counts.get_at` as a value is fine). Every `[]` answers a `T?`. A
  list also has `remove_swapping(index)`; how much room it keeps is the compiler's.
  `Dictionary<Key, Value>` (insertion order) names its key type, then its value type:
  `Dictionary<String, Integer>()`, `Dictionary<Integer, Monster>()`, `Dictionary<Player, Party>()`; one type
  argument is an error. `dictionary["key"] = value`, `contains(key)`, `remove_at(key)` (nothing happens when the key is absent), `count`, `is_empty`, `clear`, `keys` (a `List<Key>`),
  `values`, `dictionary["key"]` (a `Value?`, through `get_at` like every `[]`, like `list[index]`: `inventory["shield"] == 0` is false for an absent
  key). Any number, `Boolean`, `String`, enum or object can be a key: numbers by value (key by the number
  itself, `created_layouts[entity]` in a `Dictionary<Integer, Layout>`, never by text made from it), an object
  by identity (one object, one key; the dictionary holds it). A class that declares `equals` cannot be a key
  (key by what `equals` compares); nor can a nullable, a union, a `type`, a function value or a value class. A dictionary literal writes its
  entries, `{"cactus": "cacti", "foot": "feet"}` or `{1: 1, 2: 4}` (keyed `String` or `Integer` unless declared);
  a key written twice is an error, and an empty one is `{}` where its type is declared
  (`var scores: Dictionary<String, Integer> = {}`), `Dictionary<Key, Value>()` elsewhere. A list or a dictionary is not printable: `console.print(list)` is `'List<Integer>' does not fit type
  'Printable'`; print `list.join(", ")`, or `console.debug(list)`.
- A chain of templates, `teams.filter_active().map_leads().sum_age()`, runs as one loop with no list in between.
- To remove many elements, never call `remove_at` or `remove_swapping` in a loop: `creatures.remove_where_dead()`
  or `numbers.remove_where(is_odd)` removes every one the test is true for in one pass and keeps the rest in
  order (never on a chain or a `Dictionary`). `truncate(count)` drops the tail. When the test is not a function of the element (a mask of
  rows), walk the rows, `swap(row, kept)` each one that stays, then `truncate(kept)`.
- `List<T>` is the only list. Whether a list keeps its items inline is the
  compiler's choice, never yours: an item read from a list is the item itself, so
  `var velocity = velocities[index]` then `velocity.across = 3.0` writes the item every other name sees, and the
  item may be kept, passed, returned and read after the list grows.
- A row of list items for a system: `var row: Moving = {position: positions[index], velocity:
  velocities[index]}` (`Moving` a `type`), then `mover.update_each(row)`. Its attributes are the items themselves, so
  `moving.position.left = 3.0` writes the list's item.
- A generic runner builds the same row with a walk: `var row = $row_type()`, then on the next line
  `fill_attributes(row, index)` whose template is `crash columns.attributes[attribute][index]` and then the line
  `row.attributes[attribute] = columns.attributes[attribute][index]`: a `[]` read answers a `T?` and a row holds
  items, so the template states each read with a `crash` line before the fill. The compiler writes those checks
  and the literal in its place.
- Over sparse sets (each entity at a different place in each column), the runner first walks the attributes to
  fill `found: List<Integer>` with each one's place, then the fill template reads each attribute from its own
  generic singleton, after a `crash` line for each read (the places arrive as the parameter `rows`): `crash
  rows[attribute.index]`, `var stored_row = rows[attribute.index]`, `crash
  Column<attribute.class>().values[stored_row]`, then `row.attributes[attribute] =
  Column<attribute.class>().values[stored_row]`. An index never holds another `[]` read. Choose
  per attribute with an `if` on `attribute.class == Entity` or `attribute.class.is_fixed_size` (both decided while
  compiling), each branch one such line: `Entity(entity)`, or a reference column's value
  (`ReferenceColumn<attribute.class>().at(rows[attribute.index])`, after `crash rows[attribute.index]`) is counted
  for the row. Keep the index
  expression to parameters, `attribute.index` and whole numbers; work anything else out before the walk. The
  places may live in another object: pass its list straight in (`fill_attributes(row, matcher.rows)`) or read
  `matcher.rows[attribute.index]` into the line's `var` (its items are numbers in a `List`), and never copy them into a
  list of the runner's first.
- In a generic class that keeps values of a type it does not know (an engine's `Column<$component_type>`), use
  `List<$component_type>()`: one column class serves every component, and the walked row's line is
  `Column<attribute.class>().values[stored_row]` (after `var stored_row = found[attribute.index]`) for all of them.
- A system may take components as arguments (`update_each(position: Position, velocity: Velocity)`); the runner
  passes them with `system.phase_each(made_arguments(found))` on a line of its own, where `made_argument(argument:
  Spite.Argument, rows: List<Integer>): argument.class` is `crash rows[argument.index]`, `var
  stored_row = rows[argument.index]`, `crash Column<argument.class>().values[stored_row]` and `return
  Column<argument.class>().values[stored_row]` (an `if` on `argument.class == Entity` may choose another
  line, such as `return Entity(entity)`). The template's `Spite.Argument` parameter must be a word of its name (`made_argument(argument: ...)`), or
  it has no plural.
- `null` is only for `T?`: `var target: Monster = null` is an error; write `var target: Monster? = null`
  for none yet, or `var target = Monster()` for a default. In a generic class write `var system = $system_type()`,
  in a template `var made = argument.class()`, and a local a walk fills is `var row = $row_type()` then
  `fill_attributes(row, ...)`. An attribute the constructor gives a value on every path has no default:
  `var target: Pet` (and `var held: $held_type` in a generic class), below the attributes with a default; a path that
  leaves it without a value is an error naming the path. A default the constructor always replaces that makes
  nothing else (a literal, `null` for a `T?`, an empty `List`/`Dictionary`) is removed by a `fixed:` line; one like `Pet()` stays and runs, and `= null` on a type without `?` stays an error.
- `null` assigned, passed, appended or answered where a type without `?` is wanted is a compile error too, never a
  default (`0`, `false`, `""`, an enum's first value, an object of a class or a `type`): `'null' is given where a
  Crate is needed, and a Crate never holds null: write 'Crate?' where none is a real answer, otherwise give it a real
  Crate`.
- To hand one stored component to a caller, return it straight from the singleton's storage: `func of(entity:
  Integer): $component_type?` ending `return column.values[row]`. The caller gets the stored item, so
  `layout.order = 3` writes the column; never return `.copy()` for this, since writes to a copy are lost.
- A runner that treats systems of one row and of several differently asks `if phase.arguments.count() == 1 { }`
  of each `phase: Spite.FunctionDeclaration` it walks (`var phases = $system_type.functions.filter_name_ends_with("_each")`,
  then `phases.each(run_phase)`),
  or of one function, `$system_type.functions["update_each"]` narrowed first. It is decided while compiling, so the branch for the other arity is
  never compiled: a `Stream<$system_type, argument.class>` that calls `system.phase_each(row)` is made only for
  one-row systems. With a pattern whose functions take different counts, ask inside the walk instead.
- `fallback` and `crash` on a compile-time question fold like `if`: `fallback $slot_type == Entity` then `return
  value.id`, in a function returning `Integer?`, compiles for every `Slot<T>`, answering `null` where `T` is not
  `Entity`, with no run-time test (in one returning `Integer`, write `if $slot_type != Entity { return 0 }`);
  `crash $component_type.is_fixed_size` halts every call in an instance whose type does not fit.
- Do not hand-optimise: the compiler folds `Build` fields and codegen tests, fuses chains, appends to text in
  place, puts short-lived buffers in the frame and shakes out what is unused, on its own. Every such optimisation,
  built or planned, and what it could ever change that you see, is in [optimizations.md](https://github.com/Spite-Language/spite_documentation/blob/master/optimizations.md).
- On a list or dictionary of a class or of text: `filter_<member>()`, `count_<member>()`, `any_`, `all_` (a `Boolean` member),
  `sum_<member>()` (a number), `sort_by_<member>()`, `find_by_<member>(value)` (a `T?`),
  `find_index_by_<member>(value)` (an `Integer?`, on a list, never a dictionary), `map_<members>()`,
  `each_<member>()` (a function that answers nothing). On a list of a union, `filter_<classes>()` keeps one member class and answers a
  list of it: `entries.filter_files()` is a `List<File>`, and `count_files()`, `any_directories()`, `all_files()`
  and `remove_where_directories()` take a member class the same way. A member is an attribute or a function that takes nothing; text has them too
  (`names.map_upper_cases()`, `names.filter_is_empty()`, `names.sort_by_length()`). A member that does
  not fit is an error naming what the template needs: `count_stars()` on a number member says `but 'count_' needs
  it to return Boolean (to add up a numeric member use 'sum_stars')`, and `each_size()` on an attribute says
  `'each_' needs it to be a function`.
- The yes-or-no ones (`filter_`, `count_`, `any_`, `all_`, `remove_where_`) also take a value of the enum of the
  one member typed with it: `posts.filter_published()` keeps the posts whose `stage == 'published'`. Two members
  whose enums list the value, or a member of that name, is an error. A question member `is_alive` may be asked as
  `filter_alive()`, `count_alive()` and so on when the class has no member `alive`. `map_` takes the plural
  (`map_names`; `map_name` is an error), the plural `pluralize()` spells (`map_people` for `person`, never
  `map_persons`), and two members with one plural are an error: rename one. A template of your own answering
  `List<member.class>` is declared in the plural too (`collect_members(member: ...)`). Over a list of a union, a name that
  is both a member class and an attribute every member has (`count_wheels`) is an error: rename one.
- A `while` that only walks a list doing what one of these does (`var index = 0`, `while index <
  items.count()`, `total = total + items[index].price`, `index = index + 1`) is an error naming
  `items.sum_price()`: `this 'while' walks every element of 'items' only to add up 'price': write 'var total =
  items.sum_price()'`. The same goes for a `while` that only passes each element to one function of yours:
  `say_hello(names[index])` is `names.each(say_hello)`, and likewise `filter`, `count`, `sum`, `find`,
  `any` and `all`, on a list of anything, numbers and text included. Keep `while` for loops that need the index,
  pass more than the element, or walk state.
- A template sees only the element and the list, never your class: `names.each_say_hello()` looks for a member
  `say_hello` of each name, and says `a template reads a member of each element, never a function of this class,
  so pass this class's 'say_hello' instead: 'each(say_hello)'`. To call a function of yours with each element, pass it: `names.each(say_hello)`,
  `names.filter(is_short)`, and `any`, `all`, `count`, `find` (the first element it is true
  for, a `T?`), `sort_by` and `sum` the same way, on a list of anything (a `Dictionary` takes only `each`, `filter` and `sum`,
  through its values; ask `dictionary.values().any(f)` for the others), chained with the member
  templates or not (`people.filter_active().each(greeter.greet)`). The function takes the element as its only
  argument and is bound to its owner: `greeter.greet` is `greeter`'s, and a library value's works the same:
  `keys.filter(counts.contains)`, `words.filter(greeting.contains)`, and each may be held too (`var check =
  greeting.contains` keeps its own copy of the text). `counts.get_at` answers `T?`, so it cannot order a
  `sort_by`: pass a function of yours that narrows it. A function that needs more than the element
  (`print_statement(statement, depth)`) keeps its `while`. There is no `map(f)`: a value collected from each
  element is a read-only attribute of its class (`get_double()`), collected in the plural with `map_doubles()`, and a value
  that needs more than the element is collected by a `while`. `map_` on a `Boolean` member (`map_is_alive()`)
  is an error naming `filter_`, `count_`, `any_` and `all_`. `Monster.is_alive` is no function value (a class is
  its `Spite.Class` object, which has no `is_alive`): pass an instance's, `monster.is_alive`.
- `enum`, `union` and `type` declarations take no `=`, one entry per line, no commas:

  ```gdscript
  enum Job {
      'knight'
      'mage'
  }
  ```

- An enum's values are single quoted and resolve from where they are used. A reopening file that declares the
  enum again replaces it: its list is all the values the enum has, so adding one means restating the rest. An enum is a class, and `Course.values` lists its values
  in order, an ordinary list: `Course.values.each(list_course)` calls `list_course(course: Course)` once per value,
  unrolled while compiling (`$value_type.values` in a generic). Text becomes a value only as a `Course?`:
  `var course: Menu.Course? = name`; a conversion is never called by name.
  An enum named as a value, or `value.class` on one of its values, is its `Spite.Enum` (`.owner`, and `.values`, its
  `Spite.EnumValue` declarations by name, each with `.name` and `.number`).
  An enum, a union and a type are always declared inside a class: a file holding only one of its own name is an error. A value may be given a number with `=` on its line
  (`'admin' = 99`; the next unnumbered value counts on, the first from 0; two values with one number are an
  error), only where a C library, a file format or the network fixes the numbers.
- A union is declared in the class that consumes it. One name and content declared by two classes of a root is moved
  by the compiler to the entry class of the lowest namespace holding both (the class named like its folder, its file
  created if missing), and every use is rewritten: `Bunda.Pet` in a class of that namespace or below, plain `Pet` only in
  the entry class. Outside that namespace it is never named. A union value fits another
  union when all its members are members of it, and a caller passes the member value itself; a `Gitty?` fits a `Place?`
  (a member's `T?` into the union's `T?`) with no narrowing, a plain `Place` only once narrowed. Unions with the same
  members are one type inside a generic (`List<A.Pet>` is `List<B.Pet>`).
- A `type` or a `union` makes no object: `Tag("favourite")` is an error (`'Tag' is a type, so it makes no object: ...`). Make an object of a class that fits it.
- `union Enemy { Player Monster }`, written one member per line: `switch enemy { Player: ... Monster: { ... } }`
  must cover every member and narrows `enemy` inside each case; a function or attribute every member has can be
  used on the union directly. Every member is a class of objects: `String`, a number, `Boolean` or `List<Integer>` is
  refused as a member, so wrap it in a class of your own that keeps it in an attribute.
- A `type` declares a shape (`label: String` and `render: Spite.Function<Integer, String>`, one per line, with `Spite.Function<Nothing>` for no answer and never a call form `render(Integer): String`, which the compiler rewrites; a function of the class or an attribute holding such a function meets it, and `item.render(12)` calls either; a required function
  naming the *types* it takes and never the names) and accepts any class, or
  object literal `{ label: "x" }`, with those attributes and functions, typed as the `type` says. One rule decides
  every fit (a value given, a member's type, a `generic` constraint, `$T == Named`, `satisfies_type`): if the
  contract matches it fits, numbers included (`Integer` fits `type Textual { to_string: Spite.Function<String> }`),
  an enum fits a `type` asking only `to_string`, a union fits when each class does. To admit only some classes write
  a `union`, not a `type`. A `T?` fits a `type` that asks only for
  `to_string()` and `to_debug()` (null answers both with `null`); any other `type` needs it narrowed or proven present
  first (`this value may be null (it is a Rock?), but a Singer is needed: ...`).
- `Anything` is the built-in empty `type`, the counterpart of `Nothing`: `component: Anything` and
  `List<Anything>()` accept any object (a number keeps its class, with no allocation). Never declare an empty `type` of your own.
  `Anything` never holds `null`: `null` assigned, passed, appended, answered or declared into a plain `Anything` is a
  compile error (`'null' is given where an Anything is needed, and an Anything never holds null: ...`). `Anything?`
  is a smell: write it only where none is a real answer, and prefer a real type.
- No value or constraint is typed `Number`. Write number code as a generic class that declares the `type` its codegen
  value needs, listing only what it uses (`generic $number_type: Measure` with `type Measure { sum: Spite.Function<Measure, Measure> }`);
  every number class (`Tiny` to `Double`, not `Boolean`) has the operators' functions, `sum` to `remainder`,
  `less_than`, `greater_than`, and so may a class of your own (a custom math library). Ask `$value_type.is_number`
  (never ten comparisons) to tell a library number apart in a walk, and `kind.satisfies_type(Number)` for any class
  answering a number's operators (`sum`, `subtract`, `multiply`, `divide`, `remainder` taking and answering the
  class, `equals`, `less_than`, `greater_than` answering a `Boolean`).
  Inside a `type`, its own name in a signature is the class that fits.

## Nothing, null, and failure

- **Anything that can go wrong silently is a bug**. Write every failure loud: a compile error where the
  compiler can know, a `crash` naming its cause where only the run can. Never let a caller get a default it cannot
  tell from a real answer, lose a write, skip a step, leak or hang. When a function cannot answer, its result says
  so (`T?`, or an empty list), or it returns a value you chose on purpose, or it crashes because a caller broke the
  rule ([failure.md](https://github.com/Spite-Language/spite_documentation/blob/master/failure.md#nothing-fails-silently)).
- A call that answers a value is never a line of its own: use the answer (keep, pass, return or compare it), or at
  least `fallback file.write_string(text)` to carry on without it or `crash file.write_string(text)` when its absence
  is a bug (`answers a 'Boolean' and drops it`). Only a function that answers nothing is called as a statement. A walk
  counts too: `speeds.each_described()` where `described` answers a value `drops each answer`; collect with
  `speeds.map_described()`, or walk a member that answers nothing. A `DynamicLibrary` call is not checked.
- `crash` and `fallback` take a condition and never a message: the report names the file and line and shows
  the values there. Never add a clause only to get text printed (`crash found or name == ""`).
- `Monster?` is a value that may be `null`. It must be narrowed before use: `if target { }` (with `else`),
  `fallback target`, `crash target`, `while target { }`, or `switch target { Monster: ... Null: ... }`. One
  `fallback a.b.c` narrows the whole path. `null` is never compared against: `value == null` is an error. Two `T?` values compare without narrowing, and two missing ones are equal.
  Narrow the name or the path itself (`fallback target`, `fallback target.weapon`), never a local copied from it,
  which is an error. Only a path with no call in it narrows: after `if thing.pop().held`, a second `thing.pop().held`
  calls again and is still a `T?`, so write `var held = thing.pop().held` and narrow `held` (a local holding a call's
  answer is not a copy). A getter path (`thing.reading`) narrows while `get_reading` only reads, until a call or a
  write may assign what it reads; a getter that may write anything is refused on its second read
  (`'thing.reading' calls 'get_reading' again here, and 'get_reading' may change what it answers, ...: hold it once
  with 'var held = thing.reading' and narrow 'held'`). Comparing needs no narrowing: `target.name == "rat"` needs `target` narrowed, but
  `maybe_name == "rat"` is simply false when it is null.
- A write or a call that may assign an earlier attribute of a proven path undoes the proof whatever it writes
  (`alias.shelf = empty` after `fallback rack.shelf.item`). A write through another object to the last attribute
  undoes it only when the value could break it (`alias.item = null` does, `alias.item = other` does not). A new
  object stays out of this only until it is stored, copied, passed, called on or returned: fill it first.
- After a proof is undone, a `fallback`, `crash` or `if` on the same path is a new proof: it stays in the file,
  tests the value at run time and narrows again. With nothing undoing the proof between, the same check is a repeat
  and the compiler removes it.
- A write or a call in a branch that always leaves (a `return`, a bare `crash`) does not reach what follows the
  branch, so the proof below it stands, a write to the narrowed path itself included (`rack.shelf.item = null`,
  `return`). In a loop no next pass reads it either. A branch that may fall through still undoes it, and inside the
  branch a read after the write is refused.
- A call in a condition counts like one on the line before the next read: after `if rack.item and clear(rack)`,
  `fallback clear(rack)` or `crash rack.item and clear(rack)`, reading `rack.item` is refused when `clear` may write
  it. A side after the call is proven after it: `if clear(rack) and rack.item { rack.item.number }` stands.
- A call between a proof and a read undoes the proof when the called code (followed all the way down) may assign
  an attribute the proof reads or shrink a list it reads (`'items[0]' was proven before line 9, where 'trim(...)' may shrink what it reads, ...`); that holds however the call's answer is used, `fallback items.remove_first()` and `if trim() { }` included. Prove it again after such a call, or read what you need
  before it. A call that cannot change it keeps the proof. Calling a function value reaches every function taken as a value with its signature.
  A call earlier in the same statement counts the same way (`var sum = clear_count(rack) + rack.item.number` is refused; read first, `rack.item.number + clear_count(rack)`, or hold it in a `var` before the call), and so does a getter that writes. A write or call in a `then` does not undo a proof for its `else`.
  A list proof also falls to a shrink through anything that may be the same list: a local (`var alias = items`, `alias.clear()`), a parameter, a call or write that gives the attribute a new list (`shelf.renew()`), and a count kept in a `var` before such a call (`var total = items.count()`, `trim()`, `while index < total` is refused). A list made in the function, or a `copy()`, counts only once it is stored or passed.
- A `T?` number of any width is narrowed before arithmetic, a comparison, a store or a call: `total + pair` with a `Short?` `pair` is an error (`this value may be null (it is a Short?), but an Integer is needed: ...`), and so is `-pair`. A `Short?` widens into an `Integer?` keeping its null. Only `==` and `!=` need no narrowing.
- Every proof the compiler makes (narrowing, proven `[]` reads, a proven divisor, frame objects,
  singleton locks) is listed in [proofs.md](https://github.com/Spite-Language/spite_documentation/blob/master/proofs.md), with when it does **not** apply. Read its "Which proofs
  apply to my code" first: where a proof does not hold, you write the check yourself.
- A `switch` is over a union, a `T?`, an enum or a `Boolean`, and names every member or value: there is no `_:`,
  so a member added later is an error at every switch until it has a case. Cases may share a one-line body
  (`Cat: walk()` and `Dog: walk()`); when all but one share it, write `if value.class == Fish { swim() } else { walk() }`, unless the shared body reads the switched value, which stays a `switch`.
  Value cases are `'red': return 30` or `true:`/`false:`, and a `Boolean` needs both (`this switch over 'count > 2' has no case for false: ...`); for one of them write an `if`. A `switch` over a number or a text is an error: choose with
  `if`s. A case is one statement (a call, an assignment, a `return` or a `crash`), on its line or alone in `{ }`; two
  statements are `this case runs 2 statements: ... move them into a function named for what they do`, and a case is one line, so it opens no `if`, `while` or `switch` with lines inside (`the 'if' this case opens spans several lines: a case is one line, so move the 'if' into a function named for what it does and call it here`). A switch over
  a `T?` needs a `Null:` case (`this switch over 'target' has no case for Null`). Three `if`s in a row that each compare one name with one of its enum values and only return are an error that
  writes the switch for you, naming every value; an `else if` chain of three is the same.
  The subject runs once however many cases there are, whatever it holds (a call, a chain through a call, an index, a getter). A subject holding a call narrows nothing in its cases, since writing it again calls again: `var held = thing.pop().target`, then `switch held`.
- `value.class == Monster` is a class test (false for `null`), and `if value.class == Monster { }` narrows `value` inside.
  A `.class` known while compiling of something holding a call (`thing.pop().class == Thing`, `count().class.name`)
  still makes the call once; only the answer folds.
  In a generic class, `value.class == $wanted_type` tests for the class the codegen value is bound to; `.class` against a
  union or a `type` (written out or bound to `$wanted_type`) is an error, since a value's class is one class.
- A `T?` the compiler proves present is a `T` already: a local every assignment of which cannot be null, an
  attribute of your class whose default and every assignment cannot be null, the answer of your function every
  `return` of which cannot be null. Read it with nothing written; a `fallback`, `crash` or `if` on it is `is already
  narrowed by the proof that every value it can hold is present`. A `T?` parameter and what the library answers are
  narrowed as usual.
- Proving what is already proven is a mistake too: `fallback` on a value that cannot be null here (`so 'fallback' on it
  proved nothing; removed the check`, any local that cannot be null, a copy included), and `crash list[index]` inside `while index < list.count()` (`'list[index]'
  is already proven by the loop condition`). The compiler deletes such a `crash` or `fallback` line, or
  replaces such an `if` with the code it runs, and prints `fixed: ...`; in a `while` condition it stays an error. A
  local nothing reads whose value is a literal or a name is deleted the same way.
- A `Boolean?` is not a condition (`which would only test that it is there, not that it is true`): narrow it
  first, or compare it `== true`. The exceptions are `fallback flag` and `crash flag`, which mean the flag is there
  and `true` (as on a `Boolean`); there is no `flag != null`.
- There are no exceptions and no error values. Three outcomes only:
  - the compiler can know it: a compile error;
  - absence is fine: `fallback condition` ends the function when the condition is false, answering what the answer
    type builds with no arguments: nothing, `null` for a `T?`, `0`, `false`, `""`, an empty `List`/`Dictionary`, or
    the object of a class made without arguments. It is never allowed in a constructor (`'fallback' is not allowed
    in a constructor`: take resolved values, or `crash`). Where the answer type builds nothing (an enum, a class
    whose constructor takes arguments, a singleton) it is an error (`'fallback' returns what the answer type
    builds with no arguments, and a Day has none`): make the result `Day?`, or `crash`. A caller that must tell
    "missing" from a real `0` or `""` gets a function answering a `T?`. An `if` with no `else` that only returns
    that same value (`if x { return null }`, `{ return 0 }`, `{ return "" }`, `{ return }`) or is the last
    statement and only checks a value is there is rewritten by the compiler into the `fallback` (`fixed: ...`). An
    `if` with no `else` that ends in `return` proves the opposite of its condition after it: `if not found { return -1
    }` narrows `found`, `if index >= names.count() { return "" }` proves `names[index]`;
  - absence is a bug: `crash condition` halts with
    `spite: the program stopped at path:line in Class.function, site <id> (name=value, ...)`, followed by the
    fallbacks that failed before it (each with `returned=nothing`, `null` or `empty`, what its function gave back)
    and the call chain as `spite.frame` lines, innermost first. Every name and call the condition read is there with its value, whatever its
    shape: `crash record or cooked.count() > 2` reports `record is null, cooked.count()=1`; a part an `and` or `or`
    skipped is left out. Then come the other texts, numbers and enums in scope (parameters, locals, the object's
    attributes). The condition is not on the line: open the line it names; the position identifies each value, and
    `grep <id> program.crashes` finds a fallback's condition. A bare `crash` marks a branch that cannot happen (`crash false` is formatted to it).
    A failed narrowing names what is absent instead of a value: `clip.keys[start + 9] is missing: index 11990,
    count 11500`, `scores[key] is missing: key "bea"`, `rig.skeleton is null`. Read it as an index past the end or
    a key never set, not as a zero. A `fallback` taken prints nothing as it fails; it shows up only in a crash's
    trace, unless the program is built with `--trace-fallbacks`, which prints each one as it fails.
- Narrowing tests presence, never the value: `crash keys[index]` passes on an element holding `0.0`, and `if count`
  runs on an `Integer?` holding `0`.
- Reading a number from text answers a `T?`: `var read: Integer? = text` is an `Integer?`, `null` for `"forty two"`,
  `"12abc"`, `""` or a number the type cannot hold, so narrow it (`crash count` where the text is yours, `if`/`fallback`
  where it came from outside). `var age: Integer = text` is an error; `var age: Integer? = text` reads it, and a literal is read while compiling
  (`var age: Integer = "42"`; `"forty two"` is a compile error).
- Text is read as an enum value the same way: `var mood: Cat.Mood? = text`, `null` when no value is spelled that
  way, and a literal is decided while compiling (`var mood: Cat.Mood = "calm"`); `var mood: Mood = text` is an error, and `var mood: Mood? = text` reads it.
- A function that declares a result ends every path with a `return` (or a bare `crash`): a path that reaches the
  closing `}` is an error naming it (`'sign_of' answers a String, but when 'value < 0' is false (line 12, an 'if'
  with no 'else') it reaches its end without a 'return'`); in a generic class it names the instance that falls off
  (`but in Marker<Label> it reaches its end`). An `if`/`else` whose branches both return, and a `switch`
  whose cases all return (or `crash`), end the path; an `if` with no `else` and any `while` do not (`while true` is an
  error), so write the last `return` after them, `return null` included.
- A native fault (a null read or an illegal instruction inside a foreign library, a stack overflow) is never
  silent either: the program prints `spite.fault<TAB>kind<TAB>path:line<TAB>Class<TAB>function<TAB>...`, then
  `spite.frame` lines, innermost Spite function first, and ends. `-	-	-` in the place means the fault is outside
  Spite: `at=` names the library and offset, `foreign=` the last C function this thread called, and `from=` the
  Spite line that called it (start there). `stack-overflow` with `repeated=` on a frame is a recursion that never
  stops. The line in `path:line` is where the function starts, not the faulting line
  ([failure.md](https://github.com/Spite-Language/spite_documentation/blob/master/failure.md#what-a-native-fault-reports)).
- A test is a function named `test` or `test_...` that takes nothing and crashes when wrong, in a class of a
  namespace of its own (the entry class's functions are not found). A test program loads the project and runs
  `var tester = Test(Checks)` then `tester.run()`: each class is built with its constructor (the setup) and each test
  runs in a run of its own, printing `pass`/`fail` per test and a count, exiting 1 when any fails; `SPITE_TEST=<Class>.<test>`
  runs one ([testing.md](https://github.com/Spite-Language/spite_documentation/blob/master/testing.md)).

## Metaprogramming

- Every class is an instance of `Spite.Class`. A class's members are declarations (`Spite.FunctionDeclaration`,
  `Spite.AttributeDeclaration`, no value); an instance's are bound (`Spite.Function`, `Spite.Attribute` with
  `.value` and `.owner`). Members are ordinary lists, so "for every attribute" is a walk:
  `Monster.attributes.each(describe)` with `func describe(attribute: Spite.AttributeDeclaration)`, unrolled while compiling into
  one call per attribute, each copy typed by its attribute (a function of another object handed a constant,
  `helper.describe(attribute)`, is compiled once for it too). A member by name is `[]` and answers a `T?`
  (`Runner.functions["run_each"]`); a selection is a member template (`Runner.functions.filter_name_ends_with("_each")`).
  A name is selected, never built from text. A walk sees private `_` attributes and
  reads and writes them through the walked attribute; a serializer skips them, and attributes holding a singleton,
  with `if not attribute.name.starts_with("_") and not attribute.class.is_singleton`, which folds. So do
  `attribute.name == "health"` and `attribute.class == List<Integer>`, which compares the generic arguments too
  (`List<String>` is false; `attribute.class == List` is any list), so the branch not taken is never compiled.
  `attribute.class.satisfies_type(Number)` folds the same way: whether the class can stand in for a type, given by
  name (a `type`, a union, a class, `Anything`, or `Number`, the compiler's own type of a number's operators, which
  every number class `Byte` to `Double` and a program's own number class with them satisfy; `Number` cannot be
  declared or hold a value). Asked of a class known
  only at run time (`value.class` through `Anything`) it is a compile error. A number's, `Boolean`'s or `String`'s `.attributes` is empty.
  A reflection object a walk hands a function has no run-time value: never `append` it to a list or keep it in an
  attribute (an error); keep what you read from it, such as `attribute.name`. A template that would make a function
  the class already has is an error.
- A **template** is a function with a parameter that is a word of its name, typed by what it ranges over:
  `Spite.AttributeDeclaration`, `Spite.FunctionDeclaration` or `Spite.Argument` (the class is never written on the type:
  the body says it). `func show_attribute(attribute: Spite.AttributeDeclaration, label:
  Label)` answers `show_text(label)` and `show_copies(label)`, reading the member as `label[attribute]`, short for
  `label.attributes[attribute]` (a member that is a function taking nothing is called); the class walked is the class of
  the value the body reads it on. `func set_attribute(attribute: Spite.AttributeDeclaration, value: attribute.class) {
  attributes[attribute] = value }` makes `person.age = 2` and `person.name = "x"` work. An exact function
  always wins. A parameter that is not a word of its name makes an ordinary function a walk calls, and a template
  whose only parameter is its word's (`show_function(function: Spite.FunctionDeclaration)`), handed to `each` over a
  class's `.functions` or a function's `.arguments`, known while compiling or only at run time, is that walk's walker.
- A template over **functions**: `func run_phase_each(phase: Spite.FunctionDeclaration, amount: Integer) {
  system.phase_each(amount) }` ranges over the functions of `system`'s class that fit `<phase>_each` (the call in the
  body says the class and the pattern); `run_phases_each(2)` runs it for every one in the order they were made,
  `run_render_each(2)` for one. Inside, `phase.name`, `phase.hole` (the text, a `String`, in place of `phase`) and `phase.index`.
  A name that fits nothing is an error listing the functions that fit. A template over **arguments**: `func
  made_argument(argument: Spite.Argument, start: Integer): argument.class`, called as its plural spread into a call,
  `system.update_each(made_arguments(start))`, once for each argument of `update_each`. A plain walker over arguments
  may leave the word out of its name (`describe_taken`). `Spite.Attribute` and `Spite.Function` (bound to an object, with
  `declaration`, `value`, `owner`) make no template. A folder or an enum is walked as plain code, unrolled while
  compiling: `Shop.classes.each(make)` with `make(part: Spite.Class)`, `Phase.values.each(run)` with `run(phase: Phase)`.
  A `Spite.Class` known only at run time, called with no arguments (`part()`, an attribute `kind()`, `thing.class()`,
  `argument.class()`), makes an object of it typed `Anything`.
- A call built before it runs: `var call = Spite.Call(MoveSystem.functions["run_each"], system)` (a declaration and
  an instance of its class, the only way to build one), `call.arguments["position"] = value`, `call.call()`; known
  while compiling it is the direct call. Known only at run time, `call()` halts on an argument missing, of another
  type, or not taken; `call.arguments["count"] = null` is a filled argument, taken by `count: Integer?` and halting
  for `count: Integer`.
  There is no `call_with`.
- `Whisper.attributes["to"]` names an attribute as a value: a class's `.attributes` is a `Dictionary<String,
  Spite.AttributeDeclaration>`, an object's a `Dictionary<String, Spite.Attribute>` (`.value`, `.declaration`,
  `.owner`), and a literal key on a class named in the code that declares no such attribute is a compile error.
- Questions are get-only attributes, folded on a constant: `Loader.functions["load_each"]` (narrow it), then
  `.is_resumable` (it can reach a wait), `.arguments[1].is_mutated` (it changes what argument 1 is given),
  `.arguments.count()`, `.reads` and `.writes` (the `Spite.AttributeDeclaration`s of its class it reads and writes), `.source_file`
  (the `File` a function or an attribute is written in); `Loader.is_stateful`,
  `Vector3<Float>.is_fixed_size` (a generic class is named with its codegen values), `Monster.is_singleton`. A function that answers nothing `.returns` `Nothing`, never
  `null`; only `.namespace`, `.parent` and a bound attribute's `.value` can be `null`. A reflection field the program never reads costs nothing: it has no slot and is not built, so reading more of a class is what pays. A class's `.generics` is a
  `Dictionary<String, Spite.Class>` of its `generic` lines by name without the `$`
  (`numbers.class.generics["element_type"]` of a `List<Integer>` is `Integer`); there is no `.element_type` on a class. [reflection.md](https://github.com/Spite-Language/spite_documentation/blob/master/reflection.md)
- In a generic class, `if $value_type == List { }` (also `Dictionary`, `Null` for any `T?`, `Enum` for any enum,
  or an exact type) is decided while compiling, and `$value_type.element_type` names what the type holds. It is
  tested the same way (`else if $value_type.element_type == Float`) in any branch of a chain. Only
  what the taken branch reaches is compiled (helper functions, and the code after a chain whose branch returns),
  so keep one generic class with a helper per kind, not one class per kind.
- Names, classes and counts over known classes fold (`Part.name`, `Part.classes.count()`, `Part.classes.map_names().contains("X")`, `Part.classes.first()`, `Part.classes[0]`, `Part.classes["X"]`, `namespace.name_with_namespaces.starts_with("Y.")`, `Spite.Namespace.instances.each(check)`), so a `crash` on them is decided. An item that is not there (`Part.classes["Nope"]`, `[n]` past the end) is read at run time. A name over a class's `.functions` or `.attributes` does not fold: `Part.Axle.functions["spun"]` reads its class's table at run time (a `T?`, narrow it) and `Part.Axle.attributes["turns"]` too (already a `T`); only a missing name folds, a function to `null` (its `if` keeps one branch), an attribute to a compile error.
- `fallback`/`crash` on such a condition folds too. A `crash` that folds to false refuses the build
  (`'crash ...' always halts: its condition is decided while compiling and is false`, naming the instance, as
  `Slot<List<String>>`) when the function reaches it every time it runs and the program reaches the function; one
  under an `if`, `while` or `switch` case only the run decides is left to the run. Write a library's rules as
  `crash $row_type.functions["update_each"]` and a class that breaks one fails the build.
- A getter with no setter makes a read-only attribute: `get_fahrenheit()` answers `.fahrenheit`, and assigning
  it is an error.
- `person.age = 1` calls `set_age(1)` and `person.age` calls `get_age()` when the class has them, with or without
  an attribute named `age` (`get_id()` and `set_id(value)` over a private `_id` make a checked `.id`).
- Operators are functions a class may define: `sum`, `subtract`, `multiply`, `divide`, `remainder`, `equals`,
  `less_than`, `greater_than`, `negate`, `get_at(index)`, `set_at(index, value)`. Without `equals`, `==` compares
  identity. `a[x]` calls `get_at(x)`, which must answer a `T?` (`func get_at(index: Integer): Integer?`, `null` when
  nothing is there); a plain `T` is an error. A read through your own class is narrowed by what you write (`crash
  shelf[0]`): a loop over `shelf.count()` proves nothing about it. The operator is the only way to call its function:
  `a.sum(b)` is an error saying `write 'a + b'`, and `person.get_age()` / `person.set_age(3)` say to write
  `person.age` / `person.age = 3`; inside the class a bare `set_turns(3)` is refused the same way when an attribute or
  local named `turns` exists. Only a function passed as a value (`run_callback(point.get_x)`) is written by name.
  Functions the compiler calls have fixed shapes: `func drop()`, `func to_string(): String`, `func to_debug():
  String`, `func equals(other: ThisClass): Boolean`, `less_than`/`greater_than(other: ThisClass): Boolean`; another
  signature is an error that writes the one it must have.
- Codegen values: `generic $key_type` and `generic $value_type`, one per line at the top of `pair.spite`, and
  `Pair<String, Integer>("a", 1)` supplies them in that order; a codegen value is a type, never a literal
  (`Weapon<Integer, true>` is an error: pass `true` to the constructor, and the compiler splits the class by the
  values its objects are made with and decides the branch in each copy); `Pair("a", 1)` may leave them out when the constructor's
  arguments say them (through `$T`, `$T?`, `List<$T>`, `Dictionary<$K, $T>` or a function value). The constructor
  lists none: `func Pair(key: $key_type, value: $value_type)`. `$` is for generics only: a `$name` with no `generic` line is
  an error. Only a generic class takes a type in `<>`: an enum, a union, a `type`, a number or a codegen value written
  with one is an error (`'Phase' is not a generic class, so it takes no type in '<>': write 'Phase' alone`). The entry
  class is never generic (`'Game' is the entry class and declares 'generic $item_type'`): put what needs `generic`
  lines in a class the entry class makes. A function never has codegen values of its own (`func pick<$value_type>(...)` is an error): a
  function that takes any object takes `Anything` or a `type`. `if $value_type == Integer { }` is decided at compile time. `generic $item_type: Printable` (with the class's own `type Printable`) accepts
  only classes that fit that `type` (a function value never fits; a `T?` fits when its `T` does and the `type` needs
  only `to_string()` or `to_debug()`, which null answers as `null`, so `Shelf<String?>` prints a null item as `null`); any other is an error where the class is
  named, not inside the generic. Only an enum is named outside its class (`Player.Job`): a `union` or `type` belongs to
  the class that declares it, or to its namespace's entry class. One value per `generic` line, at the top of the file
  after any `singleton` line: `generic $a, $b`, a `generics` list and a `generic` line inside a function are errors.
- Settings: reopen `Environment` in the program's `environment.spite` with one `var` per setting and a literal
  default (`var serve = false`), then bind `var environment = Environment()` and read `environment.serve`. The
  value comes from `--serve=true` (or bare `--serve` for a `Boolean`) on the executable's command line,
  kebab-case like the compiler's flags (`game.exe --player-name=x` sets `environment.player_name`; `spite` itself
  refuses it and runs the program with no flags), else the `SERVE` environment variable, else a `.env`
  file loaded with `environment.load_from_file(".env")` (answers `false` when it cannot be read), else the
  default. A value not of the setting's type stops the program with an error naming the setting and exit code 1.
  A setting named like a `Build` field is a compile error naming both. A program given any argument makes
  `Environment()` before its entry starts, so an undeclared flag is refused even where nothing else makes it.
  In a `--hot-reload` build, a save of `environment.spite` that gives a setting a new default reads every setting
  again the same way (the command line it started with, the environment, a loaded `.env`, the new default).
- Build settings: reopen `Build` in `build.spite` the same way. A `Build` field is decided when compiling (`spite --serve=true`, else the `SERVE` variable, else a `SERVE=` line of the `.env` file in the folder built, else its default) and is a constant in the program, so `if build.serve { }`
  keeps only one branch; in a `--hot-reload` build the program's own reads of a field are made while it runs, and
  the standard library's stay constants. The compiler's own options (`optimized`, `debug_memory`, `check`, `build`, ...) and
  `build.target_operating_system` are `Build` fields too; `target_operating_system`, `operating_system` and `program` cannot be declared in `build.spite` (the target is the flag's, the others the compiler's). A flag is kebab-case (`--debug-memory`) and sets the
  snake_case field; an unknown flag is an error. A `Build` default may be arithmetic on literals or another field
  (`var attempts: Integer = retries`); the compiler's own options take only a literal. There is no `format` option: every compile formats.
- Reflection: `value.class` (a `Spite.Class`: `.name` (always the short name, however the class was reached; `.name_with_namespaces` is the dotted one), `.namespace` (a `Spite.Namespace?`; narrow it before
  reading its members: `fallback value.class.namespace` narrows the path itself and its prefixes for the rest of
  the block; its members are `.name_with_namespaces`, `.parent`, `.classes`,
  `.namespaces`), `.functions` (empty for the entry class, which is never made twice), and `.enums`, `.unions` (each with `.members`) and `.types`, all keyed by name),
  `value.attributes`
  (`.name`, `.class`, `.value`: the value itself, an `Anything?` whose text is `.value.to_string()`), `value.functions` (`.name`, `.declaration` with its `.arguments` and `.returns`, `call_function()` for
  functions that take nothing, a returned value dropped; on one that takes arguments it halts), a function named without calling it (`shouter.shout`, a
  `Spite.Function<String, String>` bound to `shouter`, called as `change(text)`), and
  `Spite.Class.instances` (every class of the program and the packages it loads, not the standard library's). `class`, bare inside a class's function, is the class
  of the instance it answers on, and a class name reads its own class object: `Monster.name` is `"Monster"`.
  A bound `Spite.Function` answers only `.name`, `.owner`, `.declaration` and `call_function()`: ask `.arguments`,
  `.returns`, `.reads` and the rest of `function.declaration` (`a Spite.Function has no 'returns' of its own`). A walk
  over `Monster.attributes` hands `Spite.AttributeDeclaration`s and a walk over `troll.attributes` hands
  `Spite.Attribute`s; a parameter of the other kind is an error. An object literal's `.class` is `Object`.
- A class whose file starts with a `singleton` line has one instance: `Journal()` always returns it. A
  singleton whose constructor takes arguments has one instance per literal argument list: `Channel(1)` is the same
  object everywhere, `Channel(2)` a second one, and a non-literal argument is an error. A singleton is bound as an attribute (`var journal = Journal()` beside the
  others) and used through the name. `Journal().record(entry)`, `Build().program`, `keep(Console())` and
  `return Console()` are errors (`'Console' is a singleton: bind it once beside the attributes`), and so are a
  binding nothing reads, even in a loaded package, and a binding inside a function (`'Build' is a singleton, bound
  here as a local`), except in a function added to a value class (`String`, a number, `List`, `Dictionary`), which
  binds it as a local `var`, made the first time that line runs. Write no lock for one: work the compiler runs at once never
  shares a singleton one of them writes. Singletons whose attributes make each other, even through an ordinary object
  (`World` binds `Column<Entity>`, which makes an `Entity`, which binds `World`), are an error naming the circle
  (`singletons initialise each other in a circle`): bind one of them in the class that uses both instead.
- Memory and threads belong to the compiler: a program never names an address, an allocator, an allocation count,
  where an object lives, or the live instances of a class. A class that needs room for bytes holds a `Buffer`; a
  container of your own is written over a `List` or a `Buffer`. Only `--debug-memory` reports leaks, from outside the
  program. How a text is held is the compiler's too: a short text a function keeps to itself allocates nothing, a
  literal is made once in the program's data, `text.split(separator).contains(word)` makes no list, and an
  interpolation only compared (`key == "r{index}"`, `line.contains("r{index} ")`) makes no text, so write the plain
  join or split and never reshape code to make fewer texts.

## Built in classes

`Console()` (`print`, `write`, `error`, `debug`, `read_line(): String?`; each value printed is its `to_string()`, so
a class prints once it declares `func to_string(): String`, and `debug` shows any value's state, a class as
`Name { attribute: value }`, through the `to_debug()` every value has), `File(path)` (a path is a `Path`, and text casts into it wherever one is wanted: a literal needs nothing, `File("data.txt")`, and `File("")` is a compile error; computed text is a `Path?` narrowed first, `var level: Path? = "{folder}/forest.txt"`, `crash level`, `File(level)`, since empty text and text holding a zero byte are not paths; `Directory`, `File` and `Process.working_directory` take one, and `path.name`, `extension`, `parent`, `is_rooted` and `joined(other)` ask it; `GitPath` is a pinned place in a repository, kept apart so it cannot reach `File`: `var engine: GitPath = "github.com/example/engine@a3f2c91/core"`, with `repository`, `commit`, `folder` (empty for none), `is_local`, `is_remote`, `is_a_url`, and `null` for a commit that is not 7 to 40 lowercase hexadecimal digits, no repository, or a folder outside it; `file_name`, `read_as_string(): String?`, `write_string(text)`,
`append_string(text)`, `byte_count()`, `last_written()`, `exists`, `remove`, `move_to(path)`: renames or moves, `false` when something is already there; bytes are a `Buffer`: `read_as_buffer(): Buffer?` (the whole file; read or mapped is the compiler's choice), `read_buffer_at(position, count): Buffer?` for a file too big for one buffer, `write_buffer(buffer)`, `append_buffer(buffer): Long?` where they start), `Directory(path)` (`path`, `folder_name`, `entries(): List<Directory.Entry>?` (`null` when the folder cannot be opened; each entry a `Directory` or a `File`, switched on;
one kind is `entries.filter_files()` or `filter_directories()`, never a helper of its own), `exists`, `create`, `move_to(path)`),
`Process(command, arguments)` (`run(): Integer?`, `null` when the command could not start; `read_output_as_string(): String?` or `read_output_as_buffer(): Buffer?`, `null` before a run: standard output only, `with_errors = true` adds the error stream, `keeps_errors_apart = true` reads it apart through `read_errors_as_string()`; `time_limit = Duration(...)` stops the child past it, `run()` answering `null` and `was_stopped` true; each argument reaches the child whole, `-key=value with spaces` as `-key="value with spaces"` on Windows; `working_directory` and `environment_variables["NAME"] = "value"` set for the child alone), `Program()` (`exit(code)`, `sleep(duration)`,
`environment(name): String?`, `executable_path(): String?`, `working_directory(): String?`). The library halts only on the program's own mistakes (an index outside a text, a negative count); what the outside world does is an answer (`T?` or `Boolean`): a file, a folder, a lock, a process, a line of input that is not UTF-8, the system refusing a call, random bytes not given (`SecureRandom().random_buffer(count): Buffer?`), the machine's zone (`zones.system(): TimeZone?`), and the program decides with `fallback` or `crash`. `Console` is a singleton: `Console()` is the same instance everywhere, bound once
as `var console = Console()`.
`print`, `error` and `debug` write their line out at once, so a log redirected to a file shows every line as it
happens; `write` waits for the next line end, the next `read_line()` or the exit. There is no `flush()`: lines from
two threads never split each other, and everything is out before an exit, a crash report or a read.
An address is an `IpAddress` (IPv4 or IPv6 only, the port a number beside it): `var read: IpAddress = "::1"` (a literal address is read while compiling)
reads one from text or resolves a host name (`var read: IpAddress? = "example.com"`, its first address, IPv4 first, keeping
the name for HTTP's `host`; `null` when neither), `"example.com".resolve_ip_addresses(): List<IpAddress>` gives every
address (IPv4 first), `to_string()` writes it as RFC 5952 does, `==` compares; no constructor and no network call
takes text, so a program writes the literal (`var read: IpAddress = "127.0.0.1"`) or narrows a read `IpAddress?` and passes the `IpAddress`.
Text has no `to_integer()` or `to_float()`, and there is no `to_ip_address()`: each is a conversion,
`var count: Integer? = text`, `var address: IpAddress? = text`. `split("")` splits text into characters. `snake_case()`,
`camel_case()` and `pascal_case()` respell a text's words (`"total_count".camel_case()` is `totalCount`, `.pascal_case()`
`TotalCount`, `"TotalCount".snake_case()` `total_count`); words end at `_`, `-`, `.` and spaces (dropped, so `"_kit"`
is `kit`), before a capital after a small letter or digit, and before the last capital of a run a small letter follows
(`"HTTPHelper"` is `http_helper`, `httpHelper`, `HttpHelper`); a digit never starts a word (`vector_2d` is `Vector2d`).
Reflection has no casing of its own: `attribute.name` is the name as written, and `attribute.name.camel_case()` in a
walk is worked out while compiling.
TCP is `Tcp.Server()` (`listen_locally(port)`, `listen_everywhere(port)`, `listen_at(address, port)`,
`accept(): Tcp.Connection?`) and `Tcp.Client()` (`connect(address, port)`, `connect_locally(port)`); a client and a `Tcp.Connection` talk alike: lines
(`read_line(): String?`, `write_line(text)`), text (`read_as_string(): String?`, `null` for bytes that are not UTF-8, waiting only for the rest of a character the network cut at the end; `write_string(text)`) or bytes as a
`Buffer` (`read_as_buffer(): Buffer?`, a fresh buffer of what arrived, `write_buffer(buffer)`). Every read waits:
there is no `_now` read. A peer that hung up is not an error: `closed` turns `true`, reads answer `null` and
writes send nothing.
**Bytes are a `Buffer`**, never a `List<Byte>`: the type is a compile error written or made (a literal of `Byte`
items, a call answering one such as `map_<plural>` over a `Byte` attribute or `keys()` of a `Dictionary<Byte, ...>`,
a generic class of yours bound to `Byte` that keeps a list of it), and a list has no byte functions (calling one is a compile
error naming `Buffer`). A `Buffer` is still a run of bytes (`count()`, `is_empty()`, `buffer[index]`, `append(byte)`, `truncate(count)`,
`clear()`, `copy()`; it has no `each` or other list templates, so walk it with a `while` and an index) and adds:
made with a size, `Buffer(size)` (`Buffer(0)` when nothing is known; the size is room, never a limit; a foreign
function that writes into it is passed `Buffer(4096)`, then `take_written(count)` says how much it wrote),
a byte order named in every number of two bytes or more (`_little_endian` or `_big_endian` after the number, like
`read_integer_big_endian_at(position)`; `byte` and `tiny` have none),
no reading position: every read is `read_<number>_at(position): <number>?` and a walk keeps its own `var at = 0`
(`tiny`, `short`, `unsigned_short`, `integer`, `unsigned_integer`, `long`,
`unsigned_long`, `float`, `double`, each with its order; one byte is `buffer[position]`), `append_<number>(value)`,
`write_<number>_at(position, value)` (halts outside), `read_buffer_at(position, count): Buffer?`,
`append_buffer(other, start, count)`, `read_string_at(position, count): String?` and `read_as_string(): String?`. Prove `_at` reads
instead of narrowing each: `crash body + 13 <= bytes.count()` proves every read inside those 13 bytes, a loop `while position + 15 <
bytes.count()` stepping `position = position + 16` reads each record with no test, and `crash height * stride <=
pixels.count()` proves `pixels[row * stride + column]` inside `while row < height` and `while column < stride`.
Memory a C library hands out (a mapped GPU buffer) is `ForeignBytes(address, count)`, with the same `_at` reads and
writes, orders named, bounds checked. A program never reads or writes bytes at an address: a `File`,
a `Tcp.Connection` and a `BinaryReader` take and answer a `Buffer`.
**Bytes that may be text are one holder with two reads**: `read_as_string(): String?` (`null` when not UTF-8) and
`read_as_buffer()`, on a `File`, a `Tcp.Connection`, a `WebSocket.Message`, a datagram, an HTTP body and a `Process`'s
output, so nothing that carries bytes forces them through a `String`.
`Udp.Server()` opens on a known port with `open_locally(port)`, `open_everywhere(port)`, or `open_at(address, port)`, and `Udp.Client()` on a port the system picks with `open()`; both send one `Buffer` datagram per `send_to(address, port, buffer)` and receive one `Udp.Datagram?` per `receive()` (`read_as_string()`, `read_as_buffer()`, `sender_address: IpAddress`, `sender_port`); `Http.Server` (`next_request(): Http.Request?`, `respond(request, response)`) and `Http.Client` (`send(address, port, request): Http.Response?`) speak HTTP/1.1 and keep connections alive on their own; a body is bytes, `read_body_as_string(): String?`, `read_body_as_buffer()`, `write_body_string(text)`, `write_body_buffer(buffer)`.
WebSocket (RFC 6455, `ws`) is a namespace: a server is `var sockets = WebSocket.Server(http)` over the program's own
`Http.Server`, like an Express app; in the program's HTTP loop, `if sockets.asks_to_upgrade(request)` then
`sockets.accept(request): WebSocket.Connection?` (`null` after it refused a malformed upgrade itself), and every other
request is answered with `http.respond`. A client is `WebSocket.Client()` with `connect(address, port, request): Boolean`.
Both sides have `send_string(text)`, `send_buffer(buffer)`, `send_ping(buffer)` and `receive(): WebSocket.Message?`,
whose `is_text()`, `read_as_string(): String?` and `read_as_buffer()` read it. Pings are answered by itself and the latest pong is `last_pong`. `null` from `receive()`
means the connection ended: `close_code` says how (`1000` a normal close, `1002` a broken frame, `1006` no close at
all), so a protocol violation is never silent. `close()` sends the close and `receive()` keeps answering until the
peer's close arrives.
A program never says what runs at once: call the function itself. The compiler runs independent calls in a row at once (only `var name = attribute.function(numbers)` declarations
whose functions only compute a number; blocking IO runs on the pool's helper threads),
a loop over a list of different classes at once, an `each_<member>()` walk in bands when it proves each pass
touches only its own element and what it made, and a `while index < list.count()` loop that takes `list[index]`
first and steps `index` last in bands when each pass writes only its item, what the item alone reaches, what it
made and its own slot (checked held once when it runs); what it runs at once prints what running in order prints, and the first crash stops
the program. There is no lock, atomic or thread-local class to name: keep per-thread data in locals and shared data
in singletons. There is no `async`/`await`: a function
that reads, sleeps or waits is an ordinary function, and the compiler suspends it there when something else can
run ([concurrency.md](https://github.com/Spite-Language/spite_documentation/blob/master/concurrency.md)), wherever the
wait is written (an `and`, a function value, a constructor), keeping the written order: in `count + bump()`,
`count` is read before `bump()` waits.
`$system_type.functions["update_each"].is_resumable` is decided while compiling (and `phase.is_resumable` for each
function a walk visits): `true` when the function can reach a wait. A system never says that it does IO. A wait in a frame does not hold the frame: in a `while` that sleeps on a
`Program` (itself or through its own class's functions), and in its own class's functions it calls, a statement
`report.fetch()` or `systems[index].update()` (no arguments, answering nothing, on another object of the program)
whose function can wait is started by the compiler as a state machine of its own and finishes in a later frame,
eight per statement at most, all of them before the program ends. Write the IO plainly in the system's own class.
`update.arguments[1].is_mutated` (with `var update = $system_type.functions["last_each"]` narrowed by `crash update`)
is decided while compiling the same way: `true` when the function, or anything it calls, changes what its argument
number 1 (from 0) is given or anything reached through it; an engine writes `crash not update.arguments[1].is_mutated`
to make a system that breaks its rule a compile error
([metaprogramming.md](https://github.com/Spite-Language/spite_documentation/blob/master/metaprogramming.md#asking-a-question-while-compiling)).
`JsonWriter<T>()` made once, then `write(value): String`, writes JSON, and `JsonReader<T>()`, then
`read(text): T?`, reads it (there is no `Json` class; `read` is `null` on text that is not one whole value, `{} x`
or a missing brace) (`crash` it when bad text is a bug), for any class, list, dictionary, enum, number, `Boolean`, `String` or `T?`;
`read` skips unknown keys, keeps defaults for missing ones, reads a camelCase or PascalCase key (`buyPrice`,
`BuyPrice`) into its snake_case attribute (`buy_price`) when no attribute has the key's exact name, so never
rename attributes to match foreign JSON, and is `null` on a value of the wrong kind (`3.7` for an `Integer`
too); an enum reads back only as one of its values' names. A read class needs a constructor without parameters
(the reader fills it) or a conversion such as `func Version(from: String): Version?` (the reader converts what it
reads); `BinaryReader` needs the constructor. A key that is not an attribute's name goes in the serializer's
`keys`, a map keyed by the attribute: a literal
(`writer.keys = {Order.attributes["buy_price"]: "price"}`) is folded into the code, a map made at run time (in a
loop, from a function) is copied when set and turned into a table by the compiler, and a map that is mixed,
changed, held or passed on is read as the program runs (never an error); a run-time map keying anything but a public attribute holding a value halts. Writing a `Float` or `Double` that is infinity or not-a-number
to JSON crashes showing the value and the attribute (`shown=inf	path=Order.price`): check the number first
if `null` is wanted. **Between Spite programs, and for files a Spite program reads back, use bytes instead**:
`BinaryWriter<T>()` with `write(value): Buffer` and
`BinaryReader<T>()` with `read(bytes): T?`, `null` on bytes that are not exactly one `T`; many values are one
value, a `List<T>` or a class holding them; both are made once and reused;
a quarter of JSON's size and more than ten times faster, with no keys, so both ends must be built from the same
classes. A socket's or a file's bytes are a `Buffer` already, read the same way. The pair is a typed serializer, the binary twin of the JSON one: whole values in, whole values out; single numbers at positions are the `Buffer`'s own reads. To catch two ends built from
different classes, write `writer.schema()` (a `Long` the compiler works out from the classes, free to ask) once at
the start of a file or connection and compare it with `reader.schema()` before reading. Attributes named `_...`
and attributes holding a singleton are left out of both. A JSON key that is not an attribute's name comes from a map
given to the serializer, keyed by the attribute (`{MeshRecord.attributes["minimum_level_of_detail"]: "min_lod"}`);
a class never declares `json_key_<attribute>()`, which is an error. A union, a `type` (`Anything` included) or a function value anywhere in what they see is a compile error at
the line that makes the writer or reader (`JsonWriter cannot write 'Owner': 'Owner.pet' is the union Pet, ...`):
keep what they see to the kinds above ([json.md](https://github.com/Spite-Language/spite_documentation/blob/master/json.md)).
Measure with `clock.elapsed()`, the monotonic clock: a `Duration`; subtract two readings (`clock.elapsed() - started`). Every wait is a `Duration` too: `var frame_time = Duration(16, 'milliseconds')`, then `program.sleep(frame_time)`.
To time one function, `var benchmark = Benchmark(work)` calls `work` (no arguments, passed by name) once between two
readings: `benchmark.answer` is what it answered (a `Benchmark<Nothing>` for one that answers nothing) and
`benchmark.duration` a `Duration` (`benchmark.duration.total('microseconds')`). There is no `clock.benchmark(...)`.
Time is stored as an `Instant` and nothing else: `clock.now()`, or `Instant(since_1970)` with
`var since_1970 = Duration(1710054000, 'seconds')`.
`Duration(90, 'minutes')` is exact time (no days: `Duration(1, 'days')` is an error); `Period(1, 'months')` is
calendar time, added to a `Date` or a `DateTime` and never to an `Instant`, and never compared with `<`;
`Date(year, month, day)` is a calendar date, `Time(hour, minute, second, nanosecond)` a clock reading and
`DateTime(date, time)` both, none of them an instant. A zone only shows or
reads a local reading: `var zones = TimeZones()`, `zones.find("America/New_York"): TimeZone?`, `zones.utc()`,
`zones.fixed_offset(duration)`, then `instant.to_date_time_in(zone)`, `instant.to_string_in(zone)` and
`local.to_instant_in(zone, 'compatible')` (or `'earlier'`/`'later'`, always written); `zone.identifier` is its
IANA identifier. Text reads ISO 8601 through each type's conversion (`var read: Instant? = "2024-03-10T07:00:00Z"`,
`Date?`, `Duration?`, ...) and `to_string()` writes it. `Date(2023, 2, 29)` halts, so read outside text
with `var read: Date? = text`, which answers `null`, as a leap second and a bracketed zone that disagrees with the offset
do ([time.md](https://github.com/Spite-Language/spite_documentation/blob/master/time.md)).
`DynamicLibrary("ucrtbase.dll", 'identity', "")` calls a native library's functions as members
(`c_runtime.strlen(text)`, `_as_long`/`_as_double`/`_as_text` for wider results; one instance per file, naming rule and header, a singleton like any other; a `void` function named by the header is called on a line of its own, and reading its answer is a compile error); the standard library's
`library/windows/`, `linux/` and `mac/` folders reopen the classes each system changes (docs/foreign_libraries.md).
C calls back into Spite without any C written: a function value given as a foreign call's argument is a C function
pointer for that call only (`c_runtime.qsort(values, count, 4, ascending)`); a function C keeps or calls from its
own thread is handed over as `var handler = ForeignCallback(owner.function, 'context_last')` (or `'context_first'`),
passing `handler.address` as the pointer and `handler.context` as C's user data, or `'no_context'` for a function
of a singleton when C gives none (a window procedure; the singleton tells objects apart by the handle C passes). A
callback takes and returns only numbers, `Boolean` (a 32-bit `BOOL`) and addresses as a `Long`; a callback cannot take text
yet, so text C passes arrives as its address, a `Long`, kept or handed back to C (a `String` parameter is a compile error). Keep the `ForeignCallback` in an attribute of the object that owns the C
registration and unregister in its `drop()`: C may call only while it is kept, and a call after the drop halts
naming the function. A handed-over function of an ordinary object
may reach only its own value attributes, its locals and singletons. A COM-style table of functions is filled on the C side: hand the library each `'no_context'` `ForeignCallback`'s
`address` (`mixer.set_voice_started(started.address)`); a program cannot write memory at an address. A C constant
is read as its name in snake_case (`user32.mouseeventf_leftdown`) and needs a header as the third argument. A
struct's size is `PointPair.size`, a `Long`, only on a number-only `type`, whose layout and padding you declare in
C's order; with a readable header, a `type` passed where the prototype takes a pointer to a struct the header declares must have that struct's size, or the call is a compile error naming both sizes. A `Buffer` crosses by address too.
Game maths (docs/game_maths.md): `Vector2`, `Vector3`, `Vector4` are generic over a number class taken from their
parts, `Vector3(1.0, 2.0, 3.0)` a `Vector3<Float>` and `Vector2(5, 1)` a `Vector2<Integer>` (never `Vector3i`), written
`Vector3<Float>` where a type is named; every member answers the vector's own class (whole numbers truncate).
Read as `x`, `y`, `z`, `w`, with `+ - * /` part by part, `scaled(factor)`,
`dot`, `cross`, `length()`, `normalized()`, `distance_to`, `linear_interpolate(target, amount)`. `Matrix4<Float>()` and
`Quaternion<Float>()` (or `<Double>`, never a whole-number class; always written, nothing infers it) are the identity and are set in place: `matrix.set_transform(translation, rotation, scale)` (the one-value setters are assignments: `matrix.translation =
offset`, `matrix.scale = factor`, `matrix.rotation = rotation`),
`set_perspective(field_of_view, aspect, near, far)` (Vulkan: y down, depth 0 to 1), `set_look_at`,
`rotation.set_axis_angle(axis, angle)`, `set_euler(angles, 'xyz')`; `a * b` applies `b` first;
`matrix.transform_point(point)`, `rotation.rotate(vector)`, `inverse()` a `Matrix4?`. Matrices are column-major,
parts `column_0_row_0` to `column_3_row_3`. Each answer is a new object, so one step per line:
`var moved = velocity.scaled(delta)` then `position = position + moved`. Culling and picking: `AxisAlignedBox(lowest,
highest)`, `Plane()` with `set_point_normal`, `Frustum()` set with `frustum.from_view_projection = matrix` then
`intersects_box`/`intersects_sphere`, and `Ray(origin, direction)` whose `hit_plane`, `hit_box` and `hit_triangle`
answer a `Float?` distance, `null` for a miss. Colours: `Color(0.2, 0.5, 0.9, 1.0)` holds `red`, `green`, `blue`, `alpha` from 0 to 1, its one
constructor; text becomes one only through a conversion, `var read: Color? = text`, a `Color?` (a literal needs no narrowing, `var read: Color = "#ff6347"`) (also `rgb()`, `rgba()`, `hsl()`,
`hsla()` and CSS names; `null` for text naming none, so `crash` it), written with `to_hexadecimal_string()`, `to_rgb_string()`, `to_hsl_string()` (each clamps every channel, alpha too, to 0 to 1), converted with
`to_linear()` and `to_standard_rgb()`. Animation and shading: `CubicBezier(start, first_handle, second_handle, end)`
with `y_at_x(x)`, `var easing = Easing()` then `easing.in_out_cubic(amount)` and the rest, `Noise(seed)` with
`gradient_2d(x, y)` and `interleaved_gradient`, and `value.to_half_precision()` / `bits.half_precision_to_float()`
for 16-bit floats. A number's raw bits are `Float.bits(): UnsignedInteger` and `Double.bits(): Long`, and back
`UnsignedInteger.bits_as_float()` and `Long`'s or `UnsignedLong`'s `bits_as_double()`, and the same width with the
other signedness is `bits_as_unsigned()` on a signed number and `bits_as_signed()` on an unsigned one (storing a
negative number in an unsigned type halts); they cost nothing (a C union), so use them rather than writing a value
into memory to read its bytes.
Every class here, the numbers and `List` included, is a Spite file in `library/`, and a program's own file of the
same name reopens it: `list.spite` adds a member template (`dictionary.spite` one over a dictionary's values,
`member: Spite.AttributeDeclaration`, read on a `$value_type`), `integer.spite` a function on every `Integer`.

A statement is one line: anything after it on the same line (after a `return` value, a call, or the `}` that
closes an `if`, `while` or `switch`) is a parse error (`'attribute' is left over after the end of the statement`),
never a second statement. `return` converts to the function's type, so a local declared only to be returned on
the next line (`var whole: Integer = scaled.round()` then `return whole`) is an error: write
`return scaled.round()`.

## Bindings: a library from another language

Bind a mature library (compression, a database, a codec, physics, a graphics API) instead of rewriting it; rewrite
only what is small or what the program is about. Anything with a C ABI binds the same way: C as it is, C++ functions
under `extern "C"` (a method becomes a function taking the object's address), Rust `#[no_mangle] pub extern "C" fn`
in a `cdylib`, Zig `export fn` in a dynamic library, Go `//export` with `-buildmode=c-shared`.

- One class wraps the library and is the only code that touches its `DynamicLibrary`. Only Spite shapes leave it:
  full-word snake_case names, `T?` instead of a null handle or a "not found" code, an object that holds the handle and
  releases it in `drop()`, `String` for text.
- A C status or enum becomes a Spite enum the binding writes, with Spite names, each value mapped to its C number;
  the mapping is `if`s on the number ending in a bare `crash`, so a number it does not list halts at the boundary:

  ```gdscript
  func packing_of(code: Integer): Packing {
      if code == 0 {
          return 'packed'
      }
      if code == 1 {
          return 'output_too_small'
      }
      crash
  }
  ```

- What crosses: numbers, `Boolean`, enum values, text, lists of numbers and number-only `type`s by address, a handle
  as a `Long`, a function as a `ForeignCallback`. What cannot: a class instance, exceptions and panics (catch them on
  the library's side and answer a status), generics (export one function per type), a struct returned by value, a
  union past its first member, varargs, function-like macros, and memory another runtime owns (give it back through
  the library's own function, in `drop()`).
  [foreign_libraries.md](https://github.com/Spite-Language/spite_documentation/blob/master/foreign_libraries.md#libraries-written-in-other-languages)

## Habits from other languages that Spite rejects

Almost every one of these is a compile error naming the Spite form (the middle column is the start of what the
compiler says), so it will not pass silently; but each costs a round trip, and this table is cheaper to read than
to rediscover. The rows marked *silent* compile, and do something you did not mean.

| Written elsewhere | The compiler says | Written in Spite |
|---|---|---|
| `days / 146097`, `x * 3.14159` inside a function | `the number 146097 does not say what it means: declare a private attribute that names it, like 'var _what_it_means = 146097', and write the name here` / `the number 3.14159 is Double.Pi() or Float.Pi()` | `var _days_in_400_years = 146097` beside the attributes, then `days / _days_in_400_years`; `Float.Pi()` (a whole number of three or more digits that is not a power of ten or two, or a decimal that is a preset, D1777) |
| `a && b`, `a \|\| b`, `!a` | `Spite writes 'and' and 'or' as words` / `Spite writes 'not' as a word` | `a and b`, `a or b`, `not a` |
| `a << 3`, `a >> 3`, `a & mask`, `a \| mask`, `a ^ mask`, `~a` | `Spite has no '<<': bits are functions on the whole numbers` (each names its function) | `a.shifted_left(3)`, `a.shifted_right(3)`, `a.bits_and(mask)`, `a.bits_or(mask)`, `a.bits_exclusive_or(mask)`, `a.bits_inverted()` |
| `x.sqrt()`, `x.sin()`, `x.atan2(y)`, `x.pow(y)`, `x.abs()`, `a.min(b)`, `x.ceil()`, `x.isnan()`, ... | `Float has no function 'sqrt': Spite spells it 'square_root', since no name is abbreviated` | every name in full: `square_root()`, `sine()`, `cosine()`, `tangent()`, `arc_sine()`, `arc_cosine()`, `arc_tangent()`, `power(y)`, `exponential()`, `logarithm()`, `logarithm_base_2()`, `logarithm_base_10()`, `absolute()`, `minimum(b)`, `maximum(b)`, `ceiling()`, `truncate()`, `is_not_a_number()`, `is_infinite()`, `is_finite()`; `floor()`, `round()` and `clamp(low, high)` keep their names |
| `Math.atan2(y, x)`, `atan2f(y, x)` | `'Math' is not a local, an attribute or a function of ...` | `y.arc_tangent_over(x)`: the angle of the point `(x, y)` |
| `Math.sqrt(x)`, `Math.PI` | `'Math' is not a local, an attribute or a function of ...` | `x.square_root()`, `Float.Pi()` |
| `sqrt(x)`, `sqrtf(x)`, `pow(x, y)` | `this class has no function 'sqrt'` | `x.square_root()`, `x.power(y)` |
| `M_PI`, `f32::consts::PI`, `FLT_MAX`, `INT_MAX`, `INFINITY`, `NAN` | `'M_PI' is not a local, an attribute or a function of ...` | `Float.Pi()` (`Double.Pi()` for the 64-bit one), `Float.Largest()`, `Integer.Largest()`, `Float.Infinity()`, `Float.NotANumber()` |
| `angle.pi`, `Float.pi()` | `'pi' is a constant of the class Float, not of a value` | `Float.Pi()` |
| `count++`, `count += 1` | `Spite has no '++': write 'count = count + 1'` | `count = count + 1` |
| `condition ? a : b` | `Spite has no 'condition ? a : b'` | an `if` with an `else`, or a function that returns one or the other |
| `int`, `Int`, `bool`, `Bool` | `'Int' is spelled 'Integer'`, `'int' abbreviates 'int', which could mean 'integer' or 'internal'` | `Integer`, `Boolean`: no name is abbreviated, the language's own included |
| `x.to_int()`, `int(text)`, `Integer.parse(text)`, `Integer(text)` | `an Integer has no attribute or function 'to_int'` / `'Integer(text)' calls a conversion by name, and the compiler applies a conversion only where a value meets the class it makes` | `var number: Integer? = text`, then narrow it; `var wide: Float = count` for a number that fits |
| `LocalDate`, `PlainDate`, `NaiveDate` | `'LocalDate' is spelled 'Date'` | `Date`, `Time`, `DateTime`: a calendar reading with no zone |
| `let x = 1`, `const x = 1` | `Spite has no 'let': a variable is declared 'var name = value'` | `var value = 1`: every `var` has a default, and there is no `const` |
| `var total: Integer` | `expected '=' (every variable needs a default value)` | `var total = 0` |
| `new Monster()` | `there is no 'new' in Spite` | `Monster()` |
| `this.name`, `self.name`, `this.helper()` | `a class reads its own attributes by name: write 'count', not 'this.count'` | `name`, `helper()`; `this` only passes the object itself (`registry.append(this)`) |
| `import`, `load("folder")` | `there are no imports`; `'load' is a keyword, not a function` | `load "folder"`, a keyword on its own line inside a function |
| `elif` | `Spite writes 'else if'` | `else if` |
| `class Monster { }` | `there is no 'class' keyword: a file is a class` | nothing: the file *is* the class |
| `func greet(name: String = "world")` | `a parameter is declared 'name: Type', and found '='` | a second function, or an attribute holding the value |
| `func one() { return 1 }` | `'one' answers nothing, so 'return' takes no value` | `func one(): Integer { return 1 }` |
| `print(value)` | `this class has no function 'print'` | `var console = Console()` beside the attributes, then `console.print(value)` |
| `Console().print(value)`, `Build().program` | `'Console' is a singleton: bind it once beside the attributes` | `var console = Console()`, `var build = Build()`, then `console.print(value)`, `build.program` |
| `Report(text)` as a statement, to "do" something | `'Report(text)' makes a 'Report' and drops it` | a function (`report(text)`) on the class that needs it |
| `toString()`, `__str__`, `Display` | (nothing calls it) | `func to_string(): String` in the class, which `console.print` and text holes call |
| `console.log(object)`, `dbg!`, `__repr__`, `{:?}` | | `console.debug(value)`: every value has `to_debug()`, and a class may declare its own |
| `"hello " + name` | `text written down is not joined with '+'` | `"hello {name}"`; two values still join with `+` |
| `label.text = "{clicks}"`, to turn a value into text | `'"{clicks}"' is a text of one value and nothing else: assign the value directly, 'label.text = clicks'` | `label.text = clicks`: a value casts to text where text is wanted; `clicks.to_string()` elsewhere |
| `"hello ${name}"` | *silent*: prints `hello $` and the name | `"hello {name}"` |
| `'hello'` for text | `'hello' in single quotes is an enum value literal, and text is wanted here: text is always written in double quotes` | `"hello"`: single quotes are enum values only |
| `flags.bits_and(mask)` with a wider `mask` | `'flags.bits_and(mask)' works in a Byte, since a bitwise function takes its receiver's type` | widen the receiver first: `var wide: Long = flags` |
| `var held: $held_type = null` | `the attribute 'held' is declared '$held_type', not '$held_type?', so it never holds null: ...` | `var held: $held_type` with no default when the constructor gives it its value; `$held_type?` for none yet |
| `var target: Pet` the constructor fills on one side of an `if` only | `the attribute 'target' has no default, so the constructor gives it its value, but the 'if' at line 5 gives it one on one side only: ...` | give it a value on every path, or write a default |
| `for item in list` | `Spite only has 'while' loops; there is no 'for'` | `each_<member>()`, `map_`/`filter_<member>()`, or `list.each(function)` with a function of yours; `while index < list.count()` when the body needs more |
| `break`, `continue` | `Spite has no 'break': a loop stops in its own condition` | the loop's own condition: `while index < count and not found` |
| `list.add(x)`, `list.pop()` | `List has no method 'add', which does not say where` | `append`/`prepend`, `remove_last()`/`remove_first()` |
| `Heap<Node>`, `&Node` | `there is no 'Heap<T>'`, `Spite has no '&Type'` | `Node`: every class is a reference already |
| `if value do name { }` | `Spite has no 'do'` | `if value { }`: the name itself is narrowed inside |
| `items.map(item => item * 2)` | `expected a value and found '='` | no lambdas and no `map(function)`: give the element's class a read-only attribute, `get_double()`, and write `items.map_doubles()` (the member in the plural) |
| `value == null`, `value != null` | `'null' is not a value to compare against or pass around` | `if value { } else { }`, `fallback value`, `crash value`, or `switch` |
| `Parallel(...)`, `task.wait()`, `await task` | `'Parallel' is not part of Spite: the compiler decides what runs at once, so call the function itself` | call the function itself |
| `new Date()`, `DateTime.Now`, `datetime.now()` | | `clock.now()`, an `Instant`; shown through a zone from `TimeZones()`, never stored as a local reading |
| `timestamp + 86400000` for tomorrow | *silent*: wrong across a daylight-saving change | `instant.to_date_time_in(zone) + Period(1, 'days')`, then `tomorrow.to_instant_in(zone, 'compatible')` |
| `text[0].upper_case()` | `'text[0]' may be missing, since every '[ ]' answers a 'T?'` | `crash text[0]` first: `text[index]` is the whole character at that byte, `null` out of range or inside one |
| `// comment`, `/* comment */` | `Spite has no '//' or '/* */' comments` | nothing, or `# docs/page.md#section` on its own line outside a function |
| `;` at the end of a line | `';' is not something Spite reads` | nothing |
| `var memory = Memory()`, an allocator | `unknown type 'Memory'` | nothing: the compiler places memory; a container of your own is written over a `List` or a `Buffer` ([memory.md](https://github.com/Spite-Language/spite_documentation/blob/master/memory.md)) |
| `--repl_port=4000` | `'--repl_port' is written '--repl-port'` | kebab-case flags; the `Build` field behind it stays `repl_port` |
| `spite -- --player-count=2` | `'--' is not a separator` | `spite --player-count=2`: Build flags sit beside the compiler's |
| `spite --player_count=2` | `'--player_count' is written '--player-count'` | flags are kebab-case on the command line |
| `spite --player-name=ada` for an `Environment` setting | `'--player-name' is not a compiler option` | `game.exe --player-name=ada`, or `PLAYER_NAME=ada spite` |
| `spite game` | `'game' is given by position` | `cd game` then `spite` |
| renaming `short`, `static` or `near` because C takes them | | the name you meant: Spite reserves nothing for C |

Text is written with its values inside it: `"hello {name}"`, where `{ }` holds one value of any type and
`\{` is a brace meant literally.
