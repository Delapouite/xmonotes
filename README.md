# xmonotes

Personal journey to learn Haskell by understanding Xmonad.

Here's the deal: I'm more of JavaScript kind of guy. My Xmonad config has been working great for many years.
But now I need to make deeper adjustments and my Haskell-fu is still weak.

You know the feeling. You can *guess* what some lines do, but it's more of a hit & miss experience.

Let's fix that!

I hope it may also help you along the way. Feel free to contribute.

## Key mapping

### As patterns

First line and weird stuff already. Haskell is that magical!

```
myKeys conf@(XConfig {XMonad.modMask = modm}) = M.fromList $ [
  -- …
]
```

What is the purpose of `@`? It's used in what's called *as patterns*. `conf` is a more convenient way to reuse `(XConfig {XMonad.modMask = modm})` in the following code.

### List comprehensions

The effects of the following snippet are well commented.
Changing [keys](https://github.com/Delapouite/xkb-walkthrough) is also trivial. But the meaning of the `|` and `<-` symbols can be quite alien.

```haskell
-- mod-[` 1 .. 0], Switch to workspace N
-- mod-shift-[~ ! .. )], Move client to workspace N

[((m .|. modm, key), windows $ f i)
    | (i, key) <- zip (XMonad.workspaces conf)
        [ xK_grave
        , xK_1
        , xK_2
        , xK_3
        , xK_4
        , xK_5
        ]
    , (f, m) <- [(W.greedyView, 0), (W.shift, shiftMask)]]
```

The code is encompassed in `[]` so its goal is to produces a list.  Since the
generating code is written between this couple of `[]`, we've got our first
clue: it's a [list comprehension](https://en.wikipedia.org/wiki/List_comprehension).
This pattern is not available in JavaScript yet but is pretty common in many languages like
Python and Ruby.

Go ahead and read the Overview section of the Wikipedia link provided above.
We're lucky, Haskell is used to illustrate their example:

```haskell
s = [ 2*x | x <- [0..100], x^2 > 3 ]
```

`s` is the set (or list in our case) we want to generate. What's on the left side of `|`, `2*x` is the **output expression**, the expression producing each item in our list.

But where's the `x` coming from? From the right side of `|`. Two parts there, separated by the comma:
- The first one is the **input set** from which we *extract* `x` values.
The extraction is done by the `<-` operator.
- The second one is the **predicate**. It's an expression evaluating to a boolean. If true we feed x to the output.

Here's the Python equivalent which may inspire future versions of ECMAScript:

```python
s = [ 2*x for x in range(100) if x**2 > 3 ]
```

`|` → `for` and `<-` → `in`. The predicate is just an `if`.

OK. Back to our snippet. What do we have here?

The output expression at the beginning: `((m .|. modm, key), windows $ f i)`.
The right hand side of `|` looks a bit more complicated. We've got two `<-`:
- `(i, key) <- zip (XMonad.workspaces conf) [ xK_grave , xK_1 , xK_2 , xK_3 , xK_4 , xK_5 ]`
- `(f, m) <- [(W.greedyView, 0), (W.shift, shiftMask)]`

No predicate to be seen, they're in fact not mandatory. [Another Haskell example](https://en.wikipedia.org/wiki/Comparison_of_programming_languages_(list_comprehension)) to the rescue:

```haskell
pyth = [(x,y,z) | x <- [1..20], y <- [x..20], z <- [y..20], x^2 + y^2 == z^2]
```

In order to generate those Pythagorean tuples, `x`, `y` and `z` are needed in the output expression. So they are extracted from 3 different input sets. Finally we have a predicate.
One thing to notice is that to extract `y` in `y <- [x..20]`, we can use `x`.

### Record access

Let's focus on this line:

- `(i, key) <- zip (XMonad.workspaces conf) [ xK_grave , xK_1 , xK_2 , xK_3 , xK_4 , xK_5 ]`

`zip` is not the most mysterious piece, as many JS libs such as Lodash provide a [similar function](https://lodash.com/docs#zip).

```js
_.zip(['fred', 'barney'], [30, 40], [true, false]);
// → [['fred', 30, true], ['barney', 40, false]]
```

On top of my config file I set workspace with no meaningful names:

````haskell
myWorkspaces = map show [0..10]`
```

And at the bottom:

```haskell
main = do
  xmonad $ defaultConfig {
    -- …
    workspaces = myWorkspaces
  }
```

So we expect to get something like that:

```haskell
[("0", xK_grave), ("1", xK_1), ("2", xK_2), ("3", xK_3), ("4", xK_4), ("5", xK_5)]
```

OK. But what about `(XMonad.workspaces conf)`?

Coming from a OO background I was expecting `XMonad.conf.workspaces`. I tried to `grep` the whole XMonad code without any success on where the magic could happen.

GHCI to the rescue!

```sh
Prelude>:m + XMonad
Prelude XMonad>:t XMonad.workspaces
XMonad.workspaces :: XConfig l -> [String]
```

A function taking a XConfig as argument and giving a List of strings.

This function is automagically created for us when using the Record way to declare data types as described in this chapter of [LYHGG](http://learnyouahaskell.com/making-our-own-types-and-typeclasses#record-syntax). It is attached to XMonad by the game of re-exports.
Hint: the exact same formula can be found on this line:

```haskell
((modm, xK_Return), spawn $ XMonad.terminal conf)
```

### KeyMask Modifiers

`(f, m) <- [(W.greedyView, 0), (W.shift, shiftMask)]]`

The `0` above is quite intriguing. It's coupled with a function `W.greedyView`. In the other tuple of the list, the function `W.shift` is this time coupled with an explicit `shiftMask`. What could be the real name of the `0` mask? Let's visit the `Grawphics.X11` module which is needed by XMonad. This [Types.hs file](https://github.com/xmonad/X11/blob/master/Graphics/X11/Types.hsc) has an interesting part:

```haskell
type KeyMask            = Modifier
#{enum KeyMask,
 , noModMask            = 0
 , shiftMask            = ShiftMask
 , lockMask             = LockMask
 , controlMask          = ControlMask
 , mod1Mask             = Mod1Mask
 , mod2Mask             = Mod2Mask
 , mod3Mask             = Mod3Mask
 , mod4Mask             = Mod4Mask
 , mod5Mask             = Mod5Mask
 }
```

Here are the 8 original KeyMasks from the X protocol (before xkb era). `0` is just a shorter way to write `noModMask`. `shiftMask` could have been written `1` but the result would have been even harder to decipher. Those masks are then used in a OR bitwise operation with the `.|.` operator.

### IO

The `return` keyword can be misleading in regards of other languages. In Haskell it's found in `do` blocks where IO steps are described. The goal of this construct is to encapsulate a normal value into a IO one.
At the opposite, the `<-` is used to extract value.

Here's a handy analogy if you already know the concept of Promises in JavaScript :

```js
let items = []
// in every case, we expect to grab a Promise
getItemsPromise = () =>
  items.length
    // `Promise.resolve` has the same encapsulation purpose than Haskell's return
    ? Promise.resolve(items)
    // `then` serves the same extraction role as Haskell's <-
    : fetch('https://example.com/items').then(res => res.json())

```
