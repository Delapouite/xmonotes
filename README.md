# xmonotes

Personal journey to learn Haskell by understanding Xmonad.

Here's the deal: I'm more of JavaScript kind of guy. My Xmonad config has been working great for many years.
But now I need to make deeper adjustments and my Haskell-fu is still weak.

You know the feeling. You can *guess* what some lines do, but it's more of a hit & miss experience.

Let's fix that!

I hope it may also help you along the way. Feel free to contribute.

## Key mapping

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
s = [ 2*x | x <- [0..], x^2 > 3 ]
```

`s` is the set (or list in our case) we want to generate. What's on the left side of `|`, `2*x` is the **output expression**, the expression producing each item in our list.

But where's the `x` coming from? From the right side of `|`. Two parts there, separated by the comma:
- The first one is the **input set** from which we *extract* `x` values.
The extraction is done by the `<-` operator.
- The second one is the **predicate**. It's an expression evaluating to a boolean. If true we feed x to the output.

OK. Back to our snippet. What do we have here?

The output expression at the beginning: `((m .|. modm, key), windows $ f i)`.
The right hand side of `|` looks a bit more complicated. We've got two `<-`:
- `(i, key) <- zip (XMonad.workspaces conf) [ xK_grave , xK_1 , xK_2 , xK_3 , xK_4 , xK_5 ]`
- `(f, m) <- [(W.greedyView, 0), (W.shift, shiftMask)]`

No predicate to be seen, they're in fact not mandatory. [Another example](https://en.wikipedia.org/wiki/Comparison_of_programming_languages_(list_comprehension) to the rescue:

```haskell
pyth = [(x,y,z) | x <- [1..20], y <- [x..20], z <- [y..20], x^2 + y^2 == z^2]
```

To generate those Pythagorean tuples `x`, `y` and `z` are needed in the output expression. So they are extracted from 3 different input sets. Finally we have a predicate.
One thing to notice is that to extract `y` in `y <- [x..20]`, we can use `x`.
