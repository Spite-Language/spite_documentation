# Reflection

**Every class is an instance of `Spite.Class`.** A class is not a special thing beside the values of a program: it
is an ordinary object, made from an ordinary class in `common/spite/`, and writing its name reads that object.
Every namespace is an instance of `Spite.Namespace`, every function a `Spite.FunctionDeclaration` (a `Spite.Function` once bound to an object), and every
attribute and argument an instance of `Spite.Attribute` and `Spite.Argument`. Reflection is not a second language
beside Spite: it is Spite describing itself, with the same lists, the same member templates and the same `[]` the
moron already uses everywhere else.

```gdscript title=every_class_an_object/monster.spite
var name = "troll"
var health = 10

func alive(): Boolean {
    return health > 0 and not name.is_empty()
}
```
```gdscript title=every_class_an_object/every_class_an_object.spite entry
var console = Console()

func EveryClassAnObject() {
    var kind: Spite.Class = Monster
    console.print("a class is a value:", kind)
    Monster.attributes.each(describe)
    var alive = Monster.functions["alive"]
    if alive {
        console.print("function", alive.name, "returns", alive.returns)
    }
    if not Monster.functions["fly"] {
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
| `.attributes` | information: a `Spite.AttributeDeclaration` for each attribute, with its `.name`, `.class`, `.index`, and no value | data: a `Spite.Attribute<Integer>` **bound to `troll`** for an `Integer` attribute, with everything the declaration has plus `.value`, read and written, typed as the attribute; reached by a name written in the code, a declaration or a walk |
| `.functions` | information: a `Spite.FunctionDeclaration` for each function, with its `.name`, `.arguments`, `.returns` and the class that declares it | a `Spite.Function` bound to `troll`: its `.name` and `.declaration`, which answers everything else the function is, and `call_function()` |
| `.class` | `Spite.Class`, the class every class is an instance of | `Monster`, so `troll.class == Monster` |
| What a read costs | nothing: it is known while compiling and folds | a read or write of `troll`'s own field, typed, the same as `troll.health` |

An instance's `.class` folds too when its type has one class: `troll.class == Monster` costs nothing. When the
instance is something a call answers, `next_monster().class == Monster` still calls `next_monster` once, and only the
class is known.

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
    Monster.attributes.each(describe)
    var troll = Monster()
    troll.hurt(3)
    troll.attributes.each(show)
    troll.attributes.each(heal)
    console.print("health is now", troll.health)
    console.print("troll's class:", troll.class)
    var bound = troll.functions["alive"]
    crash bound
    console.print("bound to troll:", bound.name)
}

func describe(attribute: Spite.AttributeDeclaration) {
    console.print("declared", attribute.index, attribute.name, attribute.class)
}

func show(attribute: Spite.Attribute<Console.Printable>) {
    console.print("troll's", attribute.name, "=", attribute.value)
}

func heal(attribute: Spite.Attribute<Integer>) {
    attribute.value = attribute.value + 5
}
```
```output
declared 0 name String
declared 1 health Integer
troll's name = troll
troll's health = 7
health is now 12
troll's class: Monster
bound to troll: alive
```

A function taking attributes says which it takes (D2638): `heal` takes a `Spite.Attribute<Integer>`, so the walk
calls it for `health` alone and `+ 5` is ordinary arithmetic; `show` takes a `Spite.Attribute<Console.Printable>`, so
it is called for every attribute whose class has a `to_string()`. The walk compiles a copy for each attribute, and
in each `.value` has that attribute's own class, as a function taking a `type` is compiled for each class that
reaches it: the body must make sense for each of them. The value type is a class, a union or a `type`, and a nullable
attribute is walked only by a nullable one (`Spite.Attribute<Console.Printable?>`). A bare `Spite.Attribute` says nothing of its value and is refused, and a
walk that no attribute fits is an error (`'heal' takes a 'Spite.Attribute<Boolean>', and no attribute of 'Monster'
holds one, so this walk would call it on none: ...`). A function reading every attribute whatever it holds takes the
declaration, `Spite.AttributeDeclaration`, which has its `.name`, `.class` and `.index` and no value. Inside a walk
`attribute.owner` is the walked object; elsewhere a `Spite.Attribute` has no `.owner`, since the object whose
`.attributes` was read is it.

A declaration has no value to read, so `.value` on one of `Monster.attributes` is an error that names the instance
form:

```gdscript title=declaration_value/monster.spite
var health = 10
```
```gdscript title=declaration_value/declaration_value.spite entry error
var console = Console()

func DeclarationValue() {
    Monster.attributes.each(peek)
}

func peek(attribute: Spite.AttributeDeclaration) {
    console.print(attribute.value)
}
```
```diagnostic
'attribute.value' reads an attribute of an instance, and 'attribute' describes a declaration
```

The walked attribute is an object of the class its parameter declares, `Spite.AttributeDeclaration` (or
`Spite.Attribute` in a walk over an instance), and not text. Its members are read while compiling, so
`attribute.name` costs nothing; used whole it is that class's object, the same as `Monster.attributes["health"]`
(or `troll.attributes["health"]`, a walked private attribute included): a hole or a `String` parameter shows its
`to_string()`, its name, a local copy keeps its class, `.attributes` and `.functions` list the reflection object's
own, it can be kept in a list, `==` with text compares its name, and `[0]`, `+` or `<` on it, or a `String` function, is the
error that class's object gives. A template's attribute parameter is the same object, for the class the template
walks (`describe_attribute` in `Tag` called as `tag.describe_cost()` sees `Tag.attributes["cost"]`), and so is a walked
attribute handed on to another function. Every template parameter is its declaration object the same way: a member
that is a function (`tag.describe_urgent()` over `func urgent()`) is `Tag.functions["urgent"]`, a
`Spite.FunctionDeclaration`; a template over functions sees the function it stands for (`phase` in `run_phase_each`
is `Mover.functions["update_each"]`); and a member of a `type` or a union, which no class lists, is a
`Spite.AttributeDeclaration` made for it, with its name, its place among the type's members and its class. The text
is `attribute.name`:

```gdscript title=walked_member_text/monster.spite
var health = 10
```
```gdscript title=walked_member_text/walked_member_text.spite entry error
var console = Console()

func WalkedMemberText() {
    Monster.attributes.each(shout)
}

func shout(attribute: Spite.AttributeDeclaration) {
    var shouted = attribute.upper_case()
    console.print(shouted)
}
```
```diagnostic
a Spite.AttributeDeclaration has no attribute or function 'upper_case'
```

### An instance indexed by a declaration

A declaration still reaches one instance's attribute: `troll.attributes[health]`, with `health` a declaration of
`Monster`, always answers troll's `Spite.Attribute<Integer>` for it, never the field, so read `.value` for the field. The
declaration must be one of the instance's own class. The compiler checks that when the declaration is kept in a
`var` that is never assigned again (`var health = Monster.attributes["health"]`), and another class's declaration
is an error naming both classes. A declaration it cannot tie to a class, such as a
parameter or a `var` assigned again, is refused, and so is a name known only while running: walk the attributes
instead. In a walk the walked declaration's name is known, so `world.attributes[attribute.name]` there is `world`'s
attribute of that name.

