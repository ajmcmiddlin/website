--------------------------------------------------------------------------------
title: Revisiting 'Monadic Parsing in Haskell'
published: 2018-02-02
tags: haskell, programming, monads
--------------------------------------------------------------------------------

Monadic parsing in Haskell is what sold me on all three. Before Haskell my experiences with parsing had involved buggy regexes for lexers and wrangling tools like `bison` and `flex`, and although I'd heard that Haskell was good for parsing I couldn't see how this could be the case when I couldn't find any robust regex libraries! An aside in some documentation pointed me to Attoparsec and when I saw the [example RFC2616 parser](https://github.com/bos/attoparsec/blob/master/examples/RFC2616.hs) it seemed like a magic trick. How could it be so small? After a few weeks of trying it myself I was convinced. This was the first application of monads I encountered that actually made my life simpler, and I started to realise that there was more to monads than smugness and being inaccessible to newcomers.

['Monadic Parsing in Haskell'](http://www.cs.nott.ac.uk/~pszgmh/pearl.pdf) laid the groundwork for libraries like Attoparsec, and after using these libraries for a while I found it quite approachable. Although it was published in 1998 (almost 20 years ago!) it has aged gracefully and the code samples will run with almost no changes. However, the state of the art has advanced since then and I think the use of modern Haskell can make this material simpler to follow and implement. I'm assuming you have the original paper nearby to compare this with.

The first change I want to make is the type definition. The paper uses the type

```haskell
newtype Parser a = Parser (String -> [(a,String)])
```

and although this is a famous enough definition that it has [its own rhyme](http://www.willamette.edu/~fruehr/haskell/seuss.html), I think the flexibility of lists is unnecessary. The authors don't use it, and instead define a 'deterministic choice' operator that gives at most one result and use that everywhere instead. There is already a perfectly good datatype in Haskell for this, `Maybe`, so I'll use that:

```haskell
newtype Parser a = Parser (String -> Maybe (a, String))
```

Renaming `String` to `s` and `Maybe` to `m` reveals a more interesting pattern:

```haskell
newtype Parser s m a = Parser (s -> m (a, s))
```

This is [`StateT`](https://hackage.haskell.org/package/transformers/docs/Control-Monad-Trans-State-Strict.html#t:StateT)! Recognising this pattern makes instance definitions much easier, so much easier in fact that GHC can do it for us automatically with `-XGeneralizedNewtypeDeriving`! For completeness I will resist the temptation to do this, but you can try it yourself with

```haskell
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
newtype Parser a = Parser (StateT String Maybe a) deriving (Functor, Applicative, Alternative, Monad)
```

The second change is also for completeness: the authors jump straight into the `Monad` instance without defining `Functor` and `Applicative` first. To be fair, the `Applicative` abstraction hadn't been discovered yet, and this is also the reason why the authors define `mzero` and `mplus` (which they call `(++)`) instead of `Alternative`. Because of our `Maybe` change, I'll be able to get away with defining just `Alternative` and won't need to bother with their `(++)`

Finally, I'll try to avoid do-notation where possible in favour of a more Applicative style using e.g. `<*>` because a lot of the benefits of these parsers don't require it.

Let's begin!


```haskell
{-# LANGUAGE InstanceSigs #-}

import Control.Applicative (Alternative(..))
import Control.Monad.Trans.State.Strict
import Control.Monad.Trans (lift)
import Control.Monad (guard)
import Data.Char (isSpace, isDigit, ord)
```

For convenience I've defined an `unParser` that unwraps a `Parser a` to its underlying `StateT String Maybe a`.


```haskell
newtype Parser a = Parser { unParser :: StateT String Maybe a }
runParser = runStateT . unParser
```

`fmap` is as simple as unwrapping the `Parser` and using the underlying `StateT`'s `fmap`.


```haskell
instance Functor Parser where
    fmap :: (a -> b) -> Parser a -> Parser b
    fmap f p = Parser $ f <$> unParser p
```

More unwrapping for `Applicative` and `Alternative`. The `Alternative` definition matches `(+++)` from the paper which means we don't need to define it separately and can just use `(<|>)` everywhere instead.


```haskell
instance Applicative Parser where
    pure :: a -> Parser a
    pure a  = Parser $ pure a
    (<*>) :: Parser (a -> b) -> Parser a -> Parser b
    f <*> a = Parser $ (unParser f) <*> (unParser a)

instance Alternative Parser where
    empty :: Parser a
    empty   = Parser $ lift empty
    (<|>) :: Parser a -> Parser a -> Parser a
    a <|> b = Parser $ (unParser a) <|> (unParser b)
```

The `Monad` definition is slightly more interesting, because we have to manually construct the `StateT` value, but this also boils down to unwrapping and rewrapping.


```haskell
instance Monad Parser where
    (>>=) :: Parser a -> (a -> Parser b) -> Parser b
    a >>= f = Parser $ StateT $ \s -> do
        (a', s') <- runStateT (unParser a) s
        runStateT (unParser (f a')) s'
```

Notice that `anyChar` is the only function below that manually constructs a `Parser`, and `satisfy` is the only one that requires the `Monad` interface.


```haskell
anyChar :: Parser Char
anyChar = Parser $ StateT $ \s -> case s of
    []     -> Nothing
    (c:cs) -> Just (c, cs)

satisfy :: (Char -> Bool) -> Parser Char
satisfy pred = do
    c <- anyChar
    guard $ pred c
    pure c

char :: Char -> Parser Char
char = satisfy . (==)

string :: String -> Parser String
string []     = pure []
string (c:cs) = (:) <$> char c <*> string cs
```

I haven't defined `many` and `many1`, because they are provided for free by `Applicative` as `many` and `some` after I defined `(<|>)`.


```haskell
sepBy :: Parser a -> Parser b -> Parser [a]
sepBy p sep = (p `sepBy1` sep) <|> pure []

sepBy1 :: Parser a -> Parser b -> Parser [a]
sepBy1 p sep = (:) <$> p <*> many (sep *> p)
```

These are almost identical to the definitions in the paper. I've included `chainr` for completeness.


```haskell
chainl :: Parser a -> Parser (a -> a -> a) -> a -> Parser a
chainl p op a = (p `chainl1` op) <|> pure a

chainl1 :: Parser a -> Parser (a -> a -> a) -> Parser a
chainl1 p op = p >>= rest
    where 
        rest a = (do
            f <- op
            b <- p
            rest (f a b)) <|> pure a

chainr :: Parser a -> Parser (a -> a -> a) -> a -> Parser a
chainr p op a = (p `chainr1` op) <|> pure a

chainr1 :: Parser a -> Parser (a -> a -> a) -> Parser a
chainr1 p op = scan
    where
        scan   = p >>= rest
        rest a = (do
            f <- op
            b <- scan
            rest (f a b)) <|> pure a
```

The only difference here is the replacement of `(>>)` with `(*>)`.


```haskell
space :: Parser String
space = many (satisfy isSpace)

token :: Parser a -> Parser a
token p = p <* space

symbol :: String -> Parser String
symbol = token . string

apply :: Parser a -> String -> Maybe (a, String)
apply p = runParser (space *> p)
```

The calculator example is almost unchanged.


```haskell
expr, term, factor, digit :: Parser Int
expr   = term   `chainl1` addop
term   = factor `chainl1` mulop
factor = digit <|> (symbol "(" *> expr <* symbol ")")
digit  = subtract (ord '0') . ord <$> token (satisfy isDigit)

addop, mulop :: Parser (Int -> Int -> Int)
addop = (symbol "+" *> pure (+)) <|> (symbol "-" *> pure (-))
mulop = (symbol "*" *> pure (*)) <|> (symbol "/" *> pure (div))
```

Finally, the payoff!


```haskell
runParser expr "(1 + 2 * 4) / 3 + 5"
```


    Just (8,"")


What have we gained in 20 years? With only minor changes, the code is more composable and uses finer-grained abstractions. For example, if we change our minds about replacing `[]` with `Maybe`, we can switch it back and would only have to update `anyChar` and `apply`! If we want better error messages, we could use a type such as `Either String` to keep track of locations and error messages.

Another big difference is the `Applicative` family of functions, which we can leverage whenever we don't have to branch on a previously parsed value (which turns out to be surprisingly often). I'm a huge fan of the `x <$> y <*> z` idiom and I think it's useful to be able to parse this way.

Otherwise, the code is largely the same and I think it's pretty incredible that so little has changed in 20 years!
