Some evolving rationale for style decisions.

# packaging

- Do use a valid Nim identifier for your package name; this is loosely enforced by Nimble, but it's a rule nonetheless.
- Don't use `.nim` or `nim-` or `nim` in your package name; we all know it's nim, that's why we're here -- use keywords if you want to influence search results.
- Don't use a name that starts with a Capital letter; that's just annoying.
- Do use git in preference to other VCS; 99% of all packages use git and (for this reason) Nimph won't support anything else.
- Do use tags as often as possible.
- Do use https://semver.org/ to inform your tag names.
- Don't use `v.1.2.3` as a tag; `1.2.3` is sufficient and easier to parse.
- Don't use `v1.2.3` as a tag; `1.2.3` is sufficient and easier to parse.
- Do use tags that aren't version numbers, like `foobar`, in preference to specifying a commit hashes.
- Do use `bump` and `nimph tag` to keep your tags synchronized with your `.nimble` file.
- Do make sure that your repository name matches your package name.
- Don't use underscores unless they are truly helpful:
  - `ast_pattern_matching` is a good choice
  - `html_canvas` is clearly a mistake
- Don't add `nim.cfg` files into your repository; these are conventionally expected to be replaced by the package manager in order to setup the local build environment.
- Do add `project.nim.cfg` files to your repo -- these won't be touched by package managers and you can put special `--define` switches (or whatever) into these files.
- Don't use `config.nims`; see the subsequent section...
- Do include your dependencies for your tests in your `.nimble` requirements.

## NimScript

- NimScript is a poor choice for build tooling; you cannot `getCurrentDir()` without shelling out... C'mon, man, be serious.
- It's slow to run, every time it runs, and it usually runs more often than
your project.
- It turns your configuration from static into dynamic, which requires that
anything that depends upon your configuration (like, say, a package manager)
must evaluate your configuration, which means it slows down any package
management operations performed on anything that requires your package.
- Worst of all, it imposes a limited subset of Nim on the user; a dialect that you have to debug using different assumptions and expertise.  Just, no.
- If you need dynamic build-time behavior, write a Nim program, compile it, and run it.  You may be surprised at how much better an experience this is.

# `continue` and other forms of "early return"
- It's lazy control-flow.  Do not use it.
- Prefer "dominating" conditionals that allow the structure of the code to highlight control-flow.
- A very early `return` is preferable to an `else:` clause that dominates the remainder of the `proc` body.
- Set the `result` later for observability reasons.

## uses of `continue` that aren't too smelly 
```nim
case $NimMajor & "." & $NimMinor
of "1.2":
  if gc > arc:
    continue     # arc arrived in 1.2; ignore anything later
of "1.4":
  if gc > orc:   # orc arrived in 1.4; ignore anything later
    continue
else:
  discard
quit "i expected this to work on " & $gc
```

# `result` and `return`
- Omit `result =` for any single-statement body; it's noise.
- Use `return` with an argument when the `proc` has a return type and you are making an early return; it's more explicit.
- Use `result` over `return` unless using `result` would make you look like an asshole; `result` allows for later changes to the `proc` without necessitating control-flow changes to older code.

```nim
proc foo(x: ref Bar): Bar =
  if x.isNil:
    # very early returns read well
    return nil   # be explicit with return value
  x.bif = true
  # ...
  # setting result later also helps refactoring
  result = x
```

# super-dominating conditionals

I find the former easier to read (and modify) than the latter. This will be
controversial, but you are entitled to disagree. :wink:

## Do
```nim
if foo != nil:
  if foo[] > 3:
    result = foo[] + 7
```
## Don't
```nim
if foo != nil and foo[] > 3: result = foo[] + 7
```
# tuples

- Do not use them as a "lightweight object".  If you find yourself defining a multi-line `tuple` in a `type` section, you're probably doing it wrong.
- If you expose a `tuple`, some nimpleton will rely upon its unique semantics and prevent you from making the inevitable shift to an `object`.
- `distinct tuple` doesn't work correctly.
- Do use them to return multiple values.
- Do use them for simple deconstruction assignment when it improves readability:
```nim
let (x, y) = (80, 25)
```

