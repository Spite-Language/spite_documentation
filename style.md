# Style: the compiler is the formatter and the linter

Style in Spite is not a matter of taste, and there is no style guide to follow: the compiler either rewrites your
code into the one style or refuses it with an error that names the fix. It has no warnings and no switch to turn
a rule off. What cannot be rewritten safely, such as a name or a structure, is an error; everything else is rewritten.

## Why one style, and no warnings

A formatter that offers choices, and a linter that warns, hand every choice back to whoever is writing. Two ways of
doing one thing is a decision somebody makes again at every place it appears, so Spite keeps one: before adding a
way to do something, look for the one that already exists, and when two exist, remove one. Conversions are the
example: `Float.from_integer(count)` and `count.to_float()` both look reasonable, nobody could say which class owns a
conversion, and so only the conversion in the class it makes exists, `func Float(from: Integer): Float?`, which the
compiler applies wherever the value meets the class.

For the same reason there are no warnings. A warning is a decision left to the reader, and a reader who skims will
skip it. The compiler either rewrites your code into the one style or refuses it. Syntax follows the same rule:
before a new form is added, check whether something that exists already says it. The old `generics` header line
went because it sat outside the patterns the rest of the language uses.

## What the formatter rewrites

Every compile formats the program's own files, printing `formatted <path>` for each one it changed, and then reads
the program again, so what it compiles is the formatted file. Nothing turns this off, and there is no command
that only formats: `--check` formats and checks without building ([compiler.md](compiler.md#formatting)):

- 4 spaces per level, never a tab; no trailing whitespace; one line break at the end of the file.
- K&R braces: `func name() {`, `} else {`, `} else if other {`. An empty body is `{ }`, and an empty `else` is
  dropped.
- A comment is `# text`, one space after `#`. What is between a text's quotes is never touched, escapes and all.
- One space around binary operators and after `,` and `:`; none inside `( )` and `[ ]`.
- The fewest parentheses that keep the meaning: `1 + 2 * 3`, not `1 + (2 * 3)`.
- A list, object or call that fits in 120 columns goes on one line with `, `; a longer one is broken one entry
  per line, with no commas for a list or object literal and with trailing commas for a call.
- One blank line between declarations at file level; consecutive `var`s may stay grouped.
- A switch case with one short statement stays on its line; `crash false` becomes a bare `crash`, the one form
  for a branch that cannot happen ([failure.md](failure.md#crash)).

It never reorders code or drops a comment: it re-reads its own output and checks that the program and every
comment survived, and if it cannot prove that, it leaves the file alone and says why, which is an error that stops
the compile, so a program is never compiled from text that is not in the one style.

Everything on this page happens while compiling. None of it reaches the built program, which carries no trace
of the formatter or of any of these checks.

## Why code is written this way

Every rule on this page comes from one fact: Spite is written mostly by AI and read by people who skim. A long way
round that the compiler accepts is a pattern that spreads, and a model copies the messy code it has seen, so the
short form is made the only form the compiler accepts. In the same spirit:

- **No clever code.** A program is not improved by being surprising, and almost all of it is written by a model that
  needs no trick to be fast.
- **No `for`.** There is `while`, and there are the member templates that write the loop for you
  ([collections.md](collections.md#member-templates-loops-you-do-not-write)).
- **No destructuring and no lambdas.** They are pleasant to type, but a person is not the one typing, so they cost
  readability for nothing. Read members by name, and pass a named function, which is bound to its instance
  ([functions_and_operators.md](functions_and_operators.md)).
- **No overloading.** One name means one function, and an argument is cast to the parameter's type, so a reader
  never works out which of several meanings was chosen.
- **No copying a value into a local only to narrow it.** `var watcher = tracker` then `fallback watcher` is a habit
  from languages that cannot narrow a path. Narrow the name or the path itself: `fallback tracker`,
  `fallback tracker.target` ([failure.md](failure.md#narrowing-a-path)).
- **No em dashes, anywhere**: not the character, and not two hyphens between spaces standing in for one, in code,
  comments, documentation, diagnostics or commit messages. End the sentence, or use a colon, a comma or
  parentheses. Command-line flags such as `--optimized` are not dashes.

## A file is ordered

A file holds, in this order: the `singleton` line, `generic` lines, enums, unions, types, variables, the
constructor, then functions ([classes_and_files.md](classes_and_files.md#what-a-file-holds)). A file of yours
that is out of order is put back in order by the compiler, which moves each declaration (with the comment lines
right above it) and says so, then compiles the fixed file:

```gdscript title=declaration_order_error/declaration_order_error.spite entry
var console = Console()

func DeclarationOrderError() {
    load "../spite_standard_library/windows"
    console.print(label)
}

var label = "late"
```
```output
late
```
```diagnostic
fixed: a variable came after the constructor here; moved it up, since a file is ordered: the singleton line, generic lines, enums, unions, types, variables, the constructor, conversions, presets, then functions
```

## The compiler fixes what has one fix

Anything with exactly one fix is treated like the order of a file (D1767): the compiler fixes it in the file and prints a
`fixed:` line for it on the error output, `<path>:<line>: fixed: <what was wrong>; <what it did> (in
<Class>.<function>)`, the line being the one the mistake was on, so you read what changed. Then it compiles the
fixed file. A mistake whose fix needs a choice stays an error, and a file of
the standard library or of a package is never rewritten, so there it stays the error.

What the compiler fixes for you, and what it leaves as an error:

- **Dead checks.** A `fallback`, `crash` or `if` on a value that is already narrowed or proven present is removed (an
  `if` becomes the code it runs, an `if not` its `else`); an `if` or `while` decided while compiling goes, or
  becomes its body; a `fallback` that is always false goes with the code after it. A `while` that is always true,
  an `if` that holds only part of a class test and a check that something undid stay errors, and so does a write or
  a call that may undo a proof, which keeps the check and answers at run time. A local taken from a narrowed path
  is proven too: after `crash thing.store`, `var held = thing.store` and `if held` loses its `if`. These fixes rest
  on what was narrowed, and a later proof may still take that back (a call between the check and a read, a loop's
  next pass): when one does, the compile fails with those lines as you wrote them.
- **Shapes with one short form.** A last `if` that only checks a value becomes `fallback` (except in a constructor,
  where it is an error), an `if` that only returns the default becomes `fallback`, and an `else if` that only returns
  is hoisted to a `fallback` above its `if` when one choice fits. A copy made only to narrow goes and its uses name
  what it copied. Statements after a `return` go.
- **Unused things with no effect.** An unread local or attribute, or a `load` nothing names, whose value and writes do
  nothing is removed with every write; one a call makes or writes stays an error.
- **Loads.** A load of the standard library's `common` folder alone becomes a load of the `windows` folder beside it,
  which loads `common` ([programs.md](programs.md#how-a-program-is-loaded)).
- **Names.** A local or parameter with one right spelling is renamed, and a keyword one letter off is corrected. A
  file or folder of the program with one right spelling is renamed on the disk, with every `load` that names it
  ([classes_and_files.md](classes_and_files.md#a-misnamed-file-or-folder-is-renamed)).
- **Placement.** A declaration out of order is moved with the comment lines above it, a comment in the wrong place
  moves to the top of the file, and a `union` or `type` two classes of one root declare alike moves to the entry
  class of the lowest namespace, with every use rewritten.

## Names

A local or a parameter, and a file or folder of the program, with one right spelling is renamed for you, a file or
folder with every `load` that names it. Any other name (a function, an attribute, a class, an enum value) is never
rewritten, because renaming it can change what a program means to someone searching for the old name. These are
errors, each with the fix:

- `snake_case` for variables, attributes, parameters, functions, enum values, files and folders (a package's
  root folder too: `window_plugin`, never `window-plugin`); `PascalCase` for classes, enums, unions and
  types. A file or folder of the program with one right spelling is renamed, with every `load` that names it
  ([classes_and_files.md](classes_and_files.md#a-misnamed-file-or-folder-is-renamed)).
- **Never a single letter**, except the axis names `x`, `y`, `z` and `w`, which are the names of what they
  hold and abbreviate nothing: `position.x`, `var x = 0.0`.
- **Never an abbreviation.** Every `_`-separated word of a name is checked against a list (`msg`, `idx`, `val`, `cfg`,
  `tmp`, `str`, `len`, `max`, `min`, `init`, `env`, `dir`, `doc`, `btn`, `cnt`, `addr`, ...), and the error or the fix
  spells every matching word out (`max_len` is `maximum_length`). The plural of an abbreviation is the same (`msgs`,
  `idxs`, `bufs`), and so are the language's own words used as names: `int`, `char`, `bool`, `fn`. `id` is allowed.
- **Any word another language reserves is yours too**: `register`, `short`, `default`, `unsigned`, `static`, `stdout` or
  `near` are ordinary names for a variable, attribute, parameter or function, and the compiler keeps your spelling in
  reflection, `--final-classes`, crash traces and errors. `int`, `char`, `bool`, `min` and `max` are still
  abbreviations, because they are abbreviations here.
- **One underscore between words**, and at most one in front to make a name private: `hit__count`, `__strike`
  and `strike_` are errors. That is what keeps the names the compiler makes for a class (how it allocates, counts
  and releases one, under names with `___` in them) out of the way of yours, so `allocate`, `make`, `release`,
  `destroy`, `discard`, `default`, `perform`, `hot`, `slot` or `waiting` are ordinary names. The exceptions are the
  few words reflection answers for every object: a function or attribute cannot be named `attributes`, `functions`,
  `instances`, `class` or `memory`.
- **Never a function's name.** A local, parameter or attribute named like a function of its class is an error,
  since the function's name is already a value ([functions_and_operators.md](functions_and_operators.md#functions-are-values)):
  `var stem = file_stem(path)`, not `var file_stem = file_stem(path)`.
- **Never a class that hides another.** A class named like a class its code can see from an enclosing namespace
  (`Physics.Plugin` beside a root `Plugin`), or like a class of the standard library, is an error naming both
  ([A class name means one class](classes_and_files.md#a-class-name-means-one-class)).
- **Names say what they do.** The standard library follows the same rule: `append` and `prepend` rather than
  `add`, `upper_case()` rather than `upper()`, `remove_last()` rather than `pop()`.

```gdscript title=lint_error/lint_error.spite entry error
var console = Console()
var msg = ""

func LintError() {
    load "../spite_standard_library/windows"
    console.print(msg)
}
```
```diagnostic
'msg' abbreviates: write 'message' instead of 'msg'
```

A local or a parameter with one right spelling is renamed in its function instead, and the compiler says so: `itemCount` to
`item_count`, `total__price` to `total_price`, `item_cnt` to `item_count` (`fixed: 'item_cnt' abbreviated, and Spite has no abbreviations
outside its own keywords; renamed it 'item_count'`). A name that is already taken stays an error, and so does one that would
become a word of the language. A word that could mean several things stays an error, and you write the one
you mean: `'temp_file' abbreviates 'temp', which could mean 'temporary' or 'temperature'`. The words with several meanings
are `temp`, `res`, `dir`, `init`, `ext`, `desc`, `pos`, `cur`, `curr`, `auth`, `proc`, `ver`, `sep`, `conf`, `req`, `prev`,
`val`, `doc`, `spec`, `calc`, `arr`, `cb`, `int`, `opt`, `lst` and `fmt`. A keyword one letter off at the start of a line is
corrected the same way: `retrun` to `return`.

The old spellings of the number types are errors that name the new one: `Int` is `Integer`, `Bool` is `Boolean`,
`UnsignedInt` is `UnsignedInteger`.

```gdscript title=single_letter/single_letter.spite entry error
var console = Console()
var n = 0

func SingleLetter() {
    load "../spite_standard_library/windows"
    console.print(n)
}
```
```diagnostic
is a single letter: give it a name that says what it holds
```

### The words of a name

An abbreviation is a name you have to translate, so there are none: `DLL` became `DynamicLibrary`. The exceptions are
small and exact. A well-known acronym followed by a full word that carries the meaning, as in `Http.Client`, because
most people know HTTP and few know what it stands for; the acronym is written as a word, `Http`, not `HTTP`. And the
axis names `x`, `y`, `z` and `w`, which are the names of what they hold.

A name should say where and what without a comment to disambiguate it. `add` does not say where, so lists have
`append` and `prepend`.

## Nothing unused

A local variable that is never read is an error: remove it. Only reading counts, so a variable that is
assigned and never read is unused too, and there is no spelling that silences it: a variable nobody reads
is work the program did for nothing. A parameter is the one exception, because a signature can need a parameter
its body ignores: name it `_name` to say so. A `_name` that *is* read is renamed `name` in the file, since the prefix says it is
not, and a parameter whose signature is dictated from outside (an operator function's, for one) needs no `_` at
all (see the exemptions below). Everywhere else a leading `_` means private to its
class ([reflection.md](reflection.md#reflection-is-read-only)).

```gdscript title=unused_error/unused_error.spite entry error
var console = Console()

func UnusedError() {
    load "../spite_standard_library/windows"
    var greeting = "hello"
    var forgotten = greeting.length()
    console.print(greeting)
}
```
```diagnostic
'forgotten' is never read: remove it
```

An unread local whose value does nothing, a literal, a text with no hole or a name, has one fix, so the compiler
removes it from the file and says so: `fixed: 'forgotten' was never read, and making it does nothing; removed it`. It
removes one only ever assigned such values the same way, with every write. One made or written by a call stays an error, since
removing it would drop the call.

```gdscript title=written_not_read/written_not_read.spite entry error
var console = Console()

func WrittenNotRead() {
    load "../spite_standard_library/windows"
    var total = 1
    total = "hello".length()
    console.print("hello")
}
```
```diagnostic
'total' is never read: remove it
```

An attribute nothing reads is an error the same way, and goes from the file the same way when its value and everything written to it in its class do nothing. A `load` in the entry class of a folder no class outside it names is removed too. A private `_name` attribute is no
exception, since only its own class can read it. Writing an attribute is not reading it, so one that is only
assigned is still unused. A read is anything that takes the attribute's value: its name in the class's own
functions, `thing.world` from another class, `x.attributes[attribute]` in a template or a walk (which is how the JSON
and binary writers and `to_debug()` read), and a union's members or a `type` a class is admitted to. A walk that only looks
at `attribute.name` or `attribute.class` reads the attribute's description, not the attribute, so an attribute
kept only as a marker for such a walk is an error, and so is one kept only for what its constructor starts
(`var server = Server()` that nothing reads), whatever its name:

```gdscript title=unused_marker/world.spite
singleton
```
```gdscript title=unused_marker/mover.spite
var world = World()
```
```gdscript title=unused_marker/unused_marker.spite entry error
var console = Console()
var mover = Mover()

func UnusedMarker() {
    load "../spite_standard_library/windows"
    Mover.attributes.each(classify)
    console.print(mover.class)
}

func classify(attribute: Spite.AttributeDeclaration) {
    console.print(attribute.name, "is a", attribute.class)
}
```
```diagnostic
the attribute 'world' is never read: remove it
```

**A package's public attributes are judged where their readers are.** A public attribute is data a class offers
to code it cannot see, so a folder the program `load`s is not held to what this one program reads of it: a
`ui` package's `color` that only a `render` package reads is fine in a program that loads `ui` without `render`.
The check covers the program's own folder and the standard library (always compiled whole, so its reads are the
same in every program). A singleton binding is checked everywhere, loaded
packages included: `var world = World()` offers nothing another class could not bind itself, so an unread
one is the error above wherever it is.

A read is a read of one class's attribute: `tag.label` reads `Tag.label`, so a `label` the entry declares and
never reads is the error above even when another class's `label` is read everywhere.

Unused means unread anywhere in the source, not unreachable. A function nobody calls is not an error, private or
public: its callers may be code this compile cannot see, and the production build drops it anyway
([compiler.md](compiler.md#development-builds-and-tree-shaking)). It still reads what it names, so what it
reads counts as used; since such a body is not compiled, its `x.name` counts for every class with a `name`, and
so does one in a function a codegen or `Build` test cuts a branch from. A few attributes are never checked, because the compiler reads them itself: a number
class's or `String`'s storage, and `Build` and `Environment` settings.

**A parameter that needs no `_`** is one whose signature is dictated from outside: the parameter of an operator function
(`equals(other)`, `sum(other)`, `get_at(index)`), the single parameter of a setter that names a real attribute
(`set_age(value)`), and every parameter of a template over members, a function with a parameter named by a word of
its own name and typed `Spite.AttributeDeclaration`, `Spite.FunctionDeclaration` or `Spite.Argument`
(`save_attribute(attribute: Spite.AttributeDeclaration)`), whose signature is the template's
([metaprogramming.md](metaprogramming.md)). A two-parameter `set_age` follows the
ordinary rule. Every other unread parameter is `the parameter 'amount' is never
read: remove it, or name it '_amount' if the signature needs it`.

**The check waits for the rest.** The unread-attribute check runs only when the program has no other error, so a
statement that fails and was the only reader of an attribute gives you its own error first; the same goes for a local
declared by a statement that failed.

## Comments are links

A comment is one line that holds nothing but a link to a heading in a markdown file, and it sits right above a
function or at the top of the file, as the class's heading. The compiler moves one written anywhere else between
declarations (above an attribute, an `enum` or a `type`) to the top of the file and says so; one inside a function, or
inside the body of an `enum`, a `union` or a `type`, is an error. Prose above a two-line function that explains the compiler instead of the code is the habit this
rule ends, and a note that is needed becomes a section somewhere a reader can open:

```gdscript
# notes.md#why-this-program-exists
func Game() {
```

The path is resolved from the folder of the file the comment is in, as `load` resolves its folder, and the compiler
checks that the file exists and has that heading, so a link cannot rot silently. A heading is found the way GitHub
makes its anchor: lowercase, punctuation dropped, spaces turned into hyphens (`What's new in "version" 2?` is
`whats-new-in-version-2`), and a heading repeated in the file takes `-1`, `-2` and so on after the first, so the
second `## Details` is `#details-1`; a `#` line inside a fenced code block is not a heading. Prose in a comment, `//` and `/* */` are errors, and each one asks
whether the note is needed at all: if a reader could work it out from the code, delete it; if it is lasting
knowledge, write the section and link to it; if it warns against a change, a test or a compile error pushes
harder than prose. A line inside a function that seems to need explaining becomes a named function instead:
a name, unlike a comment, is visible to reflection, `--final-classes` and every tool.

These rules are the program's own. A file of the standard library or of a package, the files the compiler never
rewrites, has its comments dropped when it is read (D1933), so no comment rule looks at them and their links are not
checked: a library checked out alone, or loaded pinned with `@`, compiles without the markdown it links beside it.

The one other link is the one `--final-classes` writes: in the folder it prints, each attribute and function is
marked with a comment linking the `.spite` file that supplied it (`# ../../game/hero.spite`), with no anchor, so
the printed classes name where every declaration came from and still compile. Only a file under a folder
`--final-classes` wrote (it holds a `.final-classes` file) may link a `.spite` file; anywhere else it is `this
comment links to 'helper.spite', a .spite file, and only a file '--final-classes' wrote may link one`.

```gdscript title=prose_comment_error/prose_comment_error.spite entry error
var console = Console()

# prints a greeting
func ProseCommentError() {
    load "../spite_standard_library/windows"
    console.print("hello")
}
```
```diagnostic
this comment is not a link: a comment is one line holding only a link to a markdown section
```

## A number gets a name

A number inside an expression says nothing about what it means, so it is named first, by a private attribute of
the class:

```gdscript
var _days_in_400_years = 146097

func days_of(cycles: Integer, remaining: Integer): Integer {
    return cycles * _days_in_400_years + remaining
}
```

`146097` in the middle of the arithmetic is an error that asks for this form:
`the number 146097 does not say what it means: declare a private attribute that names it, like 'var _what_it_means = 146097', and write the name here`.
The rule starts narrow and grows: today it covers a whole number of three or more digits that is neither a power of
ten nor a power of two, and a decimal that is a preset of the library (`3.14159` is refused, naming `Float.Pi()`).
An attribute's default is where the number may stand, and so is the body of a preset. The name costs nothing: an
attribute nothing writes is read as its default where it is read, a constant as the number was
([values_and_types.md](values_and_types.md#named-numbers)).

## The file content budget

A file holds at most 1200 statements. That number is its file content budget, it holds for every Spite program, and
no flag, setting or comment lifts it. A file that long is rarely one thing: past the budget it is probably missing some cleanups.

```text
drawing.spite: error: this file holds 1201 statements, over its file content budget of 1200: clean it up until it is back within it, grouping state into an object and giving each concern a file of its own (in Drawing)
```

The compiler counts from the parsed file, so formatting changes nothing:

| Counts as one | Counts as nothing |
|---|---|
| a local, an assignment, a call, `return`, `fallback`, `crash`, `load` | comments and blank lines |
| `if`, each `else if`, `while`, and each statement inside them, at any depth | closing braces and a bare `else` |
| a `switch` | the cases of that `switch` |
| a class-level `var` | function headers |
| a statement spread over many lines | the entries of an `enum`, `union` or `type` |

Each of the exceptions is there for a reason:

- **An enum's entries.** An enum is one entry per line and reopening it replaces it, so it has no way to get smaller. If
  lines counted, an enum over the budget could not exist. Counting statements forbids nothing.
- **A switch's cases.** Each case is one line, and a switch over an enum is exhaustive with one value per case, so a
  switch over a large enum cannot be split either. A case that opens an `if` or a `while` still pays for what is inside it.
- **A statement over many lines.** A printed call with one argument per line, or a long list literal, is still one thing
  said once, so it costs one.

It is an error and not something the formatter rewrites, because where to split a file is a design choice the
compiler cannot make. Get back within the budget by cleaning up, not by cutting the file in two at the number: group
state into an object, name the concerns, and give each one a file of its own.

## A function does not only answer a member

A function that only hands on a member of the same name from one of its attributes says nothing a caller could not
say more plainly, so it is a compile error, and the error names what to write:

```text
token.spite:3: error: 'get_name' only answers 'owner.name': remove it, and write 'token.owner.name' where it is called
```

A `get_` function that reads another object's attribute is refused under any name too, so renaming it does not help.
Three things are not an alias, and stay:

- **The attribute is private with no getter**, so there is no other way in: `Tcp.Client.read_line` over `_connection`.
  Keep a thing to yourself by naming it `_connection`, and wrap what you choose to show.
- **The name adds something**: `is_atomic` over `atomic_singletons.contains(key)` is a question with a name of its own.
- **The call passes something else**: `rack.scale(size, 2)`, or the parameters in another order.

"The whole body" is one statement: `return` of the member, or the call alone, with or without `return`; a `get_name`
answers `name`, so `get_name` returning `owner.name` or `owner.get_name()` is the same alias. An attribute a caller
can reach is a public one, or a private one with a getter in the same file, which the error names by its public
name. The rule reads each file on its own, so it runs on every file a program loads, used or not.

## A function body holds no empty lines

A blank line inside a function is where a second function wants to be: the part below it is a step with a
name. Name it and call it. The formatter deletes an empty line inside a function body, and turns a run of empty
lines outside functions into one; every compile formats first, so a body written with an empty line
compiles, and the file comes back without it (`formatted path`). What stays is the rule that no formatted file has
one, and the push to name the part below is yours, since the formatter only joins the halves. Between
declarations one blank line stays.

## One call per line

**No call is an argument.** A call is computed first into a named `var`, and the name is passed:

```gdscript title=call_argument_error/call_argument_error.spite entry error
var console = Console()

func CallArgumentError() {
    load "../spite_standard_library/windows"
    var source = "spite is small"
    console.print(source.slice(0, 5))
}
```
```diagnostic
is called inside an argument of 'console.print': compute it first into a named 'var' and pass the name
```

```gdscript title=call_argument_fixed/token.spite
var text = ""

func Token(new_text: String) {
    text = new_text
}
```
```gdscript title=call_argument_fixed/call_argument_fixed.spite entry
var console = Console()

func CallArgumentFixed() {
    load "../spite_standard_library/windows"
    var source = "spite is small"
    var first_word = source.slice(0, 5)
    crash first_word
    console.print(first_word)
    var tokens = List<Token>()
    var token = Token(first_word)
    tokens.append(token)
    var word_count = source.split(" ").count()
    var first_token = tokens.first()
    crash first_token
    console.print("{word_count} words, first token {first_token.text}")
}
```
```output
spite
3 words, first token spite
```

A method called on a call's result (`source.split(" ").count()`) and the holes of a text
(`"{names.count()} names"`) are not arguments, and neither is a call in an `if` or `while` condition, a `return` or an
assignment. Anything inside an argument counts: `counts.append(count_words(text) + 1)` and
`print(names[index_of(name)])` each put a call in an argument. A call in a hole that is itself an argument of another
call (`"{shout(count_words(text))} words"`) is an error. A method called on a call's result and then passed on is an
error for the call it is passed to.

**A constructor call is not an argument either.** An object is made on a line of its own and passed by
name, however short the line:

```gdscript title=constructor_argument_error/label.spite
var text = ""

func Label(starting_text: String) {
    text = starting_text
}
```
```gdscript title=constructor_argument_error/constructor_argument_error.spite entry error
var console = Console()

func ConstructorArgumentError() {
    load "../spite_standard_library/windows"
    var labels = List<Label>()
    labels.append(Label("new"))
    var count = labels.count()
    console.print(count)
}
```
```diagnostic
'Label("new")' is constructed inside an argument of 'labels.append': a constructor call is never an argument, so make it first on a line of its own, 'var label = Label("new")', and pass 'label'
```

A constructor is a call whose last name starts with an upper-case letter, so this covers a constructor inside
another constructor's arguments (`Label(Font())`), a generic or dotted constructor (`List<String>()`,
`Game.Player()`) and a collection made for a call (`buffers.set(List<String>())`), in every place an argument list
appears: a `var`, an assignment, a call standing alone, a `return`, a `fallback` or `crash`, an `if`, `while` or
`switch`, an attribute's default and a text's hole. An object made to be read at once is not an argument: `JsonWriter<Order>().write(order)` is
fine, and so is `var label = Label("new")` itself.

**An `if` and its `else` do not repeat the same work.** When both branches compute the same call, it is
computed once before the `if`. It stays narrow: only a call with at least one argument counts (a constructor does not),
that prints identically in every branch of an `if`/`else` or of a whole `else if` chain ending in `else`, at the start
of the branches. A call that stands alone as a statement, differing arguments, a call that mentions a name the branch
declared, assigned or asserted before it, anything after the branch's first nested `if`, `while` or `switch`, and the
right side of `and` or `or` are not reported.

```gdscript title=repeated_branch_error/repeated_branch_error.spite entry error
var console = Console()

func RepeatedBranchError() {
    load "../spite_standard_library/windows"
    var source = "spite"
    var shout = true
    if shout {
        var word = source.slice(0, 3)
        console.print(word.upper_case())
    } else {
        var word = source.slice(0, 3)
        console.print(word)
    }
}
```
```diagnostic
'source.slice(0, 3)' is computed in both branches: compute it once before the 'if'
```

**Only a name calls a function.** A function is called on a name, a literal, or the result of another call (a
chain), never on a value computed in place: `(start + bytes * last).copy_to(target, bytes)` makes the reader
work out the receiver before reading the call, so the value is named first:

```gdscript title=computed_receiver_error/computed_receiver_error.spite entry error
var console = Console()

func ComputedReceiverError() {
    load "../spite_standard_library/windows"
    var first = 3
    var second = 4
    var largest = (first + second).maximum(5)
    console.print(largest)
}
```
```diagnostic
'(first + second).maximum(...)' calls a function on a value computed in place: name it first ('var named = first + second') and call 'named.maximum(...)', since only a name calls a function
```

Write `var total = first + second`, then `total.maximum(5)`. The same holds for `(not done).to_string()` and
`(-offset).absolute()`. A negative number written out (`(-2.5).floor()`) is a literal, not a computation, and may be
called on, and so may a text with holes, an index (`names[0].length()`) and a member path. Reading an attribute of a
computed value, `(first + second).x`, is not a call, and is allowed.

## The short form is the only form

When the language has a short way to say something, the long way is an error that names it. Each has its own
page:

| Written | Is an error naming | Page |
|---|---|---|
| `if not ready { return }`, `if names.is_empty() { return null }` | `fallback ready`, `fallback not names.is_empty()` | [failure.md](failure.md#an-if-that-only-returns-the-default-is-an-fallback) |
| a last `if value { ... }` with no `else` | `fallback value` | [failure.md](failure.md#the-last-if-of-a-function) |
| `var watcher = tracker` then `fallback watcher` | `fallback tracker` | [failure.md](failure.md#narrowing-a-path) |
| `fallback tracker` on a value that cannot be null | removing it | [failure.md](failure.md#narrowing-a-path) |
| a switch that is one class case and `_:`, each a `return` | `if value.class == Class` or `return value.class == Class` | [control_flow.md](control_flow.md#valueclass--class) |
| a switch whose cases all share one body but one | `if value.class == Class { ... } else { ... }` | [control_flow.md](control_flow.md#switch-over-a-union) |
| three `if`s comparing one value with its enum values, each only returning | the `switch` it spells | [control_flow.md](control_flow.md#switch-over-values) |
| a `while` over `items` whose body only adds up `items[index].price` | `var total = items.sum_price()` | [collections.md](collections.md#member-templates-loops-you-do-not-write) |
| an `if`/`else` directly inside a branch of another `if`/`else` | a function named for what the inner one decides, or one `switch` | [control_flow.md](control_flow.md#if) |
| `"hello " + name` | `"hello {name}"` | [values_and_types.md](values_and_types.md#string) |
| `count * total` with an `Integer` `count` and a `Long` `total` | `total * count` | [values_and_types.md](values_and_types.md#wider-arithmetic-goes-wider-operand-first) |
| `(65536 - 120) * 65536` | `4287102976` | [values_and_types.md](values_and_types.md#wider-arithmetic-goes-wider-operand-first) |
| `func Holder() { }` | deleting it: a class without a constructor is made from its defaults | [classes_and_files.md](classes_and_files.md#constructors) |
| `Report(text)` as a statement of its own | a function, `report(text)`, on the class that needs it | [classes_and_files.md](classes_and_files.md#constructors) |
| `this.name` | `name` | [classes_and_files.md](classes_and_files.md#this) |
| `Console().print(value)`, `Build().program`, `greet(Console())` | `var console = Console()` beside the attributes, then `console.print(value)` | [classes_and_files.md](classes_and_files.md#singletons) |
| `enum Job = { 'knight', 'mage' }` | one entry per line, no `=`, no commas | [values_and_types.md](values_and_types.md#enums) |

The reason is the same everywhere: Spite is written mostly by AI and read by people, and a long way round that the
compiler accepts is a pattern that spreads.

---

Next: [Memory](memory.md), how values are allocated, counted and released.
