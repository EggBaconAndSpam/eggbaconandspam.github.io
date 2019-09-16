# Seekable Parsers in Haskell

In which we implement a combinator `seek :: Int -> Parser ()` that causes parsing to continue at a given offset. To be upfront about it: we will run our (megaparsec) parser on top of a `Reader` monad, which allows us to access the full input at any point.

## Motivation
Haskell is great at parsing, to the point where writing parsers in Haskell is actually rather fun! It's not always quite as straightforward as one would hope, though -- there are a few patterns that don't translate *directly* to your favourite parser monad (e.g. `Parsec` from *megaparsec*, `Parser` from *attoparsec*, etc.). One example of such a pattern is the use of *offsets* and random access inside files; while we can easily `skip` forward in the input stream (by parsing and discarding a number of tokens), neither of the popular parser libraries allows us to rewind the input or seek to an arbitrary position!

Take the following example: a given file starts with an "Image Vector Table" (IVT), which contains the offset to a "Device Configuration Data" (DCD) structure; we want to extract the latter. In other words, our parser needs to parse the IVT, then seek to `offset_dcd` and parse the DCD.

| Offset | Size (bytes) | Field        | | | Offset          | Size (bytes) | Field       |
|--------|--------------|--------------|-|-|-----------------|--------------|-------------|
| 0      | 4            | header_ivt   | | | offset_dcd      | 4            | header_dcd  |
| 4      | 4            | offset_entry | | | offset_ivt + 4  | 4            | dcd entry 1 |
| 8      | 4            | reserved_1   | | | offset_ivt + 8  | 4            | dcd entry 2 |
| 12     | 4            | offset_dcd   | | | offset_ivt + 12 | 4            | dcd entry 3 |
| (...)  |              |              | | | (...)           |              |             |

The clever reader might notice that the DCD necessarily starts after the IVT, and therefore we could just use `skip` to seek to the offset without having to implement `seek` proper; but that wouldn't be a general solution, and would also feel a bit of a hack.

## Solution

For simplicity's sake, let us assume we are using `megaparsec`, and that the input is a `ByteString`. As mentioned before, the idea is to wrap the parser *monad transformer* `ParsecT` around a reader monad, the latter giving us access to the whole input at any point from within the parser. That is, we define our custom parser monad `SeekableParser` as follows:

```haskell
import Control.Monad.Reader (Reader, lift, ask)
import Data.ByteString (ByteString)

import Data.Void (Void)
import Text.Megaparsec (ParsecT)

import qualified Data.ByteString as BS
import qualified Text.Megaparsec as MP
import qualified Text.Megaparsec.Pos as MP

type SeekableParser = ParsecT Void ByteString (Reader ByteString)
```

To avoid any confusion, the first parameter of `ParsecT` is a custom error type that we won't make use of here. Now, assuming that the underlying `Reader` monad does indeed contain the full input we can define a first `seek` combinator. We merely need to query the reader monad and change the parser state to the correct suffix of the input.

```haskell
-- A first attempt
seek' :: Int -> SeekableParser ()
seek' offset = do
  input <- lift ask
  let input' = BS.drop offset input
  MP.setInput input'
```

The only problem with `seek'` as defined above is that it causes some auxiliary state, namely the current offset and position in the input, to become inconsistent. Fixing this requires some meddling with the details of megaparsec's parser state, but there is nothing interesting going on.

```haskell
seek :: Int -> SeekableParser ()
seek offset = do
  input <- lift ask
  let input' = BS.drop offset input

  state <- MP.getParserState
  let statePos = MP.statePosState state
  let sourceName = MP.sourceName $ MP.pstateSourcePos statePos

  -- Update the parser state so that we continue parsing `input'`. We also need
  -- to update the position information. The actual (line,col) position will be
  -- calculated the next time getSourcePos is used.
  let statePos' = statePos { MP.pstateInput = input,
                             MP.pstateOffset = 0,
                             MP.pstateSourcePos = MP.initialPos sourceName,
                             MP.pstateLinePrefix = "" }
  MP.setParserState $ state { MP.stateInput = input',
                        MP.statePosState = statePos' }
```
A bit more technically involved, but behaves exactly as you would expect it to.

Finally, how do we use `SeekableParser`s? Easy: we just 'run' both layers of the monad!
```haskell
parseSeekable :: SeekableParser a  -- ^ The parser to run
              -> String            -- ^ Input filename, used in error messages
              -> ByteString        -- ^ The input
              -> Either (ParseError (Token ByteString) Void) a
parseSeekable p s b = (flip runMaybe) b $ runParserT p s b
```

Now we can simply write our parsers in terms of the `SeekableParser` monad, and use `seek` whenever we feel like! For an example of this in action, check out my [*imx-extract*](https://github.com/EggBaconAndSpam/imx-extract) tool.

## Discussion

- Carrying around the input in a Reader monad is not ideal for huge input files; in that case, a better approach would be to use `IO` as the underlying monad and read the content lazily.
- Our solution relies on the parser library exposing a monad transformer interface -- this rules out *attoparsec* amongst others. Any ideas?
- Other solutions?

If you have anything to add or would like to leave a comment, you are invited to open an issue on the github repo for this [blog](https://github.com/EggBaconAndSpam/eggbaconandspam.github.io) or write me an email at [frederikramcke@mailbox.org](mailto:frederikramcke@mailbox.org).
