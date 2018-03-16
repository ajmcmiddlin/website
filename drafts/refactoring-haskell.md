--------------------------------------------------------------------------------
title: "Refactoring Haskell: A Case Study"
published: 2018-02-28
tags: programming, haskell
--------------------------------------------------------------------------------

Many people claim that [refactoring Haskell is a
joy](https://twitter.com/search?q=haskell%20refactoring). I've certainly found
this to be the case, but what does that mean in practice? I thought it might be
useful to demonstrate by refactoring some of my own code from when I was less
familiar with Haskell.

The code we're looking at today is an implementation of [Tarjan's Strongly
Connected Components
algorithm](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm)
used to determine whether a given 2SAT problem is satisfiable or not, and was
written to complete [an online
course](https://online.stanford.edu/course/algorithms-design-and-analysis-part-1)
that is now offered in a different form. I've [written about Tarjan's algorithm
previously](http://vaibhavsagar.com/blog/2017/06/10/dag-toolkit/) and it can be
used to determine the satisfiability of a 2SAT problem by checking if any SCC
contains both a variable and its negation. If it does, we have a contradiction
and the problem is unsatisfiable, otherwise the problem is satisfiable.

The initial version of the code is as follows:

<details>
<summary>Initial 2SAT.hs</summary>

```haskell
{-# LANGUAGE LambdaCase #-}

import qualified Data.Graph      as G
import qualified Data.Map.Strict as M
import qualified Data.Set        as S
import qualified Data.Array      as A
import qualified Prelude         as P

import Prelude hiding (lookup)

import Control.Monad.ST
import Data.STRef
import Control.Monad (forM_, when)
import Data.Maybe (isJust, isNothing, fromJust)

tarjan :: Int -> G.Graph -> Maybe [S.Set Int]
tarjan n graph = runST $ do
    index    <- newSTRef 0
    stack    <- newSTRef []
    stackSet <- newSTRef S.empty
    indices  <- newSTRef M.empty
    lowlinks <- newSTRef M.empty
    output   <- newSTRef (Just [])

    forM_ (G.vertices graph) $ \v -> do
        vIndex <- M.lookup v <$> readSTRef indices
        when (isNothing vIndex) $
            strongConnect n v graph index stack stackSet indices lowlinks output

    readSTRef output

strongConnect
    :: Int
    -> Int
    -> G.Graph
    -> STRef s Int
    -> STRef s [Int]
    -> STRef s (S.Set Int)
    -> STRef s (M.Map Int Int)
    -> STRef s (M.Map Int Int)
    -> STRef s (Maybe [S.Set Int])
    -> ST    s ()
strongConnect n v graph index stack stackSet indices lowlinks output = do
    i <- readSTRef index
    insert v i indices
    insert v i lowlinks
    modifySTRef' index (+1)
    push stack stackSet v

    forM_ (graph A.! v) $ \w -> lookup w indices >>= \case
        Nothing     -> do
            strongConnect n w graph index stack stackSet indices lowlinks output
            vLowLink <- fromJust <$> lookup v lowlinks
            wLowLink <- fromJust <$> lookup w lowlinks
            insert v (min vLowLink wLowLink) lowlinks
        Just wIndex -> do
            wOnStack <- S.member w <$> readSTRef stackSet
            when wOnStack $ do
                vLowLink <- fromJust <$> lookup v lowlinks
                insert v (min vLowLink wIndex) lowlinks

    vLowLink <- fromJust <$> lookup v lowlinks
    vIndex   <- fromJust <$> lookup v indices
    when (vLowLink == vIndex) $ do
        scc <- addSCC n v S.empty stack stackSet
        modifySTRef' output $ \sccs -> (:) <$> scc <*> sccs
    where
        lookup value hashMap     = M.lookup value <$> readSTRef hashMap
        insert key value hashMap = modifySTRef' hashMap (M.insert key value)

addSCC :: Int -> Int -> S.Set Int -> STRef s [Int] -> STRef s (S.Set Int) -> ST s (Maybe (S.Set Int))
addSCC n v scc stack stackSet = pop stack stackSet >>= \w -> if ((other n w) `S.member` scc) then return Nothing else
    let scc' = S.insert w scc
    in if w == v then return (Just scc') else addSCC n v scc' stack stackSet

push :: STRef s [Int] -> STRef s (S.Set Int) -> Int -> ST s ()
push stack stackSet e = do
    modifySTRef' stack    (e:)
    modifySTRef' stackSet (S.insert e)

pop :: STRef s [Int] -> STRef s (S.Set Int) -> ST s Int
pop stack stackSet = do
    e <- head <$> readSTRef stack
    modifySTRef' stack tail
    modifySTRef' stackSet (S.delete e)
    return e

denormalise     = subtract
normalise       = (+)
other n v       = 2*n - v
clauses n [u,v] = [(other n u, v), (other n v, u)]

checkSat :: String -> IO Bool
checkSat name = do
    p <- map (map P.read . words) . lines <$> readFile name
    let pNo    = head $ head p
        pn     = map (map (normalise pNo)) $ tail p
        pGraph = G.buildG (0,2*pNo) $ concatMap (clauses pNo) pn
    return $ (Nothing /=) $ tarjan pNo pGraph
```

</details>

I've included 2SAT-specific functionality for completeness, but I'll omit this
in future listings as I'll only be changing the `tarjan` function and the
functions it depends on (`strongConnect`, `addSCC`, `push`, and `pop`).

The first change is using more suitable data structures. Tarjan's algorithm is
only linear in the size of the graph when operations such as checking if `w` is
on the stack and looking up indices happen in constant time ($O(1)$). I'm currently
using `Data.Map` and `Data.Set` which are both implemented with trees and are
$O(\log{}n)$ in these operations.