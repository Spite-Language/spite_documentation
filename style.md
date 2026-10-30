# Style: the compiler is the formatter and the linter

Style in Spite is not a matter of taste, and there is no style guide to follow: the compiler either rewrites your
code into the one style or refuses it with an error that names the fix. It has no warnings and no switch to turn
a rule off. What cannot be rewritten safely, such as a name or a structure, is an error; everything else is rewritten.

## Why one style, and no warnings

A formatter that offers choices, and a linter that warns, hand every choice back to whoever is writing. Two ways of
doing one thing is a decision somebody makes again at every place it appears, so Spite keeps one: before adding a
way to do something, look for the one that already exists, and when two exist, remove one. Conversions are the
example: `Float.from_integer(count)` and `count.to_float()` both look reasonable, nobody could say which class owns a
conversion, and so only the conversion in the class it makes exists, `func Float(from: Integer): Float`, which the
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
- K&R braces: `func name() {`, `} else {`, `} else if other {`. An empty body is `{ }`.
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
constructor, then functions ([classes_and_files.md](classes_and_files.md#what-a-file-holds)). Anything out of
order is an error naming what came before it:

```gdscript title=declaration_order_error/declaration_order_error.spite entry error
var console = Console()

func DeclarationOrderError() {
    load "../spite_standard_library/windows"
    console.print(label)
}

var label = "late"
```
```diagnostic
but a file is ordered: the singleton line, generic lines, enums, unions, types, variables, the constructor, then functions
```

## Names

A name is never rewritten for you, because renaming a symbol can change what a program means to someone
searching for the old name. These are errors, each with the fix:

- `snake_case` for variables, attributes, parameters, functions, enum values, files and folders (a package's
  root folder too: `window_plugin`, never `window-plugin`); `PascalCase` for classes, enums, unions and
  types.
- **Never a single letter**, except the axis names `x`, `y`, `z` and `w`, which are the names of what they
  hold and abbreviate nothing: `position.x`, `var x = 0.0`.
- **Never an abbreviation.** Every word of a name is checked against a list (`msg`, `idx`, `val`, `cfg`, `tmp`,
  `str`, `len`, `max`, `min`, `init`, `env`, `dir`, `doc`, ...; the full table is in
  [Naming and abbreviations](https://github.com/Spite-Language/spite_spec/blob/master/specs/style.md#naming-and-abbreviations-compile-errors-not-auto-fixed)),
  and the error spells the name out. `id` is allowed.
- **Any word another language reserves is yours too**: `unsigned`, `static`, `stdout` or `near` are ordinary names.
- **One underscore between words**, and at most one in front to make a name private: `hit__count`, `__strike`
  and `strike_` are errors. That is what keeps the names the compiler makes for a class (how it allocates, counts
  and releases one, under names with `___` in them) out of the way of yours, so `allocate`, `make`, `release`
  or `class_of` are ordinary function names.
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
its body ignores: name it `_name` to say so. A `_name` that *is* read is an error, since the prefix says it is
not, and a parameter whose signature is dictated from outside (an operator function's, for one) needs no `_` at
all ([the exemptions](https://github.com/Spite-Language/spite_spec/blob/master/specs/style.md#unused-is-an-error)). Everywhere else a leading `_` means private to its
class ([reflection.md](reflection.md#reflection-is-read-only)).

```gdscript title=unused_error/unused_error.spite entry error
var console = Console()

func UnusedError() {
    load "../spite_standard_library/windows"
    var forgotten = 3
    console.print("hello")
}
```
```diagnostic
'forgotten' is never read: remove it
```

```gdscript title=written_not_read/written_not_read.spite entry error
var console = Console()

func WrittenNotRead() {
    load "../spite_standard_library/windows"
    var total = 1
    total = 2
    console.print("hello")
}
```
```diagnostic
'total' is never read: remove it
```

An attribute nothing reads is an error the same way: remove it. A private `_name` attribute is no
exception, since only its own class can read it. Writing an attribute is not reading it, so one that is only
assigned is still unused. A read is anything that takes the attribute's value: its name in the class's own
functions, `thing.world` from another class, `x.attributes[attribute]` in a template or a walk (which is how the JSON
and binary writers and `to_debug()` read), and the rest [in the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/style.md#unused-is-an-error). A walk that only looks
at `attribute.name` or `attribute.class` reads the attribute's description, not the attribute, so an attribute
kept only as a marker for such a walk is an error:

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
The check covers the program's own folder, the standard library (always compiled whole, so its reads are the
same in every program), and every private attribute wherever it lives, since its own class is its only reader
. A singleton binding is checked everywhere, loaded
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

## Comments are links

A comment is one line, outside functions and declaration bodies, and it holds nothing but a link to a heading in
a markdown file. Prose above a two-line function that explains the compiler instead of the code is the habit this
rule ends, and a note that is needed becomes a section somewhere a reader can open:

```gdscript
# notes.md#why-this-program-exists
func Game() {
```

The path is resolved from the folder of the file the comment is in, as `load` resolves its folder, and the compiler
checks that the file exists and has that heading, so a link cannot rot silently. Prose in a comment, `//` and `/* */` are errors, and each one asks
whether the note is needed at all: if a reader could work it out from the code, delete it; if it is lasting
knowledge, write the section and link to it; if it warns against a change, a test or a compile error pushes
harder than prose. A line inside a function that seems to need explaining becomes a named function instead:
a name, unlike a comment, is visible to reflection, `--final-classes` and every tool.

The one other link is the one `--final-classes` writes: in the folder it prints, each attribute and function is
marked with a comment linking the `.spite` file that supplied it (`# ../../game/hero.spite`), with no anchor, so
the printed classes name where every declaration came from and still compile. Only a file under a folder
`--final-classes` wrote (it holds a `.final-classes` file) may link a `.spite` file; anywhere else it is `this
comment links to 'helper.spite', a .spite file, and only a file '--final-classes' wrote may link one`
(`diagnostics/spite_link_outside_final`).

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

A number inside an expression says nothing about what it means, so it is named first:

```gdscript
var days_per_cycle = 146097
var days = cycles * days_per_cycle + remaining
```

`146097` in the middle of the arithmetic is an error that gives this form. The name costs nothing: the compiler folds
it as it folded the number. `0`, `1` and `-1` need no name, and neither does the number on the right of the `var`
that names it. The rule is in [the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/style.md#a-number-is-named-before-it-is-used).

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
(`"{names.count()} names"`) are not arguments. Every case is
[in the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/style.md#one-call-per-line-and-nothing-said-twice).

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
another constructor's arguments (`Label(Font())`) and a collection made for a call
(`buffers.set(List<String>())`). An object made to be read at once is not an argument: `JsonWriter<Order>().write(order)` is
fine, and so is `var label = Label("new")` itself. `Parallel(worker.run)` passes a function value, not an object.

**An `if` and its `else` do not repeat the same work.** When both branches compute the same call, it is
computed once before the `if`:

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

Write `var total = first + second`, then `total.maximum(5)`. A negative number written out (`(-2.5).floor()`) is a
literal, not a computation, and may be called on.

## The short form is the only form

When the language has a short way to say something, the long way is an error that names it. Each has its own
page:

| Written | Is an error naming | Page |
|---|---|---|
| `if not ready { return }`, `if names.is_empty() { return null }` | `fallback ready`, `fallback not names.is_empty()` | [failure.md](failure.md#an-if-that-only-returns-the-default-is-an-fallback) |
| a last `if value { ... }` with no `else` | `fallback value` | [failure.md](failure.md#the-last-if-of-a-function) |
| `var watcher = tracker` then `fallback watcher` | `fallback tracker` | [failure.md](failure.md#narrowing-a-path) |
| `fallback tracker` on a value that cannot be null | removing it | [failure.md](failure.md#narrowing-a-path) |
| a switch that is one class case and `_:`, each a `return` | `if value == Class` or `return value == Class` | [control_flow.md](control_flow.md#value--class) |
| two switch cases with the same body | `_:` | [control_flow.md](control_flow.md#switch-over-a-union) |
| three `if`s comparing one value with a constant, each only returning | the `switch` it spells | [control_flow.md](control_flow.md#switch-over-values) |
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
