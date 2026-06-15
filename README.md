# Weather Markets Trading Bot

An automated quantitative trading bot for Polymarket's daily high-temperature
markets. It builds its own probabilistic temperature forecast from several
independent weather sources, compares that forecast against the prices the market
is quoting, and trades the brackets where it has a measurable edge. It manages
every position through to settlement, behind a hard risk gate. Built and run with
real capital.

> **About this repo:** this is a technical overview of a private project. The
> source code is not public because the bot trades with real money and its edge
> lives in the data and the calibration, not in the plumbing. Happy to talk
> through the architecture in an interview. Published for portfolio purposes only.

## The market

Polymarket lists daily markets on the high temperature of a city, split into
degree brackets (for example "high of 72 to 73F"). Exactly one bracket wins and
pays out, the rest settle at zero. The market prices each bracket as an implied
probability. If you can estimate the real probability of each bracket better than
the market does, that gap is the edge.

## What it does

1. Forecasts tomorrow's high for each tracked city as a probability distribution,
   not a single number.
2. Turns that distribution into a probability for each market bracket.
3. Compares its own probability against the market price and finds the brackets
   where the difference is large enough to be worth trading.
4. Sizes each position against its own confidence and a strict risk budget, places
   limit orders, and then manages them: repricing, hedging, re-bidding when
   outbid and exiting early when live weather turns against the position.
5. Learns from every resolved market and adjusts which cities it trusts and how
   hard it sizes them.

## Architecture

```
   THREE INDEPENDENT SIGNALS
   +-- ensemble models      (multi-model probability spread)
   +-- deterministic source (independent cross-check)
   +-- live METAR           (the observations the market settles on)
                 |
                 v
   forecast   -> per-bracket probability distribution
                 |
                 v
   decision   -> edge = our probability minus market price
                 |
          +------+---------------------+
          v                            v
   +-------------+              +----------------+
   |   brain     |  grades      |   risk gate    |  locks a margin
   | (learning)  |  each city   | (every order)  |  on every order
   +------+------+              +--------+-------+
          ^                              |
          |                              v
          |                   +-----------------------+
          |                   |   execution engine    |
          |                   | place / reprice /     |
          |                   | hedge / outbid / exit |
          |                   +-----------+-----------+
          |                               |
          |                               v
          |                    +----------------------+
          +----- resolutions --+  Polymarket CLOB      |
                               |  (REST + WebSocket)   |
                               +----------------------+
```

The system is layered: signal, decision, risk, execution, learning. Each layer
has one job and hands a clean result to the next.

## The three signals

Rather than one forecast with backups, it runs three sources that each do
something different:

- **An ensemble forecast** that pulls every member of an independent multi-model
  ensemble. The spread between members is a real probability distribution for
  tomorrow's high, not a single guess.
- **A second, independent provider** used as a cross-check. Because it gives one
  deterministic forecast, the bot reconstructs a distribution around it using that
  city's own historical forecast error, so the same math still applies. When the
  two providers disagree, the bot trusts the market less and itself less.
- **Live airport observations (METAR)**, the exact data the market settles on. The
  bot polls these through the day, tracks the running daily high, and projects the
  final peak from the current heating rate. This is what lets it get out before a
  position goes bad instead of after.

Everything is computed in one unit internally and rounded the way the market
actually settles, so the model always speaks the market's language.

## The interesting parts

- **One risk gate every order has to pass.** A spread of brackets where exactly
  one wins is only entered if its total cost leaves a guaranteed margin. Entry,
  hedge, reprice and re-bid all go through the same check, so a model bug or a
  fast competitor can never quietly place a losing trade. The safety property is
  structural, not something each call site has to remember.
- **A self-learning layer that grades cities.** It blends a Bayesian win rate from
  real resolutions with how stable the forecasts are and how much the sources
  agree. A city only starts trading once it has earned enough confidence, and the
  sizing is tuned per city. The Bayesian prior means a brand-new city is not
  judged on a single lucky or unlucky day.
- **Dual outbidding.** It reacts the instant someone outbids over a live order-book
  stream, with a slower polling loop as a fallback if the stream misses an event.
  Either way the new bid still has to clear the risk gate.
- **Settlement-fidelity discipline.** It polls the same observation source the
  market resolves on, resets the daily high at the right moment, and rounds the
  way the contract rounds. It models how the contract actually settles, not an
  approximation of the weather.
- **Built to run unattended.** Simulation mode is on by default, so the whole flow
  can run end to end without touching the exchange. Orders are persisted to disk
  so a restart resumes cleanly, and shared state is guarded for thread safety.

## What it's built with

Python 3. Polymarket CLOB and Gamma APIs over REST and WebSocket. Public weather
APIs: a multi-model ensemble, an independent deterministic provider and aviation
METAR observations. Threaded execution with persisted state. Around 9,000 lines
across cleanly separated layers.

## Why it exists

A prediction market is only as sharp as the crowd pricing it. Weather is one of
the few domains where a small operator can genuinely hold a better signal than the
market, because professional numerical weather models are public and the market
mostly prices off a single forecast. The hard part is not getting a forecast, it
is knowing how much to trust it and turning that into correctly sized,
risk-bounded positions. Getting that calibration right is where the real work is.

---

Contact: kizozr@gmail.com
