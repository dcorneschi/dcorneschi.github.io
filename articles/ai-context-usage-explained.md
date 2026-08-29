# Understanding Context Usage in AI Assistants

When you work with an AI coding assistant or chat model, everything it "knows" in the moment lives in its **context window** — a fixed-size working memory measured in tokens. Understanding what fills that window, how it's consumed, and how to manage it is the difference between crisp, accurate answers and vague, forgetful ones. This article explains context usage in practical terms for anyone using LLM-based tools (Kiro, Claude, ChatGPT, Copilot, etc.).

> The model has **no memory between requests** beyond what's placed in the context window on each call. It doesn't "remember" your project — every turn, the tooling re-sends whatever context it wants the model to see.

## What Is the Context Window?

The context window is the maximum amount of text (input + output) a model can consider at once, counted in **tokens**. A token is a chunk of text — roughly **¾ of a word** in English, or about **4 characters** on average. "unhappiness" might be 3 tokens; code and punctuation tokenize differently than prose.

- The window covers **both** what you send (prompt, files, history) **and** what the model generates (its reply). They share the same budget.
- Window sizes vary by model — from a few thousand tokens to hundreds of thousands or more. Bigger isn't automatically better: larger windows cost more and can dilute focus.

```
[ system prompt ][ tools ][ steering/rules ][ conversation history ][ attached files ][ your message ] -> [ model's reply ]
\___________________________________ input tokens ___________________________________/    \_ output _/
                              all counts toward the same window
```

## What Consumes Context

Everything the tool feeds the model on a given turn uses budget:

| Consumer | What it is |
|----------|-----------|
| **System prompt** | The hidden instructions defining the assistant's behavior |
| **Tool definitions** | Descriptions/schemas of the tools the model can call |
| **Steering / rules** | Project standards and preferences injected into context |
| **Conversation history** | Prior messages in the session (yours and the assistant's) |
| **Attached files / code** | Files you reference, open, or paste in |
| **Tool results** | Output from commands, searches, file reads the model requested |
| **Your current message** | The new prompt you just sent |
| **The reply** | The model's generated response (reserved from the same budget) |

In agentic tools, **tool results are often the biggest consumer** — reading a large file, a long command output, or a big search result can eat thousands of tokens in one step.

## Why It Matters

- **Truncation / forgetting**: when the conversation exceeds the window, the oldest content is dropped or summarized. The model can "forget" earlier decisions, file contents, or instructions.
- **Cost and latency**: most APIs bill per input+output token, and more tokens generally mean slower responses.
- **Focus and accuracy**: a window stuffed with marginally relevant material can bury the important parts ("lost in the middle" — models attend less reliably to content buried in a long context). Precise, relevant context usually beats maximum context.

## Reading the Context Indicator (Usage %)

Many tools show **context utilization** as a percentage — the share of the window currently in use. "60% usage" means ~60% of the token budget is occupied (e.g. ~60K of a 100K-token window) by history, file contents, system/tool prompts, and outputs.

| Usage | Status | What it means |
|-------|--------|---------------|
| 0–30% | Excellent | Plenty of room; optimal performance |
| 30–60% | Good | Comfortable; room for complex, multi-file operations |
| 60–80% | Moderate | Still functional, but approaching limits |
| 80–95% | High | Older context may start getting truncated/summarized |
| 95–100% | Critical | Significant context loss; reasoning space squeezed |

So **60% is a healthy range** — the model can still see the full conversation with room to spare. As you climb past ~80%, expect the tooling to drop or summarize the oldest turns; past ~95%, reasoning quality can suffer because there's little room left for the model to "think."

**When to start fresh or compact:**

- Usage climbs above ~85%.
- The model starts forgetting earlier decisions or file contents.
- Responses noticeably degrade in quality.
- You've switched to a completely different topic or task.

## How Tools Manage Context

Modern assistants use several techniques so you don't hit the wall constantly:

- **Compaction / summarization**: when the window fills, older turns are condensed into a summary that preserves key facts while freeing tokens. (In Kiro, `/compact` does this on demand.)
- **Selective retrieval**: instead of loading whole files, the tool reads only the relevant ranges or searches for specific symbols.
- **Reserved output budget**: the tool holds back space so the model always has room to answer.
- **Session boundaries**: starting a fresh session clears history; per-directory persistence reloads only what's relevant.

## Managing Context Effectively

Practical habits that keep answers sharp:

1. **Be specific about scope.** Point the assistant at the exact file, function, or lines rather than "the whole repo."
2. **Start fresh for new tasks.** A long thread about topic A pollutes a question about topic B. Open a new session or compact.
3. **Prune what you attach.** Don't paste a 2,000-line file when 40 lines are relevant.
4. **Summarize long threads yourself.** A short "here's where we are" recap can be more reliable than trusting auto-truncation.
5. **Watch the context indicator.** Many tools show remaining context; compact or trim before it's exhausted.
6. **Keep steering/rules lean.** Always-on instructions cost tokens on every turn — include only what genuinely applies.
7. **Prefer references over dumps.** Let the tool retrieve what it needs rather than front-loading everything.

## Quick Reference

| Concept | Takeaway |
|---------|----------|
| Token | ~¾ word / ~4 chars; the unit of context |
| Context window | Fixed input+output budget for one request |
| No cross-request memory | Only what's in the window is "known" |
| Biggest consumers | Tool results, attached files, long history |
| Overflow behavior | Oldest content dropped or summarized |
| "Lost in the middle" | Buried content gets less attention — keep it relevant |
| Compaction | Summarize to free space (e.g. `/compact`) |
| Best practice | Precise, relevant context > maximum context |

## Related

- [Kiro CLI Cheatsheet](articles/kiro-cli-cheatsheet.md) — an AI terminal tool with per-directory context and slash commands
- [VS Code Git Actions and Git CLI Equivalents](articles/vscode-git-cli-equivalents.md) — another day-to-day dev tool reference
