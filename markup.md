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
declared type). Its statements are inserted before that line; `var view = <View />` makes the object in `view` itself.
A tag anywhere else, such as a call's argument (`show(<View />)`), is an error: no constructor is ever an argument.

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
```
