# Implementing the Type-On-Hover Feature

[Dhall](https://dhall-lang.org/) is an implementation of a _typed lambda calculus_ (i.e. a _functional programming language_; like Haskell, but with substantially different design decisions). Arguably the most useful features one would expect from "editor integration" for a typed language like Dhall is "type-on-hover", that is, when I point the cursor at an _identifier_ (a variable name) I would like a tooltip displaying its type to appear. Like so:

![Image](/images/hover-type-intro.png)

In this article I will talk you through my implementation of "type-on-hover" as part of [dhall-lsp-server](https://github.com/dhall-lang/dhall-haskell/tree/master/dhall-lsp-server), which together with [vscode-dhall-lsp-server](https://github.com/PanAeon/vscode-dhall-lsp-server) provides editor integration for Dhall files in VSCode/[ium](https://vscodium.com/). While Dhall is a specific instance of a typed functional programming language, its relative lack of idiosyncrasies hopefully means that the following should be helpful to someone looking to implement the same feature for a different language.

## The Problem, Informally
Suppose we are given an _AST_ ([abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree)) of a Dhall file; in order for the following to make sense, assume the file passes Dhalls typechecker. We are now given a textual "position" inside the file (i.e. a line and column number) and are expected to produce the type of the identifier at that location.

Note that this leaves some room for interpretation regarding what the result should be if the position did not in fact point at an identifier. I believe the way my implementation handles such cases to be quite elegant; stay tuned!

## Dhall's AST
Dhall's AST type `data Expr s a` is defined in [Dhall.Core, lines 348-484](https://github.com/dhall-lang/dhall-haskell/blob/3a120d277f62fe83f8d9b35f14e3c93b9a6076cf/dhall/src/Dhall/Core.hs#L348-L484). Having a look at it you might feel overwhelmed by the sheer number of constructors&mdash;luckily, we only really need worry about a small subset:

- __Binders&nbsp;__ The constructors
  - `Lam Text (Expr s a) (Expr s a)` (function abstraction),
  - `Pi Text (Expr s a) (Expr s a)` (function space),
  - and `Let (NonEmpty (Binding s a)) (Expr s a)`

  each _bind_ variable names. We will need to keep track of these bound variables as we traverse the AST in order to be able to infer the types of subexpressions we might be interested in.

- __Parser annotations&nbsp;__ The precise type of Dhall AST we are dealing with is `Expr Src X`. Such an expression may contain "notes" of the form `Note Src (Expr Src X)`; Dhall's parsers uses these notes to mark each node of the AST with the location in the source code it came from. (A `Src` consists of a start and end position, along with the text in between.) We will rely on notes to find out which part of the AST the user pointed at.

- __Imports&nbsp;__ Dhall uses the `Embed a` constructor to embed imports into the AST. Since the ASTs we are dealing with had their imports resolved (`a` = `X`, the empty type), they do not contain `Embed` constructors.

- __Variables&nbsp;__ We wanted to display the types of variables, remember? It turns out to be more convenient to implement "type-on-hover" in such a way that it displays the type of the _smallest subtree_ of the AST containing the cursor position instead, but, if we wanted to, we could restrict ourselves to only consider variables (and not display anything if the user did not point at a variable).

- __Everything else&nbsp;__ When dealing with any of the remaining constructors we only need to consider two cases:
  1. The position lies in one of the subexpressions of the current node of the AST; in this case we recursively look at that subexpression. Through the use of [lenses]() (this is getting exciting, isn't it?) we are able to handle this case generically.
  2. The position does not lie in any of the subexpressions (or we reached a leaf node without any subexpressions); in this case we simply return the type of the current node to the user.

## The implementation (a few lines of Haskell)
The heart of this feature is implemented in [Dhall.LSP.Backend.Typing](https://github.com/dhall-lang/dhall-haskell/blob/8995efe69233d36fccea4f14df28a2b073e9390b/dhall-lsp-server/src/Dhall/LSP/Backend/Typing.hs#L32-L65) in the function `typeAt'`. (The link points to the exact version of the file on github.)
```haskell
typeAt' :: Position -> Context (Expr Src X) -> Expr Src X -> Either (TypeError Src X) (Expr Src X)
````
This function expects a _position_ (a line-column-tuple), a _typechecking context_ representing the binders we passed so far and the current _expression_ (a subexpression of a well-typed `Expr Src X`). The result of `typeAt'` is either a _type error_ (this should never happen, assuming the input was well-typed), or the type of the _smallest subexpression_ containing the given position.

Let us look at each of the clauses that make up the definition of `typeAt'` in turn. Note that I renamed some of the variable names from the actual code to be a bit more expressive (the following excerpts are equivalent to the referenced lines in [Dhall.LSP.Backend.Typing](https://github.com/dhall-lang/dhall-haskell/blob/8995efe69233d36fccea4f14df28a2b073e9390b/dhall-lsp-server/src/Dhall/LSP/Backend/Typing.hs#L32-L65)).

- The [first clause](https://github.com/dhall-lang/dhall-haskell/blob/8995efe69233d36fccea4f14df28a2b073e9390b/dhall-lsp-server/src/Dhall/LSP/Backend/Typing.hs#L34-L44) concerns the case where the cursor points inside the body of a let expression. For example:

  ![](/images/type-hover-example-let.png)

  In this case, before we can recurse, we need to keep track of the bound variable. Since Dhall allows for "type aliases" like

  ![](/images/type-hover-example-typesynonym.png)

  we need to handle the case where the bound term is a type separately. Don't look at the details too closely&mdash;they are quite specific to Dhall's semantics.
  ```haskell
  -- Dhall.LSP.Backend.Typing ll. 34-44
  typeAt' position context (Let (Binding variable _ value :| []) body@(Note src _))
    | position `inside` src = do
      typ <- typeWithA absurd ctx value

      kind <- fmap normalize (typeWithA absurd context typ)

      case kind of
          Const Type -> do  -- we don't have types depending on values
              let context' = fmap (shift 1 (V x 0)) (insert x (normalize typ) context)

              result <- typeAt' position context' body

              return (shift (-1) (V x 0) result)

          _ -> do  -- but we do have types depending on types
              let value' = shift 1 (V x 0) (normalize a)

              typeAt' position context (shift (-1) (V x 0) (subst (V x 0) value' body))
  ```

- The [second clause](https://github.com/dhall-lang/dhall-haskell/blob/8995efe69233d36fccea4f14df28a2b073e9390b/dhall-lsp-server/src/Dhall/LSP/Backend/Typing.hs#L46-L49) concerns the case where the cursors points inside the body of a lambda-abstraction, e.g.:

  ![](/images/type-hover-example-lambda.png)

  In this case we merely add the type of the bound variable to the typechecking context and recurse.
  ```haskell
  -- Dhall.LSP.Backend.Typing ll. 46-49
  typeAt' position context (Lam variable typ body@(Note src _))
    | position `inside` src = do
      let typ' = Dhall.Core.normalize typ

      context' = fmap (shift 1 (V x 0)) (insert variable typ' context)

      typeAt' position context' body
  ```

- The [third clause](https://github.com/dhall-lang/dhall-haskell/blob/8995efe69233d36fccea4f14df28a2b073e9390b/dhall-lsp-server/src/Dhall/LSP/Backend/Typing.hs#L51-L54) mirrors the second one for "forall" binders, e.g.:

  ![](/images/type-hover-example-forall.png)

- The [next clause](https://github.com/dhall-lang/dhall-haskell/blob/8995efe69233d36fccea4f14df28a2b073e9390b/dhall-lsp-server/src/Dhall/LSP/Backend/Typing.hs#L57) is particular to this implementation: it peels off an outer `Note` constructor. This is to make sure that the generic last clause is only ever applied to expressions that start with a "meaningful" constructor (i.e., not a `Note`).
  ```haskell
  typeAt' position context (Note _ expr) = typeAt' position context expr
  ```

- The [last clause](https://github.com/dhall-lang/dhall-haskell/blob/8995efe69233d36fccea4f14df28a2b073e9390b/dhall-lsp-server/src/Dhall/LSP/Backend/Typing.hs#L60-L65) is where the magic happens!
  ```haskell
  typeAt' position context expr = do
  let subExprs = toListOf subExpressions expr

  case [ (src, expr') | (Note src expr') <- subExprs, position `inside` src ] of
    [] -> do type <- typeWithA absurd context expr

             return (normalize typ)

    ((src, e):_) -> typeAt' position context (Note src expr')
  ```
  First of all, `subExprs = toListOf subExpressions expr` gives us the list of all immediate subexpressions of `expr`. This makes use of the lense combinator [toListOf](http://hackage.haskell.org/package/lens-4.17.1/docs/Control-Lens-Combinators.html#v:toListOf). Noite that `subExpressions` is a "traversal" defined in [Dhall.Core]() with the following type:
  ```haskell
  subExpressions :: Applicative f => (Expr s a -> f (Expr s a)) -> Expr s a -> f (Expr s a)
  ```
  If you find the type of `subExpressions` (and in fact of any of the combinators in `Control.Lens`) utterly unenlightening&mdash;you are not alone. What matters it that `toListOf subExpressions expr` does the right thing!

  To digest the rest, first a reminder: if `expr` was produced by [Dhall.Parse]() we know that every subexpression is wrapped in a `Note` constructor telling us which part of the source code it came from. This means that the case expression considers the following two cases:
  - Either the position is not contained in any of the subexpressions of `expr`. In this case we return the type of the entire expression `expr`.
  - Otherwise we recurse with the corresponding subexpression.

  This in particular causes the following behaviour:

  ![Image](/images/type-hover-lambda.png)

  Rather neat, eh?

## Conclusion
Basically, we ended up re-implementing the behaviour of Dhall's [typechecker](https://github.com/dhall-lang/dhall-haskell/blob/8995efe69233d36fccea4f14df28a2b073e9390b/dhall/src/Dhall/TypeCheck.hs#L100-L846) in the "interesting cases". The structurally simpler cases are all handled neatly using the `toListOf` lens combinator (whose behaviour is indistinguishable from magic).

Note that we did not talk about the "frontend" at all, i.e. the plumbing needed to handle LSP [Hover Requests](https://microsoft.github.io/language-server-protocol/specification#textDocument_hover) and produce the corresponding "hover result". Currently this amounts to all of [15 lines of "plumbing code"](https://github.com/dhall-lang/dhall-haskell/blob/3a120d277f62fe83f8d9b35f14e3c93b9a6076cf/dhall-lsp-server/src/Dhall/LSP/Handlers.hs#L160-L175). Feel free to have a look at it on your own, but don't expect any interesting new insights.
