# Reflection

**Every class is an instance of `Spite.Class`.** A class is not a special thing beside the values of a program: it
is an ordinary object, made from an ordinary class in `library/spite/`, and writing its name reads that object.
Every namespace is an instance of `Spite.Namespace`, every function an instance of `Spite.Function`, and every
attribute and argument an instance of `Spite.Attribute` and `Spite.Argument`. Reflection is not a second language
beside Spite: it is Spite describing itself, with the same lists, the same member templates and the same `[]` the
moron already uses everywhere else.

```gdscript title=every_class_an_object/monster.spite
var name = "troll"
var health = 10

func alive(): Boolean {
    return health > 0
}
```
```gdscript title=every_class_an_object/every_class_an_object.spite entry
var console = Console()

func EveryClassAnObject() {
    load "../spite_standard_library/windows"
    var kind: Spite.Class = Monster
    console.print("a class is a value:", kind)
    Monster.attributes.each(describe)
    var alive = Monster.functions['alive']
    if alive {
        console.print("function", alive.name, "returns", alive.returns)
    }
    if not Monster.functions['fly'] {
        console.print("no function fly")
    }
    Monster.source_files.each(show_file)
}

func describe(attribute: Spite.AttributeDeclaration) {
    console.print("attribute", attribute.name, attribute.class)
}

func show_file(file: File) {
    var from_monster = file.path.ends_with("monster.spite")
    console.print("declared in monster.spite:", from_monster)
}
```
```output
a class is a value: Monster
attribute name String
attribute health Integer
function alive returns Boolean
no function fly
declared in monster.spite: true
```

`Monster` read as a value is its `Spite.Class`. `.attributes` and `.functions` are dictionaries keyed by the member's
name, so `each` walks them and `[]` with a name answers the member or `null`. Every member of a reflection object is read as an
attribute, with no `()`, and none of them can be assigned. A list member template works on them like on any list,
so the names of every attribute are one call:

```gdscript
var names = Monster.attributes.keys()
```

