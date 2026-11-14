# Markup: JSX as sugar

A tree of objects, the kind a user interface is built from, can be written as tags. The compiler turns the tags into
the statements you would write by hand, before it resolves a single name, so nothing after that knows tags existed:
no runtime object, no new type, no rule that holds only for tags. A function written with tags and the same function
written by hand build to the same program (D2300, which replaced D18).

```gdscript
func render(): Anything? {
    return <Theme.Window title="Wardrobe" on_close={close_sidebar}>
        <View justify_content='center' padding={24} wrap>
            <Text content="Face"/>
            <health_bar filled={health} missing={missing}/>
            {buttons}
            {render_badge()}
        </View>
    </Theme.Window>
}
```

is built exactly as

```gdscript
func render(): Anything? {
    var window_2 = Theme.Window()
    window_2.title = "Wardrobe"
    window_2.on_close = close_sidebar
    var view_3 = View()
    view_3.justify_content = 'center'
    view_3.padding = 24
    view_3.wrap = true
    var text_4 = Text()
    text_4.content = "Face"
    view_3.append_child(text_4)
    var health_bar_5 = render_health_bar(health, missing)
    view_3.append_child(health_bar_5)
    view_3.append_children(buttons)
    var badge_7 = render_badge()
    var children_7: List<Anything?> = [badge_7]
    view_3.append_children(children_7)
    window_2.append_child(view_3)
    return window_2
}
```

Markup by ordinary calls still works: a library may keep constructors with arguments and chained modifiers
(`Framework.View(heading, cells).row().gap(4)`), and a program may mix both.

## What a tag becomes

- A tag naming a class, `<View>` or a dotted `<Theme.Window>`, makes an object with no arguments:
  `var view_3 = View()`. A class whose constructor takes parameters is the ordinary error of a call missing its
  arguments, at the tag's line.
- An attribute is an assignment, so it goes through `set_<name>` when the class declares one, exactly as
  `view.padding = 24` does. `name="text"` assigns the text (with every conversion a text literal has, such as a
  `Color` from `"#ff8000"`), `name='value'` an enum value, `name={expression}` any expression, and a bare `name`
  assigns `true`. An attribute the class has no attribute or setter for is the ordinary assignment error.
- Attributes are assigned in the order written, then the children are added in order.

## Where a tag stands

A tag stands where a value stands on its own line: `return <View/>` and `var view = <View/>` (with or without a
declared type). Its statements are inserted before that line; `var view = <View/>` makes the object in `view` itself.
A tag anywhere else, such as a call's argument (`show(<View/>)`), is an error: no constructor is ever an argument.

## Children

A nested tag is built first, leaves first, then added with `parent.append_child(child)`. The library a tag's class
comes from declares `append_child` and `append_children`; Spite adds neither.

## An expression as a child

`{expression}` between tags is always added as children, with `parent.append_children(...)`, even when it is one
value (D2302):

- a `List` is passed as it is: `parent.append_children(buttons)`;
- anything else is passed as a one-item list, held in a local first since a construction is never an argument:
  `var children_7: List<Anything?> = [badge_7]`, then `parent.append_children(children_7)`. A name is passed as it
  is; any other expression is held in a local before that (named for the function it calls, `badge_7` for
  `render_badge()`), since a call is never an argument. A child that answers `Anything?` and is `null` goes in its
  one-item list and the library skips it, which is how a child is shown only sometimes: a `render_badge()` that
  answers `null` when there is no badge. So the library's `append_children` takes a `List<Anything?>`.
- A `Boolean` is an error: there is no `condition && <Tag/>`.

The compiler tells a list from one value by what the file declares, since it builds the statements before any name
is resolved: a literal, a comparison, a parameter, a local or attribute with a declared type or a value it can read,
and a call of a function of the same class. A list it cannot see is wrapped like one value, which is never silent:
a list literal takes its values one by one, so the ordinary error refuses it at the child's line (`this call passes a
whole List<Anything?>, but the last parameter is written with '...', so it takes the values one by one`); give the
list a local with its type (`var shown: List<Anything?> = ...`).

## Text between tags

Text between tags, `<Text>Face</Text>`, is the `content` attribute: `text_4.content = "Face"`. Its spaces and line
breaks are each one space, and a hole joins it: `<Text>{count} items</Text>` assigns `"{count} items"`. A class with no
`content` attribute or setter makes it the ordinary assignment error. A `{` or `<` cannot be written in text; put
such text in an attribute, `content="a < b"`.

## A lowercase tag calls a function

`<health_bar filled={health} missing={missing}/>` calls `render_health_bar` of the same class, written in the same
file. The attributes are matched to its parameters by name and passed in the order it declares them:
`render_health_bar(health, missing)`. Nested tags go to a parameter named `children: List<Anything>`, built first in a
local (`var children_5 = List<Anything>()`, then `children_5.append(...)` for each); a function with that parameter and
no nested tags is given an empty list. Its answer is held in a local and added like any child.

## Generated names

A generated local is named for its tag in snake case and the line it starts on: `window_2`, `view_3`, `health_bar_5`.
A second one with the same name in the function adds a count, `view_3_2`. So a crash report and `locals` at a
breakpoint read like the code.

## Positions

Every generated statement carries the line of the tag, attribute or child it came from, so an error, a crash report,
`spite.frame` and a breakpoint (`break file.spite:4`) point at the line of the tag. A breakpoint on a line inside the
tags stops before the first statement built from that line.

## How it is written

The formatter writes tags in one style, as it writes everything else: a tag and its attributes on one line when they
fit in 120 columns, otherwise one attribute per line with the closing `>` or `/>` on its own line; each child on its
own line one level deeper; text with its holes on one line, and a tag holding only text on the tag's line when it
fits (`<Label>{count} items</Label>`); a tag with no children self-closing; `name="text"` and
`name='value'` without braces. A file that holds tags is rewritten by the fixes found while reading it, and not by
those found after building, which it tells as errors instead.

## What the compiler refuses

```
'<health_bar>' calls 'render_health_bar', and this class declares no 'render_health_bar': declare it, or name a class to make an object
'<health_bar>' sets 'colour', and 'render_health_bar' has no parameter 'colour': it takes filled, missing
'<health_bar>' leaves out 'missing', a parameter of 'render_health_bar': set it on the tag, as in 'missing={value}'
'<health_bar>' holds children, and 'render_health_bar' has no parameter 'children: List<Anything>' to take them
'{is_open}' is a Boolean, and a child is an object, a list of them or null: there is no 'condition && <Tag/>', so answer null from a 'render_' function when there is nothing to show
a tag stands only after 'return' or 'var name =', which it is built before: build it into a local there and pass the local
```
