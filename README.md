# dep-t

`DepT` is Readerlike monad transformer for dependency injection.

The difference with
[`ReaderT`](http://hackage.haskell.org/package/mtl-2.2.2/docs/Control-Monad-Reader.html)
is that `DepT` takes an enviroment whose type is parameterized by a monad:
itself. 

## Motivation

To achieve dependency injection in Haskell, one common solution is having a
record of functions, and passing it to the program logic using a `ReaderT`.

Sometimes the functions in the record work directly in the `IO` monad.

But sometimes we want to abstract the concrete monad. This can be achieved by
parameterizing the record:

    type Env :: (Type -> Type) -> Type
    data Env m = Env
      { logger :: String -> m (),
        logic :: Int -> m Int
      }

Lets write some implementations for the functions in the record:

    _logger :: MonadIO m => String -> m ()
    _logger msg = liftIO (putStrLn msg)

    -- To avoid depending on the concrete record, we pass a getter to the logger
    _logic :: MonadReader e m => (e -> String -> m ()) -> Int -> m Int
    _logic getLogger x = do
      logger <- reader getLogger
      logger "I'm going to multiply a number by itself!"
      return $ x * x

Now let's tie everything together using `ReaderT`:

    env' =
      Env
        { logger = _logger,
          logic = _logic logger
        }

    result' :: IO Int
    result' = runReaderT (logic env' 7) env'

Oops, this doesn't seem to work:

    * Couldn't match type `r0' with `Env (ReaderT r0 IO)'

Also, what type should `env'` have?

However, using `DepT`, which is just a newtype wrapper over `ReaderT` to pacify
the compiler, it works:

    env :: Env (DepT Env IO)
    env =
      Env
        { logger = _logger,
          logic = _logic logger
        }

    -- We select "logic" as the entrypoint and run it.
    result :: IO Int
    result = runDepT (logic env 7) env

Notice that the use of `DepT` was limited to the moment of assembling and
running the concrete record. The record definition itself is independent of
`DepT` (and of `ReaderT` for that matter). 

As for the implementation functions, they use `MonadReader` to get hold of the
environment, but they know nothing of `DepT`.

## Links

This library was extracted from my answer to
[this Stack Overflow question](https://stackoverflow.com/a/61782258/1364288).

