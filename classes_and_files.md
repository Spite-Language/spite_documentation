# Classes and files

Every `.spite` file is exactly one class, named by PascalCasing the file name: `person.spite` is `Person`,
`repository_list.spite` is `RepositoryList`. This cannot be changed: there is no `class` keyword and no way to
put two classes in one file. Every other file of the program is another class, and a folder is a namespace
([packages.md](packages.md)).

## Names

The shape of a name says what it is. `lower_snake_case` names a variable, an attribute, a function or a folder;
`PascalCase` names a class, a `type` or an enum, and the whole standard library is written that way. Text is
`"in double quotes"` and nothing else: single quotes are for enum values (`'mage'`), and a `'text'` written for a
string is refused. A comment starts with `#` and runs to the end of the line. A newline ends a statement and
separates the entries of a list, an enum, a union or a `type`; on one line, commas do.

A `$name` is a codegen value, replaced when the code is generated ([metaprogramming.md](metaprogramming.md)). A
name starting with `_` is private ([below](#private-names)).

The keywords are `var func return if else while switch type enum union generic fallback crash and or not null true
false this`, and none of them can name anything. `load` is a keyword only at the start of a statement, written
without parentheses (`load "engine"`), and it cannot name a function, a variable or a parameter. `singleton` is a
keyword only as the first line of a file, and `type` and `singleton` are never a name, so `var union = 1` stops at
"expected the name of a variable and found 'union'". `generics` is reserved only so that a header list gets an
error that shows the right form: `there is no 'generics' list: a class declares each codegen value on its own line
at the top of the file, like 'generic $damage_type'`.

Words from other languages get an error that shows the Spite form, and stay usable as names:

| Written | Error says |
|---|---|
| `do` | an `if` on a value that may be null narrows it in place, `if value { ... } else { ... }` |
| `let total = 1`, `const` | `Spite has no 'let': a variable is declared 'var name = value'` |
| `class Name {` | `there is no 'class' keyword: a file is a class, named after the file, and its attributes and functions are written at the top of that file` |

Reflection gives every object a few names, so no attribute or function of yours may be called `class`,
`attributes`, `functions`, `instances` or `memory`: one of its own would hide the one reflection gives
([reflection.md](reflection.md)). `class` written bare inside a function is the instance's own `Spite.Class`.

## What a file holds

A file contains only declarations, in this order:

1. a `singleton` line, when the class has one instance ([below](#singletons));
2. `generic $name` lines, one per codegen value a caller supplies, each optionally constrained by a `type`
   (`generic $name: Printable`, [metaprogramming.md](metaprogramming.md#generics-and-codegen-values-));
3. `enum`, `union` and `type` declarations, which belong to the class: plain `Job` inside it, `Player.Job`
   everywhere else ([packages.md](packages.md)). An enum always lives inside the class that uses it: a file
   holding nothing but an enum, a union or a type of its own name is an error; a union or a type is declared in
   the class that consumes it, and one that two classes of a namespace declare moves to that namespace's entry class
   ([packages.md](packages.md)). Other classes give its values by their short form
   (`'mage'`) and write `Player.Job` only where a variable must hold one;
4. `var` declarations: the attributes, each with a default value;
5. the constructor;
6. the other `func`s.

Anything out of that order is an error naming what came before it ([style.md](style.md#a-file-is-ordered)).
There are **no file-level statements**: no loop, `if` or call outside a function. Move the statement into a
function, or into the constructor:

```gdscript title=file_scope_error/file_scope_error.spite entry error
var console = Console()

console.print("not allowed")

func FileScopeError() {
    load "../spite_standard_library/windows"
    console.print("a statement inside a function is fine")
}
```
```diagnostic
a file holds declarations and nothing else
```

A whole file in order, with a codegen value, an enum and a `type`:

```gdscript title=ordered_file/inventory.spite
generic $item_type

enum Sorting {
    'by_name'
    'by_count'
}

type Counted {
    count: Integer
}

var items = List<$item_type>()
var sorting: Sorting = 'by_name'

func add(item: $item_type) {
    items.append(item)
}

func total(): Integer {
    return items.sum_count()
}
```
```gdscript title=ordered_file/crate.spite
var count = 0

func Crate(new_count: Integer) {
    count = new_count
}
```
```gdscript title=ordered_file/ordered_file.spite entry
var console = Console()

func OrderedFile() {
    load "../spite_standard_library/windows"
    var inventory = Inventory<Crate>()
    var small_crate = Crate(3)
    inventory.add(small_crate)
    var large_crate = Crate(4)
    inventory.add(large_crate)
    var total = inventory.total()
    console.print(total, inventory.sorting)
}
```
```output
7 by_name
```

### A union or type two classes declare

Declare a union or a `type` in the class that takes, holds or switches on it, and leave it there. When two classes
of one namespace declare the same union, with the same name and the same members (in any order), the compiler
moves it for you: it goes to the entry class of the namespace, the class named like its folder (`Bunda.Bunda` in
`bunda/bunda.spite`), it creates that file if the namespace has none, and each class that used the union now names
it through the entry class, `Bunda.Snack`. Each move is a `fixed:` line, and the entry class writes the plain name.
A union declared by classes in different folders goes to the lowest namespace that holds every copy, and an outside
class that needs one declares a copy of its own, which the fix then moves up with the others. Only the same name
with the same content counts: the same members under another name are another union, since they may mean another
thing, and a union with fewer members is a different one too.

Nothing moves between the program, a package and the library; each keeps its own. The compiler refuses, with
every declaring class named, what it cannot move: two different unions of one name landing in one entry class, a
name that already means a class there (a union `Bunda` repeated inside `bunda/`), an entry class that is generic,
a copy in files the compiler does not write (the library, a pinned package), and an entry class whose new file
would take a name the namespace already reaches. The fix for the refusal is a rename or moving the union by hand.
A class outside the namespace cannot write the plain name, and neither can a side class of the same namespace:
`'Pet' is a union of 'Bunda.Bunda', the entry class of this class's namespace, and only that class names it
plainly: write 'Bunda.Pet'`.

## Constructors

A function named exactly like its class is the constructor:

```gdscript title=person_basics/person.spite
var age = 0

func Person(new_age: Integer) {
    age = new_age
}
```
```gdscript title=person_basics/person_basics.spite entry
var console = Console()

func PersonBasics() {
    load "../spite_standard_library/windows"
    var person = Person(30)
    console.print("age", person.age)
}
```
```output
age 30
```

A class with no constructor is made from its defaults with `Member()`, and writing an empty one is an error,
because it says nothing. A generic class needs no constructor either:

```gdscript title=team_roster/member.spite
var name = "unnamed"
var level = 1
```
```gdscript title=team_roster/team_roster.spite entry
var console = Console()

func TeamRoster() {
    load "../spite_standard_library/windows"
    var members = List<Member>()
    var first_member = Member()
    members.append(first_member)
    var second_member = Member()
    members.append(second_member)
    var members_count = members.count()
    console.print("count", members_count)
    crash members.count() == 2
    console.print("first name", members[0].name, "level", members[0].level)
}
```
```output
count 2
first name unnamed level 1
```

```gdscript title=empty_constructor_error/holder.spite
var held = 0

func Holder() { }
```
```gdscript title=empty_constructor_error/empty_constructor_error.spite entry error
var console = Console()

func EmptyConstructorError() {
    load "../spite_standard_library/windows"
    var holder = Holder()
    console.print(holder.held)
}
```
```diagnostic
'Holder' has an empty constructor: delete it, since a class without one is made from its defaults
```

Every `var` is written with its default, so an object made with no arguments is already complete: a fresh
`Member()` has every field at its declared default. There is no `null` for anything but a `T?`
([failure.md](failure.md)). A constructor is setup, not logic: `fallback` is not allowed in one, and `crash` is how
it says something is a bug ([failure.md](failure.md#fallback-is-not-allowed-in-a-constructor)). There is no `new`:
`Person(30)` makes one.

There is one function per name. A file declaring a name twice is an error, `'DuplicateFunction' declares 'twice'
twice: a later file may reopen a class and replace a declaration, but one file declares each name once`, and there
is no overloading: an argument is cast to the parameter's type instead
([functions_and_operators.md](functions_and_operators.md)).

A constructed object is kept and used. A constructor call written as a statement of its own, `Report(text)`,
makes an object and throws it away. The construction was the whole point, so it is an error saying to write a
function instead. `Box<Integer>(3)` on a line of its own is the same error. An object stored in a variable that
nothing reads is already the unused-name error, and a singleton is exempt, since it is bound as an attribute:

```gdscript title=dropped_report/report.spite
var console = Console()

func Report(text: String) {
    console.print("report: {text}")
}
```
```gdscript title=dropped_report/dropped_report.spite entry error
func DroppedReport() {
    load "../spite_standard_library/windows"
    Report("the door opens")
}
```
```diagnostic
'Report("the door opens")' makes a 'Report' and drops it: a constructed object must be kept and used, so a class whose construction is the whole point should be a function instead
```

## Constructors, conversions and presets

A class is built in three ways, and each has a fixed shape so it can't be mistaken for another:

| Kind | Shape | Who calls it |
|---|---|---|
| **constructor** | named after the class, no return type, fills `this` | the program: `Meters(42195)` |
| **conversion** | named after the class, one parameter named `from`, returns the class or its `?` | the compiler, wherever a value of the `from` type meets the class |
| **preset** | a PascalCase name of its own, no parameters, returns exactly the class | the program: `Meters.Marathon()` |

```gdscript title=distances/meters.spite
var amount = 0

func Meters(starting_amount: Integer) {
    amount = starting_amount
}

func Meters(from: String): Meters? {
    var read: Integer? = from
    fallback read
    return Meters(read)
}

func Marathon(): Meters {
    return Meters(42195)
}
```
```gdscript title=distances/distances.spite entry
var console = Console()

func Distances() {
    load "../spite_standard_library/windows"
    var run = Meters.Marathon()
    console.print(run.amount)
    var walked: Meters? = "1200"
    crash walked
    console.print(walked.amount)
}
```
```output
42195
1200
```

**A conversion** lives in the class it builds, so `String` knows nothing of `Meters`, `IpAddress` or `Color`. Its
`?` says it can fail: `Meters(from: String): Meters?` answers `null` for text that isn't a number, and the caller
narrows it. A conversion without `?` can't fail. A class has at most one conversion from each type. It is never
called by name: where a `String` is given and a `Meters?` is wanted, the compiler calls it, and writing the call
yourself is an error. `var walked: Meters? = Meters("1200")` is refused with "'Meters("1200")' is what writing
'"1200"' here already does: the compiler casts a value to the class its place wants, so write '"1200"'", and
`var walked = Meters("1200")`, where nothing wants a `Meters`, asks for a place that does, `var converted: Meters? =
"1200"`. The same holds for numbers and enums: `Float(count)` and `Mood(text)` are written `count` and `text` where a
`Float` or a `Mood?` is wanted. An enum or a union declared in a class gets its conversions in that class, named by the type
alone (`func Job(from: String): Job?`). Text already converts to an enum by its values with nothing written.

**A preset** names a special value of its class: `Float.Pi()`, `Integer.Largest()`. It builds a fresh
value on each call, which the compiler folds, so nobody can change a shared one. Its body is a single `return` of
literals, other presets or a constructor call on literals: it reads nothing and does no work but building.

A conversion and a preset answer with `return` and never assign `this`. Only these three shapes build a class.

## Singletons

A file whose first line is `singleton` has one instance: calling its constructor anywhere answers that same
instance, made the first time it is asked for (one instance *per argument values* when the constructor takes
arguments, [below](#one-instance-per-argument-values)). `Console`, `Program`,
`Environment`, `Build` and `Clock` are singletons; `File`, `Directory` and `Process` are not, since
several may exist at once.

```gdscript title=singleton_basics/scoreboard.spite
singleton

var points = 0

func score(amount: Integer) {
    points = points + amount
}
```
```gdscript title=singleton_basics/singleton_basics.spite entry
var console = Console()
var home = Scoreboard()
var away = Scoreboard()

func SingletonBasics() {
    load "../spite_standard_library/windows"
    home.score(2)
    away.score(3)
    var is_singleton = Scoreboard.is_singleton()
    console.print(home.points, home == away, is_singleton)
}
```
```output
5 true true
```

Call sites never change: `var console = Console()` reads the same whether or not the class is a singleton, and
the class file is where that is said. `is_singleton()` is answered by `Spite.Class` for every class, from the
line. Writing a `func is_singleton()` in a class of your own is an error that tells you to write `singleton` as
the file's first line. A later file may reopen a class and add the `singleton` line, and the class is then a
singleton everywhere.

A singleton lives until the program ends, and its `drop()` runs then, newest singleton first, so a `drop()` may
use any singleton its class keeps in an attribute. A singleton counts as made when its constructor finishes, so the
ones its attributes make are made before it and outlive it. At exit the calls a frame loop started finish first,
the output is flushed, the singletons are destroyed, and last of all every `DynamicLibrary` is unloaded, so a
`drop()` may call a library. A `drop()` that fetches a singleton made after its own, which is already gone, halts
with `spite: a drop() at exit used the singleton Archive after it was destroyed`, and says to keep that singleton
in an attribute of the one whose `drop()` uses it. Nothing is read from freed memory either way: a singleton's
memory is given back only after all of them are destroyed. What one costs at run time is a static slot, filled the first
time it is asked for: fetching it is one load, it is never reference counted, and a program never pays for a
singleton it does not reach ([optimizations.md](optimizations.md#a-singleton-is-made-on-first-use-by-a-function-of-its-own)).
Work the compiler runs at once never shares a singleton one of them writes, so a singleton needs nothing written
for threads ([concurrency.md](concurrency.md#no-lock-to-write)). The sections below
give the rest.

### One instance per argument values

**A singleton constructed with different argument values is a different instance**. `Singleton(1)` and
`Singleton(2)` are two objects; `Singleton(1)` asked for again, anywhere in the program, is the first one. So a
singleton is not "one object of this class" but "one object per value of its arguments": the arguments are the
instance's name. Two bindings with the same values are the same object, and a write through one is seen through
the other; bindings with different values share nothing but the class.

```gdscript title=radio/channel.spite
singleton

var number = 0
var messages = List<String>()

func Channel(channel_number: Integer) {
    number = channel_number
}
```
```gdscript title=radio/radio.spite entry
var console = Console()
var news = Channel(1)
var music = Channel(2)
var news_again = Channel(1)

func Radio() {
    load "../spite_standard_library/windows"
    news.messages.append("storm tonight")
    music.messages.append("song")
    news_again.messages.append("roads closed")
    var news_count = news.messages.count()
    var music_count = music.messages.count()
    console.print(news == news_again, news == music, news_count, music_count)
}
```
```output
true false 2 1
```

`news` and `news_again` are one channel, so both messages land in it; `music` is a second channel with its own
list. The constructor runs once per argument values, the first time those values are asked for, and each instance
lives until the program ends like any singleton.

**The arguments are literals** the compiler reads while compiling: strings, numbers (a leading minus
included, decimals too), `true`, `false` and enum values. Each distinct argument list is its own instance, made once and found
again without looking anything up at run time. Anything else is an error that names the rule:
`var registry = Registry(starting_name)` is "'starting_name' is not a literal: a singleton's arguments are literals
the compiler reads, because each distinct argument list is its own instance of 'Registry', made the first time it
is asked for".

Only a generic class takes a type in `<>`. An enum, a union, a `type` or a number written with one is refused:
`var phase: Phase<Integer> = 'idle'` is "'Phase' is not a generic class, so it takes no type in '<>': write 'Phase' alone".

A singleton with `generic` lines has one instance per set of codegen values: `Column<Health>()` is the same object
everywhere, and `Column<Label>()` is a second one. This is what per-type storage is written with.

```gdscript title=generic_singleton/column.spite
singleton

generic $component_type

var values = List<$component_type>()
```
```gdscript title=generic_singleton/generic_singleton.spite entry
var console = Console()
var numbers = Column<Integer>()
var again = Column<Integer>()
var words = Column<String>()

func GenericSingleton() {
    load "../spite_standard_library/windows"
    numbers.values.append(1)
    again.values.append(2)
    var number_count = again.values.count()
    var word_count = words.values.count()
    console.print(numbers == again, number_count, word_count)
}
```
```output
true 2 0
```

A singleton is bound once, as an attribute beside the others, and used through that name, so every singleton a
class uses is visible at the top of its file. Binding it as a local inside a function is an error, `'Build' is a
singleton, bound here as a local: bind it once beside the attributes`. The one exception is a class that is a
value, `String`, the numbers, `List` and `Dictionary`: an attribute there would be carried by every text or list,
so a function you add to one of them binds the singleton as a local `var`, and it is made when the object that
binds it is made, not when the function first runs. A generic singleton whose codegen values come from the
function's own metaprogramming (`Query<argument.class>()`) stays a local too, since no attribute can name that type.

Reading or calling a member on the constructor call itself is an error, and so is passing it (`greet(Console())`),
returning it, assigning it, using it in an operation, writing it as a statement of its own (a bare `Console()`) or
storing it anywhere but a `var` of its own. The error names the fix. It covers a generic singleton such as
`Column<Health>()` the same way, and only in code the compiler generates, so a function nothing calls is not
checked. A binding nothing reads is an error everywhere, a loaded package included: `the attribute 'world' is never
read: remove it`.

```gdscript title=singleton_inline_error/singleton_inline_error.spite entry error
var console = Console()

func SingletonInlineError() {
    load "../spite_standard_library/windows"
    console.print("built for {Build().target_operating_system}")
}
```
```diagnostic
'Build' is a singleton: bind it once beside the attributes, 'var build = Build()', and use 'build.target_operating_system'
```

Singletons never wait for each other in a circle. When the attributes of singletons make one another, directly,
through a generic singleton or through an ordinary object whose attributes bind one, the compiler cannot finish
making any of them, and it says so at the attribute that begins the circle: `singleton 'World' binds 'Column<Entity>',
which binds 'World': singletons initialise each other in a circle, so one of them has to reach the other some other
way`. ("makes" replaces "binds" where the link is an ordinary object.) A circle only a constructor body closes is
not followed at compile time; it halts the program when it runs, naming the singletons, as `spite: singleton
'Registry' binds 'Catalog', which binds 'Registry': singletons initialise each other in a circle`; one singleton reaching itself reads `singleton 'Solo' binds
'Solo'`. The fix is to bind one of them in the class that uses both, or to reach it some other way. A program that starts threads still
makes a singleton once, whichever thread asks first.

## Classes are references

A scalar (`Integer`, `Float`, `Boolean`, an enum value) is passed by value, copied at the call site. Everything else
(a class instance, `List<T>`, `Dictionary<Key, Value>`, `String`, a union, an object literal) is a reference: passing one
shares the exact same object, so a function can change it and the caller sees the change. There is nothing
special to write at the call site or in the parameter's type ([memory.md](memory.md)):

```gdscript title=level_up/member.spite
var name = "unnamed"
var level = 1

func Member(new_name: String) {
    name = new_name
}
```
```gdscript title=level_up/level_up.spite entry
var console = Console()

func LevelUp() {
    load "../spite_standard_library/windows"
    var hero = Member("Aria")
    bump_level(hero)
    console.print("name", hero.name, "level", hero.level)
}

func bump_level(member: Member) {
    member.level = member.level + 1
}
```
```output
name Aria level 2
```

Two names can hold the same object, and a change through either shows up through the other. `copy()` makes an
independent object with the same field values when that sharing is not what you want
([memory.md](memory.md#do-call-copydeep_copy-for-an-independent-object)):

```gdscript title=shared_member/member.spite
var level = 1
```
```gdscript title=shared_member/shared_member.spite entry
var console = Console()

func SharedMember() {
    load "../spite_standard_library/windows"
    var original = Member()
    var alias = original
    var independent = original.copy()
    alias.level = 5
    independent.level = 9
    console.print("original level", original.level)
    console.print("independent level", independent.level)
}
```
```output
original level 5
independent level 9
```

## `this`

A class reads and calls its own members by name: `level`, `score(2)`. There is no `self.` to write, and writing
`this.level` is an error that says `level`. What `this` is for, in every class, is handing the instance itself
to something else or returning it:

```gdscript title=this_basics/registry.spite
var names = List<String>()

func record(drink: Drink) {
    names.append(drink.name)
}
```
```gdscript title=this_basics/drink.spite
var name = ""

func Drink(new_name: String) {
    name = new_name
}

func join(registry: Registry) {
    registry.record(this)
}
```
```gdscript title=this_basics/this_basics.spite entry
var console = Console()

func ThisBasics() {
    load "../spite_standard_library/windows"
    var registry = Registry()
    var tea = Drink("tea")
    tea.join(registry)
    var coffee = Drink("coffee")
    coffee.join(registry)
    var joined_names = registry.names.join(", ")
    console.print(joined_names)
}
```
```output
tea, coffee
```

Inside a function of a number class, `this` is the number itself ([values_and_types.md](values_and_types.md#numbers-are-classes)).
Likewise `class`, written bare inside any function, is the instance's own `Spite.Class`
([reflection.md](reflection.md)).

## Printing a class

`console.print` writes each value's `to_string()`, so a class prints once it says how, and printing one that does
not is a compile error naming `to_string`. `console.debug` needs nothing: every class has a `to_debug()` that shows
its name and attributes ([standard_library.md](standard_library.md#console)).

```gdscript title=printing_a_class/lamp.spite
var room = ""
var lit = false

func Lamp(starting_room: String) {
    room = starting_room
}

func to_string(): String {
    return "the {room} lamp"
}
```
```gdscript title=printing_a_class/printing_a_class.spite entry
var console = Console()

func PrintingAClass() {
    load "../spite_standard_library/windows"
    var lamp = Lamp("hall")
    console.print(lamp)
    console.debug(lamp)
}
```
```output
the hall lamp
Lamp { room: "hall", lit: false }
```

## Private names

A name starting with `_` is private: it is read or called only inside its own class (a reopening of
the class counts as inside), and using it from anywhere else is an error that names the getter when there is
one: `'_hidden' is private to 'Secret': a name starting with '_' is used only inside its own class, and 'hidden'
reads it from outside`. A private function with no getter gets the same error without that last clause. On a parameter, and only there, the prefix says
instead that the body ignores it on purpose; a private attribute nothing reads is an error like any other, and
no spelling silences it ([style.md](style.md#nothing-unused)).

## Reopening a class

A second file with the same name, in a later loaded root, **reopens** the class instead of colliding with it:
its functions and attributes replace the earlier ones of the same name and add the rest. That is how a program
adds settings to `Environment`, a member template to `List`, a function to `Integer`, and how a mod changes a game
([packages.md](packages.md#monkey-patching-mods)). The program's entry class is the one class nothing may
reopen, and it cannot declare a `generic` line, since it is the class that runs.

An attribute replaced this way keeps its type: a reopening changes its default, never its type, however each file
gives it (`var level: Integer = 1`, then `var level = "one"`, is `'level' is an Integer where the class first
declares it, and this reopening makes it a String: a reopening replaces an attribute's default, never its type, so
give the new attribute a name of its own`). The check runs on the program's own reopenings of a library class as
well, and, like the other rules that refuse code that would otherwise build, it is told once the program is built,
after any mistake of a type or a name. A reopening may also add the `singleton` line.

Reopening a class of the standard library adds what it lacks; it does not copy what it has. A function that
rewrites the object from two values of its own class, beside the library's `*`, is an error that tells you to write
`target = left * right`: the compiler builds that answer in `target`'s place. A function that only reads the object,
or that takes no other value of its class, stays legal.

## There is no `static`

Other languages need `static` for a function that belongs to the class and not to an instance. In Spite a class is
itself an instance of `Spite.Class` ([reflection.md](reflection.md)), so a function that works on the class is an
ordinary member of that class object: `Spite.Class` declares it, and the class file overrides it. There is nothing
for a keyword to say, so there is no keyword.

## A class name means one class

A class may not hide another class. If `physics/plugin.spite` (`Physics.Plugin`) sat beside a root
`plugin.spite` (`Plugin`), the name `Plugin` would mean one class inside `Physics` and the other everywhere else,
and a reader would have to work out which one each line reaches. So that is an error naming both, asking for a
clearer name: `the class 'Physics.Plugin' hides the class 'Plugin': inside 'Physics' the name 'Plugin' would mean
this class, so give one of them a clearer name`. It holds against the classes of a package the program loads, between
two packages, and against the standard library (`game/tcp/connection.spite` would hide the library's `Tcp.Connection`). Two classes of one name in namespaces side by side, `Physics.Plugin` and `Audio.Plugin`, hide
nothing, and neither does an `enum`, a `union` or a `type` declared inside a class, which is named through that
class.

A class of your own named like a library class is an error naming the library's class, since a `Vector3` or a `Color`
of your own is most likely one the library already has. To add what the library's class lacks, reopen it. A
library class that is always written with its namespace, `Spite.Function` or `Spite.Call`, cannot be mistaken for
yours, so a `Function` or a `Call` of your own is fine: the name is refused only where a reader could not tell at
first glance which class it means.

Names that all start with the same word say the same thing: `HighBlock`, `HighFunction`, `HighValue` are a
namespace written into each name. Make it a folder instead, `high/block.spite` and the rest, and the classes are
`High.Block`, `High.Function` and `High.Value` outside it and `Block`, `Function` and `Value` inside it, where a
reader already knows which namespace they are in. Three or more such names in one folder are an error naming the
folder to move them into, listing the first three and `(and 9 more)` for the rest. The shared word must be a whole
word of each name: `Higher` and `Highway` beside `HighBlock` and `HighValue` are not four matches, and two matches
are allowed. A class named for the word alone, `High`, counts as one of them. A class named for its own folder
(the folder's entry class, `backend/backend.spite` beside `BackendBlock` and `BackendValue`) does not count: its name
is the namespace itself, not a word repeated in it. And name the namespace for what it holds: `High` alone does not
say high what.

```gdscript title=own_vector/geometry/vector3.spite
var height = 0.0
```
```gdscript title=own_vector/own_vector.spite entry error
var console = Console()

func OwnVector() {
    load "../spite_standard_library/windows"
    var point = Geometry.Vector3()
    console.print(point.height)
}
```
```diagnostic
the class 'Geometry.Vector3' is named like the standard library's class 'Vector3': use 'Vector3', reopening it to add what it lacks, or give this class a clearer name
```

---

Next: [Programs](programs.md), the entry file, how a program loads the standard library, and its settings.
