---
title: "A minimal Haskell effect system"
date: 2025-04-01T22:04:51-03:00
summary: Structure large applications with no dependencies
draft: true
---

## Introduction

For quite some time I've been interested approaches for structuring large Haskell programs, in particular in those that allow developers to build programs where interfaces and implementations are fully decoupled: programs should list the "operations" they're interested in performing (the "what") while the implementations of these operations are defined somewhere else (the "how").

In recent years, effect systems have gained a lot of popularity in functional programming languages as a way to structure programs in this fashion, examples being [freer-simple](https://hackage.haskell.org/package/freer-simple), [polysemy](https://www.youtube.com/watch?v=kIwd1D9m1gE), and [effectful](https://www.youtube.com/watch?v=L21FkWWqW98) to name a few. Nevertheless, these are usually quite intricate while also compromising in boilerplate, performance and vendor lock-in: it's extremely hard for junior developers to understand how or why these effect systems work, they require some level of code generation that usually is handled via `TemplateHaskell`, and can negatively impact performance in unexpected ways.

## Goal and non-goals.

Our goal is to come up with a minimal solution that covers as many use cases as possible without trying to be a general solution. To produce this, let's work backwards and start by writing the program we would like to have, filling in the blanks as we go. For example, let's assume that we want to write the following `echo` program:

```haskell
echo = do
  writeMsg "echo; write `exit` to quit the program"
  fix $ \continue -> do
    msg <- getMsg
    case msg of
      "exit" -> writeMsg "goodbye"
      _ -> do
        writeMsg msg
        continue
```

that is, `echo` will repeat any message that it receives, stopping execution once it receives a "exit" message. Notice how we're not tying the program to any specific IO operation: how we `getMsg` and `writeMsg` is an implementation detail and we want to delay such choice as long as possible: this is fundamentally [dynamic dispatch](https://www.youtube.com/live/0jI-AlWEwYI?feature=shared&t=2928).

With this in mind, our `echo` program cannot actually be executed since we need to instantiate it to some implementations of `getMsg` and `writeMsg`. Before moving forward, let's give name to the concepts we've been working with so far:

- An effect is a collection of operations like `getMsg` and `writeMsg`. A program might be described using multiple effects.
- A handler is a mapping from operations in an effect to executable code. A program might require multiple handlers to be fully materialized into executable code.

In this example, `getMsg` and `writeMsg` form an effect that we might want to call `Teletype`, and stdin/stdout a handler for such effect (read from `stdin`, write to `stdout`). In Haskell, we can encode such effect and handler using records and values as follows:

```haskell
data Teletype = Teletype
  { getMsg :: IO ()
  , writeMsg :: String -> IO ()
  }

stdTeletype :: Teletype
stdTeletype = Teletype
  { getMsg = getLine
  , writeMsg = putStrLn
  }
```

One question that we'll answer later is which type should operations like `getMsg` and `writeMsg` should have. For now we'll use `IO` since it does not restrict possible handler implementations in any way.

## Initial implementation: Handler pattern

If we wanted to make use our `Teletype` effect in our `echo` program, as a first approach we could pass a handler for such effect as an argument and use it as follows:

```haskell
echo :: Teletype -> IO ()
echo tty = do
  tty.writeMsg "echo; write `exit` to quit the program"
  fix $ \continue -> do
    msg <- tty.getMsg
    case msg of
      "exit" -> tty.writeMsg "goodbye"
      _ -> do
        tty.writeMsg msg
        continue
```

This approach, named the [handler pattern](https://jaspervdj.be/posts/2018-03-08-handle-pattern.html), works well enough and can get you pretty far in practice, but has some downsides that make it a bit unergonomic in my opinion.

### Parameter pollution

The first downside of such approach is the pollution of parameters, where we mix effects with "standard" parameters. For example, in our `echo` program, if we wanted to parametrize the "exit" message the signature would look like:

```haskell
echo :: Teletype -> String -> IO ()
echo tty exitMsg = ...
```

If we continue adding effects and "standard" arguments, then signatures end up generally looking like:

```haskell
program :: E1 -> E2 -> ... -> Arg1 -> Arg2 -> ... IO ()
```

It's hard at quick glance to distinguish these type of parameters and personally I consider them quite different when it comes to expressing the intent of a program.

### Parameter fatigue

One of the fundamental aspects of writing code is the ability to easily compose programs: we want to use the output of certain programs as inputs of others, creating new programs that themselves can be composed. If we had two programs that make use of the `Teletype` effect, composing them looks like:


```haskell
programA :: Teletype -> IO ()
programB :: Teletype -> IO ()

program :: Teletype -> IO ()
program tty = do
  programA tty
  programB tty
```

Imagine now how the code would look if we wanted to compose multiple programs, each using several effects.

These two issues hint us that we want some form of "implicitness" when dealing with handlers: we want to implicitly pass the appropriate handlers to programs that make use of effects, without stating these requirements as parameters. In Haskell we can leverage typeclasses to achieve such "implicitness"

## Introducing `Eff`

Instead of making our programs work in `IO` as we have done so far we'll introduce a new type which is defined as follows:

```haskell
newtype Eff es a = MkEff (es -> IO a)
```

`Eff` is equivalent to a function `env -> IO a`, so a "supercharged" `IO` program that takes an additional parameter `es`. This parameter will play the role of an "environment" where we'll store the handlers for a set of effects.

Some readers might quickly point out that `Eff es a = ReaderT es (IO a)`, and they would be correct. We'll use an approach named the [ReaderT design pattern](https://tech.fpcomplete.com/blog/2017/06/readert-design-pattern/), but we'll make some small improvements along the road to improve ergonomics and reduce boilerplate.

`Eff` is a Monad, thus we define:

```haskell
instance Functor (Eff es) where
  fmap f (MkEff ea) = MkEff $ \env -> do
    a <- ea env
    let b = f a
    return b

instance Applicative (Eff es) where
  pure x = MkEff $ \_ -> return x
  MkEff eab <*> MkEff ea = MkEff $ \env -> do
    ab <- eab env
    a <- ea env
    let b = ab a
    return b

instance Monad (Eff es) where
  return = pure
  MkEff ea >>= faeb = MkEff $ \env -> do
    a <- ea env
    let (MkEff eb) = faeb a
    eb env
```
