---
name: spite
description: Use when writing, reviewing, fixing or explaining Spite code (.spite files), or when choosing how to build, run or debug a Spite program.
---

# Writing Spite

Spite is a small language with one way to do each thing. The compiler is also its formatter and its linter, it
never warns, and every error names the fix. So the fastest way to write correct Spite is to write the plain
version, compile, and do exactly what the compiler says. Where a mistake has exactly one fix, the compiler makes it
in your file and prints a `fixed:` line.

The whole language, every rule the compiler enforces and the habits from other languages it rejects, is in
[reference.md](reference.md) beside this file. Read it before writing more than a few lines of Spite: each habit
it lists costs a compile round trip to rediscover. The full documentation is
[docs/](https://github.com/Spite-Language/spite_documentation/blob/master/README.md).

## How to iterate: keep the program running

Do not compile the whole program after every change. Start it once with live reload and a REPL, and let every save
swap into the running program:

```
cd game
spite --hot-reload --repl-port=4000
```

It keeps running, even after its entry constructor returns, until a line ends it (`Program().exit(0)`). Then, for
each change:

1. **Edit and save the file.** The watcher compiles the program again and swaps in every function whose code
   changed, keeping every object's state.
2. **Ask the program instead of printing**, with lines of Spite: `spite --connect=4000 --command="goblin.roar()"`.
   A line first swaps in every save the watcher has seen, so it runs against the new code: never poll, never sleep.
   Every answer is one line of JSON, `{"ok":true,"value":"Goblin roars!","type":"String"}`, or `{"ok":false,"error":...}`
   with the compiler's error when the line, or the saved code, does not compile.
   - A line is ordinary code run in the program: an expression answers its value, a statement runs. `program` is
     the entry instance and its attributes and functions are named bare (`player.health`, `World().entity_count()`).
   - `var strongest = monsters[1]` and `func healthy(): Integer { ... }` stay for later lines.
   - Reflection answers questions about the program: `Monster.attributes`, `Monster.functions`,
     `Program().live_allocations()`.
3. **When a reload is refused**, read why. A change that adds, removes, renames or retypes an attribute, or
   changes an enum's values other than by adding new ones last, is refused: the program keeps all of its old code,
   and the answer says to start the program again. Send `Program().exit(0)` and start it again.

There are no commands and no breakpoints: to see a value, send a line; to stop somewhere, add a print and save.

Leave the loop only when you need to:

- `spite --check` formats and compiles without building or running, for a program that cannot stay running,
  or to see the errors; `spite --build` writes the executable without running it.
  Every compile formats first; there is no `spite format`.
- A full run from a fresh start, and the tests (`spite` in `tests`, [docs/testing.md](https://github.com/Spite-Language/spite_documentation/blob/master/testing.md)),
  before you call the work done.
- Measure speed only on a production build: no `--repl`, `--repl-port`, `--hot-reload` or `--debug-memory`.
  `--optimized` only sets `Build.optimized`; it does not make the program faster.
- A loaded package may reopen `Build` in its own `build.spite`; a field both it and the program declare is the one
  loaded last's (the package's), so a package names its fields after itself (`assets_cooked`, never `production`).

Every compile, a reload included, rewrites your files into the one style first: read a file back before editing it
again. Every error comes as `path:line: error: message (in Class.function)` and says what to write instead: write
that, and nothing cleverer. A compile lists its errors together, at most 25 in the program, followed by `and N more in
the program`, so fix them all before compiling again. A file that does not read lists every mistake in it at once:
its spelling mistakes (a mark, text or number Spite does not read) first, and once it spells right every syntax
mistake, one for each broken statement. A later stage does not run on a broken input: type errors come
once every file reads clean, and the errors that need the whole build (unread names, proven checks, a value that may
be null) once it builds, so a clean list can be followed by a new one. A failure at run time is a `spite: the program stopped at ...` or `spite.fault` line naming the file (from the program's folder, as `shop/orders/cart.spite`), the line
and the values there: open that line, and do not add logging. The REPL is in
[docs/repl.md](https://github.com/Spite-Language/spite_documentation/blob/master/repl.md).

Write the plain, readable version first. Do not hand-optimise: the compiler fuses chains, specialises generics,
places memory and removes what is unused on its own.

## Bind a library instead of rewriting it

When a mature library already does the job (compression, a database, a codec, physics, a graphics API), bind it
rather than rewriting it in Spite; rewrite only what is small or what the program is about. Any library with a C ABI
can be bound: C as it is, C++ through `extern "C"`, Rust through `extern "C" fn` in a `cdylib`, Zig through
`export fn`, Go through `//export` in a `c-shared` build.

- Open it once beside the attributes: `var library = DynamicLibrary("libpack.so", 'identity', "")`, and call its
  functions as members (`library.pack_block(...)`; `_as_long`, `_as_double`, `_as_text` for wider results).
- Wrap it in a class of its own, and let nothing else touch the `DynamicLibrary`. Only Spite shapes come out: full
  word snake_case names, a Spite enum for each status (map each C number with an `if` per number, then a bare
  `crash`, so a number it does not list halts at the boundary), `T?` instead of a null handle or a not-found code,
  an object that holds the handle and gives it back in `drop()`, and `String` for text.
- What crosses is numbers, `Boolean`, text, lists of numbers and number-only `type`s; a class instance never does.
  Exceptions, generics, structs returned by value, varargs and another runtime's memory cannot cross: wrap them on
  the library's side.

[docs/foreign_libraries.md](https://github.com/Spite-Language/spite_documentation/blob/master/foreign_libraries.md#a-binding-speaks-spite)

## Rules that matter most

- A file is a class named after the file; a program is a folder whose entry file is named after the folder. A file
  holds declarations only, in a fixed order, and at most 1200 statements (its file content budget: split a concern
  into a file of its own, never cut at the number); every `var` has a default.
- Full words only: `message`, never `msg`; `Integer`, never `Int`. `snake_case` values and functions, `PascalCase`
  types. No single letters except the axis names `x`, `y`, `z`, `w`.
- No `for`, `break` or `continue`. Prefer the list templates (`monsters.filter_alive().sum_health()`,
  `names.each(say_hello)`); a `while` that only does what a template does is an error naming the template.
- A `while` states how it ends: `while true` (or any condition known to be always true) is an error; loop
  `while running` and set `running = false` where it ends. A `while` whose condition reads only locals the body never
  changes, with no call, `return`, `fallback` or `crash`, is an error too.
- Use the operator, never its function: `a + b`, not `a.sum(b)`; `person.age`, not `person.get_age()`; `list[0]`,
  not `list.get_at(0)`.
- Metaprogramming is ordinary code on reflection objects, run while compiling: `Monster.attributes.each(describe)` with
  `describe(attribute: Spite.AttributeDeclaration)`. A function with a parameter named by a word of its own name is a
  template over that kind of member: `Spite.AttributeDeclaration` (read as `monster.attributes[attribute].value`),
  `Spite.FunctionDeclaration` (`run_phase_each(phase: ...)` walks the `<phase>_each` functions its name says, called as
  `system.phase_each(...)`, which the copy for `update_each` fills to `update_each`, refused if the class also has a
  `phase_each` of its own; used whole, `phase` is that function's `Spite.FunctionDeclaration` and a member
  parameter its member's declaration, with `.name` the text) or `Spite.Argument` (spread as
  `system.update_each(made_arguments(row))`); the body says which class. A class's `.name` is always its short name
  and `.name_with_namespaces` the dotted one; `.source_files` lists the files that declare it. Find classes by walking,
  never by registering them: `Spite.Class.instances.each(bind)`, `Theme.classes.each(bind)` and
  `window.register_namespace(Theme)` each make a copy of the walker per class, where the class is a type
  (`Binding<part>()`, `value.class == part` narrows). On an object known only at run time (`held: Anything`),
  `held.attributes` hands bound `Spite.Attribute`s whose `.value` is read and written: `destination.value =
  attribute.value` copies a field by the assignment rules it desugars to, judged at that line with the assignment's
  words (the held value has no range: a decimal or wider number into a plain whole-number field is refused, into a
  `T?` field it is `null` when it does not fit; a value that may be null into a plain field is refused, or 0 for a
  number), and a class reaching the value that no attribute the write can reach takes is a compile error at that line.
- No exceptions and no error values. Absence is a `T?` the caller must narrow (`if`, `fallback`, `crash`, `switch`).
  `null` only goes where a `T?` is wanted: given to any type without `?` (assigned, passed, appended, answered or
  declared) it is a compile error, never a silent `0`, `""`, first enum value or default object.
  `fallback condition` ends the function with what its answer type builds with no arguments (`null` for a `T?`,
  `0`, `false`, `""`, an empty list, or a fresh object of a class made with no arguments; an enum or a class whose
  constructor takes arguments has none, so a `fallback` there is an error: answer `T?` or `crash`); answer a `T?` when a
  caller must tell that from a real answer. `crash condition` halts on
  a developer mistake. Neither takes a message: the crash report shows the values.
  An `if` with no `else` that only returns what a `fallback` would is rewritten into that `fallback` (`fixed:`), its
  condition turned around: `<` becomes `>=` only between whole numbers (`fallback count >= 0 and count <= limit`), and a
  comparison of decimals stays under `not` (`fallback not (scaled > limit)`), since NaN fails every comparison.
- `crash flag` and `fallback flag` on a `Boolean?` pass only when it is there and `true`: `crash flags[index]` on a
  `List<Boolean?>` halts on `false` too. To test that it is there alone, write `crash flag == true or flag == false`.
- A narrowing, a `[]` proof included, lasts until a write or a call that may undo it or shrink the list (the compiler follows the call, whatever its answer is used for). Narrow again after
  one: the new check stays and tests at run time. A write or a call inside a branch that returns leaves it standing
  below the branch. A call in a condition counts too: `if rack.item and clear(rack)` proves nothing about `rack.item`
  when `clear` may write it; `if clear(rack) and rack.item` does. A proof made before a `while` and read in its
  body is undone by such a call anywhere in the body, since the next pass reads after it: prove it inside the loop.
  A call earlier in the same statement counts too (`clear_count(rack) + rack.item.number` is refused; read first),
  and a `then` never undoes its `else`. A `[]` proof also falls to a shrink through a local or parameter that may be
  the same list (`var alias = items`, `alias.clear()`) and to a count kept in a `var` before a shrinking call.
- Only a path with no call in it narrows: `if thing.pop().held` proves nothing about a second `thing.pop().held`
  (it calls again); write `var held = thing.pop().held` and narrow `held`. A getter path (`thing.reading`, running
  `get_reading()`) narrows while the getter only reads; if it may write anything, a second read after the check is an
  error: hold it in a `var` the same way. `thing.pop().class == Thing` still calls `pop` once.
- `while given and turns < 2` narrows `given` in the body like an `if`, and each pass tests it again, so the body may
  write it. An `or` narrows nothing inside, except an `or` of class tests of one value: `if value.class == String or
  value.class == Integer { }` compiles the block once per class, each copy narrowed to its class, and a use is refused
  only where one of them does not fit (`a Holder cannot be used where an Integer is needed: ...`). `if amount > 100 or
  amount < 0 { return 1 }` proves both sides below it.
- A number that may be null is narrowed before arithmetic, a comparison, an argument or a `return`, whatever its width; only `==` and `!=` take it as it is, and `var zeroed: Integer = pair` on its own line is 0 when `pair` is null.
- A comparison is a call on its left side (`a == b` is `a.equals(b)`, `a < b` is `a.less_than(b)`), so the right side goes into the left's type like any argument: write the wider number on the left, `total > count` with a `Long` `total`, `color_type == buffer[0]` with an `Integer` `color_type` (a `Byte?`; a missing byte equals no number). The other order is swapped by the compiler with a `fixed:` line when the swapped form fits, and refused naming both fixes (narrow by hand on its own line, or swap the sides) where it cannot rewrite the line. Where neither order fits (an `UnsignedInteger` or `UnsignedLong` beside a signed number) it is the assignment's error: narrow one side first. Arithmetic also wants the wider operand first.
- A number put into a narrower one is `null` when it does not fit: `var small: Tiny? = count`, then narrow `small`; `var small: Tiny = count` compiles only where the compiler proves it fits: inside `if count >= smallest and count <= largest { }`, or past `fallback held and held <= largest` for a `T?` local `held`. A decimal into a whole number or a `Double` into a `Float` is written only on its own line, `var whole: Integer? = reading`, or `var whole: Integer = reading` where a condition that holds bounds it on both sides (`if reading >= smallest and reading <= largest { }`); `if reading > largest or reading < smallest { return 0 }` does not bound a decimal, since `nan` fails every comparison and gets past it.
- A whole number of three or more digits (not a power of ten or two) is named by a private attribute
  (`var _days_in_400_years = 146097`), except the answer a test compares in its `crash`: `crash boxes.count() == 1001`
  inside a `test_...` function stands as written.
- Never write defensive code: a check on a value the compiler already proves is removed, and says so.
- A `crash` whose condition the compiler decides is false, in a function the program always reaches, refuses the build.
- No lambdas, no destructuring, no overloading, no `async`/`await`. Pass a named function; one name is one
  function; the compiler decides what runs at once.
- A call or a constructor is never an argument of another call: compute it into a named `var` first.
- A user interface tree may be written as tags (JSX), only after `return` or `var name =`: each tag is a construction
  with no arguments, each attribute an assignment, and the children one assignment of a list, `view.children =
  [text, badge]` (each nested tag or `{value}` one entry, typed by the `children` attribute, through `set_children`
  when declared; a whole list is `children={list}`); a lowercase tag calls `render_<tag>` with its attributes as the
  parameters, by name (`<weapon grade={1}/>` is `render_weapon(1)`), and takes no children. A tag names a class as
  code does: `<ClassPicker/>` inside `sidebar/`, never `<Sidebar.ClassPicker/>` (rewritten with a `fixed:` line). See the markup section of [reference.md](reference.md).
- A comment is one line outside functions and nothing but a link to a markdown heading. Function bodies hold no
  blank lines: name the part and call it.
- Every object is a reference counted by the compiler. Two objects that hold each other (a parent and its
  children) are plain references: the compiler finds the back reference and frees the cycle. Keep the back
  attribute nullable (`var parent: Parent? = null`), set where the child is made or stored (through any number of
  calls and attributes); it may hold a union or a `type` when every class it may hold keeps the child the same way. A
  plain `var parent: Parent` frees only when no child outlives its parent: made by it (`Child(name, this)`), kept only
  in its list, used only while the parent is held by `this`, a parameter or an outer-block local never assigned again;
  never returned, stored elsewhere, copied or handed to a function value, and neither class's `drop()` reads the list
  or `parent`. Anything else leaks. Two objects made side by side and linked to each other directly are freed when
  neither leaves that function, through one `partner` attribute both ways too (the compiler clears the slot that
  closed the cycle as the names go, so that `drop()` sees `null`); when one leaves it, another function links them, or
  a function that calls itself links the pair the other way one attribute deeper, the pair leaks until one side is
  cleared. A function that calls itself one attribute deeper each call (`pair_down(link.next, second)`) links a pair
  like any other function. An attribute whose default names its own function (`var act:
  Spite.Function<Nothing> = rest`) is freed with its object when it is only called, narrowed or set to `null`;
  `copy()` binds the copy's value to the copy. Freeing takes no stack per level: a structure a million levels deep
  frees wherever its depth is (first child, `left`, last link), so build it the way that reads best.
- Singletons are bound once beside the attributes (`var console = Console()`), never called inline.
- An attribute the constructor fills on every path has no default (`var pet: Pet`), written below the attributes
  with one; nothing may read it before the constructor gives it (no earlier line, called function, `this` or
  default; a default only naming a function, `var act: Spite.Function<Nothing> = rest`, reads nothing until
  something calls `act`); every other attribute has a real default, never `var pet: Pet = null` (`null` only goes where a `T?` is
  wanted).
- Files the program carries in its executable are preloaded, never read beside it at run time: `preload "assets"`
  (written like `load`, a folder or one file) and `var assets = Volume("assets")` beside the attributes, named by the
  last folder as spelled on disk; `assets.file("levels/first.txt")` is a read-only `File` whose path is
  `assets://levels/first.txt`. A `preload` no `Volume` reads, or a `Volume` nothing preloads, is an error.
- An attribute's type may be left out only when its default is a literal, a list or dictionary literal, a constructor,
  a preset, or a chain of calls on one of those whose functions declare their answers (`var base = Thing().size(2)`
  with `func size(given: Integer): Thing`); otherwise write it, `var base: Thing = ...`.
- A `[` on a new line starts a list literal, never an index into the line above: `[Root].each(registry.register)`
  is a statement of its own.
- `Anything` is a union the compiler fills: each attribute, parameter, answer or local written `Anything` holds only
  the classes that reach it, so printing a `Spite.Attribute`'s `.value` asks `to_string()` only of what the
  attributes held, and a hole `"{value}"` on an `Anything`, like `value.to_string()`, is refused unless every class
  reaching it has its own `to_string()`, as on any union. An `Anything?` holding none shows `null` when printed,
  debugged or put in a hole, even when no class ever reaches it. Each `List<Anything?>` (or `Anything` in a function type)
  written is its own place too: a list of crates does not stop printing a list of gauges, unless one is passed into
  the other's place. `troll.attributes[declaration]` is troll's `Spite.Attribute`
  everywhere, in a walk or a template too: read and write the field through `.value`
  (`copy.attributes[attribute].value = price.attributes[attribute].value`); any other declaration as the key, in a
  walk too, must be of troll's own class and kept in a `var` given once, and `troll.attributes[health].value` read
  straight off it is troll's `health` with its own type (an `Integer` adds and prints), as is `.value` off a proven
  literal name (`crash troll.attributes["health"]` first; unproven, `+` or a call on that `.value` is refused as
  "may be missing ... narrow it first") and off a `Spite.Attribute` kept in a `var` given once from
  either (`var bound = troll.attributes[health]`, the field as it was then); one assigned again or handed in is an
  `Anything?`. Inside its own class it is bare,
  `attributes[declaration]`, as for its own attributes (`this.attributes` is refused). `troll[attribute]` is `troll.get_at(attribute)` there too, refused on a class without `get_at`.
- A function taking a `union` or a `type`, or a generic of one (`List<Anything>`, `List<Named>`), is compiled once per
  argument type a call hands it, and each copy is checked as if written for that type: `shown(healths)` with a
  `List<Health>` compiles while the body only does what a `List<Health>` can, and `items.append(sword)` in it is
  `a Sword cannot be used where a Health is needed: ..., in its copy for (List<Health>) that the call at line 3 in
  Armory.Armory makes`. A list is never cast: a value whose class is known only while running reaches the function as written.
- `list.sum()` adds the members with their own `+` (numbers, text, any class with `sum(other)`) and answers a `T?`,
  `null` when empty; `list.sum_by(measure)` adds what a function answers for each, an `Integer`.
- Bytes are a `Buffer` (`Buffer(0)`, `append(byte)`), never a list: `List<Byte>` is an error, written or made (a literal
  of `Byte` items, a call answering one such as `map_<plural>` over a `Byte` attribute, a generic class of yours bound
  to `Byte` that keeps a list of it).
- One character of a text is a `Rune`, not a `String`: `text[index]` answers a `Rune?` (byte index; `null` inside a
  character), `rune.unicode` is its number, `Rune(233)` makes one, and `rune.to_string()` is its text. `letter == "/"`
  compares as text: `==` becomes `equals`, and the rune becomes its text to meet the text.
- `a == b` and `a != b` are `a.equals(b)`: `b` casts to what `equals` takes (the class itself without an `equals`),
  so comparing a `Price` with `"cost"` is a compile error naming `Price(from: String)` unless that cast exists or
  `Price` has a `to_string()` (then it compares as text). Never expect a comparison across types to be `false`.
- Every class has `equals`: one without its own compares every attribute with that attribute's `==` (private ones
  too, back references such as `parent` included; cycles never loop), so two `Point(1, 2)` are equal, a deep copy
  equals its original, and `list.contains(item)` finds an equal item. A value typed by a `type`, a union or `Anything`
  compares through the class it holds: the same class calls its `equals`, different classes are not equal, two `null`s
  are equal and `null` never equals a value.
  To ask whether two names hold the very same object write `a.is_same_instance(b)`, and only where that is truly
  meant (a copy is a new object, a back reference points at this very node): it pins the object, so the compiler can no
  longer share a copy with its original. A `Dictionary` keyed by an object still keys by identity.
- Only a generic class takes `<Type>`: `Phase<Integer>` on an enum, a union, a `type` or a number is an error. The
  entry class is never generic; put what needs `generic` lines in a class it makes.
- A union or `type` that two classes of a namespace declare alike is moved by the compiler to the entry class (the class
  named like the folder); the other classes of that namespace write `Bunda.Pet`, a class outside it
  `Bunda.Bunda.Pet`, and only the entry class writes `Pet`.
- A union lists classes directly: a `type`, another union or `Anything` as a member is an error at its line. Write the
  inner union's classes into the outer one; for a `type`, list the classes that fit, or take the `type` itself where
  any class that fits will do (a function taking one is copied per class). Every member
  error points at the member's own line, and a union in a generic class nothing makes is checked all the same.
- Name another class's union or `type` through its class, never bare and never by declaring a copy: `Sibling.Type` in
  the same namespace, `Namespace.Class.Type` from elsewhere (`func shown(component: Framework.Component)`). It compiles
  exactly as a copy declared in your own class would, so a shared `type` serves where `Anything` was a stand-in.
- A class of the file's own namespace is written plain, never through that namespace: inside `sidebar/`,
  `Picker()`, `List<Picker>`, `Picker.Mode`, `<ClassPicker/>`. `Sidebar.Picker` there is rewritten to `Picker` with a
  `fixed:` line, unless the class itself declares a `Picker` (then the path is what tells them apart).
- The compiler fixes what has one fix, and prints `fixed:` for each: declarations out of order, a local or parameter
  with one right spelling, a file or folder of the program with one right spelling (renamed on the disk with every
  `load` naming it: `ToolHelper.spite` becomes `tool_helper.spite`), a keyword one letter off, a local or attribute that does nothing and is never read, a
  `load` of the standard library (every program has it with no `load`; `--standard-library-path=<folder>` names another), a line ending in `\r\n` or `\r` (a
  Spite file ends every line with `\n`), a check on a proven value, an `if` or `while` decided while compiling, statements after a `return`. A choice stays
  an error. A loaded package's files are fixed the same way; it never rewrites the standard library or a repository
  pinned to a commit. A check or `if` it would remove because of a narrowing
  stays when a later call or loop pass takes that narrowing back: the compile fails on that instead.

## Installing this skill in a project

Copy this folder (`SKILL.md` and `reference.md`) into the project's `.claude/skills/spite/`, or into
`~/.claude/skills/spite/` for every project.
