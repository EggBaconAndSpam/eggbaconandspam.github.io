# Final Report: Dhall Language Server
This is my final report as part of GSoC 2019, working on the [Dhall language server](https://summerofcode.withgoogle.com/projects/#5991057626497024).

## Introduction
Error highlighting and code completion are among the features programmers nowadays expect from IDEs. As part of the Google Summer of Code 2019 I had the chance to work on the ‘Dhall language server’, which implements those features (and more) in an editor agnostic way for the [Dhall language](https://dhall-lang.org/).

The project was successful – you can now use VSCode/[ium](https://vscodium.com/) to enjoy the full Dhall experience with all its bells and whistles! Note that there is no reason for us not supporting other editors, other than no one has written the necessary glue code yet (feel free to contribute!).

## Screencast (Code Completion)

![Code completion](https://raw.githubusercontent.com/EggBaconAndSpam/eggbaconandspam.github.io/master/images/completion.png)

## Main Contributions
Thanks to [PanAeon](https://github.com/PanAeon)'s efforts I was able to start from a working prototype, and was able to contribute user-facing features right away after a short getting-to-know-the-code period. I managed to implement all features I set out to. Note that we dropped the originally proposed alpha/beta-normalisation features, as their usefulness wasn't immediately obvious; I ended up implementing code completion instead.

These were my main contributions to [*dhall-lsp-server*](https://github.com/dhall-lang/dhall-haskell/tree/master/dhall-lsp-server) (and [*vscode-dhall-lsp-server*](https://github.com/PanAeon/vscode-dhall-lsp-server)):

- Better type errors with optional detailed explanations ([gif](../images/explain-on-hover.png); code at [#974](https://github.com/dhall-lang/dhall-haskell/pull/974) and [#982](https://github.com/dhall-lang/dhall-haskell/pull/982); client code at [#4](https://github.com/PanAeon/vscode-dhall-lsp-server/pull/4))
- Linting ([gif](../images/lint-and-format.png); [#1003](https://github.com/dhall-lang/dhall-haskell/pull/1003); [#7](https://github.com/PanAeon/vscode-dhall-lsp-server/pull/7))
- Displaying the inferred type when hovering over any part of the code ([gif](../images/type-hover.png); [#1008](https://github.com/dhall-lang/dhall-haskell/pull/1008))
- Annotating let bindings ([gif](../images/annotate-let.png); [#1014](https://github.com/dhall-lang/dhall-haskell/pull/1014), [#8](https://github.com/PanAeon/vscode-dhall-lsp-server/pull/8))
- Caching (performance enhancement) ([#1040](https://github.com/dhall-lang/dhall-haskell/pull/1040))
- Clickable import statements ([gif](../images/follow-imports.png); [#1121](https://github.com/dhall-lang/dhall-haskell/pull/1121))
- Freezing imports ([gif](../images/freezing-imports.png); [#1123](https://github.com/dhall-lang/dhall-haskell/pull/1123), [#9](https://github.com/PanAeon/vscode-dhall-lsp-server/pull/9))
- Code completion ([gif](../images/completion.png); [#1190](https://github.com/dhall-lang/dhall-haskell/pull/1190))

Due to the rapid pace of the project I was able to take some time to work outside the originally intended scope, and contribute directly to [*dhall-haskell*](https://github.com/dhall-lang/dhall-haskell/tree/master/dhall) proper (i.e. the `dhall` executable).
- I implemented ‘semi-semantic’ caching of imports ([#1113](https://github.com/dhall-lang/dhall-haskell/pull/1113), [#1128](https://github.com/dhall-lang/dhall-haskell/pull/1128) and [#1154](https://github.com/dhall-lang/dhall-haskell/pull/1154)).
- I implemented a mock http client, to allow all tests to run successfully when compiled without proper http support ([#1159](https://github.com/dhall-lang/dhall-haskell/pull/1159)).

I even ended up making a very minor contribution ([#618](https://github.com/dhall-lang/dhall-lang/pull/618)) to the [Dhall language standard](https://github.com/dhall-lang/dhall-lang).

## Personal Retrospective
First of all, I really enjoyed working on the project! I certainly achieved my main goal, which was to learn as much as possible!
- This was my first experience writing ‘real-life’ Haskell code; before I had only written code for my own use. As one example, I was finally forced to embrace lenses, having shied away from them for so long.
- This was also my first time contributing to an active open source project! I was introduced to the ‘pull request’ based work flow (in particular, the Dhall project uses ‘Trunk-based development’), and had other contributors review my code, and also tried to give feedback myself.

I was incredibly lucky to have Gabriel as a mentor. He helped guide me on the technical side of things, but he also took the time to give me a look ‘behind the scenes’, giving me advice on software and open source development in general.

Dhall won't be the last project for me to contribute to!

## Acknowledgements
Thank you to [Gabriel](http://www.haskellforall.com/) for mentoring me, and also to [Luke](https://lukelau.me/) for volunteering as co-mentor! Thank you also to [Haskell.org](https://www.haskell.org/) for hosting the GSoC project, and to Google for providing the funding.
