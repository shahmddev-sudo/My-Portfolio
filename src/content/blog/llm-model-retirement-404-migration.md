---
title: "Ten OpenAI Model IDs Die on October 23rd"
description: "A pile of model ids I inherited from config files stop serving on a fixed date, and the only error you get is a bare 404. How to find every model id your repo references before the date lands."
pubDate: 2026-10-10
tags: [ai, llm, operations, dotnet]
draft: false
---

Ten of the model ids I inherited from config files stop serving on **October 23rd** — thirteen days from when I checked. `gpt-4-turbo`, `gpt-4-1106-preview`, `gpt-4o-2024-05-13`, `gpt-4.1-nano`, `o1-2024-12-17`, `o1-pro`, `o3-mini`, `o4-mini`. Six more go on December 11th.

Nothing in my code breaks today. The code keeps compiling, the tests keep passing, and every environment I check still answers. Then on a Tuesday afternoon a scheduled job starts returning 404s, and the only thing in the log is:

```text
The model `gpt-4-1106-preview` does not exist or you do not have access to it.
```

That message is the whole failure. It does not say "this id was retired", it does not say when, and it does not say what to use instead. It reads exactly like a typo in an API key. That ambiguity is what makes this worth writing down.

## The dates are real and they come from the provider

I pulled these from OpenAI's own deprecation page rather than a third-party list, and that turned out to matter. An aggregator I checked first had the right dates but **stale replacement model names** — it recommended `gpt-5.5` and `gpt-5.4-mini`, where OpenAI's page says `gpt-5.6-sol`, `gpt-5.6-terra`, and `gpt-5.6-luna`. Had I trusted the convenient summary, I'd have migrated to models that don't exist.

The October 23rd batch, from the provider page:

| Retiring | Replace with |
|---|---|
| `gpt-4-0613`, `gpt-4-1106-preview`, `gpt-4-turbo` | `gpt-5.6-sol` |
| `gpt-4o-2024-05-13` | `gpt-5.6-sol` |
| `gpt-4.1-nano` | `gpt-5.6-luna` |
| `o1-2024-12-17`, `o3-mini`, `o4-mini` | `gpt-5.6-sol` |
| `gpt-image-1` | `gpt-5.6-sol` |

And December 11th takes `gpt-5-2025-08-07`, `gpt-5-mini`, `gpt-5-nano`, `gpt-5-pro`, `o3`, and `o3-pro`.

This is not only OpenAI. Google's `gemini-2.5-pro`, `-flash`, and `-flash-lite` go on October 16th. Anthropic's `claude-haiku-4-5` goes October 15th, and `claude-opus-4-5` on November 24th. I count 39 announced retirements in flight across five providers right now, fourteen of them inside two weeks.

## The error is a 404, and that's the worst part

Every major provider fails a retired id the same way — HTTP 404, not 410 Gone, not a 400 with a helpful message:

- OpenAI → `404`, body mentions the model does not exist or you lack access
- Anthropic → `404`, `error.type: "not_found_error"`
- Google → `404 NOT_FOUND`, `models/gemini-pro is not found for API version v1`

404 is the wrong status for this. 404 means "you asked for a resource that isn't here" — true, but it says nothing about the fact that this specific id worked last month and will never work again. And because it shares a status with a genuine bad-id typo, your error handling can't distinguish "fix your config" from "this dependency is gone".

If your client has a generic 404 handler that retries or falls through to a default model, a retirement can quietly rewrite your behaviour instead of failing loudly.

## Find your ids before they find you

The ids that matter are not in your code. They're in the places nobody greps: a `.env` file that predates you, a Kubernetes ConfigMap, a Helm values file, a CI workflow that pins a model for a review step, a runbook, and one comment that someone copy-pasted a working curl command out of.

Model ids have a shape that's worth exploiting — they're provider-prefixed and hyphenated, with a version and often a date snapshot:

```bash
# Model ids hiding in tracked files
rg -n --hidden -g '!.git' \
   '\b(gpt|o[1-9]|claude|gemini|qwen|glm|llama|mistral|deepseek|grok|mimo)[a-z0-9.-]*-[0-9][a-z0-9.:-]*' \
   .

# and the ones that only exist as bare env values
rg -n --hidden -g '!.git' \
   '(MODEL|_MODEL|DEFAULT_MODEL|LLM_)[A-Z_]*=' . | grep -iE 'gpt|claude|gemini|qwen|glm|llama'
```

The first pass finds ids in code, config, docs and workflows. The second catches the bare `MODEL=gpt-4o` case where there's no prefix to key off — and those are the ones most likely to be ancient.

Run it against your whole history, not just `HEAD`, because the id that will bite you may have been removed from `main` six months ago and still be live in a branch, a tag, or a stale deployment:

```bash
git log --all -p -S'gpt-' -- '*.yaml' '*.yml' '*.env*' '*.json' '*.md' | rg -o 'gpt-[0-9a-z.-]+' | sort -u
```

## What I'd actually do about it

Wrap the id in config, not in code, so there's exactly one place to change. Then put the retirement date next to it, because a config value with no date is how you end up rediscovering this from a 404:

```json
{
  "Summarizer": {
    "model": "gpt-5.6-sol",
    "retired": null,
    "note": "replaced gpt-4o-2024-05-13, retired 2026-10-23"
  }
}
```

And give the 404 a specific branch in your error handling, because the generic handler will not save you:

```csharp
catch (ModelRetiredException ex) when (ex.StatusCode == 404 && ex.ModelId is not null)
{
    // Distinguish "typo" from "this dependency has a shutdown date".
    _logger.LogError(ex,
        "Model {ModelId} returned 404. It may be retired — check the provider's "
        + "deprecation page and the `retired` field in appsettings.",
        ex.ModelId);

    throw; // Fail loudly. Do not silently fall back to another model.
}
```

The fallback is the part I'd argue hardest about. A silent default model turns a two-minute fix into an incident where nobody notices for a week, and by then your outputs have quietly changed shape.

## The judgment rule

**Treat a 404 on a model id as a dependency with an expiry date, not a typo — and go to the provider's page, never an aggregator, to find the replacement.**

An aggregator got the dates right and the replacements wrong for me this month. That's not a small error: it points you at model ids that don't exist, so you trade a scheduled outage for an immediate one.

The same caution applies to the tracker I leaned on. It reported its own headline count as 39 models across 5 providers on one fetch and 33 across 4 on another the same day, and it lists some never-announced models with a sentinel date of 2098-12-31 — a 26,000-day deadline that means "we don't know", not "you have time". Useful as a starting index. Not something to migrate against.

---

*Dates and replacements verified against OpenAI's deprecation page on 2026-10-10. Other providers' dates come from their own documentation; where a third-party tracker disagreed, I noted the conflict rather than picking a winner silently. Re-verify before you migrate — this post will go stale.*