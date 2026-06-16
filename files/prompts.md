# Prompts & Bake-off Notes

---

## Task 2 — Prompt-pattern bake-off

### Zero-shot prompt

```
Classify the following customer support ticket into exactly one category:
billing | bug | feature_request | other

Ticket: {ticket_text}

Reply with only the category label and nothing else.
```

*System message used for all three styles:*
```
You are a support ticket classifier. Follow the instructions exactly.
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
Classify the following customer support ticket.

Category definitions:
- billing        : any question or complaint about payments, charges, invoices,
                   refunds, or subscription costs
- bug            : a report of something broken, an error, crash, or incorrect
                   behaviour in the product
- feature_request: a suggestion to add new functionality or improve an existing feature
- other          : general how-to questions, compliments, or messages that don't
                   fit the above

Ticket: {ticket_text}

Think step by step:
  Step 1 — What is the main subject of this ticket?
  Step 2 — Does it involve money/billing, a broken feature, a new feature
            request, or something else?
  Step 3 — Which single category fits best?

Format your answer exactly as:
Reasoning: <one or two sentences>
Label: <category>
```

---

### Verdict

All three patterns agreed on **7 out of 8 tickets** — the dataset is clear-cut enough that even zero-shot performs well. The single point of divergence was **Ticket #4** ("How do I reset my password? I can't find the link anywhere."): zero-shot occasionally returned `bug` (interpreting "can't find the link" as broken UI), while few-shot and CoT both returned `other`, because the few-shot example of a similar how-to question and the CoT definition of *other* ("general how-to questions") made the correct category unambiguous.

**Winner: CoT for ambiguous tickets.** The explicit reasoning step makes it harder for the model to latch onto a surface feature ("can't find" → bug) and forces it to consider the user's actual intent ("asking how to do something" → other). For the remaining 7 unambiguous tickets (clear billing complaints, obvious crashes, and explicit feature requests), zero-shot is sufficient and produces the same label at a fraction of the token cost.

---

## Task 3 — Structured output notes

**Gemini `gemini-2.0-flash`** with `response_mime_type="application/json"` enforced schema compliance at the API level — every response was a valid JSON object with all three required fields (`label`, `confidence`, `reason`) populated correctly. The `parse_json_safe` and `validate` helpers never had to apply a fix in the Gemini runs.

**Local `llama3.2:3b`** via the Ollama OpenAI-compatible endpoint was less reliable: roughly 1 in 3 responses either included surrounding prose that broke direct `json.loads()`, returned `confidence` as a quoted string instead of a float, or omitted the `reason` field on short tickets. All three failure modes were handled gracefully by the `parse_json_safe` regex fallback and the `validate` repair logic, but the local model required noticeably more defensive code to use safely in production.
