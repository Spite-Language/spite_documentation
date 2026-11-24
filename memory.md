# Memory

Spite uses **reference counting**, JavaScript-like. A scalar (a number, a `Boolean`, an enum value) is a plain
value, copied wherever it goes. Everything else (a class instance, `List<T>`, `Dictionary<Key, Value>`, `String`, a
union, an object literal) is a **reference**: assigning it, passing it, storing it in a field, a list or a
dictionary, and returning it all share the exact same object. There is no reference syntax to write: a reference
is the default. When the last reference to an object goes (a scope ends, a field is overwritten, an element is
removed), the object's `drop()` runs, if it has one, its own references are released, and it is freed. There is
no garbage collector and no pause.

A parameter follows the same rule. A number passed to a function is a copy: a callee that adds ten to its `Integer`
parameter leaves the caller's number alone. An object is passed by reference, so the callee sees and can change the
caller's object, but assigning the parameter a different object only changes which object the callee's name holds. A
`Dictionary` that holds an object holds that same object, so a change made through the original shows when the object
is read back through the dictionary.

What that costs while the program runs is a count in every object and one addition or subtraction each time a
reference is kept or let go (plain arithmetic, atomic only where another thread may count it at once) and nothing
else: no collector, no runtime to ship. A singleton is not counted at all, and a number, a `Boolean` or an enum
value is never an object.

## No lifetimes, and no value classes

Some languages keep memory safe by making you describe who owns what and for how long, and the compiler checks the
description: a great deal of annotation that has nothing to do with what the program is for. Spite gets the same
safety from a smaller set of tools: reference counting, a value that may be missing is a `Monster?` that must be
narrowed with `fallback` or `crash` before it is used.
Where the safe form needs a rule, the compiler has it. You do not write lifetimes.

Some languages also ask you to decide, for each type, whether it is a value or a reference. In Spite that is the
compiler's question, never yours. There are only classes, and every object behaves as a shared object. The compiler
chooses registers, the stack, inline storage or the heap for each use, whichever keeps that behaviour cheapest, and
work it could have shown to be unnecessary and did anyway is a bug in the compiler, not something you tune by
splitting a type in two.

## Do: know that sharing is visible

Two names can hold the same object. A mutation through either one shows up through the other, because there is
only ever one object:

```gdscript title=sharing_basics/box.spite
var label: String

func Box(starting_label: String) {
    label = starting_label
}
```
```gdscript title=sharing_basics/sharing_basics.spite entry
var console = Console()

func SharingBasics() {
    var original = Box("a")
    var alias = original
    alias.label = "b"
    console.print("original label", original.label)
}
```
```output
original label b
```