None of this costs anything when the program runs. `Monster` is named in the code, so the compiler knows the
object, everything read from it, and every element of its lists: `each` above is two calls to `describe` written
out while compiling, each compiled for its own attribute, and no list of attributes exists in the program
([Known while compiling](#known-while-compiling)). The same expressions typed at a REPL prompt are answered from
tables only a REPL build carries ([Known only at run time](#known-only-at-run-time)).

## A class and an instance of it

Both a class and an instance have `.attributes` and `.functions`, and they answer different things. The case of the
receiver says which one is meant: `Monster` is the class, `troll` an instance.

| | The class, `Monster` | An instance, `troll` |
|---|---|---|
| `.attributes` | information: a `Spite.AttributeDeclaration` for each attribute, with its `.name`, `.class`, `.index`, and no value | data: a `Spite.Attribute` **bound to `troll`**, with everything the declaration has plus `.value`, read and written, typed as the attribute, and `.owner`, which is `troll` |
| `.functions` | information: a `Spite.FunctionDeclaration` for each function, with its `.name`, `.arguments`, `.returns` and the class that declares it | a `Spite.Function` bound to `troll`, with everything the declaration has plus `.owner`: a function value you can pass on, call, or build a `Spite.Call` from |
| `.class` | `Spite.Class`, the class every class is an instance of | `Monster`, so `troll.class == Monster` |
| What a read costs | nothing: it is known while compiling and folds | a read or write of `troll`'s own field, typed, the same as `troll.health` |

```gdscript title=class_and_instance/monster.spite
var name = "troll"
var health = 10

func alive(): Boolean {
    return health > 0
}

func hurt(amount: Integer) {
    health = health - amount
}
```
```gdscript title=class_and_instance/class_and_instance.spite entry
var console = Console()

func ClassAndInstance() {
    load "../spite_standard_library/windows"
    Monster.attributes.each(describe)
    var troll = Monster()
    troll.hurt(3)
    troll.attributes.each(show)
    troll.attributes.each(heal)
    console.print("health is now", troll.health)
    console.print("troll is a Monster:", troll.class == Monster)
    var bound = troll.functions['alive']
    crash bound
    console.print("bound to troll:", bound.name)
}

func describe(attribute: Spite.AttributeDeclaration) {
    console.print("declared", attribute.index, attribute.name, attribute.class)
}

func show(attribute: Spite.Attribute) {
    console.print("troll's", attribute.name, "=", attribute.value)
}

func heal(attribute: Spite.Attribute) {
    if attribute.class == Integer {
        attribute.value = attribute.value + 5
    }
}
```
```output
declared 0 name String
declared 1 health Integer
troll's name = troll
troll's health = 7
health is now 12
troll is a Monster: true
bound to troll: alive
```

In `heal`, `attribute.value` is an `Integer` in the copy compiled for `health`, so `+ 5` is ordinary arithmetic, and
the copy compiled for `name` has no `if` body at all. A declaration has no value to read, so `.value` on one of
`Monster.attributes` is an error that names the instance form:

```gdscript title=declaration_value/monster.spite
var health = 10
```
```gdscript title=declaration_value/declaration_value.spite entry error
var console = Console()

func DeclarationValue() {
    load "../spite_standard_library/windows"
    Monster.attributes.each(peek)
}

func peek(attribute: Spite.AttributeDeclaration) {
    console.print(attribute.value)
}
```
```diagnostic
'attribute.value' reads an attribute of an instance, and 'attribute' describes a declaration
```

A declaration still reaches one instance's field: `troll.attributes[attribute]` indexes the instance with it, which
is how a function handed a class's attribute reads or writes it on a value it was given. Indexed by a declaration
the compiler knows, it is the field itself, the short form of `troll.attributes[attribute.name].value`.

### An attribute named as a value

`Whisper.attributes["to"]` is the declaration of `Whisper`'s attribute `to`, a value a program can keep and hand
on, so a library can take the attribute it works through (`Audience.Linked(Whisper.attributes["to"])`). The key is
checked while compiling: a class named in the code that declares no attribute of that name is an error naming the
class, so renaming the attribute without the places that name it does not build. A key on an instance's
`.attributes` is not checked, and answers `null` when the name is missing.

```gdscript title=named_attribute/whisper.spite
var from = 1
var to = 2
var text = ""
```
```gdscript title=named_attribute/named_attribute.spite entry
var console = Console()

func NamedAttribute() {
    load "../spite_standard_library/windows"
    var to = Whisper.attributes["to"]
    console.print(to.name, to.index, to.class)
    var whisper = Whisper()
    whisper.text = "psst"
    var bound = whisper.attributes["text"]
    crash bound
    var declaration = bound.declaration
    console.print(declaration.name, bound.owner.class)
}
```
```output
to 1 Integer
text Whisper
```

### A call, built before it runs

Calling a function value calls it, as always: `greet("Ann")`, `tick()`. To fill a call's arguments one at a time,
by name, before running it, make a `Spite.Call` from the function's declaration and the instance to run it on:

```gdscript
var call = Spite.Call(MoveSystem.functions['run_each'], system)
call.arguments['position'] = position
call.arguments['velocity'] = velocity
call.call()
```

Nothing runs until `call.call()`. `system` must be a `MoveSystem`, or the line is a compile error. This is the only
way to build a call; a bound function, `system.functions['run_each']`, is for passing the function as a value and
calling it. When the function is named in the code (`MoveSystem.functions['run_each']`, or a `var` holding it that is
never assigned again), the whole thing is the plain call `system.run_each(position, velocity)`, with no object made.
An argument left unfilled is a compile error at `call.call()`; so is filling one twice, or naming one the function
does not have. A call built from a function known only at run time is a real object, and `call()` halts when the
instance is not of the class that declares the function, or when an argument is missing, of another type, or one the
function does not have; a parameter that takes a function value still halts. Calling a class known only at run time
makes an object of it, typed `Anything`, wherever the class object is held: `part()` with `part: Spite.Class`, an
attribute `kind()`, or a member read, `thing.class()` or `argument.class()`.

## Reaching a reflection object

| Written | Answers |
|---|---|
| a class name, `Monster`, `Shop.Tools.Hammer` | its `Spite.Class` |
| a namespace name, `Shop.Tools` | its `Spite.Namespace` |
| an enum name, `Phase` | its `Spite.Class`, whose `.values` are its values in order |
| `class` inside a function | the class the function answers on |
| `namespace` inside a function | that class's namespace, or `null` at the root of a program |
| `$T` in a generic class | the class it was given ([metaprogramming.md](metaprogramming.md#asking-what-a-generic-was-given)) |
| `value.class` | the class `value` really is, through a `type` or a union too |
| `Spite.Class.instances` | every class of the program |
| `Spite.Namespace.instances` | every namespace of the program |

`class` and `namespace` are not keywords: they are members every instance has, so a local of that name takes
precedence inside its function. A class and a namespace with the same dotted name is a compile error, so a name
always means one thing.

A class name given where a `Spite.Class` is wanted (an argument, a `var` declared `Spite.Class`, a `return`) is
its class object, and `==` between a `Spite.Class` and a class name compares the two objects. Anywhere else a bare
class name is the error that says to write `Health()`, since making one is nearly always what was meant, and
`var kind = Health` with no type is that error too:

```gdscript title=passing_a_class/ui/pressed.spite
func count(): Integer {
    return 1
}
```
```gdscript title=passing_a_class/health.spite
func amount(): Integer {
    return 10
}
```
```gdscript title=passing_a_class/passing_a_class.spite entry
var console = Console()
var components = List<Anything>()

func PassingAClass() {
    load "../spite_standard_library/windows"
    var health = Health()
    components.append(health)
    var pressed = Ui.Pressed()
    components.append(pressed)
    remove_component(Ui.Pressed)
    var kind: Spite.Class = Health
    var kept = components.count()
    crash components[0]
    console.print(kept, components[0].class == kind, kind == Health)
}

func remove_component(component_class: Spite.Class) {
    var kept = List<Anything>()
    var index = 0
    while index < components.count() {
        var component = components[index]
        if component.class != component_class {
            kept.append(component)
        }
        index = index + 1
    }
    components = kept
}
```
```output
1 true true
```

A class name is not a receiver for the class's own functions: there are no static functions, so `Gadget.boost()`
is an error saying to make a `Gadget()` first ([Functions of `Spite.Class`](#functions-of-spiteclass-and-no-static-functions)).

A class in a folder has a namespace ([packages.md](packages.md)); a class at the root of its program has none.
`.namespace` is a `Spite.Namespace?`, narrowed before its members are read, and printing a namespace prints its
dotted name.

```gdscript title=namespace_walk/shop/tools/hammer.spite
func weight(): Integer {
    return 2
}
```
```gdscript title=namespace_walk/namespace_walk.spite entry
var console = Console()

func NamespaceWalk() {
    load "../spite_standard_library/windows"
    var hammer = Shop.Tools.Hammer()
    describe(hammer)
}

func describe(hammer: Shop.Tools.Hammer) {
    var described = hammer.class
    fallback described.namespace.parent
    console.print(described.name, described.namespace, described.namespace.name)
    console.print("parent", described.namespace.parent)
}
```
```output
Hammer Shop.Tools Tools
parent Shop
```

One `fallback` proves the whole path it names and every prefix of it, so `described.namespace` is a plain value
too after `fallback described.namespace.parent` ([failure.md](failure.md#narrowing-a-path)).

## The `Spite` classes

Each class below is ordinary Spite in `library/spite/`. Every member is a get-only attribute named for what it
answers: a question is `is_...` and answers `Boolean`, a collection is a `List` named in the plural. A question
that would take an argument is a collection indexed or filtered instead, so there is one way to ask it.

### `Spite.Class`

| Member | Answers | Example |
|---|---|---|
| `.name` | the class's own name, a `String` | `Monster.name` is `Monster` |
| `.namespace` | its `Spite.Namespace?`, `null` at a program's root | `Shop.Tools.Hammer.namespace` is `Shop.Tools` |
| `.attributes` | a `Dictionary<String, Spite.AttributeDeclaration>` keyed by name, in declaration order | `Monster.attributes['health']` |
| `.functions` | a `Dictionary<String, Spite.Function>` keyed by name, in the order they were made | `Monster.functions['alive']` |
| `.values` | an enum's values, in order | `Phase.values[1]` is `'update'` |
| `.is_singleton` | the class has a `singleton` line | `Console.is_singleton` is `true` |
| `.is_stateful` | a function besides the constructor and `drop()` changes the object, or a singleton it binds does | `Counter.is_stateful` |
| `.is_fixed_size` | its size is known while compiling: only numbers, `Boolean`s, enums and `String`s | `Vector3<Float>.is_fixed_size` is `true` |
| `.is_list`, `.is_dictionary`, `.is_optional`, `.is_enum` | the kind of a type | `$value_type.is_list` |
| `.element_type` | a list's element, or what an optional holds | `$list_type.element_type` |
| `.value_type` | a dictionary's value | `$map_type.value_type` |
| `.source_files` | a `List<File>`: every file that declares or reopens the class, in load order | `Monster.source_files.first()` |

A class has no folder of its own: a reopening may live anywhere a program loads, so `.source_files` is the whole
list, and a question about a path is asked of those `File`s.

### `Spite.Namespace`

| Member | Answers | Example |
|---|---|---|
| `.name` | the last segment | `Shop.Tools.name` is `Tools` |
| `.name_with_namespaces` | the dotted name | `Shop.Tools.name_with_namespaces` is `Shop.Tools` |
| `.parent` | the namespace above, a `Spite.Namespace?` | `Shop.Tools.parent` is `Shop` |
| `.classes` | the classes declared directly in it | `Shop.Tools.classes.count()` |
| `.namespaces` | the namespaces directly below it | `Shop.namespaces` |
| `.every_class` | its classes and those of every namespace below it | `Shop.every_class` |
| `.source_directories` | a `List<Directory>`: every directory its classes come from, in load order | `Shop.source_directories.first()` |

There is no root namespace object: a namespace at the top has `null` as its `.parent`.

### `Spite.Function`

| Member | Answers | Example |
|---|---|---|
| `.name` | the function's name | `alive.name` is `alive` |
| `.arguments` | a `List<Spite.Argument>`, in order | `hurt.arguments[0].name` is `amount` |
| `.returns` | a `Spite.Class`, `Nothing` when none is declared | `alive.returns` is `Boolean` |
| `.owner` | on a bound function: the instance it is bound to | `troll.functions['alive'].owner` is `troll` |
| `.is_resumable` | it can reach a wait (a sleep, a file or socket read), so it is also compiled as a state machine that can pause there and be resumed ([concurrency.md](concurrency.md)) | `load_each.is_resumable` |
| `.returned_literal` | the text literal it returns | `greeting.returned_literal` is `grr` |
| `.reads` | a `List<Spite.AttributeDeclaration>`: every attribute of its class its code reads, in declaration order | `update_each.reads.map_names()` |
| `.writes` | a `List<Spite.AttributeDeclaration>`: every attribute of its class its code writes | `update_each.writes.count()` |

A class's `.functions` holds `Spite.FunctionDeclaration`s: the members above, none bound to an instance, and in a
template over functions `.hole` (the text that stands in the function's name for the template's word) and `.index`. An
instance's `.functions` holds `Spite.Function`s bound to it, and `call_function()` calls one. A call built argument by argument is a `Spite.Call` of a declaration and an instance
([above](#a-call-built-before-it-runs)).

### `Spite.Call`

| Member | Answers | Example |
|---|---|---|
| `Spite.Call(declaration, instance)` | a call of a `Spite.FunctionDeclaration` on an instance of its class, nothing run yet | `Spite.Call(MoveSystem.functions['run_each'], system)` |
| `.arguments` | its arguments by name, each written once before the call runs | `call.arguments['position'] = position` |
| `call()` | runs it, answering what the function answers | `call.call()` |

### `Spite.Argument`

| Member | Answers | Example |
|---|---|---|
| `.name` | the argument's name | `hurt.arguments[0].name` is `amount` |
| `.class` | its declared class | `hurt.arguments[0].class` is `Integer` |
| `.index` | its place, 0 for the first | `hurt.arguments[1].index` is `1` |
| `.is_mutated` | the function changes the object passed there, or anything reached through it | `save_each.arguments[0].is_mutated` |
| `.function` | the function it belongs to | `argument.function.name` |

### `Spite.Attribute`

| Member | Answers | Example |
|---|---|---|
| `.name` | the attribute's name | `attribute.name` is `health` |
| `.class` | its declared class | `attribute.class` is `Integer` |
| `.index` | its place in declaration order, 0 for the first | `Monster.attributes['health'].index` is `1` |
| `.declaration` | on a bound attribute: its `Spite.AttributeDeclaration`, so `attribute.declaration.name` and `attribute.name` agree | `troll.attributes['health'].declaration` |
| `.owner` | on a bound attribute: the object it belongs to | `troll.attributes['health'].owner` is `troll` |
| `.camel_case_name`, `.pascal_case_name` | its name as other systems spell it | `buy_price` is `buyPrice`, `BuyPrice` |
| `.is_singleton` | its class is a singleton | `ReflectionObjects.attributes['console'].is_singleton` |
| `.value` | on an instance's attribute only: that instance's value, read and written | `troll.attributes['health'].value` |

A class's `.attributes` holds `Spite.AttributeDeclaration`s: `.name`, `.class`, `.index`, the case spellings and
`.is_singleton`, never a `.value`. An instance's holds `Spite.Attribute`s bound to it: those members, read through the
declaration, plus `.value` and `.owner`. Which one a function takes depends on what it is for: a template, or a walk of the
class, takes the declaration, and a walk of an instance's data takes the bound attribute.

### What a function reads and writes

`function.reads` and `function.writes` answer the attributes of its class that its code reads and writes, as
`Spite.AttributeDeclaration`s in declaration order. Both follow every call to a function of the same class, however
deep, so `remember()` below writes `trail` for `update_each`. Calling a function on an attribute
(`trail.append(position)`) reads it, and writes it when that function changes the object it is called on. What a
function does with an argument is the argument's own `.is_mutated`.

```gdscript title=access_report/movement.spite
var position = 0
var speed = 1
var trail = List<Integer>()

func update_each(steps: Integer) {
    position = position + speed * steps
    remember()
}

func remember() {
    trail.append(position)
}
```
```gdscript title=access_report/access_report.spite entry
var console = Console()

func AccessReport() {
    load "../spite_standard_library/windows"
    var update = Movement.functions['update_each']
    crash update
    update.reads.each(describe_read)
    var writes = update.writes
    var written = writes.map_names()
    var joined = written.join(", ")
    console.print("writes:", joined)
    var movement = Movement()
    movement.update_each(2)
    console.print(movement.position)
}

func describe_read(attribute: Spite.AttributeDeclaration) {
    console.print("reads", attribute.name)
}
```
```output
reads position
reads speed
reads trail
writes: position, trail
2
```

A function known only at run time answers the same lists: one kept in a list and read later, a function value
like `movement.remember`, or one read from `value.class.functions` through `Anything`. The program carries what
each function reads and writes only when some code asks `.reads` or `.writes` of such a function. The functions of
the program and the packages it loads answer; a function of the standard library, a function value taken through a
`type` (which has no single function behind it) and any function in a `--hot-reload` build, where reloading would
make the answer stale, do not, and asking them at run time halts.

## Members are lists

Every collection above is an ordinary `List` (and `.attributes` and `.functions` ordinary `Dictionary`s keyed by name), so everything a list has
works on it: `[]` with a name answers the member or `null`, and `count()`, `each`, `filter` and every member
template do what they do on any list.

```gdscript title=constant_selection/runner.spite
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
```gdscript title=constant_selection/constant_selection.spite entry
var console = Console()

func ConstantSelection() {
    load "../spite_standard_library/windows"
    var update = Runner.functions['update_each']
    if update {
        console.print(update.name, "changes its label:", update.arguments[1].is_mutated)
    }
    if not Runner.functions['draw_each'] {
        console.print("no draw_each")
    }
    var walked = Runner.functions.filter(walks_each)
    walked.each(describe)
}

func walks_each(function: Spite.Function): Boolean {
    return function.name.ends_with("_each")
}

func describe(function: Spite.Function) {
    var count = function.arguments.count()
    console.print(function.name, "takes", count)
    function.arguments.each(describe_taken)
}

func describe_taken(argument: Spite.Argument) {
    var fresh = made(argument)
    console.print(" ", argument.index, argument.name, argument.class, "'{fresh}'")
}

func made(argument: Spite.Argument): argument.class {
    return argument.class()
}
```
```output
update_each changes its label: false
no draw_each
update_each takes 2
  0 amount Integer '0'
  1 label String ''
render_each takes 1
  0 scale Float '0'
```

`filter(walks_each)` keeps the functions whose test answers true, and a walk over what it kept is unrolled: the compiler
writes one `describe` for each function of `Runner`. `filter_name_ends_with("_each")` keeps the same functions: a
member template chains through a function of the member, and nothing in it is special to reflection
([collections.md](collections.md)). A name is selected, never built: `[]` takes a name written in the code, and
anything else is a filter over the names that exist. `made` answers `argument.class`, a different class for each
argument, which is fine because it is compiled once for each.

A member template that collects the members' values takes the member in the plural, and a dictionary's member
templates work over its values:

```gdscript
var names = Monster.attributes.map_names()
var counted = Shop.every_class.count_is_singleton()
var written = update_each.writes.map_names()
```

## Known while compiling

Most reflection is asked of something the compiler can already identify: a class, namespace or enum named in the
code, `$T`, `class`, `namespace`, and everything read from one of them. Such an object is a **constant**, and
reflection on it is ordinary code that costs nothing at run time:

- **A question folds.** `Monster.is_singleton`, `update.is_resumable` or `argument.is_mutated` is a `true` or a
  `false` in the generated code, and an `if` on it keeps only the branch taken.
- **`each` over a constant list is unrolled.** `Monster.attributes.each(show)` calls `show` once for `name` and
  once for `health`; no list is made. A filter or a member template over a constant list is a constant list.
- **A function handed a constant is compiled once for it.** Inside that copy the parameter is the constant
  itself, so `attribute.class` is a type, `attribute.name` is text known while compiling, and
  `monster.attributes[attribute]` is a plain read of the field.
- **A constant class is a type and makes its own instances**: `part()` makes one, `Column<part>()` a column of
  them.

```gdscript title=constant_namespaces/part/wheel.spite
var size = 3
```
```gdscript title=constant_namespaces/part/axle.spite
var length = 7
```
```gdscript title=constant_namespaces/constant_namespaces.spite entry
enum Phase {
    'start'
    'update'
    'finish'
}

var console = Console()

func ConstantNamespaces() {
    load "../spite_standard_library/windows"
    var parts = Spite.Namespace.instances.filter(holds_parts)
    parts.each(make_every_class)
    var count = Phase.values.count()
    var last = Phase.values.find_by_name('finish')
    console.print(count, "phases, the last is", last)
}

func holds_parts(namespace: Spite.Namespace): Boolean {
    return namespace.name == "Part"
}

func make_every_class(namespace: Spite.Namespace) {
    namespace.classes.each(make)
}

func make(part: Spite.Class) {
    var fresh = part()
    var shown = fresh.to_debug()
    console.print(part.name, shown)
}
```
```output
Axle Axle { length: 7 }
Wheel Wheel { size: 3 }
3 phases, the last is finish
```

`make` is compiled twice, as `make_for_axle` and `make_for_wheel` in `--final-classes`: `part()` is
`Part.Axle()` in one copy and `Part.Wheel()` in the other. Every folder named `part` anywhere is found by this
filter over every namespace, with nothing registered.

## Known only at run time

A value typed `Anything` or a union, a reflection object kept in an attribute and read later, and everything typed
at the REPL prompt are only known at run time. Reflection still answers them, from tables the program builds only
for the classes such a read can reach, and there `attribute.value` is an `Anything?`: the value itself (a number
keeps its class, held with its class's tag, and text is boxed), `null` only when the attribute holds `null`. Its text is `.value.to_string()`.

```gdscript title=attribute_objects/health.spite
var amount = 10
```
```gdscript title=attribute_objects/player.spite
var health = Health()
var speed = 3
var target: Health? = null
```
```gdscript title=attribute_objects/attribute_objects.spite entry
var console = Console()
var components = List<Anything>()

func AttributeObjects() {
    load "../spite_standard_library/windows"
    var player = Player()
    add_every_attribute(player)
    components.each(show_component)
}

func show_component(component: Anything) {
    var described: String = component.class
    if component.class == Health {
        described = "Health {component.amount}"
    }
    console.print(described)
}

func add_every_attribute(bundle: Anything) {
    var attributes = bundle.attributes
    attributes.each(add_value)
}

func add_value(attribute: Spite.Attribute) {
    fallback attribute.value
    components.append(attribute.value)
}
```
```output
Health 10
Integer
```

`Anything` is the library's empty `type`: every class fits it, so a function that takes any object says
`component: Anything`. What needs a constant says so while compiling: a function whose result is typed by its
reflection parameter cannot be handed an object known only at run time.

```gdscript title=run_time_type/gadget.spite
var power = 3
```
```gdscript title=run_time_type/run_time_type.spite entry error
var console = Console()

func RunTimeType() {
    load "../spite_standard_library/windows"
    var gadget = Gadget()
    var anything: Anything = gadget
    var attributes = anything.attributes
    var first = attributes.values().first()
    crash first
    var made = fresh(first)
    console.print(made)
}

func fresh(attribute: Spite.Attribute): attribute.class {
    return attribute.class()
}
```
```diagnostic
'fresh' answers a type read from a reflection object, so it is compiled once for each object the compiler can identify, and this call hands it one known only at run time: pass a class, attribute, function or argument named in the code, or walked from one
```

One function body serves both: called with a constant it is typed and unrolled, and called from the REPL with a
live object it takes the run-time path. A `--repl`, `--repl-port` or `--development` build keeps the tables for
every class, so the prompt can ask anything the program can:

```gdscript
> Monster.attributes.map_names()
name, health
> Monster.functions['alive'].returns
Boolean
> troll.attributes['health'].value = 40
```

## Every class in the program

`Spite.Class.instances` is every class of the program, its own folder and every package it loads but not the
standard library, in the order of their dotted names, because a class is an instance of `Spite.Class`. Walking a namespace's
`.classes` the same way is how the library's `Test` class finds the tests of a namespace without being told about them ([testing.md](testing.md)). Memory and threads are not
reflected: a program cannot ask where a value lives, how many objects exist or which are alive, so nothing it
could watch ever stands in the way of how the compiler places them.

```gdscript title=every_class_of_the_program/monster.spite
var name = ""

func Monster(starting_name: String) {
    name = starting_name
}
```
```gdscript title=every_class_of_the_program/every_class_of_the_program.spite entry
var console = Console()

func EveryClassOfTheProgram() {
    load "../spite_standard_library/windows"
    var classes = Spite.Class.instances
    var class_count = classes.count()
    var found = classes['Monster']
    crash found
    console.print("the program has", class_count, "classes, among them", found)
}
```
```output
the program has 2 classes, among them Monster
```

Describing a class makes no object of it: a class object's `.attributes` are its declarations, which need no object,
and its `.functions` are read from the class as the compiler built it. The program's entry class is described the
same way, but its `.functions` is empty, since making one would run the program again.

## The names reflection gives every object

`class`, `attributes` and `functions` belong to reflection on every class, so a class
cannot declare an attribute or a function of its own by those names, nor a `get_` or `set_` function that would be
read as one. One named `attributes` would otherwise hide the real list from everything that walks a class:
`JsonReader` would read every key as a key the class does not have. The error names what the member means and a
name to use instead:

```gdscript title=reflection_names/reflection_names.spite entry error
var console = Console()
var attributes = List<String>()

func ReflectionNames() {
    load "../spite_standard_library/windows"
    attributes.append("heavy")
    console.print(attributes.count())
}
```
```diagnostic
the attribute 'attributes' has a name reflection gives every object: 'value.attributes' lists its attributes, and JsonReader, JsonWriter and every attribute walk read through it, and one of its own would hide it. Name it something else, like 'reflection_names_attributes'
```

A local variable or a parameter may still use any of these names, since nothing reads it through an object.

## Reflection is read-only

Every member describing the program is a getter with no setter: `library/spite/class.spite` keeps its data in
private fields (`_name`, `_namespace`, `_attributes`, ...) and answers `get_name()`, `get_namespace()` and the rest,
so `.name` reads through the ordinary getter and writing it is an error. A name starting with `_` is used only
inside its own class, so the private fields cannot be written from outside either. The one thing reflection
writes is an instance's own value, through a bound attribute's `.value`.

```gdscript title=reflection_read_only/reflection_read_only.spite entry error
var console = Console()

func ReflectionReadOnly() {
    load "../spite_standard_library/windows"
    var described = console.class
    described.name = "Terminal"
}
```
```diagnostic
'name' is read-only
```

A reopening of a `Spite` class is inside it, so it may read those fields; that is how a program adds a member
to every class object at once ([packages.md](packages.md#the-spite-namespace-is-reserved)).

## Functions of `Spite.Class`, and no static functions

There are no static functions: a class is an instance of `Spite.Class`, so a member that belongs to the class rather
than to its instances is an ordinary member of `Spite.Class`, declared there with its default. `.is_singleton` is
one: every class answers it, from its `singleton` line ([classes_and_files.md](classes_and_files.md#singletons)),
and declaring it in a class is an error that says to write the `singleton` line instead. A class's name, `Gadget`,
is a constant that names one instance of `Spite.Class`; a class file's functions are its instances' functions, never
its class object's, so no class answers a class question its own way. Reopening `Spite.Class` is the one way to
change what a class object answers, and it changes it for every class in the program. It is a foot you are allowed
to shoot, and one that shows in `--final-classes`
([the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/reflection.md#functions-of-spiteclass-and-why-there-are-no-static-functions)).

---

Next: [Packages, namespaces, and mods](packages.md), how folders become namespaces and how a program loads packages.