# exceptions

- Do **not** use exceptions for control-flow.  Use `Option`, `Result`, or a `tuple` instead.
- Do use exceptions for **exceptional** scenarios that you find yourself unable to handle otherwise.
- Do use exceptions without regard to performance; `--exceptions:goto` is the default for ARC/ORC and it's fast.
- Use `try` *expressions* only without an `except` clause.

```nim
# elegant syntax for longer exception messages
raise Defect.newException:
  &"it's bad because x={xAxisValue} and y={yAxisValue}"
```

# miscellaneous

- Don't specify `void` as a return-type; just omit a return-type.
- Don't change the type of shadowed `proc` arguments, which is confusing; _do_ change the mutability.
- Use `except as` in preference to `getCurrentException`; it's easier to read.
- Use `case ... elif:` sparingly; it's hard to read.
- Use `isNil` in preference to `== nil` because it's less error-prone with respect to `==[T](a, b: ref T): bool`.
- Don't use value types for object inheritance unless you know what you're doing; use `ref object` instead.
- **Never** use `cast[X](y)` when `X(y)` (conversion) will do; you're giving up safety needlessly.
- **Never** use `ptr` when `ref` will do; you're giving up safety needlessly.
- `if` expressions start on a new line and are indented when they span more than one line; thus they line-up with each other while suggesting a sub-scope:

```nim
let x =
  if foo:
    bar
  else:
    var bif = bar
    bif.add 42
    bif
```

# loops
- Use named `block` in preference to temporary variables:
```nim
block found:
  for n in foo.items:
    if n.isNil:
      break found
  echo "no nil present"
```
- Specify `pairs` and `items` calls explicitly.
- `pairs` and `items` follow the iterating instance because that's the important information to convey early:
  1. **do** `for n in foo.items:`
  1. **don't** `for n in foo.items():`
  1. **do** `for key, value in foo.pairs:`
  1. **don't** `for k, v in pairs(foo):`


# method-call syntax, command syntax, function-call syntax

## Do
```nim
if x.isNil:
  result = 3
```
- Reads as *if `x` is `nil`*.

## Don't
```nim
if isNil(x):
  result = 3
```
- Reads as *is nil x?*.

## Do
```nim
var s: string
s.add "foo"
s.add "bar"
```

- Procedures that can apply to multiple types should be prefixed with the first argument for easy early disambiguation.
- Single arguments don't need `()`.

## Don't
```nim
var s: string
add(s, "foo")
add s, "bar"
```

- Knowing that I'm performing an `add` isn't really all that useful.
- The `()` are simply noise.
- But removing them can actually hurt readability!

## Do
```nim
discard 3.sizeof
```
- Behaves like a property with relatively low cost of invocation, like a field?  Field notation.

## Don't
```nim
discard sizeof(3)
```
- Looks like a call to `sizeof` could have performance ramifications or side-effects because, hey, it's a function call.

## Do
```nim
proc specialOperation(x: int; y = 13.0) = echo "work was performed"
var z = 44
specialOperation z
specialOperation(z, y = 15.0)
```

- One argument?  No `()`.  The important bit to convey here earliest is the special operation.
- More arguments?  More `()`.

## Don't
```nim
proc specialOperation(x: int; y = 13.0) = discard
var z = 44
z.specialOperation
z.specialOperation y = 15.0    # yeah, no
```
- I'm thinking it's a field but... it's discarded, so it must be a call.

## Do
```nim
proc square(x: int): int = x * x
proc halved(x: float): float = x / 2.0
var y = square x
var z = x.halved
```
- Operations and properties are easily inferred.

## Don't
```nim
proc squared(x: int): int = x * x
proc half(x: float): float = x / 2.0
var y = squared(x)
var z = x.half
```
- C'mon, man...

## Do
```nim
var x = 0
while x < 10:
  echo x
  inc x
```
- Reads as _increment x_.

## Don't
```nim
var x = 0
while x < 10:
  echo x
  x.inc
```
- Reads as _something's missing here_.

## Don't
```nim
var x = 0
while x < 10:
  echo x
  x.inc()
```
- Reads as _Hi!  I'm new to Nim!_.
