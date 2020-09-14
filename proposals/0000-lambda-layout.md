---
author: Jakob Brünker
date-accepted: ""
proposal-number: ""
ticket-url: ""
implemented: ""
---

This proposal is [discussed at this pull request](https://github.com/ghc-proposals/ghc-proposals/pull/302).

# Lambda extension for `case`

This proposal introduces a new extension `-XLambdaCaseOf`, which introduces a
lambda-like expression capable of handling multiple clauses as well as guards.
This proposal expands the functionality of the `-XLambdaCase` extension, by
extending the `case` syntax to allow lambda expression functionality.

## Motivation

In Haskell 2010, there are two syntaxes to define a function: via the function
definition syntax, or via lambda expressions. The most obvious difference is
that the former assigns a name to the function, whereas the latter can be used
for anonymous functions. However, the differences go significantly beyond that:

 - Lambda expressions can only have a single clause, function declarations
   can have an arbitrary non-zero number of equations
 - Lambda expressions cannot have guards
 - Lambda expressions must have at least one parameter

There have been multiple attempts in the past to bring the capabilities of
lambda expressions closer to those of function declarations:

 1. The extension `-XLambdaCase` introduces a `\case` construct which allows
    lambda expression to have multiple clauses, however, only one pattern can
    be matched on. Like a regular case-expression, this can also have guards.
    [During its
    implementation](https://gitlab.haskell.org/ghc/ghc/issues/4359#note_44819)
    as well as [after it](https://github.com/ghc-proposals/ghc-proposals/pull/18),
    there were attempts to make it possible to match on multiple patterns. No
    solution was found, in part because this would make it different from
    regular case-expressions.
    - If there were an expression that had pattern matching syntax more similar
      to lambda expressions but which could also have guards and multiple clauses,
      it would be strictly more powerful than the existing `-XLambdaCase`,
      since it would be able to match on multiple patterns.
 2. The extension `-XMultiWayIf` essentially introduces standalone guards,
    simplifying the use of guards that aren't at the outermost level of a
    function declaration or case-expression. Among other things, this made it
    easier to use guards inside of lambda expressions.
    - If there were an expression similar to lambda expressions that could have
      guards, `-XMultiWayIf` would not be necessary as a workaround for guards
      in lambda expressions in those cases.
 3. During the implementation of `-XLambdaCase`,
    [some suggested](https://gitlab.haskell.org/ghc/ghc/issues/4359#note_51110)
    allowing lambda expressions to have multiple clauses. This was not
    implemented: The most obvious approach of turning `\` into a layout herald
    had the disadvantage of making some common idioms invalid.
    - The syntax extension of introduced in this proposal is fully backwards
      compatible with those idioms.

This proposal, then, aims to overcome the shortcomings of lambda expressions
and allows the `case` expression to have the same capabilities as function
declarations, which can be used instead of `-XLambdaCase`, many instances of
`-XMultiWayIf`, and most of function declaration syntax, if a user wishes to
use the new expression instead. Furthermore, it can be used in situations that
are not conveniently covered by existing constructs (see Example section). As
an example of how the `case` construct is extended, here is a definition of
filter using it:

```haskell
filter = case of
  \_ []                 -> []
  \p (x:xs) | p x       -> x : rest
            | otherwise -> rest
```

By combining the functionality of several features into one feature in
a way that's consistent with the rest of the language, it enables users who
wish to use this functionality to work with a simpler and more consistent
language.

## Proposed Change Specification

The functionality of `-XLambdaCase` is extended, according to the following
schema (for a more formal treatment, see BNF below):

```haskell
case [ scrutinee ] of
  [ Pattern_0a ] \ Pattern_1a ... Pattern_na -> Expression_a
  [ Pattern_0b ] \ Pattern_1b ... Pattern_nb -> Expression_b
  ...
```

This is equivalent to

```haskell
\var_1 ... var_n -> case ([ scrutinee, ] var_1, ..., var_n) of
  ([ Pattern_0a, ] Pattern_1a, ..., Pattern_na) -> Expression_a
  ([ Pattern_0b, ] Pattern_1b, ..., Pattern_nb) -> Expression_b
```

The `case` expression is now able to define anonymous functions. The scrutinee
may be omitted, in which case the corresponding pattern in each clause must also be
omitted. Furthermore, in each clause, between the usual pattern (if it is present) and
the arrow, a backslash and a number of patterns may be written. The
number of patterns must be consistent across all clauses, and the types of
corresponding patterns must match (e.g., the first pattern after the backslash
must have the same type for all clauses). As usual, `case` clauses can
contain guards as well.

If there is no scrutinee, it is not immediately clear what the meaning of an
expression without clauses, i.e. the expression `case of {}`, should be, since
the number of arguments to the anonymous function is not specified. The most
useful and most obvious variant is to assume that this function takes one
argument, thus, `case of {}` will be equivalent to `\x -> case x of {}`.
(Other approaches are possible, see Alternatives section.)
This expression is only valid with the `-XEmptyCase` extension.

Like the existing behavior for alternatives in `case`
expressions, and equations in function declaration syntax, it is
possible to use `where` clauses within each clause of the extended `case`
expression.

Once the [*Binding type variables in lambda-expressions*](https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0155-type-lambda.rst)
proposal is being implemented, with `-XTypeAbstractions`, `case`-expressions will also be able to bind type
variables.

### BNF of changed syntax

<table>
    <tr>
        <td><i>lexp</i></td><td>&rarr;</td><td><tt>case</tt> <i>exp</i> <tt>of</tt> { <i>alts</i> }</td>
    </tr>
    <tr>
        <td></td><td>|</td><td><tt>case</tt> <i>exp</i> <tt>of</tt> {}</td><td>(<i>with</i> <tt>-XEmptyCase</tt>)</td>
    <tr>
        <td><i>alts</i></td><td>&rarr;</td><td><i>alt<sub>1</sub></i> ; &hellip; ; <i>alt<sub>n</sub></i></td><td>(<i>n</i> &ge; 1)</td>
    </tr>
    <tr>
        <td><i>alt</i></td><td>&rarr;</td><td><i>pat</i> <tt>-&gt;</tt> <i>exp</i> [<tt>where</tt> <i>decls</i>]</td>
    </tr>
    <tr>
        <td></td><td>|</td><td><i>pat</i> <i>gdpat</i> [<tt>where</tt> <i>decls</i>]</td>
    </tr>
    <tr>
        <td></td><td>|</td><td></td><td>(empty alternative)</td>
    </tr>
    <tr>
    </tr>
    <tr>
        <td><i>gdpat</i></td><td>&rarr;</td><td><i>guards</i> <tt>-&gt;</tt> <i>exp</i> [ <i>gdpat</i></td>
    </tr>
    <tr>
        <td><i>guards</i></td><td>&rarr;</td><td>|<i>guard<sub>1</sub></i>, &hellip;, <i>guard<sub>n</sub></i></td><td>(<i>n</i> &ge; 1)</td>
    </tr>
    <tr>
        <td><i>guard</i></td><td>&rarr;</td><td><i>pat</i> <tt>&lt;-</tt> <i>infixexp</i></td><td>(pattern guard)</td>
    </tr>
    <tr>
        <td></td><td>|</td><td><tt>let</tt> <i>decls</i></td><td>(local declaration)</td>
    </tr>
    <tr>
        <td></td><td>|</td><td><i>infixexp</i></td><td>(boolean guard)</td>
    </tr>
<table>

Aside from the explicit layout using `{`, `}`, and `;`, implicit layout as described in the Haskell
report can also be used.

In expressions that have zero scrutinees and multiple guards, there is an ambiguity as to whether
the expression has multiple alternatives with one guard each or one alternative with multiple guards
(or any combination thereof). However, the semantics for these are equivalent, so this ambiguity can be
resolved in an arbitrary way.

## Examples

Using multi-way lambda expressions with guards allows shortening some definitions:

```Haskell
{-# LANGUAGE MultiWayIf, BlockArguments #-}
take' :: Int -> [a] -> [a]
take' = flip $ flip foldr (const [])
  \x more n -> if | n > 0 -> x : more (n - 1)
                  | otherwise -> []

-- becomes

take' :: Int -> [a] -> [a]
take' = flip $ flip foldr (const [])
  \of x more n | n > 0 -> x : more (n - 1)
               | otherwise -> []
```

Multi-way lambdas can always replace `-XMultiWayIf`:

```Haskell
foo = bar baz if | g1 -> a
                 | g2 -> b

-- with -XBlockArguments becomes

foo = bar baz \of | g1 -> a
                  | g2 -> b
```

`\case` can be replaced by a `\ of`-expression:

```Haskell
\case Bar baz -> Just baz
      Quux -> Nothing

-- becomes

\of (Bar baz) -> Just baz
    Quux -> Nothing
```

Lambda expressions are more powerful since they can match on multiple patterns:

```Haskell
-- \case can't be used here!
-- At least not as easily as in the previous example
\foo bar baz -> case (foo, bar, baz) of
  (Just 4, 3, False) -> 42
  _ -> 0

-- becomes

\of
  (Just 4) 3 False -> 42
  _ _ _ -> 0
```

`\ of`-expressions can be used instead of regular function declaration syntax,
potentially resulting in more concise definitions:

```Haskell
extremelyLengthyFunctionIdentifier (Just a) False = Just 42
extremelyLengthyFunctionIdentifier (Just a) True  = Just (a / 2)
extremelyLengthyFunctionIdentifier _        _     = Nothing

-- becomes

extremelyLengthyFunctionIdentifier = \of
  (Just a) False -> Just 42
  (Just a) True  -> Just (a / 2)
  _        _     -> Nothing
```

This also makes it possible to have `where` bindings that scope over multiple
equations

```Haskell
-- have to repeat the definition of `magicNumber` or place it outside the definition of
-- foo
foo (Just x) | x < 0 = ...
             | let y = blah + 1 = ...
  where blah = x + magicNumber
        magicNumber = 5
foo Nothing = magicNumber
  where magicNumber = 5

-- becomes

-- note that the first `where` clause belongs to the first `\ of`-expression
-- clause, rather than the function declaration, because it is indented further
foo = \of
  (Just x) | x < 0 -> ...
           | let y = blah + 1 -> ...
    where blah = x + magicNumber
  Nothing -> magicNumber
  where
    magicNumber = 5
```

To illustrate with some real-world examples, this section shows
how some snippets found on hackage would look if they used this new syntax:
  
megaparsec-tests-8.0.0/tests/Text/Megaparsec/Char/LexerSpec.hs
```Haskell
forAll mkFold $ \(l0,l1,l2) -> do
  let {- various bindings -}
  if | end0 && col1 <= col0 -> prs p s `shouldFailWith`
       errFancy (getIndent l1 + g 1) (ii GT col0 col1)
     | end1 && col2 <= col0 -> prs p s `shouldFailWith`
       errFancy (getIndent l2 + g 2) (ii GT col0 col2)
     | otherwise -> prs p s `shouldParse` (sbla, sblb, sblc)

-- with -XMultiWayLambda

forAll mkFold $ \(l0,l1,l2) -> do
  let {- various bindings -}
  \of | end0 && col1 <= col0 -> prs p s `shouldFailWith`
        errFancy (getIndent l1 + g 1) (ii GT col0 col1)
      | end1 && col2 <= col0 -> prs p s `shouldFailWith`
        errFancy (getIndent l2 + g 2) (ii GT col0 col2)
      | otherwise -> prs p s `shouldParse` (sbla, sblb, sblc)
```

caramia-0.7.2.2/src/Graphics/Caramia/Texture.hs:
```Haskell
return $ if
    | result == GL_CLAMP_TO_EDGE -> Clamp
    | result == GL_REPEAT -> Repeat
    | otherwise -> error "getWrapping: unexpected wrapping mode."

-- with -XMultiWayLambda and -XBlockArguments

return \of
    | result == GL_CLAMP_TO_EDGE -> Clamp
    | result == GL_REPEAT -> Repeat
    | otherwise -> error "getWrapping: unexpected wrapping mode."
```

red-black-record-2.1.0.3/lib/Data/RBR/Internal.hs
```Haskell
_prefixNS = \case
    Left  l -> S l
    Right x -> case x of Here fv -> Z @_ @v @start fv
_breakNS = \case
    Z x -> Right (Here x)
    S x -> Left x

-- with -XMultiWayLambda
_prefixNS = \of
    (Left  l) -> S l
    (Right x) -> case x of Here fv -> Z @_ @v @start fv
_breakNS = \of
    (Z x) -> Right (Here x)
    (S x) -> Left x
```

recursors-0.1.0.0/Control/Final.hs
```Haskell
map (\case PlainTV n    -> n
           KindedTV n _ -> n) binders
           
-- With -XMultiWayLambda

map (\of (PlainTV n)    -> n
         (KindedTV n _) -> n) binders
```

roc-id-0.1.0.0/library/ROC/ID/Gender.hs
```Haskell
printGender :: Language -> Gender -> Text
printGender = \case
  English -> printGenderEnglish
  Chinese -> printGenderChinese

printGenderEnglish :: Gender -> Text
printGenderEnglish = \case
  Male   -> "Male"
  Female -> "Female"

printGenderChinese :: Gender -> Text
printGenderChinese = \case
  Male   -> "男性"
  Female -> "女性"

-- With -XMultiWayLambda - this makes use of the capability to have multiple parameters

printGender :: Language -> Gender -> Text
printGender = \of
  English Male   -> "Male"
  English Female -> "Female"
  Chinese Male   -> "男性"
  Chinese Female -> "女性"
```

## Effect and Interactions

Enabling the extension enables users to use the suggested syntax. This obviates
the need for `-XMultiWayIf` and `-XLambdaCase`.

As `of` is already a keyword, no currently allowed syntax is stolen by this extension,
and the behavior of no currently legal program would be changed with the extension
enabled.

## Costs and Drawbacks

It is one additional syntactic construct to maintain, however the maintenance
cost should be fairly low due to the similarity to already existing constructs.

While this also means one additional construct to learn for beginners, the
syntax is consistent with similar constructs in the existing language, and as
such users might be surprised that a construct with these capabilities
doesn't yet exist.

## Alternatives

 - Zero clauses could be permitted. In this case, however, a way would have to be found
   to indicate how many arguments a given `\ of`-expression matches on, as otherwise, it would
   be ambiguous.
   The number of arguments a `\ of`-expression pattern matches on becomes obvious from the
   clauses, e.g. `\ of a b -> ...` clearly matches on two arguments. Without clauses, this remains
   unclear. This means it would also be unclear whether the patterns are non-exhaustive:
   Consider the expression `f = \of {} :: Bool -> Void -> a`. If the expression is supposed to match on
   both arguments, the patterns are exhaustive. If it is only supposed to match on the first argument
   and evaluate to a funtion of type `Void -> a`, it is not exhaustive. Moreover, in the former case,
   ``f undefined `seq` ()`` evaluates to `()`, whereas in the latter case, it evaluates to bottom.
   With `\case {}` this problem doesn't arise, since it always matches on exactly one argument,
   and similarly for `case x of {}`, which only matches on `x`.
   A syntax to resolve this has been proposed in the discussion: `(\of)` for matching on no arguments,
   `(\of _)` for one, `(\of _ _)` for two, and so on.
    -- TODO -- absurd patterns?
   
 - Regular lambda expressions could be extended to use layout and guards, however,
   this necessitates some potentially controversial decision on when exactly to
   herald layout, since always doing so would disallow existing idioms; these would not
   be legal when the extension is enabled:
   ```haskell
   do
     f a >>= \b ->
     g b >>= \c ->
     h c
     
   foo = \x -> do
     a x
     b
   ```
   Two alternatives would be to only herald layout
     - if a newline immediately follows the `\` or
     - if, given that token `t` is the token after `\`, the line below the one with `t` has
       the same indentation as or greater than `t`

   Both of these would avoid the problem, but both rules are dissimilar from how layout heralding
   is handled in other Haskell constructs.
   
 - `\ of`-expressions with zero patterns could only be allowed if the expression contains guards.
   This would make them somewhat less consistent, but it is how lambda expressions work
   (i.e. `\ -> ...` is illegal) and only disallows expressions that are needlessly verbose (i.e.
   `\of -> exp` can always be replaced by `exp`).
   
 - `\case` could be deprecated, since all its use cases would be subsumed by `\of`, albeit with additional
   parentheses around patterns that consist of more than one token. However, the discussion of this proposal
   has shown that such a deprecation would be a controversial
   change of its own and that some working out has to be done as to the exact details of it, thus,
   this might be better suited to being its own, separate proposal. Combined with this, an alternative to the keyword would be
   to reuse `\case` and change the way it works to the behaviour described in this proposal.
 
 - There are also other alternatives for the keyword that have been raised: `\cases` and `\mcase`.
 
 - The possibility to have a construct similar to `-XMultiWayIf` but without the keyword, i.e. using guards directly as
   an expression, was also raised in the discussion. If this were to be used instead of `\of` expressions, any pattern
   matching would have to be done with pattern guards.

## Implementation Plan

I (Jakob Brünker) will implement this proposal.
