---
date: 2026-09-29
title: "Conversation Anchors, Context Compression, and Summary Drift in Doubao"
tags: [ai, llm, agents, conversation, retrieval, context-window, prompt-engineering]
---

# Conversation Anchors, Context Compression, and Summary Drift in Doubao

## Background

I asked Doubao to summarize a long conversation from an earlier point rather than from the latest turn. The request was not simply "write a short summary". It required the system to identify a historical starting point, select the following range, and then summarize that range.

The Fast edition (极速版) repeatedly selected the wrong range. The Expert edition (专家版) eventually selected the intended range from the same visible conversation.

## Case Study: The Missing Middle

The conversation was organized roughly like this:

```text
Questions 1-4: Topic A
Questions 5-6: Star Wars
Question 7: What writing techniques can I learn from Star Wars?
Questions 8-13: How to write fiction
```

The intended writing-related range was therefore questions **7 through 13**. Question 7 is a bridge: its surface topic is Star Wars, but its purpose has already shifted to writing.

The Fast edition behaved as follows:

1. I asked it to summarize the recent group of questions about writing. It returned only questions 11, 12, and 13, leaving out 7-10.
2. I said that the result was incomplete and that earlier questions were missing. It returned question 4, which was about Topic A, followed by 11-13. Questions 7-10 were still absent.
3. I then tried to make the start more explicit by telling it: "start summarizing from 'What is the core point of writing an engaging story?'(question 7) " The result was still wrong.
4. After switching to the Expert edition, the system finally recovered the intended writing-related range.


| Attempt | Returned range | Result |
|---|---|---|
| Expected | [7, 8, 9, 10, 11, 12, 13] | Intended contiguous range |
| Fast attempt 1 | [11, 12, 13] | Missing the middle and the bridge |
| Fast attempt 2 | [4, 11, 12, 13] | Non-contiguous; an early unrelated turn reappears |
| Expert | [7, 8, 9, 10, 11, 12, 13] | Intended contiguous range recovered |

| Turn | Content | Feature |
|---|---|---|
| Q7 | Star Wars → writing techniques | Bridge turn; surface topic is Star Wars, intent is writing |
| Q8–Q10 | How to write fiction | Middle of the range; less lexically distinctive |
| Q11–Q13 | Recent writing questions | Recent tail; strongest writing-related keywords |

## The Actual Task

"Summarize from that question onward" is not one operation but four:

1. **Resolve the anchor.** Identify which historical question is meant.
2. **Define the boundary.** Decide whether that question is included.
3. **Select a contiguous range.** Keep everything from the anchor onward.
4. **Summarize the range.** Compress the frozen source without adding or dropping facts.

Only the last is ordinary summarization. The first three are indexing, retrieval, and range selection.

Robust:

```text
start_id = resolve_anchor(user_description)
selected = raw_messages[start_id : end]
summary  = summarize(selected)
```

Fragile:

```text
selected = retrieve_messages_similar_to(user_description)
summary  = summarize(selected)
```

The fragile version can return 11-13 without 7-10, or question 4 after a correction — because each correction triggers a new retrieval instead of continuing from a frozen cursor.

## Why This Happens

It is a context-assembly failure, not a summarization failure. Before the model sees anything, the system has already built a reduced view.

**Chunk-based retrieval.** If history is split into semantic chunks and the request is a similarity query, the result need not be contiguous. The recent tail (11-13) scores high. The bridge turn (7) may be chunked with 5-6 because its surface vocabulary comes from Star Wars. The middle turns (8-10) fall below the cutoff. A correction cannot restore them; it is a new query over the same index.

**Lossy compression.** If older history is replaced by a summary, the original wording of 7-10 is gone. LlamaIndex's `ChatSummaryMemoryBuffer` keeps the newest messages verbatim and summarizes the older ones into one synthetic message. If 7-10 are not in that summary, no later prompt recovers them exactly.

