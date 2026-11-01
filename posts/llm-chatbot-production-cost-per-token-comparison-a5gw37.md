# LLM Chatbot Production Cost per Token Comparison Explained: Quality Floors

Short answer: the cheapest production token is the one that does not force a second answer. For an in-app chatbot answering questions over a private developer-tools knowledge base, a per-token comparison is useful only after every candidate passes the same structured-output contract. Measure valid answers, repair attempts, retrieval payload size, and latency together. A low input rate can lose its advantage when the model emits malformed JSON, asks for a retry, or receives a bloated context window.

The practical unit is not a token. It is a usable answer.

Keep a ledger.

## How should an LLM chatbot compare production cost per token?

A chatbot response can look fluent and still be operationally unusable. Our application needs an answer, citations to internal documents, a confidence label, and a refusal when the knowledge base has no support. Those fields are not decoration: downstream code uses them to render the response and decide whether to open a human-review path.

I initially treated token cost as a simple multiplication: prompt tokens plus completion tokens, multiplied by a published rate. That model answered the invoice question and missed the product question. A 900-token context that produces a valid answer on the first attempt may be preferable to a 500-token context that needs a repair pass. The comparison has to include the work needed to reach a valid result.

Before comparing providers or model families, freeze the contract. For example, require an object with `answer`, `sources`, `confidence`, and `needs_review`; reject extra keys; and reject a source that is not present in the retrieved set. JSON syntax alone is not enough.

## Make structured correctness the first gate

A small validation loop makes the hidden cost visible. The example below is deliberately provider-neutral. It assumes the runtime exposes a callable that returns text, while the application owns schema validation and the retry policy.

```python
import json


def parse_reply(raw_text, retrieved_ids):
    try:
        reply = json.loads(raw_text)
    except json.JSONDecodeError:
        return None

    required = {"answer", "sources", "confidence", "needs_review"}
    if set(reply) != required:
        return None
    if not isinstance(reply["sources"], list):
        return None
    if any(source not in retrieved_ids for source in reply["sources"]):
        return None
    if reply["confidence"] not in {"high", "medium", "low"}:
        return None
    if not isinstance(reply["needs_review"], bool):
        return None
    return reply


def answer_with_budget(call_model, prompt, retrieved_ids, max_attempts=2):
    attempts = 0
    while attempts < max_attempts:
        attempts += 1
        raw_text = call_model(prompt)
        reply = parse_reply(raw_text, retrieved_ids)
        if reply is not None:
            return reply, attempts
        prompt += "\nReturn only the required JSON object and use retrieved source IDs exactly."
    return {"answer": "No verified answer available.", "sources": [],
            "confidence": "low", "needs_review": True}, attempts
```

The important number here is not `max_attempts` by itself. It is the share of requests that consume the second attempt, plus the tokens spent on the failed output. Log both. A strict contract can raise the apparent cost while lowering the cost of bad answers that leak into the product.

## Build a comparison ledger, not a price table

For each test question, record input tokens, output tokens, retrieval tokens, validation result, attempt count, and time to a valid object. Keep the retrieved document IDs fixed across candidates. Otherwise, a token comparison quietly becomes a retrieval comparison.

The ledger should have separate rows for direct answers, ambiguous questions, missing-document questions, and questions whose answer spans several documents. Those cases expose different failure modes. A short direct answer can hide an expensive failure on the long-context case; a refusal test can reveal whether the system invents a source when evidence is absent.

For a concrete test, take one question whose answer is split between two internal pages and one question whose answer is absent. Give each candidate the same retrieved IDs, then inspect the object before counting it as a success. The first case tests whether the answer can cite both pieces without padding the prompt with an entire manual. The second tests whether the runtime can leave `sources` empty and set `needs_review` rather than filling the gap with a plausible sentence. This is slower than collecting a single average, but it tells the team what the tokens purchased: grounded structure, a bounded refusal, or another attempt. That distinction matters more than a neat cost-per-token column once the chatbot is in a user-facing screen.

I would set the decision rule before looking at totals: discard any candidate below the structured-correctness floor, then compare effective spend among the survivors. Effective spend includes failed attempts and the context required to produce a valid answer. It also includes embedding work when documents are prepared or refreshed; the embeddings guide describes embeddings as vector representations used for tasks such as search, so indexing and query representation belong in the system ledger rather than being treated as free setup.

This is a useful place for an eval harness. Keep a versioned set of questions and expected source IDs, run the same prompt and validator, and report distributions rather than one average. The median can look healthy while a small set of long-context requests dominates failures.

The first trap is context padding. Sending whole documents feels safe, but it increases input tokens and gives the model more irrelevant text to reconcile. Chunking, source selection, and a maximum evidence budget are product decisions, not details to hide inside a billing calculation.

The second trap is repair inflation. If malformed output triggers a full second generation, the original completion was not cheap. A bounded repair attempt is easier to reason about than an open retry loop, and a final review state is preferable to silently accepting an object with invalid sources.

The third trap is schema drift. A prompt can say “return JSON” while the application expects fields, types, and allowed values that were never checked together. Treat the schema as an executable boundary. Store validator failures by category so prompt changes can be tied to real regressions. With JSON Schema Draft 2020-12 available as a published specification, the team can keep the contract reviewable instead of hiding it in prompt prose.

Prompt design also has a cost dimension. Put stable instructions in a form the runtime can reuse where supported, keep retrieved passages compact, and avoid repeating formatting prose on every turn when the contract is already enforced in code. The goal is not the smallest prompt; it is the smallest prompt that preserves answer quality and a valid object.

The tradeoff is real. A strict validator and a second attempt add latency and tokens, so this method is a poor fit for a disposable exploratory chat where malformed structure has no downstream consequence. It is a good fit for a knowledge-base answer that another program must render, cite, or route to review.

| Strategy | Useful when | Main limitation |
| --- | --- | --- |
| Cheapest-first | A human reads every answer | Understates retries and review work |
| Quality-floor first | A product consumes structured fields | Can spend more tokens to reject bad output |
| Bounded fallback | A request needs a degraded answer path | More states to test and observe |

I use that table as a decision aid, not a leaderboard. The right row depends on what happens after generation.

## A decision rule that survives production

Choose the candidate with the lowest effective token spend after it clears the quality floor on every important case class. Publish the floor with the application contract: valid structured responses, source fidelity, refusal behavior, and an acceptable latency boundary. Re-run the ledger when the corpus, prompt, validator, or retrieval settings change.

That rule is intentionally less exciting than a cheapest-model chart. It remains useful after a model catalog changes, because it measures the work your application actually asks the runtime to do. In a private knowledge base, a correct refusal is cheaper than a confident unsupported answer, even when the refusal took a few more tokens.

The first useful benchmark is therefore small: a fixed question set, fixed retrieved IDs, a strict parser, and enough logging to count retries. Measure that before copying any per-token comparison.

## Further reading

- OpenAI, “Embeddings guide”: https://platform.openai.com/docs/guides/embeddings
- IETF, RFC 8259, “The JavaScript Object Notation (JSON) Data Interchange Format”: https://www.rfc-editor.org/rfc/rfc8259
- JSON Schema, “Specification”: https://json-schema.org/specification
- NIST, “AI Risk Management Framework”: https://www.nist.gov/itl/ai-risk-management-framework
