# The Lazy Baker &ndash; Calculating Bread Recipes with Z3

Example: _(A simple sourdough rye bread): 85% hydration, 2% salt, 25% prefermentation. (For the uninitiated: this recipe calls for one part flour, 0.85 parts water, 0.02 parts salt, and for 25% of the flour to be prefermented.)_

The ingredients in bread recipes are commonly given in ['Baker's Percent'](https://en.wikipedia.org/wiki/Baker_percentage), that is in weight relative to the total weight of flour. To bake a loaf of say 1.2kg following such a 'recipe sketch' we will have to do some calculations &ndash; or rather, instruct Z3 to do them for us.

## Spoiler: the full example recipe for a loaf of 1.2kg
_To keep you engaged, here the full sourdough rye recipe that we will calculate using Z3 by the end of this blog post. (The variable names in parentheses will appear again later.)_

Prepare the _preferment_ the day before by mixing
- some starter (ca. a tablespoon) &ndash; this is a bit of preferment saved from your previous loaf, alternatively you'll have to start one from scratch;
- 241g (H_p) of water,
- 160g (F_p) of flour.

Save a tablespoon as the starter for the next loaf. To be particularly precise you can also use a bit more flour and water (using the same 'hydration' ratio of 1.5 parts water to flour (h_p)) and save the excess to the target weight of 401g.

Cover and let ferment at room temparature for 8&ndash;16 hours.

Prepare the _final dough_ by mixing
- the preferment,
- 305g water (H_a),
- 12.8g salt (S),
- 481g flour (F_a).

There is no need to knead the dough. I suggest you start by mixing together preferment, water and salt, and only then add the flour. (If you mix the salt in after the flour you might end up with salty hotspots if you don't mix thoroughly enough; this way you can be a bit more lax.)

Put in a floured banneton (alternatively a bowl covered with a tea towel, also floured) and let ferment at room temperature for 5&ndash;6 hours.

Bake at 230 Celsius: 30 mins covered (e.g. with aluminium foil) and 25 mins uncovered. The precise baking times depend on the oven and might need experimenting with.

Let cool for at least half a day.


## The constraints of bread baking

```
  W = F + H + S
  H = F * h
  S = F * s
```

Where `h` and `s` stand for hydration (water percentage) and saltiness (salt percentage); `W`, `F`, `H`, `S` for the total dough weight and the amount of flour, water and salt in grammes.

```
  W_p = F_p + H_p
  F_p = F * p
  H_p = F_p * h_p
```

Where `p` stands for the prefermentation percentage, `h_p` for the hydration of the preferment and W_p, F_p, H_p for the weight, flour, and water of the preferment.

```
  F_a = F - F_p
  H_a = H - H_p
```

Where `F_a` and `H_a` are the additional flour and water to be added to the preferment to arrive at the final dough.


## Having Z3 do the calculations for us
Save the following as `recipe.smt2`:
```Lisp
; recipe constants
(define-const W Real 1200.0)
(define-const h Real 0.85)
(define-const s Real 0.02)
(define-const p Real 0.25)
(define-const h_p Real 1.5)

; total ingredients in final dough
(declare-const F Real)
(declare-const H Real)
(declare-const S Real)

; preferment
(declare-const W_p Real)
(declare-const F_p Real)
(declare-const H_p Real)

; dough additions
(declare-const F_a Real)
(declare-const H_a Real)

; constraints
(assert (= W (+ F (+ H S))))
(assert (= H (* F h)))
(assert (= S (* F s)))

(assert (= W_p (+ F_p H_p)))
(assert (= F_p (* F p)))
(assert (= H_p (* F_p h_p)))

(assert (= F_a (- F F_p)))
(assert (= H_a (- H H_p)))

; magic incantation to get Z3 to spit out the answer
(check-sat)
(set-option :pp.decimal true)
(get-model)
```
Then `z3 -in < recipe.smt2` tells us what we need to know:
```
sat
(
  (define-fun h_p () Real
    1.5)
  (define-fun h () Real
    0.85)
  (define-fun H_a () Real
    304.8128342245?)
  (define-fun s () Real
    0.02)
  (define-fun p () Real
    0.25)
  (define-fun S () Real
    12.8342245989?)
  (define-fun H_p () Real
    240.6417112299?)
  (define-fun W () Real
    1200.0)
  (define-fun F () Real
    641.7112299465?)
  (define-fun W_p () Real
    401.0695187165?)
  (define-fun H () Real
    545.4545454545?)
  (define-fun F_p () Real
    160.4278074866?)
  (define-fun F_a () Real
    481.2834224598?)
)
```

## Postscript

TODO: photos of the final bread as well as the flour used.

_Flour:_ This recipe, and in fact all bread recipes, are rather specific to the type and brand of flour used &ndash; you might have to tweak the hydration (h) constant to get the best result with a different flour.

_Fermentation times:_ The precise fermentation times are a matter of experimentation (and they vary depending on what 'room temperature' ends up being). If you don't ferment the main dough for long enough it will taste sweet and bland; if you overferment the texture of the bread might end up a bit sticky.

_The Z3 Theorem Prover_ is free and open source software and readily available, e.g. via `dnf install z3` on fedora or `nix-shell -p z3` using nix. Using Z3 directly is perhaps not the most user friendly approach, what with its lisp syntax and somewhat unstructured output, but I prefer it due to its portability and simplicity.

## Postpostscript

The reason I ended up baking my own bread in the first place is that &ndash; to my German sensibilities &ndash; Swedish bread is terrible. As is Norwegian bread for that matter (though in a different way).

On the way to developing my own baking skills I particularly enjoyed [this instruction video](https://www.youtube.com/watch?v=XiaSmVZxkYs) (caution: French!).

Enjoy your bread/bröd/brød/brot/pain.