Since such a declaration is tied to one attribute, `.value` read straight off the index has that attribute's own
type and compiles to a plain read of the field: `troll.attributes[health].value` is troll's `health`, an `Integer`, so
it prints and adds as one. A literal name of an attribute troll's class declares means the same:
`troll.attributes["health"].value` is that `Integer` too, and a name it does not declare is an error while compiling
(see [An attribute kept in a var](#an-attribute-kept-in-a-var) for one kept first).

A class indexes its own instance the same way, bare, as it reads its own attributes: inside `Monster`,
`attributes[health]` is this monster's `Spite.Attribute<Integer>`, under the same checks (`attributes[health]` with a `Rat`
declaration inside `Monster` is "'health' is a declaration of Rat, and 'attributes[health]' indexes this Monster: an
instance is indexed only by a declaration of its own class"), and `attributes.each(f)` walks them.
`this.attributes` is refused like any `this.name`.

It means the same inside a walk over a class's attributes or a member template: `troll.attributes[attribute]` with
the walked `attribute` is troll's `Spite.Attribute`, and `troll.attributes[attribute].value` reads and writes the
field. There the walked `attribute` is known in each copy ([Known while compiling](#known-while-compiling)), so
`.value` compiles to a plain read or write of `troll.health` and no `Spite.Attribute` is made. Only the walked
declaration does that: another one used as the key in the same copy, such as
`var other = Monster.attributes["name"]`, indexes as it would anywhere else, and a declaration parameter is refused for
the same reason it is outside a walk. There is no shorter
form: `troll[attribute]` is `troll.get_at(attribute)` here too, as every `[ ]` is, so a class that declares no
`get_at` refuses it: "indexing a 'Monster' needs it to define 'get_at(index)'". An instance's attributes are
never replaced, so a walk copies a field through `.value`:

```gdscript
func copied(attribute: Spite.AttributeDeclaration) {
    copy.attributes[attribute].value = price.attributes[attribute].value
}
```

Writing `copy.attributes[attribute] = price.attributes[attribute]` is "'copy.attributes[attribute]' is a
Spite.Attribute, and an instance's attributes are never replaced: write its field through it,
'copy.attributes[attribute].value = ...'". A template member that is a function taking nothing has no
`Spite.Attribute`: `given.attributes[attribute].value` calls it, and `given.attributes[attribute]` alone is an error
saying so.

```gdscript title=indexed_by_a_declaration/monster.spite
var name = "troll"
var health = 10
```
```gdscript title=indexed_by_a_declaration/indexed_by_a_declaration.spite entry
var console = Console()

func IndexedByADeclaration() {
    var troll = Monster()
    var health = Monster.attributes["health"]
    var bound = troll.attributes[health]
    console.print(bound.name, bound.value, bound.declaration.index)
}
```
```output
health 10 1
```

With a class `Rat` that declares a `health` of its own:

```spite
var rat = Rat()
var bound = rat.attributes[health]
```
```diagnostic
'health' is a declaration of Monster, and 'rat' is a Rat: an instance is indexed only by a declaration of its own class
```

### An attribute kept in a var

A `Spite.Attribute` kept in a `var` given once from such an index, `var bound = troll.attributes[health]` or
`var bound = troll.attributes["health"]`, is a `Spite.Attribute<Integer>` for an `Integer` attribute: `bound.value`
has that attribute's own type, the field as it was when `bound` was made. A name the instance's class does not
declare is an error while compiling, naming the class, so the answer is never `null`.

```gdscript title=kept_attribute/shelf.spite
var box: Box? = null
```
```gdscript title=kept_attribute/box.spite
var size = 1
```
```gdscript title=kept_attribute/monster.spite
var health = 10
```
```gdscript title=kept_attribute/kept_attribute.spite entry
var console = Console()

func KeptAttribute() {
    var troll = Monster()
    var shelf = Shelf()
    var health = Monster.attributes["health"]
    var bound = troll.attributes[health]
    troll.health = 4
    var more = bound.value + troll.health
    console.print(bound.value, more)
    var box = shelf.attributes["box"]
    if box.value {
        console.print("a box")
    } else {
        console.print("no box")
    }
}
```
```output
10 14
no box
```

### An attribute named as a value

`Whisper.attributes["to"]` is the declaration of `Whisper`'s attribute `to`, a value a program can keep and hand
on, so a library can take the attribute it works through (`Audience.Linked(Whisper.attributes["to"])`). The key is
checked while compiling, on a class and on an instance alike: one its class does not declare is an error naming the
class, so renaming the attribute without the places that name it does not build.

```gdscript title=named_attribute/whisper.spite
var from = 1
var to = 2
var text = ""
```
```gdscript title=named_attribute/named_attribute.spite entry
var console = Console()

func NamedAttribute() {
    var to = Whisper.attributes["to"]
    console.print(to.name, to.index, to.class)
    var whisper = Whisper()
    whisper.text = "psst"
    var bound = whisper.attributes["text"]
    var declaration = bound.declaration
    console.print(declaration.name, bound.value)
}
```
```output
to 1 Integer
text psst
```

### A call, built before it runs

Calling a function value calls it, as always: `greet("Ann")`, `tick()`. To fill a call's arguments one at a time,
by name, before running it, make a `Spite.Call` from the function's declaration and the instance to run it on:

```gdscript
var call = Spite.Call(MoveSystem.functions["run_each"], system)
call.arguments["position"] = position
call.arguments["velocity"] = velocity
call.call()
```

Nothing runs until `call.call()`. The declaration is one named in the code (`MoveSystem.functions["run_each"]`, or a
`var` holding it that is never assigned again), so the compiler makes a class for its arguments with one typed field
per parameter: `call.arguments["position"]` is that field, and the whole thing is the plain call
`system.run_each(position, velocity)`, with no object made. `system` must be a `MoveSystem`, or the line is a
compile error. A declaration known only while running (`value.class.functions[name]`) has no class of arguments the
program can name, so such a call fills none: `call.call()` runs a function that takes no arguments on the instance,
after checking the instance is of the class that declares it (it halts otherwise, and halts for a function that takes
arguments), and drops what it answers. Filling an argument of it is refused:

```gdscript title=call_known_at_run_time/monster.spite
var health = 10

func hurt(amount: Integer) {
    health = health - amount
}
```
```gdscript title=call_known_at_run_time/call_known_at_run_time.spite entry error
func CallKnownAtRunTime() {
    var troll = Monster()
    var name = "hurt"
    var call = Spite.Call(troll.class.functions[name], troll)
    call.arguments["amount"] = 3
    call.call()
}
```
```diagnostic
this call runs a declaration known only while running, so it fills no arguments: such a call runs a function that takes none, and a function that takes arguments is called through a declaration named in the code, as 'Spite.Call(Monster.functions["hurt"], troll)'
```

An argument left unfilled is a compile error at `call.call()` (`'call' leaves the argument 'velocity' of 'hurt'
unfilled`); so is filling one twice (`'call.arguments["amount"]' is already set`), naming one the function does not
have (`'hurt' has no argument 'speed', so 'call.arguments["speed"]' fills nothing`), and using the call as a value,
since it is only the direct call (`'call' is a call known while compiling ... has no object to pass on`). An instance of
another class than the one declaring the function is a compile error too. `call.call()` answers what the function
answers, typed from the declaration (`var worth = call.call()` is an `Integer` for `func worth(): Integer`).

Calling a class known only while running makes an object of it, of the classes the compiler can make there, wherever
the class object is held: `part()` with `part: Spite.Class`, an attribute `kind()`, or `thing.class()`. What it answers
is a `Console.Debuggable`, the library's `type` every class fits, so the program says what it uses it through, and
`if made.class == Gear` narrows it. It halts, saying why, for the program's entry class and for a class whose
constructor takes arguments. In a walk over classes each copy knows its class, so `part()` there is that class.

### A name built while compiling

A name in `[]` may be built from holes, `Watched.functions["set_{name}"]`, when every value the holes read is known
while compiling: text written in the code, or a name read off a constant, such as `attribute.name` in a walk over a
class's attributes, where each copy knows its attribute ([Known while compiling](#known-while-compiling)). Such a
lookup is the declaration it names, as if the name were written out: a `var` holding it is known, an `if` on it keeps
only its branch, and a `Spite.Call` through it is the direct call, its arguments filled by name. This copies a fresh
object's attributes onto a kept one through their setters, so that whatever a setter does (telling observers, say)
happens for each:

```gdscript title=copy_through_setters/watched.spite
var label = ""
var told = 0

func set_label(value: String) {
    told = told + 1
    attributes["label"].value = value
}
```
```gdscript title=copy_through_setters/copy_through_setters.spite entry
var console = Console()

func CopyThroughSetters() {
    var kept = Watched()
    var fresh = Watched()
    fresh.label = "fresh"
    copy_attributes(kept, fresh)
    console.print(kept.label, kept.told)
}

func copy_attribute(attribute: Spite.AttributeDeclaration, kept: Watched, fresh: Watched) {
    var name = attribute.name
    var setter = Watched.functions["set_{name}"]
    if setter {
        var call = Spite.Call(setter, kept)
        call.arguments["value"] = fresh.attributes[attribute].value
        call.call()
    } else {
        kept.attributes[attribute].value = fresh.attributes[attribute].value
    }
}
```
```output
fresh 1
```

The walk makes `copy_label` and `copy_told`; in each, `name` is known, so `setter` is that attribute's setter and
the copy is the plain call `kept.set_label(fresh.label)`, which the compiler inlines. Since every attribute has a
setter ([functions_and_operators.md](functions_and_operators.md#every-attribute-has-a-setter-and-a-getter)), the
lookup finds one for `told` as well, the one the compiler writes, so `if setter` holds in every copy and the `else`
branch is never compiled: `copy_told` is the plain write `kept.told = fresh.told`. The `else` stays legal, for a
lookup that finds nothing (`functions["reset_{name}"]` for an attribute with no `reset_` function), where it is the
branch kept.

A setter or getter the compiler writes has no declaration object while running, so a `var` naming one is only tested
(`if setter`) and called through (`Spite.Call(setter, kept)`); anything else is refused (`'setter' is 'set_told',
which the compiler writes for every attribute and keeps no declaration of while running: ...`). A walk over
`.functions` lists the functions a class writes, as a lookup by a name known only while running does.

A hole that reads a value known only while running leaves the lookup to run time, as above: a `Spite.Call` through
it fills no arguments.

## Reaching a reflection object

| Written | Answers |
|---|---|
| a class name, `Monster`, `Shop.Tools.Hammer` | its `Spite.Class` |
| a generic class with its codegen values, `List<Float>`, `Shop.Crate<Float>` | the `Spite.Class` of that instance; without them it is an error, since `List` alone is no class yet |
| a namespace name, `Shop.Tools` | its `Spite.Namespace` |
| an enum name, `Phase` | its `Spite.Enum`; `Phase.values` alone is the list of its values, as `Phase`s |
| `class` inside a function | the class the function answers on |
| `namespace` inside a function | that class's namespace, or `null` at the root of a program |
| `$T` in a generic class | the class it was given ([metaprogramming.md](metaprogramming.md#asking-what-a-generic-was-given)) |
| `value.class` | the class `value` really is, through a `type` or a union too; an enum value's is its `Spite.Enum`, and an object literal's is `Object` |
| `Spite.Class.instances` | every class of the program |
| `Spite.Namespace.instances` | every namespace of the program |

`class` and `namespace` are not keywords: they are members every instance has, so a local of that name takes
precedence inside its function. A class and a namespace with the same dotted name is a compile error, so a name
always means one thing.

A class name given where a `Spite.Class` is wanted (an argument, a `var` declared `Spite.Class`, a `return`) is
its class object, and `==` between a `Spite.Class` and a class name compares the two objects. Anywhere else a bare
class name is the error that says to write `Health()`, since making one is nearly always what was meant, and
`var kind = Health` with no type is that error too, as is an argument whose parameter is a `type`
(a file is a class named after the file, so `class Health {` at the top of one is "there is no 'class' keyword"):

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
type Component {
    to_debug: Spite.Function<String>
}

var console = Console()
var components = List<Component>()

func PassingAClass() {
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
    var kept = List<Component>()
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

Each class below is ordinary Spite in `common/spite/`. Every member is a get-only attribute named for what it
answers: a question is `is_...` and answers `Boolean`, a collection is a `List` named in the plural. A question
that would take an argument is mostly a collection indexed or filtered instead; the few that take one
(`has_function(name)`, `satisfies_type(type)`) are named where they are taught.

### What a reflection object always has

A reflection object is made whole, so almost nothing in it is nullable: an attribute's `.declaration`, `.class` and
`.owner`, a function's `.declaration` and `.owner`, a declaration's `.source_file` and the `.owner` of an enum, a union
or a type are always there. A function that answers nothing `.returns` `Nothing`, the empty union, and an argument or attribute
whose type is not a class (a function type, a `type`) has `.class` `Nothing`. Only what can really be absent is
nullable: `Spite.Class.namespace` and `Spite.Namespace.parent`, because the chain of namespaces ends and there is no
root object, and a bound attribute's `.value`, which is `null` when the instance's field is. A declaration's
`.returned_literal` exists only for a function whose body is one `return` of a text literal, and reading it of another
halts, naming it.

### What the compiler leaves out

Every reflection class takes all its fields in its constructor and the compiler makes it through that constructor,
collections and facts included. A class's own `_fill_` builds its collections first, then runs the constructor on the
class object, which is an object of the executable at a fixed address, so a collection that names the class itself
gets that object; a class object given only constants has no `_fill_` and is written whole
([optimizations.md](optimizations.md#a-class-object-is-static-data)). A field the program never
reads costs nothing, because the compiler leaves it out: no slot, no write, no constructor argument, and what it would
have held (a function's wrappers, its facts, its file) is not worked out. A write to such a field still
runs what its value does (a call, or an object whose making does something), and lets go of what it answered. A
program that reads no reflection has none of it, and a class object nothing reads any more, once the reads the
compiler can answer are answered, has no `_fill_` in the program.
[optimizations.md](optimizations.md#a-reflection-field-nothing-reads-has-no-slot)

Five fields stay nullable, each because the value can really be absent:

| Field | Why it can be absent |
|---|---|
| `Spite.Class.namespace` | a class at a program's root has no namespace: there is no root object |
| `Spite.Namespace.parent` | a namespace at the top has no parent, for the same reason |
| `Spite.Attribute<Value>.value` | the instance's field holds `null` when the attribute is itself nullable |
| `FunctionDeclaration._returned_literal`, behind `.returned_literal` | only a function whose body is one `return` of a text literal has one; reading it of another halts |

A class's `.functions` lists the functions it declares and the template instances generated for it, once compiling
ends and in the order they were made, so it does not depend on the order classes were compiled in. A constructor is
not one of them: it makes an instance, it does not answer on one. A class of the standard library describes its members
the same way, `Console.functions` included.

A number's, a `Boolean`'s and a `String`'s `.attributes` is empty, because the compiler stores those values itself,
and a `List`'s or `Dictionary`'s attributes are its entries, named by index or by key, not the fields of the class that
stores them. A name starting with `_` is private to its class, reflection's own fields (`_name`) included: from outside,
the error names the getter that reads it. A walk over another class's attributes still sees its private ones, so a
serializer or a column built from a walk is never silently incomplete; naming `_x` outside its class stays the error.

### `Spite.Class`

| Member | Answers | Example |
|---|---|---|
| `.name` | the class's own name, a `String`, without its namespaces however the class was reached (`.name_with_namespaces` has them) | `Monster.name` is `Monster`, and `Part.Axle.name` and a walk's `part.name` are `Axle` |
| `.namespace` | its `Spite.Namespace?`, `null` at a program's root | `Shop.Tools.Hammer.namespace` is `Shop.Tools` |
| `.attributes` | a `Dictionary<String, Spite.AttributeDeclaration>` keyed by name, in declaration order | `Monster.attributes["health"]` |
| `.functions` | a `Dictionary<String, Spite.FunctionDeclaration>` keyed by name, in the order they were made | `Monster.functions["alive"]` |
| `.enums` | a `Dictionary<String, Spite.Enum>` of the enums it declares, keyed by name, in declaration order | `Player.enums["Job"]` |
| `.unions` | a `Dictionary<String, Spite.Union>` of the unions it declares | `Component.unions["Replication"].members` |
| `.types` | a `Dictionary<String, Spite.Type>` of the types it declares | `Player.types["Named"].owner` |
| `.is_singleton` | the class has a `singleton` line | `Console.is_singleton` is `true` |
| `.is_stateful` | a function besides the constructor and `drop()` changes the object, or a singleton it binds does | `Counter.is_stateful` |
| `.is_fixed_size` | its size is known while compiling: only numbers, `Boolean`s, enums and `String`s | `Vector3<Float>.is_fixed_size` is `true` |
| `.is_list`, `.is_dictionary`, `.is_optional`, `.is_enum` | the kind of a type | `$value_type.is_list` |
| `.generics` | a `Dictionary<String, Spite.Class>`: the class each `generic` line was given, keyed by its name without the `$`, in `generic`-line order; empty for a class that is not generic | `numbers.class.generics["element_type"]` of a `List<Integer>` is `Integer` |
| `.source_files` | a `List<File>`: every file that declares or reopens the class, in load order | `Monster.source_files.first()` |
| `satisfies_type(...)` | whether it can stand in for a type, decided while compiling ([below](#whether-a-class-satisfies-a-type)) | `Integer.satisfies_type(Number)` is `true` |

`.generics` holds the same names a codegen value reads while compiling, `$list_type.element_type`; a dictionary's are `"key_type"` and `"value_type"`. A literal name on a
class known while compiling folds and is proven present, and a name the class has no `generic` line for is a compile
error. Inside a walk, a generic argument can name one: `Query<argument.class.generics["element_type"]>()`.

A class has no folder of its own: a reopening may live anywhere a program loads, so `.source_files` is the whole
list, a namespace's `.source_directories` answers its directories, and a question about a path is asked of those `File`s. One function or attribute is written in one file, which
its declaration's `.source_file` answers: a tool that rebuilds what a function's code makes can watch the file that
code comes from.

### Whether a class satisfies a type

`kind.satisfies_type(Named)` answers whether the class can stand in for a type: whether a value of it fits where
that type is wanted, by the rule the compiler checks every fit with. The argument is the name of a type written in
the code, the way a class name is written for a `Spite.Class`:

| Written | Answers `true` for |
|---|---|
| a `type` the code can name, `Named` | a class with each of its members, typed as the type says ([values_and_types.md](values_and_types.md#inline-types-and-duck-typing)) |
| a union the code can name, `Pet` | each of its member classes |
| a class, `Monster` or `List<Integer>` | that class, its generic arguments included |
| `Number` | a class answering a number's operators: every number class, `Byte` to `Double`, and a program's own |

`Number` is the compiler's own `type`, the operators a number answers: `sum`, `subtract`, `multiply`, `divide` and
`remainder` (`+`, `-`, `*`, `/` and `%`), each taking the class and answering it, and `equals`, `less_than` and
`greater_than` (`==`, `<` and `>`), each taking the class and answering a `Boolean`. It is broad, so a program's own
number class fits it the way any class fits a `type`: a fixed-point or big-number class of a custom math library
with those eight functions satisfies `Number`, and one missing `remainder` does not. A program names it here and
cannot declare it, nor hold a value typed `Number`. `$value_type.is_number` is narrower: it asks whether a type is
one of the library's number classes, the ones stored as machine numbers.

The rule is the one the compiler checks every fit with ([values_and_types.md](values_and_types.md#what-fits-a-type)),
numbers included: `Integer.satisfies_type(Textual)` is `true`, as `var shown: Textual = 5` compiles, and an enum
satisfies a `type` that asks only for `to_string`.

The answer is decided while compiling, so it costs nothing at run time: asked of a class named in the code, `$T`, or
a walk's `attribute.class`, it is a `true` or a `false`, and an `if` on it keeps only the branch taken. That is how a
walk branches on numbers:

```gdscript title=walk_numbers/monster.spite
var name = "troll"
var health = 10
var speed = 2.5
```
```gdscript title=walk_numbers/walk_numbers.spite entry
var console = Console()
var monster = Monster()

func WalkNumbers() {
    monster.attributes.each(double)
    console.print(monster.name, monster.health, monster.speed)
}

func double(attribute: Spite.AttributeDeclaration) {
    if attribute.class.satisfies_type(Number) {
        monster.attributes[attribute].value = monster.attributes[attribute].value * 2
    }
}
```
```output
troll 20 5
```

The copy of `double` compiled for `name` has no `if` and no multiplication; the copies for `health` and `speed` are
the multiplication alone. Asked of a class known only at run time, `value.class` of a value reflection reads, it is an error
that names the test that works on a value, `if held.class == Number`, which narrows it
([control_flow.md](control_flow.md#a-value-narrowed-by-a-type-it-may-fit)): `'held.class.satisfies_type(...)' is
answered while compiling, and 'held.class' is a class known only at run time: to test whether the value fits
'Number', write 'if held.class == Number', which narrows it to that type inside`. An argument that is not a type's
name is an error too.

### `Spite.Namespace`

| Member | Answers | Example |
|---|---|---|
| `.name` | the last segment | `Shop.Tools.name` is `Tools` |
| `.name_with_namespaces` | the dotted name | `Shop.Tools.name_with_namespaces` is `Shop.Tools` |
| `.parent` | the namespace above, a `Spite.Namespace?` | `Shop.Tools.parent` is `Shop` |
| `.classes` | the classes declared directly in it | `Shop.Tools.classes.count()` |
| `.namespaces` | the namespaces directly below it | `Shop.namespaces` |
| `.source_directories` | a `List<Directory>`: every directory its classes come from, in load order | `Shop.source_directories.first()` |

There is no root namespace object: a namespace at the top has `null` as its `.parent`.

### `Spite.Enum`

An enum is a class of its own, and its reflection object is a `Spite.Enum`, not a `Spite.Class`: `Phase` named as a
value, `phase.class` on one of its values, or the class that declares it asked for its enums.

| Member | Answers | Example |
|---|---|---|
| `.name` | the enum's own name | `Phase.name` is `Phase` |
| `.name_with_namespaces` | its name through the class that declares it | `Phase.name_with_namespaces` is `Game.Phase` |
| `.owner` | the `Spite.Class` that declares it | `Phase.owner` is `Game` |
| `.values` | a `Dictionary<String, Spite.EnumValue>` keyed by name, in declaration order | `phase.class.values["update"].number` |

`Phase.values` written on the enum itself stays the list of its values as `Phase`s, the list `each` walks. Like
every `[ ]`, `Phase.values[0]` answers a `Phase?`, even with a number written in the code, so narrow it with
`crash Phase.values[0]` before use; `Phase.values.find_by_name(text)` reads text as the enum, also a `Phase?`. The
declarations are the `.values` of the `Spite.Enum`, reached through a value (`phase.class.values`) or a variable
that holds one. A value's own conversions are not reflection: `var number: Integer = phase` and
`var phase: Phase? = text` are the casts every enum has ([values_and_types.md](values_and_types.md#enums)).

```gdscript title=enum_declarations/enum_declarations.spite entry
enum Part {
    'wing'
    'tail' = 7
    'beak'
}

var console = Console()

func EnumDeclarations() {
    var tag: Part = 'tail'
    var described = tag.class
    console.print(described, described.owner)
    var tail = described.values["tail"]
    crash tail
    console.print(tail.name, tail.number, tail.index)
    var names = described.values.keys().join(" ")
    console.print(names)
}
```
```output
Part EnumDeclarations
tail 7 1
wing tail beak
```

### `Spite.EnumValue`

The declaration of one value of an enum.

| Member | Answers | Example |
|---|---|---|
| `.name` | the value's name | `tail.name` is `tail` |
| `.number` | its whole number, the one `var number: Integer = value` gives | `tail.number` is `7` |
| `.index` | its place in declaration order, 0 for the first | `tail.index` is `1` |

### What a class declares

An enum, a union and a `type` belong to the class that declares them, never to a namespace. Another class names one
through its class (`Component.Replication`), and a walk over a class reaches them all through reflection: the class's
`.enums`, `.unions` and `.types`, each a dictionary of declarations keyed by name like `.attributes`.

```gdscript title=class_declarations/component.spite
union Replication {
    Mirrored
    Local
}

type Sized {
    size: Integer
}

var replication: Replication = Mirrored()

func measure(item: Sized): Integer {
    return item.size
}
```
```gdscript title=class_declarations/mirrored.spite
var size = 3
```
```gdscript title=class_declarations/local.spite
var size = 1
```
```gdscript title=class_declarations/class_declarations.spite entry
var console = Console()

func ClassDeclarations() {
    var replication = Component.unions["Replication"]
    crash replication
    var members = replication.members.keys().join(", ")
    console.print(replication.name_with_namespaces, "holds", members)
    var types = Component.types.keys().join(", ")
    console.print("types:", types)
    var component = Component()
    var mirrored = Mirrored()
    var size = component.measure(mirrored)
    var local = Local()
    var small = component.measure(local)
    console.print(component.replication.class, size, small)
}
```
```output
Component.Replication holds Mirrored, Local
types: Sized
Mirrored 3 1
```

### `Spite.Union` and `Spite.Type`

| Member | Answers | Example |
|---|---|---|
| `.name` | its own name | `replication.name` is `Replication` |
| `.name_with_namespaces` | its name through the class that declares it | `Component.Replication` |
| `.owner` | the `Spite.Class` that declares it | `replication.owner` is `Component` |
| `.members` | a union's: a `Dictionary<String, Spite.Class>` of its member classes, keyed by their names with namespaces, in declaration order | `replication.members["Network.State"]` |

A `Spite.Type` answers `.name`, `.name_with_namespaces` and `.owner`.

The language's own empty union is read by its name: `Nothing.members` is empty, its `.name_with_namespaces` is its
plain name (`Nothing`), and its `.owner` is the library's `Spite` class. There is no union of every class
([values_and_types.md](values_and_types.md#every-place-names-what-it-holds)): `Spite.Class.instances` lists the classes
of the program.

### `Spite.FunctionDeclaration`

A class's `.functions` is a `Dictionary<String, Spite.FunctionDeclaration>`: what each function is, bound to no
object, so nothing in it can be called by itself.

| Member | Answers | Example |
|---|---|---|
| `.name` | the function's name | `alive.name` is `alive` |
| `.arguments` | a `List<Spite.Argument>`, in order | `hurt.arguments[0].name` is `amount` |
| `.returns` | exactly the answer it declares: a `Spite.Class` for a class (`Integer?` too), a `Spite.Union` for a union and for `Nothing` (the empty union, also when none is declared), a `Spite.Type` for a `type`. `.name` and `.name_with_namespaces` are read on any of them, and `declared.returns == Integer` or `== Nothing` compares it | `alive.returns` is `Boolean` |
| `.is_resumable` | it can reach a wait (a sleep, a file or socket read), so it is also compiled as a state machine that can pause there and be resumed ([concurrency.md](concurrency.md)) | `load_each.is_resumable` |
| `.returned_literal` | the text literal it returns | `greeting.returned_literal` is `grr` |
| `.reads` | a `List<Spite.AttributeDeclaration>`: every attribute of its class its code reads, in declaration order | `update_each.reads.map_names()` |
| `.writes` | a `List<Spite.AttributeDeclaration>`: every attribute of its class its code writes | `update_each.writes.count()` |
| `.source_file` | the `File` it is written in: the reopening's file for a function a reopening declares | `alive.source_file.path` |
| `.hole`, `.index` | in a template over functions: the text that stands in the function's name for the template's word, and its place | `phase.hole` is `update` |

A declaration runs on an object through a `Spite.Call` ([above](#a-call-built-before-it-runs)), and a function value
given where a `Spite.FunctionDeclaration` is wanted is its declaration (`describe(troll.alive)`).

### `Spite.Function`

An object's `.functions` is a `Dictionary<String, Spite.Function>`: each function bound to that object.

| Member | Answers | Example |
|---|---|---|
| `.declaration` | its `Spite.FunctionDeclaration`, which answers `.arguments`, `.returns`, `.is_resumable`, `.returned_literal`, `.reads`, `.writes` and `.source_file` | `troll.functions["alive"].declaration.returns` |
| `.name` | its name | `troll.functions["hurt"].name` is `hurt` |
| `call_function()` | calls it on its owner when it takes no arguments, dropping what it answers; on one that takes arguments it halts | `troll.functions["roar"].call_function()` |

A function named without calling it (`troll.alive`) given where a `Spite.Function` is wanted is the same bound function.
What the function is (`.arguments`, `.returns` and the rest) is asked of its `.declaration`: `function.returns` is
the error `a Spite.Function has no 'returns' of its own: write 'function.declaration.returns'`. In the same way,
whether an attribute's class is a singleton is `attribute.class.is_singleton`, never `attribute.is_singleton`.

#### A function value is a `Spite.Function`

A function value is an instance of `Spite.Function`, so it fits any `type` whose members `Spite.Function` has
(`type Shown { to_debug: Spite.Function<String> }`) like any other object. Named where it is made
(`var held: Shown = shout`), it is the bound
function above. Read from a place that only holds it at run time, such as an attribute
`var on_press: Spite.Function<Nothing>? = null` walked through `fresh.attributes` with `fresh: Shown`, it is the
same `Spite.Function` as if it were named there: the compiler knows every function the program takes as a value, finds
the one held by its signature and its address, and answers its declaration. `.owner` is answered on a function value
named in the code (`shout.owner`), where its class is known; a `Spite.Function` listed by `.functions` has none, since
the object whose `.functions` was read is it. A function's name is its
declaration's, so `.name` is never empty. A `null` there stays `null`.

### `Spite.Call`

| Member | Answers | Example |
|---|---|---|
| `Spite.Call(declaration, instance)` | a call of a `Spite.FunctionDeclaration` named in the code on an instance of its class, nothing run yet; a declaration known only while running is refused | `Spite.Call(MoveSystem.functions["run_each"], system)` |
| `.arguments` | its arguments by name, each a typed field of the class the compiler makes for the declaration, written once before the call runs; `null` fills a nullable parameter | `call.arguments["position"] = position` |
| `call()` | runs it, answering what the function answers, typed from the declaration | `var shown = call.call()` |

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
| `.index` | its place in declaration order, 0 for the first | `Monster.attributes["health"].index` is `1` |
| `.declaration` | on a bound attribute: its `Spite.AttributeDeclaration`, so `attribute.declaration.name` and `attribute.name` agree | `troll.attributes["health"].declaration` |
| `.owner` | inside a walk over an instance: the walked object | `attribute.owner` |
| `.source_file` | on a declaration: the `File` it is written in | `Monster.attributes["health"].source_file` |
| `.value` | on an instance's attribute only: that instance's value, typed by `Spite.Attribute<Value>`, read and written (`set_value(value)`, through the attribute's own class) | `troll.attributes["health"].value` |

`.name` is the name as it is written. Another spelling of it is the `String`'s own:
`attribute.name.camel_case()` is `buyPrice` and `attribute.name.pascal_case()` is `BuyPrice`
([standard_library.md](standard_library.md#snake-camel-and-pascal-case)), and in a walk, where the name is known,
the call is worked out while compiling.

A class's `.attributes` holds `Spite.AttributeDeclaration`s: `.name`, `.class`, `.index` and `.source_file`, never
a `.value`; whether its class is a singleton is `attribute.class.is_singleton`. An instance's are `Spite.Attribute<Value>`s bound to it, `Value` being what the attribute holds: those members, read
through the declaration, plus `.value`. What a walk hands its function is decided by the parameter's type: over
an instance, `troll.attributes.each(describe)` hands a `Spite.Attribute<Integer>` parameter each of troll's attributes
that holds an `Integer`, and a `Spite.AttributeDeclaration` parameter every declaration, as
`Monster.attributes.each(describe)` would. A class has no values to bind, so `Monster.attributes.each(describe)` with
`describe(attribute: Spite.Attribute<Integer>)` is `'describe'
takes a 'Spite.Attribute', an attribute bound to an instance, and 'health' here is a declaration of 'Monster', which
has no value: declare the parameter 'Spite.AttributeDeclaration', or walk an instance's attributes`.

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
    var update = Movement.functions["update_each"]
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
like `movement.remember`, or one read from `value.class.functions` of a value known only at run time. The program carries what
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
    var update = Runner.functions["update_each"]
    if update {
        console.print(update.name, "changes its label:", update.arguments[1].is_mutated)
    }
    if not Runner.functions["draw_each"] {
        console.print("no draw_each")
    }
    var walked = Runner.functions.filter(walks_each)
    walked.each(describe)
}

func walks_each(function: Spite.FunctionDeclaration): Boolean {
    return function.name.ends_with("_each")
}

func describe(function: Spite.FunctionDeclaration) {
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
var counted = Shop.classes.count_is_singleton()
var written = update_each.writes.map_names()
```

## Known while compiling

Most reflection is asked of something the compiler can already identify: a class, namespace or enum named in the
code, `$T`, `class`, `namespace`, and everything read from one of them. Such an object is a **constant**, and
reflection on it is ordinary code that costs nothing at run time:

- **A question folds.** `Monster.is_singleton`, `update.is_resumable` or `argument.is_mutated` is a `true` or a
  `false` in the generated code, and an `if` on it keeps only the branch taken.
- **`each` over a constant list is unrolled.** `Monster.attributes.each(show)` calls `show` once for `name` and
  once for `health`; no list is made. A list you write naming only classes or only namespaces is such a list, `[Part, Part.Inner].each(check)`, as long as nothing writes to it. So is a walk over a class's `.enums`, `.unions` or `.types` and over a union's
  `.members`, where each member class is a constant: `Shelter.unions["Pet"].members.each(adopt)` can make one with
  `member()`. A filter or a member template over a constant list is a constant list.
- **A function handed a constant is compiled once for it.** Inside that copy the parameter is the constant
  itself, so `attribute.class` is a type, `attribute.name` is text known while compiling, and
  `monster.attributes[attribute].value` is a plain read of the field. That holds for a call with a class or a namespace
  written as the argument, on this object or another (`window.register_namespace(Theme)`, `made_by(Parts.Wheel)`), and
  for a walked declaration handed on to another function (`describe(attribute, "declared")` inside a walk over
  attributes): the callee gets its own copy for that class's attribute, so `Price.cost` and `Offer.cost` each get
  one, and a walk inside it folds as well.
- **A test in a copy folds.** `attribute.class == List<Integer>` asks for the whole class, its generic arguments
  included, so it is `false` for a `List<String>`, while `attribute.class == List` is any list; `attribute.name ==
  "health"` folds as well. An `if` on either keeps only the branch taken, and the code of the other is never
  compiled, so a branch may do what only its fields' class can (`field.value.join(",")` for a list). `$T ==
  List<Integer>` in a generic class compares the same way.
- **A constant class is a type and makes its own instances**: `part()` makes one, `Column<part>()` a column of
  them, and `value.class == part` narrows `value` to that class, so `value.render()` is a call of its function. A class known only at run time is not: a parameter, local or attribute holding a `Spite.Class` written
  where a type is (`List<kind>`, or `func made(kind: Spite.Class): kind`) is `'kind' is not a class known while
  compiling, so it cannot be a type here`.
- **Asking a constant list is a constant too**: `Part.classes.is_empty()` and `Part.namespaces.is_empty()` fold, as
  do `count()`, `filter(...)` over the list and `map_names()`, so an `if` on them keeps one branch and the branch
  not taken is never compiled. `first()`, `last()`, a whole number written in `[]` and a name in `[]`
  (`Part.classes["Alpha"]`) over a constant list of classes or namespaces are the item itself, and no list is made; `first()`, `last()` and a
  name answer it as a `T?` proven present, so a `crash` on a `var` holding one is already proven and is removed;
  `[n]` is the `T` itself. When there is no such item (an empty list, `[n]` past the end, a name no item has) the
  read stays at run time, so `[n]` past the end halts with its words and the name answers `null`.
- **A name over a class's `.functions` or `.attributes` is still read at run time**, unless it is held in a `var`
  that only an `if` and a `Spite.Call` read ([A name built while compiling](#a-name-built-while-compiling)). They are
  dictionaries, and a declaration lives only in its class's table, so on a class named in the code
  `Gear.Axle.functions["spun"]` reads that table and answers a `T?` that an `if`, `fallback` or `crash` narrows, and `Gear.Axle.attributes["turns"]`
  reads it already proven present, a `T`. What folds is a missing name: one the class has no function of is a
  constant `null`, so `if not Gear.Axle.functions["rolled"]` keeps only its branch, and an attribute name the class
  does not declare is a compile error.
- **A name that is both an enum and a class is the enum**: inside the class that declares `enum Shade`, `Shade` is
  the enum even when a `shade.spite` exists, and `Shade.values` is its values.
- **A walk over an instance's attributes reads the class's declaration** of what it stands for:
  `attribute.declaration.name`, `.index` and `.class` fold in each copy as `attribute.name` does, and so does
  `attribute.declaration.name.pascal_case()`,
  and in a template over functions `phase.returns`, `phase.is_resumable`, `phase.returned_literal`, `phase.reads`
  and `phase.writes` are what `Mover.functions["update_each"]` answers for the function the copy stands for.

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

Every folder named `part` anywhere is found by this filter over every namespace, with nothing registered. The filter
runs when the program does, so `make` is compiled once and `part()` makes whichever class it is handed. Walked with
no filter, `Spite.Namespace.instances.each(make_every_class)` is a copy of `make_every_class` per namespace, and the
walk inside each folds per class. Walked from a
namespace the code names, `Part.classes.each(make)`, `make` is compiled once for each class instead
(an error inside a copy names it, as `make_over_Part_Axle`), and `part()` is `Part.Axle()` in one copy and
`Part.Wheel()` in the other. A copy is not one of the class's `.functions`; the function itself is compiled only
when something calls it with an object known at run time or takes it as a value.

## Known only at run time

A value typed by a `type` or a union whose class is not known in its copy, a reflection object kept in an attribute
and read later, and everything typed at the REPL prompt are only known at run time. Reflection still answers their
declarations, from tables the program builds only for the classes such a read can reach: `.class`, `.name`,
`.functions` and the declarations of `.attributes`. What it does not answer is a value whose type no program could
write ([every place names what it holds](values_and_types.md#every-place-names-what-it-holds)): an instance's
attributes are reached by a name written in the code, a declaration kept in a `var` or a walk, each with its own
type, and listing them as a value is refused, since which attribute each item is would be known only while running:

```gdscript title=attribute_listing/gadget.spite
var power = 3
var label = "lamp"
```
```gdscript title=attribute_listing/attribute_listing.spite entry error
var console = Console()

func AttributeListing() {
    var gadget = Gadget()
    var attributes = gadget.attributes
    console.print(attributes.count())
}
```
```diagnostic
'gadget.attributes' lists an instance's attributes as a value, so which attribute each item is would be known only while running, and its value would have no one type: walk them, 'gadget.attributes.each(visit)' with 'visit' taking the 'Spite.Attribute<...>' it reads, index one by a name written in the code, 'gadget.attributes["health"]', or by a declaration kept in a 'var', and list their names through the class, as 'Monster.attributes.keys()'
```

Walk them instead, one function for each kind of value it reads, and the names come from the class:

```gdscript title=attribute_kinds/gadget.spite
var power = 3
var label = "lamp"
var ratio = 1.5
```
```gdscript title=attribute_kinds/attribute_kinds.spite entry
var console = Console()

func AttributeKinds() {
    var gadget = Gadget()
    gadget.attributes.each(double_whole)
    gadget.attributes.each(shout_text)
    gadget.attributes.each(show)
    var names = Gadget.attributes.keys()
    console.print(names.join(", "))
}

func double_whole(attribute: Spite.Attribute<Integer>) {
    attribute.value = attribute.value * 2
}

func shout_text(attribute: Spite.Attribute<String>) {
    attribute.value = attribute.value.upper_case()
}

func show(attribute: Spite.Attribute<Console.Printable>) {
    console.print(attribute.name, attribute.value)
}
```
```output
power 6
label LAMP
ratio 1.5
power, label, ratio
```

`attribute.value = given` is the assignment it desugars to, judged at the write's line with the assignment's words:
inside a walk it writes the walked field, whatever `Value` the function names. A `Spite.Attribute` handed on or kept
is written through the attribute's own class; through a union or a `type` (`Spite.Attribute<Console.Printable>`) its
value is read, not written: `a 'Spite.Attribute<Console.Printable>' is read through 'Console.Printable', which
several classes fit, so its value is not written through it: write the attribute through a 'Spite.Attribute' of its
own class, as 'Spite.Attribute<Integer>' for an Integer attribute`.

What needs a constant says so while compiling: a function whose result is typed by its reflection parameter cannot be
handed an object known only at run time.

```gdscript title=run_time_type/gadget.spite
var power = 3
```
```gdscript title=run_time_type/run_time_type.spite entry error
var console = Console()

func RunTimeType() {
    var power = Gadget.attributes["power"]
    var kept = List<Spite.AttributeDeclaration>()
    kept.append(power)
    var first = kept.first()
    crash first
    var made = fresh(first)
    console.print(made)
}

func fresh(attribute: Spite.AttributeDeclaration): attribute.class {
    return attribute.class()
}
```
```diagnostic
'fresh' answers a type read from a reflection object, so it is compiled once for each object the compiler can identify, and this call hands it one known only at run time: pass a class, attribute, function or argument named in the code, or walked from one
```

One function body serves both: called with a constant it is typed and unrolled, and called from the REPL with a
live object it takes the run-time path. A `--repl`, `--repl-port` or `--development` build keeps the tables for
every class, so the prompt can ask anything the program can. Outside those builds the tables hold only what some code
reads: a run-time function's `.reads` and `.writes` come from a table of the functions the program makes an object
for, a standard library function and every function of a `--hot-reload` build are not in it and asking them halts,
and so does `.returned_literal` of a function whose body is not one `return` of a text literal.

```gdscript
> Monster.attributes.map_names()
name, health
> Monster.functions["alive"].returns
Boolean
> troll.attributes["health"].value = 40
```

## Every class in the program

`Spite.Class.instances` is every class of the program, its own folder and every package it loads, wherever the
package's folder is (`load "../shared"` too), but not the standard library, in the order of their dotted names, because a class is an instance of
`Spite.Class`. No code can add a class while the program runs, so the list is a constant: a walk over it,
`Spite.Class.instances.each(bind)`, is one copy of `bind` per class, in which the class is a type, and a library finds
the components of an application with no registration line:

```gdscript
func bind(part: Spite.Class) {
    if part.has_function("render") {
        var binding = Binding<part>()
        binding.mount()
    }
}
```

The `if` folds in each copy, so a class without `render` leaves an empty copy and `Binding<part>` is made only for the
classes that have it. Walking a namespace's
`.classes` the same way is how the library's `Test` class finds the tests of a namespace without being told about them ([testing.md](testing.md)). Memory and threads are not
reflected: a program cannot ask where a value lives, how many objects exist or which are alive, so nothing it
could watch ever stands in the way of how the compiler places them.

```gdscript title=every_class_of_the_program/monster.spite
var name: String

func Monster(starting_name: String) {
    name = starting_name
}

func greeting(): String {
    return "I am {name}"
}
```
```gdscript title=every_class_of_the_program/every_class_of_the_program.spite entry
var console = Console()

func EveryClassOfTheProgram() {
    var classes = Spite.Class.instances
    var class_count = classes.count()
    var found = classes["Monster"]
    crash found
    console.print("the program has", class_count, "classes, among them", found)
}
```
```output
the program has 2 classes, among them Monster
```

A generic class is no class until it is given its classes, so the list holds its copies instead: every copy its code
writes with its classes named, `Custom<Glow>` anywhere in the program or a package it loads, whether that line runs or
not, and, inside the copy of a generic class, a copy named through its generic names (`Custom<$first_type>` in
`Pair<Bloom>` is `Custom<Bloom>`). Each copy is a class of its own: its `.name` is the generic class's (`Custom`), its
`.name_with_namespaces` says which copy (`Custom<Glow>`), and `.generics` answers the classes it was given. A copy
whose classes are inferred from a constructor's arguments, one a walk makes from its walked class (`Binding<part>()`),
and one whose generic name sits inside another argument (`Custom<List<$first_type>>`) are not listed, so a walk never
lists the classes it makes and the list stays finite. Reading the list is what makes the copies it holds: a program
that never reads it makes none.

```gdscript title=generic_copies_are_classes/custom.spite
generic $uniforms_type

var key = "custom"
```
```gdscript title=generic_copies_are_classes/glow.spite
var strength: Float = 1.0
```
```gdscript title=generic_copies_are_classes/generic_copies_are_classes.spite entry
var console = Console()

func GenericCopiesAreClasses() {
    var glow = Custom<Glow>()
    var strength = Glow().strength
    console.print(glow.key, strength)
    Spite.Class.instances.each(show)
}

func show(part: Spite.Class) {
    var shown = part.name_with_namespaces
    var given = part.generics.count()
    console.print(part.name, shown, given)
}
```
```output
custom 1
Custom Custom<Glow> 1
GenericCopiesAreClasses GenericCopiesAreClasses 0
Glow Glow 0
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
    attributes.append("heavy")
    console.print(attributes.count())
}
```
```diagnostic
the attribute 'attributes' has a name reflection gives every object: 'value.attributes' lists its attributes, and JsonReader, JsonWriter and every attribute walk read through it, and one of its own would hide it. Name it something else, like 'reflection_names_attributes'
```

The same goes for an attribute named `class` or `functions`, and for `get_class`, `get_functions` and the `set_`
forms of all three: the message names the member the function would be read as, and suggests the name with the
class's own name in it (`set_table_attributes`). The classes of the `Spite` namespace are exempt, since they are
reflection, and a local variable or a parameter may still use any of these names, since nothing reads it through an
object.

## Reflection is read-only

Every member describing the program is a getter with no setter: `common/spite/class.spite` keeps its data in
private fields (`_name`, `_namespace`, `_attributes`, ...) and answers `get_name()`, `get_namespace()` and the rest,
so `.name` reads through the ordinary getter and writing it is an error. A name starting with `_` is used only
inside its own class, so the private fields cannot be written from outside either. The one thing reflection
writes is an instance's own value, through a bound attribute's `.value`, whether the attribute is known while
compiling or only at run time ([above](#known-only-at-run-time)).

```gdscript title=reflection_read_only/reflection_read_only.spite entry error
var console = Console()

func ReflectionReadOnly() {
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
add to what a class object answers, and it adds it for every class in the program. It adds functions and may change a
default, but it cannot replace a function `Spite.Class` already has: `is_singleton`, `name` and the others stay the
library's ([packages.md](packages.md#the-spite-namespace-is-reserved)). What it adds shows in `--final-classes`: the
class is printed with the function or default you wrote, and a comment above it names the root it came from. The compiler has no warnings, so visible output is the whole
mitigation.

---

Next: [Packages, namespaces, and mods](packages.md), how folders become namespaces and how a program loads packages.
