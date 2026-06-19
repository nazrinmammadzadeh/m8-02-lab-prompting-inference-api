# Prompts & Bake-off Notes

---

## Task 2 — Prompt-pattern bake-off

### Zero-shot prompt

The exact prompt string passed as `contents` to `client.models.generate_content()`.
System message used for all three styles: `"You are a support ticket classifier. Follow the instructions exactly."`

```
Classify the following customer support ticket into exactly one category:
billing | bug | feature_request | other

Ticket: {ticket_text}

Reply with only the category label and nothing else.
```

---

### Few-shot prompt

```
Classify the following customer support ticket into exactly one category:
billing | bug | feature_request | other

Here are labeled examples:

Example 1:
Ticket: "I was billed twice last month and need a refund."
Label: billing

Example 2:
Ticket: "Clicking the Save button shows a 404 error every time."
Label: bug

Example 3:
Ticket: "It would be useful to have keyboard shortcuts for the main actions."
Label: feature_request

Example 4:
Ticket: "Where can I find the documentation for the API?"
Label: other

Now classify this ticket:
Ticket: {ticket_text}

Reply with only the category label and nothing else.
```

---

### Chain-of-thought / decomposition prompt

```
Classify the following support ticket.

Category definitions:
  billing        — payments, charges, invoices, refunds, subscription costs
  bug            — broken functionality, errors, crashes, incorrect behaviour
  feature_request — requests for new or improved features
  other          — how-to questions, compliments, off-topic messages

Ticket: {ticket_text}

Think step by step:
  Step 1 — What is the main subject of this ticket?
  Step 2 — Does it involve money, a broken feature, a new feature, or something else?
  Step 3 — Which single category fits best?

Format your answer exactly as:
Reasoning: <one or two sentences>
Label: <category>
```

---

### Verdict

**Results table (from notebook, temperature = 0.1):**

| # | Ticket (summary) | Zero-shot | Few-shot | CoT | Ground truth |
|---|-----------------|-----------|----------|-----|--------------|
| 1 | Charged twice, wants refund | billing | billing | billing | billing |
| 2 | Export button 500 error | bug | bug | bug | bug |
| 3 | Dark mode request | feature_request | feature_request | feature_request | feature_request |
| **4** | **Can't find password reset link** | **bug ✗** | **other ✓** | **other ✓** | **other** |
| 5 | App crashes on Android 14 | bug | bug | bug | bug |
| 6 | Invoice copy request | billing | billing | billing | billing |
| 7 | PDF export request | feature_request | feature_request | feature_request | feature_request |
| 8 | Compliment on new UI | other | other | other | other |
| | **Accuracy** | **7/8 (87.5%)** | **8/8 (100%)** | **8/8 (100%)** | |

**Winner: few-shot and CoT (tied at 100%), with CoT providing stronger reasoning transparency.**

Zero-shot failed on ticket #4 ("How do I reset my password? I can't find the link anywhere.") — returning `bug` instead of `other`. The phrase "can't find the link" surface-matches the *bug* pattern (missing UI element) without examples to anchor what the *other* category looks like. Few-shot fixed this immediately: example 4 ("Where can I find the documentation?") is structurally identical to ticket #4, giving the model a direct template. CoT fixed it through a different mechanism — the explicit reasoning step forced the model to distinguish *"what the user is asking"* (how to perform an action) from *"how they phrased it"* (negative language about a missing element).

For the remaining 7 clear-cut tickets, all three methods agreed — demonstrating that on unambiguous support tickets, even zero-shot is sufficient. Few-shot is the recommended default: it adds ~120 tokens of overhead per call but eliminates the 12.5% error seen with zero-shot, and its reasoning is transparent from the examples rather than hidden inside a chain of thought.

---

## Task 3 — Structured output notes

**Gemini `gemini-2.0-flash`** with `response_mime_type="application/json"`:
- **8/8 responses** were syntactically valid JSON with all three fields correctly typed.
- `parse_json_safe()` and `validate()` applied **0 fixes** across all 8 tickets.
- Average confidence: **0.95**, with the lowest confidence on ticket #4 (0.87) — the most ambiguous case — which is a meaningful signal the model encodes uncertainty correctly.

**Local `llama3.2:3b` via Ollama** — tested on first 5 tickets:
- **3/5 responses** required fixes before the result was usable:
  - Ticket #3: JSON wrapped in prose (`"Sure! Here's the JSON: {...}"`) — fixed by regex extraction in `parse_json_safe()`.
  - Ticket #4: `confidence` returned as string `"high"` instead of float — fixed by `float()` coercion in `validate()`, clamped to 0.5.
  - Ticket #5: `label` = `"technical_issue"` (not in allowed set) — corrected to `"other"`, but this caused a **silent misclassification** (ground truth is `"bug"`).
- **Label accuracy** (5 tickets): Gemini 5/5, Ollama 4/5 — the invalid-label failure on ticket #5 produced a wrong answer that `validate()` could not recover because it had no way to know the intended label.

The critical lesson: robust JSON parsing handles structural failures (prose wrapping, wrong types), but it cannot fix a semantically wrong label from a model that ignores the category definitions. Schema enforcement at the API level (Gemini) prevents both categories of failure. For production classifiers, this makes hosted APIs with schema enforcement significantly more reliable than local models with prompt-based constraints.
