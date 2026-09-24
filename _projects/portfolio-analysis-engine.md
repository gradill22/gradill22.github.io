---
layout: page
title: LLM Trading Agent
description: Leveraging SKILL.state memory architecture to enable an LLM agent to manage a \\$1,000,000 paper portfolio using a custom quantitative finance MCP server.
img: assets/img/portfolio-analysis-engine.png
importance: 1
category: work
---

[Open the live analysis dashboard →](https://frontend-production-39dc.up.railway.app/)

## Abstract

By leveraging the SKILL.state memory architecture for LLMs, we can harness our own local LLM to manage its own stock trading portfolio. This program uses a paper trading portfolio from Alpaca while using a custom quantitative finance MCP server to handle portfolio forecasting and analysis.

This program can only place trades during normal trading hours: Monday through Friday from 9:30am EST to 4:30pm EST. This runs 100% locally, but still requires an internet connection for placing trades to Alpaca.

## Tools

* LM Studio
    * Model used: `qwen/qwen3.5-9b`[^3]
* Python (3.14.5)
* Alpaca

### Python Packages

* `alpaca-py` - Alpaca Python SDK
* `mcp`
* `numpy`
* `openai` - LLM Interface
* `pandas`
* `pydantic`
* `python-dotenv`
* `scipy`
* `yfinance` - Backup for Alpaca historical data collection

### System Hardware

This program runs 100% locally. Here are the specifications for the host development system[^2]:

* CPU Cores/Threads: 16/32
* CPU Core/Boost Clock: 4.2/5.7 GHz
* GPU VRAM: 12 GB
* GPU Core/Boost Clock: 2325/2587 MHz
* RAM: 64 GB

## Setup

**Overview:** This program leverages several technologies to enable the LLM agent to manage a portfolio. It uses the SKILL.state memory architecture to efficiently manage our LLM agent's memory without sacrificing performance. It connects to Alpaca for all of our stock market needs, including a \\$1,000,000 paper portfolio and access to historical data. It also has access to our custom quantitative finance MCP server that provides deterministic computations for analyzing stocks, mock portfolios, and our live portfolio to help our LLM agent consider risk, expected returns, and forecasting. To best work with SKILL.state, our program leverages state machines to handle overall workflow, directional decision making, and checkpoint enforcement. 

### Alpaca

We will leverage Alpaca for all of our portfolio and historical data needs. If, for whatever reason, Alpaca is insufficient at supplying historical data, we will use Yahoo finance (`yfinance`) as our backup provider. If neither Alpaca nor Yahoo finance are sufficient at providing historical data... well, it was obviously meant to be.

This program uses the Alpaca Python SDK (`alpaca-py`) to interact with the Alpaca service. It provides everything the program needs to execute trades, monitor our portfolio, check the status of our account, and download historical data.

### Custom Quantitative Finance MCP

LLMs are unreliable at performing math given the nature of their probabilistic output. Instead, we can leverage an MCP server to allow our LLM agent to call whatever analytical tool it wants. This way, our program can still reliably quantify our stock market data while leveraging its language engine to reason and make decisions.

The LLM should never make a calculation: it should only ever make a decision given pre-computed, determistic data.

Here, our program leverages the `mcp` Python package to build and host our custom MCP server. This enables our LLM to connect to our tools with their defined schemas. Our LLM agent will have access to field descriptions of what information is required for each tool.

Our MCP server is capable of working at the individual stock and overall portfolio levels. On each level, we can quantify basic summaries (returns, variance, current price from high, etc.), forecasts, measures of risk (e.g. Value at Risk), and expected returns via CAPM modeling.

### SKILL.state

A Google-sponsored research article from Purdue University proposed a new way to manage an LLM's context and memory with what they called SKILL.state[^1]. This leverages representing an environment as a state, or series of states, and giving an LLM the power to modify these states as it makes decisions toward completing a task. In other words, these states are mutable.

The paper[^1] publishes findings consistent with extreme memory efficiency and higher accuracy with respect to task completion compared to traditional conversation memory. Ultimately, the accuracy of the model is subjective and highly correlated to the difficulty and specificity of a given task. This program aims to focus on the efficiency of this memory paradigm rather than accuracy. We wouldn't be able to properly measure accuracy without some kind of look-ahead bias.

This program should demonstrate effective memory management for local LLMs without sacrificing performance. This will be especially important for cloud deployments of LLMs with respect to memory costs.

### State Machines

This program leverages state machines to seamlessly integrate with our LLM's memory architecture. After mapping out the entire process from observing the market to executing trades, the program defines 5 major states: OBSERVE, ANALYZE, DECIDE, EXECUTE, MONITOR.

Throughout the program cycle between states, the LLM agent will always have access to our live portfolio (updated every call), watchlist of known stocks, news/corporate event sentiment for each known stock, and current market hours (CLOSED, PRE-MARKET, OPEN, POST-MARKET). This data isn't included in memory when clasifying news and corporate events.

| | |
| :--- | :--- |
| **OBSERVE (Start)**<br>This program fetches several data feeds, namely stock discovery, recent news for known stocks, and corporate events for known stocks. Then, it spawns six instances of the selected LLM model to classify news articles and corporate events in parallel. The sentiment analysis is saved in a separate part of memory that the LLM agent won't see in full, though it has access to aggregated sentiment analysis for each stock. | **ANALYZE**<br>Given the known stocks and their overall sentiments, the LLM agent can begin analyzing which stocks to pick in the portfolio. The agent has several tools at its disposal for analysis via our custom quantitative finance MCP server. By default, the program analyzes every single stock in our watchlist unless otherwise specified by the LLM agent. The MCP server also provides optimized weights for a given suite of stocks when called upon by the LLM agent. |
| **DECIDE**<br>After analyzing a suite of stocks, our LLM agent will decide which stocks to buy/sell/hold. Here, the LLM agent can still play with mockup portfolios before making a final decision. | **EXECUTE**<br>The selected stocks our LLM agent picked are traded on Alpaca. This includes a rebalance of the current portfolio, closing positions that are no longer included, and padding the surviving stocks in our portfolio. |
| **MONITOR**<br>This is the final state in our program loop. This prevents the LLM agent from moving on to the OBSERVE state without first verifying that all of our orders filled and closed. | **FINISH**<br> This isn't a real state. Once we reach the end of the program loop, we start again at OBSERVE. |

### The portfolio

The program is connected to an Alpaca paper trading portfolio with a starting amount of \\$1,000,000. By default, the account uses 4x leverage for a total of \\$4,000,000 in buying power.

Although this simulates an enormous amount of capital, it will be more than plenty for a proof-of-concept of an LLM managing a portfolio. This is especially important for exploring how LLM agents handle large portfolios.

### System Prompts

There are two primary system prompts: the primary system prompt and the news classification prompt. They are still a work in progress and should not be taken as gospel.

#### Primary System Prompt

```{markdown}
You are an autonomous trading agent whose goal is to maximize your portfolio's
Compound Annual Growth Rate (CAGR) while minimizing risk on an Alpaca paper account.

When justifying how to trade a particular stock, weigh its respective news sentiment,
corporate actions, quantitative analysis, and recent portfolio performance (if already in our portfolio).
You may also trade crypto and options, but your primary focus is equities.
You have access to a small set of tools for market data, research, analytics, and portfolio management.

Each turn you are given:
1. The live PortfolioState (read-only, authoritative — never echo it back).
2. Market Memory (READ-ONLY): the watchlist, a sentiment_summary giving the
   program's catalyst-weighted NET news read per symbol (net_sentiment +
   a signed score in [-1,+1] where high-catalyst stories weigh more, plus
   article_count), and tracked_corporate_events. THE PROGRAM builds and maintains
   all of this from live data every cycle — you do NOT write to it. The raw news
   articles themselves are NOT in Market Memory: the program reads and classifies
   them behind the scenes and gives you only the distilled sentiment_summary, so
   weigh that per-symbol net read alongside the quantitative analysis when deciding
   what to trade — a strong bearish net read is a reason to avoid or trim a name
   even if the quant looks attractive, and vice versa. A symbol with no entry in
   sentiment_summary simply had no rated news this cycle — treat it as
   Neutral/unknown.
3. A RECENT ACTIONS trail: the last several actions you took and what they
   returned. This is how you remember what you already did.
4. The Observation from the tool you called last turn.

Each turn you choose exactly ONE next action. You do not maintain Market Memory —
the program refreshes the watchlist, news, and corporate actions for you before
each OBSERVE phase. Your job is to READ that data and DECIDE which names to trade.

DISCOVERY & WATCHLIST OWNERSHIP: the program discovers new candidates every cycle
and owns the watchlist (it mirrors your Alpaca watchlist and caps its size). You
never add symbols. The one watchlist edit you may make is 'prune_watchlist' during
OBSERVE — drop names you no longer want to consider (currently-held positions are
never pruned). Everything else about the watchlist is the program's job.

Your response is grammar-constrained to a fixed set of typed actions. There is
NO general-purpose or "other tool" escape hatch: if an action is not in the
catalogue, you cannot take it. To see the actions ALLOWED RIGHT NOW choose
'list_tools'; to see one action's exact fields choose 'describe_tool' with its
target_tool_name.

YOU WORK IN A FIXED WORKFLOW. Every turn the prompt shows your OUTER SESSION
(market open/closed — this gates equity trading; crypto trades 24/7), your
current WORKFLOW PHASE, the ALLOWED ACTIONS for that phase, and exactly what
MUST BE TRUE before you may advance. The phases cycle:
  OBSERVE  -> the program has already gathered watchlist news, corporate actions,
              and fresh discoveries into Market Memory. Review it, pull any live
              reads you still want (account, positions, orders, price summaries,
              snapshots, sector trends), and optionally 'prune_watchlist'.
  ANALYZE  -> run quant (forecast_returns, optimize_weights, portfolio_var,
              get_portfolio_risk, suggest_position_sizing, get_portfolio_cagr).
  DECIDE   -> commit a plan (run optimize_weights so the program commits
              the target). No orders here.
  EXECUTE  -> act: execute_rebalance runs your committed plan (the PROGRAM builds
              the orders — you do NOT re-type weights or sizes), or place a
              single discretionary order (place_stock_order / place_crypto_order
              / place_option_order), cancel_order_by_id, close_position.
  MONITOR  -> watch fills (get_orders, get_all_positions), then wrap to OBSERVE.

To move forward choose 'advance_phase' with the next phase. The PROGRAM decides
whether you may: if the phase's exit condition is not yet met, the phase does not
change and you are told exactly what is missing. You cannot skip phases forward.
'wait' is always allowed and never advances the phase.

YOU ARE NOT TRAPPED MOVING FORWARD. If work upstream turns out to be unsound —
a low-quality universe or stale data, an analysis you no longer trust, or a plan
that no longer fits the facts — do NOT force a bad trade through the pipeline:
- 'revert_phase' with an earlier target_phase moves you BACK to redo that step.
  It clears that phase's and every later phase's progress, so you genuinely rework
  it (e.g. revert to OBSERVE to rebuild the universe, or to ANALYZE to re-run the
  quant). target_phase must be strictly earlier than your current phase.
- 'restart_workflow' abandons the whole cycle and returns to OBSERVE with this
  cycle's data, analysis, and plan wiped.
Neither undoes orders already executed, and both are refused while orders are
still open for monitoring (wait for fills or the monitor timeout first). Prefer
reverting/restarting over advancing on poor inputs — a clean redo beats a bad trade.

ZERO DATA-CARRY: analytics tools fetch their own data — you pass only symbols and
knobs (gamma, lookback, alpha), never a price series. Portfolio tools get your
live positions injected automatically. execute_rebalance reads the committed plan
from program state — never transcribe numbers from an earlier observation.

Capabilities (subset exposed per phase):
- Account/positions/orders: get_account_info, get_all_positions, get_orders,
  cancel_order_by_id, close_position.
- Trade: place_stock_order (buy/sell/short; market/limit/stop/bracket/OCO/OTO/
  trailing-stop), place_crypto_order, place_option_order, execute_rebalance.
- Market data & research: get_price_summary, get_snapshot, get_sector_trends.
  (News and corporate actions are pre-collected into Market Memory by the program;
  there is no news/corporate-actions/most-actives action for you to call.)
- Watchlist: prune_watchlist (remove names only; the program owns all additions).
- Analytics (self-fetching): forecast_returns, optimize_weights, portfolio_var,
  get_portfolio_risk, suggest_position_sizing, get_portfolio_cagr.

DECISION PROCEDURE (every turn):
1. Read the WORKFLOW STATE block: what phase are you in, and what must be true to
   advance? Work toward that condition.
2. Look at RECENT ACTIONS. If an action already returned what you need, DO NOT
   repeat it — advance instead.
3. Do the current phase's job, then 'advance_phase'. In EXECUTE, prefer
   execute_rebalance to act on your committed plan; reserve a single
   place_*_order for a specific discretionary trade.
4. If there is genuinely nothing to do in this phase, 'advance_phase' (or 'wait'
   if you are waiting on the market). Never loop on analysis.
```

#### News Classification Prompt

```{markdown}
You are a financial news analyst. You are given a single news story and ONE
specific stock ticker it mentions. Judge the story ONLY as it affects THAT ticker.

Return two things:
1. sentiment — the story's likely effect on that stock's price:
   - "Bullish": the news is favorable for the stock (beats, upgrades, wins,
     approvals, strong guidance, buybacks, positive demand signals).
   - "Bearish": the news is unfavorable (misses, downgrades, lawsuits, recalls,
     probes, weak guidance, dilution, a rival's gain that hurts this company).
   - "Neutral": routine, ambiguous, already-priced, or not clearly directional.
   The SAME story can be bullish for one company and bearish for another — judge
   it for the ticker you are given, not for the market in general.
2. catalyst_rating — how market-moving the story is for THIS stock, 1 to 10:
   - 1-3: minor noise (routine coverage, small partnerships, reiterated views).
   - 4-6: moderate (analyst actions, product news, notable but not decisive).
   - 7-10: major catalyst (earnings surprises, M&A, guidance cuts/raises,
     regulatory decisions, executive shakeups, litigation outcomes).

Base your judgment only on the provided headline and summary. Do not invent facts.
```

## Challenges

There were several challenges along the way when developing this program.

### Schema Design

The goal of this program is to more efficiently provide the context the LLM agent needs to make decisions in our trading environment. So, these schemas can't have any expanding memory. Otherwise, that would defeat the whole purpose of this project.

This was especially challenging when we are exploring live stocks and their respective news feeds. How can we provide this critical data to the LLM agent without blowing up its memory? This was especially difficult to navigate given the size of the news and corporate events data relative to simple stock tickers.

### LLM Selection

LLMs are probabilistic by nature. They are language models, and like all models, they work within a certain margin of error when predicting the next token/byte/word.

Also, we need a model small enough to quickly provide answers but large enough to reason through its decisions. In the first iteration of this project, the model used for the LLM agent was far too big and slow. We need lean and speed without sacrificing quality decision making.

## Solutions

Given the challenges above, here are some of the solutions to navigate those barriers.

### Offload Expanding Data

Instead of throwing our entire news feed into the LLM agent's memory, we'll store the news feed in a separate part of the program. Then, we can provide aggregate data for each stock to get a general sentiment without providing all the headlines and feeds directly. This exponentially reduced our overall memory usage from loop-to-loop, maintaining high throughput and data integrity.

### Cherry-Picking Qwen 3.5

After experimenting with a few LLM models, the program assumed the Qwen 3.5 9B parameter model[^3]. It's about 10 GB in size and can be fully offloaded to the system GPU.

Here are some model specifications:
* Size: 10.4 GB
* Quantitization: Q8_0
* Parameters: 9 billion
* Context length: 262,144 tokens
* Supports: reasoning, external tools, images

Better yet, this model specifically handles JSON input and output, which is perfect for representing our states and schemas. Depending on how large our memory is (though there is not much variance), we can get 20-40 tokens/second. When we are classifying six news articles/corporate events in parallel, each process can get 10-15 tokens/second.

## Final Product

It's too early to tell how well our LLM agent manages its portfolio compared to some common benchmarks like the S&P 500, NASDAQ 100, and Dow Jones. But, we have a working end-to-end system that enables an LLM to manage a portfolio with efficient memory management without sacrificing performance.

If you want to see how the portfolio is doing, you can see a live analytics dashboard [here](https://frontend-production-39dc.up.railway.app/).

---

[^1]: https://arxiv.org/html/2608.26263v2
[^2]: https://pcpartpicker.com/user/gdillon/saved/#view=6QshmG
[^3]: https://huggingface.co/Qwen/Qwen3.5-9B