**Tail retention without a head.** If the window is trimmed from the front, recent turns survive and early turns drop. That explains a first answer of only 11-13, but not the later reappearance of 4. The reappearance implies an extra retrieval layer on top of truncation. What is missing is a stable, frozen starting point.

| System | Stable ID | Head | Middle | Anchor |
|---|---|---|---|---|
| Ollama | No | None | None | None |
| LangChain | Optional | None | Trim/summarize | None |
| AutoGen | No | Kept | Removed by token budget | None |
| LlamaIndex | No | Newest kept | Summarized (lossy) | None |
| Semantic Kernel | No | None | Truncate/summarize | None |
| Hermes Agent | Sessions | Head + bounded tail | Summarized | Anchor index + FTS5 |

Most frameworks expose a model-facing message list, not a conversation database. None provides "summarize from turn 17." Hermes is the positive control: it persists full history, protects a head and a bounded tail, and returns real messages around a verified anchor via FTS5 search plus `around_message_id` scrolling.

The product may store a full transcript with IDs and timestamps while the model receives a reduced list. The UI's full history is not proof that the full history reached the model.

A token position is not an application-level primary key. Unless the orchestration layer injects a stable `message_id` or `turn_id`, "start from the question about X" is a semantic matching task, not an indexed lookup.

## Best-Fit Explanation

```text
1. History is compressed, truncated, or selectively retrieved.
2. The recent tail (11-13) stays visible or scores highest.
3. The bridge turn (7) is classified by Star Wars wording, not writing intent.
4. The middle turns (8-10) are absent or below the cutoff.
5. The correction is a new query, so question 4 can appear.
6. No anchor was frozen, so 7-10 cannot be restored.
7. The Expert route likely has a larger context, a stronger retriever, or both.
```

Pure front truncation cannot explain the later appearance of 4. That points to an additional retrieval layer or a model-generated guess.

## Solution

Exact range selection is an application feature, not a model-native capability. Open-source implementations show why products often choose cheaper history policies by default: Ollama exposes a `role`/`content` message interface; LangChain makes message IDs optional; LlamaIndex keeps recent messages in full text while replacing older messages with a summary; and Semantic Kernel provides reducers that truncate or summarize history to control context size and retained data. [Ollama API](https://github.com/ollama/ollama/blob/main/docs/api.md#generate-a-chat-completion), [LangChain `BaseMessage`](https://github.com/langchain-ai/langchain/blob/master/libs/core/langchain_core/messages/base.py#L1399-L1406), [LlamaIndex summary memory](https://github.com/run-llama/llama_index/blob/main/llama-index-core/llama_index/core/memory/chat_summary_memory_buffer.py#L1132-L1184), [Semantic Kernel history reduction](https://github.com/MicrosoftDocs/semantic-kernel-docs/blob/main/semantic-kernel/concepts/ai-services/chat-completion/chat-history.md#chat-history-reduction)

These policies reduce prompt size and latency for ordinary chat, but they do not guarantee an exact chronological answer to an archival request. A product that does not support exact range selection should disclose that limitation rather than presenting a guessed range with confidence.

When the user explicitly asks for a faithful historical range, the product can opt into a more expensive archival path:

```text
raw transcript store
    -> exact + semantic anchor search
    -> candidate message IDs
    -> ambiguity check / user confirmation
    -> contiguous range expansion
    -> summary from the raw selected range
```

Hermes is a concrete reference for this design: it stores full sessions in SQLite/FTS5 and can search for a message, then read a window around its `message_id`. [Hermes sessions](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/sessions.md#how-sessions-work), [Hermes anchored search](https://github.com/NousResearch/hermes-agent/blob/main/tools/session_search_tool.py#L2616-L2695)

In this mode, the application resolves the description to a stable message ID outside the LLM, returns multiple candidates when it cannot decide, freezes the contiguous range, and summarizes the original messages rather than an earlier summary. If the product does not maintain this archival layer, it should ask the user to select or paste the anchor and clearly state that the range may be incomplete.

The durable lesson is: exact historical summarization should be an explicit workflow, not an assumption about ordinary chat memory.
