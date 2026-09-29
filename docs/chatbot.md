# 02 · Chatbot

A conversational agent that answers questions from the ingested documents and cites the source file and page for every answer.
A second LLM grades each draft. Answers without a valid citation never reach the user and are escalated to a human instead.

← [Back to overview](../README.md)

![Chatbot workflow](../images/chatbot-workflow.png)

---

## Use cases

The workflow is domain-agnostic. Only the system prompt and the tool description need to be adapted:

| Use case | Documents | Example question |
|---|---|---|
| **Customer / technical support** (primary) | product manuals | "What does error code F3 mean?" |
| **University & learning** | lecture slides, scripts | "How is backpropagation explained in lecture 4?" |
| **Internal knowledge base** | policies, handbooks | "How many vacation days carry over?" |

The included configuration is set up for the support scenario.

---

## Flow

```
Chat Trigger
  → Support Agent (RAG)  ⇄  Tool: Search Manuals
        │                     ├─ Embeddings: Mistral
        │                     └─ Reranker: Cohere (top 20 → top 5)
        ├─ LLM: Mistral Medium
        └─ Memory: Window Buffer
  → Judge: Grade Answer (Mistral Small)
  → Passed?
        ├─ PASS → Return Answer
        └─ FAIL → Escalate: Email Support → Return Fallback
```

---

## Components

### 1 · Self-correcting agent
The vector store is connected as a **tool**, not as a fixed retrieval step. The agent decides when to search, evaluates the results and reformulates the query (synonyms, broader terms) if they don't answer the question.

| Setting | Value |
|---|---|
| Model | Mistral Medium |
| Max searches (prompt) | 3 |
| Max iterations (hard cap) | 5 |
| Memory | Window buffer per session, enables follow-up questions |

The hard cap sits slightly above the prompted limit (3 searches + final answer + buffer), so it acts as an emergency brake rather than cutting off normal answers.

**Citation format** required by the system prompt:

```
[Manual.pdf / Seite(n): 7, 12]
```

If nothing relevant is found after 3 searches, the agent must reply with a fixed "no information found" sentence instead of guessing.

### 2 · Retrieve wide, rerank narrow

| Stage | Setting | Cost |
|---|---|---|
| Vector search | top 20 candidates | no LLM tokens |
| Cohere `rerank-multilingual-v3.0` | keeps top 5 | cheap, specialized model |
| Agent context | 5 chunks (~7,500 chars) | LLM tokens |

Fetching wide increases the chance that the right chunk is included at all. Reranking narrow keeps the context small and focused. Pure similarity search on manuals tends to return many near-duplicate table fragments.

### 3 · LLM-as-judge
A separate LLM chain (Mistral Small) grades every draft against two criteria:

1. Does it give a concrete, factual answer to the question?
2. Does it contain a citation with **filename and at least one page number**?

The IF node checks:

```
{{ $json.text.trim().toUpperCase() }}  contains  "PASS"
```

`contains` instead of `equals` tolerates outputs like `PASS.` or `**PASS**`. Anything that is not PASS, including unexpected output, is treated as FAIL. The system fails safe toward human review.

A smaller model is sufficient here because the judge only performs a binary classification.

### 4 · Human handoff
On FAIL:

- The support team receives an email with the session ID, the original question and the rejected draft.
- The user receives an honest fallback message instead of an unverified answer.
- The email node is set to **continue on error**, so the user gets the fallback even if sending the email fails.

---

## Configuration notes

- **Chat Trigger response mode must be "When Last Node Finishes".** Streaming would send the agent's draft directly to the user and bypass the judge.
- **Escalation address:** replace `support@example.com` in the email node.
- **Adapting to another domain:** change the system prompt (role, e.g. "tutor for lecture slides"), the tool description ("search in manuals" → "search in lecture slides") and the fallback text. The pipeline itself stays unchanged.
- **Cross-node references** use `.first()` instead of `.item`. Each execution handles exactly one chat message, and `.first()` avoids item-pairing errors across the LLM chain and IF node. Conversation history is handled separately by the memory node.

---

## Example

**Question:**
> Wie reinige ich den Getränkekühler?

**Answer:**
> Trennen Sie das Gerät vom Netz ... Verwenden Sie keine scheuernden Reinigungsmittel ...
>
> [Manual.pdf / Seite(n): 14, 15]

---

## Known limitations

- The judge verifies that a citation *exists* and the answer is plausible, not that the cited page actually contains the stated information.
- Agent and judge come from the same model family, which can lead to correlated blind spots.
- The judge adds one LLM call per answer (~1–2 s latency).
- Searches run across all documents in the collection; there is no per-document filter yet.

## Roadmap

- Grounding check: re-retrieve the cited chunk and verify the claim against it
- Metadata filter by `filename` for multi-product or multi-course setups
- Log PASS/FAIL rates to detect gaps in the knowledge base
