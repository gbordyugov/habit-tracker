# Project: A Habit Tracker CLI in Haskell

A small command-line habit tracker designed as a learning project for practicing
application architecture in Haskell using type classes for dependency management
and clear separation of business logic from IO.

## What it does

- `habit add "Read 30 minutes"` — add a new habit
- `habit check "Read 30 minutes"` — mark it done for today
- `habit status` — show streaks and today's progress
- `habit report --week` — weekly summary

## Why this project fits the goals

It naturally has **three IO dependencies** that you'll want to abstract:

1. **Persistence** — reading/writing habits to a JSON or SQLite file
2. **Clock** — getting "today's date" (critical to abstract, since it makes business logic testable)
3. **Logger/Console** — output to the user

And it has **real business logic worth isolating**: streak calculation,
determining whether a habit is "due" today, weekly aggregations. This is the
stuff you want to unit-test without touching the filesystem or the system clock.

## Suggested architecture

Define a capability per dependency as its own type class:

```haskell
class Monad m => MonadHabitStore m where
  loadHabits :: m [Habit]
  saveHabits :: [Habit] -> m ()

class Monad m => MonadClock m where
  currentDay :: m Day

class Monad m => MonadLogger m where
  logInfo :: Text -> m ()
```

Then your business logic functions get constraints like:

```haskell
checkHabit :: (MonadHabitStore m, MonadClock m, MonadLogger m)
           => HabitName -> m ()
```

You'll end up with roughly these layers:

- **Domain types** (`Habit`, `Streak`, `CheckIn`) — pure, no IO
- **Pure logic** (`calculateStreak`, `isDueToday`) — plain functions, trivially testable
- **Capabilities** (the type classes above) — the seam between logic and the world
- **App monad** — a `newtype AppM a = AppM (ReaderT Env IO a)` with instances implementing the capabilities against real IO
- **Test monad** — something like `newtype TestM a = TestM (State TestWorld a)` with instances backed by in-memory maps and a fixed date, so tests run without IO

## Concepts you'll practice

- **The "tagless final" style** (capability type classes) vs. a concrete `ReaderT Env IO` — you can try both and feel the tradeoffs
- **Why abstracting the clock matters** — try writing a streak test without it and you'll see immediately
- **Monad transformer stacks** via `ReaderT` over `IO`, and why `ReaderT IO` is often the practical sweet spot ("ReaderT design pattern" by Michael Snoyman is worth reading once you've started)
- **Separation of decision from effect** — functions that return "what should happen" vs. functions that do it

## Suggested build order

1. Domain types + pure streak logic with HSpec tests (no IO at all yet)
2. Define the capability classes
3. Write business logic against the classes
4. Implement real instances in `AppM` with JSON persistence via Aeson
5. Implement test instances and write integration-style tests
6. Wire up a CLI with `optparse-applicative`

Once it works, a great follow-up exercise is swapping JSON for SQLite — if your
architecture is right, only the `MonadHabitStore` instance should change.

## About the tagless final style

"Tagless final" is a technique for representing effectful programs where
**effects are expressed as type class constraints** on a polymorphic monad `m`,
rather than as a concrete data type (like a free monad AST) or a fixed
transformer stack.

The name comes from a 2009 paper by Carette, Kiselev, and Shan ("Finally
Tagless, Partially Evaluated"). "Tagless" because, unlike an interpreter that
pattern-matches on tagged constructors (`Add x y`, `Lit n`, ...), you never
build or inspect such tags — the program *is* the composition of method calls.
"Final" because you're working with the final denotation (the meaning)
directly, not an intermediate syntax tree.

### The core idea

Instead of writing:

```haskell
checkHabit :: HabitName -> ReaderT Env IO ()
```

which hardcodes *how* effects are performed, you write:

```haskell
checkHabit :: (MonadHabitStore m, MonadClock m, MonadLogger m)
           => HabitName -> m ()
```

The function says *what* capabilities it needs, not *how* they're implemented.
Any monad `m` that satisfies those constraints can run this code. You pick the
implementation by choosing which monad to run in:

- In production: `AppM`, whose instances hit the real filesystem and `getCurrentTime`
- In tests: `TestM`, whose instances use an in-memory `Map` and a fixed date
- In a dry-run mode: a third monad where `saveHabits` is a no-op that just logs

The business logic never changes. Only the interpretation does.

### Why people like it

- **Testability without mocks** — a test instance is just a normal type class instance. No IO involved, no mocking library needed, tests are fast and deterministic.
- **Precise effect tracking in types** — a function's signature tells you exactly which capabilities it uses. `calculateStreak` has no constraints, so you know at a glance it's pure. `checkHabit` lists its effects explicitly.
- **Composability** — capabilities combine freely. A function needing both `MonadClock` and `MonadHabitStore` just lists both, no transformer plumbing.
- **No AST overhead** — unlike free monads, there's no intermediate data structure being built and interpreted. It's just function calls; GHC can inline and optimize normally.

### The tradeoffs to feel for yourself

- **Instance boilerplate** — every capability needs an instance per monad. For a handful of capabilities and two monads (real + test), it's fine; at scale it grows.
- **"n × m" problem** — n capabilities and m monads means n×m instances. Usually manageable, sometimes annoying.
- **Constraint creep** — it's tempting to add "just one more" capability to a function's signature. Discipline helps.
- **Often overkill** — for many real apps, a plain `ReaderT Env IO` with functions-in-a-record (the "handle pattern" or "ReaderT design pattern") gives you 80% of the benefit with less ceremony. Part of the value of this project is building intuition for *when* tagless final is worth it and when it isn't.

### A tiny concrete example

```haskell
-- Pure logic, no constraints needed
calculateStreak :: [CheckIn] -> Day -> Int
calculateStreak checkIns today = ...

-- Effectful logic, constraints declare what's needed
checkHabit :: (MonadHabitStore m, MonadClock m, MonadLogger m)
           => HabitName -> m ()
checkHabit name = do
  today   <- currentDay
  habits  <- loadHabits
  let updated = markChecked name today habits
  saveHabits updated
  logInfo ("Checked: " <> unHabitName name)
```

`markChecked` is a pure function. `checkHabit` orchestrates effects but
contains no IO itself — it's polymorphic. The IO happens only when you pick a
concrete `m` and run it.