`==` on two class instances calls the class's `equals`
([functions_and_operators.md](functions_and_operators.md#every-operator-is-a-function)): one the class declares, or
the one every class without it is given, which compares each attribute with its own `==`, so a copy equals its
original. A value typed by a `type` or a union compares through the class it holds, never by which object
it is. `a === b` asks whether two names hold the very same object; write it only where that is
what you mean, since it keeps the compiler from sharing a copy with its original. A `String` always compares by
content.

### A name read from a list names the item, until you assign it

`var box = boxes[0]` names the box the list holds, so `box.label = "b"` changes the box in the list. Assigning
`box` is different: `box = Box("c")` makes the name hold another box and leaves the list holding the first one.
When nothing reads `box` after that, the assignment was meant for the list, and the compiler says so:

```gdscript
var boxes = List<Box>()

func relabel(replacement: Box) {
    crash boxes[0]
    var box = boxes[0]
    box.label = "b"
    box = replacement
}
```

```
'box' was read from 'boxes[0]', so assigning it here changes only 'box', and nothing reads 'box' after:
'boxes' still holds the old item. Write 'boxes[0] = replacement' to replace the item, or give the new value a
'var' of its own
```

The same holds for a name answered by a function that returns an item of a list, such as a lookup that answers
`values[row]`: the error names the class holding the list and its function that writes it. The compiler follows
such a function through every call it makes, however many, and a `Dictionary` item is treated like a `List` item. It looks only at
the last place the local is named, in its own block or in the branches of an `if` or a `switch` that is the last
statement naming it: a local assigned twice, a local that is read afterwards, one made by a constructor or a
`copy()`, and an answer from a function that may return something else are all left alone.

## Do: call `copy()`/`deep_copy()` for an independent object

`copy()` makes a fresh object with the same attributes (still shared references for any attribute that is
itself a reference). `deep_copy()` recurses, giving every reference-kind attribute its own independent copy
too. Inside a class, a bare `copy()` or `deep_copy()` copies the object itself, as every own function is called
by name:

```gdscript title=copy_basics/box.spite
var label: String

func Box(starting_label: String) {
    label = starting_label
}
```
```gdscript title=copy_basics/copy_basics.spite entry
var console = Console()

func CopyBasics() {
    var original = Box("a")
    var independent = original.copy()
    independent.label = "b"
    console.print("original label", original.label)
    console.print("independent label", independent.label)
}
```
```output
original label a
independent label b
```

`deep_copy()` makes all new objects and keeps the shape of what it reaches: one object of the original is one new
object in the copy, however many ways the copy reaches it. Two attributes holding the same leaf, a list and an
attribute holding it, a union holding it, or a list holding it twice all hold one new leaf in the copy, shared there
as in the original and never shared with the original. Through a cycle the same holds, so a child's `parent` in the
copy is the copied parent, and a back reference in the copy holds the copy of its object when the same `deep_copy()`
copied it. A copy that starts at the child alone leaves the `parent` on the original object, since nobody copied the
parent. A `Group` listing the child before its parent copies them as one graph, in either order, and a list or a
dictionary that two objects of the graph hold is one list or dictionary in the copy too. `copy()` is shallow for a
list or a dictionary too: the copy holds the very same items, so a change made to an item shows through both.

```gdscript title=shared_leaf/leaf.spite
var name: String

func Leaf(new_name: String) {
    name = new_name
}
```
```gdscript title=shared_leaf/box.spite
var left: Leaf
var right: Leaf

func Box(new_left: Leaf, new_right: Leaf) {
    left = new_left
    right = new_right
}
```
```gdscript title=shared_leaf/shared_leaf.spite entry
var console = Console()

func SharedLeaf() {
    var leaf = Leaf("shared")
    var box = Box(leaf, leaf)
    var copied = box.deep_copy()
    var one_leaf = copied.left === copied.right
    var is_the_original = copied.left === leaf
    copied.left.name = "changed"
    console.print(one_leaf, is_the_original, copied.right.name, leaf.name)
}
```
```output
true false changed shared
```

An attribute typed as a union or a `type` is deep copied as the class it holds when the program runs, so a `Cargo`
holding a `Crate` gets a new `Crate`, and a union or `type` value answers `deep_copy()` itself. A `String`, a number,
an enum value and a singleton are shared by the copy, since nothing can change them through it. A class that
declares its own `func deep_copy()` is copied through that function wherever a deep copy reaches it. An object that
holds a `ForeignCallback` is never copied: a copy or deep copy that would reach one is a compile error
([foreign libraries](foreign_libraries.md#for-as-long-as-the-program-keeps-it-foreigncallback)).

## Components in lists, handed to a system

An engine keeps each component in its own `List` and hands a system one entity's components at a time. A list
holds its items like any other reference, so `positions[0]` is the position itself: a system that writes it writes
the list's item, and the item may be kept, passed on or read after the list grows. How a list stores its items is
the compiler's choice, never yours: a list whose items are never held anywhere else may keep them inline, one block
of their attributes, and nothing the program can see changes ([collections.md](collections.md#one-list)).

### A row of items, for one call

The row is an object literal of items, and the system takes it as a `type`:

```gdscript title=row_doc/position.spite
var left = 0.0
```
```gdscript title=row_doc/velocity.spite
var across: Float

func Velocity(new_across: Float) {
    across = new_across
}
```
```gdscript title=row_doc/mover.spite
type Moving {
    position: Position
    velocity: Velocity
}

func update_each(moving: Moving) {
    moving.position.left = moving.position.left + moving.velocity.across
}
```
```gdscript title=row_doc/row_doc.spite entry
var console = Console()
var positions = List<Position>()
var velocities = List<Velocity>()
var mover = Mover()

func RowDoc() {
    var position = Position()
    positions.append(position)
    var velocity = Velocity(2.5)
    velocities.append(velocity)
    crash positions[0] and velocities[0]
    var tick = 0
    while tick < 4 {
        var row: Mover.Moving = {position: positions[0], velocity: velocities[0]}
        mover.update_each(row)
        tick = tick + 1
    }
    console.print(positions[0].left)
}
```
```output
10
```

A generic runner does not know the attributes of the `type` it is given, so it cannot write the literal. It fills
the row with a walk over its attributes instead, and a local of the `type` declared `= null` and filled by the walk on the
next line is a row exactly like the literal:

```gdscript title=walked_row_doc/position.spite
var left = 0.0
```
```gdscript title=walked_row_doc/velocity.spite
var across: Float

func Velocity(new_across: Float) {
    across = new_across
}
```
```gdscript title=walked_row_doc/mover.spite
type Moving {
    position: Position
    velocity: Velocity
}

func update_each(moving: Moving) {
    moving.position.left = moving.position.left + moving.velocity.across
}
```
```gdscript title=walked_row_doc/columns.spite
singleton

var position = List<Position>()
var velocity = List<Velocity>()

func size(): Integer {
    return position.count()
}
```
```gdscript title=walked_row_doc/runner.spite
generic $row_type

var columns = Columns()

func run(mover: Mover) {
    var index = 0
    while index < columns.size() {
        var row = $row_type()
        fill_attributes(row, index)
        mover.update_each(row)
        index = index + 1
    }
}

func fill_attribute(attribute: Spite.AttributeDeclaration, row: $row_type, index: Integer) {
    crash columns.attributes[attribute].value[index]
    row.attributes[attribute].value = columns.attributes[attribute].value[index]
}
```
```gdscript title=walked_row_doc/walked_row_doc.spite entry
var console = Console()
var columns = Columns()
var runner = Runner<Mover.Moving>()
var mover = Mover()

func WalkedRowDoc() {
    var position = Position()
    columns.position.append(position)
    var velocity = Velocity(2.5)
    columns.velocity.append(velocity)
    runner.run(mover)
    runner.run(mover)
    crash columns.position[0]
    console.print(columns.position[0].left)
}
```
```output
5
```

Every `[]` answers a `T?` and a row's attributes are items, so the template states each
read with a `crash` line before the fill: nothing the walk knows proves that `index` is inside each column. The
compiler writes the walk out where it is called: for `Runner<Moving>` the two lines become `crash
columns.position[index]`, `crash columns.velocity[index]` and `var row: Moving = {position:
columns.position[index], velocity: columns.velocity[index]}`, where each `crash` line's read is made once and
the literal uses it, so the row costs what the literal costs and `fill_attribute` is never called.

An engine that adds and removes components all the time keeps each one in a **sparse set**: a generic singleton
`Column<Position>` whose list is packed, so each entity sits at a different place in each column. The runner
works out those places first, one per attribute, and the walk's line reads each attribute from its own column at
its own place: `Column<attribute.class>().values[stored_row]`, after `var stored_row = found[attribute.index]`,
where `attribute.index` is the attribute's place in the `type`, with one `crash` line for the
place and one for the item (an index never holds another `[]` read).
A generic singleton made with no arguments is the same object every time, so `Column<Position>()` starts a path
that `crash` narrows like a name. The places are numbers in a `List`, so they may come from any `List<Integer>`:
the runner's own, or another object's, `fill_attributes(row, matcher.rows)`. The row may also hold a value made
from the id, such as an `Entity`, chosen per attribute while compiling:

```gdscript title=sparse_row_doc/position.spite
var left = 0.0
```
```gdscript title=sparse_row_doc/velocity.spite
var across: Float

func Velocity(new_across: Float) {
    across = new_across
}
```
```gdscript title=sparse_row_doc/entity.spite
var id: Integer

func Entity(new_id: Integer) {
    id = new_id
}
```
```gdscript title=sparse_row_doc/trail.spite
var lefts = List<Float>()
var last = 0.0
var owner = -1
```
```gdscript title=sparse_row_doc/mover.spite
func update_each(moving: SparseRowDoc.Moving) {
    moving.position.left = moving.position.left + moving.velocity.across
    moving.trail.lefts.append(moving.position.left)
    moving.trail.last = moving.position.left
    moving.trail.owner = moving.entity.id
}
```
```gdscript title=sparse_row_doc/column.spite
singleton

generic $component_type

var dense_of = List<Integer>()
var values = List<$component_type>()

func insert(entity: Integer, value: $component_type) {
    while dense_of.count() <= entity {
        dense_of.append(-1)
    }
    dense_of[entity] = values.count()
    values.append(value)
}

func dense_index(entity: Integer): Integer {
    crash dense_of[entity]
    return dense_of[entity]
}
```
```gdscript title=sparse_row_doc/runner.spite
generic $row_type

var found = List<Integer>()

func run(mover: Mover, entity_count: Integer) {
    var entity = 0
    while entity < entity_count {
        found.clear()
        var row = $row_type()
        find_attributes(row, entity)
        fill_attributes(row, found, entity)
        mover.update_each(row)
        entity = entity + 1
    }
}

func find_attribute(attribute: Spite.AttributeDeclaration, row: $row_type, entity: Integer) {
    if attribute.class == Entity {
        row.attributes[attribute].value = Entity(entity)
        found.append(entity)
    } else {
        var column = Column<attribute.class>()
        var dense = column.dense_index(entity)
        found.append(dense)
    }
}

func fill_attribute(attribute: Spite.AttributeDeclaration, row: $row_type, rows: List<Integer>, entity: Integer) {
    if attribute.class == Entity {
        row.attributes[attribute].value = Entity(entity)
    } else {
        crash rows[attribute.index]
        var stored_row = rows[attribute.index]
        crash Column<attribute.class>().values[stored_row]
        row.attributes[attribute].value = Column<attribute.class>().values[stored_row]
    }
}
```
```gdscript title=sparse_row_doc/sparse_row_doc.spite entry
type Moving {
    entity: Entity
    position: Position
    velocity: Velocity
    trail: Trail
}

var console = Console()
var runner = Runner<Moving>()
var mover = Mover()
var positions = Column<Position>()
var velocities = Column<Velocity>()
var trails = Column<Trail>()

func SparseRowDoc() {
    var first = Position()
    positions.insert(0, first)
    var second = Position()
    positions.insert(1, second)
    var slow = Velocity(1.0)
    velocities.insert(1, slow)
    var fast = Velocity(4.0)
    velocities.insert(0, fast)
    var first_trail = Trail()
    trails.insert(0, first_trail)
    var second_trail = Trail()
    trails.insert(1, second_trail)
    runner.run(mover, 2)
    runner.run(mover, 2)
    var first_steps = first_trail.lefts.count()
    var second_steps = second_trail.lefts.count()
    console.print(first_trail.owner, first_trail.last, first_steps)
    console.print(second_trail.owner, second_trail.last, second_steps)
}
```
```output
0 8 2
1 2 2
```

`find_attribute` asks about each attribute's class, and fills the row's `Entity` while it is there: that read of
`row.attributes[attribute].value` is how its body says the template walks the attributes of `$row_type`. A template whose
body reads its parameter on no value is an error, even when one of its parameters is of the class it would walk.

For `Runner<Moving>` the walk is written out as `{entity: Entity(entity), position:
Column<Position>().values[found[1]], velocity: Column<Velocity>().values[found[2]], trail:
Column<Trail>().values[found[3]]}`. Each item is counted once for the row and let go at the end of the row's block,
so a call that removes an entity from its column cannot free a component under the row.

### Arguments filled for one call

A system may take its components as arguments instead of a row, `update_each(position: Position, velocity:
Velocity)`, and a generic runner fills them with a template over the function's arguments whose plural is the
call's whole argument list ([metaprogramming.md](metaprogramming.md#walking-a-programs-structure)).
The template's line is the walked row's line, read per argument: `argument.index` is the argument's place, so
`Column<argument.class>().values[stored_row]`, after `var stored_row = rows[argument.index]`, is that component's
item. The plural stands for exactly one call the compiler writes:

```gdscript title=filled_arguments_doc/position.spite
var left = 0.0
```
```gdscript title=filled_arguments_doc/velocity.spite
var across: Float

func Velocity(new_across: Float) {
    across = new_across
}
```
```gdscript title=filled_arguments_doc/mover.spite
func update_each(position: Position, velocity: Velocity) {
    position.left = position.left + velocity.across
}
```
```gdscript title=filled_arguments_doc/column.spite
singleton

generic $component_type

var values = List<$component_type>()
```
```gdscript title=filled_arguments_doc/runner.spite
generic $system_type

enum Phase {
    'update'
}

var system = $system_type()
var found = List<Integer>()

func run(index: Integer) {
    found.clear()
    found.append(index)
    found.append(index)
    run_phases_each()
}

func run_phase_each(phase: Spite.FunctionDeclaration) {
    system.phase_each(made_arguments(found))
}

func made_argument(argument: Spite.Argument, rows: List<Integer>): argument.class {
    crash rows[argument.index]
    var stored_row = rows[argument.index]
    crash Column<argument.class>().values[stored_row]
    return Column<argument.class>().values[stored_row]
}
```
```gdscript title=filled_arguments_doc/filled_arguments_doc.spite entry
var console = Console()
var positions = Column<Position>()
var velocities = Column<Velocity>()
var runner = Runner<Mover>()

func FilledArgumentsDoc() {
    var position = Position()
    positions.values.append(position)
    var velocity = Velocity(2.5)
    velocities.values.append(velocity)
    runner.run(0)
    runner.run(0)
    crash positions.values[0]
    console.print(positions.values[0].left)
}
```
```output
5
```

For `Runner<Mover>` the call is written out as `system.update_each(Column<Position>().values[found[0]],
Column<Velocity>().values[found[1]])`, after the `crash` lines that narrow those reads: `update_each` writes the
columns' own items, and `made_argument` is not called for it. Called anywhere else, `made_argument` is an
ordinary function. The template may also fill a whole walked row for an argument that is a `type`.

## `drop()` runs once, right before the object is freed

A class may define a zero-argument `func drop() { ... }` for cleanup (closing a handle, clearing a
back-reference). The compiler calls it automatically the moment the last reference goes away, never by name:

```gdscript title=drop_basics/resource.spite
var console = Console()
var name: String

func Resource(new_name: String) {
    name = new_name
}

func drop() {
    console.print("dropped", name)
}
```
```gdscript title=drop_basics/drop_basics.spite entry
var console = Console()

func DropBasics() {
    var resource = Resource("first")
    console.print("using", resource.name)
}
```
```output
using first
dropped first
```

## Self-referential classes and unions just work

A tree, a linked list, or any other recursive structure needs nothing beyond an ordinary field, since a field of
a class type already holds a reference:

```gdscript title=tree_basics/leaf.spite
var held_value: Integer

func Leaf(starting_value: Integer) {
    held_value = starting_value
}

func total(): Integer {
    return held_value
}
```
```gdscript title=tree_basics/branch.spite
var left: TreeBasics.TreeNode?
var right: TreeBasics.TreeNode?

func Branch(left_node: TreeBasics.TreeNode, right_node: TreeBasics.TreeNode) {
    left = left_node
    right = right_node
}

func total(): Integer {
    var result = 0
    if left {
        result = result + left.total()
    }
    if right {
        result = result + right.total()
    }
    return result
}
```
```gdscript title=tree_basics/tree_basics.spite entry
union TreeNode {
    Leaf
    Branch
}

var console = Console()

func TreeBasics() {
    var first_leaf = Leaf(1)
    var second_leaf = Leaf(2)
    var small_branch = Branch(first_leaf, second_leaf)
    var third_leaf = Leaf(3)
    var tree: TreeNode = Branch(small_branch, third_leaf)
    var total = tree.total()
    console.print("total", total)
}
```
```output
total 6
```

When the last reference to a structure goes, its `drop()`s run in a fixed order: the object's own first, then what it
holds (a list's items in order, then its attributes in the order the class declares them), each with everything it
alone holds before the next. Freeing takes no stack per level, however deep the structure and wherever its depth is: a
chain of a million links, a tree a million levels deep through its first child or its `left`, and a list whose items
hold lists all free in a loop, in that same order. Build a structure the way that reads best; its depth never decides
whether it can be freed. Only a structure deep through what function values hold still takes stack per level.

## Cycles are freed

Two objects that hold each other, directly or through several hops, would keep each other's count above zero. You never
write a special reference for that, nothing is refused for it, and every cycle a program writes is freed: a parent
that owns its children and a child that knows its parent are plain references, and the compiler picks, place by
place, the cheapest way that frees the cycle. Where it reads from how the program links its objects which reference
of the cycle can hold no count, the cycle is freed exactly where its owner goes; every other cycle is freed by a count
that knows the cycle ([below](#every-other-cycle-is-freed-by-a-count-that-knows-it)). A child's `parent` is the usual case: the parent makes the child with `Child(child_name, this)`, the
constructor stores the parent in `parent`, and the parent stores the child in `children`, so the compiler reads
`parent` as the back reference and the cycle is freed with its owner.

```gdscript title=back_reference_family/parent.spite
var console = Console()
var children = List<Child>()
var name: String

func Parent(new_name: String) {
    name = new_name
}

func adopt(child_name: String) {
    var child = Child(child_name, this)
    children.append(child)
}

func drop() {
    console.print("{name} is freed")
}
```
```gdscript title=back_reference_family/child.spite
var name: String
var parent: Parent?

func Child(new_name: String, new_parent: Parent) {
    name = new_name
    parent = new_parent
}
```
```gdscript title=back_reference_family/back_reference_family.spite entry
var console = Console()

func BackReferenceFamily() {
    visit()
    console.print("the family is gone")
}

func visit() {
    var parent = Parent("ada")
    parent.adopt("grace")
    var child = parent.children.first()
    crash child
    crash child.parent
    console.print(child.name, "belongs to", child.parent.name)
}
```
```output
grace belongs to ada
ada is freed
the family is gone
```

A back reference reads `null` once its object is freed, so no read is ever dangling, and the narrowing above is the
usual one. A back reference holds no count, so a proof that it is present lasts only until a call that may let its
object go: after `child.parent.leave()` drops the last other holder of the parent, a later `child.parent.name`
that leans on the earlier `crash child.parent` stops the program with `'Child.parent' was proven present, but its
object was freed since, and a back reference holds no count: prove it again after the call that lets it go`. A call
on the back reference itself, `child.parent.leave()`, always runs on a live object, which is freed when the call
returns. Everything above is freed when `visit` returns, which is what `--debug-memory`, below, checks. The compiler picks the back reference of a cycle
from how the code links its objects: one function that sets `child.parent = owner` and stores the child in the
owner's list (or in one of its attributes, or in a dictionary) is the link, and so is a pair of functions that
together do both, however many calls deep and through however many attributes
(`box.inner.deeper.deepest.first.beta = second` in a called function), a function that calls itself one attribute
deeper each call (`pair_down(link.next, second)`) included. The owner's list may hold a union or a `type` the child belongs to
(`List<Entry>` with `union Entry { File Folder }`). A function whose name two classes declare is followed into the
one of the receiver's class, when that class is known: `this`, a parameter, an object made in the function, or an
attribute of one of these. A singleton or a generic class is never part of that link. Two objects made side by side
and then linked to each other directly (`question.answer = answer` beside `answer.question = question`) are freed
when neither leaves the function that made them: the one made later points back. So is a pair the function makes
where one object is reached through an attribute of another it made (`last.first`, an `Alpha` that `Link`'s
constructor made) and the other is held by a `var` of the function's outer block, linked both ways there or by a
function it calls (`pair_down(head, second, 2)`), when nothing the function makes leaves it and every call of the
linking function is one such: the object reached through the attribute points back, so a `drop()` that runs as the
names go may read its back attribute as `null`. Two such objects linked through one
attribute both ways are freed the same way, below. Some cycles keep their counts, since nothing proves which side
outlives the other, and the count that knows the cycle frees them: such a pair where one of them leaves its function (`return answer`), a pair linked
both ways through one attribute by another function (`first.join(second)` beside `second.join(first)`), a pair
reached through attributes on both sides, or whose linking function is also called elsewhere, a pair a
function that calls itself may link the other way one attribute deeper (the deeper calls closing a link the
program also makes the other way round), and a back attribute that cannot be null, unless no child outlives its parent
([below](#a-parent-that-cannot-be-null)). A back attribute may hold a union or a `type`
(`var holder: Holder? = null` with `union Holder { Drive Shelf }`) when every class it may hold links back that way:
each class of the union, or for a `type` each class of the program that declares all of its members, so one class
that fits the `type` and never holds the child keeps the attribute counted. There is no `Weak<T>` to write, and naming one is an unknown-type
error.

A reader that subscribes one of its functions to what it holds (`names.subscribe(changed)` in a `Row` that keeps
`names`) closes a cycle through the observer, and that one is always freed: an observer holds every subscribed
function value without a count on its object, however the value reached `subscribe` (made there, held in a local or
an attribute, read from a list, kept or answered by another function). Once that object is freed, `notify` skips it
and the observer lets go of its link at its next `notify`, when a `subscribe` or `unsubscribe` meets it, or at the
walk a `subscribe` makes whenever the links have doubled ([collections.md](collections.md#observer-tells-who-subscribed)).
A reader that also keeps its own function value closes a second cycle without the observer, and that one is freed
when the value is stored into one of the reader's own attributes, by any of its functions (`act = given` in
`keep(given)`, called as `keep(changed)`), or appended to a `List` the reader holds (`acts.append(changed)`), as long
as the program never lets it out: every mention of the attribute is a call through it, a narrowing, a store, a `null`
store or a hand-off to `subscribe`/`unsubscribe`, and every mention of the list is `append`, `insert`, `count`,
`is_empty`, `clear`, an item called directly or an item kept in a local that is only narrowed and handed to `subscribe`; and the compiler makes no
`copy()` of the class. Any other such value keeps its count (one stored in another object, one read for anything
else), and its cycle is freed by the count that knows it.

### Two objects linked both ways through one attribute

`first.partner = second` beside `second.partner = first` links two objects through one attribute, so neither side
points back. When both are made side by side in one block of a function, each under one name, and neither leaves the
function, the compiler clears the slot that closed the cycle (`second.partner`, the store written later) right before
the two names go, so the pair is freed there. That object's `drop()` reads the attribute as `null`; the other one's
still reads its partner, which is freed right after it. Nothing of this runs but the one store the compiler writes
where the names go.

```gdscript title=partner_pair_doc/person.spite
var console = Console()
var partner: Person? = null
var name: String

func Person(new_name: String) {
    name = new_name
}

func drop() {
    if partner {
        console.print(name, "is freed, still with", partner.name)
    } else {
        console.print(name, "is freed alone")
    }
}
```
```gdscript title=partner_pair_doc/partner_pair_doc.spite entry
var console = Console()

func PartnerPairDoc() {
    pair_up()
    console.print("the pair is gone")
}

func pair_up() {
    var first = Person("ann")
    var second = Person("bob")
    first.partner = second
    second.partner = first
    console.print(first.name, "and", second.name, "are partners")
}
```
```output
ann and bob are partners
ann is freed, still with bob
bob is freed alone
the pair is gone
```

A pair linked by calls of another function, made in two blocks, held under a second name, or one of whose objects
leaves its function keeps its counts, and the count that knows the cycle frees it
([below](#every-other-cycle-is-freed-by-a-count-that-knows-it)).

### A parent that cannot be null

A back attribute may be a plain `Parent`, set by the constructor and never `null`. It has no `null` to answer once
its parent is freed, so the compiler frees such a family only when it proves that no child outlives its parent:
the parent makes each child with `Child(name, this)` and keeps it only in its own list, and every function of the
program uses a child only while the parent it was read from is still held by `this`, a parameter, or a local of the
function's outer block that is never assigned again. Within that, a child can be kept in a local, read, called,
compared, narrowed, handed to your own functions, and the list read by index, `first()`, `last()`, `remove_first()`,
`remove_last()`, walked by `each` and the like with a function that takes a child, searched, counted or cleared. When
the last holder of the parent goes, its `drop()` runs, then each child's, in the list's order.

```gdscript title=parent_without_null_doc/parent.spite
var console = Console()
var children = List<Child>()
var name: String

func Parent(new_name: String) {
    name = new_name
}

func adopt(child_name: String) {
    var child = Child(child_name, this)
    children.append(child)
}

func drop() {
    console.print(name, "is freed")
}
```
```gdscript title=parent_without_null_doc/child.spite
var console = Console()
var parent: Parent
var name: String

func Child(new_name: String, new_parent: Parent) {
    name = new_name
    parent = new_parent
}

func describe(): String {
    return "{name} of {parent.name}"
}

func drop() {
    console.print(name, "is freed")
}
```
```gdscript title=parent_without_null_doc/parent_without_null_doc.spite entry
var console = Console()

func ParentWithoutNullDoc() {
    visit()
    console.print("the family is gone")
}

func visit() {
    var parent = Parent("ada")
    parent.adopt("grace")
    parent.adopt("linus")
    parent.children.each(introduce)
}

func introduce(child: Child) {
    var described = child.describe()
    console.print(described)
}
```
```output
grace of ada
linus of ada
ada is freed
grace is freed
linus is freed
the family is gone
```

Anything else keeps the counts, and the count that knows the cycle frees the family
([below](#every-other-cycle-is-freed-by-a-count-that-knows-it)): a child
returned, stored in an attribute or in a list of its own, bound in or handed to a function value, or carried as a
union or a `type`; the list kept in a local or copied; a child that adds itself to its parent's list; a `copy()` or
`deep_copy()` of either class; a `drop()` of either class that reads the parent's list or the back attribute, or calls
a function of its own class. One such use anywhere in the program keeps the attribute counted everywhere, and no child
is ever freed while something still reads it.

### Every other cycle is freed by a count that knows it

A cycle the compiler does not read as one of the forms above is still freed, and nothing is written for it. The
compiler finds while compiling every store that can close a cycle (an attribute or a list slot of an object whose
class can, through the classes it holds, hold an object of its own kind), and only the objects those stores write into
count in a way that knows the cycle. Where a function lets go of a local that may be part of such a cycle (the drop
point), the compiler checks the cycles those objects are in, and frees every one that nothing outside it holds any
more: first each of their `drop()`s runs, while all of them are still there to read, then their memory goes. Checking
costs about as much as what it walks, so a small cycle is freed at the drop point that let it go, while a program
that holds a large structure of such objects checks it ever more rarely as it grows, and at the latest once those
objects have grown fourfold, so what waits to be freed is at most three times what lives; the end of the program frees whatever
is left.
Objects that can never be in a cycle pay nothing, and neither does a store into an object nothing else holds yet:
a tree a function builds by making a node and appending its children before it hands the node on, a list a
function fills before it returns it, or one handed down to helpers that only fill it, can close no cycle, since no
object reaches what is being filled, so the compiler marks nothing there.

```gdscript title=joined_pair_doc/person.spite
var console = Console()
var partner: Person? = null
var name: String

func Person(new_name: String) {
    name = new_name
}

func join(other: Person) {
    partner = other
}

func drop() {
    if partner {
        console.print(name, "is freed, still with", partner.name)
    } else {
        console.print(name, "is freed alone")
    }
}
```
```gdscript title=joined_pair_doc/joined_pair_doc.spite entry
var console = Console()

func JoinedPairDoc() {
    pair_up()
    console.print("the pair is gone")
}

func pair_up() {
    var first = Person("ann")
    var second = Person("bob")
    first.join(second)
    second.join(first)
    console.print(first.name, "and", second.name, "are partners")
}
```
```output
ann and bob are partners
ann is freed, still with bob
bob is freed, still with ann
the pair is gone
```

The same holds for a function value an object keeps that is bound to another object which holds it back
(`box.act = row.changed` beside `row.box = box`), a list item set by index (`acts[0] = changed`), a function value read
for anything else, a class the compiler copies, a ring of a hundred thousand links, and a child that adds itself to
its parent's list. A program that runs work at once on threads frees them the same way: the check runs where no
other thread runs the program's code, so a cycle a band of a loop leaves behind, or one a waiting call lets go, is
freed at the next drop point of the program's own thread, and at the end at the latest. A function C calls from a
thread of its own (a `ForeignCallback`) waits at its start while a check walks, which takes as long as what it walks.

## `--debug-memory`

`spite --debug-memory` builds with an allocation table and prints `allocations: N frees: N` right before
the program exits. A mismatch means something leaked; when the two do not balance, it also prints a
**leaked-object summary by class name**, naming which classes' instances are still alive, which is what makes a
leaked object visible instead of an unexplained count. Every program on these pages is run this way, and must
balance. It also fills every byte it hands out that nobody has written yet with the same pattern, so a program that
reads memory it never wrote fails the same way on every run and in every environment, instead of reading whatever
the heap held. The table exists only in a `--debug-memory` build, so the other builds carry none of it. Since every cycle
is freed, a program written in Spite balances; the summary is there for a fault of the compiler or of the standard
library, and looks like this when two `Person` objects are still alive at the end:

```
spite: --debug-memory: 2 allocation(s) leaked
  Person: 2 leaked object(s)
```

The same program prints the same summary on every run.

## Memory belongs to the compiler

A program never names an address, an allocator, an allocation count or where an object lives. Memory and threads are
constructs of the language that each generator implements its own way, so nothing a program could watch ever stands in
the way of how the compiler places a value: in a register, in the frame when it never leaves its function, together with
the objects around it, or on the heap. You write the plain version, and the compiler decides.
No class stands for memory, and there is no list of the live instances of a class. Only
`--debug-memory`, above, reports what leaked, and it does so from outside the program.

A class that needs room for bytes holds a `Buffer`, a class of the library that keeps its bytes in one block and reads
and writes the numbers in them, and the compiler lays it out from what it proves
([standard_library.md](standard_library.md#bytes-are-a-buffer)). How much room a `Buffer`, a `List` or a `String`
keeps, and when it grows, is the compiler's too: the library only raises a count or writes past the end, and the
compiler makes the room before it, at least doubling it. Bytes from outside are a `Buffer` too: a file's or a
socket's bytes. Memory a C library hands out has no checked reader yet (`ForeignBytes` is decided, not built yet,
[spite_compiler#467](https://github.com/Spite-Language/spite_compiler/issues/467)). A container of your own is written over a `List` or a `Buffer`, with nothing the compiler does for
`List<T>` that it would not do for yours: the standard library's `List` is a complete one to read.

---

Next: [Metaprogramming](metaprogramming.md), writing code that writes code for every attribute of a class.
