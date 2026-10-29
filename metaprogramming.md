# Metaprogramming

There are no macros. **Every class is an instance of `Spite.Class`**, every namespace an instance of
`Spite.Namespace` and every function an instance of `Spite.Function`, so "do this for every attribute" is ordinary
code over ordinary objects: `Monster.attributes.each(show)`. The compiler knows those objects while compiling, so
it runs that code there: a walk is unrolled into one call per attribute, each call compiled for its own attribute
and typed by it, and nothing is looked up when the program runs.

Two more tools complete it. A **template** is a function whose name says what it is given, like `show_health(troll)`
or `filter_alive()`: a parameter named by a word of the function's own name, typed `Spite.AttributeDeclaration`,
`Spite.FunctionDeclaration` or `Spite.Argument`, says what the function ranges over, and the compiler writes one copy
for each name a program calls. A **generic** is a class that takes codegen values, `List<Integer>`. All three are
ordinary functions and classes, so what they make is typed, visible in `--final-classes`, and removed when nothing
calls it ([Tree shaking](#tree-shaking)).

```gdscript title=walks_and_templates/monster.spite
var name = "troll"
var health = 10

func Monster(new_name: String, new_health: Integer) {
    name = new_name
    health = new_health
}

func alive(): Boolean {
    return health > 0
}
```
```gdscript title=walks_and_templates/walks_and_templates.spite entry
var console = Console()

func WalksAndTemplates() {
    load "../spite_standard_library/windows"
    Monster.attributes.each(describe)
    var troll = Monster("troll", 10)
    show_health(troll)
    var monsters = List<Monster>()
    monsters.append(troll)
    var ghost = Monster("ghost", 0)
    monsters.append(ghost)
    var living = monsters.filter_alive()
    var total = living.sum_health()
    var living_count = living.count()
    console.print("alive", living_count, "with", total, "health")
}

func describe(attribute: Spite.AttributeDeclaration) {
    console.print("Monster has", attribute.name, "of class", attribute.class)
}

func show_attribute(attribute: Spite.AttributeDeclaration, monster: Monster) {
    console.print(monster.name, attribute.name, monster[attribute])
}
```
```output
Monster has name of class String
Monster has health of class Integer
troll health 10
alive 1 with 10 health
```

`describe` is compiled twice, once per attribute, and the `each` is two calls to those copies. `show_health` is the
template `show_attribute` written for `health`: `monster[attribute]` is the attribute of that name read on the monster
it is given, so the body is what tells the compiler the template ranges over `Monster`. `filter_alive` and `sum_health`
are the standard library's member templates written for `alive` and `health`. The objects and every member they answer
are [reflection.md](reflection.md)'s; this page is what a program builds with them.

## Walking a program's structure

A walk is a call on a list of reflection objects. Every list a reflection object has is an ordinary `List`, so a
walk is `each`, a selection is a filter or a member template, and a member by name is `[]`, which answers it or
`null`:

| To reach | Write |
|---|---|
| every attribute of a class | `Monster.attributes.each(show)` |
| every class of a folder | `Shop.Tools.classes.each(make)`, with `make(part: Spite.Class)` |
| every value of an enum | `Phase.values.each(run)`, with `run(phase: Phase)` |
| one function, if the class has it | `Runner.functions['run_each']`, a `Spite.FunctionDeclaration?` narrowed by `if` or `fallback` |
| the same function bound to an instance | `runner.functions['run_each']`, a `Spite.Function?` |
| a call filled argument by argument | `Spite.Call(Runner.functions['run_each'], runner)`, then `call.arguments['name'] = value` and `call.call()` |
| the functions whose names end in `_each` | `Runner.functions.filter(walks_each)`, or `Runner.functions.filter_name_ends_with("_each")` to count or to ask |
| a function's arguments | `function.arguments.each(describe_taken)` |
| an enum's values, in order | `Phase.values` |
| every folder named `part`, anywhere | `Spite.Namespace.instances.filter(holds_parts)`, then `namespace.classes` |
| every class of the program | `Spite.Class.instances` |

A name is selected, never built from text: `[]` takes a name written in the code, and anything else is a filter
over the names that exist. Each of these is a constant while compiling, so a walk is unrolled and the function it
calls is compiled once for each element ([reflection.md](reflection.md#known-while-compiling)). A function whose
result is typed by its reflection parameter answers a different class in each copy:

```gdscript title=walked_arguments/runner.spite
var total = 0

func update_each(amount: Integer, label: String) {
    total = total + amount + label.length()
}

func render_each(scale: Float) {
    if scale > 1.0 {
        total = total + 1
    }
}

func reset() {
    total = 0
}
```
```gdscript title=walked_arguments/walked_arguments.spite entry
var console = Console()

func WalkedArguments() {
    load "../spite_standard_library/windows"
    var phases = Runner.functions.filter(walks_each)
    phases.each(describe)
    var count = Runner.functions.count_name_starts_with("re")
    console.print(count, "functions start with re")
}

func walks_each(function: Spite.Function): Boolean {
    return function.name.ends_with("_each")
}

func describe(function: Spite.Function) {
    console.print(function.name)
    function.arguments.each(describe_taken)
}

func describe_taken(argument: Spite.Argument) {
    var fresh = made(argument)
    console.print(" ", argument.index, argument.name, "starts as '{fresh}'")
}

func made(argument: Spite.Argument): argument.class {
    return argument.class()
}
```
```output
update_each
  0 amount starts as '0'
  1 label starts as ''
render_each
  0 scale starts as '0'
2 functions start with re
```

`filter(walks_each)` keeps the functions whose test answers true, and a walk over what it kept is unrolled like a walk over
`Runner.functions` itself: the test still runs when the program does, but the compiler writes one `describe` for each
function the class has. `count_name_starts_with("re")` counts `render_each` and `reset`. A walk over nothing walks nothing: a class with no
attributes, or a filter that keeps none, calls the function zero times. The walker is called `describe_taken`, not
`describe_argument`: a parameter named by a word of its function's name makes the function a template
([below](#templates-when-the-name-says-what-it-is-given)), so a plain walker leaves that word out of its name.

### Asking a question while compiling

Every question a reflection object answers is a get-only attribute, so asking one is reading it, and on a constant
the answer is a `true` or `false` in the generated code. An `if` on it keeps only the branch taken:

```gdscript title=asking_functions/loader.spite
var loaded = List<String>()

func load_each(path: String, into: List<String>) {
    var file = File(path)
    var text = file.read_as_string()
    fallback text
    into.append(path)
    loaded.append(text)
}

func tick_each(amount: Integer) {
    loaded.append("tick {amount}")
}
```
```gdscript title=asking_functions/asking_functions.spite entry
var console = Console()

func AskingFunctions() {
    load "../spite_standard_library/windows"
    var loading = Loader.functions['load_each']
    if loading {
        console.print("load_each can wait:", loading.is_resumable)
        console.print("it changes 'into':", loading.arguments[1].is_mutated)
        console.print("it changes 'path':", loading.arguments[0].is_mutated)
    }
    var ticking = Loader.functions['tick_each']
    if ticking {
        console.print("tick_each can wait:", ticking.is_resumable)
    }
    console.print("a Loader has state:", Loader.is_stateful)
}
```
```output
load_each can wait: true
it changes 'into': true
it changes 'path': false
tick_each can wait: false
a Loader has state: true
```

`is_resumable` is true when a function can reach a wait (here the file read), so it is also compiled as a state
machine that pauses there; an engine asks it to start such a system as a `Concurrent` it polls between frames and
to call every other system directly ([concurrency.md](concurrency.md)). `is_mutated` follows every call, however deep:
`into.append(path)` changes the list passed as `into`, and nothing changes `path`.

What a function does with each of its attributes and arguments is one dictionary, keyed by name, whose values are
`Spite.Access`es, and a dictionary takes the member templates over its values:

```gdscript
var update = Movement.functions['update_each']
fallback update
var written = update.accesses.filter_written()
var only_read = update.accesses.count_read()
```

**A question that means a program is wrong is a `crash`**, and a `crash` whose condition is decided while compiling
and is false is a compile error where the program reaches it ([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/metaprogramming.md#codegen-values-)). A library writes the
rule it keeps, and a class that breaks it fails the build at the line that breaks it, not the run:

```gdscript
func run_between_frames(system: $system_type) {
    var update = $system_type.functions['update_each']
    crash update
    crash not update.arguments[0].is_mutated
}
```

In a generic class, `$T` is a constant too, so a question on it folds for each class the generic is given, and the
branch it rules out may call what that class does not have:

```gdscript title=generic_questions/caller.spite
generic $target_type

func call(target: $target_type): String {
    var run = $target_type.functions['run_each']
    if run {
        target.run_each(3)
        return "{$target_type.name} runs"
    }
    return "{$target_type.name} has no run_each"
}
```
```gdscript title=generic_questions/walker.spite
var steps = 0

func run_each(amount: Integer) {
    steps = steps + amount
}
```
```gdscript title=generic_questions/rock.spite
var weight = 3
```
```gdscript title=generic_questions/generic_questions.spite entry
var console = Console()

func GenericQuestions() {
    load "../spite_standard_library/windows"
    var walker = Walker()
    var walkers = Caller<Walker>()
    var walked = walkers.call(walker)
    console.print(walked, walker.steps)
    var rock = Rock()
    var rocks = Caller<Rock>()
    var rocked = rocks.call(rock)
    console.print(rocked, rock.weight)
}
```
```output
Walker runs 3
Rock has no run_each 3
```

`Caller<Rock>` keeps only its last `return`, so `target.run_each(3)` is never compiled for a `Rock`. The question is
kept in a `var` and tested by name (`if run {`), which narrows it like any `T?`.

## Templates: when the name says what it is given

A walk is for "every member". A **template** is for one member named at the call, where the name reads as well as
an argument would: `show_health(troll)` reads like `show(troll.health)`, and `filter_alive()` like "those that are
alive". A parameter whose name is a word of its own function's name makes the function a template, and the parameter's
type says what it ranges over:

| Parameter type | The function ranges over | Inside, the parameter is |
|---|---|---|
| `Spite.AttributeDeclaration` | the attributes of a class (and its functions that take no arguments) | the constant declaration of that member |
| `Spite.FunctionDeclaration` | the functions of a class whose names fit a pattern | the constant declaration of that function |
| `Spite.Argument` | the arguments of a function | the constant argument |

There is no `Symbol<...>` to write: the name of the parameter is the signal, and the context says the rest.
`show_attribute(attribute: Spite.AttributeDeclaration, monster: Monster)` answers `show_name` and `show_health`, one
copy for each name a program calls. Inside, `attribute` is the constant declaration of that member (the instance
comes as its own argument), so `attribute.class` is its type, and `monster[attribute]` is that member of the
`Monster` passed in, read or written; `monster[attribute]` is short for `monster.attributes[attribute]`, and both
work. The member may also be a function that takes no arguments: the read is then a call to it.

The **class a template walks is not written on the parameter: the body says it**. It is the class of the value the
body reads the declaration on, so `monster[attribute]` makes this a template over `Monster`, and a body that reads it
on a `Label` and a `Monster` walks the attributes they both have. The value must be known from its declaration: a
parameter, an attribute, or a local declared with its type or made by a constructor. A value whose class is not
known that way, or a body that reads the declaration on nothing, is an error that says what to write
([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/metaprogramming.md#templates)).

```gdscript title=name_templates/label.spite
var text = "fragile"
var copies = 2

func urgent(): Boolean {
    return copies > 1
}
```
```gdscript title=name_templates/name_templates.spite entry
var console = Console()

func NameTemplates() {
    load "../spite_standard_library/windows"
    var label = Label()
    show_text(label)
    show_urgent(label)
    double_copies(label)
    show_copies(label)
}

func show_attribute(attribute: Spite.AttributeDeclaration, label: Label) {
    console.print(attribute.name, "is", label[attribute])
}

func double_attribute(attribute: Spite.AttributeDeclaration, label: Label) {
    label[attribute] = label[attribute] * 2
}
```
```output
text is fragile
urgent is true
copies is 4
```

A template over a class's own members is how a class answers `set_age`, `get_age` and the rest without writing
each one. Inside the class, `attributes[attribute]` is the template's own class. An exact function always wins over a
template, for that one name and direction:

```gdscript title=own_accessors/person.spite
var age = 0
var name = ""

func set_attribute(attribute: Spite.AttributeDeclaration, value: attribute.class) {
    attributes[attribute] = value
}

func get_attribute(attribute: Spite.AttributeDeclaration): attribute.class {
    return attributes[attribute]
}

func set_name(new_name: String) {
    name = new_name.upper_case()
}
```
```gdscript title=own_accessors/own_accessors.spite entry
var console = Console()

func OwnAccessors() {
    load "../spite_standard_library/windows"
    var person = Person()
    person.name = "ann"
    person.age = 30
    console.print(person.name, person.age)
}
```
```output
ANN 30
```

Writing `person.name` goes through `set_name`, while reading it goes through the `get_attribute` template, since no
exact `get_name` exists. Inside the class, `attributes[attribute]` is that member of the object the function runs
on, read or written directly, so the template never calls itself.

A parameter of the bound types, `Spite.Attribute` (an attribute bound to one object: its `declaration`, its `value`
and its `owner`) or `Spite.Function`, makes no template. It is an ordinary parameter, and the function is an
ordinary function that a walk over an instance's members calls, or a caller hands a value to. A template is written
over the declaration, with the object as a parameter of its own.

### Templates over functions

The same idea ranges over the functions of a class. A parameter typed `Spite.FunctionDeclaration` and named by a
word of its function's name makes the function a template over the functions of a class whose names have that word as
a hole, and the class and the pattern come from the call the body makes with the word in its name:

```gdscript title=function_templates/mover.spite
var steps = 0

func update_each(amount: Integer) {
    steps = steps + amount
}

func render_each(amount: Integer) {
    steps = steps + 10 * amount
}

func reset() {
    steps = 0
}
```
```gdscript title=function_templates/runner.spite
generic $system_type

var console = Console()
var system = $system_type()

func run_all(amount: Integer) {
    run_phases_each(amount)
}

func run_one(amount: Integer) {
    run_render_each(amount)
}

func run_phase_each(phase: Spite.FunctionDeclaration, amount: Integer) {
    console.print("running", phase.name, "for", phase.hole, "at place", phase.index)
    system.phase_each(amount)
}
```
```gdscript title=function_templates/function_templates.spite entry
var console = Console()

func FunctionTemplates() {
    load "../spite_standard_library/windows"
    var all = Runner<Mover>()
    all.run_all(2)
    console.print(all.system.steps)
    var one = Runner<Mover>()
    one.run_one(2)
    console.print(one.system.steps)
}
```
```output
running update_each for update at place 0
running render_each for render at place 1
22
running render_each for render at place 1
20
```

`system.phase_each(amount)` is the call that says the pattern: the functions of the system's class named
`<something>_each`, with `phase` standing for the part that varies. `run_phases_each` runs the template for every
function that fits, in the order they were made, and `run_render_each` for the one named. The hole is any part of the
name between the pattern's fixed words, with no enum behind it, so `phase_each` would also walk a `sum_each` the class
happened to have; a name that fits nothing (`run_rendr_each`) is an error that lists the functions that fit. Inside the
copy, `phase.name` is the function's name (`update_each`), `phase.hole` the text that took the place of `phase`
(`update`), and `phase.index` its place; `phase.argument_count()` stays. A class can also name the pattern as text on a
class value: `$system_type.argument_count("<phase>_each")`.

### Templates over arguments

A parameter typed `Spite.Argument` ranges over the arguments of a function. A template over arguments is called as the
**plural** of its name, and it is spread into the call of the function it fills:

```gdscript title=argument_templates/mover.spite
var console = Console()

func update_each(amount: Integer, label: String, scale: Float) {
    console.print("update", amount, "'{label}'", scale)
}
```
```gdscript title=argument_templates/filler.spite
generic $system_type

var console = Console()
var system = $system_type()

func run(start: Integer) {
    system.update_each(made_arguments(start))
}

func made_argument(argument: Spite.Argument, start: Integer): argument.class {
    console.print("making", argument.index, argument.name, "as", argument.class)
    if argument.class == Integer {
        return start
    }
    return argument.class()
}
```
```gdscript title=argument_templates/argument_templates.spite entry
func ArgumentTemplates() {
    load "../spite_standard_library/windows"
    var filler = Filler<Mover>()
    filler.run(7)
}
```
```output
making 0 amount as Integer
making 1 label as String
making 2 scale as Float
update 7 '' 0
```

`made_arguments(start)` is `made_argument` once for each argument of `update_each`, in order, and its answers are the
call's arguments; each answer is typed by its own argument (`argument.class`), so the call is the plain call
`system.update_each(start, "", 0.0)`. The function is the one the plural is spread into or, inside a template over
`Spite.FunctionDeclaration`, the function that template stands for. Written anywhere else the plural is an error that
says to spread it into a call. A plain walker over arguments, called by `each`, takes a name without the word
(`describe_taken(argument: Spite.Argument)`).

### Walking a folder or an enum

Walks over the classes of a namespace and the values of an enum are ordinary code on the reflection objects, unrolled
while compiling. `Shop.Tools.classes.each(make)` makes one copy of `make` for each class of the folder, in which
`part` is the class itself: a type (`Column<part>()`), a constant for `part.name`, and the class object where a value
is wanted. `Phase.values.each(run)` calls `run` once for each value. A walker takes one parameter, and state it needs
goes in attributes. A parameter typed `Spite.Namespace` is an instance of one, so `class.namespace.classes` and
`Spite.Namespace.instances` are the lists to walk ([reflection.md](reflection.md#known-while-compiling)).

### Member templates

The standard library's list functions `filter_<member>()`, `sum_<member>()`, `count_<member>()` and the rest are
templates over the element's members ([collections.md](collections.md)), written in Spite in `library/list.spite`
with a parameter `member: Spite.AttributeDeclaration`, so `item[member]` reads a member of an element (the body
reads it on a `$element_type`, which is how it says the template ranges over that class's attributes). A generic class writes its own the same way:

```gdscript
func average_member(member: Spite.AttributeDeclaration): Float? {
    if item_count == 0 {
        return null
    }
    var total = sum_member(member)
    return total.to_float() / item_count.to_float()
}
```

A member template chains through a function of the member: `filter_<member>_<function>(arguments)` keeps the
items whose `member` answers `function(arguments)`, so `players.filter_name_starts_with("a")` keeps the players
whose `name` starts with `a`, and `functions.filter_name_ends_with("_each")` does the same for functions. Nothing in
it is special to reflection.

**A template that collects the members' values names the member in the plural**, because that is what it gives
back. Every other template keeps the member singular, because its name asks a question of each element:

| Template | Member | Reads as |
|---|---|---|
| `map_<members>` | plural | "the names": `classes.map_names()` answers a `List` of names |
| `filter_<member>`, `remove_where_<member>` | singular | "those that are alive": `filter_alive()` |
| `count_<member>`, `any_<member>`, `all_<member>` | singular | "how many / any / all are alive" |
| `sum_<member>` | singular | "the sum of health": `sum_health()` |
| `find_by_<member>`, `sort_by_<member>` | singular | "find by name", "sort by health" |

The plural comes from `String.pluralize()`, the rules every English plural follows plus a table of irregular and
uncountable words a program can add to ([standard_library.md](https://github.com/Spite-Language/spite_spec/blob/master/specs/standard_library.md#the-string-class)): `map_names` is
`name`, `map_people` is `person`, and `map_health` is `health`, its own plural. A question is never collected:
`monsters.map_is_alive()` is an error pointing at `filter_is_alive()`, `count_is_alive()`, `any_is_alive()` and
`all_is_alive()`, which are what a question is asked for. A value worth collecting that is not already a member
becomes one: a get-only attribute of the element's class, collected with its plural.

A list of a union keeps one member class with that class in the plural, and narrows to it, while an attribute every
member has is templated as on any list:

```gdscript
var entries = Directory("levels").entries()
crash entries
var files = entries.filter_files()
var sources = entries.filter_path_ends_with(".spite")
var paths = entries.map_paths()
```

`entries()` is a `List<Directory.Entry>?`, a union of `Directory` and `File` (`null` when the folder cannot be
opened); `filter_files()` answers a `List<File>`, and `path` is the member both classes have.
The other templates take a member class the same way: `entries.count_files()`, `any_directories()`,
`all_files()` and `remove_where_directories()`. A name that matches both a member class and an attribute is a
compile error naming the fix.

### Templates and walks, side by side

A template is not a loop in disguise. "Every attribute" is a walk, `Label.attributes.each(describe)`, and "this
attribute" is a template, `show_copies(label)`; there is no third form. The two meet in the same function: a
function taking a `Spite.AttributeDeclaration` is a template when its parameter is a word of its name and is called by
the member's name, and an ordinary function when a walk calls it. Either way it is compiled once per member it is used
for.

## Generics and codegen values (`$`)

`$name` means "replaced at code generation", and it is for generics only. A class declares each one on a
`generic` line of its own at the top of the file, and a caller gives the values positionally, in the order of
those lines: `Pair<String, Integer>(...)`. A `$name` with no `generic` line is an error; used as a value, as in
`if $verbose { }`, the error points at [`Environment`](programs.md#run-time-settings-environment), which is where
a program's settings live.

```gdscript title=generics_basics/pair.spite
generic $left_type
generic $right_type

var left: $left_type = null
var right: $right_type = null

func Pair(new_left: $left_type, new_right: $right_type) {
    left = new_left
    right = new_right
}

func describe(): String {
    return "{left} and {right}"
}
```
```gdscript title=generics_basics/generics_basics.spite entry
var console = Console()

func GenericsBasics() {
    load "../spite_standard_library/windows"
    var scoreboard = Pair<String, Integer>("Aria", 42)
    var description = scoreboard.describe()
    console.print(description)
    var inferred = Pair("Hero", 7)
    var inferred_description = inferred.describe()
    console.print(inferred_description)
}
```
```output
Aria and 42
Hero and 7
```

One line declares one value, and the lines sit at the top of the file, so skimming it shows what a class
accepts. Every hole is filled at every call: there are no defaults, and the wrong number of values is an error
that lists the class's values in order (the second program below). A generic class needs no constructor
(`library/list.spite` is `generic $element_type` and its functions), and when the constructor's arguments say
every value, the `<...>` may be left out: `Pair("Hero", 7)` is a `Pair<String, Integer>` and `Concurrent(file.read)` a `Concurrent<String?>`
([concurrency.md](concurrency.md)). `Pair` may declare `var left: $left_type = null` because its constructor
assigns `left` at once; anywhere else `null` belongs to `T?` alone, and `var left = $left_type()` makes
the default of whatever `$left_type` is bound to.

Only a class has codegen values. A function never lists its own (`func pick<$value_type>(...)` is an error), and
there are no generic functions to write instead: a function that takes any class takes a
`type`, such as the built-in empty `Anything` ([reflection.md](reflection.md#known-only-at-run-time)), or it belongs to a
generic class. The exact forms and error texts are in [Codegen values](https://github.com/Spite-Language/spite_spec/blob/master/specs/metaprogramming.md#codegen-values-).

A codegen value may be a value rather than a type. A condition on it is decided while compiling, so each set of
values is its own class with only its own branch in it:

```gdscript title=codegen_values_doc/weapon.spite
generic $damage_type
generic $is_magic

var damage: $damage_type = 0

func Weapon(new_damage: $damage_type) {
    damage = new_damage
}

func hit(): $damage_type {
    if $is_magic {
        return damage * 2
    }
    return damage
}
```
```gdscript title=codegen_values_doc/codegen_values_doc.spite entry
var console = Console()

func CodegenValuesDoc() {
    load "../spite_standard_library/windows"
    var staff = Weapon<Integer, true>(10)
    var club = Weapon<Float, false>(2.5)
    var staff_hit = staff.hit()
    var club_hit = club.hit()
    console.print(staff_hit, club_hit)
}
```
```output
20 2.5
```

```gdscript title=codegen_count_error/codegen_count_error.spite entry error
var console = Console()

func CodegenCountError() {
    load "../spite_standard_library/windows"
    var pairs = List<String, Integer>()
    var pairs_count = pairs.count()
    console.print(pairs_count)
}
```
```diagnostic
takes 1 codegen value(s), in this order: $element_type
```

### Constraining what a generic accepts

A `generic` line may name a `type` after a colon. The class accepts only types that fit it, and a class that does
not is an error where it is named, saying which function or attribute is missing, rather than an error deep inside the
generic's body, the first time some function happens to call what the class lacks.

```gdscript title=generic_constraint/shelf.spite
generic $item_type: Printable

type Printable {
    to_string(): String
}

var items = List<$item_type>()

func add(item: $item_type) {
    items.append(item)
}

func describe(): String {
    var first = items.first()
    crash first
    return first.to_string()
}
```
```gdscript title=generic_constraint/book.spite
var title = ""

func Book(new_title: String) {
    title = new_title
}

func to_string(): String {
    return "'{title}'"
}
```
```gdscript title=generic_constraint/generic_constraint.spite entry
var console = Console()

func GenericConstraint() {
    load "../spite_standard_library/windows"
    var books = Shelf<Book>()
    var book = Book("Dune")
    books.add(book)
    var numbers = Shelf<Integer>()
    numbers.add(3)
    var book_text = books.describe()
    var number_text = numbers.describe()
    console.print(book_text, number_text)
}
```
```output
'Dune' 3
```

```gdscript title=generic_constraint_error/generic_constraint_error.spite entry error
var console = Console()

func GenericConstraintError() {
    load "../spite_standard_library/windows"
    var sounds = Chorus<Rock>()
    var count = sounds.singers.count()
    console.print(count)
}
```
```gdscript title=generic_constraint_error/chorus.spite
generic $singer_type: Singer

type Singer {
    sing(): String
}

var singers = List<$singer_type>()
```
```gdscript title=generic_constraint_error/rock.spite
var weight = 3
```
```diagnostic
'Rock' does not fit type 'Singer', which 'Chorus' requires of $singer_type: it has no function 'sing'
```

The constraint is optional and names a `type` the class declares itself: a `type` is never named outside the class
that declares it, so `Shelf` declares its own `Printable`, the same as `Console`'s. What fits is what the `type` would accept anywhere, so
`Shelf<Integer>` above is fine, and so is a `T?` (`null` prints as `null`), while a `T?` against a `type` that needs
more than printing, or a function value, is not. It is checked wherever a class is given,
written out or read from the constructor's arguments, and it costs nothing at run time: the check is the
compiler's, and the class made is the one an unconstrained line would make
([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/metaprogramming.md#codegen-values-)).

### Asking what a generic was given

When a codegen value is a type, `if $value_type == String { }` asks which type it is. The answer is known when
the class is made, so only the branch taken is compiled, and each branch may use what only that type has. Besides
naming a type exactly, four names ask for a kind: `List`, `Dictionary`, `Null` (any `T?`) and `Enum` (any enum),
and a `type` name asks whether the type fits it (`$value_type == Number` for any number class).
The table is in [the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/metaprogramming.md#codegen-values-).

What the branch ruled out is not compiled at all, and neither is what only it reaches. Below, `Field<Integer>`
compiles `write` without the `List` branch, never compiles `write_list`, and so never checks its `.count()`
against an `Integer`; the statements after a taken branch that returns are left out the same way. One generic
class can therefore keep a helper per kind (`conformance/stage6/folded_helpers`):

```gdscript
func write(value: $value_type): String {
    if $value_type == List {
        return write_list(value)
    }
    return write_number(value)
}
```

Inside such a branch the types it was built from are named after the container's own codegen values:
`$value_type.element_type` for a `List<$element_type>`, `$value_type.key_type` and `$value_type.value_type` for a `Dictionary<$key_type, $value_type>`
or a `$value_type?`, and a generic class's own names for one of its instances. They are tested like `$value_type`
itself, `else if $list_type.element_type == Float`, and fold in every branch of the chain.

```gdscript title=describe_kind/kind.spite
generic $kind_type

func name(): String {
    if $kind_type == String {
        return "text"
    } else if $kind_type == Integer {
        return "a whole number"
    } else if $kind_type == Null {
        var inner = Kind<$kind_type.value_type>()
        var inner_name = inner.name()
        return "{inner_name}, or nothing"
    } else if $kind_type == List {
        var element = Kind<$kind_type.element_type>()
        var element_name = element.name()
        return "a list of {element_name}"
    } else {
        return "something else"
    }
}
```
```gdscript title=describe_kind/describe_kind.spite entry
var console = Console()

func DescribeKind() {
    load "../spite_standard_library/windows"
    var nested = Kind<List<List<Integer>>>()
    var nested_name = nested.name()
    console.print(nested_name)
    var maybe = Kind<String?>()
    var maybe_name = maybe.name()
    console.print(maybe_name)
    var flag = Kind<Boolean>()
    var flag_name = flag.name()
    console.print(flag_name)
}
```
```output
a list of a list of a whole number
text, or nothing
something else
```

A class test on a value folds the same way inside a generic. In `Detector<Label>`, `item == Health` can never be
true, so it is `false` and its branch is removed from that copy, while `Detector<Health>` keeps it. Outside a
generic the same never-true test is an error, since there it is always a mistake; inside one, code a particular
copy cannot use is what tree shaking is for ([control_flow.md](control_flow.md#value--class)).

```gdscript title=generic_never_true/detector.spite
generic $item_type

func is_health(item: $item_type): Boolean {
    if item == Health {
        return true
    }
    return false
}
```
```gdscript title=generic_never_true/health.spite
var points = 3

func total(): Integer {
    return points
}
```
```gdscript title=generic_never_true/label.spite
var text = "fragile"

func shown(): String {
    return text
}
```
```gdscript title=generic_never_true/generic_never_true.spite entry
var console = Console()

func GenericNeverTrue() {
    load "../spite_standard_library/windows"
    var labels = Detector<Label>()
    var healths = Detector<Health>()
    var label = Label()
    var label_answer = labels.is_health(label)
    var health = Health()
    var health_answer = healths.is_health(health)
    console.print(label_answer, health_answer)
}
```
```output
false true
```

A codegen value that is a type reads as its class wherever a class name would: `$component_type.name` is the
bound class's name the way `Health.name` is, and `$component_type.attributes` its attributes, with no value of
the type made to ask. On its own, `$name` is still not a value: only a member read through it is.

```gdscript title=codegen_class_name/column.spite
generic $component_type

func label(): String {
    var name = $component_type.name
    return "a column of {name}"
}
```
```gdscript title=codegen_class_name/codegen_class_name.spite entry
var console = Console()

func CodegenClassName() {
    load "../spite_standard_library/windows"
    var words = Column<String>()
    var label = words.label()
    console.print(label)
}
```
```output
a column of String
```

### Asking whether a class is fixed size

`$component_type.is_fixed_size` asks, while compiling, whether a class's size is known: only numbers, `Boolean`s,
enums and `String`s, nothing a list or another object makes grow. A generic can branch on it, and the branch not
taken is never compiled:

```gdscript title=fits_vector_doc/store.spite
generic $component_type

var inline = $component_type.is_fixed_size

func describe(): String {
    var kept = List<$component_type>()
    var kept_count = kept.count()
    if $component_type.is_fixed_size {
        return "fixed size, {kept_count} so far"
    }
    return "grows, {kept_count} so far"
}
```
```gdscript title=fits_vector_doc/fits_vector_doc.spite entry
var console = Console()

func FitsVectorDoc() {
    load "../spite_standard_library/windows"
    var points = Store<Vector3<Float>>()
    var point_answer = points.describe()
    var lists = Store<List<Integer>>()
    var list_answer = lists.describe()
    console.print(points.inline, point_answer)
    console.print(lists.inline, list_answer)
}
```
```output
true fixed size, 0 so far
false grows, 0 so far
```

How a `List` of them is stored is not this question's to answer: the compiler chooses that
([collections.md](collections.md#one-list)).

Inside a walk of attributes, `attribute.class.is_fixed_size` asks the same of each attribute, and
`attribute.index` is the attribute's place in the walk, 0 for the first: together they let a runner keep a
sparse set per component and fill a row of borrowed items from it
([memory.md](memory.md#a-row-of-items-for-one-call)).

## Markup is metaprogramming too

A language can give markup, a tree of tags with children, a grammar of its own, and then give that grammar settings
for what it turns into. Spite does neither. The tags are ordinary calls that a library answers with the same
function and template features this page teaches: a tag becomes a function of the library, its children are variadic
arguments, a component is picked by the shape of what it is given, and an event handler is a function bound to its
instance. There is no second syntax to learn and nothing to configure below the source, so the markup in one project
means what it means in every other.

## Tree shaking

Spite removes what a program does not use, and it can do so exactly, because everything it generates is decided
at compile time:

- A **condition on a codegen value** or on a [`Build`](programs.md#compile-time-settings-build) field is decided
  while compiling, and the branch not taken is never generated: `Weapon<Integer, true>` and `Weapon<Integer, false>`
  are two classes, and each keeps only the branch of `if $is_magic { }` that it takes.
- A **template** is compiled only for the names a program calls: a program that never calls `sum_price()` has no
  `sum_price`.
- **Reflection on a constant** folds: a question is `true` or `false`, a walk is unrolled into one call per member,
  and each function a walk calls is compiled once per member. No list of attributes, functions, classes or enum
  values exists at run time, and a program that walks nothing carries nothing.
- **Reflection at run time** is built only where it is read, for the classes such a read can reach
  ([reflection.md](reflection.md#known-only-at-run-time)), and a generic class such as
  [`JsonWriter` and the rest of json.md](json.md) exists only for the types a program uses it with.
- A **generic class** exists only for the values a program gives it, and each of its functions is compiled for one
  set of values only when surviving code calls it; a [constraint](#constraining-what-a-generic-accepts) is checked
  by the compiler and emits nothing.
- A **function that takes a `type`** (`Anything` included) is compiled once for each class that reaches it, and a
  value whose class is known only at run time picks its copy by a `switch` over the classes the program admits to
  that `type` ([values_and_types.md](values_and_types.md#inline-types-and-duck-typing)). A class that
  never reaches the function has no copy of it.
- In a production build, a **function or class** nothing reachable uses is not emitted
  ([optimizations.md](optimizations.md#tree-shaking)).
- The **concurrency machinery** (the state machines of a `Concurrent` and the loop that runs them, the thread pool,
  atomic reference counts) exists only in a program that makes a `Concurrent` or a `Parallel`, or is built with
  `--repl-port` or `--hot-reload`
  ([optimizations.md](optimizations.md#concurrency-machinery-only-where-it-is-used)).

This is tree shaking over the program's own model.

An inspectable build (`--development`, `--hot-reload`, `--repl` or `--repl-port`) keeps every function and class the
program generated instead of only the ones reachable from `main`, so live reload has all of them to swap and the REPL
can reach them, along with the run-time tables of every class
([compiler.md](compiler.md#development-builds-and-tree-shaking)). What is decided while compiling is decided there
too: templates are still generated only for the names called, and [conditions on codegen values](https://github.com/Spite-Language/spite_spec/blob/master/specs/metaprogramming.md#codegen-values-)
still fold. Every optimisation the compiler makes on its own, and the builds it applies in, is listed in
[optimizations.md](optimizations.md).

---

Next: [Reflection](reflection.md), the `Spite` classes every class, namespace and function is an instance of.
