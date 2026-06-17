# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pipenv install          # install dependencies (Python 3.10 required)
pipenv run python app.py  # start Flask dev server at http://localhost:5000
pipenv run pytest .     # run test suite
pipenv run black .      # format code
pipenv run isort .      # sort imports
pipenv run pylint <module>  # lint
```

## Required Environment Variables

Copy `.env.example` to `.env` and populate:

| Variable | Required | Purpose |
|---|---|---|
| `OPENAI_API_KEY` | Yes | GPT models for all chains and agents |
| `SCRAPIN_API_KEY` | Yes | LinkedIn profile scraping |
| `TAVILY_API_KEY` | Yes | Web search for agent profile discovery |
| `TWITTER_BEARER_TOKEN` / `TWITTER_API_KEY` / etc. | No (mock active) | Live Twitter data |
| `LANGCHAIN_API_KEY` / `LANGCHAIN_TRACING_V2` | Optional | LangSmith tracing |

Twitter data currently defaults to `scrape_user_tweets_mock()` in `ice_breaker.py` (reads from a hardcoded GitHub Gist), so Twitter API keys are not required.

## Architecture

The app takes a person's name, discovers their LinkedIn and Twitter profiles via AI agents, scrapes them, and returns a summary, topics of interest, and ice breakers.

**Request flow** for `POST /process?name=<name>`:

```
app.py
  -> ice_breaker.py: ice_break_with(name)
      |
      +--> agents/linkedin_lookup_agent.py
      |      ReAct agent (gpt-4o-mini) + Tavily search -> LinkedIn URL
      |
      +--> third_parties/linkedin.py
      |      Scrapin.io API (or mock) -> profile dict
      |
      +--> agents/twitter_lookup_agent.py
      |      ReAct agent (gpt-4o-mini) + Tavily search -> Twitter username
      |
      +--> third_parties/twitter.py
      |      Tweepy / mock -> list of tweet dicts
      |
      +--> chains/custom_chains.py  (all use gpt-3.5-turbo)
               get_summary_chain()     -> Summary (summary + facts[])
               get_interests_chain()   -> TopicOfInterest (topics[])
               get_ice_breaker_chain() -> IceBreaker (ice_breakers[])
      |
      returns: (Summary, TopicOfInterest, IceBreaker, photoUrl)
  <- JSON rendered by templates/index.html (vanilla JS)
```

**Key modules:**

- `ice_breaker.py` — main orchestration; edit here to change the overall pipeline
- `agents/` — LangChain ReAct agents for profile URL/username discovery; use `tools/tools.py` (Tavily wrapper)
- `chains/custom_chains.py` — three LCEL chains (`prompt | llm | parser`); prompt templates live here
- `output_parsers.py` — Pydantic output models (`Summary`, `IceBreaker`, `TopicOfInterest`) and their `PydanticOutputParser` instances
- `third_parties/` — external API integrations; each has a `mock` parameter or falls back to a GitHub Gist URL for local dev without real credentials
