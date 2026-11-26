# Markup: JSX as sugar

A tree of objects, the kind a user interface is built from, can be written as tags. The compiler turns the tags into
the statements you would write by hand, before it resolves a single name, so nothing after that knows tags existed:
no runtime object, no new type, no rule that holds only for tags. A function written with tags and the same function
written by hand build to the same program (D2300, which replaced D18). A class's `render()` with tags also gets the
`jsx_` members a framework uses to keep what it made ([below](#a-render-keeps-what-it-made)).

```gdscript
func render(): Theme.Window {
    return <Theme.Window title="Wardrobe" on_close={close_sidebar}>
        <View justify_content='center' padding={24} wrap>
            <Text content="Face" />
            <health_bar filled={health} missing={missing} />
            {render_badge()}
        </View>
    </Theme.Window>
}
```

is built exactly as

```gdscript
func render(): Theme.Window {
    var window_2 = Theme.Window()
    window_2.title = "Wardrobe"
    window_2.on_close = close_sidebar
    var view_3 = View()
    view_3.justify_content = 'center'
    view_3.padding = 24
    view_3.wrap = true
    var text_4 = Text()
    text_4.content = "Face"
    var health_bar_5 = render_health_bar(health, missing)
    var badge_6 = render_badge()
    view_3.children = [text_4, health_bar_5, badge_6]
    window_2.children = [view_3]
    return window_2
}
```

Markup by ordinary calls still works: a library may keep constructors with arguments and chained modifiers
(`Framework.View(heading, cells).row().gap(4)`), and a program may mix both.

## What a tag becomes

- A tag naming a class, `<View>` or a dotted `<Theme.Window>`, makes an object with no arguments:
  `var view_3 = View()`. A class whose constructor takes parameters is the ordinary error of a call missing its
  arguments, at the tag's line. A tag names its class the way code does: inside `sidebar/`, a class of that folder
  is `<ClassPicker />`, and `<Sidebar.ClassPicker />` is rewritten to it with a `fixed:` line
  ([packages.md](packages.md#a-class-of-its-own-folder-is-named-plainly)); a class of another folder keeps
  its path, `<Theme.Panel>`.
- An attribute is an assignment, so it goes through `set_<name>` when the class declares one, exactly as
  `view.padding = 24` does. `name="text"` assigns the text (with every conversion a text literal has, such as a
  `Color` from `"#ff8000"`), `name='value'` an enum value, `name={expression}` any expression, and a bare `name`
  assigns `true`. An attribute the class has no attribute or setter for is the ordinary assignment error.
- Attributes are assigned in the order written, then the children are built in order and set, all at once.

## Where a tag stands

A tag stands where a value stands on its own line: `return <View />` and `var view = <View />` (with or without a
declared type), or, in a constructor that takes no arguments, on its own line as a statement
([below](#a-constructor-assembles-its-tags)). Its statements are inserted before that line; `var view = <View />` makes the object in `view` itself.
A tag anywhere else, such as a call's argument (`show(<View />)`), is an error: no constructor is ever an argument.

## A constructor assembles its tags

In a constructor that takes no arguments, a tag can stand on its own line, as a statement (spite_compiler#675). It is
built as above and attached to its closest parent: the tag it is written inside, or, at the top level, the component
whose constructor it is, `this`. Attaching is two ordinary statements: the parent becomes the child's `parent`, and
the child is appended to the parent's `children`, in the order written. A tag inside an `if`, `while` or `switch` of
the constructor attaches to the component, never to the block, so a component holds as many top-level children as it
writes and needs no fragment. A `switch` case holding one tag, `'box': <Cube />` or `'box': return <Cube size={2} />`,
is the one statement a case holds, however many statements the tag is built as.

```gdscript
var title = "Inventory"
var children = List<Component>()
var parent: Component? = null

func Inventory() {
    <Panel title={title}>
        <Note content="first" />
    </Panel>
    <Note content="below" />
}
```

is built exactly as

```gdscript
func Inventory() {
    var panel_6 = Panel()
    panel_6.title = title
    var note_7 = Note()
    note_7.content = "first"
    note_7.parent = panel_6
    panel_6.children.append(note_7)
    panel_6.parent = this
    this.children.append(panel_6)
    var note_9 = Note()
    note_9.content = "below"
    note_9.parent = this
    this.children.append(note_9)
}
```

Each child is attached once it is whole, so a tree is linked from its leaves up, as it is built. `parent` and
`children` are what the classes declare, so a class with no `parent` is the ordinary assignment error at the tag's
line, and so is a component with no `children`. Inside such a tag, a `{list}` child the compiler can see is a list
(the same reading as [below](#an-expression-as-a-child)) is attached entry by entry, in order. A tag on its own line
anywhere else, in a function or in a constructor that takes arguments, is refused.

## A brace runs again when what it reads is written

In a constructor that assembles its tags, an attribute a tag sets from what the component's own attributes hold runs
again whenever one of them is written (spite_compiler#675, the first step of its box 2). A parent passes props after
the constructor has run (a tag makes its object with no arguments, then assigns), so this is what makes a brace show
the prop and not its default:

```gdscript
var label = "none"
var count = 0
var children = List<Panel>()

func Slot() {
    var shown = "{label} x{count}"
    <Panel title={shown} />
}
```

`slot.label = "cup"` sets the panel's title to `"cup x0"`. The compiler keeps the panel in a private
`_live_panel_7`, writes the assignment again as a function of its own, `_live_0()`, and calls it from the end of the
setter of every attribute it reads, `set_label` and `set_count`, writing those setters where the class has none
(`attributes["label"].value = value`, then the call). A local the brace reads joins it: `_live_0` makes `shown` again
before it assigns, so a local is never a stale copy. Nothing else in the constructor runs again.

Only what can be followed is live: an attribute of the class with a type the desugar can read (a declared type, or
a default that is a number, a `Boolean`, a text, a making or a `List`), the locals the brace reads, what another
object it reaches holds, the changes of a list or dictionary it reads, and what a function of the class it calls
reads. A tag inside an `if`, `while` or `switch` runs once, in the constructor, as before. A class with a `render()`
keeps its kept render instead.

### A local given again

A local the brace reads joins it with every line that gives it a value before the tag: its `var`, a later
assignment at the constructor's top level, and an `if`, `while` or `switch` at the top level that assigns it. The
piece runs those lines again, in the order written, before it assigns:

```gdscript
func Slot() {
    var shown = "none"
    if count > 0 {
        shown = "{label} x{count}"
    }
    <Panel title={shown}/>
}
```

`_live_0()` makes `shown`, runs the `if`, then sets the title, so `slot.count = 2` shows `"cup x2"`. A line after the
tag is not part of it, as it was not when the constructor ran. A local given a value inside a block among the
children, or inside a top-level statement that also writes tags, cannot be run again on its own: it is a read the
compiler cannot follow ([below](#a-read-the-compiler-cannot-follow)).

### Another object's attribute

A brace that reads an attribute of another object, `label={item.name}` with `var item = Item()`, runs again when
that object's attribute is written, and when `item` itself is given another object. The compiler decides it over the
whole program while compiling (D2755): only the classes some live piece reads get a hidden list of the pieces
watching each object, a private `_jsx_writes` (an `Observer<String>`, made the first time a piece watches the object),
and the setter of each attribute read tells it the attribute's name; a class no piece reads keeps its plain setters.
The setter is the one the class writes, with the telling appended, or one the compiler writes.

The piece fills and empties that list itself: after it runs, `_live_watching_0()` reads `item` again and, when it
holds another object than last time, leaves the old one's list and joins the new one's, so a piece hears only the
object it shows. A chain is followed link by link (`item.owner.name` watches `item` for `owner` and the owner for
`name`), and so is a list entry (`items[0].name`). A function called on the object (`item.describe()`) counts as
reading every attribute of its class.

### A list a brace reads

A `List` or `Dictionary` the brace reads (`items.count()`, `items[0]`, `prices["tea"]`) is followed through its own
`subscribe`: every change of the list runs the piece again, and assigning another list moves the subscription to it,
the same way as another object above.

### A call reads what its function reads

A brace calling a function of the class, `title={describe()}`, reads what that function's own lines read, and what
the functions they call read, the same way as the brace's own lines.

### A live piece only reads

A live piece runs again whenever what it reads is written, so it must not write itself: a brace, a block or a call
that assigns an attribute of the component (`count = count + 1`), writes through one (`item.name = "cup"`), or
changes a list or dictionary it holds (`items.append(item)`), in its own lines or in a function of the class it
calls, is refused (spite_compiler#675 box 5):

```
shelf.spite:9: error: 'if shown' is live, so it runs again whenever what it reads is written, and a live piece only reads: it writes in 'count = count + 1'. Write it in a function a handler calls, and let the piece read what it wrote
```

Locals are its own, so a loop's counter and the tags it builds are not writes. A write inside a function of another
class it calls is not found yet.

### A read the compiler cannot follow

A read the compiler cannot follow while compiling makes the piece run again whenever any attribute of the component
is written, and the compiler reports where, as a note that does not stop the build:

```
slot.spite:6: note: '{first.name}' reads 'first.name', which the compiler cannot follow while compiling, so it runs again whenever an attribute of 'Slot' is written
```

These are: a member of what a call answers (`pick().name`), a member of a local whose value is not a chain of
attributes (`var first = pick()`, then `first.name`), a member of a parameter or local of a called function, a call
through a function value kept in an attribute or local, and a local given a value where it cannot be run again
([above](#a-local-given-again)). An object whose class the compiler cannot read (a `type`, a union, or a class of the
standard library other than a list or dictionary) is reported the same way, and its writes are not heard.

### A live block among the children

An `if`, `while` or `switch` that writes tags, at the constructor's top level or among the children of a tag there,
is a live block (spite_compiler#675 box 3): it is written again as a function of its own, `_live_<n>()`, which the
constructor calls in its place and which runs again whenever something it reads is written, its condition and every
line and brace inside it alike. Its tags take the place they were written in among their parent's children: each run
removes the tags the last run attached there (their `parent` becomes `null`) and puts the new run's tags in, in
order, between the children before and after the block, so the tags written after it stay where they are.

### A tag in a live block is matched

A tag in a live block is not made again on every run. Each tag written in the block keeps the objects its last run
made, and a run takes one back before it makes a new one:

- a tag with a `key` takes the kept object whose `key` equals its own: `===` when the class declares `key` as one
  of the program's classes, `==` for a number, a text or another value;
- a tag without one takes the kept objects in order, the first run's first object for the first tag made.

A kept object taken back keeps its own state and its own live pieces, and gets every attribute the tag writes
again; one no run takes back is dropped with the block's other leftovers. The children a tag in the block writes
are attached again on each run, after the ones its own constructor attached, so a kept object's own children stay.
`key` is an ordinary attribute, so a class with tags in a live block that are keyed declares it.

### A brace among the children

A brace among a tag's children, `{rows()}` or `{chosen}`, is live the same way when it reads something that can be
written: it runs again as a whole, and its tags replace the ones it attached last time, in their place
(spite_compiler#675 boxes 2 and 4). A call returning tags, `{render_rows()}`, is live for what its function reads
directly; the tags it returns keep their own live braces, so a write one of them reads runs that tag's piece, not
the call. A `List` it answers is attached entry by entry.

## Statements among the children

Inside a tag a constructor assembles, a line among the children that starts with `if`, `while`, `switch` or `var` is
a statement, written as anywhere else, and the tags it writes attach to the tag around it, never to the block. So a
child shown only sometimes is an `if`, and a child per entry is a `while`:

```gdscript
func Inventory() {
    <Panel title="Inventory">
        var used = items.count()
        <Note content="{used} of {capacity}" />
        if has_choice {
            <Note content="chosen" />
        }
        var index = 0
        while index < items.count() {
            var item = items[index]
            <Note content={item.name} />
            index = index + 1
        }
    </Panel>
}
```

The statement runs where it is written, between the children before and after it, so `used` is counted before the
note that shows it is built. A text that starts its own line with one of those four words followed by a space is read
as that statement; put such a text in an attribute, `content="if ..."`. A tag that a constructor does not assemble
(one after `return` or `var name =`) builds its `children` as one list, so a statement there is refused: build the
list in a local and hand it as `children={list}`.

## Children

A tag's children are set, not added: they are built first, leaves first, and assigned to `children` as one list
literal in the order written, `view_3.children = [text_4, health_bar_5, badge_6]`, always a list, even for one child
(D2546). It is an ordinary assignment, so it goes through `set_children` when the class declares one, and the list
takes its element type from the attribute it is assigned to: with `var children = List<Component>()`, `Component` a
`type`, every child is checked against `Component` when the program is built, and a child that does not fit is the
ordinary error. A class that takes no children declares no `children`, so a tag of it holding children is the
ordinary error at the tag's line: `a Label has no attribute or function 'children'`. Spite declares `children` on no
class; the library a tag's class comes from does.

## An expression as a child

`{expression}` between tags is one child, one entry of the list, in its place among the nested tags (D2547). A name
goes in as it is; any other expression is held in a local first, named for the function it calls without `render_`
(`badge_6` for `render_badge()`), since a call is never an argument. A child shown only sometimes is built into a
list with an `if` and handed as the attribute, below.

A whole list is not a child: hand it as the `children` attribute, `<View children={buttons} />`, which is the same
assignment. A tag that sets `children` and holds children too is refused, and so is a child the compiler can see is a
`List` or a `Boolean` (there is no `condition && <Tag />`). It reads that from what the file declares, since it builds
the statements before any name is resolved: a literal, a comparison, a parameter, a local or attribute with a
declared type or a value it can read, and a call of a function of the same class. A list it cannot see is never
silent when the element type is a class or a `type`, since a list does not fit it: `'List<...>' does not fit
type 'Component': it has no function 'describe'`.

## Text between tags

Text between tags, `<Text>Face</Text>`, is the `content` attribute: `text_4.content = "Face"`. Its spaces and line
breaks are each one space, and a hole joins it: `<Text>{count} items</Text>` assigns `"{count} items"`. A class with no
`content` attribute or setter makes it the ordinary assignment error. A `{` or `<` cannot be written in text; put
such text in an attribute, `content="a < b"`.

## A lowercase tag calls a function

`<health_bar filled={health} missing={missing} />` calls `render_health_bar` of the same class, written in the same
file. The attributes are matched to its parameters by name and passed in the order it declares them:
`render_health_bar(health, missing)`. So a function's parameters are its tag's props: `func render_weapon(grade: Integer)`
is used as `<weapon grade={1} />`, and each value goes into its parameter with every conversion a value has there, as it
would in the hand-written call (`grade="3"` gives an `Integer`, `tint="#ff8000"` a `Color`, `rarity='rare'` an enum value,
a bare `shiny` a `true`). Its answer is held in a local and is one child like any other.

A lowercase tag takes no children: nested tags or `{expression}` children inside it are refused, so a component
written as a function stays simple and builds what it holds itself. A parameter named `children` is an ordinary one,
given as `children={parts}`.

## A render keeps what it made

A class whose `render()` takes no arguments and holds tags gets more than `render` itself: the desugar also adds the
members a framework needs to keep the objects a render made and to change only what a write changed (the batched
form, spite_compiler#471). Every function of them starts with `jsx_` and every attribute with `_jsx_` (private, so a
framework copying public attributes never copies them), and a function's declaration's `.source` is `"compiler jsx"`
(a function written in the source says `"written"`), so a framework finds them through reflection:
`Card.functions["jsx_update"]`, or a walk keeping the functions whose `.source` is `"compiler jsx"`. `render()` still
builds a fresh tree each time it is called.

| Member | What it does |
|---|---|
| `jsx_replay(): Boolean` | runs the render on the kept objects: each class tag takes the object it made last time, or makes one the first time, and a tag in a branch not taken lets its object go; a tag inside a `while` keeps one object per pass. It answers `true` when anything was made or let go, or a lowercase tag (a `render_` function) built something, so a framework links the tree again |
| `jsx_root()` | what the last replay answered, `null` before the first |
| `jsx_update(bits: Integer): Boolean` | the update for the bits that changed: an attribute a tag sets outside every `if`, `while` and `switch` from what no local of the render holds is one binding and runs alone when a bit it reads is set; when a bit the rest of the render reads is set (a branch's condition, a loop's list), it replays instead and answers what the replay answers |
| `jsx_attach(queue: List<Card>?)` | the framework hands its queue: the first write after an update appends the object to it once, and once per frame the framework runs `jsx_update(card.jsx_pending())` for each queued object; `null` detaches |
| `jsx_pending(): Integer` | the bits marked since the last update |
| `jsx_replays_each_frame(): Boolean` | `true` when the render reads something no write marks (an attribute of another object, an attribute of its class with no type the desugar can read), so a framework replays it every frame |
| `jsx_release()` | lets every kept object go and stops watching lists |

Which bits a binding reads is the binding's `.reads` (its own function, `jsx_bind_<n>`), and what the rest of the
render reads is that of `jsx_flow`, a copy of the render without its bindings that nothing calls; both are folded
while compiling ([reflection.md](reflection.md#what-a-function-reads-and-writes)), and the update learns them once.

### What a write marks

Each attribute of the class with a default whose type the desugar can read (a declared type, or a default that is a
number, a `Boolean`, a text or a making like `List<Item>()`) has a bit, the first 30 in declaration order, and a setter the
desugar adds: `set_title(value: String)` stores the value and marks the bit, for a number, a `Boolean` or a text
only when the value changed (`!=`); any other value marks it on every write. A setter the class writes itself keeps
its body and marks the bit at its end. A constructor writes the storage, so it marks nothing.

### A list a render reads

A `List` attribute is watched through its own `Observer<ListChange<Item>>` from the first replay until
`jsx_release()`: `items.append(x)` marks `items`'s bit as assigning a new list does. Each replay watches the list the
attribute holds then, so a setter never subscribes (a setter that did would reach whatever the list's observers
reach, and the framework's own proofs saw that).

## Generated names

A generated local is named for its tag in snake case and the line it starts on: `window_2`, `view_3`, `health_bar_5`.
A second one with the same name in the function adds a count, `view_3_2`. So a crash report reads like the code.

## Positions

Every generated statement carries the line of the tag, attribute or child it came from, so an error, a crash report and
`spite.frame` point at that line: an attribute on a line of its own is
reported at its own line, whether it is set through a setter or passed to a `render_` function, and only what is
about the tag as a whole (an unknown tag, a parameter left out) is reported at the tag's line. A call written by hand
over several lines does the same: an error about one argument names that argument's line.

## How it is written

The formatter writes tags in one style, as it writes everything else: a tag and its attributes on one line when they
fit in 120 columns, otherwise one attribute per line with the closing `>` or `/>` on its own line; each child on its
own line one level deeper; text with its holes on one line, and a tag holding only text on the tag's line when it
fits (`<Label>{count} items</Label>`); a tag with no children self-closing, with one space before `/>` on its line
(`<Ui.Image />`) and none before an opening tag's `>` (`<Ui.Text>`), whichever way it was written; `name="text"` and
`name='value'` without braces. A file that holds tags is rewritten by the fixes found while reading it, and not by
those found after building, which it tells as errors instead.

## What the compiler refuses

```
'<health_bar>' calls 'render_health_bar', and this class declares no 'render_health_bar': declare it, or name a class to make an object
'<health_bar>' sets 'colour', and 'render_health_bar' has no parameter 'colour': it takes filled, missing
'<health_bar>' leaves out 'missing', a parameter of 'render_health_bar': set it on the tag, as in 'missing={value}'
'<health_bar>' holds children, and a lowercase tag calls 'render_health_bar', which takes none: build them inside 'render_health_bar', or name a class whose 'children' takes them
'<View>' sets 'children' and holds children: give them one way, as nested tags or as 'children={list}'
'{buttons}' is a list, and a child is one entry of the children list: hand the whole list as 'children={buttons}' on '<View>', with no nested children
'{is_open}' is a Boolean, and a child is one entry of the children list: there is no 'condition && <Tag/>', so build the list with an 'if' and hand it as 'children={list}'
a tag stands only after 'return' or 'var name =', which it is built before: build it into a local there and pass the local
'<Note>' stands on its own line in 'show', and only a constructor that takes no arguments attaches a tag written there: write it in 'Inventory()', or after 'return' or 'var name ='
'<View>' holds the statement 'if shown \{' among its children, and only a tag a constructor that takes no arguments assembles holds statements: build its children in a local list and hand it as 'children=\{list}'
```
